---
entry_type: note
subtype: diary
title: "The hive that knows its health"
series: issue-1104-hive-mind
projects: [casehubio/engine]
author: mdp
date: 2026-09-21
tags: [hive-mind, evolution, health-sensors, compliance, architecture]
---

A self-improving system that can't measure its own health is just a random walk with extra steps.

We built the entire continuous evolution loop — circuit breakers, regression detectors, category trackers, conflict serialisation, a PRISMA-inspired research pipeline. It compiles, it passes 296 tests, and it does absolutely nothing useful. Because `CapabilityArea.assess()` was an empty SPI. No data in, no signal out. The circuit breaker that's supposed to trip when health drops below 0.6? It's evaluating `0.0 / 0.0` and reporting everything's fine.

This is the sensor problem, and it's the same problem that kills every monitoring system I've seen in the wild: teams build dashboards before they build data pipelines. The visualisation is beautiful. The graphs are empty.

## The readiness gap

The evolution loop has a harder version of this problem. It's not just "install some metrics." The loop needs to know:

- Is the project healthy enough to accept autonomous improvements?
- Which dimensions of health are actually being measured?
- Is there enough data to distinguish signal from noise?
- Are the safety mechanisms (rollback policy, budget limits) configured to catch regressions?

These aren't binary questions. A project that just deployed CaseHub has no EventLog history — forcing it to L3 autonomous improvement would be irresponsible. A mature project with years of case execution data and well-tuned signal sources is being held back if it can't self-improve because "evolution isn't enabled."

What we needed was a progression model. Not "on or off" but "how ready are you, and what's missing?"

## L0 through L3: a compliance ladder

Four levels, each a superset of the previous:

| Level | What it means | What's needed |
|-------|--------------|---------------|
| **L0 — Inert** | Default. Evolution infrastructure is present but asleep. | Nothing. Every project starts here. |
| **L1 — Observe** | Health data flows. You can see the scores. No action taken. | ≥1 capability area producing real assessments. |
| **L2 — Propose** | The system proposes improvements. Human approves each one. | L1 + evolution enabled + signal sources + consensus threshold. |
| **L3 — Autonomous** | Proposals execute without human approval. Safety nets active. | L2 + rollback policy + health threshold configured. |

The critical design decision: compliance is **per-area**, not global. A project can be L3 for stability (CI data flows, automated fixes work, rollback catches regressions) while sitting at L0 for cognitive-reasoning (no cognitive layer yet — that's Epic 2). The global project level is `min(non-L0 areas)` — areas that haven't started don't hold back the ones that have.

This avoids the cliff problem. Without per-area compliance, a single unconfigured area blocks the entire project from progressing. With 10 capability areas and the cognitive layer shipping later, that cliff would have been permanent until Epics 2-3 land.

## Ten health sensors

The [#1115 spec](https://github.com/casehubio/engine/issues/1115) defined a bootstrap taxonomy of 10 capability areas. Today we gave them all bodies:

**Data-driven areas** — these query EventLog for real events and compute health as a ratio:

- **Stability** — `CASE_COMPLETED / (COMPLETED + FAULTED + CANCELLED)`. The simplest sensor: are cases finishing successfully? A project where half the cases fault has a stability problem.
- **Execution** — worker success rate. `WORKER_EXECUTION_COMPLETED` vs failed, declined, and outcome-failed events.
- **Safety** — asymptotic decay from violation count. Zero violations = 1.0. Four violations = 0.2. Uses `1/(1+n)` rather than a threshold because safety is about trends, not absolute counts.
- **Performance** — case duration against an SLA threshold. Binary per-case (fast enough or not), then the ratio across cases.
- **Integration** — orchestration and workflow step success rates.
- **Coordination** — stigmergy convergence and swarm team formation vs storm events and provision failures.

**Heuristic areas** — these produce meaningful scores where data exists, neutral otherwise:

- **Perception** — signal activity rate, normalised. More observation and pheromone events = more active perception.
- **Autonomy** — improvement goals formed and provisions requested, normalised.
- **Cognitive reasoning** — `GOAL_REACHED / GOAL_FORMED`. A 50% ratio means half the goals the system forms are achievable — possibly too ambitious, possibly too many abandoned.
- **Cognitive memory** — returns neutral (ABSENT). CBR traces don't emit EventLog entries yet. This is the placeholder that the neocortex replaces.

Every area extends `AbstractCapabilityArea`, which provides `neutralAssessment()` (the L0 default) and `positionFromScore()` (the landscape position derivation). When an area has no data, it reports `LandscapePosition.ABSENT` — and `HealthScoreTracker` skips it entirely from the weighted average. This was a spec review catch: without the ABSENT exclusion, 7 neutral areas at 0.5 would drag three healthy areas scoring 0.9 down to 0.66 — barely above the circuit breaker's 0.6 threshold. The composite health score would lie.

## The tenancyId thread

A surface-level change with pipeline-wide implications. Every `EventLogRepository` query requires `tenancyId` — that's the tenancy model, non-negotiable. But `CapabilityArea.assess()` only took `caseId`. No external consumers implement this SPI yet (the whole point of this issue), so we added `tenancyId` now and threaded it through the complete pipeline:

```
EvolutionTicker.tick(caseId, tenancyId, config)
  → HealthScoreTracker.refresh(caseId, tenancyId, policy)
      → area.assess(caseId, tenancyId)
  → ImprovementCircuitBreaker.evaluate(caseId, tenancyId, tracker, policy)
      → tracker.computeScore(caseId, tenancyId, policy)
```

Six source files changed, nine test files updated. The kind of change that's trivial if you do it now and a week-long migration if you do it after consumers adopt the SPI.

## The validator

`ReadinessValidator` is the diagnostic tool. Give it a case, a target compliance level, and the current config — it returns a `ReadinessReport` with per-area compliance levels, per-check results, and a global pass/fail verdict.

```java
var report = validator.validate(caseId, tenancyId,
    ComplianceLevel.L2_PROPOSE, config);
// report.projectLevel() → L1_OBSERVE (not ready for L2)
// report.areas() → stability at L2, performance at L1, cognitive-memory at L0
// report.passed() → false
```

Each check carries a remediation hint — "Set ImprovementConfig.evolutionEnabled = true" or "Ensure EventLog has relevant events for this area." The command centre UI (#1132) will render these as a progression dashboard. What I care about now is that the data model is right: per-area breakdown, actionable gaps, and a conservative global level that doesn't lie about readiness.

## Why this matters

The autonomous improvement loop was designed to be safe — circuit breakers, regression detection, rollback policies, budget enforcement, conflict serialisation. All of that machinery existed. None of it could activate because the health signal was empty.

Now the signal flows. A project at L1 can see its stability score drop and know which cases faulted. At L2, the system proposes dependency updates when staleness signals reach consensus — and a human reviews each one before it lands. At L3, the improvement case spawns, the code changes, the PR merges, the outcome feeds back into the category tracker, and the next evolution tick sees the result.

The hive can finally feel its own health. Next: building the command centre that lets a developer conduct it.
