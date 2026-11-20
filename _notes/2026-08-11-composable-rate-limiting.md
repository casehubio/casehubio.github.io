---
title: "Composable rate limiting — one interface, three strategies, zero scattered code"
date: 2026-08-11
entry_type: note
subtype: diary
projects: [casehub-examples]
series: issue-30-rate-limiter-and-housekeeping
tags: [rate-limiting, platform, agent-gate, architecture]
status: draft
---

Three codebases had rate limiting. All three were doing the same thing differently.

Wacky-manor had a `GatedAgentProvider` — a semaphore wrapper around LLM calls, manually constructed in `ScenarioOrchestrator`. The platform already had its own `GatedAgentProvider` in `agent-gate`, more capable (token bucket + concurrency + CDI decorator), but wacky-manor didn't know about it. And trellis had a sliding-window counter baked into `ActionService` — 20 lines of timestamp management doing something the platform could do if the abstraction existed.

The discovery that the platform already had the rate limiter changed the scope entirely. The issue said "extract from wacky-manor to platform." The reality was: the platform has it, the consumers don't use it, and the platform version can't express what trellis needs (sliding windows).

## The design

One interface with four methods: `tryAcquire`, `release`, `rollback`, and `scope`.

The `release` vs `rollback` distinction came from the design review. When you chain strategies — say, token bucket then concurrency semaphore — and the semaphore times out, you need to return the token. A single `release()` can't distinguish "operation finished, token consumed" from "chain failed, give it back." The review caught this before I wrote the code. The existing platform code already handled it correctly with explicit `tokenBucket.release()` rollback calls — the interface just needed to make that contract explicit.

`scope` partitions strategies into SESSION (acquired at session open, released at close) and INVOCATION (acquired per call). Without this, a `GatedAgentSession.query()` call would re-acquire the concurrency permit it already holds — deadlock. The existing code handled this with separate boolean flags. The refactored version makes the distinction structural.

Three strategies implement the interface: `ConcurrencyStrategy` (semaphore), `TokenBucketStrategy` (wraps the existing `TokenBucket`), and `SlidingWindowStrategy` (extracted from trellis, rebuilt with `ReentrantLock` for thread safety). The `GatedAgentProvider` CDI decorator orchestrates them as a list — cheapest-to-reject first.

## Two consumption modes

The CDI decorator handles transparent gating for `AgentProvider` consumers. Configure via properties, inject normally, the decorator wraps everything. Wacky-manor deleted 244 lines and added 4 lines of config.

But trellis's sliding window isn't gating LLM calls. It's application-level autonomy pacing — "if the coordinator has auto-executed 3 actions in the last minute, show a countdown timer instead of auto-executing the next one." The CDI decorator can't express this because the decision point isn't inside `AgentProvider.invoke()`.

So the module gained a second API: `AdmissionGate.builder()`. Same strategies, same composition, explicit control. Trellis builds a gate with a `SlidingWindowStrategy` and calls `tryAcquire(Duration.ZERO)` as a non-blocking admission check. The hand-rolled timestamp management in `ActionService` became a one-liner.

## What this opens up

The Phase 2 issues — TPM-aware rate limiting, circuit breakers, multi-tenant awareness — all slot into the `AdmissionStrategy` interface. Each is a new strategy implementation, composed through the same builder or CDI config. The interface may need to evolve (context parameters for TPM token counts, outcome parameters for circuit breakers), but the composition machinery is in place.

The more interesting question is whether `AdmissionGate` becomes the general answer for application-level resource control across the platform — anywhere an app needs composable admission with backpressure. The rate limiting use case proved the pattern; the next test is whether it holds for a genuinely different resource type.
