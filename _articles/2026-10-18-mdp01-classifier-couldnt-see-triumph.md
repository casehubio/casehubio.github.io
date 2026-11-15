---
layout: post
title: "When the Classifier Couldn't See Triumph"
date: 2026-10-18
entry_type: article
subtype: diary
projects: [casehubio/examples]
tags: [cognitive-architecture, CAPS, situation-classification, Barrett, Mischel-Shoda, emotional-granularity, drive-expression, evaluation]
series: issue-115-drive-expression-gap
---

*Drive Expression Gap — Part 5 ([Part 4](2026-10-17-mdp01-teaching-llm-what-makes-drive-unique.md))*

# When the Classifier Couldn't See Triumph

"The trap worked. The cellar door is sealed. Nobody heard a thing."

The CAPS situation classifier looked at this and returned: `agency_granted:0.85, choice_available:0.75, aversive_removal:0.70, abundance:0.65`. Agency. Choice. Relief. Abundance.

No `reward`. No triumph. No victory signal.

This is the Hooded Claw's defining gloating moment — a trap sprung, a plan fulfilled — and the classifier reads it as an autonomy event. The gloating drive's profile includes `reward:0.850` as its top node. Zero overlap. Drive filtered. The character walks away from his moment of triumph with no gloating expression at all.

## The Encoding Gap

The classifier prompt listed each CAPS node like this:

```
- reward (consequence) — vocabulary: triumph, rewarded, victory, prize, success, earned, won
```

The vocabulary contains "triumph" and "success." So why didn't the classifier match "The trap worked" to those words?

Because the classifier isn't doing keyword matching. We moved to an LLM classifier specifically because keyword matching couldn't capture semantic context. But the LLM was doing something almost as shallow: scanning a flat vocabulary list and pattern-matching against the situation text. "The trap worked" doesn't contain "triumph" or "success" or "rewarded." It contains "trap" and "worked" — words that read as agency and control, not as reward.

The vocabulary gives surface forms. What the LLM needed was the construct's meaning. Not "these are example words for reward" but "this is what reward IS as a psychological construct."

Mischel and Shoda (1995) describe the encoding step of CAPS — mapping a raw situation to the network's input feature space. The encoding is concept-driven. The person doesn't pattern-match keywords; they recognise that a situation instantiates a psychological construct. "The trap worked" is a successful plan, which is a positive outcome, which is reward. That chain of inference requires knowing what reward MEANS, not just what it sounds like.

Barrett (2017) makes the same point from the constructed emotion side: categorisation requires a concept. A feature list is necessary but not sufficient. The LLM had features without the concept.

## The Fix

We added a `description` field to every input node in the CAPS topology:

```
- reward (consequence: positive outcome — a plan succeeding, a goal achieved,
  something desired obtained or won) — vocabulary: triumph, rewarded, victory, ...
```

The vocabulary stays — it gives the LLM example surface forms. The description gives the conceptual bridge. Now "The trap worked" maps through the description: "plan succeeding" matches directly. The classifier returns `reward:0.85`.

The gloating drive's profile has `reward:0.850`. Overlap is now 0.484. Combined with CAPS resting activation (0.108 for intensity 0.7), the effective activation is 0.592 — well above the 0.15 threshold. Gloating renders.

In the 300-event eval, Hooded Claw gloating went from 2.37 to 3.56 quality-when-triggered. The classifier fix didn't change what gloating says — it changed whether gloating gets to speak at all.

## The Bug Hunt

With the classifier fixed, I expected the pipeline test to go from 7/8 to 8/8. It did. But we also discovered something unexpected when we expanded the test suite to include Penelope's drives.

The cognitive architecture already generates full multi-channel descriptions at consolidation time — somatic, attentional, temporal, and action tendency channels, derived from formation memories via LLM. The `DriveDescriptionGenerationPhase` was producing rich, vivid descriptions for every character. These generated descriptions automatically replaced the hand-authored ones in the prompt.

Except for Penelope. Her cognitive dump showed zero generated descriptions. Every other character had three or four.

I set Claude loose on this as an autonomous investigation — our first attempt at a conductor-style research session where Claude traces bugs independently. The root cause was a delete-before-write pattern in the description generation phase. Each consolidation tick deleted all existing descriptions before regenerating them. If the tick was interrupted between delete and generate — which it was, when the server shut down during the second tick — the descriptions from the first tick were permanently lost. HashMap iteration order determined which characters survived. Penelope lost.

The conductor also found three other rendering gaps. A `findFirst()` call was dropping all but one differentiation text per character. Template metadata (`{{serves: social-harmony|strong:`) was leaking raw into the prompt. And the competition detection only generated differentiation for drive pairs with shared CAPS mediating nodes — which meant 11 of 21 possible pairs had no differentiation at all, including the gallantry/protection pair that Peter Perfect needed most.

## Gallantry Finds Its Voice

With all the rendering gaps fixed, we ran the eval again. Peter Perfect's gallantry went from 2.50 to 3.34 — the biggest single improvement we'd measured.

The key fix was the differentiation system finally reaching the prompt. Before the bug fixes, the LLM had no way to distinguish gallantry from protection. Both are "helping someone." Without differentiation text explaining "one pull is grace and devotion — the gesture IS the feeling; the other is urgency and shielding — closing the gap before it becomes a cost," the LLM defaulted to protection-like behaviour in every helping situation. Protection's quality score was inflated at 4.16; gallantry's was suppressed at 2.50.

After differentiation reached the prompt, the quality redistributed: gallantry up 0.84, protection down 0.71. Combined quality conserved: 6.66 before, 6.79 after.

And that conservation is the finding.

## The Ceiling

Pipeline improvements — encoding, profiling, filtering, differentiation — redistribute quality between drives. They don't create new quality.

Two eval runs confirmed it. The classifier fix moved quality from "not rendering" to "rendering" (gloating). The differentiation fix moved quality from "protection getting credit for gallantry" to "both drives correctly attributed." Each fix was real progress. Each fix was also zero-sum within its local scope.

The overall mean after all pipeline work: 3.57. Seventeen drives measured. The top (scheming at 4.79) and the bottom (adventure at 2.78) are 2.01 apart. Pipeline tuning can narrow that gap by correctly allocating quality to the right drives. It can't widen the total envelope.

To generate more total quality — richer, more distinctive expression — the LLM needs richer cognitive material in the prompt. The current prompt has drive descriptions, differentiation texts, and situation-primed formation memories. These determine WHETHER a drive renders and HOW it's differentiated. What's missing is cognitive depth: the self-conscious emotions that distinguish gallant pride from protective urgency. The theory of mind that lets a character sense what someone needs before they say it. The emotion regulation that turns raw self-preservation into composure.

Barrett's work on emotional complexity says it comes from two components: differentiation AND integration. We've built the differentiation system. Integration — understanding how drives work together as a coherent personality — is what's next. The pipeline is the plumbing. Cognitive capability is the water.

## What's Next

The architecture roadmap has 14 literature-backed epics. Three are Tier 1:

Self-conscious emotions (Tracy & Robins 2004) would enrich the bottom-scoring drives directly. Gallantry isn't just "helping" — it's "pride in being the one who helps." Gloating isn't just "succeeding" — it's "pride layered with contempt." Dominance isn't just "controlling" — it's "the quiet certainty that you're the one who should be deciding." These are appraisals of the self in relation to standards — a cognitive dimension the current pipeline doesn't have.

Theory of Mind (Baker, Saxe & Tenenbaum 2017) would let characters model what others are thinking and feeling. Right now, Peter Perfect's gallantry fires when Clara is struggling with a door. It doesn't fire when Clara is quietly worried but hasn't said anything — because the situation classifier can't see Clara's internal state.

Emotion regulation (Gross 1998) would give characters the ability to manage competing drives. The Hooded Claw doesn't just feel self-preservation — he feels it and then masters it, producing the cold composure that makes him dangerous.

Each of these adds a new cognitive section to the prompt. Each gives the LLM material to express drives more distinctively. The measurement target is clear: overall mean above 3.57, bottom drives above 3.00, no drive dropping more than 0.4.

But before any of that, there's a regression to reverse. The all-pairs differentiation that helped gallantry may have fragmented drives that were already integrated. AHM's protection dropped from 4.71 to 4.26 — and AHM went from zero differentiation to three differentiation texts in the prompt. Easterbrook (1959) on attentional narrowing suggests the fix: only show differentiation for drive pairs where both drives are currently active. When protection renders alone, it gets the full prompt attention. The differentiation is there when it's needed and absent when it isn't.

The pipeline has a ceiling, but the ceiling is higher than where we started.
