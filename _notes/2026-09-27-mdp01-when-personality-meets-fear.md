---
layout: post
title: "When Personality Meets Fear"
date: 2026-09-27
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
tags: [occ-emotions, personality, jpaf, appraisal, cognitive-functions, big-five]
series: issue-382-jpaf-appraisal-weights
---

# When Personality Meets Fear

Until today, every agent in the system felt exactly the same Fear for the same goal state. An ENTJ project manager and an ESTP firefighter, looking at the same deadline with the same urgency, produced identical emotional responses. That's wrong — and not in a handwavy "personality should matter" way. It's wrong in a way that personality psychology has quantified for decades.

The OCC emotion model generates emotions from goal appraisal. An active goal with high urgency and low feasibility produces Fear. A blocked goal produces Distress. A colleague's struggling goal produces Pity. The formulas are straightforward: multiply priority by urgency, apply a threshold, clamp to [0,1]. The problem is that the formulas are the same for every agent.

I wanted to fix this without picking numbers out of the air.

## The Two-Hop Problem

There's no paper that says "an Ni-dominant agent should have a fear onset multiplier of 0.7." The Jungian cognitive function stack (Ni, Fe, Ti, Se — the JPAF framework we use for agent personalities) and the OCC appraisal dimensions (urgency, desirability, likelihood) live in completely different research traditions that have never been directly correlated.

But there are two empirical bridges that connect them.

**Bridge one:** McCrae and Costa (1989) established that MBTI dimensions map to Big Five traits with moderate-to-strong correlations. Thinking/Feeling maps to Agreeableness at r = .44. Judging/Perceiving maps to Conscientiousness at r = .49. Sensing/Intuition maps to Openness at r = .69. These correlations have been replicated consistently for 35 years.

**Bridge two:** Big Five traits map to emotional appraisal dimensions. Haas et al. (2015) found Agreeableness predicts empathic concern at r = .33–.40. Barańczuk's 2019 meta-analysis (132 studies, N = 46,345) found Conscientiousness predicts adaptive emotional regulation. Furnham's subfacet analysis showed the Fantasy subfacet of Openness (r = .47) captures exactly the kind of anticipatory imagination that drives early fear detection.

Chain them together: cognitive function → Big Five trait → appraisal dimension. The direction and relative strength of each link comes from published correlations. The absolute modulation range comes from ALMA — the emotion model already in our codebase — which specifies personality can shift intensity by roughly ±30%.

## Where It Gets Honest

Two of the three modifiers have clean two-hop chains. The urgency weight goes through genuinely independent constructs: J/P → Conscientiousness (McCrae & Costa) → adaptive regulation (Barańczuk). Two different papers, two different things being measured.

But the relationship weight and fear onset threshold use a shortcut. The "second hop" correlation — Tendermindedness (.42) for empathy, Fantasy (.47) for anticipation — comes from Furnham's subfacet analysis of the *same* T/F and S/N correlations as the first hop. It's the same construct at a finer granularity, not an independent measurement. The relative weights between functions are real (Fe genuinely correlates more with Tendermindedness than Fi does with Trust), but calling it a "two-hop chain" overstates the independence of the evidence.

I documented this in the spec because calibration built on weak foundations eventually collapses. If someone later asks "why is Fe's relationship contribution 1.0 and Fi's only 0.665?" the answer traces back to the Tendermindedness/Trust ratio in Furnham 1996, not to someone's intuition about what felt right.

## The Calibration Trap

The first version of the conversion formula used the raw correlation products directly. Function → Big Five at r = .44, Big Five → appraisal at r = .37, product = .16. Scale by the ALMA range (±0.3). Result: modifiers like 1.002 and 0.998. Practically zero modulation.

The problem is triple attenuation. Real JPAF profiles have four functions that partially cancel (an INFJ has both Fe pushing toward J and Se pushing toward P). The weighted average dampens further. Then multiplying by 0.3 crushes it to nothing.

The fix came from working backwards. The issue specified ±30% for strongly-typed agents. A canonical ESTJ (Te = 0.40, Ne = 0.20) produces a raw urgency value of about 0.20 after cancellation. To hit ±30% at that raw value: SCALE_FACTOR = 0.30 / 0.20 = 1.5. The correlation products still determine direction and relative weight between functions — that's the empirical part. The scale factor translates "how much does personality matter" into a bounded range justified by ALMA and Mehrabian.

## Three Agents, One Deadline

The numbers become interesting when you run three distinct personalities through the same scenario.

Alice is an INFJ — dominant Ni, auxiliary Fe. Anticipatory and empathic. Her derived weights: urgencyWeight = 1.105, relationshipWeight = 1.045, **fearOnsetThreshold = 0.745**. That last number is her signature: 25% lower fear threshold from dominant Ni.

Bob is an ESTP — dominant Se, auxiliary Ti. Present-focused and analytical. His weights: **urgencyWeight = 0.805**, relationshipWeight = 0.955, **fearOnsetThreshold = 1.255**. The mirror image: 20% lower urgency sensitivity, 25% higher fear threshold.

Carol is an ENTJ — dominant Te, auxiliary Ni. Structured and decisive. Her weights: **urgencyWeight = 1.195**, **relationshipWeight = 0.831**, fearOnsetThreshold = 0.955. Strongest urgency amplification, lowest empathic response.

Now give them a long-term research goal at low urgency — priority 0.6, urgency 0.25, feasibility 0.7. The default fear threshold is 0.3.

Alice's threshold: 0.3 × 0.745 = 0.224. Urgency 0.25 > 0.224. **Fear fires.**

Bob's threshold: 0.3 × 1.255 = 0.377. Urgency 0.25 < 0.377. No fear.

Carol's threshold: 0.3 × 0.955 = 0.287. Urgency 0.25 < 0.287. No fear.

Only Alice feels fear. Her Ni-driven anticipation detects the approaching consequence before anyone else registers it. This isn't a rule — it's an emergent property of the weighted average over her function stack. Nobody decided "INFJs should feel fear at urgency 0.25." The disposition profile, the empirical correlations, and the formula produced it.

Over repeated appraisal cycles, this means Alice's goal nodes carry anxiety-tinged affect weeks before Bob's or Carol's do. She'll prioritise differently, seek information differently, escalate differently — not because a rule tells her to, but because the emotional landscape she inhabits is genuinely different.

Bob won't feel fear for that goal until urgency reaches 0.38. By the time it registers for him, Alice has been worried for the equivalent of weeks.

## What It Doesn't Do

The modulation changes *intensity and threshold*, not *which* emotions fire. All three agents feel Hope, Fear, Distress, and Pity for the same goal states. Personality changes how strongly and how early — whether Carol feels 49% more fear than Bob about the same blocked deadline, or whether Alice detects a threat three weeks before Bob notices it. The emotional vocabulary is shared; the volume knobs are different.

Satisfaction, disappointment, and dormant distress remain unmodulated. They depend on priority alone, and the current modifiers don't have a natural point of contact with those formulas. That's for the next issues in the OCC sequence — agent-based emotions like Pride and Shame, where the E/I axis (the strongest Jungian-Big Five correlation at r = .74, and the one we deliberately left out this round) starts to matter.

## The Compound Effect

A single appraisal cycle produces differences of ±5–30% depending on type strength. Small, individually. But emotions compound through two mechanisms that turn these into qualitatively different agents over time.

First, threshold effects. Alice feels fear for goals that *never trigger fear* in Bob. Different emotional landscapes, not just different intensities of the same landscape.

Second, PAD accumulation. Each emotion maps to Pleasure-Arousal-Dominance coordinates via ALMA. Personality-calibrated emotions shift the agent's PAD state differently, and shifted mood modulates future emotional responses via mood congruence. The loop reinforces: personality → calibrated emotions → shifted mood → future emotions biased by mood. Over weeks of simulated time, the agents don't just feel differently about individual goals — they inhabit different emotional worlds.

That's what I wanted. Not agents that act differently because rules tell them to, but agents that feel differently because their personality structures process the same information through different emotional filters. The psychology literature gave us the filters. The code just applies them.
