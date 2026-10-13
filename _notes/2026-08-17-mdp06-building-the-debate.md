---
layout: post
title: "Building the Debate: Where the Spec Met the JVM"
date: 2026-08-17
entry_type: note
subtype: diary
projects: [casehub-fsitrading]
tags: [blocks, deliberation, convergence, cdi, hibernate, implementation]
series: issue-20-trade-deliberation
---

*Continues from [Debate as Architecture](2026-08-16-deliberation-architecture.md).*

C3's design spec ran to 700 lines. The implementation ran to 14 commits. The gap between them was mostly plumbing — but the interesting bits were where the platform's type system disagreed with the spec's assumptions.

## The Type Mismatch Theme

`ChannelAgentRequest` doesn't have a `taskType()` method. The spec assumed one. `MessageDispatch.Builder` requires `actorType` — not mentioned in the spec's code snippets. `MessageType.EVENT` strips content on receipt, making sentinel metadata invisible to the projection. HQL's `CURRENT_TIMESTAMP` resolves to `java.sql.Timestamp`, which Hibernate 7's semantic checker rejects when the target field is `java.time.Instant`. Every one of these is a case where the design was correct at the abstraction level but wrong at the implementation level.

The Hibernate one is worth a moment. The HQL `UPDATE ... SET d.endedAt = CURRENT_TIMESTAMP` looks valid — `CURRENT_TIMESTAMP` is standard HQL. But Hibernate 7 evaluates the SET assignment at parse time and rejects the type mismatch. The fix is a query parameter: `.setParameter("now", Instant.now())`. Parameters go through the type conversion layer; the built-in function doesn't. The asymmetry — SELECT converts, SET rejects — isn't documented anywhere I could find.

## The CDI Proxy Catch

`executeWithTimeout()` catches a `CompletionException` from `CompletableFuture.orTimeout()` and calls `failDeliberation()` to clean up the database record. Except `failDeliberation()` is `@Transactional`, and calling it on `this` bypasses the CDI proxy — the annotation is dead. The record stays IN_PROGRESS forever, permanently blocking that instrument's concurrency guard.

Fix: `CDI.current().select(FsiDeliberationOrchestrator.class).get()` to get the proxied instance. This is a known CDI pattern but it's one of those things where knowing the theory doesn't prevent hitting it in practice. The test caught it — the record came back as IN_PROGRESS when it should have been FAILED.

## Five Paths, One Handler

The outcome handler worked first time. Five convergence paths, each with different threshold logic and execution gates. The `CommonGroundState` API — `establishedFacts()`, `pendingClaims()`, `disputedPoints()` — maps cleanly onto the ratio calculations. The trade parsing regex handles "BUY 200 AAPL at market" and "SELL 50 MSFT limit 425.00" and routes everything else to human escalation. DIMINISHING_RETURNS with established ratio below 0.5 escalates; above 0.5 executes with the signal's reduced confidence. CONVERGING uses a higher threshold (0.7) because you're betting on a debate that hasn't fully converged.

The interesting design choice: when the regex can't parse a trade from an established PROPOSE fact, the handler doesn't throw — it routes to human escalation. The LLM-generated debate content is inherently unpredictable. An unparseable consensus is still a consensus; it just needs a human to interpret the trade parameters.

## H2 and Partial Indexes

PostgreSQL's `CREATE UNIQUE INDEX ... WHERE status = 'IN_PROGRESS'` is the right production guard — one in-flight deliberation per instrument, enforced at the database level. H2 in PostgreSQL compatibility mode doesn't support `WHERE` on indexes. The guard is enforced in the application layer for dev/test; the SQL is commented and ready for production.

What's deferred: WebSocket push for deliberation events. The REST endpoints serve the immediate need — list, get by ID, manual trigger with 409 conflict detection. The WebSocket wiring is integration plumbing that doesn't affect the core deliberation logic. The prior entry's architecture — `FsiDeliberationStateObserver` bridging `MessageObserver` and `EventSource` — held up exactly as designed. The implementation just filled in the bodies.
