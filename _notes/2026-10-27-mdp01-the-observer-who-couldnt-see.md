---
layout: post
title: "The Observer Who Couldn't See"
date: 2026-10-27
entry_type: note
subtype: diary
projects: [casehubio/examples]
tags: [cognitive-architecture, mood-appraisal, performance, inline-prompts, session-management]
series: issue-150-reduce-tick-time
---

# The Observer Who Couldn't See

Each tick of the Wacky Manor scenario was taking 44 seconds. The original theory blamed three sequential LLM round-trips per agent: main query, mood appraisal, BDI extraction. Five agents times three calls equals a lot of waiting.

The theory was wrong on every count.

## What the instruments showed

I instrumented the tick loop with nanosecond timing and found a different picture entirely. The query phase — five parallel character queries — consumed 71% of the tick (31 seconds). Mood appraisal took 29% (13 seconds, sequential). And BDI extraction? Zero. Every single BDI call was failing silently with `AgentSessionLimitException: Agent session limit reached (12 active sessions)`.

The session pool was full before BDI ever got a chance to run. Twelve sessions allocated at startup — five character, one narrator, five classification, one mood — and nothing left for BDI or UserModel. The cognitive system had been running with its mental model and personality inference permanently disabled. Every tick, every agent, every run. No error in the UI, no degradation signal. Just a WARN-level log line that nobody reads.

The fix was straightforward: lazy classification sessions. The five classify sessions were opened eagerly at startup but fired sporadically — only when the DegreeAccumulator triggered, which might be every few ticks. Moving to on-demand allocation freed five session slots immediately. BDI and UserModel started working for the first time.

## The separate observer problem

The mood appraisal architecture was a textbook case of the independent-observer pattern. After the character generates its response, a separate LLM call reads a 200-character truncated snippet of the exchange and rates the emotional impact on three PAD axes: pleasure, arousal, dominance. Independent assessment. No self-report bias. Clean separation of concerns.

Except the observer was blind.

Two hundred characters of truncated dialogue, stripped of the character's personality, drives, goals, memories, relationships, and the full situation context. The "independent" appraiser was guessing the emotional tone of an exchange from a sentence fragment. And it was costing 13 seconds per tick to do it — five sequential LLM calls, each taking about 2.5 seconds, because mood used a single shared session that couldn't be parallelised.

We batched the mood calls first — five parallel sessions via `executeBatch`, dropping the wall clock from 13 seconds to 3. But then the real question surfaced: what if the character just reported its own mood inline, as part of its response?

## Marking your own homework

The concern was real. Self-report bias is well-documented. Characters might systematically over-report positive affect, fail to recognise when they've been manipulated, or filter through self-image. An independent appraiser, the argument goes, is more objective.

But objectivity requires context. An observer who sees 200 characters of a multi-thousand-token conversation isn't more objective — they're more ignorant. The character, generating its response with full access to its drives, memories, appraisal state, and social context, is the only entity in the system that has enough information to appraise the emotional impact accurately.

I added a `moodImpact` field to the character response JSON — four values: pleasure, arousal, dominance, cause. The model returns them alongside dialogue, action, and thinking. If the field is missing, the system falls back to batch mood. In testing: 15 out of 15 responses returned inline mood. Zero fallbacks.

The causes were better. Not "social delight at dramatic setting" from a truncated snippet, but "protective anxiety surging, self-reproach for not staying close" — ant-hill-mob's mood after Penelope left with the suspicious butler while it was distracted. The inline version knew the relationship, knew the suspicion, knew the self-assigned protective role. The batch version had none of that.

## The numbers

| Phase | Baseline | After |
|-------|----------|-------|
| Mood appraisal | 13s (sequential) | **0ms** (inline — free) |
| RecordInteraction | 13,000ms | **0ms** |
| Sessions needed | 12 (starved) | **6** |
| BDI success rate | 0% | **100%** |
| **Tick total** | **44s** | **~31s** |

The remaining 31 seconds is pure query API latency — five parallel Opus 4.6 calls. Application overhead is zero. Further reduction is a model selection question, not an architecture question.

## The lesson

When you separate an assessment into its own call for "independence," check what context the independent assessor actually has. If the answer is "a truncated fragment," the independence is theatre. The assessor isn't unbiased — it's uninformed.

The principle generalises: an independent assessor needs at least as much context as the entity being assessed. Otherwise you're paying for a worse answer and calling it objectivity.

There's still a valid concern about self-report bias in the inline approach. I filed #152 to investigate — run the same scenario with both approaches using full-context separate appraisal (not the truncated version) and compare PAD distributions. But the truncated-input separate call was never the right baseline for that comparison. It was broken by design.
