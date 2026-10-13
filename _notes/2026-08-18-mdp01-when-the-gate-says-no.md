---
layout: post
title: "When the Gate Says No"
date: 2026-08-18
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [gate-rejection, accountability, regulatory-compliance, sar-filing, casehub-engine]
series: issue-72-gate-rejection-routing
---

The AML harness has had oversight gates since Layer 9 — when a worker declares a consequential action like filing a SAR, the engine creates a WorkItem that a human must approve before the action proceeds. What it didn't have was an answer to: what happens when the human says no?

Until now, gate rejection was a dead end. The MLRO rejects the SAR filing, the engine marks the worker as faulted, and the case sits there with no completion path. The investigation-triage spec even calls this "the correct regulatory posture" — which is true in the narrow sense that an inconclusive case shouldn't auto-resolve, but doesn't help when the system needs to actually route the case somewhere.

## The asymmetry

The first thing that became clear is that the two rejection types have opposite risk profiles. When the MLRO rejects a SAR filing, the system is potentially under-reporting — that's regulatory exposure, fines, personal liability for the compliance officer. When a compliance officer rejects an investigation clearance, they're saying "I need more evidence" — which is conservative and carries no regulatory penalty.

This shapes everything downstream. SAR rejection needs careful re-evaluation; clearance rejection is essentially a request for more work.

## The engine already knew

I expected rejection routing to require engine changes — a new SPI, a new event type, something. Instead, digging into the decompiled `ActionGateRejectedHandler` revealed that the engine already writes `actionGateRejected` to the case context and fires `CaseContextChangedEvent`. The metadata includes what was rejected, who rejected it, and their rationale.

This means YAML bindings conditioned on `.actionGateRejected` fire after rejection without any cross-repo work. The entire rejection routing flow lives in AML application-layer YAML — new capabilities, new bindings, new goals. No engine PR, no version coupling.

## The design

Seven decisions, each with alternatives considered. The core: a hybrid pattern where senior analyst review is the re-investigation step and head-of-compliance escalation is the backstop. The deterministic triage evaluator re-runs with extended input — senior analyst findings can shift the risk score enough to change the outcome. Hard gates (sanctions, confirmed PEP, shell company) are absolute and cannot be overridden by any analyst — those cases always reach escalation.

A design review caught the authority paradox: if the head of compliance says "file the SAR," sending it back through the same MLRO gate that triggered the escalation creates a loop where a subordinate can overrule a superior's decision. The escalated SAR drafting worker produces the narrative without a `PlannedAction` gate — the head of compliance's approval IS the authorisation.

The review also caught that INCONCLUSIVE re-triage results shouldn't auto-close the case. The original triage spec explicitly requires human sign-off for inconclusive evidence — auto-closing on INCONCLUSIVE after a rejection would silently drop that safeguard. Both SAR_FILING and INVESTIGATION_CLEARANCE rejections now route INCONCLUSIVE to escalation.

## What's built, what's left

Five of seven implementation tasks are done: domain records (`RejectionContext`, `SeniorAnalystReview`), extended `TriageInput` and `RiskScorer` with rejection-aware factors, three new workers (rejection review, post-rejection triage, rejection escalation), the `RejectionEscalationLifecycle` observer that writes the head-of-compliance decision back to case context, and the full YAML wiring — four new capabilities, six new bindings, a new `investigation-closed-no-sar` goal, and an updated completion block.

Two tasks remain: extending the supervisor's context projection with rejection fields, and the six integration tests that exercise both rejection paths end-to-end.

The new terminal outcome — `investigation-closed-no-sar` — makes the MLRO override visible at the case level. A case that was triage-scored as SAR_WARRANTED but overruled by human authority doesn't silently merge into the "cleared" bucket. The trust-routing system can score override accuracy as a separate signal from clearance accuracy.
