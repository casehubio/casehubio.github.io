---
layout: post
title: "When Three Recovery Mechanisms Don't Compose"
date: 2026-08-20
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [recovery, plan-versioning, error-classification, spi-design]
series: issue-696-multi-level-recovery
---

CaseHub had three ways to handle worker failure: Quartz retry for transient errors, plan adaptation for re-decomposition, and OutcomePolicy for agent rerouting. Each worked independently. None knew the other existed.

The problem isn't that they were bad — each solved its scope well. The problem is that retry exhaustion didn't trigger adaptation, adaptation failure didn't trigger replan, and nothing classified the error to decide which mechanism should fire first. A timeout and a hallucination got the same treatment. That's the gap Graph Harness identified with its three-level escalation model: retry, local patch, full replan.

I wanted a clean unification, not a coordination layer bolted on top. Pre-release means we can make the bold call — replace all three mechanisms with a single `RecoveryCoordinator` that owns the entire failure flow. Classification drives which level the coordinator starts at (transient errors don't waste time on plan revision; reasoning errors skip retry entirely), and escalation is sequential from there.

The classification model landed on a dual-input approach: workers can declare a `FailureClass` hint — `TRANSIENT`, `REASONING`, or `FUNDAMENTAL` — but an `ErrorClassifier` SPI makes the final call. The hint is advisory. A heuristic default catches the obvious patterns (timeouts, connection refused, HTTP 503) and treats everything else as a reasoning error. An escalation threshold prevents a buggy worker that always reports `TRANSIENT` from burning through all its retries without escalation.

The interesting design tension was Level 3 scope. Level 1 operates on a single task; Level 2 patches a single compound. The natural escalation for Level 3 is the entire case — re-evaluate all goals, all compounds, produce a coherent new plan. That's significantly more complex than compound-scoped replan, but it gives you a clean scope ladder: task, compound, case. We went with it.

Plan versioning was the companion piece. Every structural change — initial decomposition, compound adaptation, case replan — produces an immutable `PlanVersion` with a trigger record (what caused the change), a delta (what was materialized and obsoleted), and a snapshot. The version history is case-level with a monotonic counter. We deliberately chose audit-only versioning for now — no runtime rollback — but designed the version structure so SAGA compensation can layer on later by walking the history in reverse. The follow-up issue linking this to the existing SAGA pattern issue (#105) makes the connection explicit.

One thing caught us during implementation: `PlanVersion` references `CasePlanModelSnapshot`, which lives in `engine-common`. But the module boundary protocol (PP-20260727-5267d2) says plan-definition types go in `engine-api`. The protocol's test — "does a consumer need to import this type?" — is necessary but not sufficient. The real constraint is whether the type's imports can resolve from the target module's dependencies. `engine-api` can't depend on `engine-common`, so `PlanVersion` had to move to `engine-common` despite being conceptually a plan-definition type. The triggers and deltas that only reference `engine-api` types stayed in `engine-api`. A small thing, but the kind of discovery that saves someone else an hour.

Batch 1 — all the foundation types — is implemented. `RecoveryLevel`, `RecoveryPolicy`, `RecoveryOverride`, `ErrorClassifier` SPI, `PlanVersionStore`, `CompoundLockRegistry`, and the `DefaultErrorClassifier` with its heuristic classification. The coordinator itself, the call site migration that removes `OutcomePolicy`, and the integration tests are next. The pre-existing compilation errors in the runtime module (unrelated `ResourceId` type mismatch) meant verifying the classifier through IntelliJ's per-file diagnostics rather than Maven — a workaround, but it confirmed the code compiles cleanly.
