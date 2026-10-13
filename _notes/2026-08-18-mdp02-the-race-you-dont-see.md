---
layout: post
title: "The Race You Don't See"
date: 2026-08-18
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [gate-rejection, binding-race, code-review, yaml-bindings, casehub-engine]
series: issue-72-gate-rejection-routing
---

The rejection routing implementation is complete — supervisor extension, integration tests, the full end-to-end path from gate rejection through senior analyst review and head-of-compliance escalation. But the interesting story isn't the implementation. It's what the code review found.

## Two bindings, one context change

The stall detection binding was designed as a safety net — if the rejection routing cycle reaches a state where no binding can fire and no goal is met, mark the case as stalled so it doesn't sit silently forever. The condition looked right: `actionGateRejected != null and rejectionReview != null and postRejectionTriage != null and rejectionEscalation == null`.

The problem is that `rejectionEscalation == null` is also true at the exact moment the escalation binding becomes eligible. When `postRejectionTriage` is written to context, the engine evaluates ALL binding conditions simultaneously. Both the escalation binding and the stall detection binding are eligible. Both workers are dispatched concurrently.

The stall worker is a stub that returns `{investigationStalled: true}` in sub-millisecond time. The escalation worker calls `WorkItemService.create()` — a database write, roughly 50ms. Stall always wins the race.

The fix is mechanical: add mutually exclusive guards so the stall binding explicitly excludes every triage decision that has a resolution path. But the root cause is worth internalising. CasePlanModel's ChoreographyStrategy does not provide YAML ordering guarantees. Every eligible binding fires concurrently. If two bindings produce conflicting outcomes — one a success path, the other a failure marker — the result depends on which worker finishes first.

## The test I assumed couldn't work

The implementation plan noted that `ClearanceRejectionRoutingTest` wasn't feasible because "no worker produces `PlannedAction(INVESTIGATION_CLEARANCE)`." I repeated this during the code review. It felt obvious — the existing triage worker just returns a decision, and clearance paths go straight to the `investigation-cleared` goal.

Except the triage worker already handles this. Lines 86-96 of `InvestigationTriageWorker.toWorkerResult()` produce exactly this PlannedAction for INCONCLUSIVE decisions. It was there all along. The assumption was never checked against the code.

`PEP_MATCH` produces a score of 0.5545 — comfortably in the INCONCLUSIVE band — without triggering any hard gates (the PEP hard gate requires both `entityType == "PEP"` AND `pepHit == true` from OSINT, and the stub OSINT returns `pepHit=false`). The test works. The gate fires. The rejection routing processes. The head of compliance decides CLEAR. The case reaches `investigation-cleared` through the escalation path.

## The authority paradox, twice

The previous session's design review caught the authority paradox for SAR filing: after the head of compliance says "file the SAR," routing it back through the MLRO gate creates a loop where a subordinate can overrule a superior. The fix was `sar-drafting-escalated` — a separate capability without a `PlannedAction`.

The code review caught the same pattern hiding in the clearance rejection path. `sar-drafting-after-clearance-rejection` was wired to the regular `sar-drafting` capability — which does produce a `PlannedAction(SAR_FILING)`. If the MLRO rejects this second gate, `actionGateRejected` is overwritten but `rejectionReview` is still set from the first cycle. The rejection-review binding's condition (`rejectionReview == null`) blocks re-entry. The case is stuck with no binding eligible and no goal reachable.

The fix is the same: use `sar-drafting-escalated` instead of `sar-drafting` for the clearance rejection path. The rejection cycle already exhausted the review path — re-gating achieves nothing.

## The extraction that silently dropped a feature

Deduplicating `buildEvaluator()` and `toWorkerResult()` from `InvestigationTriageWorker` and `PostRejectionTriageWorker` into a shared `TriageWorkerSupport` class seemed straightforward. The shared method handled the map-building correctly. What it didn't include was the `InvestigationTriageWorker`'s special case: when the decision is INCONCLUSIVE, it wraps the result with `PlannedAction.of(INVESTIGATION_CLEARANCE)` instead of plain `WorkerResult.of(map)`.

The full test suite caught it — four timeout failures across tests that depend on the INCONCLUSIVE gate being created. The shared utility handles the common case; the caller handles the decision-specific behaviour.
