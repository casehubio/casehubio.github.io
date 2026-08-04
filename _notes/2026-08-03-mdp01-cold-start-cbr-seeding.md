---
layout: post
title: "The Cold Start Problem Nobody Warns You About"
date: 2026-08-03
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [cbr, cold-start, activation-threshold, learning-mode]
---

CBR was feature-complete. Retrieve, retain, reuse, narrative seeding — the full pipeline, wired end to end. Cases retained on completion. Similar cases injected at startup. The path advisor analysing experiences and producing routing recommendations. Everything worked.

Except on day one, there are no cases. The advisor never fires because `cbrExperiences` is empty. The triage evaluator never gets a CBR adjustment because there's nothing to adjust from. Trust-weighted routing based on historical outcomes has no history. The entire CBR subsystem — five issues of careful architecture — sits dormant until enough real investigations complete organically. For a demo, that's fatal. For production, the question is how long the dormancy period lasts, and whether anyone notices the system is routing without its most informed input.

## The activation problem is per-cluster, not global

The obvious solution is a global learning mode flag: `casehub.aml.cbr.mode=LEARNING`. Flip it to `ACTIVE` when the case base is big enough. But "big enough" depends on what you're investigating. Fifty cases globally doesn't mean CBR knows anything about PEP transactions from high-risk jurisdictions — those might have two cases in the store, both inconclusive.

The insight: CBR retrieval already solves this. When a new case arrives, the engine retrieves similar past cases — that retrieved set IS the relevant cluster. If it returns 40 matches, CBR has dense coverage for this case's profile. If it returns 3, it doesn't. The advisor already knows the count — it computes `caseCount` as part of its confidence formula. Adding activation is one comparison: `active = caseCount >= threshold`.

We put the threshold behind `PreferenceProvider` (same pattern as the triage thresholds — `AmlCbrPolicyKeys.ACTIVATION_THRESHOLD`, default 30) so it's externally tunable without a redeploy. Below threshold, the advisor outputs `active: false`. Downstream consumers respect it: the `CbrAdjuster` skips threshold adjustment, the YAML binding's CBR trigger for senior-analyst routing requires `.cbrPathAdvice.active == true`. The advisory ledger entry records the active state either way — a full audit trail of what CBR would have done, even when it wasn't allowed to do it.

## Synthetic seeding bypasses the engine

For demos and development, waiting for organic case accumulation isn't viable. The synthetic seeder writes `PlanCbrCase` entries directly to `CbrCaseMemoryStore` — no engine pipeline, no Quartz jobs, no gate approvals. Fifty cases in milliseconds.

The plan traces aren't fake in any meaningful sense. The investigation YAML defines a finite set of binding paths: SAR cases always run entity-resolution, pattern-analysis, osint-screening, triage, sar-drafting, compliance-review. Cleared cases skip the last two. PEP cases add senior-analyst. Building `PlanTrace` lists from those known sequences produces data structurally identical to what `AmlCaseProfileStoreObserver` writes on real case completion. The only difference is no Quartz job executed — which doesn't affect CBR similarity scoring.

Coverage uses deterministic randomness (`Random(99)`) cycling through the full similarity matrix — all eight flag reasons, four entity types, three jurisdiction risk levels, weighted outcome distribution. Same seed, same output. `POST /api/simulation/seed/cbr` to populate, `DELETE` to clear and re-seed.

## What the bootstrap report shows

`GET /api/cbr/bootstrap-report` queries the compliance ledger — not the CBR store, which has no `count()` API. Profile ledger entries are a 1:1 proxy for retained cases, queryable via JPA. The report aggregates coverage across flag reasons, entity types, jurisdictions, and outcomes, alongside advisory metrics: how many advisories were active vs learning, average confidence, average case count. An operator can see "47 real cases retained, activation threshold is 30, CBR is active for dense clusters" without inspecting the store directly.

The CBR epic started with a similarity model and ends with a self-activating system that bootstraps from nothing, learns by observing before influencing, and provides operational visibility into its own readiness. The open question is whether 30 is the right threshold — it's a tunable default, not a researched number. Production data will answer that.
