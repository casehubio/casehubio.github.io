---
layout: post
title: "Risk Gates and the Arena Pipeline"
date: 2026-08-11
entry_type: note
subtype: diary
projects: [casehub-fsitrading]
tags: [arena, risk, orchestration, blocks]
series: issue-18-strategy-arena
---

The [previous entry](2026-08-11-strategy-arena-architecture.md) covered why the arena is a blocks composition rather than a case engine lifecycle. This one covers what happened when we wired the full pipeline — routing through to execution — and where the design had to bend.

## Risk as two separate concerns

The risk layer splits into assessment and gating. `FsiRiskAssessor` is a pure function: consensus in, risk levels out. No side effects, no routing decisions, no CDI dependencies beyond the data it needs. Per-instrument classification against portfolio size — deadlocked instruments get HIGH, full liquidation gets CRITICAL, and everything else maps to percentage thresholds against the total portfolio.

The gating lives in `FsiRiskGateRouting`, a `RoutingStrategy<ArenaContext>` that reads the assessment and decides: pass-through or human. When the risk level is HIGH or CRITICAL, it dynamically creates an `AgentRef.human()` with a `WorkItemCreateRequest` — the approval template carries the consensus details so a trader sees exactly what they're approving. When risk is LOW or MEDIUM, a pass-through agent sets `ApprovalOutcome.NOT_REQUIRED` and the pipeline continues without pausing.

The split matters because assessment is testable without any routing infrastructure. Sixteen unit tests cover every threshold boundary without mocking a single CDI bean.

## The composition that wasn't

The spec describes the arena as Sequence[Supervisor → Parallel → Voting → ExternalAgent → Conditional → ExternalAgent]. Six blocks patterns composed into one `ExecutionModel`. It looks clean on paper.

In practice, the patterns don't compose as naturally as you'd expect. The Supervisor pattern routes and dispatches — but the dispatch results need to flow into the next step's context, and the `ArenaContext` is the shared state carrier. The Parallel pattern runs agents concurrently, but its candidate list is static at build time. The evaluation results from the Parallel need to be collected and mapped by strategy type before voting can consume them.

We ended up with each step as an `AgentRef.external()` in a Sequence. Each step is a function that reads from and writes to the `ArenaContext`. The routing step calls `FsiArenaRouting.route()` directly. The evaluation step dispatches selected agents via `CompletableFuture.allOf()`. The voting step calls `FsiMajorityVoteByInstrument.aggregate()`. The Sequence pattern provides the ordering, termination, and failure policy — the domain logic lives in the step functions.

This is less "pure pattern composition" than the spec envisioned. But it's honest about what the patterns actually give you at this stage: sequencing with failure handling, not automatic state propagation between steps.

## The execution agent

`FsiExecutionAgent` replaces `SimulatedOrderExecutor`. Same six operations — create order, fill, apply to position, write strategy evaluation ledger entry, write order execution ledger entry, write P&L attestation — but driven by the consensus rather than a single strategy's trade decision.

The interesting wrinkle: the consensus gives a quantity and side per instrument, but the execution needs a `TradeDecision` with a `strategyId` and an `Instrument` record. We trace back through the evaluations map to find the original trade decision from the strategy that proposed a trade on the winning side. The consensus quantity replaces the individual strategy's quantity, but the strategy attribution and instrument metadata come from the original decision.

## What got deleted

`StrategyEvaluator` (the SPI), `StrategyEvaluationCaseDefinition`, `StrategyEvaluationCaseDescriptor`, and `SimulatedOrderExecutor` — all retired. The case engine approach is fully replaced by blocks orchestration. The trigger endpoint at `POST /api/evaluations/trigger` is the entry point. It creates an `ArenaRunEntity` for concurrency control (partial unique index prevents concurrent in-flight runs for the same instrument), executes the arena pipeline, writes a memory record to `CaseMemoryStore`, and returns the full `ArenaResult`.

The composition gap — where the spec's pattern nesting hit reality — is worth feeding back to the blocks repo. A first-class state-propagation mechanism between Sequence steps would make the "pure composition" vision real. Right now the workaround is clean enough, but it's a workaround.
