---
layout: post
title: "When Agents Disagree — Multi-Agent Bayesian Inference on the JVM"
date: 2026-10-25
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
tags: [bayesian-networks, inference, multi-agent, divergence, theory-of-mind]
---

# When Agents Disagree — Multi-Agent Bayesian Inference on the JVM

The previous entry covered composing Bayesian networks — linking a scouting model to a build-order model via posterior transfer. That's one agent reasoning over multiple connected domains. The next question is harder: what happens when multiple agents reason over the *same* domain with different information?

In a StarCraft II match, I know what I built. My opponent knows what they scouted. We're both reasoning about the same game state — enemy strategy, map control, tech paths — but from different evidence. I observed my own gateways; they might have observed my gateways if they scouted. Our posteriors diverge because our evidence diverges, not because our models differ.

## Two Patterns, Not One

The issue description flagged two levels of this problem. The first is simple and immediately practical: model what the opponent has observed as variables inside a regular Bayesian network. A visibility variable like `opponent_saw_gateway` with a CPT conditioned on the actual game state handles theory-of-mind within a single `BayesInstance`. No new infrastructure needed — just a well-chosen network topology.

The second is the general case: N agents, each with their own evidence, all sharing the same network structure. Each agent maintains independent posteriors. Cross-agent queries — "what does agent B believe about X?" — and divergence metrics — "how much do agents A and B disagree about X?" — are first-class operations.

The key design insight was that this isn't a new inference problem. It's a composition problem. The BJT engine already has `NetworkComposition` for linking multiple networks in a DAG, each with its own `BayesInstance`. Multiple agents sharing a network is just a composition where every slot happens to use the same `JunctionTree`.

## Composition as Foundation

Rather than building a separate `MultiAgentBayesInstance` with its own state management, we layered on what already existed:

```java
var multi = MultiAgentBuilder.create("sc2")
    .sharedNetwork(scoutingTree)
    .agent("self")
    .agent("opponent")
    .commonKnowledge("map_type")
    .build();

multi.setEvidence("self", "observed_units", new double[]{0.9, 0.05, 0.05});
multi.setEvidence("opponent", "observed_units", new double[]{0.1, 0.1, 0.8});
multi.update();

BeliefDivergence div = multi.divergence("self", "opponent", "enemy_strategy");
```

`MultiAgentInstance` wraps `CompositionInstance`. `MultiAgentComposition` wraps `NetworkComposition`. The builder generates the right composition from agent declarations. All inference mechanics — evidence propagation, dirty tracking, incremental updates — come from the existing infrastructure for free.

This also meant inheriting the DAG constraint from `NetworkComposition`. Unidirectional influence (A's beliefs inform B) works. Reciprocal influence (A ↔ B, where beliefs are mutually consistent at equilibrium) doesn't — `validateAcyclic()` rejects cycles. For the SC2 use case that's fine. For Bayesian games or Nash equilibrium computation, users iterate manually: update A→B, then B→A, repeat until convergence. The API doesn't prevent this; it just doesn't automate it.

## Freezing Shared Structure

`BayesInstance` separates structure (`JunctionTree` — shared, read-only during inference) from state (clique potentials, separator states, likelihoods — per-instance). Multiple instances safely share the same tree reference. But `addLeafVariable()` mutates the tree, and in multi-agent mode one agent's structural change would silently corrupt all others.

The fix was a `frozen` flag on `JunctionTree`. `MultiAgentInstance` clones each tree via `defensiveCopy()` and freezes the copy. The original stays unfrozen — you can still use it for non-multi-agent instances. The frozen check guards both the `JunctionTree` setters and the `BayesInstance` mutation methods.

## Measuring Disagreement

Divergence between agents' beliefs uses two complementary metrics: KL divergence (asymmetric — how much information is lost approximating A's beliefs with B's) and total variation distance (symmetric, bounded [0,1], interpretable as the maximum probability of disagreement). Both are O(n) over the posterior arrays.

KL is reported in both directions because both are meaningful. If I have 90% confidence in rush and my opponent has 50%, KL(me‖opponent) tells me how wrong *they'd* be using *my* model, and KL(opponent‖me) tells me the reverse. Total variation gives a single number: how different are these beliefs, period.

## What This Opens Up

The multi-agent layer sits cleanly between single-network inference and the cognitive engine that consumes it. For the cognitive system, it enables genuine theory-of-mind — an agent can reason about what another agent believes given what that agent has observed, without conflating the two perspectives into a single posterior. For game AI, it gives a foundation for opponent modelling where the opponent's information state is a first-class object, not a hand-waved assumption.

The DAG limitation is real but not urgent. When equilibrium computation matters — poker, negotiation, strategic interaction — the iterative pattern works. If that becomes a common enough use case to justify cycle-tolerant composition, the `MultiAgentInstance` interface would survive unchanged; only the update mechanics would need to change.
