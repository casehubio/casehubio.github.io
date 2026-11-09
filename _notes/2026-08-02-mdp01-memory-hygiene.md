---
layout: post
title: "Memory Hygiene: Supersession, Retention, and the Right to Be Forgotten"
date: 2026-08-02
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-clinical]
tags: [cbr, gdpr, supersession, retention, memory-management]
series: issue-120-cbr-multi-scope-dsmb
---

*Part of a series on [#120 — CBR Phase 7](https://github.com/casehubio/clinical/issues/120). Previous: [Cross-Scope Aggregation: When Sites Stop Being Independent](2026-07-31-mdp01-cross-scope-aggregation.md).*

## Memory Hygiene: Supersession, Retention, and the Right to Be Forgotten

With scope-tagged storage, scope-aware retrieval, and cross-scope aggregation wired in prior sessions, the remaining question is: what happens when CBR cases need to die?

Three mechanisms, each solving a different problem.

**Supersession** handles the case where new knowledge replaces old. When a protocol amendment resolves for a trial that already has prior amendments, `AmendmentSupersessionObserver` marks the most recent prior amendment's CBR case as superseded — not deleted, just deprioritised in retrieval. The design deliberately supersedes only the immediately preceding amendment, not all prior amendments. Older amendments may still contain useful precedent for different kinds of changes; only the one this amendment directly replaces loses its weight.

The IRB approval path was tempting as a second supersession trigger — CRITICAL deviations go through PI approval first, then IRB review. But `DeviationResolutionCbrWriter` already uses erase-before-store: the PI observer writes an incomplete case, the IRB observer overwrites it with the complete case. A separate supersession hook would race the writer (CDI async ordering is non-deterministic) and either find nothing to supersede or supersede something about to be erased anyway.

**Retention purge** handles volume growth. `CbrRetentionPurgeJob` runs weekly, calling `purge()` per domain with configurable `max-age-days` and `max-cases`. AE cases get 730 days and 10k cap; trajectory and trial-safety cases get 365 days. Domains without config are skipped entirely. Each domain runs in its own try/catch — a storage failure in one domain doesn't kill the purge for the others.

**GDPR scope erasure** is the interesting one. When a patient withdraws consent under Art.17, `ConsentWithdrawalService` already pseudonymises the patient's identity (`patientId = "erased-" + UUID`) and erases general memory entries. The CBR store needs a separate erasure call because CBR cases are keyed by entity ID (`aeId.toString()`), not by the general store's `"patient:" + enrollmentId` key. Scope-based erasure — `eraseByScope(Path.of(trialId, siteId, patientId))` — catches all CBR cases at the patient's scope path regardless of their entity IDs.

The ordering constraint is load-bearing: CBR erasure must execute before the pseudonymisation line. The scope path is constructed from the original `patientId`. After `patientId = "erased-" + UUID`, the original value is gone — there's no way to reconstruct the path that matches the stored cases. This is a one-way door: if the code runs in the wrong order, patient CBR cases survive the withdrawal.

Site-level and trial-level aggregate cases survive by design. Deviations are site-level protocol adherence records — no patient PII in the feature vectors. Trial-safety aggregates contain counts and rates, not patient identifiers. The scope hierarchy means `eraseByScope` at patient depth leaves everything above it intact.

## What This Closes

CBR Phase 7 is complete. The scope hierarchy is wired end to end: storage, retrieval, trust weighting, temporal decay, cross-scope aggregation, supersession, retention purge, and GDPR erasure. Three follow-up issues are filed for work that's genuinely out of scope here: case compaction (needs a neocortex API that doesn't exist yet), AE regrading (needs the regrade workflow first), and DSMB WorkItem creation (needs the notification infrastructure design).

The open question is compaction. Temporal decay and retention purge handle volume for now, but as trial durations grow and case counts accumulate, retrieval performance will degrade. Merging N similar cases into one weighted representative is the fix — but it requires defining what "similar enough to merge" means in a domain where a Grade 3 hepatotoxicity case and a Grade 3 nephrotoxicity case look similar by grade but carry entirely different clinical implications.
