---
layout: post
title: "Clearing the Runway"
date: 2026-10-01
entry_type: note
subtype: diary
projects: [casehubio/examples]
tags: [wacky-manor, refactoring, sealed-types, pre-test-hygiene]
series: issue-71-wacky-manor-pre-test-hygiene
---

# Clearing the Runway

The pre-test hygiene queue was four issues — dead code, over-engineered abstractions, and a type-safety refactoring that had been deferred since the trust evolution work landed. The goal was simple: get wacky-manor's codebase clean enough that behavioral test baselines would measure the game's actual behavior, not artifacts of stale code.

`ManorTrustEvents` was the easiest: a weight-based trust model that #65 had fully superseded with CDI events. The class was dead — no production callers. We deleted it, its test, and a disabled stub in `CharacterCognitionTest` that was the last trace.

`CognitiveBudget` was more interesting. It computed per-situation norm limits based on arousal, nearby agent count, and active goals. In theory this was attention-aware section gating. In practice, the only caller passed hardcoded arousal of `0.5` and `activeGoals` of `0`, making the dynamic computation dead arithmetic. The platform's `AttentionRelevance` in `CognitionCore` handles section gating properly — signals determine what context the agent needs, not a hand-coded budget. I removed the budget and let `selectNorms` return all norms ranked by relevance. The scoring still matters; the artificial count cap doesn't.

The `ManorEvent` sealed hierarchy was the substantial one. Twelve parameters, five constructors, and four optional string arguments that could be silently swapped with no type error. The issue described the problem well: adding a thirteenth parameter would make an already fragile interface worse.

The design choice was straightforward — a sealed interface with four record subtypes (`Action`, `Dialogue`, `Aside`, `Narrator`), each carrying only its relevant fields. Default methods on the interface return `null` or `false` for fields that don't apply to a given subtype, which means callers that access `.actionType()` or `.dialogueTarget()` on the interface type still work without casting. A `ManorEvent.of()` factory handles the string-based dispatch path for `WorldState.addEvent()`. The generics invariance issue in tests — `List.of()` inferring `? extends ManorEvent` for mixed subtype lists — needed explicit type witnesses (`List.<LevelEvent<ManorEvent>>of(...)`), which is standard Java generics.

The fourth issue — voice descriptions — was already implemented. Closed it and moved on.

What this clears the way for: behavioral test baselines. The directive-minimal YAML rewrite (#76) was intentionally deferred from this queue because it needs baselines first — stripping directives without measuring what changes is guessing. With the dead code and fragile types cleaned up, those baselines will test actual game behavior.
