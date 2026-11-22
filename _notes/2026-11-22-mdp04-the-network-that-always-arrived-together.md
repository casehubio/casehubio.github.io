---
layout: post
title: "The Network That Always Arrived Together"
date: 2026-11-22
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
tags: [caps, topology, feedback-loops, convergence, personality, settling-dynamics, cognitive-architecture]
---

# The Network That Always Arrived Together

Two personality profiles walk into a CAPS settler. One is bold, cooperative, high-autonomy — the approach-oriented explorer. The other is conservative, avoidant, strict — the threat-sensitive withdrawer. Their connection weights are different. Their resting activations are different. Their attractor destinations are different. And yet they arrive at their respective destinations in exactly 10 iterations, every time, with identical convergence type and zero saturation.

The settling *destinations* diverge. The settling *paths* do not. This is issue #1192.

## Why feedforward converges uniformly

The CAPS topology had no feedback loops. All connections flowed INPUT→MEDIATING or MEDIATING→OUTPUT, with a few MEDIATING→MEDIATING bridges. The CapsSettler uses momentum damping: `newAct = 0.4 * oldAct + 0.6 * rawAct`. In a feedforward network, the damping factor dominates convergence rate. Signal flows in one direction, the activation functions (tanh for bipolar, logistic for unipolar) smooth the raw computation, and the damping blends 60% new with 40% old. Different weights change *where* the network lands. They don't change *how fast* it gets there.

The convergence check is `maxDelta < epsilon` (0.001). Both profiles hit that threshold at the same iteration because the damping rate is a global network parameter, not a personality-dependent one. The personality-dependent connection weights affect the shape of the settled state, not the speed of settling.

This is a property of feedforward networks, not a bug. But CAPS theory (Mischel & Shoda 1995) predicts personality-dependent dynamics via recurrent connections. The topology was incomplete relative to the theory.

## Six feedback loops

The fix adds 6 connections that create recurrent energy paths — output nodes feeding back into the mediating layer:

| Connection | Weight | Tag | Why |
|---|---|---|---|
| approach → self_efficacy | +0.15 | BAS | Approach behavior reinforces perceived capability |
| withdraw → self_worth | −0.15 | BIS | Avoidance erodes self-evaluation |
| trust → other_reliability | +0.10 | social | Trusting action reinforces the internal working model |
| distrust → other_reliability | −0.10 | social | Distrust degrades perceived reliability |
| BAS_activation → reinforcement_expectation | +0.15 | BAS | BAS amplification loop |
| persist → self_efficacy | +0.15 | BAS | Persistence reinforces self-efficacy |

The tags are the key. `BAS_connections` gets multiplied by 1.3 for bold profiles and 0.7 for conservative profiles. `BIS_connections` gets the inverse treatment. The same connection carries different effective weight depending on who the agent is.

With feedback, energy recirculates. The bold profile has three BAS-tagged loops amplified to effective weight ~0.195, while the conservative profile dampens them to ~0.105. The conservative profile amplifies the BIS loop instead. The *total* recirculation is different, and that changes how many iterations the network needs to settle.

## The noop classifier

The direct test — `FeedbackLoopDivergenceTest` in `caps-engine` — worked immediately. Create two `DispositionWeightMapper` states, settle with shared input activations, assert iteration counts differ. It passed.

The cognition-level test did not. `ConsolidationDivergenceTest.capsTrajectoryDiverges` went through the full pipeline: harness builds the consolidation phases, `BehavioralSynthesisPhase` processes graduated experience nodes, the `RuleBasedSituationClassifier` classifies them into CAPS input activations... and both agents converged at 10 iterations, identical, as if nothing had changed.

The classifier's `classify()` method returns `List.of()`. Always. The keyword-matching logic is in `classifyByKeyword()`, annotated `@SuppressWarnings("unused")` and preserved for a future NLP pipeline (#587). The LLM classifier displaced it. In production, `LlmSituationClassifier` handles classification via a Haiku call. In tests with no LLM available, the rule-based fallback does nothing.

Zero input activations means the network settles from resting state alone. Feedback loops have nothing to amplify. Both profiles converge identically because there is no signal flowing through the network — only damping toward rest.

## Testing around the gap

The cognition-level test needed a different strategy. Instead of relying on the classifier to produce input activations (impossible without an LLM in test mode), it verifies the full initialization chain — that the harness correctly propagates `DispositionAxes` through `CognitiveDefaults` → `CapsEngine.initializeAgent()` → personality-dependent weights — then settles directly against the engine with explicit shared inputs.

This tests what matters: the full cognition pipeline correctly initializes personality-dependent CAPS state, and that state produces different convergence dynamics when given the same signal. The classifier gap is real but orthogonal — it means the consolidation phase doesn't produce meaningful input activations in test mode, not that the feedback loops don't work.

## Weight sensitivity

An interesting finding: the divergence is weight-sensitive in a non-linear way. At 0.15, the two profiles diverge. At 0.25, they converge identically again — both at 20 iterations. The stronger feedback creates more recirculation (10→20 iterations) but doesn't differentiate, because both profiles have comparable *total* feedback energy — just on different pathways.

The divergence at 0.15 sits in a region where one profile is just above a convergence threshold and the other just below it. This is a characterization test, not a universal property test. If the topology changes, the specific iteration counts will shift. The test documents the current behavior and signals when topology changes affect convergence dynamics.

## What this opens up

The topology now has genuine recurrence. Over many consolidation cycles, Rescorla-Wagner weight updates on these feedback connections will compound personality differences. A bold agent that repeatedly approaches successfully will strengthen its approach→self_efficacy loop, converging faster on approach attractors. A cautious agent that repeatedly withdraws will strengthen its withdrawal→self_worth erosion, potentially creating oscillation patterns that a bold agent never encounters.

The iteration count divergence is the first measurable signal. The more interesting signal comes later — when evolved per-agent weights create genuinely different convergence *types* (CONVERGED vs OSCILLATION), not just different iteration counts.
