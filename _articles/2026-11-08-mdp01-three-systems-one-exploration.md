---
layout: post
title: "Three Systems, One Exploration"
date: 2026-11-08
entry_type: article
subtype: diary
projects: [casehubio/neocortex]
tags: [curiosity, search-budget, cognitive-architecture, integration, exploration, personality, modulation]
series: issue-841-cognitive-exploration
---

# Three Systems, One Exploration

The previous two posts built each piece in isolation. The curiosity engine detects information gaps and renders them through four psychological channels. The search budget allocates exploration effort across breadth, depth, laterality, and convergence. The recurring thought continuum classifies where curiosity sits on the healthy-to-pathological spectrum. Each works. Each produces cognitive state descriptions that give an LLM qualitatively different prompt contexts.

But they don't talk to each other.

The curiosity engine scores gaps and feeds into the drive orchestrator as one of four drive sources. The search budget reads from the drive orchestrator. So curiosity affects the search budget — but only indirectly, as a scalar intensity. The search budget knows the agent is curious. It doesn't know *how* the agent is curious.

That matters. An agent in diversive curiosity mode — scanning broadly, everything looks interesting, nothing specific dominates — should explore wide and shallow. High breadth, high laterality, low convergence. An agent in specific focus — locked onto one gap, Loewenstein's tip-of-the-tongue proximity high — should explore narrow and deep. High depth, low breadth, high convergence.

The search budget couldn't produce this distinction. It received composite drive intensity and dominant axis. Both scenarios had the same dominant axis (CURIOSITY) and potentially similar intensity. The channel data — specific vs diversive focus, hedonic type, comprehension level, gap proximity — stayed in the curiosity engine and never reached the budget computation.

![Three systems and their connections](images/exploration-integration.svg)

## What scalar reduction loses

The problem is general. When you reduce a multidimensional psychological model to a single number, you lose the distinctions that matter most. "Curiosity: 0.7" tells you the agent is curious. It doesn't tell you whether the curiosity is pleasurable interest in a new topic (Litman's I-type) or anxious need to resolve an uncomfortable knowledge gap (D-type). It doesn't tell you whether the agent is scanning broadly across many topics (Berlyne's diversive focus) or locked onto a specific question (specific focus). It doesn't tell you whether the agent understands most of what it's looking at (high comprehension) or is lost and grasping (low comprehension).

These aren't academic distinctions. They map directly to how exploration should work:

- An agent with **specific focus and high gap proximity** is close to understanding something. It should go deep, not wide. Narrowing breadth and boosting depth keeps it on the trail.
- An agent with **diversive focus** has multiple threads pulling at it. It should scan broadly, make lateral connections, and resist premature convergence.
- An agent with **D-type hedonic** curiosity feels the gap as deprivation. The pull to resolve is stronger than the pleasure of exploring. Convergence should increase — close the gap.
- An agent with **low comprehension** doesn't understand what it's seeing. It needs different angles — higher laterality, not deeper dives into material it can't parse.
- An agent with **high comprehension** is connecting pieces. Synthesis is within reach. Convergence should accelerate.

A scalar can't encode any of this. The four-channel curiosity model already captures it. The search budget just wasn't listening.

## The coupling

The fix is an additive modulation layer in `SearchBudgetTickParticipant`. After the existing computation — sigmoid activation from drive intensity, axis bonuses, mood and gut feeling adjustments, threat-collapse — channel data from `CuriosityDrive.lastChannel()` applies five rules that adjust the four budget parameters.

![Channel modulation rules](images/channel-modulation-rules.svg)

Two design decisions mattered here.

**Continuous gradients, not thresholds.** Comprehension is a float between 0 and 1. Gap proximity is a float between 0 and 1. I could have gated on arbitrary thresholds — "if comprehension < 0.3, boost laterality." But thresholds create discontinuities. An agent at 0.29 comprehension gets a full laterality boost; at 0.31 it gets nothing. The actual psychological relationship is continuous. So the modulation scales linearly: `laterality += (1 - comprehension) * magnitude`. At comprehension 0.1, laterality gets 90% of the magnitude. At 0.5, it gets 50%. At 0.9, it gets 10%. No arbitrary boundary, no discontinuity. The discrete inputs — focus type (SPECIFIC/DIVERSIVE) and hedonic type (I_TYPE/D_TYPE/BLENDED) — are naturally categorical. Those stay as switches.

**Per-agent configurable magnitudes.** The five rules are universal — every agent with curiosity channels gets them. But the magnitudes are personality-shaped. A `ChannelModulationConfig` record holds eight magnitude values, one per rule (some rules affect two parameters). The Cognitive Derivation Engine maps PID-5 personality facets to these magnitudes using the same linear-delta pattern as the existing search budget derivation:

- **Openness** (derived from withdrawal, intimacy avoidance, suspiciousness) scales the diversive modulations. High openness means diversive curiosity fans wider and resists convergence harder.
- **Conscientiousness** (derived from rule-following and impulsivity) scales the specific-focus modulations. A methodical agent digs deeper when locked onto a question and converges faster when comprehension arrives.
- **Anxiousness** scales the deprivation-convergence boost. An anxious agent feels unresolved gaps more acutely and pushes harder to close them.

The config sits as a nullable field on `SearchBudgetConfig`. When null — no channel modulation. Existing agents behave exactly as before. The pattern is the same nullable graceful degradation used throughout the cognitive pipeline: gut feeling, mood, appraisal, threat assessment. Each input is optional. Absence means silence, not failure.

## What it looks like in practice

Consider two agents — one with high openness, one with high conscientiousness — both encountering the same knowledge gap about quantum entanglement.

The open agent's curiosity engine fires with diversive focus. Entropy across gap scores is high; many related topics look interesting. The search budget receives this channel state and adjusts: breadth goes up by `0.17` (diversive breadth boost, personality-amplified), laterality goes up by `0.14`, convergence drops by `0.11`. The agent's prompt describes a cognitive state where "multiple angles present themselves naturally — single explanations rarely satisfy."

The conscientious agent's curiosity engine fires with specific focus. One gap dominates; gap proximity is high at 0.85. The search budget adjusts differently: depth goes up by `0.85 * 0.16 = 0.14`, breadth drops by `0.85 * 0.11 = 0.09`. Comprehension is moderate at 0.6, so convergence gets `0.6 * 0.14 = 0.08`. The agent's prompt describes something narrower and more purposeful — "a specific question has your attention; you want to understand this one thing."

Same stimulus, different personality, different curiosity channel state, different search budget, different prompt, different LLM behaviour.

## What comes next

The curiosity-to-budget coupling is the first of three connections in the exploration integration epic. The goal-directed curiosity coupling comes next — making `InformationGapDetector` weight gaps by their proximity to active goals, so an agent pursuing a research objective finds goal-adjacent gaps more interesting than distant ones. After that, empirical scenario testing: short-run pipelines with specific agent profiles and stimuli, verifying that the integrated system produces the right cognitive texture across personality differentiation, goal-directed exploration, and recurring thought escalation.

The recurring thought continuum already modulates both curiosity and search budget prompt sections — FIXATION adds urgency to curiosity rendering, OBSESSION collapses search budget output. But it doesn't yet feed back into the channel computation itself. Whether a thought that's becoming obsessive should shift curiosity from diversive to specific focus, or whether that would create a feedback loop that makes obsession worse, is a question that belongs in the empirical testing phase.
