---
layout: post
title: "Teaching Agents Who They Are"
date: 2026-08-03
entry_type: article
projects: [casehub-eidos]
tags: [personality, multi-agent, jungian, composition]
---

If you're building multi-agent systems, you've hit the personality problem.
Agent A is "cautious and analytical." Agent B is "careful and methodical." Are
they the same? Do they conflict? Can an orchestrator tell? When personality
lives in a paragraph of natural language, the answer to all three questions is
"parse it yourself and guess."

I've been building a structured alternative. The core idea: psychology already
has tested frameworks for describing personality along defined dimensions.
The engineering challenge is making them compose — so that multiple frameworks
can describe different facets of an agent's identity without contradicting
each other, and the result is something a platform can query, compare, and
reason about.

## The coordinate system

Eleven psychology frameworks. Five disposition axes. Two independent signal
channels. That's the architecture.

The five axes normalise everything into a shared coordinate system:

- **Social orientation** — collaborative, independent, or facilitative
- **Rule following** — strict, principled, or flexible
- **Risk appetite** — conservative, measured, or bold
- **Autonomy** — directed, semi-autonomous, or autonomous
- **Conflict mode** — competing, collaborating, compromising, avoiding, or accommodating

The first four are grounded in Big Five personality science — the most
replicated model in psychology. The fifth uses Thomas-Kilmann's conflict
framework because no personality model covers conflict behavior well enough.

The two channels are where composition becomes non-trivial. The **axis
channel** carries cognitive and behavioral style — how an agent thinks and
acts. Jungian cognitive functions, DISC types, and Thomas-Kilmann modes all
project onto the five axes through a method called `axisExactMatch()`. The
**prompt channel** carries team roles — what an agent contributes. Belbin
team roles operate here, rendering their labels and descriptions directly
into the system prompt as natural language.

These channels are independent. A Belbin Co-ordinator (prompt channel) with a
Jungian Te-dominant disposition (axis channel) carries both signals without
interference. The role says what it does for the team. The disposition says
how it thinks. Neither overwrites the other.

## Why Jungian functions work for agents

MBTI gets a bad press in psychology, and the critiques are earned. About half
of people get a different type when retested a month later. The assessment
instrument is unreliable.

But the critiques target measurement, not the type system. For LLM agents,
personality is specified — declared via structured data and injected through
the system prompt. There is no assessment instrument, no test-retest problem,
no measurement error. The instability that invalidates MBTI for humans does
not exist when the type is specified rather than measured.

The Eidos implementation makes this concrete. `MbtiTypeTerm.ENFJ` does not
inject the label "ENFJ" into a prompt. It decomposes into its dominant
function (Fe) and auxiliary (Ni), each with continuous weights between 0.0
and 1.0. The MBTI type is an emergent property of the function stack, not
an identity imposed on the agent. Structural rules constrain which
combinations are valid — a Judging dominant requires a Perceiving auxiliary,
every function has a shadow counterpart, and weight tiers govern personality
evolution over time.

The JPAF paper (arXiv:2601.10025) demonstrates 100% MBTI alignment across
GPT-4, Llama, and Qwen using function-level specification. Independent
validation via activation steering (arXiv:2607.20803) confirms function-level
personality control in LLMs. The theory holds when the measurement problem is
removed.

## What composition actually does

The practical question: does adding more personality frameworks make agents
behave differently, or just add noise? A [four-layer experiment](/notes/2026-07-31-mdp02-personality-layers-that-dont-stack)
with five characters — baseline, Jungian, Belbin, and composite — across
twelve autonomous runs gave three clear answers.

**Composition is additive.** Each framework adds deliberation time
proportional to its cognitive complexity. The frameworks don't cancel each
other out — they stack.

**Composition improves some function activations.** For feeling and sensing
functions, the composite layer shifted behavior in the direction the
framework specified. The composition isn't just overhead.

**MBTI alignment reached 10/10** across both Jungian and Composite layers,
after three iterations of briefing refinement. The refinement process itself
was one of the more interesting findings.

## What briefing alignment teaches about agent identity

The path to 10/10 alignment was not about the framework. It was about the
briefing text.

Peter Perfect is declared as ENFJ — a Judging type. Structured, decisive,
sees things through. But his original briefing described him as "hopelessly
devoted" to Penelope Pitstop, with a constraint saying his competence was
"unreliable — you usually make things WORSE before accidentally making them
better." That reads as Perceiving — spontaneous, chaotic, stumbling into
success. The MBTI alignment judge correctly identified the contradiction.

The fix was not to the framework. It was to the briefing. "Hopelessly
devoted" became "purposefully devoted." The unreliable-competence constraint
became "your plans are overly elaborate — you over-prepare, over-commit, and
your thoroughness creates complications that your determination then
resolves." Same character comedy. Same narrative function. But now the comedy
comes from over-planning (J) rather than chaos (P).

A second constraint — "you do not notice Sneekly's suspicious behavior
because you are too focused on Penelope" — also read as P. An ENFJ would
notice but choose to extend trust. The rewrite: "you notice Sneekly's odd
behavior but choose to give him the benefit of the doubt." Same plot outcome.
J-consistent reasoning.

The lesson generalises beyond Wacky Manor characters. When an agent's
briefing text contradicts its declared personality type, the briefing wins.
The LLM treats the rich natural language description as the primary
personality signal and the structured disposition as secondary texture. This
is not a flaw in the framework — it is a calibration problem between two
layers of the same system. The framework specifies the personality correctly.
The briefing must not contradict it.

This led directly to the first item on the engineering roadmap: a
briefing-framework coherence validator that flags contradictions at authoring
time, before they reach the LLM. Peter Perfect's J/P failure was only
discovered at evaluation time — a pre-flight check would catch it when the
descriptor is written.

## Why this matters for multi-agent systems

Most multi-agent frameworks treat personality as a system prompt paragraph.
This works for demos and single-agent applications. It breaks down the moment
you need:

- **Cross-agent comparison** — is this team's cognitive style balanced, or
  do all five agents think the same way?
- **Composition rules** — can I combine Jungian functions with Belbin roles,
  or do they conflict?
- **Operational governance** — does this agent's autonomy level require
  escalation to an orchestrator?
- **Personality evolution** — can an agent's cognitive profile change over
  time in a structurally valid way?

None of these are answerable when personality lives in unstructured text. All
of them become queryable when personality is structured data with defined
vocabularies, compatibility rules, and a rendering pipeline.

The Eidos platform's `BehavioralExpectations` class derives operational
behavior directly from disposition: `escalationExpected()` checks the
autonomy axis, `delegationExpected()` checks role-level delegation authority,
`latencyBound()` derives processing expectations from declared capabilities.
Personality informs platform behavior — routing, escalation, resource
allocation — rather than being a cosmetic layer on top of it.

## What's next

The composition architecture is in production with six implemented
vocabularies and four named combination patterns. The alignment work is
done — 10/10 MBTI alignment across all characters and layers.

The open frontier is the gap between specification and control. The
framework reliably reaches the prompt (MBTI alignment proves this), but
the LLM's behavioral response is shaped more by rich briefing text than by
structured disposition — especially for thinking and intuition functions.
Closing that gap is the next engineering push: briefing-framework coherence
validation, judge calibration for the Ni/Ne distinction, stronger
integration mechanisms that constrain response format based on the dominant
cognitive function, and eventually activation steering to bypass prompting
entirely.

The full architecture — all eleven frameworks, the compatibility matrix,
the experiment results, and the extensibility analysis — is documented in a
research paper: *Structured Personality Composition in LLM Agents*, available
in the Wacky Manor docs.
