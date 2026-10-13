---
layout: post
title: "Where the sanitiser sits — and where the verdict doesn't"
date: 2026-08-05
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [cbr, sar-narrative, design, cdi, quality-dashboard]
series: issue-114-sar-quality-wave3
---

The SAR narrative seeding plumbing (#98) was done. Seeds flow from CBR retrieval through the advisor worker into the sar-drafting worker's input projection. The workers acknowledge seeds exist — they set `narrativeSeeded` and `seedCount` flags — but they don't actually use them. Hard-coded strings come out regardless of what templates went in.

This session was about closing that gap: design the narrative adaptation service and the quality dashboard that tells you whether seeding helps.

## The SPI that should have existed

`SarDraftingService` was dead code. The workers never called it. Its signature didn't know about seeds, and it returned a bare `String` where the worker needed metadata. We replaced it with `SarNarrativeService` — a proper SPI that takes `NarrativeContext` (investigation findings + seed narratives) and returns `NarrativeResult` (narrative + seeded flag + seed count + adaptation method).

The interesting design question was the CDI pattern. The eidos-powered implementation needs to fall back to the deterministic adapter when the LLM fails. I initially reached for `@DefaultBean` — the standard platform displacement pattern. But `@DefaultBean` means "remove this bean from CDI when something better exists." If the eidos `@Alternative` activates and then needs to inject the template service by concrete type for fallback, there's nothing to inject — the `@DefaultBean` is gone.

The fix: make the template service `@ApplicationScoped` (not `@DefaultBean`). It stays in CDI regardless. The `@Alternative` wraps it and delegates on failure. This is composition, not displacement — a different CDI pattern for a different architectural need. Displacement is for progressive layer evolution (Layer 1 → Layer 3 → Layer 5), where the higher layer always wins. The eidos adapter is an optional runtime variant requiring external infrastructure — explicit opt-in activation, not automatic displacement.

## The outcome column that isn't what it says

The quality dashboard needs to segment UPHELD/WITHDRAWN/FLAGGED rates by whether seed narratives were used. I assumed `AmlCaseProfileLedgerEntry.outcome` contained the SAR verdict. It doesn't.

`outcome` stores the triage decision — `SAR_WARRANTED`, `FALSE_POSITIVE`, `INCONCLUSIVE`. These are set at case completion. The actual SAR regulatory verdict arrives later via `SarOutcomeFeedbackService`, which writes a `LedgerAttestation` with `verdict = SOUND` (UPHELD) or `verdict = FLAGGED` (both WITHDRAWN and FLAGGED map to the same value).

So the quality query is a join, not a single-table scan. `AmlCaseProfileLedgerEntry` has the seeding metadata. `LedgerAttestation` has the verdict. The join uses `subjectId` (both set to the case UUID), filtered to `capabilityTag = 'sar-drafting'` and `trustDimension = 'investigation-accuracy'`. A correlated subquery restricts to the latest attestation per case — handles SAR outcome revisions where a case's verdict changes post-submission.

Self-review caught this before it became an implementation bug. The field is literally called `outcome` on a case profile entry — the natural assumption is that it contains THE outcome. The two-entity split reflects the ledger architecture where `LedgerEntry` subclasses capture point-in-time snapshots and `LedgerAttestation` records capture post-hoc evaluations. Nothing in the naming signals this.

## Two layers of context budget

Context budget management splits cleanly between AML and eidos. AML controls `maxSeeds` (default 3) and `maxSeedLength` (default 2000 chars) via platform preferences — domain decisions about how many exemplars are useful. Eidos handles token-level truncation within whatever seeds AML passes through — model decisions about what fits in the context window. Neither layer needs to know about the other's concerns.

## The dashboard lives in Operations

The quality dashboard is a new "SAR Quality" tab in the existing Operations view. Operations is the analytics hub — Throughput, Trust Scores, Gate Activity, Intervention are all cross-case aggregate metrics. SAR quality segmentation is the same kind of question: "does this system-level feature improve outcomes?" KPI cards show seeded vs unseeded UPHELD rates and the lift between them. Tables show the segmentation detail and seed count correlation.

The design review surfaced 36 issues across coherence, structure, and robustness dimensions, with 32 verified and zero unresolved. The main improvements: eidos-api dependency moved from `api/` to `app/` (keeping the SPI module pure), PII exposure for the eidos path documented as a prerequisite on real `ContentSanitiser` (#115), and a minimum sample size guard (n<5 → "Insufficient data") for the dashboard KPIs.

The plan has 8 tasks ready for execution. The open question is how eidos's agent API actually looks when you try to build a prompt with seed narratives as exemplar documents — the spec designed around the API surface we could see from the outside, but the real integration will discover what the API actually wants.
