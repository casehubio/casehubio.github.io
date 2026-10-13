---
layout: post
title: "The Judgment Layer"
date: 2026-08-21
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [planning, adaptation, meta-reasoning, cost-benefit, sealed-types]
series: issue-927-adaptive-planning-intelligence
---

# The Judgment Layer

Continues the [when coarse fails, decompose](2026-08-20-mdp07-when-coarse-fails-decompose.md) series.

Every other issue in this epic added a capability. GOAP decomposition gives the planner classical search. Failure taxonomy tells it what kind of failure occurred. Progress-gated triggers tell it when divergence warrants attention. Reflexion critique tells rerouted agents what went wrong. Learned costs feed execution history back into plan selection. Deeper decomposition turns failed leaves into finer sub-tasks. Each one answers a question the engine couldn't answer before.

This issue doesn't answer a question. It decides which answer to use.

MPDF (Yang & Thomason, AAAI 2026) formalises the trichotomy as three meta-cognitive actions: **Persist** — the plan is still valid, keep executing. **Refine** — the plan needs adjustment, but the goal is achievable. **Concede** — the goal isn't worth pursuing at this cost, abandon it. The engine had implicit versions of all three: Persist was "no trigger fired." Refine was "trigger fired, LLM replans." Concede was "all reroutes exhausted, fault the case." What was missing was a formal decision point that evaluates which of the three is appropriate *before* spending tokens on adaptation.

The pipeline placement was the first design question. The existing adaptation pipeline runs: `AdaptationTrigger.evaluate()` → `PlanRevisionStrategy.revise()` → apply. The trigger is binary — PROCEED or SKIP. The meta-reasoner returns a richer signal — Persist, Refine with scope, or Concede. Three options: replace triggers entirely, merge the ternary decision into the trigger return type, or layer on top.

We went with layering. Triggers remain fast pre-filters — divergence score lookups, per-binding replan hints, failure status checks. The meta-reasoner runs only when the trigger says PROCEED, adding cost-benefit evaluation as a second gate. The pipeline becomes: trigger → (SKIP: stop) → meta-reasoner → (Persist: stop, Refine: revise, Concede: abandon). Two SPIs with different responsibilities and different return types, each seeing only the signals it needs.

The scope field on Refine was the next decision. `RefineScope.LOCAL` means repair one step — eventually a GOAP-backed strategy that finds an alternative path for the specific failure. `RefineScope.COMPOUND` means re-decompose the entire compound — the existing `ForwardReplanRevision` LLM call. In v1 both resolve to the same strategy because no dedicated repair implementation exists yet. The scope is structural preparation — the meta-reasoner's decision vocabulary is complete, the execution differentiation arrives with the next issue.

The most interesting correction came from the decision review. My initial design selected scope based on divergence magnitude: low divergence → LOCAL, high → COMPOUND. Claude stress-tested this and found it architecturally wrong. High divergence from one catastrophic step might need local repair. Low divergence from gradual drift across many steps might need a full replan. The magnitude doesn't correlate with the scope of the response.

The correct signal is the failure category. The TART taxonomy maps directly: Transient failures (timeout, 503) → Persist. The retry and reroute mechanisms handle these — adaptation is wasted effort. Knowledge failures (agent lacked information, wrong approach) → Refine. First occurrence gets LOCAL scope; repeated knowledge failures on the same compound escalate to COMPOUND. Infeasible failures (goal contradiction, all approaches exhausted) → Concede. The meta-reasoner evaluates failure category, not divergence, because the *type* of failure determines the scope of the response. Divergence is the trigger's job — it already gated on it.

Concede exposed a pre-existing gap in the completion model. When the meta-reasoner abandons a compound, `faultCompound()` cancels PENDING PlanItems and faults the compound definition. `CompoundCompletionEvaluator` then propagates upward — but it always transitioned to COMPLETED. Under ALL semantics, if all children are terminal, the parent completes. A FAULTED child counts as terminal. The parent "completes" even though a child failed.

This was always wrong — it just didn't matter before Concede because compound-level faults only occurred through PlanItem-level failure paths that didn't trigger `evaluateCompletion()` on the parent. Adding a deliberate compound-fault path made the gap visible. The fix checks `hasAnyFaultedParticipant()` after the completion criteria are met: if any required child is FAULTED or CANCELLED under ALL semantics, the parent faults rather than completing. MOfN and FirstWins are unaffected — they complete based on threshold regardless of individual faults.

The default `CostCeilingMetaReasoner` is purely classical — no LLM dependency. It evaluates adaptation count against a configurable ceiling (default 5), checks failure category for the routing decision, and uses the adaptation generation counter already tracked by `CasePlanModel`. An LLM-backed meta-reasoner that estimates expected adaptation benefit — the full cost-benefit analysis that SOFAI-LM describes — can be wired in as a named strategy. The classical default prevents runaway adaptation and routes failures to the right response. That's enough to earn its place.

Phase C is done. Three issues that close the feedback loops: CBR traces inform plan costs, failed coarse tasks decompose finer, and a formal decision framework evaluates whether adaptation is worth pursuing. Batch 4 has three refinements remaining — plan repair vs optimisation separation, contingent planning branches, and annotations module GOAP support. The architecture now has every axis the research literature says matters. What's left is sharpening each one.
