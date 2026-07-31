---
layout: post
title: "Multi-Scope CBR: The Wiring Session"
date: 2026-07-30
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-clinical]
tags: [cbr, scope, dsmb, trust, gdpr]
series: issue-120-cbr-multi-scope-dsmb
---

# casehub-clinical — Multi-Scope CBR: The Wiring Session

**Date:** 2026-07-30
**Type:** phase-update

---

## What I was trying to achieve: scope-aware CBR for DSMB pattern detection

Clinical's CBR layer stores and retrieves precedents — AE escalation patterns, deviation resolutions, amendment decisions, enrollment trajectories. All of it stored at `Path.root()`. Every case visible from every scope. A patient-level AE decision and a trial-level DSMB signal sitting in the same flat bucket, indistinguishable by hierarchy.

Issue #120 asks for multi-scope memory: patient-level, site-level, and trial-level CBR cases, with scope-aware retrieval that prefers local precedents and decays toward broader context. Plus cross-scope aggregation that rolls up site-level AE patterns into trial-level DSMB safety signals.

## What we believed going in: neocortex would need new APIs

The issue listed four neocortex dependencies — hierarchical scoping SPI, privacy-preserving aggregation, active memory management, trust-weighted retention. I expected significant foundation work before clinical could wire anything.

The exploration told a different story. Neocortex already ships the full scope hierarchy: `Path.of("trial", "site", "patient")`, `ScopeDecay` with three strategies (Exponential, Linear, Step), a `ScopeDecayCbrCaseMemoryStore` decorator at Priority 85, `eraseByScope()` for GDPR, `TemporalDecay` with HalfLife/Linear/Step, supersession, retention purge, and `TrustWeightedCbrCaseMemoryStore` gated by a config property. Clinical was passing `Path.root()` to every call and ignoring all of it.

The gap narrowed to one piece: cross-scope aggregation. Neocortex has no API for rolling up child-scope cases into parent-scope signals. That's genuinely clinical-side logic — "3 of 5 sites show Grade 3+ AE rates above threshold" is a domain computation, not a retrieval pattern.

## The scope hierarchy that fell out naturally

Three-level positional paths: `Path.of(trialId)` at depth 1, `Path.of(trialId, siteId)` at depth 2, `Path.of(trialId, siteId, patientId)` at depth 3. No labeled segments — depth encodes level. `Path.isAncestorOf()` handles visibility: ancestor cases visible at descendant scopes, not vice versa. A patient-level query sees its own AE cases at full weight and trial-level aggregates at decayed weight. Other patients' cases at the same site stay invisible — they're stored at their own patient scope, not at an ancestor. Cross-patient patterns surface through the trial-level materialized aggregates instead.

The design review caught several things worth the cost. `ClinicalScopeResolver` methods now return `Optional<Path>` — entity traversal can fail if the navigation chain has gaps, and `Optional.empty()` is strictly better than falling back to `Path.root()` (which would make the case visible from all scopes). The review also killed the IRB supersession hook — `DeviationResolutionCbrWriter` already does erase-before-store, and a separate supersession hook would race the CDI `@ObservesAsync` ordering. Two rounds, 19 issues raised, 13 verified in the spec.

## Four of eight tasks landed

`ClinicalScope` enum in `api/` and `ClinicalScopeResolver` in `runtime/` — the scope model. All five CBR writers updated to resolve scope via the resolver and pass it to `storeIdempotent`. All three retrievers updated with `ScopeDecay` and `TemporalDecay` from a new `ClinicalCbrConfig` bean that parses `type:value` config strings (`exponential:0.7`, `halflife:90d`). `ClinicalAgentTrustProvider` bridges the ledger's `TrustScoreSource` to the neocortex `AgentTrustProvider` SPI — averages safety-accuracy, eligibility-precision, and protocol-adherence dimension scores.

What remains: `TrialSafetyAggregationJob` (the cross-scope aggregation — periodic batch computation from JPA entities, not CBR similarity queries), `AmendmentSupersessionObserver`, `CbrRetentionPurgeJob`, GDPR scope erasure in `ConsentWithdrawalService`, and follow-up issues for compaction, AE regrade capability, and DSMB WorkItems.

The interesting architectural question going forward is whether the materialized aggregation pattern — a scheduled job computing statistics from domain entities and storing them as CBR cases — becomes the standard pattern for any cross-scope reasoning. The alternative would be query-time aggregation, but that loses auditability (no stored case to point to in the ledger) and costs more per query. The materialized approach fits the existing `SiteEnrollmentTrajectoryJob` pattern and produces first-class CBR cases that the DSMB can query directly.
