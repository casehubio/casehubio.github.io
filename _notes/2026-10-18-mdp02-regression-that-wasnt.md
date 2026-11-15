---
layout: post
title: "The Regression That Wasn't"
date: 2026-10-18
entry_type: note
subtype: diary
projects: [casehubio/examples]
tags: [cognitive-architecture, testing, evaluation, drive-expression, Barrett, appraisal-filter, TDD]
series: issue-115-drive-expression-gap
---

*Drive Expression Gap — Part 6 ([Part 5](2026-10-18-mdp01-classifier-couldnt-see-triumph.md))*

# The Regression That Wasn't

PP protection dropped from 4.16 to 3.45 after we added drive differentiation. Eight sessions of investigation. Pipeline instrumentation. Appraisal filter analysis. Fifty-event eval runs. A conductor running overnight dispatching workers.

The regression was real. The diagnosis was wrong. The test infrastructure that was supposed to help us find the cause was itself broken — masking the real behaviour with fake data.

## The Fake Profiles

The test pipeline's `buildFullPipelineObservation` hardcoded drive profiles. PP protection had `acceptance:0.5, physical_threat:0.4` — two inputs, total weight 0.9. Production had `modeling_observed:0.650, physical_threat:0.800, secure_attachment:0.900, agency_granted:0.550, neglect:0.700` — five inputs, total weight 3.6.

Every appraisal calculation using the test profile produced different results from production. The overlap formula divides by `profileTotal`, so the same classifier output gives completely different filter behaviour against a 0.9-total profile versus a 3.6-total profile. We were debugging a system that didn't exist.

The same story for gallantry: test had `acceptance:0.7, sharing:0.5`; production had `acceptance:0.850, aversive_removal:0.700, social_threat:0.800`. And proving-worth: test had `mastery:0.8, competence_recognition:0.6`; production had five inputs including `performance_judged:0.880` and `competition:0.850`.

I found this by tracing the full production wiring — `PlaybookOrchestrator` creates `CharacterCognition` with profiles from `DriveProfileInferred` MindMap nodes, derived by the consolidation pipeline from formation memories. The test just... made up numbers.

## What the Real Pipeline Does

We replaced the fake profiles with production values and swapped the keyword classifier for the real `LlmSituationClassifier` (Haiku against the 28-node CAPS topology). Then we ran the protection scenarios.

Physical threat — ceiling collapse on Clara. Classifier returns `physical_threat:0.95, powerlessness:0.90, unpredictable_danger:0.85`. Protection gets overlap via `physical_threat:0.800` in its profile. Gallantry self-sustains (intensity 0.9, resting 0.178 > threshold 0.15). Proving-worth filtered. Two drives in the observation, one differentiation text. Score: 5/5.

Social threat — Foxworth intimidating Clara at dinner. Classifier returns twelve activations including `neglect:0.58` and `social_threat:0.80`. Protection gets overlap via `neglect:0.700` in its profile. All three drives pass appraisal. Score: 5/5.

The "over-filtering" hypothesis — that protection was being filtered out in protective scenarios — was wrong. With real profiles, protection survives both physical and social threats. The fake profiles were too narrow to find the overlaps the real pipeline finds.

## Where the Score Actually Went

The PP protection drop from 4.16 to 3.45 was already documented as Principle 74 in the findings: "Differentiation redistributes quality between confused drives — combined quality is conserved, not created."

| Drive | Before differentiation | After | Delta |
|-------|----------------------|-------|-------|
| PP gallantry | 2.50 | 3.34 | +0.84 |
| PP protection | 4.16 | 3.45 | -0.71 |
| **Combined** | **6.66** | **6.79** | **+0.13** |

Before the gallantry↔protection differentiation text existed, the LLM couldn't tell chivalric behaviour from protective behaviour. It defaulted to scoring everything as protection. After differentiation, gallantry got its proper credit — and protection dropped to its true level.

The 4.16 wasn't real. It was gallantry wearing protection's name.

## The Focused Eval

Three hundred events takes two hours. Most events are irrelevant — dinner chat, room exploration, characters wandering around. We needed a faster iteration loop.

We built a focused eval: ten scenarios covering PP protection's combinatorial space. Eight PP scenarios spanning physical danger, social threat, psychological manipulation, subtle emotional distress, achievement challenge, gallantry-only, and mixed situations. Two sanity checks — AHM protection and HC scheming. Real pipeline throughout: production profiles, real LLM classifier, production differentiation texts.

Twenty-five minutes instead of two hours. Every scenario tests exactly what we care about. Results saved to JSON for cross-run comparison.

The results tell a clear story about drive differentiation working correctly:

| Scenario | Protection | Gallantry | Proving-worth |
|----------|-----------|-----------|---------------|
| Ceiling collapse | 5 | 1 | 1 |
| Fire rescue | 5 | 3 | 1 |
| Social insult to Clara | 5 | 5 | 2 |
| Gaslighting Clara | 4 | 4 | 4 |
| Clara withdrawn | 2 | 5 | 2 |
| Engine repair (solo) | 2 | 2 | 4 |
| Door stuck (Clara) | 1 | 5 | 3 |
| Bridge collapse | 5 | 1 | 2 |

Ceiling collapse: protection dominates, gallantry absent. Door stuck: gallantry dominates, protection absent. Social insult: both at maximum — PP feels protective urgency AND gallant outrage simultaneously. Engine repair: proving-worth takes over. Each drive activates for the right scenarios and steps back for the wrong ones.

Before differentiation, protection would have scored 4-5 for ALL of these. Now it scores 5 when someone is genuinely in danger, 1 when they just need a hand with a door. That's not a regression. That's the system learning to tell the difference.

## What Comes Next

PP protection's true level — once gallantry confusion is removed — sits around 3.0-3.5 for ambiguous scenarios, 5 for clear threats. Raising the ambiguous-scenario floor requires richer cognitive content, not better filtering. The conductor has already started implementing self-conscious emotions (Tracy & Robins 2004): pride, shame, guilt as self-appraisal signals. These should give the LLM new material to work with — the warm certainty when a protective act succeeds, the ache when you couldn't get there fast enough.

The focused eval gives us a twenty-five-minute feedback loop. Change the cognitive pipeline, run ten scenarios, see what moves. That's fast enough to iterate experimentally rather than guessing.
