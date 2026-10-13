---
layout: post
title: "Teaching Machines to Remember Market Crashes"
date: 2026-09-01
entry_type: article
subtype: diary
projects: [casehubio/fsitrading]
tags: [cbr, neocortex, knowledge, trading, design]
series: issue-23-knowledge-compliance
---

# Teaching Machines to Remember Market Crashes

A flash crash hits at 2am. Your overnight bot detects it, classifies it as CRITICAL, and runs the HTN decomposition: halt trading, close positions, alert the on-call trader. Standard playbook, severity-driven, works every time.

But what if the system remembered that last time a flash crash hit this sector, closing positions immediately made things worse? What if the bot remembered that the last three liquidity drops on tech equities were followed by a bounce within 90 seconds, and the traders who held through it came out ahead?

That's the gap between a reactive system and a learning one. The reactive system executes its playbook. The learning system has a playbook *and* a memory of what actually happened when it ran.

## Case-Based Reasoning in 60 Seconds

Case-Based Reasoning is an AI problem-solving approach from the 1980s and 90s that never went away. The idea is simple: when faced with a new problem, look for similar problems you've solved before. The classic formulation is Aamodt and Plaza's 4R cycle: **Retrieve** similar cases, **Reuse** their solutions, **Revise** to fit the current situation, **Retain** the result for next time.

What makes CBR interesting for trading is that it's *experience-based*, not *model-based*. A machine learning model learns parameters from training data. A CBR system remembers specific episodes. When a new flash crash arrives, a model says "based on my learned parameters, the probability of further decline is 0.73." A CBR system says "here are five past flash crashes that looked like this one, and here's exactly what happened in each."

The difference matters operationally. The CBR system can explain its reasoning — "this is like the March 2026 AAPL incident where reducing positions immediately locked in losses." The model gives you a number.

## Features: Teaching the System What "Similar" Means

The hardest design question isn't "should we use CBR?" — it's "what makes two market incidents similar?"

A flash crash on AAPL at 2am during high volatility is not the same as a flash crash on MSFT at market open during quiet trading. But it's more similar to the AAPL crash than a margin call on a bond fund. Defining "similar" means choosing features and deciding how to compare them.

We landed on seven features, each with a different similarity measure:

**Categorical features** use exact matching. Event type (FLASH_CRASH vs LIQUIDITY_DROP) and instrument sector (EQUITY vs FIXED_INCOME) are either the same or they're not. Simple, but they do the heavy lifting on the first pass — they eliminate obviously irrelevant cases.

**Numeric features with Gaussian decay** handle the continuous dimensions. Time of day uses a 2-hour standard deviation — an incident at 2am is somewhat similar to one at 4am, but not to one at 2pm. Volatility at detection uses a wider window. The decay function means "closeness" degrades smoothly rather than hitting an arbitrary cutoff.

**The interesting ones are the structured features.** Price action pattern is a time series — the last 30 price ticks before the incident. Comparing two price patterns means comparing two sequences of different lengths that might be stretched or compressed in time. Dynamic Time Warping (DTW) handles this: it finds the optimal alignment between two sequences, allowing non-linear stretching. A crash that took 10 seconds to unfold matches against one that took 30 seconds, because the *shape* of the price movement is what matters, not its duration.

Event sequence uses edit distance — the ordered list of market events leading up to the incident. How many insertions, deletions, and substitutions does it take to transform one event sequence into another? We added domain-specific substitution costs: transforming a FLASH_CRASH into a LIQUIDITY_DROP costs 0.3 (they're related), but transforming it into a NEWS_EVENT costs 1.0 (completely different mechanism). Events within the same domain group (both detected events, both operational events) get a baseline similarity of 0.2, even if they're not explicitly listed.

The schema definition is a `CbrFeatureSchema` registered at startup:

```java
CbrFeatureSchema.of("plan",
    FeatureField.categorical("event_type"),
    FeatureField.categorical("instrument_sector"),
    FeatureField.numeric("time_of_day", 0, 24,
        new SimilaritySpec.GaussianDecay(2.0)),
    FeatureField.numeric("volatility_at_detection", 0, 100,
        new SimilaritySpec.GaussianDecay(10.0)),
    FeatureField.numericList("volume_profile", 0, 1e9),
    FeatureField.timeSeries("price_action_pattern", "timestamp",
        new SimilaritySpec.DtwSpec(
            new WarpingConstraint.SakoeChibaBand(5)),
        // ... inner fields
    ),
    FeatureField.discreteSequence("event_sequence",
        editDistanceSpec));
```

The `FeatureField` sealed hierarchy in neocortex-memory-api gives you nine types: Categorical, Numeric, Text, CategoricalList, NumericList, NestedObject, ObjectList, TimeSeries, and DiscreteSequence. Each type constrains which `SimilaritySpec` can be paired with it — TimeSeries only accepts DtwSpec, DiscreteSequence only accepts EditDistanceSpec. The type system prevents you from accidentally pairing a Gaussian decay with a time series.

The weights control how much each feature contributes to overall similarity: price action pattern gets 0.25 (the most discriminating), event sequence gets 0.20, and the categorical features get 0.10–0.15 each.

## Plan Adaptation: When the Past Doesn't Quite Fit

Retrieving similar cases is only half the story. The response plan from a past incident doesn't apply unchanged to the current one. The agent that executed step 3 last time might have low trust now. The volatility might be twice as high. The market might be closed.

`FsiPlanAdapter` implements four adaptation strategies, applied in priority order to each step in a retrieved plan:

1. **Suppression** — if the market is closed, suppress steps that require market access. No point recommending "close positions" when the exchange is dark.

2. **Substitution** — if the original agent's trust score (via `AgentTrustProvider`) has dropped below 0.4, mark the step for substitution. The trust scores come from the ledger's Bayesian Beta model, fed by P&L attestations from prior trades.

3. **Boosting** — if current volatility is more than 2x the past case's volatility, boost the step's priority. Higher volatility means faster response matters more.

4. **Step addition** — for high-severity events (FLASH_CRASH, MARGIN_CALL, CIRCUIT_BREAKER), insert an advisory pre-reduce step at the beginning. The system recommends reducing exposure before running the rest of the plan.

Each adapted step carries an `AdaptationAction` enum — RETAINED, SUBSTITUTED, BOOSTED, SUPPRESSED, or ADDED — so the consuming agent can see exactly what changed and why.

## The Integration Model: Inform, Don't Override

One design decision shaped everything else: CBR *informs* the HTN decomposition without overriding it.

The static severity-based HTN decomposition (CRITICAL → halt, close, alert; HIGH → reduce, hedge, alert; MEDIUM → adjust, monitor) remains the primary response structure. CBR retrieval happens at case start, and the adapted plans go into `CaseContext` where agents and the LLM fallback can reference them. "Here's what worked before" — but the agent makes its own decision.

The alternative was full override: let CBR replace the HTN decomposition entirely, with static methods as fallback for zero-retrieval cases. We rejected this because the HTN decomposition is proven. It handles severity classification deterministically. CBR adds memory on top of proven structure — it doesn't replace structure with memory.

This also made the implementation simpler. The CBR pipeline wraps around the existing `OvernightIncidentCaseHub` without modifying its YAML case definition. At case start, the engine's `CaseStartedEventHandler` drives retrieval automatically. At case close, `FsiCaseOutcomeObserver` records the incident as a `PlanCbrCase`. The existing lifecycle is untouched.

## Gotchas That Shaped the Design

The knowledge garden surfaced eight relevant entries before I started designing. Three of them directly shaped decisions.

The most impactful: the platform's `CbrCaseRetainObserver` fires automatically whenever a case definition has `CbrConfig`, creating a generic CBR entry alongside any domain-specific one. We needed `CbrConfig` for retrieval but not for the generic retain — so we exclude `CbrCaseRetainObserver` from CDI and handle retain ourselves in `FsiCaseOutcomeObserver` with proper trading-domain features and plan traces.

Another: the `store()` method's sixth parameter is `caseType`, not `scope`. The decompiled interface has no parameter names — just `var1` through `var7`. Pass the wrong string and storage succeeds silently, but retrieval returns empty. The kind of bug you'd spend hours on without the garden entry telling you exactly where to look.

## What Memory Enables

The CBR pipeline stores one `PlanCbrCase` per incident: the 7-feature vector (what happened), the plan trace (what was done), and the outcome (how it went). Over time, the store accumulates a corpus of market incidents with their responses and results.

Temporal decay (90-day half-life) ensures recent incidents dominate retrieval — market conditions change, and a response plan from two years ago is less relevant than one from last month. Trust weighting modulates similarity by the trust score of the agents that produced each case — plans from high-trust agents rank higher.

What the system can't do yet: ensemble synthesis. The engine's `CbrRetrievalService` adapts each retrieved plan individually but doesn't synthesize them into a consensus. If three of five past plans agree on "reduce exposure first" but two disagree, the system currently shows all five individually. The `PlanEnsembleAnalyzer` SPI exists — UNANIMOUS, CONSENSUS, CONTESTED, MINORITY, UNIQUE classifications per step — but the engine doesn't invoke it yet. That's a platform issue to file, not an application gap.

The real test comes with case volume. DTW and edit distance similarity scoring are meaningful only with sufficient past cases. The system starts cold — seeded with simulated incidents — and gains value with every real incident that flows through it. The first retrieval that surfaces a relevant past response and changes an agent's decision is when CBR earns its keep.
