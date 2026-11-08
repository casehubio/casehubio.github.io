---
layout: post
title: "Grounding Pattern Views in Real Types"
date: 2026-10-26
entry_type: note
subtype: diary
projects: [casehubio/casehub-mosaic]
tags: [typescript, web-components, agency, type-systems]
series: issue-277-agency-pattern-views
---

# Grounding Pattern Views in Real Types

The infrastructure from the first entry — PatternViewMixin, exhaustive dispatch, the `views/` directory convention — was the scaffold. This entry covers what happened when we started filling it in: fourteen pattern views across six agency categories, each one shaped by the actual Java types it reflects.

The first two waves went fast. Orchestration got four views: supervisor (hub-and-spoke routing), choreography (reactive blackboard), saga (forward steps with compensation arrows), decomposition (task decomposer with strategy badges). Deliberation got three: negotiation (party cards with acceptance policies), coalition (capability coverage), normative (competing rules with arbiter judgment). Governance got one: judgment (escalation chains). Each view follows the same structure — extend PatternViewMixin, guard on `!this.spec`, cast to the category subtype, render.

The pattern held until Wave 3, when it broke in an instructive way.

I asked Claude to build perception, optimisation, and social views based on the PatternCategory types already in mosaic-core. It produced six views that looked plausible — pipeline cards, config grids, retention displays. They compiled, they rendered, the gallery showed them off nicely. But when I cross-referenced the agency repo's actual Java interfaces, the types didn't match. `PipelineSpec` in Java has `source` and `levels` with grouping and summariser fields. The fabricated TypeScript version had flattened these into generic key-value pairs. The optimisation pipeline has a `confidenceScorer` and `safety` guardrail section. The fabricated version had neither.

We reverted everything and started from the Java source. Three new type files in mosaic-core — `perception.ts`, `optimisation.ts`, `social.ts` — each modelled directly from the agency sealed interfaces. `PipelineSpec` got its source/levels hierarchy. `PromptOptimisationPipeline` got signature, optimiser, config, safety, confidenceScorer, and examples. `CognitionConfig` got all thirteen cognitive module sections. `RetentionConfig` got its four weighted scoring dimensions.

The views rebuilt against real types looked different from the fabricated ones. The summarisation view became a vertical pipeline with source at top, level cards below showing grouping badges and summariser names. The observation view gained action tags and section cards with tier threshold bars — verbatim, grouped, summarised — because `WorldSpec` has a `RendererSpec` with tier configuration that the fabricated version didn't know about. The prompt-optimisation view gained a safety guardrails section and a confidence scorer badge because those exist in the real pipeline.

The negotiation view got the same treatment. The original from Wave 2 had hardcoded buyer/seller parties — a reasonable assumption that turned out to be wrong. The Java `NegotiationDefinition` uses named `NegotiationParty` objects with roles and `AcceptancePolicy` with configurable thresholds. The reworked view renders party cards by name, shows the acceptance policy badge with its threshold, and falls back gracefully when parties are absent.

A colour scheme emerged across the six categories: peach for orchestration, purple for deliberation, red for governance, teal for perception, amber for optimisation, pink for social. Each view renders its category badge in the matching colour. It's a small thing, but when agent-viewer displays them in sequence, the colour tells you which part of the agency taxonomy you're looking at before you read a word.

The responsive card fix was a quiet correction that affected every view. Agent cards were rendered with `width: 280px` — fine in a wide panel, overlapping in a narrow one. Changing to `min-width: 200px` with flexbox and `minmax(220px, 1fr)` grids let every view work at any panel width. The fix landed early and all subsequent views inherited it through the grid convention.

Fourteen views, six categories, one mixin. The PatternViewMixin from the first entry scaled exactly as hoped — each new view is a `render()` method with domain-specific HTML and CSS, nothing more. The interesting lesson was that the UI shapes are only as good as the types they reflect. When the types were fabricated, the views looked competent but were fiction. When the types were grounded in the Java source, the views gained structure and detail that made them useful.
