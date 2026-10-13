---
title: "The Strategy Arena — Why the Case Engine Was Wrong"
date: 2026-08-11
type: diary
status: draft
project: casehub-fsitrading
tags: [architecture, blocks, orchestration, design]
---

The Strategy Arena evaluates market signals through multiple strategy agents concurrently, aggregates their trade decisions via voting, and executes the consensus. The interesting design question was where to put the orchestration.

The existing code used the CaseHub engine — a case definition with bindings that fire on context changes. Evaluate strategy, assess risk, gate for human approval, execute order. Sequential, single-strategy, reactive. It worked for the first three chapters because there was only ever one strategy evaluating one signal.

The arena needs something fundamentally different. Multiple strategies evaluate the same signal in parallel. Their decisions get aggregated per instrument via majority vote. The consensus gets risk-assessed. High-risk or deadlocked results route to a human trader. Then execution. This is not a case lifecycle — it's a deterministic multi-agent pipeline.

## The composition that fell out

Blocks has orchestration patterns — Supervisor, Parallel, Voting, Conditional, Sequence. Each is a composable `ExecutionModel<T>` with pluggable routing, aggregation, failure policy, and accountability listeners. The arena is a Sequence of five composed patterns:

```
Sequence [
  Supervisor    → triage (routing selects which strategies respond)
  Parallel      → evaluate (selected strategies run concurrently)
  Voting        → aggregate (per-instrument majority vote)
  ExternalAgent → assess risk
  Conditional   → gate (HIGH or deadlocked → human approval)
  ExternalAgent → execute (orders, positions, P&L attestations)
]
```

Three accountability listeners attach to the top-level Sequence. Every step — triage, evaluation, voting, risk gate, execution — gets a ledger entry with `causedByEntryId` chains. Nothing is outside the accountability boundary. There is no procedural orchestrator service class. The arena is an `ExecutionModel` built at startup and invoked per REST trigger.

## What the review caught

The adversarial design review (four dimensions, 71 issues) caught things I would not have found by staring at the spec.

The routing bridge was wrong. I'd designed it to delegate to the engine's `ComposableAgentRoutingStrategy`, which turned out to be a single-select interface — it picks one winner. The arena needs multi-select. The fix was to use `RoutingSignalAssembler` (a public SPI) directly and blend all six routing signals into per-candidate scores above a configurable threshold.

Deadlock handling was wrong. I'd assumed voting ties would produce `AggregationResult.Deadlocked`, which the Conditional step would then catch. But `Deadlocked` is a pattern failure — the blocks framework treats it as an error by default and aborts. The fix was to keep deadlocks as data inside `ConsensusResult` (each instrument has a status: CONSENSUS, DEADLOCKED, or NO_VOTERS) and let the risk assessment step flag deadlocked instruments as HIGH risk, routing them to human approval through the normal gate.

Risk assessment needed its own step. I'd originally put it inside the Conditional predicate — the predicate would both compute the risk level and gate on it. Claude caught that this is a side-effecting predicate, which is architecturally wrong. An explicit `FsiRiskAssessor` ExternalAgent step now sits between Voting and Conditional.

## Platform type surprises

The eidos `DispositionAxis` enum has five personality axes (social orientation, rule following, risk appetite, autonomy, conflict mode) — not the FSI-specific axes from the original spec (time horizon, market preference, reaction speed). The spec's disposition table was aspirational. Only risk appetite maps directly.

And `AgentDisposition` (eidos) is not the same type as `DispositionProfile` (blocks routing). Same concept, different modules, different representations. The routing layer needs the blocks type, not the eidos one.

The implementation is halfway through — API types, quality dimension scoring, agent registration, and voting aggregation are done. Routing, risk gating, the arena composition itself, REST endpoints, and the old-code retirement remain. The hardest part — the design — is settled.
