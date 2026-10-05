---
layout: post
title: "The Subconscious Call — Cognitive Appraisal Theory for LLM Character Agents"
date: 2026-10-05
entry_type: note
subtype: diary
projects: [casehubio/examples]
author: mdp
tags: [emergence, cognitive-appraisal, lazarus, scherer, frijda, sub-llm, wacky-manor, drives, emotion]
series: issue-097-emergent-character-behavior
---

# The Subconscious Call — Cognitive Appraisal Theory for LLM Character Agents

*Continues from [What LLM Characters Actually Need](2026-10-04-mdp01-what-llm-characters-actually-need.md).*

The previous entry documented the ceiling: monolithic drive descriptions hit a wall where fixing one drive's emotional expression regresses another. Drives were doing triple duty — defining what the character perceives, how they appraise it, and what they feel — and the LLM couldn't manage all three simultaneously. The mean score plateaued at 4.06 with 4 of 17 drives regressing.

That ceiling pointed toward the academic literature on how emotions actually work in humans. The answer has been sitting in cognitive psychology since 1966.

## Lazarus and the separation of appraisal from feeling

Richard Lazarus's cognitive appraisal theory makes a distinction that maps directly onto our problem. Emotion isn't a reaction to a stimulus — it's the output of an appraisal process. You see a bear, you evaluate whether it threatens your goals (primary appraisal), you evaluate whether you can cope (secondary appraisal), and *then* you feel fear. The fear is downstream of the appraisal, not parallel to it.

Our drive descriptions were collapsing this sequence. "You feel contemptuous glee when outsmarting someone" bundles the appraisal ("is this situation relevant to my scheming drive?") with the feeling ("contemptuous glee") into a single static instruction. The LLM has no room to do the appraisal work itself — the conclusion is already written.

Lazarus also gives us core relational themes: anger is "a demeaning offense against me," guilt is "having transgressed a moral imperative," pride is "enhancement of ego-identity by valued achievement." Each emotion has a specific person-environment relationship. These aren't prescriptions — they're the cognitive structure that produces the emotion. Give the LLM the structure, and the emotion emerges from the situation.

## Scherer's four checks, and why we don't implement them separately

Klaus Scherer's Component Process Model breaks appraisal into four sequential Stimulus Evaluation Checks: relevance detection (is this novel, pleasant, goal-relevant?), implication assessment (what caused it, what are the consequences?), coping potential (can I control this, can I adjust?), and normative significance (does this violate my standards?). The pattern of results across all four determines which emotion emerges — not a single check, but the configuration.

The temptation is to implement each SEC as a separate evaluation. I nearly went down that path. But our own experimental findings argued against it. Principle 11 from the wacky-manor experiments: analytical multi-step instructions kill emotional expression. A five-step "review your mental model, then check your voice, then assess the situation" instruction dropped optimism expression to 9.1%. More structure produced worse emotional output, not better.

Scherer's framework is a theoretical lens for understanding appraisal, not an implementation specification. An LLM asked "what are you feeling right now?" implicitly traverses all four dimensions — it evaluates relevance, considers implications, assesses its capacity to respond, and checks against norms, all within its reasoning. Making this explicit doesn't improve the output. It constrains it.

## Frijda and action readiness

Nico Frijda's contribution is that emotions aren't just feelings — they're states of action readiness. Anger produces approach and antagonism. Fear produces avoidance. Protection produces shielding behaviour and narrowed attention to the threat source. The emotion doesn't just colour the character's inner monologue — it shapes what they want to *do*.

This maps onto a gap in our current system. The L1 instruction ("name your feeling") gets the character to identify an emotion, but it doesn't connect that emotion to a behavioural tendency. A character who feels protective urgency should be *moving toward* the threat, not just noting that they feel concerned. Frijda's action readiness gives us the link from appraisal → feeling → behaviour, which is the full causal chain.

## The Chain-of-Emotion paper

Qu et al.'s 2024 PLOS ONE paper, "Chain-of-Emotion," validated experimentally what Lazarus theorised: a separate LLM appraisal call before response generation outperforms baseline on believability in game agents. Their architecture interposes an emotion-reasoning step between perception and action. The agent first reasons about its emotional state given the situation, then responds from that state. Believability scores improved significantly.

This gave us confidence that the sub-LLM pattern isn't just theoretically sound — it's been validated in a comparable domain. The question was how to apply it within our constraints.

## The constraints that shaped the architecture

Six phases of wacky-manor experiments produced 38 design principles. The ones that constrained the appraisal architecture most tightly:

**Principle 9 — structured fields create satisficing shortcuts.** When you give an LLM a structured form to fill ("EMOTION: ___, DRIVE: ___, INTENSITY: ___"), it treats form-filling as task completion. It writes plausible answers and then ignores them in the actual response. Any architecture that outputs structured appraisal results risks this.

**Principle 11 — analytical instructions kill emotional expression.** This is the strongest negative result from the experiments. Five analytical reasoning steps produced 81.8% objective consistency but catastrophic 9.1% optimism expression. The model can't be both analyst and character simultaneously in the same thinking field.

**Principle 14 — less instruction produces better results.** Two evocative sentences outperformed five procedural steps on every metric.

**Principle 37 — monolithic descriptions hit a ceiling.** Fixing one drive's description regresses another. The emotional-core reframing experiment recovered 2 of 4 regressed drives but introduced new regressions elsewhere. This is the architectural problem, not a tuning problem.

These constraints seem contradictory. We need analytical appraisal work (Lazarus, Scherer) but can't put analytical instructions in the main agent (P11). We need the appraisal to produce structured results for the mood system (PAD values, OCC types) but can't show structured fields to the LLM (P9). We need richer context but less instruction (P14).

## The insight: P11 has a scope boundary

The breakthrough came from asking a precise question: does Principle 11 apply to *all* LLM calls, or specifically to the main agent's thinking field?

P11 says analytical instructions kill emotional expression. But the emotional expression happens in the character's thinking field — the space where the agent reasons as the character, names feelings, inhabits the role. A separate sub-LLM call that runs *before* the character agent is not the character. It's pre-processing. It can be as analytical as needed without affecting the character's expressive capacity.

This is the separation of concerns that Lazarus's theory implies. Primary and secondary appraisal are pre-conscious processes — they happen before the emotional experience, not during it. The sub-LLM is the subconscious. The main LLM is conscious experience.

Similarly, P9 targets structured *fields* — JSON-like dimensions that the LLM fills as pattern completion. But narrative text isn't a structured field. If the sub-LLM outputs "You feel the pull of a plan forming — your self-preservation is alert, Hartwell is watching" rather than `{activated_drives: ["scheming", "self-preservation"]}`, the satisficing shortcut doesn't trigger. The main LLM responds to a felt state, not a form to fill.

Two new design principles emerged from this:

**Principle 39** — P11 applies to the main agent, not to sub-LLMs. Analytical prompts are safe in pre-processing.

**Principle 40** — Evocative narrative output avoids P9 satisficing. Narrative is not structured fields.

## Testing it: three scenarios, three characters

I set up an adversarial debate — three separate Claude instances reading the research docs and codebase independently, each arguing from a different angle (theory, experimental evidence, existing code). They converged on the sub-LLM approach. Then we tested it with three ad-hoc scenarios, comparing a control condition (full drive descriptions with L1 instruction) against the treatment (minimal drives — type and intensity only — plus a Haiku appraisal section injected as a cognitive observation).

The sub-LLM prompt was analytical: "For each drive, consider — does this situation touch it? How? What does it make the character want to do?" The output instruction was evocative: "Respond with a 2-3 sentence felt-state in first person — gut reaction, internal sensation, not analysis."

**Peter Perfect entering a conservatory where Clara is about to step into electrified water, with Pemberton watching from the doorframe.** Three drives: gallantry (0.9), proving-worth (0.7), protection (0.8). In the control condition, protection dominated completely — the character lunged to save Clara, noted Pemberton's suspicious smile, felt righteous urgency. In the treatment condition, the character still lunged — but the thinking field contained something richer: "shamefully, desperately — it's the wanting him to *see* me do it." The proving-worth drive surfaced alongside protection. The character was aware of wanting to be witnessed, and the tension between selfless heroism and the desire for recognition made the response more authentic, not less.

**Hooded Claw in a library with a locked cabinet and Hartwell watching.** Four drives: scheming (0.9), self-preservation (0.7), dominance (0.6), gloating (0.7). Both conditions expressed all four drives, but the treatment made the internal conflict between gloating and self-preservation sharper: "The gloat wants to rise... but the survival instinct is screaming: *not yet, not while his eyes are on you*." The drives were in explicit tension rather than coexisting peacefully.

**Penelope Pitstop entering a room where two characters are mid-argument, with a puzzle box on the table.** Three drives: curiosity (0.7), social-harmony (0.8), adventure (0.6). Both conditions produced comparable results — all three drives expressed, similar quality. The situation naturally activated all drives, so the appraisal section added little.

**Principle 41** emerged from this pattern: the sub-LLM appraisal's value is highest for non-obvious drive activations. When the situation naturally surfaces every drive, the appraisal is redundant. When secondary drives are subtle — proving-worth in a danger scenario, where the dominant response is protection — the appraisal surfaces what the character would otherwise skip. The marginal value is on the drives that need help, not the ones that fire easily.

## The architecture

The validated architecture is simple. One Haiku sub-LLM call per turn, running before the main agent invocation:

The sub-LLM receives the character's drives (type and intensity only — no descriptions), the raw observation, current mood, and disposition axes. Its prompt is analytical: evaluate each drive against the situation. Its output is a 2-3 sentence evocative narrative — the character's felt state as a gut reaction. This output is injected as a cognitive section in the observation, with the header "What You're Feeling."

The main LLM receives this section alongside mood, beliefs, and norms. The L1 instruction stays unchanged — "What are you FEELING right now? Name it. Think AS your character, not ABOUT your character." The character inhabits the pre-appraised state rather than having to both appraise and feel simultaneously.

Post-processing extracts structured data (OCC emotion type, drive activations, PAD values) from the thinking field after generation — invisible to the main LLM, feeding the mood system and tracing.

No emotion knowledge base. Haiku's training data already contains Lazarus, Scherer, and Frijda. We'll add a retrieval layer only if measurement shows specific pattern gaps where the sub-LLM's implicit knowledge falls short.

No instruction changes. L1 is empirically validated. The observation change is the only treatment variable.

No separate SEC evaluations. One call covers all four Scherer dimensions implicitly — **Principle 42**: one sub-LLM call covers all 4 SECs, and separating them reproduces the P11 failure at the architecture level.

## What this opens up

The prototype goes into the wacky-manor agent loop next. Strip drive descriptions to type and intensity, add the Haiku call, run the standard 300-event eval. The bar to clear is L1's 4.06 mean score.

If the numbers hold, the abstraction moves to the neocortex platform as an `AppraisalOrchestrator` — a reusable cognitive component that any character agent can use. The same separation (sub-LLM appraisal → evocative injection → main agent inhabits) applies to any domain, not just a murder mystery in a manor house.

The deeper question is whether this architecture scales to richer cognitive processes. Memory-seeded appraisal — where biographical memories feed into the sub-LLM alongside drives — would let the character's past shape their present emotional response without explicit behavioural prescriptions. A memory of being powerless changes the appraisal of a dominance opportunity. That's Lazarus's reappraisal: emotional meaning changes as context changes.

The academic literature gave us the architecture. The experiments told us where the boundaries are. The sub-LLM is the bridge — analytical where it needs to be, evocative where it must be, and invisible to the character who benefits from it.
