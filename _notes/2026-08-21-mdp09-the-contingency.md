---
layout: post
title: "The Contingency"
date: 2026-08-21
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [planning, contingency, goap, dag-execution, cbr, proactive-adaptation]
series: issue-927-adaptive-planning-intelligence
---

# The Contingency

Continues the [judgment layer](2026-08-21-mdp08-the-judgment-layer.md) series.

Every adaptation mechanism built so far is reactive. A worker fails, the engine classifies the failure, decides whether to repair or replan, and invokes a strategy. That pipeline works — but it means the first failure on every path is always paid in full. The contingency question is different: if we know a path is likely to fail, can we pre-compute the alternative at decomposition time and avoid the replanning cost entirely?

HQCP (2025) puts the crossover at roughly 15% failure probability — above that, pre-computing a fallback is cheaper than reactive replanning. Below it, the storage cost isn't worth the latency saved. That number depends on domain-specific factors (how expensive is your replanning? how many alternative paths exist?), but it serves as a defensible default.

The mechanism lives on `DagNode` — a nullable `DagPlan<T>` field. When the primary task throws an exception, `DagDriver` creates a nested driver for the contingency sub-plan and executes it inline. If the contingency succeeds, the original node is marked Completed and its dependents proceed normally. If it fails, the node is marked Failed and the reactive adaptation pipeline takes over. Two lines of defense: pre-computed fast, then reactive deep.

The nested driver design was the interesting decision. Three options: flatten contingencies into the plan at construction time using `ANY_OF` edges (simple but loses conditional activation — all paths execute eagerly), inject contingency nodes into the running driver's state map at failure time (preserves activation semantics but mutates an immutable plan and breaks the `CountDownLatch` invariant), or create a self-contained nested `DagDriver` that executes the contingency opaquely.

We went with the nested driver. It preserves `DagPlan` immutability, keeps `DagDriver`'s core loop untouched, and the contingency is fully transparent to dependents — they see Completed or Failed, never the mechanics. The nested driver gets an empty listener list (isolates stateful listeners like the snapshot capturer from mixing contingency events with outer plan events) and shares the parent's `AtomicBoolean` cancellation signal. If the outer driver is cancelled while a contingency is mid-execution, the nested driver observes the same flag and stops dispatching.

The CBR integration is where it connects back to the learning loop built in earlier issues. `ExperienceAnalyser.actionFailureRates()` computes raw failure rates per capability from CBR plan traces. `GoapDecompositionStrategy` queries these at decomposition time and generates contingency sub-plans for actions exceeding the threshold — by replanning with the primary action blacklisted, the same mechanism as reactive repair but pre-computed. No LLM call, no runtime cost.

One gotcha from the implementation: GOAP cost enrichment interacts with contingency generation in a way that's invisible until the tests fail. The CBR failure data inflates action costs via `actionCostFactors` — an 83% failure rate produces a 10x cost multiplier. If the primary and alternative actions have similar base costs (say 1.0 and 2.0), the enrichment flips the planner's selection to the alternative before contingency generation even runs. The primary action never appears in the plan. The fix is to design test fixtures with asymmetric costs — the primary must be cheap enough that even after 10x inflation, it still wins.

YAML support rounds it out. Bindings gain a `contingency:` field — a list of alternative capability names. `DefaultGoalDecomposer` attaches these post-decomposition by matching DagNodes to their bindings and wrapping them in sequential fallback plans. Strategy-generated contingencies from CBR take precedence over YAML declarations — the CBR version has historical evidence; the YAML version is a manual override for domains where the failure modes are known but not yet captured in execution traces.

The adaptive planning epic started with nine gaps between five existing techniques. This issue closes the last proactive mechanism — contingent planning pre-computes alternatives where all prior work reacted to failures after they occurred. One issue remains: the annotations module needs `@Cost` support to make GOAP actions declarable without YAML. After that, the adaptive planning intelligence stack is complete.
