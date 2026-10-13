---
layout: post
title: "Closing the Loop"
date: 2026-08-20
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [planning, goap, cbr, learning, cost-model]
series: issue-927-adaptive-planning-intelligence
---

# Closing the Loop

Continues the [adaptive planning intelligence](2026-08-19-mdp05-the-middle-ground.md) series.

Phase C begins — the point where measurement and classification stop being standalone features and start feeding back into the planner's own decisions.

The first intelligence issue is conceptually simple: GOAP actions have static costs, but the CBR store already knows which actions historically succeed and which don't. A worker that fails 40% of the time has a higher effective cost than one that runs clean — but the planner treats them identically unless someone manually tunes the numbers. We already had the data. We just weren't feeding it back.

The three-layer cost model composes cleanly. Layer one is the declared static cost — what the developer writes. Layer two is the dynamic `CostFunction`, evaluated against the current world state at planning time. Layer three wraps both with a learned reliability multiplier computed from CBR plan traces: `1.0 / successRate`, capped to prevent infinity. The wrapping works because `GoapAction` is a record — we create a new instance with a `CostFunction` that captures the original cost computation and multiplies by the learned factor. The existing `effectiveCost(state)` method then applies benefit on top. No changes to the planner itself.

The interesting design question was where the data comes from. Two options: a pre-computed aggregate cache updated on case completion, or on-demand similarity-weighted retrieval via the existing CBR pipeline. We went with similarity retrieval — the same mechanism that already feeds agent routing. Cases similar to the current one influence costs more than distant ones. A case that looks like your past failures inflates the costs of the actions that failed; a case that looks like your past successes doesn't. The learning is contextual, not global.

One subtlety worth noting: the A* heuristic in `GoapPlanner` uses the no-arg `effectiveCost()` — the static cost, not the learned one. After enrichment, the heuristic underestimates actual node costs. This preserves admissibility (the planner still finds optimal paths) but may explore more nodes than necessary. Acceptable for now. The alternative — threading learned costs into the heuristic — would require the heuristic to evaluate cost functions against a hypothetical world state for every frontier node. The computational cost doesn't justify the search efficiency gain.

The feedback loop has a starvation risk. An action that historically fails gets an inflated cost. The planner avoids it. No new data accumulates. The cost stays inflated even if the underlying problem was fixed. Three mitigations keep this manageable: the cost factor caps at 10× (expensive but still selectable when no alternative exists), CBR temporal decay ages out old experiences, and the minimum sample threshold prevents sparse early data from poisoning the model. A full explore/exploit mechanism is future work.

All three GOAP strategies — decomposition, dispatch, and adaptive — now enrich costs before planning. `PlanExecutionContext` already carried CBR experiences at dispatch time; we added the same to `GoalDecompositionContext` for decomposition. The enrichment logic lives in a shared utility (`GoapCostEnricher`) since both packages need it.

Phase C has two more issues: dynamic decomposition depth (failed steps decompose finer instead of retrying) and persist/refine/concede meta-reasoning (deciding whether adaptation is worth the cost before spending tokens on it). The learning loop gives them better data to work with.
