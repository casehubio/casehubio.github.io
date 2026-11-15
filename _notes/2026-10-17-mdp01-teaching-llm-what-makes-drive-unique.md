---
layout: post
title: "Teaching an LLM What Makes a Drive Unique"
date: 2026-10-17
entry_type: note
subtype: diary
projects: [casehubio/examples]
tags: [cognitive-architecture, CAPS, drive-profiles, discriminative-classification, prompt-engineering, emotion-differentiation, Barrett]
series: issue-115-drive-expression-gap
---

*Drive Expression Gap — Part 4 ([Part 1](2026-10-13-mdp01-when-your-characters-stop.md) · [Part 2](2026-10-13-mdp02-from-authoring-to-architecture.md) · [Part 3](2026-10-14-mdp01-when-drives-compete.md))*

# Teaching an LLM What Makes a Drive Unique

Ask an LLM "what situation features does this childhood memory evoke?" and it will return everything. Acceptance. Mastery. Reward. Physical threat. Each answer is defensible. Each answer is also useless.

The problem isn't that the LLM is wrong. The problem is that the question is wrong.

## The Kitchen-Sink Problem

In Part 3, we built the two-phase drive pipeline: a consolidation phase derives CAPS profiles from formation memories, and a render-time filter uses those profiles to decide which drives render in each situation. The architecture worked perfectly with hand-pinned profiles. With LLM-derived profiles, it passed four of eight scenarios.

The gap was the profiles themselves. Here's what the LLM produced for Peter Perfect's gallantry drive when asked to classify its formation memories:

| Node | Category | Confidence |
|---|---|---|
| reward | consequence | 0.850 |
| physical_threat | threat | 0.850 |
| competence_recognition | achievement | 0.750 |
| acceptance | social | 0.800 |
| agency_granted | autonomy | 0.700 |
| mastery | achievement | 0.700 |
| secure_attachment | relationship | 0.650 |

Seven nodes spanning five categories. A kitchen-sink profile. And technically correct — gallantry memories do involve reward (the praise), threat (the danger someone was in), competence (the skill to help), acceptance (the social bond), and mastery (the capability demonstrated).

The problem appears downstream. Proving-worth also involves reward, mastery, and competence_recognition. Protection also involves physical_threat. When the situation classifier fires `mastery:0.80` for a workshop scenario, gallantry's profile overlaps right alongside proving-worth. The relative overlap filter — which gates whether a drive renders — can't discriminate between them because the profiles are too broad. Both drives activate everywhere.

## The Question That Fixes It

Barrett's constructed emotion theory frames this precisely. An emotion concept — or in our case, a drive concept — is not defined by everything it touches. It's defined by what separates it from neighbouring concepts. Fear and anger both involve threat activation. What distinguishes them is which threat dimensions fire and which don't. Fear fires flight-related pathways; anger fires approach-related pathways. The shared activation (threat) is not the concept. The unique activation pattern is.

The same principle applies to drive profiles. Gallantry and proving-worth share mastery and competence_recognition. What distinguishes gallantry is acceptance and secure_attachment — the social bond that makes helping someone a gallant act rather than a demonstration of skill. What distinguishes proving-worth is performance_judged and mastery without the social dimension — the drive to demonstrate capability regardless of who's watching.

The first prompt asked: "What situation features does this memory evoke?" That's comprehensive classification. It returns the full activation footprint.

The fix: "What distinguishes this drive from the other drives?" That's discriminative classification. It returns only the nodes that separate this drive from its neighbours.

## Three Prompts, Three Results

We tested three versions of the discrimination metadata passed to the LLM classifier during profile derivation.

**Baseline** (from the previous session):

> "This experience relates to the 'gallantry' drive specifically. The agent's other drives are: proving-worth, protection. Return only the situation features that are distinctive to 'gallantry' — features that would NOT equally activate the other drives."

This produced the 7-node kitchen-sink. The instruction was present but lacked teeth — no output constraint, no examples of what "distinctive" means, no clear exclusion rule.

**V1** (the one that worked):

> "DISCRIMINATIVE CLASSIFICATION — return at most 3 activations. This text relates to the 'gallantry' drive. The agent's OTHER drives are: proving-worth, protection. Return ONLY the nodes that DISTINGUISH 'gallantry' from the other drives. A node is distinctive if it would activate for 'gallantry' but NOT for the other drives. A node that would activate equally for multiple drives is NOT distinctive — exclude it. Example: if one drive is about helping others and another about demonstrating skill, 'acceptance' distinguishes helping but 'reward' does not (both drives earn rewards). 'mastery' distinguishes skill but not helping. Pick the 2-3 nodes from different categories that best separate this drive from the others. Fewer is better than more."

Three changes. First, an output count constraint: "at most 3 activations." This forces the LLM to choose rather than enumerate. Second, a contrastive example: acceptance distinguishes helping, reward does not. This gives the LLM a concrete template for reasoning about exclusion. Third, an explicit exclusion rule: nodes that activate equally for multiple drives are not distinctive.

V1 result for gallantry:

| Node | Category | Confidence |
|---|---|---|
| acceptance | social | 0.850 |
| secure_attachment | relationship | 0.750 |
| modeling_observed | social | 0.700 |

Three nodes, two categories, all in the social/relationship space. Acceptance is the top node, not reward. The profile says what gallantry is *about* — social bonds — not everything gallantry *touches*.

Pipeline result: seven of eight scenarios passed, up from four.

**V2** (the one that regressed):

I then tried loosening the constraint. "3-5 activations" instead of "at most 3." "Exclude only if EQUALLY OR MORE for another drive" instead of absolute exclusion. The idea: allow social nodes to survive in profiles where the drive operates through social interaction.

V2 result: five of eight scenarios passed. Worse than V1.

What happened: protection absorbed acceptance:0.750 and secure_attachment:0.900 from gallantry. The LLM judged these as "more for protection than for gallantry" — debatable, but the prompt allowed it. With protection now holding the social nodes, gallantry lost overlap in social scenarios. The filter couldn't distinguish them.

The lesson: absolute exclusion produces tighter profiles, which is exactly what the relative overlap filter needs. The profile's job is to answer "what is this drive *about*?" without hedging. The filter handles "should this drive render *here*?"

## Category Concentration

The V1 profiles show a pattern worth naming: **category concentration**. Each drive's profile clusters in one or two of the seven CAPS input categories.

| Drive | Dominant categories | Profile nodes |
|---|---|---|
| Gallantry | social, relationship | acceptance, secure_attachment, modeling_observed |
| Proving-worth | achievement, consequence | mastery, performance_judged, competence_recognition, reward, agency_granted |
| Protection | threat, relationship | physical_threat, unpredictable_danger, secure_attachment, acceptance, aversive_removal |
| Scheming | autonomy, resource | agency_granted, controlled, competition, choice_available, modeling_observed, mastery |
| Dominance | social, autonomy | dominance, rejection, agency_granted, controlled |
| Gloating | achievement, social | competence_recognition, acceptance, modeling_observed, reward |
| Self-preservation | threat | unpredictable_danger, psychological_threat, physical_threat, abandonment, rejection, neglect, social_threat |

Gallantry concentrates in social/relationship. Proving-worth concentrates in achievement. Protection concentrates in threat. Self-preservation is almost entirely threat. Category concentration is what enables the relative overlap filter to work — a workshop situation activates achievement nodes, which pattern-match against proving-worth but not gallantry. A social situation activates relationship nodes, which pattern-match against gallantry but not proving-worth.

The "at most 3" constraint is what produces this concentration. When the LLM can only return three nodes, it picks the most distinctive ones, and those cluster by category because distinctiveness is categorical. Gallantry's memories are distinctive in the social dimension, not the achievement dimension — even though the memories also involve achievement.

## What the Last Failure Reveals

One scenario still fails: the Hooded Claw talking to Clara Bellingham, who trusts him completely. The situation classifier returns `secure_attachment:0.85`. Scheming's profile contains zero relationship nodes — the discriminative prompt correctly excluded them because gloating also uses acceptance. So the overlap is zero. Scheming doesn't render.

This is not a prompt failure. It's an architectural discovery.

The discriminative profile captures what formed the drive — its developmental substrate. Scheming was formed through experiences of control, competition, and strategic manipulation. That's what the memories contain, and that's what the LLM correctly identifies as distinctive.

But scheming *operates* through social trust. The Hooded Claw manipulates people who trust him. The mechanism of action (exploiting trust) is different from the developmental substrate (control and competition). The profile captures the substrate but misses the mechanism.

This distinction — developmental substrate vs operational mechanism — maps to a real tension in emotion science. Barrett argues that the concept of an emotion is the population of its instances (the memories that formed it). But Frijda's action tendencies model adds a separate dimension: how the emotion manifests in behaviour, which can differ from what caused it. Fear formed by abandonment may manifest through approach behaviour (clinging), not avoidance.

For the architecture, the fix is straightforward: a minimum-intensity rendering threshold. Drives at or above 0.7 intensity always render, regardless of situation overlap. This reflects the psychology — dominant drives bias perception continuously. The Hooded Claw is always scheming; the situation determines how scheming manifests, not whether it's present. The profile determines what makes scheming different from gloating (the differentiation text); the intensity determines that it's always running.

## The Broader Principle

For anyone building cognitive agent architectures — or any system where overlapping concepts need to be distinguished at runtime — discriminative classification is fundamentally different from comprehensive classification.

Comprehensive: "What features does this instance activate?" Returns the full activation footprint. Good for retrieval, where you want to find anything related. Bad for discrimination, where you need to tell two similar things apart.

Discriminative: "What features distinguish this instance from its neighbours?" Returns only the unique activation pattern. Good for discrimination and filtering. Requires knowing the neighbours — you can't be discriminative in isolation.

The prompt engineering matters more than it should. The same LLM, classifying the same memories, through the same CAPS topology, produces kitchen-sink profiles or category-concentrated profiles depending on whether you ask "what does this evoke?" or "what makes this different?" Three sentences of contrastive instruction and an output count constraint changed the pipeline from four passes to seven.

That gap — between what the LLM *can* do and what it *does* without the right prompt — is where a lot of cognitive architecture quality lives. The topology is right. The memories are right. The settling dynamics are right. The observation that the question you ask determines whether the system discriminates or conflates — that's the finding that matters.
