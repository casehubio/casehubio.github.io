---
layout: post
title: "When UNKNOWN Means Yes"
date: 2026-08-18
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [goap, planning, failure-handling, ternary-logic]
series: issue-927-adaptive-planning-intelligence
---

# When UNKNOWN Means Yes

The engine's GOAP planner operated on a boolean world — conditions are true or false, and absent keys default to false. Wiring it as a `DecompositionStrategy` exposed a problem I hadn't anticipated: decomposition happens *before* workers have run, so most conditions are genuinely unknown. A boolean planner can't distinguish "this condition hasn't been evaluated yet" from "this condition is known to be false."

The fix was a ternary world state — TRUE, FALSE, UNKNOWN — with what the literature calls "optimistic semantics": when a hard precondition references a key that's UNKNOWN, the planner assumes it will be satisfied and plans accordingly. If the assumption turns out wrong at runtime, the adaptation pipeline (#934) handles it. Soft preconditions use the opposite — pessimistic semantics. UNKNOWN means "penalise this path." The intuition: hard preconditions gate feasibility, so optimism enables planning. Soft preconditions express preference, so pessimism steers toward known-good paths.

What I didn't initially see was the ripple into existing callers. The dispatch-time `GoapPlanningStrategy` builds world state from the working layer — present keys are TRUE, everything else was FALSE. After the ternary change, everything else became UNKNOWN, and optimistic semantics made those conditions satisfiable. Actions that previously couldn't fire were suddenly applicable.

The resolution was two factory methods: `closedWorld()` for dispatch-time callers (absent keys stay UNKNOWN but the caller explicitly sets all precondition keys to FALSE before planning), and `openWorld()` for decomposition-time (where partial observability is the point). The decomposition strategy then runs a second pass — `buildOpenWorldState()` — that sets absent precondition keys from the action set to FALSE. Without this, the planner skips entire dependency chains because UNKNOWN satisfies preconditions everywhere.

The same session also landed the failure taxonomy. Worker failures are now classified as Transient (timeout, 503 — retry the same agent), Knowledge (declined, wrong output — the approach is wrong, try a different one), or Infeasible (all attempts exhausted). The classification drives exclusion: transient failures get a one-cycle exclusion (the agent sits out one dispatch round, then becomes eligible again), while knowledge and infeasible failures permanently exclude the agent. Previous behaviour was unconditional permanent exclusion for everything — a transient timeout burned through your agent pool the same way a fundamental capability mismatch did.

The two pieces are designed to compose. The failure category stored in `_diagnostics` is what #934 (meta-reasoning) will read to decide whether to persist with the current plan, refine it, or abandon the goal entirely. And `GoapDecompositionStrategy` — which builds causal dependency graphs from the precondition/effect structure, not arbitrary sequential chains — is what the portfolio strategy (#933) will try first before escalating to the LLM.

Two of three Phase A foundations done. #928 (plan monitoring and expectation tracking) is next — the third axis that measures how far the plan has diverged from what was expected, so the adaptation pipeline has something to react to besides discrete failures.
