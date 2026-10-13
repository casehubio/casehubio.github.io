---
layout: post
title: "The Authoring Surface"
date: 2026-08-21
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [planning, goap, annotations, cost-function, authoring]
series: issue-927-adaptive-planning-intelligence
---

# The Authoring Surface

Continues the [contingency](2026-08-21-mdp09-the-contingency.md) series. Final entry.

The previous nine entries built an adaptive planning system from the ground up — ternary GOAP, failure classification, progress monitoring, meta-reasoning, repair/optimization separation, contingent branches. All of it works, but all of it was built through the Java DSL and YAML paths. The annotations module — the third authoring surface — was still stuck on static `@Worker(cost = 0.5)` declarations.

`@Cost` is a method-level annotation that declares a dynamic cost function for a GOAP action. The method takes `GoapWorldState` and returns `double`, matching the `CostFunction` interface exactly. At build time, the deployment processor scans for `@Cost("workerCapability")`, matches it against the resolved capability name, and records the cost method name on `GoapActionDescriptor`. At runtime init, `CaseDefinitionRecorder` creates a `CostFunction` lambda via reflective invocation on the Gizmo-generated impl class. Static `@Worker.cost()` coexists — it feeds the A* heuristic while the dynamic function evaluates at planning time.

The interesting find was the action name identity bug. `inferGoapAction` was using `method.name()` for the GOAP action name, but `GoapDecompositionStrategy.decompose()` filters actions by capability name from `extractCapabilityNames()`. When `@Worker(capability = "assessRisk")` sits on a method also named `assessRisk`, they match and nobody notices. But `@Worker(value = "customName")` on method `doWork` silently drops the action from planning — the filter looks for `"customName"` in the action list and finds only `"doWork"`. No error, no warning. The fix is one line: pass `capabilityName` instead of `method.name()`.

Claude caught this during the decision review, not during implementation. I'd approved the design assuming action names matched capability names — because every example I'd written happened to use the same name for both. The reviewer read `AnnotationFeaturesTest` and found the test that proved the assumption wrong: `@Worker(value = "customName")` on method `doWork`, where line 103 asserts the GOAP action name is `"doWork"` and line 90 asserts the capability is `"customName"`. A bug hiding in plain sight, protected by examples that were too symmetrical.

The `@SoftDependency` ternary mapping turned out to need no code changes. The runtime already handles it correctly — `GoapWorldState.openWorld()` returns `UNKNOWN` for absent keys, and the soft-penalty scoring in `GoapPlanner` already penalises `UNKNOWN` soft preconditions. What looks like a gap in the annotations surface is actually the framework doing its job: the annotation is declarative, and the planner's ternary semantics handle the rest.

This closes the Adaptive Planning Intelligence epic. Twelve issues across six sessions: a classical GOAP planner with ternary world state and dynamic costs, failure taxonomy with diagnosis routing, plan monitoring with expectation tracking, progress-gated adaptation, reflexion-style critique, portfolio decomposition, learned action costs from CBR traces, dynamic decomposition depth, persist/refine/concede meta-reasoning, repair vs optimization separation, contingent planning branches, and the annotations authoring surface.

The design philosophy throughout was layered composition. Each issue added a capability that composes with the others without knowing about them. The GOAP planner doesn't know about CBR cost enrichment — `GoapCostEnricher` wraps its cost functions transparently. The meta-reasoner doesn't know about failure classification — it reads the category from `_diagnostics` and makes its decision. The portfolio strategy doesn't know which delegates it's cascading through — it just calls `decompose()` on whatever `StrategyResolver` returns. This is the same SPI-based composition pattern the rest of the engine uses, applied to planning.

What the epic doesn't include is equally deliberate. No LLM-backed meta-reasoning — `CostCeilingMetaReasoner` is purely classical, counting adaptation attempts against a ceiling. No cross-case coordination — each case plans and adapts independently. No persistent plan state beyond `EventLog` metadata and the snapshot infrastructure. These are all tractable extensions, but each would double the testing surface for capabilities that no consumer has asked for yet. The deferred integration test for EventLog metadata validation is the one item that should have been built — it needs a full case lifecycle harness that didn't justify its setup cost within this branch.
