---
layout: post
title: "Four fixes, four patterns"
date: 2026-11-19
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
tags: [cognition, testing, concurrency, scheduling, maintenance, java-records, mockito]
---

A maintenance sweep through the cognition module today. Four issues, all independent, all small enough that the fix was clear once the root cause was. Each one illustrates a pattern worth knowing about.

## Mockito and Java records don't mix

`InnerLifeOrchestratorTest` had a test that passed for months despite containing a latent NPE. The setup mocked `DriveProfile` — a Java record — with `mock(DriveProfile.class)`. Mockito returns `null` for all method calls on a mock by default. For regular classes this is fine: you stub what you need. For records, every accessor returns `null` unless explicitly stubbed, and record constructors validate their fields with `Objects.requireNonNull`. The mock bypasses the constructor entirely.

The test survived because every code path through `InnerLifeOrchestrator.buildSystemPrompt()` hit an early return before touching `driveProfile.agentId()` — until the `additionalSections` parameter was introduced. One test passed a non-empty list, reached the `CognitionRenderContext` constructor, and hit the `requireNonNull` on the null `agentId`.

The fix is obvious: use a real `DriveProfile` instead of a mock. Records are value types. Mocking them is always wrong — they have no behaviour to stub, and the mock silently violates invariants the constructor would enforce. If your test setup calls `mock(SomeRecord.class)`, replace it with a real instance.

## Guard variables beat compound conditions

`ShadowArchetypePromptSection` had a prompt that ended abruptly under a specific combination of conditions: `store` was non-null, all facet displacements fell below `minDisplacement` (so `scored` was empty), and no archetype family was found in the MindMap. The fallback closing — "This is your current reality — do not comment on or explain it" — only fired when `store == null`.

The original code used `store == null` as a proxy for "no archetype resolved." But `store != null` doesn't mean an archetype was found. The store might exist and have no matching archetype node. The condition was testing the wrong thing.

The fix: a boolean `archetypeResolved` that tracks whether `resolveArchetypeFamily()` actually returned a value. The fallback now fires when `scored.isEmpty() && !archetypeResolved` — which covers both the null-store case and the store-present-but-no-archetype case. Guard variables are clearer than compound null checks because they name the semantic condition, not the mechanical one.

## ConcurrentHashMaps accumulate forever without eviction

`AppraisalTickParticipant` maintained eight `ConcurrentHashMap` instances keyed by `agentId:tenantId`. Each tick overwrites the entry for the active agent. When an agent departs, its entries stay. On a long-running server with agent churn, memory grows without bound.

The maps are a cache pattern without the eviction half. The fix adds a `lastTickTimes` map that records when each agent key was last written. `evictIdle(Duration maxIdle)` removes entries whose last tick is older than the cutoff — including all eight maps atomically per key. `CognitionCore` registers the method with the existing `CacheEvictionScheduler`, which calls it every sixty ticks or five minutes, whichever comes later.

This pattern — ConcurrentHashMap with no removal path — is common in cognitive pipeline classes that hold per-agent state. The eviction contract (`Consumer<Duration>`) is shared with `EffectiveCognitiveDefaults`, which already uses the same scheduler. Any new per-agent cache should follow the same shape: track last-access time, expose `evictIdle`, register with the scheduler.

## Scheduled systems need bounded starvation

`ConsolidationScheduler` had two starvation problems. The first was cold-start blindness: `knownTenants` was an in-memory set populated only by `consolidateNow()`, which is called by `SignificanceAccumulator` when activity crosses a significance threshold. After a restart, tenants whose activity stays below the threshold are invisible — no consolidation phases run for them. Ever.

The second was idle-gate starvation. `tick()` returned early unless `idleTracker.isIdle(1 minute)` was true, meaning a continuously busy deployment ran no scheduled consolidation at all. Every `ConsolidationPhase` — drive adaptation, relational schema evolution, behavioural synthesis, recurring thought scoring — depends on periodic execution. Indefinite delay is a correctness problem, not just a performance one.

Three changes:

**Tenant seeding.** `seedTenantsFromStore()` calls `CaseMemoryStore.discoverTenants(null, null)` at `@PostConstruct` to populate `knownTenants` from durable storage. Cold starts now see every tenant that has memories.

**Explicit registration.** `registerTenant(String tenantId)` adds a tenant without triggering a consolidation run. Tick-time participants that discover new tenants can register them immediately.

**Bounded starvation.** Each tenant tracks its last consolidation tick. When the system is idle, all tenants run as before. When the system is busy, tenants that haven't run in `maxStarvationIntervals` ticks (default: 6, configurable) get consolidated anyway. The idle gate defers non-urgent work; it no longer blocks it indefinitely.

The bounded-starvation pattern applies to any scheduled system with a "skip if busy" optimisation. The optimisation is valid — consolidation during active processing wastes resources. But "skip if busy" needs a ceiling, or a system that's always busy is a system that never maintains itself. Track the last-run point per unit of work, and force execution after a bounded number of skips.

## What these have in common

All four are instances of the same underlying problem: an implicit assumption that became false as the system grew. The mock assumed records behave like classes. The guard assumed store-present implies archetype-found. The maps assumed agents are permanent. The scheduler assumed idle periods are frequent. Each assumption was reasonable at the time it was written and invisible until the conditions changed.

The fix in every case was to make the assumption explicit — a real value instead of a mock, a named boolean instead of a null check, a timestamp instead of unbounded accumulation, a counter instead of an unconditional gate. Explicit assumptions are testable. Implicit ones are time bombs.
