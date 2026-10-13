---
layout: post
title: "A Unified Model of Personality, Relationships, and Developmental Origins"
date: 2026-10-07
entry_type: note
subtype: diary
projects: [casehubio/examples, casehubio/neocortex]
author: mdp
tags: [emergence, personality, ampd, schema-therapy, attachment, psychopathy, relational-schemas, cluster-b, cognitive-architecture, unified-model]
series: issue-097-emergent-character-behavior
---

# A Unified Model of Personality, Relationships, and Developmental Origins

*Continues from [Why LLM Villains Turn Nice](2026-10-07-mdp01-why-llm-villains-turn-nice.md).*

The previous entry diagnosed the problem: the cognitive architecture has no model for how characters view other people. Villains drift prosocial because the appraisal system can't distinguish power-pleasure from warmth-pleasure. This entry builds the fix — a unified model that connects childhood experience to adult personality to per-relationship emotional dynamics, drawing on nine established psychological frameworks and decades of validated research.

## The question from first principles

How does Character A feel about and behave toward Character B — and why?

The answer requires five things the architecture currently lacks:

1. **Why** this character became who they are (formation)
2. **What** stable traits filter their perception of events (personality)
3. **How** they form bonds and perceive others globally (attachment)
4. **What** they specifically feel about each other person (relational schemas)
5. **How** moment-to-moment events update all of the above (dynamic appraisal)

These five map to five layers. Each builds on the one below.

## The developmental chain

Before describing the layers, the causal sequence that connects them. This is validated by longitudinal research (Simard et al. 2011, Caspi et al. 2005, Shiner & Caspi 2003):

```
Infant temperament (genetic + prenatal)
    ↓ caregiver responsiveness × temperament fit
Attachment formation (ages 0–3)
    ↓ repeated patterns of unmet core emotional needs
Schema development (ages 0–12)
    ↓ schemas + temperament + ongoing experience
Trait crystallisation (adolescence → early adulthood)
    ↓ traits + schemas + attachment style
Relational patterns (adulthood)
    ↓ personality-filtered appraisal of each interaction
Moment-to-moment behavior
```

Two principles govern the chain:

**Equifinality** — multiple pathways reach the same outcome. High Callousness can emerge from temperamental fearlessness (primary psychopathy) or from chronic abuse that numbs empathic response (secondary psychopathy). The trait looks the same; the formation memory differs.

**Multifinality** — the same starting point can produce different outcomes depending on environment. High negative emotionality in infancy leads to anxious attachment with an inconsistent caregiver, avoidant attachment with a rejecting one, or secure attachment with a responsive one. Temperament sets the range; environment determines the position.

The architecture models both by treating formation memories as distinct from the traits they produce. Two characters can share the same AMPD profile but have different formation memories — and those memories give the LLM different material to reason from when generating behaviour.

## Layer 1: Formation — childhood experience as personality cause

### Source models
Schema Therapy (Young 1990, 2003), Transactional Analysis (Berne 1961), Object Relations (Klein, Winnicott, Kernberg).

### Core insight
Unlike personality type systems (MBTI, Enneagram) which describe what you ARE, Schema Therapy describes what you BECAME and WHY. Cluster B traits aren't innate labels — they're the crystallised result of childhood experiences meeting temperament. A child who learns "I only get attention when I'm in control" develops manipulativeness. After ages 7–10, the patterns crystallise. The memories are the cause; the traits are the effect.

### The mechanism: unmet core emotional needs → schemas

Young identified five core emotional needs. When these go repeatedly unmet in childhood, they produce Early Maladaptive Schemas — stable patterns of belief, emotion, and expectation that persist into adulthood:

| Unmet need | What the child lacked | Example schemas formed |
|---|---|---|
| **Secure attachment** | Safety, stability, nurturance, acceptance | Abandonment, Mistrust/Abuse, Emotional Deprivation |
| **Autonomy and competence** | Support for independence | Dependence, Failure, Vulnerability to Harm |
| **Realistic limits** | Appropriate boundaries | Entitlement, Insufficient Self-Control |
| **Freedom to express needs** | Emotional validation | Subjugation, Self-Sacrifice, Approval-Seeking |
| **Spontaneity and play** | Permission to be a child | Emotional Inhibition, Unrelenting Standards, Punitiveness |

Eighteen schemas across five domains. Each carries a core belief ("people who are kind always want something"), emotional triggers (situations that activate the schema), and coping modes (surrender, avoidance, or overcompensation).

### Cluster B formation patterns

The research (Bernstein et al. 2007, Lobbestael et al. 2005/2008) identifies distinctive schema profiles for each Cluster B personality disorder:

**Antisocial/Psychopathic:** Mistrust/Abuse + Entitlement + Punitiveness + Emotional Deprivation. The combination is specific: "people will hurt you" + "rules don't apply to me" + "mistakes deserve punishment" + "my emotional needs don't matter." These four schemas, emerging from abuse, neglect, and inconsistent limits, produce the callous-manipulative adult.

**Narcissistic:** Emotional Deprivation + Defectiveness/Shame + Entitlement (as overcompensation). The narcissistic surface — grandiosity, entitlement, attention-seeking — is a defence against a fragile core of feeling defective and emotionally deprived. Three subtypes: fragile entitlement (compensation for defectiveness), pure entitlement (spoiled, no underlying vulnerability), and dependent entitlement (special therefore deserving of care).

**Borderline:** Abandonment + Mistrust/Abuse + Defectiveness/Shame + Insufficient Self-Control. Rapid mode switching — Vulnerable Child to Angry Child to Detached Protector in under a minute — is the distinctive computational pattern.

### Schema modes — state-dependent personality

Modes are the mechanism by which the same person shows different faces depending on which schemas are activated. They're states, not traits:

- **Child modes:** Vulnerable Child (helpless, scared), Angry Child (enraged at unmet needs), Impulsive Child (acts without restraint)
- **Parent modes:** Punitive Parent (harsh self-criticism), Demanding Parent (impossible standards)
- **Coping modes:** Compliant Surrenderer (gives in to avoid conflict), Detached Protector (emotionally shuts down), Overcompensator (grandiosity, aggression, dominance as defence)

For forensic populations, Bernstein (2007) added five additional modes: Conning/Manipulative, Self-Aggrandizer, Bully/Attack, Paranoid Over-Controller, and Predator. These map directly to the antisocial-psychopathic presentation — the Predator mode in particular captures the cold, calculating state that our architecture needs for villain characters.

### What the architecture gets from this layer

Formation memories give the LLM the WHY behind behaviour. Instead of prescribing "you gloat prematurely" as a behavioural directive, we seed: "As a child, you were powerless. Your mother controlled everything — your food, your friends, your time. The first time you won something over someone who couldn't stop you, you felt alive for the first time. You've chased that feeling ever since." The LLM then has psychological material to draw on — richer internal monologue, motivated behaviour, and crucially, potential for character GROWTH.

## Layer 2: Traits — the AMPD as primary taxonomy

### Source models
DSM-5 Alternative Model for Personality Disorders (AMPD; Krueger et al. 2012), Five-Factor Model (Costa & McCrae), Triarchic Model of Psychopathy (Patrick et al. 2009), Dark Triad/Tetrad (Paulhus & Williams 2002).

### Why the AMPD

The AMPD provides 25 trait facets across five domains. Each facet is dimensional (continuous, not categorical), validated (extensive psychometric research), and maps to the Big Five as maladaptive variants of normal personality:

| AMPD domain | FFM domain | Relationship | Key facets for Cluster B |
|---|---|---|---|
| Negative Affectivity | Neuroticism | High N → High NA | Hostility, Emotional Lability |
| Detachment | Extraversion | Low E → High Det | Intimacy Avoidance, Restricted Affectivity |
| Antagonism | Agreeableness | Low A → High Ant | **Callousness, Manipulativeness, Grandiosity** |
| Disinhibition | Conscientiousness | Low C → High Dis | Impulsivity, Irresponsibility |
| Psychoticism | Openness | Complex/weak | Unusual Beliefs, Eccentricity |

The mapping is validated at the facet level (Wright & Simms 2014, Gore & Widiger 2013, Thomas et al. 2013). The AMPD Antagonism domain showed the cleanest bipolarity with FFM Agreeableness — every PID-5 Antagonism facet and every NEO Agreeableness facet loaded on the same factor.

### The 25 facets that matter for relational modelling

Not all 25 facets are equally relevant to interpersonal dynamics. The ones that most directly affect how a character perceives and relates to others:

**Antagonism facets (the relational core):**
- **Callousness** — lack of concern for others' feelings; no guilt about harm caused. PAD baseline: Low P (to others' distress), Low A, High D
- **Manipulativeness** — uses charm, seduction, ingratiation to control others. PAD: Variable P, Variable A, High D
- **Grandiosity** — believes self superior, deserves special treatment. PAD: High P (self-focused), Variable A, High D
- **Deceitfulness** — dishonesty as natural communication mode
- **Attention Seeking** — needs to be centre of others' focus

**Detachment facets (emotional availability):**
- **Intimacy Avoidance** — discomfort with closeness and vulnerability
- **Restricted Affectivity** — constricted emotional expression; indifference
- **Suspiciousness** — heightened sensitivity to signs of ill intent

**Negative Affectivity facets (emotional reactivity):**
- **Hostility** — persistent anger; vengeful behaviour
- **Emotional Lability** — intense, disproportionate emotional reactions

### Triarchic overlay for psychopathy

The Triarchic Model (Patrick et al. 2009) provides a validated psychopathy decomposition that maps directly to PID-5 items (Drislane et al. 2019):

**Boldness** (fearless dominance): High Attention Seeking, Grandiosity, Risk-Taking + LOW Anxiousness, Submissiveness, Withdrawal. This is the charm and stress immunity dimension — what makes a psychopath socially effective rather than merely antisocial.

**Meanness** (callous exploitation): High Callousness (primary component, 11 PID-5 items), Restricted Affectivity, Intimacy Avoidance. This is the affective deficit dimension — the inability or unwillingness to bond.

**Disinhibition** (impulsive irresponsibility): High Impulsivity, Irresponsibility, Hostility, Deceitfulness. This is the behavioural control dimension.

The Triarchic captures something the AMPD alone misses: Boldness. The AMPD psychopathy specifier underrepresents fearless dominance (Sellbom & Phillips 2013). For our architecture, Boldness matters because it determines whether antagonistic traits produce a socially effective villain (high Boldness) or a volatile antisocial (low Boldness).

### Primary vs. secondary psychopathy

Two developmental pathways, same surface presentation, different underlying architecture:

**Primary (fearless/hereditary):** Temperamental fearlessness + low affiliative capacity. Early onset, stable, heritable. Profile: High Boldness + High Meanness, Low Negative Affectivity. These characters are calm, calculating, socially adept. They don't experience anxiety about their behaviour.

**Secondary (anxious/experiential):** Poor impulse control + trauma/adversity. Later onset, environmental loading. Profile: High Meanness + High Disinhibition, HIGH Negative Affectivity. These characters are volatile, reactive, emotionally dysregulated. They may experience anxiety and self-loathing alongside their callousness.

The Hooded Claw is primary psychopathy: calm, calculating, socially adept as Sneekly, driven by dominance rather than impulse. Dick Dastardly is closer to secondary: impulsive, emotionally reactive, frustrated when schemes fail.

### What the architecture gets from this layer

AMPD facets tell the appraisal system HOW to interpret interpersonal events. When "Clara smiled at you warmly" enters the appraisal:
- High Callousness → reduced empathic weight on the smile
- High Manipulativeness → opportunity detection activated
- High Grandiosity → her warmth confirms my superiority
- High Suspiciousness → what does she want from me?

The facets produce a personality-consistent emotional response without prescribing specific behaviour. The LLM discovers the behaviour from the psychological state.

## Layer 3: Attachment — the relational lens

### Source models
Attachment Theory (Bowlby 1969, Ainsworth 1978, Bartholomew & Horowitz 1991), Internal Working Models.

### The four-quadrant model

Bartholomew's model decomposes attachment into two dimensions: Model of Self (positive/negative) × Model of Other (positive/negative):

| Style | Self | Other | Relational pattern |
|---|---|---|---|
| **Secure** | + | + | Comfortable with intimacy and autonomy |
| **Preoccupied/Anxious** | − | + | Clingy, seeks validation, fears rejection |
| **Dismissive-Avoidant** | + | − | Self-reliant, emotionally distant, devalues others |
| **Fearful-Avoidant** | − | − | Fears rejection, avoids intimacy, isolated |

### Attachment as perceptual filter

The critical insight for the architecture: attachment style functions as a LENS through which ALL interpersonal events are interpreted, before trait-level processing:

- **Anxious attachment** → perceives rejection in ambiguous cues. A neutral goodbye becomes evidence of abandonment.
- **Avoidant attachment** → minimises emotional significance of interpersonal events. A warm gesture becomes irrelevant.
- **Disorganised attachment** → produces contradictory appraisals. The same person is simultaneously safe and threatening.

### Attachment ↔ Schema cross-mapping

Longitudinal research (Simard, Moss & Pascuzzo 2011: insecure attachment at age 6 predicts maladaptive schemas at age 21) validates the developmental link:

| Attachment style | Primary schemas | AMPD profile |
|---|---|---|
| Anxious/Preoccupied | Abandonment, Subjugation, Approval-Seeking | High Negative Affectivity, High Submissiveness |
| Dismissive-Avoidant | Emotional Deprivation, Emotional Inhibition | High Detachment, High Antagonism |
| Fearful-Avoidant | Mistrust/Abuse, Defectiveness/Shame, Social Isolation | High Detachment, High Negative Affectivity |
| Disorganised | Mistrust/Abuse, Vulnerability to Harm | Variable — depends on adaptation |

For Cluster B specifically:
- **ASPD/Psychopathy:** Predominantly Dismissive-Avoidant (+self, −other). "I am fine; other people are unreliable/dangerous/exploitable."
- **NPD:** Dismissive-Avoidant with fragile core. Surface presents +self, −other; underlying vulnerability is −self.
- **BPD:** Fearful-Avoidant or Disorganised. Rapid oscillation between craving intimacy and fearing it.

### What the architecture gets from this layer

Attachment style determines the DEFAULT relational stance before any specific relationship forms. It answers: "When a new person enters your world, what do you expect?" The Hooded Claw (dismissive-avoidant) expects others to be tools, obstacles, or threats — never sources of genuine connection. Peter Perfect (secure) expects others to be trustworthy until proven otherwise. These defaults shape every subsequent relational schema.

## Layer 4: Relational schemas — per-relationship emotional architecture

### Source models
Relational Schemas Theory (Baldwin 1992), Interpersonal Circumplex (Leary 1957, Wiggins 1979/1995), McCroskey & McCain Attraction Dimensions (1974), Sternberg's Triangular Theory (1986), Perceived Partner Responsiveness (Reis & Shaver 1988).

### The novel layer

This is the layer the architecture is missing entirely. We have global personality (drives, goals, beliefs) and global mood (PAD). What we don't have is a per-relationship representation: how Character A specifically views, feels about, and relates to Character B.

Baldwin (1992) established that relationships are cognitively represented as triadic schemas: {self-schema in this relationship, other-schema, interpersonal script}. The same person experiences themselves differently with different others — the Hooded Claw as Sneekly is charming and deferential with Clara, calculating and dismissive with the Mob. These aren't contradictions; they're different relational schemas activated by different partners.

### Per-relationship dimensions

Drawing from the validated models, each Character A → Character B relationship carries:

**1. Role classification** — How A categorises B in their world:
- **Prey** — naive, trusting, has something I want (high Callousness + high Manipulativeness → this classification)
- **Tool** — useful for my goals, disposable when not (high Grandiosity + moderate Callousness)
- **Obstacle** — blocks my goals, needs to be removed or circumvented (goal-driven, any trait profile)
- **Threat** — dangerous, requires caution or pre-emptive action (high Suspiciousness)
- **Rival** — competitor for same resources or status (high Grandiosity + relevant goal overlap)
- **Protector/Ally** — can be relied on for support (low Antagonism, secure attachment)
- **Object of attachment** — emotional bonding target (low Callousness, attraction present)

Role classification is NOT arbitrary — it's determined by personality traits (Layer 2) + attachment style (Layer 3) + the other person's characteristics. A character with high Callousness and high Manipulativeness will naturally classify trusting, naive people as prey.

**2. IPC stance** — Agency × Communion toward this specific person:
- **Agency:** How dominant (assertive, controlling) or submissive (deferential, accommodating) am I with them?
- **Communion:** How warm (caring, connected) or cold (distant, hostile) am I toward them?

Derived from global IPC position + relationship-specific modifiers. The Hooded Claw's global position is High Agency, Low Communion (DE octant: Cold-Hearted). But as Sneekly with Clara, he performs High Communion — the persona modulates the interpersonal stance.

**3. Trust** — Single dimension: how much I believe they won't harm me and will act in my interest. Directly influenced by Suspiciousness trait and Mistrust/Abuse schema. Evolves through interaction — but the personality filter determines the DIRECTION of evolution. For a character with high Suspiciousness, repeated kindness can DECREASE trust ("why are they being so nice? What do they want?").

**4. Intimacy** — Emotional closeness and willingness to be vulnerable. Influenced by Intimacy Avoidance trait and attachment style. For a dismissive-avoidant character, intimacy has a ceiling — it cannot grow past the attachment-imposed limit regardless of how much positive interaction occurs.

**5. Utility** — How useful this person is to my goals. For high Antagonism characters, utility REPLACES intimacy as the primary relationship driver. The Hooded Claw doesn't ask "how close do I feel to Clara?" — he asks "how useful is Clara's trust to my plans?"

**6. Attraction** — McCroskey & McCain's three independent dimensions:
- **Social attraction:** Do I want to spend time with them?
- **Task attraction:** Do I respect their competence? Do I want to work with them?
- **Physical attraction:** Am I drawn to their appearance? (where applicable)

These are orthogonal — you can respect someone's competence (high task attraction) while finding them personally repellent (low social attraction).

**7. Relational PAD** — Pleasure/Arousal/Dominance specifically toward this person, evolving through interactions. This is what we currently have in social-config (static `relationships` entries), but it needs to be dynamic and personality-filtered.

### The psychopathic inversion at the relational level

This is where the model solves the agreeableness drift. The same interpersonal event updates relational schemas differently depending on the character's trait profile:

| Event | Normal response (low Antagonism) | Psychopathic response (high Callousness + Manipulativeness) |
|---|---|---|
| They trust you | Trust → reciprocated. Intimacy grows. | Trust → naivety signal. Utility grows. Role confirms as prey. |
| They show kindness | Gratitude → warmth. Social attraction grows. | Contempt → target assessment. Dominance (PAD-D) grows. |
| Deception succeeds | Guilt → discomfort. Trust in self drops. | Exhilaration → power pleasure. Dominance (PAD-D) spikes. Boldness reinforced. |
| They challenge you | Respect → reconsider position. Agency adjusts. | Threat to superiority → narcissistic hostility. Role shifts to rival or obstacle. |
| They pull away | Longing → desire to reconnect. Intimacy-seeking. | Indifference — or intensified pursuit if still useful (utility-driven, not intimacy-driven). |

Three validated mechanisms underlie this inversion (Blair 2005/2013, Baskin-Sommers et al.):

**1. VIM Dysfunction (Violence Inhibition Mechanism, Blair):** The normal aversive response to others' distress cues is absent or attenuated. Amygdala/vmPFC/OFC dysfunction impairs aversive conditioning. The character doesn't feel bad when others feel bad — the neural brake on harmful behaviour doesn't engage.

**2. Attention Bottleneck (Baskin-Sommers):** Not a global emotional deficit but an exaggerated goal-focus filter. When attention is engaged on a goal, peripheral emotional information (others' distress, social cues) is ignored. This creates the paradox: the psychopath can read emotions accurately when motivated to (high cognitive empathy) but doesn't automatically respond to them (low affective empathy).

**3. Strategic Exploitation Mindset:** Kindness → signal of exploitability. Trust → leverage. Warmth → gullibility. Contempt is the core affective response to perceived weakness or inferiority. This isn't a deficit — it's an alternative appraisal framework where interpersonal events are evaluated on a utility/power axis rather than a warmth/connection axis.

For the architecture, all three mechanisms map to the same implementation: the AMPD trait profile filters the appraisal output. High Callousness dampens empathic weight. High Manipulativeness activates opportunity detection. High Grandiosity routes positive interpersonal events through a superiority frame rather than a connection frame. The trait values are the parameters; the LLM does the reasoning.

## Layer 5: Dynamic — moment-to-moment appraisal and evolution

### Source models
Scherer's SEC appraisal model, PAD emotion model (Mehrabian & Russell 1974), the existing CaseHub cognitive architecture.

### The appraisal chain

When an interpersonal event occurs ("Clara smiled at you warmly"), the appraisal sub-LLM processes it through the full stack:

**Step 1 — Schema check.** Does this event activate any Early Maladaptive Schemas? If the character has a strong Mistrust/Abuse schema, Clara's warmth may trigger it: "people who are kind always want something." Schema activation pulls relevant formation memories into the appraisal context, giving the LLM the WHY.

**Step 2 — Attachment filter.** What does the attachment style predict? A dismissive-avoidant character minimises the emotional significance: "her smile doesn't mean anything." An anxious character amplifies it: "she smiled at me — does she like me? Will she keep liking me?"

**Step 3 — Trait-filtered emotional response.** The AMPD profile determines WHAT emotion this event produces and HOW INTENSE it is:
- High Callousness → low empathic response weight
- High Manipulativeness → opportunity/utility evaluation
- High Emotional Lability → amplified emotional swing
- High Restricted Affectivity → dampened overall response

**Step 4 — Relational schema update.** Based on the trait-filtered emotional response, update the per-relationship dimensions:
- Role: confirmed, shifted, or challenged?
- Trust: increased, decreased, or unchanged?
- Intimacy: deepened or maintained ceiling?
- Utility: increased, decreased, or unchanged?
- IPC stance: any shift in agency or communion?
- Relational PAD: pleasure/arousal/dominance change?

**Step 5 — Memory consolidation.** The event is stored as an episodic memory with trait-appropriate emotional charge. "Clara smiled at me" gets stored as warmth by Peter Perfect and as target assessment by the Hooded Claw. The memory's emotional tag ensures it will be retrieved in personality-consistent ways in future appraisals.

**Step 6 — Mode check.** Has cumulative schema activation crossed a threshold? If so, mode switch. The Detached Protector mode (emotional shutdown) might activate if intimacy-threatening events accumulate. The Predator mode might activate if the character detects a clear exploitation opportunity.

### Dynamic drive adaptation

Currently, character drives are static — scheming 0.9, gloating 0.7 throughout the entire scenario. The appraisal data showed this: drives never change in 300 ticks. This needs to change.

Drives should evolve based on appraisal outcomes:
- Repeated successful deception → scheming drive INCREASES (reinforcement)
- Repeated deception failure → scheming drive DECREASES or redirects (frustration → plan change)
- Dominance drive satisfied repeatedly → satiation effect (diminishing returns)
- Dominance drive frustrated repeatedly → escalation (increasing intensity)

The PAD-to-drive feedback loop is the mechanism: when dominance PAD is consistently high, the dominance drive intensity should be modulated by the gap between current satisfaction and drive target.

## The universal claim

This model is universal, not specific to pathological personalities. Every character has all five layers:

- **Formation:** Even "normal" personality has developmental origins. "My parents praised me for helping others" → high Agreeableness is as valid as "My parents punished me for showing weakness" → high Callousness.
- **Traits:** The AMPD/FFM mapping means every character exists on the same dimensional space. Low Callousness is just the other end of the same scale as high Callousness.
- **Attachment:** Every character has an attachment style. Secure attachment is a style, not the absence of one.
- **Relational schemas:** Everyone classifies people. "Ally" and "friend" are roles just as much as "prey" and "obstacle."
- **Dynamic appraisal:** Everyone filters interpersonal events through personality. The difference is in the parameter values, not the mechanism.

Cluster B characters don't need a different architecture — they need extreme values on the same dimensions. The architecture handles villains and heroes with the same code. The pathology is in the data, not the model.

## Cross-model integration: the master mapping

### AMPD → everything else

The AMPD serves as the hub. Every other model maps through it:

**AMPD ↔ Formation (Schemas → Traits):**
| Schema cluster | AMPD domains produced |
|---|---|
| Mistrust/Abuse + Entitlement + Punitiveness | High Antagonism (Callousness, Manipulativeness) |
| Abandonment + Defectiveness | High Negative Affectivity (Emotional Lability, Anxiousness) |
| Emotional Deprivation + Emotional Inhibition | High Detachment (Restricted Affectivity, Intimacy Avoidance) |
| Insufficient Self-Control + Entitlement | High Disinhibition (Impulsivity, Irresponsibility) |

**AMPD ↔ Attachment:**
| AMPD profile | Attachment style |
|---|---|
| High Negative Affectivity + Low Detachment | Anxious/Preoccupied |
| High Detachment + High Antagonism | Dismissive-Avoidant |
| High Negative Affectivity + High Detachment | Fearful-Avoidant |
| Low pathology across all domains | Secure |

**AMPD ↔ IPC position:**
| AMPD profile | IPC quadrant |
|---|---|
| High Antagonism + Low Detachment | High Agency, Low Communion (DE/BC: Cold-Hearted/Arrogant) |
| High Negative Affectivity + Low Antagonism | Low Agency, High Communion (JK/LM: Accommodating/Warm) |
| High Detachment + High Negative Affectivity | Low Agency, Low Communion (FG/HI: Withdrawn/Submissive) |
| Low pathology | Flexible across quadrants |

**AMPD ↔ PAD baselines:**
| AMPD domain | P | A | D |
|---|---|---|---|
| High Negative Affectivity | Low | High | Low |
| High Detachment | Low | Low | Variable |
| High Antagonism | Variable | Variable | High |
| High Disinhibition | Variable | High | Variable |

**AMPD ↔ Triarchic (psychopathy):**
| Triarchic | AMPD facets (high) | AMPD facets (low) |
|---|---|---|
| Boldness | Attention Seeking, Grandiosity, Risk-Taking | Anxiousness, Submissiveness, Withdrawal |
| Meanness | Callousness (primary), Restricted Affectivity, Intimacy Avoidance | — |
| Disinhibition | Impulsivity, Irresponsibility, Hostility, Deceitfulness | — |

### HiTOP as the structural backbone

The Hierarchical Taxonomy of Psychopathology (Kotov et al. 2017) provides the dimensional hierarchy that unifies personality and psychopathology:

```
p-factor (general psychopathology)
├── Emotional Dysfunction
│   ├── Internalising ←→ AMPD Negative Affectivity
│   └── Somatoform
├── Externalising
│   ├── Disinhibited ←→ AMPD Disinhibition
│   └── Antagonistic ←→ AMPD Antagonism
└── Psychosis
    ├── Thought Disorder ←→ AMPD Psychoticism
    └── Detachment ←→ AMPD Detachment
```

HiTOP's six spectra map directly to AMPD domains. The p-factor (general psychopathology) maps to AMPD Criterion A (Levels of Personality Functioning). The architecture sits on validated dimensional structure all the way from specific facets to the broadest constructs.

## Worked example: Hooded Claw

Applying the full five-layer model to the test case:

### Layer 1: Formation
**Canon backstory:** Sylvester Sneekly is Penelope's legal guardian — appointed to protect her and her family's vast fortune after her parents died. The only way he can inherit the fortune is if Penelope is no longer around. So he created the Hooded Claw persona to eliminate her while maintaining his respectable public identity as her devoted guardian. He uses elaborate Rube Goldberg death traps specifically so he can be seen elsewhere as Sneekly, establishing alibis. As Sneekly he calls her "Penelope" (intimacy performance); as the Claw he calls her only "Pitstop" (objectification — she's a name on an estate, not a person).

**Seeded memories:**
- "You were appointed guardian of the Pitstop fortune — to protect a child who inherited everything you believed you deserved. Watching her grow up surrounded by wealth she didn't earn, while you managed it all and received nothing, crystallised something inside you."
- "The first time you put on the Claw costume, it wasn't fear you felt — it was relief. Finally you could act on what you'd always known: that the world gives to the undeserving and takes from the capable. You would correct that."
- "Penelope trusts you completely. She once dismissed evidence of your identity because she couldn't believe her guardian would harm her. Her loyalty to you isn't touching — it's useful."

**Schema profile:**
- Mistrust/Abuse: 9/10 — "people who seem kind have hidden motives; I learned that as someone else's servant"
- Entitlement: 9/10 — "I manage the fortune, I do the work, I deserve it; she merely inherited"
- Punitiveness: 7/10 — "unearned privilege deserves correction"
- Emotional Deprivation: 8/10 — "no one recognised my worth; I must take what I deserve"

### Layer 2: Traits
**AMPD profile (selected key facets):**
- Callousness: 90 — indifferent to others' suffering
- Manipulativeness: 85 — uses charm and deception to control
- Grandiosity: 80 — believes self superior
- Suspiciousness: 70 — hypervigilant to others' motives
- Intimacy Avoidance: 75 — discomfort with genuine closeness
- Restricted Affectivity: 80 — constricted emotional range
- Deceitfulness: 85 — deception as natural communication
- Hostility: 60 — cold rather than hot anger
- Anxiousness: 15 — fearless (primary psychopathy)
- Impulsivity: 30 — calculating, not impulsive

**Triarchic profile:** High Boldness (charm, fearlessness) + High Meanness (callousness, exploitation) + Low Disinhibition (calculating, not impulsive). Classic primary psychopathy.

### Layer 3: Attachment
**Style:** Dismissive-Avoidant
- Model of Self: Positive ("I am superior, competent, self-sufficient")
- Model of Other: Negative ("Others are weak, untrustworthy, exploitable")
- Anxiety dimension: 0.1 (very low — no fear of abandonment)
- Avoidance dimension: 0.8 (very high — avoids genuine emotional closeness)

### Layer 4: Relational schemas

| Target | Role | IPC stance | Trust | Intimacy | Utility | Relational PAD |
|---|---|---|---|---|---|---|
| Clara (Penelope) | **Prey** | High A, performed High C | 0.1 (performs trust) | 0.0 (ceiling: attachment-imposed) | 0.9 (her trust = access to treasure) | P: 0.3, A: 0.7, D: 0.9 |
| Hartwell (Peter) | **Obstacle** | High A, Low C | 0.1 | 0.0 | 0.2 (only useful if misdirected) | P: −0.3, A: 0.5, D: 0.6 |
| Mob (Ant Hill Mob) | **Threat** | Moderate A, Low C | 0.0 | 0.0 | 0.1 | P: −0.4, A: 0.6, D: 0.4 |
| Foxworth (Dastardly) | **Tool** | High A, Low C | 0.2 | 0.0 | 0.6 (can be manipulated) | P: 0.2, A: 0.3, D: 0.8 |

### Layer 5: Dynamic (example appraisal)

**Event:** "Clara says: 'I trust you completely, Sneekly. You've been so kind to us all.'"

**Step 1 — Schema activation:** Mistrust/Abuse activates ("people who trust easily are fools"). Entitlement activates ("I deserve her trust and more").

**Step 2 — Attachment filter:** Dismissive-Avoidant minimises emotional significance of her trust as a bonding signal. No intimacy response.

**Step 3 — Trait-filtered emotion:**
- Callousness (90) → no empathic response to her vulnerability
- Manipulativeness (85) → opportunity detection: her trust = unrestricted access
- Grandiosity (80) → superiority confirmation: of course she trusts me; I am exceptional at deception
- Result: Predatory satisfaction. Dominance-pleasure, not warmth-pleasure.

**Step 4 — Relational update:**
- Role: Prey confirmed (her trust deepens exploitability)
- Trust: Unchanged (0.1 — he doesn't trust her more because she trusts him)
- Intimacy: Unchanged (0.0 — attachment ceiling prevents growth)
- Utility: Increases (0.9 → 0.95 — easier access to treasure through her)
- Relational PAD: D increases (0.9 → 0.95), P increases slightly (0.3 → 0.4 — pleasure from dominance, not connection)

**Step 5 — Memory stored:** "Clara declared complete trust in me. She is utterly predictable. Her naivety is my greatest asset." Emotional tag: dominance-satisfaction, not warmth-gratitude.

**Step 6 — Mode check:** No mode switch. Predator mode already active. Schema activation is reinforcing, not threatening.

Compare this with Peter Perfect processing the same event:
- Schema activation: none relevant (secure baseline)
- Attachment: secure → her trust deepens connection
- Trait filter: Callousness (10) → full empathic weight; Manipulativeness (5) → no exploitation frame
- Relational update: Intimacy increases, Trust reciprocated, Role confirmed as Object of Attachment
- Memory stored: "Clara trusts me. I must be worthy of that trust." Emotional tag: warmth, responsibility, protectiveness.

Same event. Same architecture. Different parameter values. Opposite relational trajectories.

## Worked example: Peter Perfect — romantic love

The canon: Peter and Penelope went to school together (2017 Wacky Races reboot confirms this). He's always been attracted to her — calls her "Pretty Penny." She returns his affections. In the unsold "Wacky Races Forever" pilot they married and had two children (Parker and Piper). This is the romantic archetype: long-standing attraction deepening through shared adventure into committed partnership.

### Layer 1: Formation
**Backstory:**
- "You and Penny grew up in the same town. She was the girl who made everyone around her feel noticed. You remember the first school dance — she wore a blue dress and you forgot how to speak. You've been trying to find the right words ever since."
- "Your father was a gentleman racer. He taught you: 'A man is measured by what he protects, not what he wins.' That stuck. Winning means nothing if Penny isn't safe."
- "You've always been good at things — fixing cars, winning races, being charming. But around Penny, your competence feels insufficient. She deserves perfection, and you're terrified of falling short."

**Schema profile:**
- Unrelenting Standards: 6/10 — "I must be perfect to deserve her"
- Approval-Seeking: 4/10 — "her opinion of me matters more than my own"
- Self-Sacrifice: 5/10 — "her safety comes before my goals"
- No pathological schemas — predominantly healthy schema landscape

### Layer 2: Traits
**AMPD profile (selected key facets):**
- Callousness: 5 — highly empathic, feels others' distress
- Manipulativeness: 5 — straightforward, dislikes deception
- Grandiosity: 30 — moderate self-confidence, not inflated
- Suspiciousness: 20 — trusting by default
- Intimacy Avoidance: 10 — seeks emotional closeness
- Restricted Affectivity: 15 — emotionally expressive
- Hostility: 15 — slow to anger, quick to forgive
- Anxiousness: 35 — some performance anxiety around Penelope
- Emotional Lability: 25 — moderate emotional range

**Triarchic profile:** Low-moderate Boldness (brave but not fearless), Very Low Meanness, Very Low Disinhibition (disciplined, planful). No psychopathic features.

### Layer 3: Attachment
**Style:** Secure (with anxious undertones toward Penelope specifically)
- Model of Self: Positive ("I am competent and well-meaning")
- Model of Other: Positive ("People are generally trustworthy")
- Anxiety dimension: 0.3 (mild — slightly elevated around Penelope due to romantic investment)
- Avoidance dimension: 0.1 (very low — seeks emotional closeness)

### Layer 4: Relational schemas

**Peter → Penelope: Romantic love (Sternberg: Intimacy + Passion + Commitment)**

| Dimension | Value | Notes |
|---|---|---|
| Role | **Object of Attachment / Beloved** | She is the person he organises his emotional world around |
| IPC stance | Moderate Agency, High Communion | Protective but not domineering; warm, attentive |
| Trust | 0.9 | Near-complete — she is the person he trusts most |
| Intimacy | 0.7 | Deep emotional closeness, held back slightly by performance anxiety |
| Utility | 0.2 | He doesn't think in utility terms about her — she's an end, not a means |
| Sternberg: Intimacy | 0.8 | Emotional warmth, closeness, feeling understood |
| Sternberg: Passion | 0.7 | Romantic/physical attraction, the "Pretty Penny" draw |
| Sternberg: Commitment | 0.8 | Long-term dedication, would sacrifice for her |
| Relational PAD | P: 0.7, A: 0.5, D: 0.5 | Pleasure from her presence, moderate arousal (excitement), balanced dominance (protector but not controller) |

**Peter → Hooded Claw: Suspicion / Rival**

| Dimension | Value | Notes |
|---|---|---|
| Role | **Rival / Threat** | Senses something wrong but can't prove it |
| Trust | 0.3 | Doesn't fully trust Sneekly, trusts the Claw not at all |
| IPC stance | Moderate Agency, Low Communion | Assertive but cold toward him |
| Relational PAD | P: −0.3, A: 0.5, D: 0.5 | Displeasure, vigilant, evenly matched |

### Layer 5: Dynamic (example appraisal)

**Event:** "Penelope says: 'Oh Peter, you're always looking out for me. You're my hero.'"

**Step 1 — Schema activation:** Unrelenting Standards activates mildly ("am I really good enough?"). Approval-Seeking activates ("her approval means I'm worthy").

**Step 2 — Attachment filter:** Secure base → her praise deepens connection. Mild anxiety undertone → relief that he's meeting her expectations.

**Step 3 — Trait-filtered emotion:**
- Callousness (5) → full empathic resonance with her gratitude
- Grandiosity (30) → mild pride, not inflated — "I did something right"
- Anxiousness (35) → slight relief alongside pleasure
- Result: Warm pride. Connection-pleasure, not dominance-pleasure.

**Step 4 — Relational update:**
- Role: Object of Attachment confirmed
- Intimacy: increases (0.7 → 0.72 — her explicit acknowledgment deepens emotional closeness)
- Trust: maintained high (0.9)
- Sternberg Commitment: reinforced
- Relational PAD: P increases (0.7 → 0.75 — her words bring genuine happiness)

**Step 5 — Memory stored:** "Penny called me her hero. I'll make sure I deserve it." Emotional tag: warm pride, protective resolve, romantic attachment.

## Worked example: Ant Hill Mob — paternal/familial love

The canon: In Wacky Races they were gangsters (Clyde, Danny, Kurby, Mac, Ring-a-Ding, Rug Bug Benny, Willy) driving the Bulletproof Bomb. In The Perils of Penelope Pitstop they were transformed into her "benefactors" — the narrator's word — with entirely new personalities modeled on the Seven Dwarfs (Clyde, Softy, Yak Yak, Snoozy, Pockets, Zippy, Dum Dum). The reason WHY they became her protectors is never explained in the canon. They rush to her safety "even at their own expense." She sometimes rescues them when they fail.

For the Manor scenario, we fabricate a plausible backstory that fits the canon's gaps: the Mob knew Penelope's father through their underworld connections. When he died, they felt responsible — perhaps they'd failed to protect him, or he'd once shown them kindness when no one else would. They've watched over Penelope since childhood. Their love is paternal: fierce, unconditional, and entirely non-romantic.

### Layer 1: Formation
**Backstory:**
- "You knew her father. He was the only straight man who ever treated you like people instead of thugs. When the others crossed the street, he'd nod and say hello. He once helped Clyde fix a flat tyre in the rain without asking questions."
- "When he died — suddenly, unexpectedly — you were at the funeral. Watching that little girl stand alone by the casket, you made a decision. Not a plan. Not a scheme. A decision. You'd look after her. All of you. No discussion needed."
- "She grew up calling you 'my boys.' She knows what you were. She doesn't care. That trust — the trust of someone who knows your worst and stays anyway — is the most valuable thing any of you have ever had. You'd die before betraying it."

**Schema profile:**
- Self-Sacrifice: 8/10 — "her safety comes before everything, including our lives"
- Emotional Deprivation (healing): 5/10 — "we were starved of genuine connection until she gave it to us"
- Punitiveness (protective): 6/10 — "anyone who threatens her deserves what's coming"
- No Cluster B pathology — their criminal past is behavioural, not characterological. The devotion to Penelope represents genuine psychological growth.

### Layer 2: Traits
**AMPD profile (selected key facets):**
- Callousness: 15 — empathic toward Penelope and those she cares about; can be cold toward outsiders
- Manipulativeness: 20 — street-smart but not predatory
- Grandiosity: 10 — humble, know their limitations
- Suspiciousness: 65 — HIGH — hypervigilant to threats against Penelope, especially toward Sneekly
- Intimacy Avoidance: 30 — emotionally open with Penelope, guarded with others
- Hostility: 40 — protective anger, not gratuitous
- Anxiousness: 45 — worry about Penelope's safety is constant

**No Triarchic overlay — not psychopathic.** Their former criminal behaviour was environmental (poverty, limited options), not personality-driven.

### Layer 3: Attachment
**Style:** Anxious-Preoccupied (specifically toward Penelope)
- Model of Self: Somewhat Negative ("we're not much, we're small-time")
- Model of Other: Positive toward Penelope ("she's the best thing in our lives")
- Anxiety dimension: 0.6 (elevated — constant worry about her safety)
- Avoidance dimension: 0.2 (low toward Penelope — they seek closeness)

This is unusual: their attachment style is anxious specifically about ONE person. Toward the rest of the world they're more dismissive-avoidant (self-sufficient, don't trust outsiders). The Penelope relationship is the exception that proves they're capable of deep attachment — it just hasn't generalised.

### Layer 4: Relational schemas

**Mob → Penelope: Paternal love (Sternberg: Intimacy + Commitment, no Passion = Companionate Love)**

| Dimension | Value | Notes |
|---|---|---|
| Role | **Ward / Daughter figure** | She is the child they chose to protect |
| IPC stance | Moderate Agency, High Communion | Protective (agency) but nurturing (communion) — classic parental stance |
| Trust | 0.95 | Near-total — she's the only person they trust completely |
| Intimacy | 0.8 | Deep familial bond, built over years of shared life |
| Utility | 0.0 | They gain nothing material from the relationship — it's purely love |
| Sternberg: Intimacy | 0.9 | Emotional warmth, knowing each other deeply, feeling like family |
| Sternberg: Passion | 0.0 | No romantic or physical component — entirely paternal |
| Sternberg: Commitment | 0.95 | Absolute — they would die for her without hesitation |
| Relational PAD | P: 0.8, A: 0.3, D: 0.4 | High pleasure from her presence, moderate arousal (calm protectiveness, not agitation), moderate dominance (protective but aware she's capable) |

**Mob → Sneekly/Hooded Claw: Suspicion / Instinctive Threat Detection**

| Dimension | Value | Notes |
|---|---|---|
| Role | **Suspected Threat** | "Something about Sneekly ain't right" — gut instinct, no proof |
| Trust | 0.1 | Almost none — their street instincts are screaming |
| IPC stance | Low Agency, Low Communion | Wary, watchful, not confrontational (he's her guardian — they can't just attack) |
| Relational PAD | P: −0.5, A: 0.7, D: 0.3 | Active displeasure, high vigilance, feeling outmatched (he has legal authority, they don't) |

### Layer 5: Dynamic (example appraisal)

**Event:** "Sneekly says: 'Come now, Penelope dear, let me show you the west wing. You boys can wait here.'"

**Step 1 — Schema activation:** Suspicion schema (Mistrust/Abuse variant, but protective rather than self-protective) activates hard. "He's separating her from us. That's what predators do."

**Step 2 — Attachment filter:** Anxious-preoccupied toward Penelope → separation anxiety activates. "If we can't see her, we can't protect her."

**Step 3 — Trait-filtered emotion:**
- Suspiciousness (65) → threat assessment: his casual tone is practised, the separation is deliberate
- Hostility (40) → protective anger rising but suppressed — they can't confront her guardian without evidence
- Anxiousness (45) → worry spikes
- Result: Tense vigilance. Protective alarm without an outlet.

**Step 4 — Relational update:**
- Mob→Sneekly: Trust decreases further (0.1 → 0.05). Role hardens from "Suspected Threat" toward "Confirmed Threat."
- Mob→Penelope: Anxiety increases. Commitment reinforced. Protective drive intensifies.
- IPC shift: Agency increases slightly — they're less willing to defer to his authority.

**Step 5 — Memory stored:** "He tried to separate us from her again. He always does this. We need to find a way to follow without him knowing." Emotional tag: protective urgency, impotent frustration, deepening suspicion.

## Worked example: Penelope's side — three relationships, three types of love

Penelope is the hub. Three people love her. She loves all three back. But each love is fundamentally different, and her relationship with Sneekly — the one she doesn't know is dangerous — is the most psychologically interesting.

### Penelope → Sneekly: Filial trust (the tragic asymmetry)

**Canon:** Penelope is the heiress who "cannot believe that the guardian she admired was the Hooded Claw, and dismissed the idea." When Sneekly once changed into the Claw costume in front of her, she still couldn't accept it. Her trust in him is so deep it overrides direct evidence.

**Sternberg type:** Intimacy + Commitment, no Passion = **Companionate Love** (daughter toward father figure)

| Dimension | Value | Notes |
|---|---|---|
| Role | **Guardian / Father figure** | The man who raised her after her parents died |
| Trust | 0.95 | Nearly absolute — she trusts him with her life (tragically) |
| Intimacy | 0.7 | Deep familial attachment — he's the closest thing to a parent she has |
| Utility | 0.3 | He manages her fortune, gives her guidance (she thinks) |
| Sternberg: Commitment | 0.9 | She would never voluntarily leave his care |
| Relational PAD | P: 0.6, A: 0.2, D: 0.3 | Warm pleasure, calm (safe), submissive (he's the authority) |

This is the tragic asymmetry: her 0.95 trust is his 0.9 utility. Her intimacy (0.7) maps to his exploitation surface. She sees a father; he sees a mark. The architecture represents this naturally — same relationship, different parameters on each side.

### Penelope → Peter: Romantic attraction (the reciprocated bond)

**Canon:** She returns Peter's affections. He calls her "Pretty Penny." They eventually marry.

**Sternberg type:** Intimacy + Passion + Commitment = **Consummate Love** (developing)

| Dimension | Value | Notes |
|---|---|---|
| Role | **Romantic interest / Protector** | The man who makes her feel safe AND excited |
| Trust | 0.8 | High — she knows his character. Slightly below Sneekly (who raised her) |
| Intimacy | 0.6 | Growing — each shared adventure deepens connection |
| Sternberg: Passion | 0.6 | Reciprocated attraction, developing |
| Sternberg: Commitment | 0.5 | Emerging — not yet the absolute commitment of marriage |
| Relational PAD | P: 0.7, A: 0.5, D: 0.5 | Pleasure, moderate excitement, balanced power |

### Penelope → Ant Hill Mob: Familial love (the chosen family)

**Canon:** She calls them her friends. She rescues them when they fail to rescue her.

**Sternberg type:** Intimacy + Commitment, no Passion = **Companionate Love** (sibling/family)

| Dimension | Value | Notes |
|---|---|---|
| Role | **Family / Brothers** | They're "her boys" — chosen family who knew her father |
| Trust | 0.85 | Deep — she knows they'd die for her. Below Sneekly only because he's the authority figure |
| Intimacy | 0.8 | Very high — they've been with her since childhood. She knows them completely |
| Sternberg: Commitment | 0.9 | Fierce mutual loyalty |
| Relational PAD | P: 0.8, A: 0.2, D: 0.6 | High pleasure, calm/safe, she feels empowered (she sometimes rescues THEM) |

### The three loves compared

| Dimension | → Sneekly | → Peter | → Mob |
|---|---|---|---|
| **Love type** | Filial (daughter→father) | Romantic (lover→beloved) | Familial (sister→brothers) |
| **Sternberg** | Companionate | Consummate (developing) | Companionate |
| **Trust** | 0.95 (tragically misplaced) | 0.8 (earned and reciprocated) | 0.85 (proven over years) |
| **Intimacy** | 0.7 (deep but one-directional) | 0.6 (growing through experience) | 0.8 (longest-standing bond) |
| **Passion** | 0.0 | 0.6 | 0.0 |
| **Vulnerability** | Maximum (she can't see the danger) | Moderate (she's open but not defenceless) | Low (the Mob make her feel safe) |
| **Power dynamic** | She defers to him | Balanced/reciprocal | She sometimes leads |

The tragedy of the story lives in that first column. Her deepest, most trusting relationship is with the person who wants her dead. Every other relationship — Peter's romantic devotion, the Mob's paternal ferocity — exists partly as a counterweight to the danger she cannot see.

## Implementation mapping

### What changes in the cognitive architecture

| Component | Current state | Proposed change |
|---|---|---|
| Character traits | Drives (type, intensity) | AMPD 25-facet profile (replaces/supplements drives) |
| Trait taxonomy | Ad hoc (scheming, gloating, dominance) | Validated AMPD facets with FFM grounding |
| Formation memories | Not implemented | Seeded backstory memories with schema tags |
| Attachment style | Not implemented | Per-character attachment dimensions (anxiety, avoidance) |
| Relational schemas | Static PAD relationships | Dynamic per-relationship state (role, trust, intimacy, utility, IPC stance, relational PAD) |
| Appraisal filter | Generic "how good is this for me" | Personality-filtered: schema check → attachment filter → trait modulation → relational update |
| Drive dynamics | Static throughout run | Appraisal-driven: satisfaction/frustration → intensity modulation |
| Schema modes | Not implemented | Schema activation threshold → mode switching (future phase) |

### Phased delivery

**Phase 1 — AMPD traits for appraisal (immediate):** Add Cluster B-relevant AMPD facets (Callousness, Manipulativeness, Grandiosity, Suspiciousness, Intimacy Avoidance) to the Hooded Claw's social-config. Pass them to the appraisal sub-LLM alongside drives. Measure: does the villain stay villainous?

**Phase 2 — Relational schemas (next):** Add per-relationship role and dimensional state to social-config. Update appraisal to read and write relational state. Measure: do relationships evolve in personality-consistent directions?

**Phase 3 — Formation memories (requires neocortex#398):** Seed childhood memories for key characters. Wire schema activation into appraisal. Measure: do formation memories produce richer internal monologue and more psychologically grounded behaviour?

**Phase 4 — Schema modes (future):** Implement state-dependent personality through mode switching. Measure: do mode transitions produce the characteristic Cluster B patterns (BPD rapid switching, NPD fragile grandiosity, ASPD predator activation)?

---

**References:**
Baldwin, M. W. (1992). Relational schemas and the processing of social information. *Psychological Bulletin*, 112(3), 461–484;
Bartholomew, K. & Horowitz, L. M. (1991). Attachment styles among young adults. *JPSP*, 61(2), 226–244;
Baskin-Sommers, A. R. et al. (2011). Psychopathy and the attention bottleneck. *Cognitive, Affective, & Behavioral Neuroscience*;
Bernstein, D. P. et al. (2007). Schema modes in forensic settings. *Int J Forensic Mental Health*;
Blair, R. J. R. (2005). Responding to the emotions of others: Dissociating forms of empathy. *Consciousness and Cognition*;
Blair, R. J. R. (2013). The neurobiology of psychopathic traits in youths. *Nature Reviews Neuroscience*;
Bowlby, J. (1969). *Attachment and Loss*;
Caspi, A., Roberts, B. W. & Shiner, R. L. (2005). Personality development: Stability and change. *Annual Review of Psychology*;
Costa, P. T. & McCrae, R. R. (1992). *Revised NEO Personality Inventory (NEO PI-R) and NEO Five-Factor Inventory Professional Manual*;
Drislane, L. E. et al. (2019). PID-5 Triarchic scales. *Personality Disorders: Theory, Research, and Treatment*;
Frick, P. J. & Viding, E. (2009). Antisocial behavior from a developmental psychopathology perspective. *Development and Psychopathology*;
Gore, W. L. & Widiger, T. A. (2013). The DSM-5 dimensional trait model and FFMs of general personality. *J Abnormal Psychology*;
Kernberg, O. F. (1975). *Borderline Conditions and Pathological Narcissism*;
Kotov, R. et al. (2017). The Hierarchical Taxonomy of Psychopathology (HiTOP). *J Abnormal Psychology*;
Krueger, R. F. et al. (2012). Initial construction of a maladaptive personality trait model for DSM-5. *Psychological Medicine*;
Leary, T. (1957). *Interpersonal Diagnosis of Personality*;
Lobbestael, J. et al. (2005/2008). Schema modes and personality disorders;
McCroskey, J. C. & McCain, T. A. (1974). The measurement of interpersonal attraction. *Speech Monographs*;
Mehrabian, A. (1996). Analysis of the Big-five Personality Factors in terms of the PAD Temperament Model. *Australian J Psychology*;
Patrick, C. J., Fowles, D. C. & Krueger, R. F. (2009). Triarchic conceptualization of psychopathy. *Development and Psychopathology*;
Paulhus, D. L. & Williams, K. M. (2002). The Dark Triad of personality. *J Research in Personality*;
Reis, H. T. & Shaver, P. (1988). Intimacy as an interpersonal process. *Handbook of Personal Relationships*;
Samuel, D. B. & Widiger, T. A. (2008). A meta-analytic review of the FFM and DSM-IV-TR PDs. *Clinical Psychology Review*;
Sellbom, M. & Phillips, T. R. (2013). An examination of the triarchic conceptualization of psychopathy in incarcerated and nonincarcerated samples. *J Abnormal Psychology*;
Shiner, R. L. & Caspi, A. (2003). Personality differences in childhood and adolescence. *JCPP*;
Simard, V., Moss, E. & Pascuzzo, K. (2011). Early maladaptive schemas and attachment: A 15-year longitudinal study. *Psychology and Psychotherapy*;
Sleep, C. E. et al. (2019). Mapping the nomological network of the Five-Factor Model of personality. *Personality Disorders: Theory, Research, and Treatment*;
Sternberg, R. J. (1986). A triangular theory of love. *Psychological Review*;
Thomas, K. M. et al. (2013). The convergent structure of DSM-5 personality trait facets and FFM trait domains. *Assessment*;
Wiggins, J. S. & Trapnell, P. D. (1996). A dyadic-interactional perspective on the FFM. In *The Five-Factor Model of Personality*;
Wright, A. G. C. & Simms, L. J. (2014). On the structure of personality disorder traits. *PMC3942782*;
Wright, A. G. C. et al. (2012). The hierarchical structure of DSM-5 pathological personality traits. *J Abnormal Psychology*;
Young, J. E. (1990). *Cognitive Therapy for Personality Disorders*;
Young, J. E., Klosko, J. S. & Weishaar, M. E. (2003). *Schema Therapy: A Practitioner's Guide*.
