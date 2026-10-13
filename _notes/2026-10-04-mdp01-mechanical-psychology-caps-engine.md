---
layout: post
title: "Mechanical Psychology: Building a CAPS Engine for Emergent AI Behaviour"
date: 2026-10-04
entry_type: article
subtype: diary
projects: [casehubio/neocortex]
tags: [caps, psychology, behavioral-inference, architecture, cognitive-systems]
series: issue-406-emergent-behavioral-synthesis
---

# Mechanical Psychology: Building a CAPS Engine for Emergent AI Behaviour

Characters shouldn't be told how to behave. Behaviour should emerge from life experience filtered through personality disposition — crystallised during consolidation, rendered to the LLM as a finished profile. The system discovers "Peter is a perfectionist." No human writes it.

That's the thesis behind the work I've been designing on neocortex's emergent behavioural synthesis epic. Today I finished the architecture for the piece that makes it real: a CAPS graph engine that mechanically computes behavioural attractors from psychology research. Not prompted personality. Not fine-tuned affect. A deterministic spreading activation network whose connection weights come from published meta-analyses.

## Why Not Just Prompt It?

Every AI character system on the market right now — Inworld AI, Convai, Character.ai, Charisma.ai — does personality the same way. They write a system prompt: "You are brave but cautious. You distrust strangers." The LLM performs that personality until the context window fills up, then it forgets. If the character "changes" from an experience, it's because someone updated the prompt.

This works for short interactions. It fails for anything that needs genuine behavioural emergence: open-world games where NPCs accumulate hundreds of hours of player interaction, clinical simulations where a patient's coping mechanisms need to respond realistically to therapeutic intervention, AI companions where relationship development should be mechanistically coherent rather than scripted.

The problem isn't the LLM. The problem is that there's no underlying psychological model. The LLM is performing a personality description, not computing behaviour from a cognitive architecture.

## The CAPS Architecture

Walter Mischel and Yuichi Shoda proposed the Cognitive-Affective Processing System (CAPS) in 1995. Their central claim: personality is not a set of average traits but a set of if-then rules. The same person behaves differently in different situations — and that variation IS their personality. Mischel reached the right computational picture from the outside, by watching children at a summer camp, decades before the neuroscience caught up.

CAPS is formally a parallel constraint satisfaction network. Cognitive-affective units (beliefs, affects, goals, expectancies) are connected by weighted links. When a situation activates input nodes, activation spreads through the network and settles into a stable attractor state — the behavioural output. The same network topology, different connection weights per person, different situations → different behavioural signatures.

This is what we're building as the neocortex CAPS engine.

### Why Not Bayesian Networks?

I spent a full research session evaluating alternatives. The epic originally described the mechanical inference layer as "composable probability decision trees — like ONNX-style weighted graphs." The research showed this was wrong.

Decision trees can't handle cyclic interactions. In real psychology, beliefs affect behaviour and behavioural outcomes revise beliefs. CBT's core insight is exactly this feedback loop — activated schemas bias processing toward confirming evidence, which strengthens the schemas. A tree can't represent this. A DAG can't either.

Bayesian networks were the next candidate. I have a working junction tree inference engine — extracted from the Drools belief system I built years ago. Full exact inference, incremental evidence updates, the works. The question was whether it fit.

It doesn't. Bayesian networks require directed acyclic graphs. CAPS requires cycles. The six shared mediating nodes in the topology (self_worth, other_reliability, threat_sensitivity, FFFS_activation, reinforcement_expectation, arousal_level) all participate in feedback loops. Forcing them into a DAG would mean breaking the most important psychological interactions.

CAPS wins because the epic requires behavioural **attractors** — stable states the system settles into. Bayesian networks give probability distributions, not attractors. The difference matters: a fearful-avoidant person doesn't have a "72% probability of approach" — they oscillate between approach and avoidance, and both states are genuine attractors in their personality system. CAPS models this as period-2 oscillation during settling. A Bayesian network can't.

### The Bayesian Element That Survived

But the Bayesian analysis wasn't wasted. It revealed a genuine hybrid opportunity in a place I wasn't expecting.

The CAPS network's connection weights come from three sources: published effect sizes (BIS → anxiety, g = 1.21), clinical consensus (attachment disruption → trust erosion), and estimates (where the literature gives direction but not magnitude). These have fundamentally different confidence levels.

Under standard Rescorla-Wagner learning, all connections update at the same rate regardless of how well-calibrated their starting weight is. An empirically-validated weight from a 200,000-subject meta-analysis updates as readily as a rough guess. That's wrong.

The fix: each connection carries a `precision` field that modulates the effective learning rate. Empirical weights start at high precision (10.0) — they resist change. Estimated weights start at low precision (1.0) — they adapt rapidly to new evidence. When experience confirms a weight (small prediction error), precision increases. When experience contradicts it (large prediction error), precision drops.

This isn't full Bayesian posterior updating — the domain mismatch prevents that (connection weights span [-2, +2], not the [0, 1] support of a Beta distribution). But it captures the essential insight: uncertain weights should learn faster than confident ones. Radford Neal showed in 1992 that connectionist weights can be viewed through a Bayesian lens. Brenden Lake's 2025 BIML work showed that distilling Bayesian priors into neural networks outperforms either approach alone. The precision field is the lightweight version — one extra double per connection, about 20 lines of update logic.

## Six Models, One Network

The engine doesn't implement CAPS in the abstract. It implements a specific topology — six psychological models composed into a single network through shared mediating nodes.

**Attachment theory** (Bowlby, Ainsworth) gives us the self/other internal working models. Secure attachment builds positive self-worth and other-reliability. Anxious attachment builds hypervigilant proximity-seeking through variable-ratio reinforcement — the most extinction-resistant schedule in operant conditioning, which is why anxious attachment is so stubbornly resistant to change.

**Gray's BIS/BAS** gives us the approach/avoidance motivational systems. Three interacting subsystems: BAS (approach, dopaminergic reward), BIS (inhibition, anxiety, goal conflict), FFFS (fight/flight/freeze, fear). The Joint Subsystems Hypothesis confirms all three must be modelled together — isolated chains miss the interaction effects.

**Beck's CBT** gives us schema activation and cognitive distortions. Core beliefs operate on a quantitative continuum — the difference between normal and pathological is the degree of bias, not its kind. Distortions (catastrophising, mental filtering, all-or-nothing) are weight multipliers on existing connections, not separate nodes. They're defined as data in the topology YAML, so the engine applies them generically without knowing which distortions are "CBT" and which might come from a future model.

**Trauma response models** give us sensitisation and the four chronic patterns (fight, flight, freeze, fawn). The kindling model — repeated sub-threshold stimulation progressively lowers activation thresholds — explains the dose-response relationship in ACE data: each additional adverse childhood experience multiplies risk, not adds to it. The CAPS network models this by lowering activation thresholds alongside increasing connection weights. Standard weight-only models can't represent sensitisation.

**Operant conditioning** is the universal weight update engine. The Rescorla-Wagner prediction error rule (ΔW = α × β × (λ − ΣW)) provides the mathematical foundation for all learning across the network. The Pearce-Hall extension adds dynamic salience — attention increases when outcomes are surprising, decreases when predictable. This equation runs the same regardless of which psychological model the connection belongs to.

**Bandura's social learning** adds the observational channel. Characters can learn from watching others, not just from direct experience. Vicarious reinforcement runs at 15-25% of direct experience strength — strong enough to acquire fear of dogs by watching someone else get bitten, weak enough that it fades without subsequent direct confirmation.

The key insight is in the composition. These aren't six separate systems running in parallel. They share mediating nodes. Attachment's internal working models ARE Beck's core beliefs — they're the same bipolar self/other dimensions, written to by different input pathways. Gray's FFFS IS the trauma response system — same neurobiological substrate, different entry points. When the attachment model writes to `self_worth` and the CBT model writes to `self_worth`, their effects interact through the same node. Novel emergent behaviour comes from model interactions that no single model predicts.

## The Dual-Weight Architecture

One subtlety that turned out to be architecturally significant: extinction is NOT unlearning.

When a learned behaviour stops being reinforced — the feared dog is harmless, the critical parent is replaced by a supportive partner — the original connection doesn't weaken to zero. Instead, a competing inhibitory connection forms on top of it. The effective weight is excitatory minus inhibitory. The inhibitory overlay decays faster than the excitatory weight.

This explains spontaneous recovery. Someone who overcame a fear of dogs through repeated positive exposure can have that fear resurface under stress or in the original context. The old weight was never deleted — the inhibition just decayed.

Every connection in the CAPS engine carries two weight values: excitatory and inhibitory overlay, plus a decay resistance attribute. Trauma-sensitised and variable-ratio-reinforced connections have high decay resistance — they persist. This is three values per connection per agent, stored as a sparse overlay in SQLite (only non-default weights are persisted).

## Consolidation — Sleep As Computation

Humans don't re-derive their fear of dogs every time they see one. The connection was built during sleep — memory consolidation wired the neural pathway. The behaviour is there, ready to fire, before the stimulus appears.

The CAPS engine follows the same pattern. During consolidation phases (analogous to sleep), the `BehavioralSynthesisPhase` processes accumulated experiences through the network:

1. Classify each experience into situation types (which input nodes activate)
2. Activate input nodes and run CAPS settling (max 100 iterations of synchronous parallel update — sub-millisecond per agent)
3. Update connection weights via Rescorla-Wagner with precision modulation
4. Apply inhibitory decay across all connections
5. Write behavioural attractor nodes to the MindMap BEHAVIORAL subgraph

The LLM receives the finished behavioural profile — crystallised attractor strengths, not raw memories to reason over. A character with strong `withdraw` and `distrust` attractors will behave accordingly without needing to re-process their entire history of betrayal each conversation turn.

## The Topology Is Data, Not Code

A design decision that emerged from the spec review: cognitive distortions, disposition modulation, and the full node taxonomy are all defined in a YAML topology file. The engine reads this data and applies it generically.

```yaml
distortions:
  - id: catastrophizing
    baseThreshold: 0.7
    multiplierMin: 1.5
    multiplierMax: 2.0
    effect: MULTIPLICATIVE
    negativeWeight: 1.0
    positiveWeight: 0.3
    targetCategories: [self_model, world_model, response_outcome]
```

This means adding a seventh psychological model — schema therapy's mode model, ACT's psychological flexibility framework, whatever the research validates next — is a topology authoring exercise, not an engine rewrite. New nodes, new connections, new default weights. The settling algorithm, weight update rules, distortion application, and convergence detection don't change.

The engine is model-agnostic. The psychology is data.

## What This Makes Possible

The NPC AI market is projected at $2.4 billion in 2026, growing to $7.2 billion by 2030. Every player in this space does personality via prompting. No competitor has mechanical psychology.

But the market numbers aren't the interesting part. The interesting part is what becomes possible when behaviour emerges from a computational model rather than a prompt:

**Characters that genuinely change.** Not a prompt update that says "you are now more cautious." A weight shift on the BIS pathway, visible in the connection matrix, that produces cautiousness as an emergent property of threat-processing through shared mediating nodes. The change persists across context windows because it's stored in the weight matrix, not in the conversation history.

**Clinical simulations with validity.** A simulated patient with anxious attachment + CBT negative self-schema produces clinically recognisable behavioural patterns because the connection weights come from meta-analyses — van IJzendoorn (r=.24-.32) for attachment sensitivity, Zhang et al. (r=.42) for attachment anxiety → mental health. Medical education can validate that the simulation behaves like a real patient with that clinical profile, not just an LLM performing one.

**Extinction and relapse.** The dual-weight architecture means old behaviours genuinely return under stress. A character who overcame social anxiety through repeated positive social experiences still has the original BIS-strengthened connections underneath the inhibitory overlay. Enough stress, enough time without reinforcement, and the old pattern resurfaces. No character system on the market models this.

**Compositional personality.** Combine anxious attachment with high BIS sensitivity and catastrophising distortion. The interaction through shared nodes (self_worth, threat_sensitivity, arousal_level) produces a specific clinical profile that no single model alone predicts. The CAPS settling algorithm discovers the emergent attractors. The designer specifies ingredients; the network computes the dish.

The structural moat is that prompted personality is easy to copy. A mechanical psychology engine grounded in published research with empirical effect sizes is not. Competitors would need to replicate the entire pipeline — literature review, model composition analysis, topology encoding, weight derivation, settling algorithm, convergence detection, dual-weight architecture, consolidation integration. None of that is prompt engineering. It's computational psychology systems engineering.

There's a long road between an architecture and a product. Calibration is an open research problem — the estimated weights need empirical validation. The situation classifier is rule-based for now, pending training data for an ML approach. And the ultimate test is whether LLM agents that receive CAPS-generated behavioural profiles actually behave in ways that human observers recognise as psychologically coherent.

But the foundation is solid. The research is grounded. The topology is authored. The engine is designed. Implementation starts next session.
