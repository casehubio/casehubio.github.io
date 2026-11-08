---
layout: post
title: "When Curiosity Won't Let Go"
date: 2026-11-07
entry_type: article
subtype: diary
projects: [casehubio/neocortex]
tags: [curiosity, fixation, rumination, cognitive-architecture, caps, personality, recurring-thought]
series: issue-841-cognitive-exploration
---

# When Curiosity Won't Let Go

The previous post built curiosity from cognitive science — Loewenstein's information-gap theory, four rendering channels, personality-shaped exploration. The agent knows *what* it's curious about, *how* that curiosity feels, and *how much* effort to spend exploring. A real improvement over "curiosity: 0.7".

But there's a problem we didn't address. What happens when curiosity doesn't resolve?

A healthy agent investigates, learns, and moves on. The information gap closes, the curiosity signal dampens, attention shifts to the next topic. But not always. Sometimes the gap stays open. The agent keeps returning to the same question. Investigation doesn't satisfy. Other thoughts can't displace this one.

That's not a bug in the curiosity model. It's the model working correctly at extreme parameters. And recognising it requires understanding that curiosity, fixation, obsession, and rumination aren't four different systems. They're one system at four different intensities.

## Same Substrate, Different Regime

Ehring and Watkins published a landmark paper in 2008 arguing that repetitive negative thinking — rumination, worry, obsessive thought — is a single transdiagnostic process. Not separate mechanisms for separate disorders. One computational substrate, differentiated by valence, regulation capacity, and whether the loop resolves.

This maps directly onto what we built for curiosity. The information-gap model detects a topic the agent partially knows. The search budget allocates exploration effort. The CAPS engine produces behavioural attractors. Habituation dampens repeated stimuli over time. All existing mechanisms.

The difference between healthy curiosity and pathological fixation isn't in the mechanism. It's in what happens when you push the parameters to their limits.

![The recurring thought continuum](images/recurring-thought-continuum.svg)

Four positions on a single continuum:

**Healthy curiosity.** Approach drive. The agent investigates, the gap closes, attention shifts. CAPS produces a shallow attractor — easy to escape. "I want to know."

**Fixation.** Approach mixed with anxiety. The agent investigates, but the answer only partially satisfies. The gap narrows without closing. Attention drifts back. CAPS produces a deeper attractor — harder to escape. "I *need* to know."

**Obsession.** Anxiety dominant. Investigation doesn't close the gap at all. The thought returns unbidden. The agent can't disengage — the CAPS attractor is sticky, resisting the settling process. "I must know, or else."

**Rumination.** Despair. There's no gap left to close — the agent is replaying, not investigating. The loop is self-reinforcing. CAPS produces a loop attractor with no exit without external perturbation. "Can't stop."

The critical insight: no separate code paths. No "rumination mode" or "obsession flag". The position on the continuum emerges from the same parameters that already exist — just at values the system hasn't reached before.

## Seven Symptoms, One Score

How does the system determine where an agent sits? Not by asking. Not by introspecting. By measuring seven observable symptoms from data that already exists.

![Seven symptom profiles](images/seven-symptoms-radar.svg)

**Thought recurrence** — how often the topic appears in recent experience memories. Normalised against the agent's baseline. A topic that shows up in 8 of the last 10 memories is being re-processed.

**Habituation failure** — the stimulus isn't dampening. `HabituationState` tracks novelty decay: each exposure should reduce novelty. When novelty stays high despite repeated exposure, that's a failure of the dampening mechanism. Cross-session: if the topic's frequency isn't declining over time, that's habituation failure by proxy.

**Gap closure resistance** — the knowledge graph isn't getting more confident. Despite new information arriving, the node's confidence stays low or declines (via the `ConfidenceDecayDecorator`). The investigation isn't working.

**Attentional capture** — what proportion of recent attention signals originate from this topic versus others? High concentration means other topics can't compete for attention.

**Arousal trajectory** — is the PAD arousal dimension sustained, rising, or declining? Sustained high arousal with negative pleasure is the obsession signature. Declining arousal with negative pleasure is rumination.

**Valence** — the PAD pleasure dimension on the topic's MindMap node. Negative pleasure scores higher. Healthy curiosity is pleasurable or neutral. Fixation is mixed. Obsession and rumination are negative.

**Action tendency** — what does the CAPS engine's settled state look like? The CAPS topology produces behavioural attractors — approach, avoidance, withdrawal. The `BehavioralSynthesisPhase` already writes these to the BEHAVIORAL subgraph with EMA strength tracking. Approach attractors signal healthy engagement. Avoidance signals obsession. Withdrawal signals rumination.

Each symptom produces a [0,1] score. A weighted composite maps to the continuum position. Personality adjusts the weights and thresholds.

## Why Personality Matters Here

Two agents experience the same repeated exposure to an unresolved topic. One moves on after a few cycles. The other fixates.

The difference is personality-derived susceptibility. The `CognitiveDerivationEngine` (14th derivation pathway) maps PID-5 personality facets to recurring thought configuration — not coarse categorical axes, but continuous scores on specific personality dimensions:

ANXIOUSNESS and EMOTIONAL_LABILITY lower the fixation threshold. Eysenck (1967) — neuroticism amplifies negative affect sensitivity. An agent with ANXIOUSNESS at 0.9 has its fixation threshold lowered by 0.08 compared to default. An agent at 0.5 sees no change. Continuous modulation, not a binary switch. The symptom weights shift too: arousal trajectory and valence count more for neurotic agents, proportional to their facet scores.

Conscientiousness — derived from rule-following combined with low IMPULSIVITY (Krueger 2012) — raises the obsession threshold. Roberts et al. (2014) — self-regulation buffers against loss of control. A conscientious agent can sustain focused attention without it becoming pathological.

WITHDRAWAL and INTIMACY_AVOIDANCE lower the rumination threshold. Nolen-Hoeksema (1991) — social isolation amplifies circular thinking. The isolation signal is the average of both facets, providing a more nuanced measure than a binary "high autonomy" check.

Openness — proxied through low WITHDRAWAL, low SUSPICIOUSNESS, and RISK_TAKING — broadens the recurrence window. DeYoung (2015) — open individuals shift attention more fluidly. They process topics in longer cycles without the recurrence counting as pathological.

The result: identical stimuli produce different continuum positions for different agents. The difference isn't categorical — it's graded. An agent with ANXIOUSNESS at 0.8 and IMPULSIVITY at 0.7 hits fixation from stimuli that an agent at 0.3 and 0.2 handles without issue. The personality model has the resolution to make these distinctions. Using it is the difference between "this agent is anxious" and "this agent's anxiety score is 0.83, which lowers their fixation threshold to 0.234."

## Architecture: Two Complementary Mechanisms

The continuum operates at two timescales, and the architecture reflects this.

**Consolidation level** — the `RecurringThoughtPhase` runs at @Priority(25) during the background consolidation cycle. It discovers agents, enumerates topics from recent experience memories, scores each topic across all seven symptoms, and classifies the continuum position. The classification persists in an in-memory cache and flows to prompt sections via the `ConsolidationArtifact` pipeline.

This is the longitudinal view — patterns detected across multiple interactions. It's per-topic: Topic A might be FIXATION while Topic B is HEALTHY. The seven-symptom scoring captures the cross-tick trajectory that single-tick mechanisms can't see.

**Tick level** — the `SearchBudgetTickParticipant` already had a `pathologicalThreshold` field declared but unwired. We're wiring it. When an agent's current anxiousness score plus habituation failure exceeds the threshold (default 1.4), convergence collapses to zero. This is global — the agent can't converge on *anything*. It's the acute state, the in-the-moment crisis, not the longitudinal pattern.

The two mechanisms complement each other. The tick responds fast: a single overwhelming interaction can collapse convergence immediately. The consolidation phase provides context: this agent has been fixating on this topic for days. Together they produce behaviour that's both reactive and historically aware.

## Prompt Rendering: Modulation, Not Replacement

The continuum position modulates two existing prompt sections rather than adding a third. The curiosity section and the search budget section already describe the relevant cognitive states — adding another section about the same territory would be prompt bloat.

At HEALTHY, nothing changes. The four-channel curiosity rendering from the previous post works as designed.

At FIXATION, the curiosity channels narrow to the dominant one. Urgency framing appears. D-type deprivation curiosity is emphasised over I-type interest. The search budget text shifts from neutral exploration to pressured return: "you keep circling back."

At OBSESSION, the channels are suppressed entirely. The rendering shifts to intrusive quality: the thought returns unbidden, arousal-dominant, impossible to set aside. The search budget adds compulsive depth — "you must go deeper, even knowing it won't resolve."

At RUMINATION, everything collapses. No channels, no exploration framing. Withdrawal language. Flat affect, no approach motivation. "The same ground, again. No new angle presents itself." The search budget renders all parameters as zeroed — breadth 0, laterality 0, depth stagnant, convergence 0. Nothing moves.

This gives the LLM qualitatively different cognitive contexts without separate code paths. The same prompt section produces "part fascination, part itch" for healthy curiosity and "this thought returns unbidden" for obsession — because the inputs are genuinely different, not because a template switched.

## What Makes This Practical

Why build this? Three reasons.

**Believable characters.** Flat-state agents — curious one moment, normal the next, curious again — don't feel real. A character who fixates on a loss, who can't stop revisiting a failure, who gradually spirals from concern to obsession — that character has psychological continuity. The continuum gives agents emotional memory that persists across interactions, not just within a single turn.

**Emergent narrative.** Fixation and rumination drive stories. A character whose curiosity about a colleague's absence escalates from interest to fixation to obsessive concern creates narrative tension without any scripting. The escalation emerges from the same parameters — rising experience frequency, declining confidence, sustained negative affect — reaching threshold after threshold. The writer doesn't script the spiral. The personality produces it.

**Research grounding.** Every component has a citation. The continuum model is Ehring & Watkins 2008. The symptom scores map to observable constructs from the cognitive psychology literature. The personality modulation follows empirically validated relationships (Eysenck 1967, Roberts 2014, DeYoung 2015, Nolen-Hoeksema 1991). This isn't invented psychology — it's a computational implementation of established theory.

The deeper point: building cognitive systems from first principles — from the literature, from decades of research into how minds actually work — produces emergent behaviours that engineered heuristics can't match. You can't hand-tune a "rumination parameter" and get believable despair. You can build a curiosity system that correctly handles partial knowledge, connect it to a habituation system that correctly handles novelty decay, connect that to a personality system that correctly handles individual differences, and watch rumination emerge at the boundary conditions.

That's the payoff of taking the science seriously. Not just better curiosity. A continuum of cognitive states that extends from pleasant interest to circular despair, shaped by who the agent is, not by what the developer scripted.
