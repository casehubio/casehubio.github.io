---
layout: post
title: "The Config That Couldn't Count to Twenty-Eight"
date: 2026-11-24
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
tags: [cognition, refactoring, multi-agent, pipeline]
series: issue-1221-pipeline-spine
---

# The Config That Couldn't Count to Twenty-Eight

CognitionConfig was a 28-boolean positional record. `all()` enabled 18 of 28 subsystems. `withAllSubsystems()` enabled all 28. Production used `all()`. Tests used `withAllSubsystems()`. Two composition paths — one for the real pipeline, one for tests — and no way to tell which subsystems the split was deliberately excluding vs which were just left off because someone lost count at position 19.

The fix was an `EnumSet<Subsystem>`. `all()` becomes `EnumSet.allOf()`. The 28 boolean accessors survive as delegate methods, so callers don't change. The massive `with(String, boolean)` switch — 28 cases, each manually threading 27 unchanged booleans through a constructor — collapsed into a three-line enum lookup. And `all()` now means all.

The harder problem was what `all()` enabling everything actually does to the pipeline. With `directivePrompts` suddenly true by default, every prompt section gets wrapped in `DirectiveSection`. Tests that checked `instanceof MoodPromptSection` start failing — the section is now wrapped inside a `DirectiveSection`. Tests that called `renderTaggedSections()` with null orchestrators started hitting NPEs, because sections that were previously invisible (no block tag without directive wrapping) were now rendered.

The cascade was methodical — disable `DIRECTIVE_PROMPTS` in tests that check unwrapped section types, update trace map keys from `"MoodPromptSection"` to `"MOOD"` where the trace reads block tags. Each fix was mechanical once the pattern was clear: the tests were testing section presence, not directive wrapping.

## Per-agent state: the fix that actually matters

`CognitionCore` had seven volatile fields — `lastDescriptor`, `lastAgentId`, `lastTenantId`, `lastBriefing`, `lastConsolidationArtifacts`, `lastContinuumMap`, `lastTargetAgentId` — all set during `tick()` and read during `promptSections()`. Tick agent A, the fields hold A's state. Tick agent B, the fields hold B's state. Call `promptSections()` for agent A — you get B's soft constraints, B's attention briefing, B's arousal-based tier filter.

The fix is a `ConcurrentHashMap<AgentKey, PerTickState>` keyed by `(agentId, tenantId)`. Each tick snapshots into the map. `promptSections(agentId, tenantId)` loads from the map before building sections. The volatile fields survive as the working set for the current tick — the section customizer closures still read from them, but they're populated from the correct agent's state before the customizer runs.

## Stale appraisal and the double tick

I gated the stale appraisal bridging. The appraisal tick participant runs in DERIVED phase and produces a result. On the next tick, if there's no observation, the participant returns its last result — from the previous tick. CognitionCore was bridging that stale result to mood and drives on every tick, causing drift without input. The fix: bridge only when `observation != null`.

The double `drives.tick()` was subtler. Pre-appraisal: `drives.tick()` computes drives. DERIVED phase runs (appraisal fires). Post-appraisal: `drives.recordAppraisalModulation()` stores deltas. Then `drives.tick()` again — EMA smoothing applied twice. Retention factor 0.3 becomes 0.3² = 0.09, so drives decay three times faster than intended. I gated the second tick on observation presence — no-observation ticks get a single tick with correct smoothing. Observation ticks still double-tick for same-tick reactivity; the full fix needs drive computation refactoring in SE6.

## The small fixes that add up

`DirectiveSection.wrap()` was using `getClass().getSimpleName()` to look up directives and block tags — but after entity rendering wrapping, the class name was `EntityRenderingSection`, not the real section. The unwrap now peels both `DirectiveSection` and `EntityRenderingSection` decorator layers.

`configureRegulation()` was resolving a COGNITIVE subgraph ID against `lastTenantId` at build time — which is null. It now passes null and lets the participant handle it.

`CognitionDefaultBeans` captured `Instant.now()` once at singleton construction for recency decay — memories created after startup never aged. A `Clock`-based overload to `ModulationFactors.recencyDecay()` fixes it.

The pipeline builder's gut-feeling ordering was backwards — `configureGutFeeling` ran before `configureCognitiveMemoryRetriever`, passing a null retriever to `GutFeelingParticipant`. Reordering the calls in `build()` fixed it.

The regex BDI JSON parser broke on `]` inside text values and rejected leading `+` in numbers. Jackson ObjectMapper replaced it.

Two items deferred: wiring `configureSearchBudget` and `configureCuriosity` into the builder (needs new builder fields and factory changes), and the quality signal timing in `recordInteraction` (needs deeper investigation of the mood-then-quality ordering).

SE1 was the critical path — SE6 (affect) and SE7 (orchestrators) can't start until this merges, because they touch the same `CognitionCore` file. SE2 through SE5 were independent modules that could run in parallel all along.
