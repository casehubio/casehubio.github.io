---
title: "When the Case Is Already Dead"
date: 2026-08-03
author: mdp
entry_type: note
subtype: diary
phase: "Slice 1, Layer 3"
projects: [casehub-soc]
tags: [casehub, engine, spi, failure-handling, architecture]
---

# When the Case Is Already Dead

A SOC investigation has three automated workers — IOC enrichment, ATT&CK mapping, containment recommendation — chained sequentially through contextChange bindings. If any worker fails after exhausting retries, the engine marks the case FAULTED. Terminal. No more bindings fire.

The analyst-review binding, which sits at the end of the chain waiting for `containmentRecommendation != null`, never triggers. The investigation is stuck. Nobody knows.

This is the failure mode that soc#19 addresses: when the automated pipeline breaks, a human still needs to see the wreckage.

## The Wrong SPI

The issue description recommended a "CDI observer on CaseStatusChanged." That's wrong — `CaseStatusChanged` isn't a CDI event. It's a Vert.x event bus message consumed by `CaseStatusChangedHandler` via `@ConsumeEvent`. The naming is misleading: it reads like an observable fact, but it's an internal coordination command. The handler mutates state, persists, closes channels, cancels triggers. Not something an application should consume directly.

The actual CDI event is `CaseLifecycleEvent`, fired via `Event.fireAsync()` at the end of the handler's processing chain. `CaseLedgerEventCapture` uses it for audit entries. It's the right abstraction level for observation — but not for what we need here.

## What We Actually Need

Creating a WorkItem isn't observation. It's a reaction to an outcome. The engine already has an SPI for that: `CaseOutcomeObserver` in `casehub-engine-api`. It fires on every terminal state — COMPLETED, FAULTED, CANCELLED. `CbrCaseRetainObserver` uses it for CBR case memory retention. The pattern is proven.

The distinction matters architecturally. Bindings are case-internal — they operate within a running case's lifecycle, evaluated by `CaseContextChangedEventHandler` against the case's plan items. A FAULTED case has shut down its internal mechanisms. Observers are case-external — they react to a case closing, operating outside the case's lifecycle entirely. Even if the engine added a `statusChange` trigger type to bindings, the observer would still be the right mechanism. Creating a failure-review WorkItem is an external reaction, not an internal case step.

## The Race That Blocks Everything

During the design review, Claude traced the event chain through `WorkerRetriesExhaustedEventHandler` and found something unexpected. The handler calls `caseInstance.setState(CaseStatus.FAULTED)` — a plain field assignment, no lock — then publishes `CaseStatusChanged` carrying the same `CaseInstance` by reference. When `CaseStatusChangedHandler` receives it, `trySetTerminalState(FAULTED)` acquires a lock, checks if the state is already terminal, and returns false. The handler logs "Ignoring duplicate terminal transition" and returns early.

`fireOutcomeObservers()` is never reached.

This means `CaseOutcomeObserver` — the right SPI — silently never fires for FAULTED cases caused by worker retry exhaustion. No CBR retain. No lifecycle event. No channel cleanup. The database state is correct (persisted by the first handler), so tests checking `getState() == FAULTED` pass. The silence is total.

Filed as engine#846. The fix is straightforward: `WorkerRetriesExhaustedEventHandler` should not call `setState()` — only publish `CaseStatusChanged` and let the handler own the terminal state transition. It landed within a day.

## The Implementation

`SocFaultedCaseReviewCreator` implements `CaseOutcomeObserver`. Filters on `outcomeLabel == FAULTED` and `caseType == incident-investigation`. Creates a WorkItem via the `WorkItemCreator` SPI with the full investigation context as payload — the analyst sees exactly how far the pipeline got and what failed. `callerRef` handles idempotency; `QuarkusTransaction.requiringNew()` handles the executor-thread transaction gap (garden GE-20260721-4564db — `@Transactional` interceptors silently fail when the engine calls observers via `Instance<T>` iteration).

Priority derives from the original alert severity. A CRITICAL alert that faulted during investigation still gets an URGENT WorkItem — the failure doesn't downgrade the threat.

## Why This Matters Beyond SOC

Any CaseHub application that needs to react when a case reaches a terminal state — not just FAULTED, but COMPLETED or CANCELLED — should use `CaseOutcomeObserver`. It's the public SPI, it's proven by CBR, and it correctly models the relationship: the reaction is external to the case, not part of its internal lifecycle.

The internal/external distinction is worth internalising. Bindings answer "what should this case do next?" Observers answer "what should the world do now that this case is done?" Confusing the two leads to either coupling (consuming internal events from outside) or architecture that breaks on terminal states (trying to use bindings when the case has stopped listening).
