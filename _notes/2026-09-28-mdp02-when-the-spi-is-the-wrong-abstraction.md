---
layout: post
title: "When the SPI Is the Wrong Abstraction"
date: 2026-09-28
entry_type: note
subtype: diary
projects: [casehubio/blocks]
tags: [design, cognitive-architecture, yagni, attention]
series: issue-306-attention-override-policy
---

# When the SPI Is the Wrong Abstraction

`AttentionRelevance` is a package-private utility in blocks-core. Five `EnumSet<SignalCategory>` constants map cognitive domains to signal categories — GOALS gets URGENCY_SPIKE and GOAL_RECOGNIZED, MOOD gets AFFECT_CHANGE, and so on. `CognitionCore.isEnabled()` checks these mappings to force-enable prompt sections when attention signals arrive, even when the config flag is off.

blocks#306 asked for an SPI extraction: `AttentionOverridePolicy` interface, default implementation wrapping the current static mapping, constructor injection into CognitionCore. The motivation was real — a game runtime like wacky-manor might want AFFECT_CHANGE to also enable narrative, or suppress certain overrides entirely. Per-runtime tuning of which signals force-enable which cognitive sections.

I traced the consumer chain before designing anything. `SocialAvatarCognition` constructs CognitionCore with hardcoded `CognitionConfig.all()`. No game runtime currently customizes attention overrides — the attention system from #304 just landed. The future consumer exists but hasn't materialised yet.

The deeper problem: neocortex already has `CognitiveDefaults`, `CognitiveDefaultsRegistry`, and `CognitiveDerivationEngine` — the cognitive profile YAML system that derives per-agent parameters from personality. Attention override mappings are fundamentally configuration ("for this agent, AFFECT_CHANGE should also enable narrative"), not polymorphic behaviour requiring a code-level SPI. And #303 on the roadmap plans to migrate CognitionCore from blocks into neocortex, where those mappings would naturally become a field on `CognitiveDefaults`, derived per-agent, hot-reloadable via `CognitiveProfileWatcher`.

Building an SPI in blocks now would be temporary work replaced by the migration. The options were: close the issue, make the mappings constructor-injectable data as a lightweight stepping stone, or build the full SPI. Constructor-injectable data would give per-deployment tuning (wacky-manor gets one mapping, a business app gets another) but not per-agent tuning — that comes naturally from cognitive profiles after #303.

The decision: close #306 as superseded by #303. No temporary abstractions, no technical debt. When the migration lands, the problem solves itself through existing infrastructure. The prerequisites for #303 are nearly clear — #302 is being delivered in another slot, leaving only two S/XS wiring issues (#299, #300) before the migration can begin.
