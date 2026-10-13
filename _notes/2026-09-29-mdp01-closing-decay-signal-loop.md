---
layout: post
title: "Closing the Decay-Signal Loop"
date: 2026-09-29
entry_type: note
subtype: diary
projects: [casehubio/neocortex, casehubio/blocks, casehubio/eidos]
tags: [goal-lifecycle, cross-repo, spi, cognitive-architecture]
series: issue-300-goal-revision-decay-lifecycle
---

# Closing the Decay-Signal Loop

The neocortex cognitive subsystem has had a half-built feedback loop for weeks. `GoalPrioritizationPhase` detects when a goal's confidence has dropped — sets a `decay-signal` property on the MindMap node. `CognitiveGoalOrchestrator` reads those signals and produces `GoalRevision` records. But nothing consumed them. The revisions accumulated every tick, the eidos goals stayed ACTIVE, and the `decay-signal` never cleared.

The interesting part wasn't the code — it was mapping the data flow across three repos to find exactly where the loop broke. Five gaps, three repos:

1. Nobody called `pendingRevisions()` on the orchestrator
2. No code transitioned an `AgentGoal.lifecycleState` in eidos
3. No `GoalLifecycleProvider` implementation read from eidos (only a no-op existed)
4. `AgentRegistry` had no partial-update method — only full descriptor `register()`
5. No idempotency guard existed for re-consumption between ticks

The fix is three pieces that each do one thing. A default method on `AgentRegistry.updateGoalLifecycleState()` does targeted read-modify-write without rebuilding the entire descriptor graph. `SocialAvatarCognition.consumeGoalRevisions()` maps "dormant" to `DORMANT` and "abandon" to `ABANDONED` after each tick. And `EidosGoalLifecycleProvider` bridges eidos goal states back to neocortex so `GoalResolutionPhase.sync()` can clear the signal.

The part I didn't expect: neocortex's sync mechanism was already complete. `GoalResolutionPhase.sync()` calls `GoalLifecycleProvider.getLifecycleStates()`, imports any returned status, and removes `decay-signal`. The only thing missing was a real provider to call. One `@ApplicationScoped` bean in blocks, and the loop closes.

The `updateGoalLifecycleState` default method is worth noting as a pattern. Adding it to the interface rather than doing the read-modify-write at the call site means `InMemoryAgentRegistry` can eventually override with a `computeIfPresent` and `JpaAgentRegistry` with a direct SQL UPDATE — but the default works correctly now via the existing `findById` + `register` path. The user wanted this scoped in rather than deferred, and it turned out to be the right call — the API is cleaner and the implementations can optimize independently.
