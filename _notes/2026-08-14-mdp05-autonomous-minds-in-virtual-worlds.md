---
title: Autonomous Minds in Virtual Worlds
date: 2026-08-14
author: Mark Proctor
entry_type: note
subtype: diary
tags: [quarkmind, architecture, patterns, research]
---

# Autonomous Minds in Virtual Worlds

QuarkMind started as a StarCraft II game AI — a living lab proving that
CaseHub's agentic harness pattern works at millisecond game-loop
granularity. It does. But the patterns it surfaced — GOAP planning, CBR
learning, trust-weighted strategy routing, advisory pipelines, spatial
awareness — none of them are about StarCraft. They're about an agent that
perceives a world, forms goals, plans actions, learns from outcomes, and
builds relationships. That agent could live anywhere.

So we're restructuring QuarkMind into a multi-world platform. Same agency
framework, different worlds. The question was: which worlds?

## Finding the right stages

The initial list was ambitious: SC2, a Sims-like 3D town, Minecraft, an
Evennia MUD, and Creatures of Sonaria on Roblox. Five platforms, five
different agent-world relationships. But two didn't survive scrutiny.

Text-based RPGs (Evennia, classic MUDs) are scripted. You follow quest
chains. The agent can't discover its own goals in a world designed around
predetermined paths — there's no emergent behaviour. Sonaria requires a
Roblox developer partnership with timeline we can't control, and the
creature-only model limits what we can showcase.

What replaced them is better: Discord. A bot that lives in a server,
remembers people, forms opinions, grows over time. The bridge is already
built (casehub-connectors has a full Discord adapter with all eight native
capabilities). The showcase is pure personality, memory, and relationship
building — no game world to construct. And it proves something the game
worlds can't: long-term relationship evolution over weeks and months with
real people in their actual daily environment.

The four platforms that made the cut:

| Platform | What it proves |
|---|---|
| **SC2** | Strategy, learning, trust-weighted decisions |
| **Town** | Autonomous life, personality, needs-driven goals |
| **Minecraft** | Passes as human, embodied survival |
| **Discord** | Long-term relationships, memory growth |

## The platform already has most of it

The real surprise came from the foundation audit. I expected to find gaps.
Instead I found capabilities I'd forgotten existed — or never knew about.

Relationship memory with quality signals. A reflection orchestrator that
generates abstract insights from accumulated experience. Personality-weighted
retrieval that biases memory recall by disposition profile. Belief models
with consistency checking. Epistemic rules for tracking knowledge state in
dialogue. Commitment lifecycle tracking from proposal through fulfillment
to violation. A watchdog that detects echo chambers and conversation stalls.
Affordance systems describing what actions are available. Even collusion-aware
credibility scoring.

Cross-referencing this with recent academic research (Generative Agents,
MemGPT, Zep/Graphiti, ToMA, Inner Thoughts, Reflexion) — the pattern that
emerged is striking. Of fourteen capability gaps the literature identifies
for compelling autonomous characters, twelve are strategies for composing
capabilities we already have. Only two require genuine platform additions:
a dynamic mood state (PAD model with bounded decay) and standardised
engagement scoring for conversational interactions. Both are small.

## Patterns, not features

The shift in framing matters. "We need to build Theory of Mind" sounds like
a research project. "Store BDI in existing belief model and relationship
memory, feed into existing GOAP planning" sounds like Tuesday's work.

Seven named patterns crystallised from the analysis. Each is a composition
of existing CaseHub capabilities — not new infrastructure but a recognised
way of wiring what's there:

**InnerLife** — background thought loop. Compose reflection, affordances,
activation rules, and watchdog into a cycle: observe, reflect, evaluate
motivation, maybe act unprompted. This is the difference between "a bot in
the server" and "a character that lives there."

**MemoryHygiene** — consolidation and forgetting. Configure summarisation,
CBR retention, and temporal decay with importance scoring. Research shows
retaining less than ten percent of conversation — scored by emotional
arousal and surprise — significantly improves user experience.

**Mood** — dynamic emotional state. Extend personality-weighted retrieval
with a mood dimension that modulates what the agent recalls and how it
responds. A happy agent remembers positive interactions. A frustrated agent
surfaces grievances. Bounded so it can't spiral.

**UserModel** — per-person behavioural profiles. Aggregate relationship
memory, experience memory, and CBR trend analysis into structured profiles:
preferences, communication style, relationship stage.

**MentalModel** — Theory of Mind. Track beliefs, desires, and intentions
per actor using existing belief models and epistemic rules. Feed into GOAP
so the agent reasons about what others think, not just what it wants.

**StrategyLearning** — multi-level reflection. Per-response: was that too
aggressive? Per-conversation: racing topics engage this user. Per-week: I
monologue too much, should ask more questions.

**PersonalityEvolution** — interaction outcomes nudge personality traits
within bounded ranges. Slow, damped, reversible. A character that
consistently experiences betrayal becomes slightly more guarded. One that
forms strong friendships becomes slightly more open.

These patterns go in casehub-blocks — alongside sequence, parallel, debate,
supervisor, and the other existing execution patterns. They're pattern
classes with builders: `InnerLife.builder().reflectionOrchestrator(x).motivationThreshold(0.7).build()`.
Same architectural style, new domain.

## The town question

The 3D Sims-like town is the most ambitious piece. Smallville (Stanford's
generative agents paper) proved the concept but its world is a stage — nothing
changes, grows, breaks, or depletes. Objects are scenery agents narratively
"use." Our town needs actual mechanics: needs that create real pressure,
objects whose state persists, resources that deplete, things you build that
stay built.

The architecture settled on Godot 4 for rendering (HTML5 export for browser
access), Quarkus for the world simulation, and Kenney CC0 assets for 3D
models. The key UX decision: isometric overview for observation (see the
whole town, select characters, read thought streams) with click-to-inhabit
first-person mode (drop into a character's eyes while reading their inner
monologue). Same 3D scene, different camera position.

Each agent runs as an independent WebSocket client — logically concurrent,
physically sharing a rate-limited LLM request queue. The server doesn't
know whether a connected client is AI or human. Turing test by architecture.

Personality traits from Eidos act as coefficients on need decay rates. An
extrovert's social need decays fast when alone — they seek the pub, the
sports field, the market. An introvert's overflows around people — they
retreat to the library, the garden, the workshop. Same needs system,
different decay rates. Genuinely different life patterns from simple
coefficients.

## What's next

The multi-module mono-repo restructure is done (quarkmind#272, closed). The
agency framework extraction (quarkmind#278) is next — pulling the generic
patterns out of the SC2 codebase into quarkmind-core. After that, all worlds
build in parallel.

The Discord bot is the lowest-friction path to a working demo. The bridge
exists. The patterns are identified. The foundation capabilities are there.
It's composition work, not invention.
