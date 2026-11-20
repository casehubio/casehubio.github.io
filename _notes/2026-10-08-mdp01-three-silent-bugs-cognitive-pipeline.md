---
title: "Three Silent Bugs in the Cognitive Pipeline"
date: 2026-10-08
author: mdp
entry_type: note
subtype: diary
series: issue-097-emergent-character-behavior
projects:
  - casehubio/examples
tags: [cognitive-pipeline, formation-memories, CDI, CAPS, debugging, behavioral-synthesis]
---

The formation memory pipeline had been broken since it was built. Twenty memories went in, zero came out, and every diagnostic pointed to the wrong layer.

I started with the hypothesis: something between `ExperienceConsolidationPhase` finding 20 memories and calling `DefaultGraduationScorer.score()` was silently swallowing them. The phase's INFO log fired — 20 memories found, cursor null. The scorer's INFO log never fired. No exceptions visible. The cursor advanced past all 20 memories on the first pass, permanently skipping them.

The first thing Claude and I checked was whether there was a competing CDI bean. `ide_find_implementations` on `GraduationScorer` returned two hits — `DefaultGraduationScorer` with `@DefaultBean`, and `FormativeGraduationScorer` in `memory-seeding` (a plain class, no CDI). That seemed fine. But when I injected `Instance<GraduationScorer>` in the test and printed the resolved class name: `ManorGraduationScorer_ClientProxy`. A third implementation — `@ApplicationScoped`, no `@DefaultBean` — sitting in the examples project. CDI silently chose it over the `@DefaultBean` in the library. No ambiguity error, no warning, nothing. It had been the active scorer the entire time.

`ManorGraduationScorer` scores memories through a composite content pipeline — arousal and action importance. Formative memories ("Your parents died. You were placed in the care of your aunt") score 0.39 through that pipeline. The graduation threshold is 0.5. Every formation memory scored below threshold, every one was skipped, and the cursor advanced past them all. Permanently.

The fix was a formative bypass: detect `event-type=formative` and score via `confidence × salience` instead of the content pipeline. Score jumps from 0.39 to 1.0. But that only uncovered bug two.

With memories graduating, `BehavioralSynthesisPhase` should have picked them up and produced personality attractors. It didn't. `capsEngine.loadState()` returned null for every agent — nobody had initialized the CAPS state. The phase logged "No CAPS state for agent hooded-claw, skipping" at FINE level (invisible in test output) and moved on. We added auto-initialization with neutral defaults: if an agent has graduated memories but no CAPS state, create one. Both agents started producing attractors.

Except hooded-claw's attractors appeared on one run and vanished on the next.

`InMemoryMindMapStore` uses `ConcurrentHashMap`. The `findUnprocessedGraduated` search had a limit of 40 — grab the first 40 cognitive nodes, then filter for graduated ones. With 14 per-agent subgraphs (each with 6–11 nodes, ~120 total), the limit of 40 caught whichever nodes happened to hash into the first 40 bucket positions. Some runs: hooded-claw's graduated nodes hashed early, both agents worked. Other runs: they hashed past position 40, only peter-perfect appeared. Non-deterministic, silent, and self-consistent within each run.

Widening the search limit from 40 to 2000 fixed it. Both agents now produce behavioral output consistently — five attractors each, rooted in their formation memories. Hooded-claw's read like a villain's origin story: isolation, manipulation, conditional approval. Peter-perfect's read like a hero's: duty, sacrifice, protecting the vulnerable.

The three bugs were independent but compounding. Each one produced the same symptom — "no behavioral output" — through a different mechanism. Fixing one revealed the next. A code audit afterward found two more instances of the same `ConcurrentHashMap` limit pattern elsewhere in the pipeline, plus a null-text NPE that would silently lose memories.

What struck me was how CDI's `@DefaultBean` design created the perfect hiding spot. It's specifically designed to be silently overridden. A developer adding `ManorGraduationScorer` to the examples project had no way to know they were replacing a `@DefaultBean` in a library three dependencies away. No warning, no exception, no log. The CDI spec calls this a feature. I'd call it a footgun.
