---
layout: post
title: "Personality Layers That Don't Stack: What 90 LLM Judge Calls Taught Us About Descriptor Composition"
date: 2026-07-31
type: phase-update
entry_type: note
subtype: diary
projects: [wacky-manor]
tags: [eidos, personality-composition, mbti, belbin, jungian, llm-evaluation]
series: issue-2-staged-layer-comparison
---

*Part of a series on [#2 — staged personality layer comparison](https://github.com/casehubio/examples/issues/2). Previous: [The Hooded Claw Schemes Perfectly and Does Nothing](2026-07-28-mdp01-the-hooded-claw-schemes-perfectly.md).*

## The Question

Phase 2.5 gave us autonomous LLM characters who could actually drive a plot — the Hooded Claw finds the poison, uses it on the tea, POISONED in 6 turns. But those characters ran on flat string dispositions: `socialOrient: competitive`, `ruleFollowing: flexible`. Flat labels, no structure.

Eidos supports richer personality frameworks — Jungian cognitive functions with MBTI type shorthand, Belbin team roles with vocabulary-grounded slot descriptions. The question for Phase 2.5b: does adding these frameworks produce measurably different character behaviour, or is the briefing text doing all the work?

## The Experiment

Four descriptor variants, identical except for personality structure:

- **Baseline** — flat string dispositions, no vocabulary grounding
- **Jungian** — `mbtiType: ENTJ` with `dispositionVocabulary: urn:casehub:vocab:jungian`, axes auto-derived
- **Belbin** — flat dispositions retained, plus `slotVocabulary: urn:casehub:vocab:belbin` with role assignments
- **Composite** — Jungian + Belbin together

We ran two kinds of evaluation. First, `MbtiAlignmentJudge` — does the rendered system prompt present as the declared MBTI type? This is a precondition: if the prompt doesn't read as ENTJ, you can't attribute behavioural differences to the Jungian framework. Second, `FunctionActivationJudge` — does the character activate the expected cognitive function (Te, Fe, Se, etc.) when given a controlled scenario?

## MBTI Alignment — the Briefing Wins

The first run showed 8 out of 10 profiles aligned. The two failures were both on the J/P dimension, and both traced to the same root cause: the briefing text contradicted the framework.

**Peter Perfect (ENFJ)** scored 33% on J/P — across both Jungian and Composite profiles. His briefing said: *"You volunteer for every dangerous task. You narrate your own heroism in the third person when excited."* That reads as spontaneous, reactive, impulsive. Pure Perceiving. The MBTI framework said Judging, but the briefing was louder.

The fix: rewrite the briefing to emphasise planned, methodical gallantry — *"You always have a PLAN for every situation — you prepare, assess the danger methodically, and then execute your gallant rescue in careful steps."* Same character, same comedic energy, but the language now carries structured completion rather than impulsive reaction. J/P went to 100%.

**Ant Hill Mob (ISFP)** was more interesting. In the Jungian-only layer, J/P scored 100% — correctly perceived as Perceiving. In the Composite layer (Jungian + Belbin), J/P dropped to 0%. The Belbin role was Implementer: *"Reliable, efficient, turns ideas into practical actions, methodical."* Every word in that description codes as Judging. The Jungian framework said flexible and adaptive; the Belbin role said systematic and reliable. The judge saw the contradiction and sided with Belbin.

This is actually the most important finding in the experiment: **Belbin produced measurable signal.** It shifted the J/P perception from aligned to misaligned. The frameworks aren't invisible — they're compositionally dangerous. The fix was changing the Belbin role from Implementer to Specialist (*"dedicated, self-starting, provides specialist knowledge"*), which doesn't carry J/P connotations.

After both fixes, all 10 profiles aligned.

## Function Activation — Ne Everywhere

The function activation results were less encouraging. Target Activation Accuracy across all 20 character-profile combinations:

| Character | Baseline | Jungian | Belbin | Composite |
|---|---|---|---|---|
| Penelope (ESFJ) | 0.50 | 0.50 | 0.50 | 1.00 |
| Hooded Claw (ENTJ) | 0.50 | 0.00 | 0.00 | 0.50 |
| Ant Hill Mob (ISFP) | 0.50 | 0.50 | 0.50 | 0.50 |
| Dastardly (ESTP) | 0.00 | 0.00 | 0.00 | 0.00 |
| Peter Perfect (ENFJ) | 0.50 | 0.50 | 0.50 | 0.00 |

The Jungian and Belbin layers didn't improve function activation over baseline. In several cases they made it worse. The personality frameworks aren't reaching the LLM's generation process in a way that shifts which cognitive function gets activated under controlled scenarios.

The dominant pattern: the judge identified Ne (Extraverted Intuition) as the activated function for almost every scheming, planning, or creative scenario — regardless of the character's declared dominant function. The Hooded Claw is typed as ENTJ (dominant Te — systematic strategic planning), but his responses are a fountain of divergent, associative possibilities: *"trapdoors triggered by pressure plates connected to pulley systems activating cages suspended by ropes threaded through SEVENTEEN separate mechanisms."* That's textbook Ne, not Te.

I think this is a Claude generation artifact. The model's default creative generation style — associative, divergent, possibility-exploring — maps to Ne in the Jungian framework. When you ask a character to scheme, the LLM generates the way it generates. The personality framework describes what the character *should* do, but the model's statistical tendencies produce what it *does* do. The briefing text carries the character voice (Southern drawl, villain monologues, gangster speech) because voice is surface-level pattern matching. Cognitive function activation requires a deeper behavioural shift that prompt engineering alone may not achieve.

Dastardly at 0.00 across all layers is the starkest example. He's typed as ESTP (dominant Se — concrete, present-moment, action-first), but every response reads as Ne brainstorming. The character's theatrical personality fights the framework's intended grounding.

## What This Means for Descriptor Tuning

Three things I'll carry forward:

**Briefing text is the strongest lever.** Every MBTI alignment failure traced to briefing language, not framework configuration. When the briefing and the framework agree, alignment is perfect. When they disagree, the briefing wins. Implication for tuning: check your briefing against your MBTI type letter by letter. If the briefing says "impulsive" and the type says J, one of them needs to change.

**Composition can create contradictions that aren't visible at the individual layer level.** The Ant Hill Mob's J/P contradiction only appeared in Composite — Jungian alone was fine, Belbin alone was fine, but together they fought. This is exactly the kind of collision the Phase 2.5b experiment was designed to surface. Any serious personality composition system needs a cross-vocabulary compatibility check — ideally automated, run before the descriptor ships.

**LLM generation style has its own personality.** The Ne dominance isn't a framework failure — it's a property of the generation model. Different LLMs may have different default cognitive styles. A model trained primarily on academic text might default to Ti (analytical frameworks). A model trained on fiction might default to Ne (associative exploration). If this holds, personality framework effectiveness would vary by model, and fine-tuning or structured decoding constraints might be needed to make cognitive function activation reliable.

The open question is whether the autonomous scenario runs — 180 turns of gameplay rather than controlled 2-call evaluation scenarios — will show behavioural differences that these isolated judge calls missed. Autonomous behaviour emerges from sequences of decisions, not single responses. The experiment harness is ready; the next step is running all 12 scenarios and reading the transcripts.
