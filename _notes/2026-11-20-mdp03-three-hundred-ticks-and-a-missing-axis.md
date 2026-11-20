---
layout: post
title: "Three Hundred Ticks and a Missing Axis"
date: 2026-11-20
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
tags: [cognition-core, re-tick, mood-stability, drive-coherence, cache-eviction, code-review]
---

# Three Hundred Ticks and a Missing Axis

The re-tick mechanism from #919 added a second mood and drive tick per cognitive cycle — when appraisal is enabled, `CognitionCore.tick()` ticks mood and drives once in FOUNDATION, then again post-DERIVED after appraisal emotions propagate. The unit tests passed. The eval pipeline scored 4+/5 on coherence. But nobody had run it at scale.

The concern was cumulative drift. Mood gets two decay passes per real tick. Drive deactivation runs through asymmetric EMA smoothing (alpha=0.3) twice, so drives decay toward zero roughly twice as fast on the down-ramp. At 10 ticks, the effect is invisible. At 300 ticks across varied emotional stimuli — interest, distress, joy, fear cycling every 10 ticks — it could compound.

It doesn't. The 300-tick scenario runs clean across four dimensions: mood bounded in [-1, 1], no monotonic drift, drive intensities stable and non-zero, and re-tick composite motivation within 50% of the single-tick baseline. The reason `clampAxis` works is that it bounds mood to `baseline ± maxDisplacement`, so even with double decay, the mood can't walk past the displacement envelope. And the drive EMA's asymmetric smoothing — instant on the upswing, blended on the down — means the double-tick effect only manifests as slightly faster deactivation, which is bounded by the next activation signal arriving.

The finding: the re-tick is safe at scale. Not "probably safe" — validated with two personality profiles (open and cautious from DivergenceFixtures), seeded MindMap data for curiosity activation, and varied appraisal inputs.

What code review caught was more interesting than what the test proved. I'd written the drift assertion checking pleasure and arousal but not dominance. The issue text explicitly names all three PAD axes. Dominance is arguably the most likely to drift under repeated re-ticking because the appraisal emotions include dominance projections — high agency in positive emotions, low agency in fear and distress. If any axis were going to show cumulative bias from the double-tick, dominance was the candidate. The assertion gap was exactly where it shouldn't have been.

The eviction work (#1183) surfaced a different kind of gap. Eight orchestrators — `DriveOrchestrator`, `MoodOrchestrator`, `GoalProposalOrchestrator`, `GutFeelingParticipant`, `InnerLifeOrchestrator`, `NarrativeOrchestrator`, `SearchBudgetTickParticipant`, `StrategyLearningOrchestrator` — all accumulate state in `ConcurrentHashMap<String, ...>` keyed by `agentId:tenantId`. They're written every tick and never cleaned up. In a long-running deployment with agents coming and going, these maps grow without bound.

The fix follows the pattern `AppraisalTickParticipant` established: a `lastTickTimes` map tracks when each key was last active, and `evictIdle(Duration)` sweeps entries older than the cutoff. All eight are registered with `CacheEvictionScheduler` in `CognitionCore.configureCacheEviction()`.

What the eviction work exposed: the orchestrators don't agree on key format. Most use `agentId:tenantId` with a colon separator. `GoalProposalOrchestrator` uses a pipe (`|`). `InnerLifeOrchestrator` reverses the order to `tenancyId:agentId`. Each class's `evictIdle` correctly uses that class's own key format, so there's no functional bug — but the inconsistency means you can't write a generic eviction helper that works across all orchestrators without per-class key translation. Worth cleaning up eventually, not worth tangling with the eviction work.

The third commit (#1184) was the smallest — adding a `Consumer<String>` tenant registration callback to `CognitionCore.tick()` so any tenant that sends a tick gets registered for consolidation. Without it, tenants were only discovered via `seedTenantsFromStore()` at startup or `consolidateNow()` from `SignificanceAccumulator`. A tenant whose activity never crossed the significance threshold would never get consolidated at all.

The gap between unit tests and behavioural validation is the thread through all three issues. The re-tick unit tests proved the mechanism works for one tick. The 300-tick scenario proved it doesn't drift. The eviction wasn't testable at the unit level because the problem is accumulation over deployment lifetime, not per-request correctness. The tenant registration is correct by construction — there's nothing to assert beyond "this callback gets called" — but it fills a gap that's only visible when you think about the system running for months, not minutes.
