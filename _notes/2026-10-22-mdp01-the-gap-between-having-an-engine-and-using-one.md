---
layout: post
title: "The Gap Between Having an Engine and Using One"
date: 2026-10-22
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
tags: [bayesian-inference, causal-reasoning, bjt, api-design, probabilistic-reasoning]
series: issue-642-bjt-phase-2
---

# The Gap Between Having an Engine and Using One

The Bayesian Junction Tree engine has been sitting in neocortex for months. It could propagate evidence, compute marginals, do log-space arithmetic without numerical underflow. What it couldn't do was be used by anyone who wasn't willing to think in terms of `double[]` arrays and junction tree cliques.

That's a familiar gap. I watched Rete go through the same transition — the algorithm worked, but adoption took off when we made it work with typed Java classes instead of raw working memory elements. That was the step that turned a powerful engine into something application developers would actually reach for. Property-reactive change notification, type-safe rule authoring, an API that matched how people already thought about their domain — the engine disappeared behind the developer's own objects.

The same pattern applies here.

The BJT needed the same treatment.

## What the reader needs to know

The junction tree algorithm is exact probabilistic inference over directed graphical models. You declare variables with conditional probability tables, the engine builds a tree of cliques, and message passing gives you the posterior probability of any variable given whatever evidence you've observed. It's the engine behind medical diagnosis systems, industrial fault detection, and game AI that reasons under uncertainty.

All of which is useless if the API looks like this:

```java
instance.setLikelyhood("Smoking", new double[]{0.0, 1.0});
instance.globalUpdate();
double pCancer = instance.marginalize("Cancer")
                         .getDistribution()[1];  // which index is "true"?
```

Stringly typed. Index-guessing. No way to know when a posterior changed. No way to ask causal questions. And if you wanted to update your model as new data arrived, you'd rebuild the entire network from scratch.

## What we built

Five capabilities, each closing a specific gap between "the engine works" and "applications can use it."

**Fluent construction.** `BayesNetworkBuilder` gives you a step builder with compile-time state validation — you can't declare a CPT before its parents exist, can't specify the wrong number of probability rows. The network declares itself in the domain's language:

```java
var instance = BayesNetworkBuilder.create()
    .variable("Smoking", "false", "true").prior(0.3, 0.7)
    .variable("Cancer", "false", "true")
        .given("Smoking").cpt(0.95, 0.05, 0.10, 0.90)
    .buildInstance();
```

**Typed proxies.** Instead of indexing into `double[]` arrays, you define a Java interface and the engine gives you a JDK Proxy that reads like your domain:

```java
interface DiagnosisModel {
    @Variable double cancer();
    @Evidence void setSmoking(String state);
    boolean isDirty(String variable);
}

DiagnosisModel model = BayesModelProxy.create(
    DiagnosisModel.class, instance);
model.setSmoking("true");
double p = model.cancer();  // auto-propagates, returns P(cancer=true)
```

The dirty tracking is property-reactive — one bit per variable, set when the marginal shifts beyond epsilon after propagation. Same concept I built into Drools, decoupled from Rete. Any consumer that cares about change notification gets it without polling.

**Causal inference.** Observational queries — P(cancer | smoking=true) — conflate correlation with causation. If a confounder like genetics drives both smoking and cancer independently, observing smoking selects for genotypes that increase cancer risk even if smoking itself has a small causal effect.

Pearl's do-calculus fixes this with graph surgery: `do(Smoking=true)` severs all incoming edges to the intervention variable, removing confounding paths. The `CausalEngine` implements this by rebuilding the network with the mutilated graph:

```java
BayesInstance<?> causal = CausalEngine.intervene(
    network, "Smoking", "true");
double pCausal = causal.marginalize("Cancer")
                        .getDistribution()[1];
```

`CounterfactualResult` compares both: what you'd conclude from observation versus what's actually causal. In the confounded smoking/cancer network, the observational P(cancer | smoking=true) is 0.75 but the interventional P(cancer | do(smoking=true)) is 0.55. That 0.20 gap is the confounding bias — the correlation you'd wrongly attribute to smoking's causal effect.

**Continuous parameter refinement.** `ParameterLearner` fits CPTs from historical data via maximum likelihood estimation. But models that only learn from batch data are frozen in time. `OnlineParameterUpdater` maintains sufficient statistics and updates CPTs as each new observation arrives — exponential decay option for non-stationary distributions where recent observations should dominate. The junction tree potentials update in place; no structural rebuild.

For a game AI learning from replays: the detector reliability CPTs adapt as the meta shifts. For clinical decision support: the model refines as each patient outcome resolves. For recommendation systems: priors update with every interaction without retraining the entire model.

**Hard-evidence retraction on the fast path.** The three-phase propagation system avoids full recalculation when only a few variables change evidence. But hard evidence — clamping a variable to a definite state — injected LOG_ZERO entries into the calibrated potentials, forcing every hard-evidence update onto the slower Phase 2 path. The fix handles LOG_ZERO arithmetic natively in the absorption step: `LOG_ZERO - LOG_ZERO` is a no-op ratio, not NaN. Soft-to-hard transitions now stay on Phase 3 retraction — the fast path.

## Why this matters beyond CaseHub

Every application that reasons under uncertainty faces the same API gap. The model might be a Bayesian network, a Hidden Markov Model, or a Kalman filter — the pattern is identical. You have a mathematical engine that works in its own terms, and you have application code that works in domain terms. The bridge determines whether the engine gets used or gets replaced by a lookup table and some if-statements.

LLMs are brilliant at generating plausible text from context. They are terrible at representing structured uncertainty — the probability that a detector is lying, the expected value of an action given what you've observed, the causal effect of an intervention separate from correlation. These are different tools for different jobs. An LLM can generate a natural-language explanation of why the model recommends a treatment change; the Bayesian network computes whether the recommendation is warranted. One produces language; the other produces calibrated probabilities.

The typed proxy gives the LLM something to read. The causal engine gives the LLM something true to say.
