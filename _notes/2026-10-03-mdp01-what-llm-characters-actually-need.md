---
layout: post
title: "What LLM Characters Actually Need"
date: 2026-10-03
type: technical
entry_type: article
subtype: diary
projects: [casehub]
tags: [eidos, multi-agent, character-design, taxonomy, wacky-manor]
---

I spent a day systematically stripping personality from LLM characters and then trying to put it back with nothing but a single instruction in the thinking field. The results changed how I think about LLM agent design.

## The setup

Five characters in an autonomous multi-agent scenario — a mansion mystery with villains, heroes, and bumbling protectors. Each character has a structured identity: drives, disposition axes, speech patterns, tendencies, constraints, goals. I wanted to find out which of these are load-bearing and which are noise.

I used a generic profile — renamed characters, no pop-culture references — alongside the Wacky Races profile. The generic profile is the honest test. When the Hooded Claw loses scheme narration, the model still knows something about cartoon villains. When "Vincent Marsh" loses scheme narration, there's nothing to fall back on.

## Phase 1: what's redundant

**Round 1** stripped 7 tendencies that restated what drives already said. Zero drift. Drives carry personality traits. Tendencies that duplicate them are pure noise.

**Round 2** stripped ALL villain tendencies. The generic villain's scheme narration dropped 36 percentage points. The Wacky Races villain dropped only 19pp — model priors compensating.

The finding: "narrate your schemes step by step aloud" is not a personality trait. It's a storytelling convention — medium-specific, audience-facing. Conventions don't emerge from drives. You have to encode them explicitly, and the drive intensity needs to be above ~0.85 for the model to treat it as imperative rather than flavour.

## Phase 2: the voice mode collapse

I stripped the hero character's two tendencies — "plan obsessively" and "maintain optimistic determination." His speech-pattern still said "narrates in third person."

| Marker | Before | After | Delta |
|---|---|---|---|
| Planning | 16.7% | 26.1% | +9pp (survived) |
| Optimism | 24.1% | 4.3% | -20pp (collapsed) |
| Third-person | 61.1% | 13.0% | -48pp (collapsed) |

Third-person narration collapsed despite the speech-pattern explicitly saying "narrates in third person." The model saw it. It just didn't follow it.

## LLMs prioritise goals

This is the insight that makes sense of all the rounds. LLMs are goal-oriented. They attend to objectives, constraints, actionable instructions. Everything else — voice, speech patterns, personality quirks — is ambient context that drifts under the pressure of generating a coherent response.

Third-person narration isn't a goal. It's an ambient stylistic instruction. Without the tendency reinforcing it turn after turn, the model reverted to its default first-person voice.

This maps to something everyone who's built LLM agents has noticed: characters start strong and drift over time.

## Phase 3: six ways to fix it

I ran six experiments, each testing a different metacognitive instruction in the thinking field. Hartwell's tendencies stayed stripped throughout. The question: can the right instruction in the thinking field replace load-bearing tendencies?

### R3b — passive review

> "FIRST review your voice, drives, and constraints — then reason about the situation."

Third-person: 61.4% — **full recovery**, back to baseline. One line fixed the speech-pattern override.

Optimism: 13.6% — improved but still 10pp below baseline. The model can review "speak in third person" and apply it mechanically. But "be optimistic when things go wrong" isn't something you can review and apply — it requires being in a certain emotional state during generation.

### R3c — show your workings

> "State: MY ACCENT: [X]. MY SPEECH PATTERN: [X]. MY STRONGEST DRIVE: [X]. Then reason."

Third-person: 50.0% — **worse than passive review**.

The homework analogy breaks for LLMs. A student who shows workings is forced to think through the steps. An LLM filling in structured fields is just pattern-completing. It produces "MY SPEECH PATTERN: third person" without that actually influencing what it generates next. Structured fields give the model a way to finish the review and move on. Open-ended prompts keep the review active.

### R3d — reason in character

> "Reason IN CHARACTER — think as your character would. Then check your LAST turn for drift."

Optimism: 23.1% — **near-full recovery** and the best of any instruction. The "reason in character" framing put the model in the right emotional mode to generate optimistic responses.

Third-person: 46.2% — regressed. Immersive reasoning doesn't enforce objective patterns.

### R3f — mental-model-first

> "Build your COMPLETE response plan. Step 1: Review. Step 2: Assess. Step 3: Plan. Step 4: Recheck against spec. Step 5: Check drift."

Third-person: **81.8%** — exceeded even the baseline with tendencies (61.1%). Maximum analytical control over objective patterns.

Optimism: 9.1% — **worst result**. The five-step analytical process consumed the thinking budget that emotional expression needs. The model was in "reviewing my plan" mode, not "feeling optimistic" mode.

### The fundamental tension

The data shows two cognitive modes competing within a single thinking step:

| Mode | Good for | Bad for |
|---|---|---|
| Analytical (think ABOUT the character) | Checkable patterns (third-person) | Emotional dispositions (optimism) |
| Immersive (think AS the character) | Emotional dispositions | Mechanical pattern maintenance |

Every instruction that improved one dimension degraded the other. More structure → better voice consistency, worse emotional expression. Less structure → the reverse.

### R3g — identity activation

> "Remember who you are — your voice, your way of speaking, your drives. Feel your strongest drive. Think AS your character, not ABOUT your character. If your last turn didn't sound like you, correct it now."

| Marker | Baseline | R3g |
|---|---|---|
| Planning | 16.7% | 44.4% |
| Optimism | 24.1% | 22.2% |
| Third-person | 61.1% | 61.1% |

First variant to match or exceed baseline on all three markers simultaneously.

Two sentences instead of five steps. It works because it doesn't try to be analytical OR immersive — it activates identity. "Feel your strongest drive" puts the model in the right emotional state. "Remember your voice, your way of speaking" activates presentational patterns. "Think AS, not ABOUT" prevents the mode-switch that killed one dimension in every structured variant.

The sample is small — 18 Hartwell events from a run that crashed at tick 10 (since fixed). It needs a full 300+ event validation. But the pattern is consistent with what R3d showed for optimism, and the third-person recovery matches R3b.

## The full picture

| Run | Instruction type | Optimism | Third-person |
|---|---|---|---|
| Baseline | Tendencies present | 24.1% | 61.1% |
| R3 | None | 4.3% | 13.0% |
| R3b | Passive review | 13.6% | 61.4% |
| R3c | Structured fields | 13.6% | 50.0% |
| R3d* | In-character reasoning | **23.1%** | 46.2% |
| R3f | Five-step planning | 9.1% | **81.8%** |
| R3g* | Identity activation | 22.2% | 61.1% |

*Small sample.

R3f proved the ceiling is higher than the baseline for voice consistency — 81.8% third-person is achievable. The baseline tendencies still hold the optimism record at 24.1%. R3g matches baseline on both but there's room to push further.

## What I think this means

LLMs treat the system prompt as a priority queue. Goals and constraints are at the top — actionable, needed for coherent responses. Voice, speech patterns, personality are ambient context that gets progressively less attention as the model focuses on the task.

Tendencies worked as load-bearing anchors not because they contained special information, but because they repeated the information closer to the goal-oriented part of the prompt. The metacognitive instruction does the same thing more efficiently: instead of duplicating every attribute as a tendency, one instruction forces the model to actively engage with its character sheet.

But the form of that instruction matters enormously. Structured fields create satisficing shortcuts. Five-step processes crowd out emotional expression. The optimal instruction is short, evocative, and teaches a mindset rather than a procedure: *be yourself*.

The broader principle applies beyond character design. Anywhere an LLM is given context it should apply but doesn't — domain knowledge, style guides, safety rules — the fix isn't more context. It's an instruction that forces the model to actively engage with what it already has. And that instruction should be identity-level ("this is who you are"), not process-level ("here are the steps to follow").

## What's next

The R3g instruction needs a full validation run. If it holds, the character taxonomy simplifies: drives for personality, goals for behaviour, one evocative instruction for everything else. Tendencies become redundant except for storytelling conventions that need explicit encoding.

Beyond that, the engine can reinforce character consistency by surfacing drive states in the observation — moving attributes from the static system prompt (where they're ignored) into the dynamic context (where they're attended to). And quality tiers: gold standard with full evocative instruction, cheaper tiers with passive review for scenarios where emotional fidelity matters less.

The thing I keep coming back to is how human this problem is. The model doesn't lack the information to be in character. It just gets distracted by the task at hand and forgets to be itself. The fix isn't more information. It's a reminder.
