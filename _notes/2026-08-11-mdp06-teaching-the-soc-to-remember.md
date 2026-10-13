---
title: "Teaching the SOC to remember"
date: 2026-08-11
author: mdp
type: diary
tags: [cbr, case-based-reasoning, incident-response, trust, lifecycle]
issue: 23
epic: 21
status: published
---

# Teaching the SOC to remember

Every SOC incident starts from scratch. An analyst sees an alert, triages it, investigates, recommends containment, reviews the outcome. Tomorrow, an almost-identical alert arrives, and the process repeats with no memory of yesterday's resolution. The tribal knowledge lives in runbooks that rot, or in the heads of analysts who leave.

CBR — case-based reasoning — is the fix. Retain resolved incidents as structured cases. When a new alert arrives, retrieve the most similar past cases and inject them into the triage pipeline. The analyst doesn't start from zero; they start from "here's what happened the last five times we saw something like this."

## What we built

Issue #23 wires CBR into the SOC investigation pipeline. The work splits into two halves: **retain** (store resolved incidents for future retrieval) and **retrieve** (query similar past incidents at case start).

**Retain** is a second `CaseOutcomeObserver` — the same SPI we used for trust attestations in #22. When a case closes with a success outcome (resolved, escalated, false-positive), `SocCbrRetainService` builds a `SocIncidentCbrCase` from the case file snapshot and stores it via `CbrCaseMemoryStore.store()`. The case record captures everything: alert type, source system, ATT&CK technique IDs, IOC types, the analyst's outcome, the containment playbook, and investigation duration. Problem and solution fields are synthesised from the raw data for semantic search.

**Retrieve** is a standalone service — `SocCbrRetrieveService` — that builds a `CbrQuery` from the alert features available at case start (alert type, source system, severity, and the alert description for semantic matching). It calls `retrieveSimilar()` in hybrid mode: structured features catch recurring patterns (same alert type from the same source), while the semantic leg catches novel variants with similar descriptions.

The retrieve results land in case context as `retrievedIncidents`, where every downstream binding — IOC enrichment, ATT&CK mapping, containment recommendation, and the analyst review — can see them via input projections. The analyst sees "5 similar past incidents, 4 out of 5 resolved as confirmed threats" alongside the containment recommendation.

## The design review caught real bugs

Two rounds of adversarial review (decision review + spec review) surfaced findings that would have been production bugs.

The binding guard `.retrievedIncidents == null` creates a retry loop if the retrieve worker fails without setting the field. A store timeout would leave the guard true, the next context change would re-fire the binding, which would fail again — indefinitely. The fix: on any failure, always return an empty list. "CBR was consulted and found nothing" is different from "CBR was never consulted."

The original design used `boolean containmentSuccess` and `boolean falsePositive` to describe the case outcome. The review pointed out this collapses the nuance from Layer 4a's trust dimensions. A DOWNGRADE means the incident was real but severity was overestimated — containment was disproportionate, not failed. Collapsing that to `containmentSuccess=false` would mislead future triage. We replaced both booleans with `String containmentOutcome` — a categorical that preserves the raw analyst outcome for CBR similarity scoring.

## Incident lifecycle — declarative state tracking

The lifecycle state (`TRIAGING`, `INVESTIGATING`, `CONTAINING`, then terminal) is tracked entirely through YAML output projections. Each capability's `outputProjection` now includes an `incidentStatus` literal alongside its domain output. No new service code for state transitions — the state machine is implicit in the binding execution order.

`SocIncidentStatusObserver` watches `CaseLifecycleEvent` for `incidentStatus` changes and fires `SocIncidentStatusChangedEvent` on forward transitions. It suppresses reverse transitions via ordinal comparison on the `SocIncidentStatus` enum, and evicts completed cases from its tracking map to prevent unbounded memory growth.

## What this opens up

The CBR foundation means #24 (compliance evidence) can reference past incident resolution patterns when generating SOC2/DORA compliance reports. The `retrievedIncidents` context is available to every binding in the pipeline — a compliance evidence generator could cite "this incident was handled consistently with N prior incidents of the same type."

The `SocCbrSchemaRegistrar` registers a hybrid feature schema at boot — 4 categoricals, 1 semantic text field, 2 categorical lists. The blend parameters (`vectorWeight`, `minSimilarity`, feature weights) are hardcoded as platform defaults for now. Once real incident data flows through, tuning these — particularly weighting structured features higher than semantic similarity for SOC — will be the first optimisation pass.
