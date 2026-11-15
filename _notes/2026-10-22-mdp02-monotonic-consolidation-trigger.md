---
layout: post
title: "Building the Monotonic Consolidation Trigger"
date: 2026-10-22
entry_type: note
subtype: diary
projects: [casehubio/examples, casehubio/neocortex]
tags: [cognitive-architecture, memory-consolidation, CLS-theory, implementation, observability]
series: issue-661-monotonic-consolidation-trigger
---

# Building the Monotonic Consolidation Trigger

The previous entry (*Why Decay Is Wrong for Memory Consolidation Triggers*) diagnosed the problem and surveyed the literature. This entry covers the implementation: what we built, what design decisions came up that the literature didn't answer, and where the observability hooks are for tuning.

## Per-Agent, Not Per-Tenant

The old `SignificanceAccumulator` tracked per-tenant. Ten events from any combination of agents in a tenant would fire consolidation. This meant a chatty agent could mask a quiet one — the chatty agent's events would trigger consolidation before the quiet agent accumulated anything interesting. Consolidation would run, reset the counters, and the quiet agent's two novel events would be lost.

The new `ConsolidationTrigger` tracks per-agent. Each agent has its own event count and divergence accumulation. When *any* agent crosses *either* threshold, consolidation fires for the tenant. This means a character who witnesses something significant triggers consolidation based on their own experience, not diluted by the collective.

This distinction matters in wacky-manor, where 17 characters share one tenant. A character in a one-on-one confrontation accumulates events at a different rate than one in a group scene. Per-agent tracking ensures the confrontation character gets timely consolidation.

## Two Independent Signals

**Event count** is the Park et al. baseline — pure interaction counting with no weighting. Fire at 8 un-processed experiences (configurable). This is the simple, reliable, always-on signal.

**Divergence accumulation** is the CLS urgency signal — weighted by how much each experience diverges from the consolidated self-model. This signal is behind an SPI (`DivergenceExtractor`), and the default implementation returns 0.0. The divergence channel is inactive until a real extractor is wired.

Why ship with divergence disabled? Because divergence computation requires comparing an appraisal result against the consolidated self-model, and we don't have that comparison infrastructure yet. The event count signal delivers the core value — monotonic accumulation replacing decay. Divergence is the enhancement that makes consolidation urgency-aware.

The two signals are independent — either one crossing its threshold fires consolidation. When both are active, divergence fires sooner for novel events (that's the point — a character witnessing a betrayal should consolidate faster than one having routine small-talk). Event count is the safety net that ensures consolidation fires even for low-divergence interactions.

## Observability for Tuning

The consolidation trigger is inherently empirical. The right threshold for event count isn't derivable from first principles — it depends on character density, scene pacing, and the interaction patterns of a specific scenario. The literature gives a range (Park et al. suggest 15-20 interactions; we're using 8 for our denser agent environment), but the actual number needs tuning against scenario runs.

Three logging levels capture the trigger lifecycle:

**Per-event accumulation** (FINE) — logs every event with the current agent state: event count vs threshold, divergence vs threshold, and a truncated description of what happened. This is the high-volume diagnostic for understanding accumulation patterns.

```
trigger:accumulate agent=PP events=5/8 divergence=0.00/5.0 desc="Dick Dastardly schemes against..."
```

**Trigger firing** (INFO) — logs the moment consolidation fires: which agent crossed which threshold, and the exact values. This is the key tuning signal — if event-count always fires and divergence never does, the divergence extractor isn't contributing. If consolidation fires too early or too late, the threshold needs adjustment.

```
trigger:FIRED tenant=manor agent=PP reason=event-count events=8/8 divergence=0.00/5.0
```

**Consolidation reset** (INFO) — logs the per-agent state when counters are cleared. Shows which agents accumulated how many events and how much divergence during the consolidation cycle. Useful for spotting agents that never contribute to triggering.

```
trigger:RESET agents=[PP(events=8,div=0.00) ML(events=3,div=0.00)] triggered=[manor]
```

## Hardening

Three defensive measures to keep the trigger robust in production:

1. **Extractor exception isolation.** The `DivergenceExtractor` is an SPI — third-party code. If it throws, the exception must not propagate through the CDI observer chain into the event producer. The trigger catches exceptions from the extractor, logs a warning, and continues with the event counted but divergence not accumulated. The event-count signal is never affected by extractor failures.

2. **NaN and Infinity rejection.** The `Double.isFinite()` guard rejects NaN and Infinity from extractors. NaN comparisons silently fail (NaN > 0.0 is false), which would mask bugs. Infinity would trigger immediate consolidation for every subsequent event. Both are rejected at the boundary.

3. **Per-tenant trigger deduplication.** `ConcurrentHashMap.putIfAbsent` ensures only one consolidation fires per cycle per tenant. Without this, concurrent events from different agents could both cross the threshold and fire two consolidations for the same tenant — the lock in `ConsolidationScheduler.consolidateNow()` would serialise them, but the second would waste a reset cycle.

## What Stays, What Changes

`DegreeAccumulator` — unchanged. Still the right mechanism for CAPS situation classification. Decay is correct for detecting shifts in behavioural patterns.

`SignificanceAccumulator` — trigger action disconnected. Still observes events and accumulates metrics (for potential dashboard use), but no longer calls `consolidateNow()`. This prevents double-firing now that `ConsolidationTrigger` handles scheduling.

`ConsolidationScheduler` — swapped its accumulator reference from `SignificanceAccumulator` to `ConsolidationTrigger`. Same lifecycle: `swapAndReset()` on each consolidation tick, fixed timer for the time ceiling.

## Next: Wire the Divergence Signal

The event count signal is complete and testable in scenario runs now. The divergence signal needs a real `DivergenceExtractor` — one that compares each appraisal result against the agent's consolidated personality model and returns a distance metric. The MindMap already stores the consolidated model; the comparison is a vector distance between the appraisal output and the stored self-model.

Once wired, the divergence signal makes consolidation urgency-aware: a character whose worldview is being challenged consolidates sooner than one whose experiences confirm what they already believe. That's the CLS 2016 insight — replay frequency proportional to divergence from existing structure.
