---
layout: post
title: "When Coarse Fails, Decompose"
date: 2026-08-20
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [planning, decomposition, htn, failure-recovery, adapt]
series: issue-927-adaptive-planning-intelligence
---

# When Coarse Fails, Decompose

Continues the [closing the loop](2026-08-20-mdp06-closing-the-loop.md) series.

The second Phase C issue tackles a different kind of learning: not "which path is cheaper" but "when is the path itself too coarse." ADaPT (Prasad et al., NAACL 2024) showed that decomposing failed steps into finer sub-steps outperforms retrying at the same granularity. If "Analyse the transaction" fails because no single agent can handle the entire scope, breaking it into "Extract metadata," "Identify counterparties," "Evaluate risk indicators" gives each sub-step a narrower target and more precise capability matching.

The engine already supports arbitrary compound nesting — compounds can contain compounds — but never used it as a failure recovery mechanism. Failed leaf tasks got rerouted to a different agent or faulted. The new `DeeperDecompositionHandler` adds a third option: promote the failed leaf to a compound and decompose it finer.

The entry point turned out to be cleaner than expected. `WorkerOutcomeResolvedHandler` already receives both the `OutcomeDisposition` (EXHAUSTED = all reroutes used) and the `FailureCategory` (Knowledge = the task is too coarse, not just the wrong agent). The decomposition check slots in before `markFaulted()` — if decomposition succeeds, the old PlanItem is marked OBSOLETE instead of FAULTED, sub-steps are materialized, and `CONTEXT_CHANGED` dispatches them through normal binding evaluation.

The design review caught a critical ordering issue: the existing handler called `markFaulted()` unconditionally before any disposition branching. Since FAULTED is terminal, the promotion step couldn't mark the item OBSOLETE afterward. The fix restructured the handler flow — decomposition check first, fault only if decomposition fails or isn't applicable.

Three guards prevent runaway decomposition: a depth limit (default 3, configurable per case definition), a minimum-2-step guard (single-step decomposition = the same task that just failed), and the Knowledge-only gate (transient failures retry, infeasible failures fault). The depth is computed by walking the `getParentOf()` chain — zero storage cost, derived from the existing tree structure.

One subtlety the review surfaced: `getParentOf()` operates on binding names, not PlanItem UUIDs. Starting from `planItemId` always returns empty because PlanItem IDs are random UUIDs not present in the parent index. The spec's original depth computation would have silently returned 0, defeating the recursion limit.

Phase C's last issue — persist/refine/concede meta-reasoning — will give the engine a formal decision point that chooses between continuing, adapting, and abandoning. The deeper decomposition handler gives it one more option to select from.
