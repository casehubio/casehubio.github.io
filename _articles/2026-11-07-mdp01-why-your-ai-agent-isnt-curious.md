---
layout: post
title: "Why Your AI Agent Isn't Curious"
date: 2026-11-07
entry_type: article
subtype: diary
projects: [casehubio/neocortex]
tags: [curiosity, cognitive-architecture, information-gap, loewenstein, personality, drives]
series: issue-841-cognitive-exploration
---

# Why Your AI Agent Isn't Curious

Most AI agent frameworks model curiosity as a scalar. A number between 0 and 1. Derived from — what, exactly? Typically something like memory retention statistics, or a random noise parameter, or a hand-tuned exploration rate. The agent gets a curiosity score, the score gets injected into its prompt, and the LLM does its best to pretend that "curiosity: 0.7" means something.

It doesn't. A flat scalar tells the LLM nothing about what kind of curiosity to express, what to be curious about, or how that curiosity should shape its behaviour. "0.7 curious" is as informative as "0.7 happy" — it provides magnitude without meaning.

## The Retention Trap

The curiosity drive in neocortex's cognitive stack used to work this way. `CuriosityDrive` asked the `MemoryHygieneOrchestrator` how many low-retention memories existed, divided by total scored memories, mixed in some consolidation group diversity, and produced a number. The number went into a `DriveProfile`, which went into the prompt, which told the LLM something like "Curiosity: 0.4 — 5 low-retention memories across 3 groups."

The problem is obvious once you say it out loud: memory retention has almost nothing to do with curiosity. Low retention means the agent is forgetting things. That might correlate with curiosity — you're curious about things you can't quite remember — but it's a proxy so distant from the actual phenomenon that the signal is noise.

Curiosity isn't about what you're forgetting. It's about what you almost know.

## Loewenstein's Insight

George Loewenstein nailed this in 1994. His information-gap theory says curiosity is felt deprivation — the uncomfortable awareness of a gap between what you know and what you want to know. The key insight is that the relationship is non-linear:

![Loewenstein Information-Gap Curve](images/loewenstein-entropy-curve.svg)

Know nothing about a topic? No curiosity. You don't know what you don't know. Know everything? No curiosity either — the gap is closed. But know *some* — enough to see the shape of what you're missing — and curiosity peaks. The "tip of the tongue" phenomenon. The partial knowledge sweet spot.

This maps directly to binary entropy. For a node in the knowledge graph with confidence *p*, the gap intensity is H(p) = −p·log₂(p) − (1−p)·log₂(1−p). Maximum at p = 0.5. Zero at both extremes. It's a beautiful function for this purpose — it captures Loewenstein's non-linearity in a single line of code.

But intensity alone isn't enough. Two agents at 0.7 curiosity should feel it differently based on who they are.

## Four Channels, Not One Number

Curiosity has texture. Five decades of cognitive psychology research identified at least four orthogonal dimensions. We implemented all four:

![Four-channel curiosity model](images/curiosity-four-channels.svg)

**Hedonic channel (Litman 2005).** The same curiosity can feel pleasant or aversive. Jordan Litman's Interest/Deprivation model distinguishes I-type curiosity — pleasurable anticipation, where exploration itself is the reward — from D-type — an uncomfortable itch, where the gap is aversive and resolution is urgent. Which type an agent experiences is personality-derived: openness pushes toward I-type, neuroticism toward D-type. Same information gap, completely different felt experience.

**Comprehension channel (Silvia 2006).** Paul Silvia's appraisal theory adds a second dimension: is the agent grasping the material or drowning in it? High novelty with high comprehension produces interest ("the pieces are connecting"). High novelty with low comprehension produces confusion ("the more I look, the less I understand"). We derive this from the mean confidence of the agent's top knowledge gaps — high confidence gaps mean the agent has context, low confidence means it's lost.

**Focus channel (Berlyne 1960).** Daniel Berlyne's distinction between specific and diversive curiosity captures attentional shape. Is the agent's attention locked on one question, or is it scanning broadly? We compute this from the entropy of the gap distribution — one dominant gap produces specific focus, many similar-intensity gaps produce diversive exploration. The same curiosity intensity feels very different when it's laser-focused versus diffuse.

**Gap proximity channel (Loewenstein 1994).** The non-linear sweet spot from the entropy curve above. How close is the agent to knowing? This channel distinguishes "tip of the tongue" from "vast unknown" — both can produce curiosity, but the felt quality is different. Near-complete knowledge with one missing piece creates a specific, urgent pull. Distant partial knowledge creates a broader, more exploratory drive.

## What This Looks Like in Practice

When the cognitive pipeline ticks, `InformationGapDetector` scans the MindMap knowledge graph for nodes in the partial-knowledge sweet spot. Each node gets scored: binary entropy of its confidence × attention weight from its edge connectivity. A node at 0.4 confidence with 3 edges to known concepts produces a meaningful gap. A node at 0.05 confidence with no edges produces nothing — the agent doesn't know what it doesn't know.

The `CuriosityChannelComputer` takes those gap scores and the agent's personality-derived deprivation bias and produces a `CuriosityChannel` — the four-dimensional curiosity state. The prompt section renders this as prose:

> "Part fascination, part itch — the not-knowing is both exciting and uncomfortable. The pieces are starting to connect. Your attention keeps returning to quantum-entanglement. Curiosity is the dominant pull right now."

Compare that to "Curiosity: 0.7". The four-channel rendering gives the LLM enough information to actually differentiate its behaviour. An I-type curious agent explores playfully. A D-type curious agent investigates urgently. A specifically-focused agent drills into one topic. A diversively-curious agent scans broadly. These are qualitatively different behaviours, and they emerge naturally from qualitatively different descriptions — not from different numbers.

## Proportional, Not Absolute

One more thing the flat scalar gets wrong: it presents curiosity in isolation. Is 0.7 high or low? For whom? Compared to what?

The prompt section renders three proportional dimensions:

**Baseline deviation.** Each agent has a personality-derived curiosity baseline. An agent whose normal curiosity is 0.8 experiencing 0.7 is *below their norm* — quieter than usual. An agent whose baseline is 0.3 experiencing 0.7 is having a spike. The prompt says "stronger than your usual curiosity" or "quieter than usual" — not the number.

**Drive rank.** Curiosity doesn't exist in isolation. Four drives compete: curiosity, competence, affiliation, autonomy. The prompt renders the competition: "Curiosity is the dominant pull" versus "Competence edges ahead, but curiosity is present." Rank matters more than magnitude.

## The Search Budget Connection

This feeds directly into the cognitive search budget — a model we built earlier that converts motivational intensity into a four-dimensional exploration envelope: breadth (how many angles to explore), laterality (how far to range from the obvious), depth (how deep to go), and convergence pressure (when to stop). Personality distributes the budget. The curiosity intensity from the information-gap model is the primary fuel for that distribution.

The key design decision: curiosity provides the *intensity*, personality shapes the *allocation*. The LLM doesn't get told what investigation strategy to use — it gets the constraint envelope and fills it. A highly-curious conscientious agent goes deep on one thread. A highly-curious open agent ranges broadly. Same curiosity, different search.

## What This Opens Up

The same computational substrate — recurring attention to a knowledge gap — produces healthy curiosity at moderate intensity and pathological fixation at extreme intensity. When convergence pressure drops to zero and stays there across ticks, curiosity becomes rumination. This is the next layer: #826 models the continuum from curiosity through obsession to depressive rumination, using the same parameters at different values. Not a separate mode — the same mechanism at its boundary conditions.

That's the practical payoff of building curiosity from cognitive science rather than from a random number generator. The model doesn't just produce better curiosity. It produces a continuum that extends into territory flat scalars can't reach.
