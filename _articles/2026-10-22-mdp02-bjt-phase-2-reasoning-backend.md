---
layout: post
title: "Building a Production Bayesian Junction Tree on the JVM — Phase 2: From Inference Engine to Reasoning Backend"
date: 2026-10-22
entry_type: article
subtype: diary
projects: [casehubio/neocortex]
tags: [bayesian-inference, junction-tree, causal-inference, explanation-traces, parameter-learning, influence-diagrams, decision-theory, jvm, game-ai]
series: issue-642-bjt-phase-2
---

# Building a Production Bayesian Junction Tree on the JVM — Phase 2: From Inference Engine to Reasoning Backend

The [Phase 1 entry](2026-10-21-mdp01-bayesian-junction-tree-jvm.md) covered making the BJT engine correct and fast — log-space arithmetic, retraction safety, sensitivity analysis. That left a useful but low-level library: raw graph construction with `GraphNode` and `BayesVariable`, double arrays for everything, no vocabulary for questions like "what caused this?" or "what should I do?"

Phase 2 turns the engine into something an LLM or application can actually reason with. Ten issues, eight capabilities. The interesting parts aren't the features — they're the design tensions that emerged between them.

## The builder changes everything downstream

`BayesNetworkBuilder` is a step-builder with three inner classes that enforce valid construction at compile time. You can't call `.cpt()` without first calling `.given()`. You can't call `.build()` with an incomplete variable. The builder validates CPT dimensions, parent existence, and DAG acyclicity at build time — errors that previously manifested as silent wrong inference now fail before the junction tree is constructed.

```java
BayesNetwork network = BayesNetworkBuilder.create("diagnosis")
    .variable("Smoking", "false", "true").prior(0.3, 0.7)
    .variable("XRay", "false", "true")
        .given("Smoking").cpt(0.8, 0.2, 0.3, 0.7)
    .variable("Cancer", "false", "true")
        .given("Smoking", "XRay")
        .cpt(0.95, 0.05, 0.85, 0.15, 0.7, 0.3, 0.4, 0.6)
    .buildInstance();
```

The flat CPT varargs — `cpt(0.95, 0.05, 0.85, 0.15, ...)` — are a deliberate choice over `double[][]`. The builder knows the parent × child state counts and validates the flat array length against them. Row boundaries are implicit, but dimension errors are caught at build time, not at inference time.

Every subsequent Phase 2 capability depends on the builder. Parameter learning needs it to construct networks from learned CPTs. The causal engine needs it to build mutilated networks. Explanation traces need it for the end-to-end test path. Getting the builder right was the critical path.

## Causal inference: observation versus intervention

The `CausalEngine` answers a question the junction tree alone cannot: is this correlation causal or spurious?

Standard BJT inference computes P(C | observe B=true) — what you'd believe about C having seen B. Causal inference computes P(C | do(B=true)) — what would happen to C if you forced B to be true, severing its connection to whatever normally causes it. The difference between observing and intervening is the difference between "patients who take this drug recover" (maybe they were healthier to begin with) and "this drug causes recovery."

The implementation builds a mutilated network — the original DAG with incoming edges to the intervention variable removed — then runs standard inference on it. The structural operation is simple but the implications are significant: you can now distinguish confounding from causation using the same engine.

One limitation surfaced during the explanation trace work: the mutilated network can become disconnected. In a fork structure (A→B, A→C), intervening on B severs A→B, leaving B isolated from {A, C}. The junction tree builder can't handle disconnected graphs. For now, confounding detection silently falls back to "not confounded" when the mutilated graph disconnects. The fix — either supporting disconnected graphs in the JT builder or using structural d-separation instead of full counterfactual analysis — is deferred.

## Explanation traces: what the LLM needs

"Cancer risk is 73%" is a number. "The positive X-ray is the primary driver, increasing cancer risk from 15% to 73%, but the relationship is confounded — the causal effect is smaller than the observed correlation" is a reasoning chain.

`ExplanationEngine.explain()` returns an `ExplanationTrace` — every evidence variable ranked by influence on the target, with direction (increased or decreased the most likely outcome), the junction tree message path (which cliques participated in transmitting the evidence), and confounding flags from the causal engine.

The influence computation reuses the existing sensitivity analysis: for each evidence variable, create a fresh instance without it and measure the KL divergence between the full posterior and the leave-one-out posterior. O(n) propagations where n is the number of evidence variables. The message path is structural — BFS through the junction tree from evidence clique to target clique. The path is unique because junction trees are trees.

The structured trace is the bridge between the engine and natural language. The engine provides the reasoning chain; the LLM converts it to prose. Neither needs to do the other's job.

## Typed proxies: the Drools lineage

`BayesModelProxy` brings the BJT back to its Drools roots — type-safe access through annotated interfaces. Define an interface with `@Variable` and `@Evidence` annotations on methods, and the proxy generates a dynamic implementation backed by a `BayesInstance`. Dirty tracking is built in: the proxy knows which variables have changed since the last propagation and skips `globalUpdate()` when nothing is dirty.

This is the OO integration layer. Where the builder makes networks easy to construct, the proxy makes them easy to use in application code without touching `double[]` arrays or string variable names.

## Parameter learning and online updates

`ParameterLearner` computes maximum likelihood CPTs from tabular case data. Feed it a dataset of observations and it fills in the conditional probability tables — no hand-tuning. This is table one: static learning from historical data.

`OnlineParameterUpdater` is table two: continuous refinement from resolved cases as they arrive. The CPTs update incrementally using configurable learning rates. In the game AI context, this means the model adapts as the opponent's strategy shifts — the CPTs update across games without full retraining.

Together, these two close the loop between observation and model. The builder constructs the structure, the learner fills the parameters from data, and the updater keeps them current.

## Influence diagrams: which action should I take?

The causal engine answers "what happens if I act?" Influence diagrams answer the next question: "which action should I take?" That's the full decision support question — Howard & Matheson, 1984.

An influence diagram extends the Bayesian network with two new node types. Decision nodes represent actions the agent controls — they have outcomes (the options) but no conditional probability table. Utility nodes are scalar payoff functions over parent configurations — "if the disease is present and I choose treatment A, the payoff is 8."

The design question that took the most thought: where do utility nodes live? They don't participate in probabilistic inference — no CPT, no outcomes to marginalize over, no likelihood to set. Putting them inside the `BayesNetwork` graph would mean teaching `JunctionTreeBuilder`, `BayesInstance`, and every marginalization method about a fundamentally different node type. We kept them outside the graph as separate `UtilityNode` records. The `InfluenceDiagramEngine` reads posteriors from a standard `BayesInstance` and evaluates utility functions externally.

Decision nodes are simpler — they go inside the network as ordinary `BayesVariable` instances with uniform priors. The solver clamps each decision to a specific value via hard evidence, propagates, reads the posterior, and computes expected utility. Iterate over all decision values, take the argmax. The existing engine does all the probabilistic work unchanged.

```java
InfluenceDiagram diagram = BayesNetworkBuilder.create()
    .variable("disease", "yes", "no").prior(0.3, 0.7)
    .variable("symptoms", "mild", "severe")
        .given("disease").cpt(0.2, 0.8, 0.9, 0.1)
    .decision("treatment", "treat", "wait")
    .utility("outcome").given("disease", "treatment")
        .utilities(8, 2, 3, 9)
    .buildInfluenceDiagram();

PolicyResult policy = InfluenceDiagramEngine.solve(diagram, "symptoms");
// {symptoms=severe} → "treat", {symptoms=mild} → "wait"
```

A subtlety that wasn't obvious from the design: utility parents that include both chance and decision variables can be disconnected in the BN, because the utility node itself isn't in the graph. The junction tree builder can't handle disconnected graphs. The fix connects them by making the chance parents into parents of the decision node with a uniform conditional CPT — informationally equivalent (uniform conditioned on anything is still uniform), but ensures the graph is connected and the parents share a clique for joint posterior computation.

Claude caught a bug during the post-implementation audit that none of the tests had exposed. The connectivity fix method had `return` where it needed `continue` — when the first utility node had only decision parents, subsequent utility nodes' connectivity was never processed. The tests passed because the utility with chance parents happened to be declared first. Reversed the declaration order, watched it crash, fixed it, added a regression test.

## Where this is going

One issue remains under the Phase 2 epic: LLM-elicited structure (#651) — letting a domain expert describe a problem in natural language and getting a working Bayesian network structure back. With influence diagrams landed, the engine now covers the full decision loop: build a network, learn parameters from data, infer posteriors, explain the reasoning, intervene causally, and compute optimal policies. LLM elicitation closes the last gap — the domain expert shouldn't need to know what a CPT is to get a working model.
