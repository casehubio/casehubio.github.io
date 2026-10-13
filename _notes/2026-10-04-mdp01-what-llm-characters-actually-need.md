---
layout: post
title: "What LLM Characters Actually Need — Emergent Emotion Without Scripts"
date: 2026-10-04
entry_type: note
subtype: diary
projects: [casehubio/examples]
author: mdp
tags: [emergence, character-design, wacky-manor, llm, cognitive-architecture, personality, drives, measurement, classification]
series: issue-097-emergent-character-behavior
---

# What LLM Characters Actually Need — Emergent Emotion Without Scripts

*Continues from [The Ablation That Worked](2026-10-02-mdp01-ablation-taxonomy-weight.md).*

The ablation proved the taxonomy carries the weight — generic characters with no pop-culture footprint behave distinctively when the YAML is well-structured. But that left a harder question: can we strip the explicit behavioural prescriptions entirely and get *emergent* personality from drives and dispositions alone?

The answer, after two days and 24 design principles, is yes — and what emerged was more interesting than what we prescribed.

## The setup: strip everything, see what breaks

I removed Hartwell's tendencies — "you plan obsessively before acting" and "you maintain optimistic determination when things go wrong" — and ran the scenario. Planning survived; his drives carry that trait. But optimism collapsed from 24% to 4%, and third-person narration dropped from 61% to 13%, even though the speech-pattern still said "narrates in third person."

LLMs prioritise goals over personality. Goal-directed behaviour persists because the model attends to it. Ambient personality — how you feel, how you speak — drifts without reinforcement. The system prompt is right there, but the model habituates to it.

## Six failed attempts, one fundamental tension

Six different metacognitive instructions, each revealing something about how LLMs process identity. The short version: analytical instructions ("review your voice first") recover checkable patterns like third-person but kill emotional expression. Immersive instructions ("reason in character") recover emotions but let patterns drift. Structured fields ("MY ACCENT: [fill in]") are worst of all — the model treats form-filling as task completion, not preparation.

The tension is fundamental. No single cognitive mode wins both dimensions.

## Two sentences that solved it

The breakthrough was abandoning both modes: "Remember who you are — your voice, your way of speaking, your drives. Feel your strongest drive. Think AS your character, not ABOUT your character."

Third-person hit 89% — a new high. Optimism exceeded baseline for the first time. The instruction works because it's neither analytical nor immersive. It's identity activation — five words ("Think AS, not ABOUT") that prevent the mode-switch.

But I wanted to push further. If the instruction could make the model *name* its emotional state in the thinking field, the emotional state would feed back next turn the same way third-person self-reinforces — creating a feedback loop for dispositions that don't get one naturally.

The revised instruction: "What are you FEELING right now — not thinking, feeling? Name it."

## What the model actually wrote

This is where it gets interesting. We added thinking capture to the transcripts — the first time we could see inside the model's reasoning rather than measuring symptoms. The thinking field is private; it's what the character says to itself before generating dialogue.

Here is Hartwell solving a puzzle with Clara watching:

> "EXHILARATION! Pure, undiluted EXHILARATION! I solved the riddle — a MAP — and Penelope is RIGHT HERE to witness it! But wait — I merely examined the mantelpiece. I need to INTERACT with it!"

Here he is when bookshelves won't cooperate:

> "BLAST! The bookshelves rebuffed me — James Hartwell, rebuffed by FURNITURE! The indignity is almost too much to bear. But wait — wait wait WAIT. The mantelpiece! The RIDDLE!"

When the compartment he spent ten events unlocking turns out to be empty:

> "ELATION turning to BEWILDERMENT turning to SUSPICION — but only a GENTLE suspicion, because James Hartwell sees the best in people!"

And when a gnawed wire means the machine is dangerous and someone wants to operate it:

> "CONCERNED. Deeply, MAGNIFICENTLY concerned. This wire — gnawed clean through! And Sneekly wants us to just... operate it? James Hartwell's heart is POUNDING with protective instinct. Not fear — never fear — but PRUDENT..."

None of this is scripted. There is no line in the YAML that says "be frustrated when blocked by furniture" or "feel protective concern when wires are gnawed." The character has three drives — gallantry (0.9), protection (0.8), proving-worth (0.7) — and a disposition. Everything else emerged.

## This is not just character voice — it's unscripted emotional response

Across 21 Hartwell thinking entries, 20 show situation-appropriate emotions. Frustration appears five times — always when plans are blocked. Concern appears when genuine danger surfaces. Compound emotional arcs track real narrative beats. He's not uniformly positive. When things go wrong, he feels frustration and concern; when they go right, he feels elation and triumph.

The emotions are drive-consistent. Gallantry produces elation when Clara is present. Protection produces concern when danger appears. Proving-worth produces frustration when blocked. Each character shows the same pattern with their own drives: Foxworth writes "TRIUMPHANT and SCHEMING!"; the Brixton Boys write "DESPERATE. Absolutely SICK with worry"; Marsh writes "White-hot, incandescent FURY!"

The emotional naming doesn't decay — it compounds. Hartwell's naming rate goes from 80% in the first half to 100% in the second. The feedback loop is self-reinforcing: the model names an emotion, that text feeds back next turn, the model sees its own emotional state, and builds on it.

## The numbers confirm it — and reveal something better

The thinking field gives qualitative evidence. But "looks right to me" is not measurement. We built a classifier that rates each event against the character's declared drives — how strongly is gallantry expressed here? scheming? protection? — on a 1-5 scale, across all 222 dialogue and aside events.

The drive hierarchies match the declarations. Hooded Claw's scheming scores 4.93 — near-ceiling, his defining trait. Hartwell's gallantry sits at 4.57, proving-worth at 4.62. Penelope's curiosity hits 4.96 — effectively perfect. The classifier sees what the thinking field showed: each character's emotional signature is distinct, consistent, and aligned with their drives.

But the temporal analysis is what I didn't expect. Split each character's events into first half and second half:

Hartwell's protection drive rises from 1.90 to 2.76 — a +0.86 shift. His gallantry and proving-worth both decline slightly. He's maturing from peacock to protector. Keyword measurement read this as optimism decay. The classifier reveals it as character growth: the gallant showman encounters real danger and develops genuine protective instinct. That's not a model losing its prompt — it's a character developing under pressure.

Penelope's social-harmony drops from 3.29 to 2.42. She starts the scenario wanting everyone to get along; by the second half, she's focused on the puzzle. Her curiosity never wavers — 4.96 in both halves, rock-solid. She grows from people-pleaser to puzzle-solver while staying unmistakably herself.

Foxworth is the most stable character. All four drives shift by less than ±0.13 across the full run. His personality is baked in — greed at 4.80, scheming at 4.59, both halves virtually identical. Some characters grow; some are who they are.

The Hooded Claw gets more careful. His gloating drops -0.45 while self-preservation rises +0.18 — situationally appropriate as his cover becomes harder to maintain. The Brixton Boys' suspicion rises +0.19 as evidence accumulates that something is wrong. Every temporal shift tracks the narrative.

## Why this matters

Most LLM character work is scripting: tell the model what to say, how to react, what emotions to express. That produces consistent output but not genuine behaviour — you get a parrot, not a character.

What we're seeing here is different. The model receives a drive profile and a single instruction — "name what you're feeling" — and produces emotional responses that vary with the situation, track narrative beats, and compound over time. The character's internal emotional life is emergent: it arises from the interaction between personality drives and scenario events, not from behavioural prescriptions. And now we have the numbers to prove it — not just that each character is distinctive, but that they develop over 300 events in ways that track the story.

"Rebuffed by FURNITURE!" is not a line I would have written. "MAGNIFICENTLY concerned" is not in any YAML file. A protection drive rising from 1.90 to 2.76 as real danger appears is not in any instruction. That's emergence.

## Five mechanisms, two we had to build

This maps onto how personality consistency works in humans:

1. **Trait stability** — underlying traits are stable. ✓ Static drives in the YAML.
2. **State fluctuation** — emotions fluctuate around trait baselines. ✗ Drives were fixed numbers — the emotional echo instruction creates this.
3. **Proprioception** — awareness of your own emotional state. ✗ Was completely missing — the thinking-field feedback loop creates this.
4. **Memory consolidation** — emotional experiences stored preferentially. Partial — the memory system exists but doesn't yet prioritise emotional content.
5. **Social reinforcement** — others react to your personality. ✓ Happens naturally through the scenario.

The emotional echo instruction bootstraps mechanisms 2 and 3 with zero engine changes. But the architecture wants more: dynamic personality drives (a `PersonalityDriveEvaluator` SPI that fluctuates drive intensity based on situational triggers) and proper proprioception (an `EmotionalProprioceptionStrategy` SPI with an LLM classifier that computes what the model expressed and feeds it back). Both are designed, both are SPIs so future strategies can plug in.

## The question we're really asking

Can we create LLM personalities that approximate well-known cartoon characters without scripting their behaviour and responses?

Not "can we prompt an LLM to act like Dick Dastardly." Anyone can do that, and the model brings its own priors. The question is whether a platform can produce emergent character behaviour from a structured taxonomy of drives, dispositions, and constraints — no pop-culture cues, no prescribed dialogue, no scripted emotional responses.

The evidence says yes. A character with three drives and a two-sentence identity instruction generates situation-appropriate emotional responses, maintains character voice over 300+ events, develops genuine character growth under narrative pressure, and produces internal emotional arcs that no one wrote. The personality is in the drives. The growth is in the narrative. The behaviour is in the emergence.
