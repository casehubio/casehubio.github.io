---
layout: post
title: "Why LLM Villains Turn Nice — and What Cognitive Psychology Says About Fixing It"
date: 2026-10-07
entry_type: note
subtype: diary
projects: [casehubio/examples]
author: mdp
tags: [emergence, psychopathy, cluster-b, ampd, schema-therapy, agreeableness-drift, pad, appraisal, wacky-manor]
series: issue-097-emergent-character-behavior
---

# Why LLM Villains Turn Nice — and What Cognitive Psychology Says About Fixing It

*Continues from [The Subconscious Call](2026-10-05-mdp01-sub-llm-appraisal.md).*

We had the appraisal sub-LLM wired up and character drives flowing from the MindMap into the appraisal context. Time to see what actually happens when the system runs. We built a cognitive snapshot recorder — dumps mood PAD, system drives, character drives, and goals to JSONL at configurable intervals. Then started a GENERIC profile run (no pop-culture character priors — generic names, stripped drive descriptions) with appraisal enabled and snapshots every five ticks.

1227 events. 60 ticks. 300 cognitive snapshots across five characters. What the data showed was not what I expected.

## Watching the mood saturate in real time

At tick 5, the first snapshot landed. Penelope was slightly positive (pleasure 0.083) — normal early-scenario curiosity. The Hooded Claw was at zero across all PAD axes — his appraisal hadn't fired yet. Dick Dastardly, Peter Perfect, and the Ant Hill Mob were all at init. Nothing remarkable.

By tick 10, things got interesting:

| Character | Pleasure | Arousal | Dominance | What happened |
|---|---|---|---|---|
| Hooded Claw | **0.606** | **0.303** | **0.151** | Massive positive surge — schemes going well |
| Dick Dastardly | -0.092 | 0.086 | -0.062 | Slightly frustrated — something went wrong |
| Penelope | 0.083 | 0.042 | 0.021 | Stable, gently positive |
| Peter Perfect | 0.0 | 0.0 | 0.0 | Still at init — appraisal never fired |
| Ant Hill Mob | 0.0 | 0.0 | 0.0 | Still at init — appraisal never fired |

The Hooded Claw jumped from zero to 0.606 pleasure in five ticks. That felt fast. I wanted to see if it would level off.

It didn't. By tick 15, pleasure was 0.996. By tick 20, all three PAD axes were at 0.995. By tick 25, they locked at 0.998 — and stayed there for the remaining 275 ticks. Three hundred snapshots of data, and the villain spent 285 of them in maximum positive mood. The system drives stabilised early too: composite motivation rose from 0.25 to 0.67 by tick 20 and flatlined.

Meanwhile, Dick Dastardly went the other direction. By tick 20: pleasure at -0.802, dominance at -0.995. Something thoroughly frustrated him. The delta from tick 10 to 20 was dramatic — pleasure dropped 0.6, dominance dropped 0.74 in a single interval. The appraisal was clearly detecting negative situations for him and responding appropriately.

And all three PAD axes on the Hooded Claw became *identical*. Pleasure ≈ arousal ≈ dominance ≈ 0.998 for 275 straight ticks. That's not an emotional state — it's a flatline. A real character would have different values across axes. The uniformity suggests the appraisal is producing undifferentiated "everything is great" signals with no personality filtering.

## What worked

The cognitive snapshot recorder itself proved its worth immediately. Without per-tick visibility into the mood trajectory, the agreeableness drift just looks like "the villain got nicer over time" — vague and unmeasurable. With snapshots, we could see the exact mechanism: mood saturation at tick 15, permanent lock at tick 25, zero recovery across 285 subsequent ticks.

The appraisal sub-LLM also worked — for two of the five characters. It correctly registered that Dastardly's situation was deteriorating and produced negative mood signals. It correctly registered that the Hooded Claw's schemes were succeeding and produced positive signals. The machinery is functional. The problem is upstream.

Character drives are visible in every snapshot — scheming 0.9, gloating 0.7, dominance 0.6 for the Hooded Claw throughout. The drives never change. The MindMap seeder puts them in; they stay static. This is a separate problem: drive adaptation should be dynamic, responding to what happens during the scenario.

## What didn't work — and the hypotheses

**Problem 1: Mood saturation.** The Hooded Claw maxes out positive and stays there. The appraisal correctly evaluates "situation is good for me" but cannot distinguish *what kind* of good. Successful deception and genuine friendship both register as high conduciveness, high controllability, positive mood shift. **Hypothesis:** The system lacks an affective personality dimension. Without it, every positive situation maps to the same undifferentiated "joy" emotion, which maps to the same PAD shift. A psychopathic character needs a different mapping: high conduciveness + high controllability + high callousness → predatory satisfaction (dominance-based), not warmth (affiliation-based).

**Problem 2: Peter Perfect and Ant Hill Mob never trigger.** Their appraisal sat at zero for all 300 ticks. **Hypothesis:** They have no emotional backstory to create salience. Peter's gallantry drive is 0.9 — but it's just a number. Without a seeded memory of *why* he feels protective of Clara ("the first time I saw her, something caught"), the appraisal has nothing to work with. The drive exists but carries no emotional charge. Same for the Mob's protection drive — "loyalty: 0.9" is meaningless without "we raised her from childhood."

**Problem 3: The LLM normalises.** Even when the mood trajectory was reasonable (Dastardly's frustration, Penelope's gentle positivity), the character behaviours still drifted prosocial over extended runs. **Hypothesis:** The architecture models what a character *thinks* and *wants* but not how they *view other people*. Trust scores are behavioural. Familiarity is a count. There's no representation of "I see this person as prey" or "her trust makes me feel powerful." Without an explicit relational model, the LLM fills the gap with its training prior — be warm, cooperate, help.

## The psychopathic inversion

This is where the investigation got interesting. For someone with high callousness and manipulativeness, the normal emotional response to kindness is *inverted*:

| Event | Normal response | Psychopathic response |
|---|---|---|
| Someone trusts you | Reciprocate → connection builds | Classify as naive → dominance opportunity |
| Kindness received | Gratitude → warmth | Contempt → power pleasure |
| Deception succeeds | Guilt | Exhilaration → escalate |
| Being caught | Shame, repair attempt | Narcissistic rage, retaliate |
| Someone pulls away | Longing, desire to reconnect | Indifference — or intensify if still useful |

The nicer people are to a psychopath, the more they see them as weak, naive, prey. The pleasure IS real — but it comes from the power asymmetry. Each successful deception creates a dopamine hit that reinforces the predatory pattern. The mood *should* saturate positive. The problem isn't the saturation — it's that the system has no way to route that positive mood through a personality that derives satisfaction from dominance rather than connection.

## The missing dimension maps to known psychology

The gap sits in territory that clinical psychology mapped decades ago. We catalogued the relevant models:

The PCL-R four-factor model (Hare, 2003) identifies Interpersonal, Affective, Lifestyle, and Antisocial facets. Our architecture captures Lifestyle and Antisocial well. We partially capture Interpersonal. We completely miss the **Affective** facet — callousness, shallow affect, lack of empathy, no remorse. That's the critical gap.

The DSM-5 Alternative Model (AMPD) provides 25 validated trait facets across five domains — maladaptive variants of the Big Five. The ones we need: Callousness, Manipulativeness, Suspiciousness, Intimacy Avoidance, Grandiosity. These are dimensional and measurable.

Schema Therapy (Young, 1990) maps 18 Early Maladaptive Schemas from childhood experiences to triggers to emotional responses to coping modes. The Mistrust/Abuse schema — "people who are kind always want something" — is exactly the Hooded Claw's relational pattern. But here's the key insight: these traits are the *result* of memories, not innate labels. Unlike MBTI ("you ARE this type"), Cluster B patterns are developmental ("you BECAME this way"). A child who learns "I only get attention when I'm in control" develops manipulativeness. After age seven to ten, the patterns crystallise. The memories are the cause; the traits are the effect.

## Four layers, not one

The architecture needs:

**Formation** — seeded memories from Schema Therapy's model. Childhood experiences that produced the personality. These give the LLM the *why*.

**Traits** — AMPD facets. Callousness, Manipulativeness, Grandiosity, Suspiciousness. Stable after formation. These tell the appraisal *how to interpret* each situation.

**Relational** — per-person emotional dimensions that evolve. Three dimensions from the research: attraction/repulsion (McCroskey & McCain, 1974), connection/disconnection (Sternberg, 1986), longing/indifference (attachment theory). The personality traits determine how these evolve — same event, different emotional charge depending on the character's affective capacity.

**Dynamic** — appraisal + PAD + memory formation moment-to-moment. "Clara smiled at you" gets stored as warmth by Peter and as target assessment by the Hooded Claw. The relational PAD updates per encounter — dominance increases every time Clara is kind to the Claw, building the predatory feedback loop.

## Where this goes

The controlled experiment is possible. Same characters, same scenario, same LLM. Three conditions: no cognitive architecture, current architecture (shows the drift), and full relational model with AMPD facets and personality-filtered appraisal. The snapshot data gives per-tick mood trajectories. The classifier gives behavioural quality scores. Together they can show quantitatively why the modelled villain stays villainous while the unmodelled one normalises.

The Hooded Claw is the first test case. The design lives in [neocortex#477](https://github.com/casehubio/neocortex/issues/477).

---

**References:**
Hare (2003), *Manual for the Revised Psychopathy Checklist*;
Patrick, Fowles & Krueger (2009), Triarchic conceptualisation of psychopathy;
Young (1990), *Cognitive Therapy for Personality Disorders*;
Berne (1961), *Transactional Analysis in Psychotherapy*;
DSM-5 Alternative Model for Personality Disorders (Section III);
Paulhus & Williams (2002), The Dark Triad of personality;
Kowalski, Vernon & Schermer (2021), The Dark Triad and facets of personality;
McCroskey & McCain (1974), The measurement of interpersonal attraction;
Sternberg (1986), A triangular theory of love;
Bowlby (1969), *Attachment and Loss*.
