---
layout: post
title: "From dead interface to live service"
date: 2026-08-06
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [spi, eidos, quality-dashboard, cbr]
series: issue-114-sar-quality-wave3
---

*Part of a series on [#114 — LLM sar-drafting narrative adaptation](https://github.com/casehubio/aml/issues/114). Previous: [Where the sanitiser sits](2026-08-05-mdp01-where-the-sanitiser-sits.md).*

## The dead code that shaped the design

`SarDraftingService` was dead. The workers never called it — they had inline lambdas doing their own narrative assembly. Its signature didn't know about seed narratives. Its return type was a bare `String` where we needed metadata about how the narrative was produced.

We replaced it with `SarNarrativeService`: a clean SPI that takes `NarrativeContext` (investigation findings + seed narratives from CBR retrieval) and returns `NarrativeResult` (the narrative text, whether it was seeded, how many seeds were used, and which adaptation method produced it). The records live in `api/` — zero framework dependencies.

The interesting design question was the CDI pattern for the two implementations. `TemplateSarNarrativeService` is `@ApplicationScoped` — not `@DefaultBean`. `EidosSarNarrativeService` is `@Alternative @Priority(1)`. The reason matters: the eidos service injects the template service *by concrete type* for fallback delegation. If the template service were `@DefaultBean`, CDI would remove it entirely when the alternative activates — breaking the composition chain. `@ApplicationScoped` keeps both beans alive; `@Alternative` means eidos only activates with explicit `selected-alternatives` configuration.

## The quality dashboard

The second issue (#116) was straightforward once the data model was in place. `AmlCaseProfileLedgerEntry` already had `narrative_seeded` and `seed_count` from the plumbing work in #98. We added `adaptation_method` (V3010 migration) and built a JPQL query joining case profiles with `LedgerAttestation` records — the attestations carry the post-submission verdict (UPHELD vs FLAGGED), which is what we're actually measuring.

The query has one subtlety worth noting: it uses a correlated subquery to pick the *latest* attestation per subject. Without this, a SAR that was initially UPHELD then revised to FLAGGED would appear in both segments — inflating the total and skewing the rate.

The frontend is a Lit tab in Operations with KPI cards and two tables. A minimum sample size guard (n<5) replaces percentages with "Insufficient data" — early in deployment, a single outlier would make the rates meaningless.

## Eidos brought CDI trouble

Adding `casehub-eidos` as a runtime dependency brought `DefaultCapabilityHealth` — a `@Default @ApplicationScoped` bean that conflicts with the engine's `NoOpCapabilityHealth`. Both implement `CapabilityHealth`; neither has `@DefaultBean`. The fix was a CDI exclusion in both `application.properties` files.

A separate engine SNAPSHOT issue surfaced: `WorkerDecisionEntry` gained a `routing_rationale` column without a matching migration in AML's Flyway chain. V3011 patched the gap.

The `SarNarrativeSeedingIntegrationTest` now times out (#121), but the Layer 9 integration tests — which exercise the same sar-drafting worker with the same `SarNarrativeService` delegation — pass all nine tests. The timeout is in the `AmlEngineCoordinator.startInvestigation()` entry path, not the worker itself.

The eidos service is a skeleton today — the `callEidos()` method delegates to the template service and marks the result as `LLM`. MicroProfile Fault Tolerance annotations are in place: `@Timeout(10s)`, `@CircuitBreaker`, `@Fallback`. When real eidos integration lands, the circuit breaker ensures investigations never block on a slow or failing LLM. The fallback produces a deterministic narrative and tags it `LLM_FALLBACK_DETERMINISTIC` — visible in the quality dashboard so we can measure whether the LLM adaptation is actually worth the infrastructure.
