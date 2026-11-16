---
layout: post
title: "Extracting the Cognitive Builder"
date: 2026-10-26
entry_type: note
subtype: diary
projects: [casehubio/neocortex, casehubio/examples]
tags: [cognition, refactoring, observation-rendering, eval-migration]
---

# Extracting the Cognitive Builder

`CharacterCognition` had been accumulating responsibilities since Phase 0. It started as a thin wrapper around drives and social config — the thing that turned YAML character definitions into observation sections for the LLM prompt. Over the months it absorbed every new cognitive rendering capability: Theory of Mind readings, CAPS-filtered drive appraisal, attractor rendering, priority signatures, differentiation texts, situation-primed memories. By the time I looked at it today, it was 985 lines of interleaved concerns — generic cognitive rendering tangled with manor-specific social config.

The trigger wasn't the size. It was the eval test migration. We'd moved the zero-dependency and fixture-dependency eval tests to neocortex in #699 and #700, but ten tests remained stuck in wacky-manor because they needed `CharacterCognition` to render observations. These tests evaluate neocortex cognitive capabilities — drive discrimination, attractor influence, priority signature effects — but they couldn't move because the rendering logic was locked inside a class that imports `ActionType`, `CharacterState`, and `ManorSocialConfigLoader`.

The extraction question was which methods are genuinely generic and which are manor-specific. I mapped every method:

**Generic** — works entirely with neocortex types (MindMapStore, CognitionCore, SituationClassifier, DriveSnapshot): `renderInternalState`, `renderTheoryOfMind`, `renderAttractorAsFelt`, `readDriveDifferentiation`, `readSelfStandards`, `readPrioritySignatures`, `selectSituationPrimedMemories`, `appraisalFilterDrives`, `resolveAdaptedDrives`, `classifySituation`, `renderCognitionCoreSections`, `renderTaggedSections`, `observeCharacterState`.

**Manor-specific** — depends on `SocialConfig`, `ManorContextStrategy`, `TrustEvolutionConfig`, or agency types: tendencies, beliefs (the agency `beliefsSection` API), norms, social awareness, personality facets, formation memories, attachment style, relational schemas, trust rendering, persona activation.

The generic methods accounted for roughly 70% of the code.

The issue proposed inheritance with a type parameter — `CognitionObservationBuilder<C extends CharacterContext>`. I went with composition instead. The type parameter would have carried a single method (`situationText()`), which doesn't justify the abstraction. Composition gives cleaner boundaries: the builder is independently instantiable (which is exactly what the eval tests need), and `CharacterCognition` composes it with manor-specific sections in its own `renderCognitiveSections()`.

The one design wrinkle was the `ObservationSection` type split. Neocortex has its own sealed interface at `io.casehub.neocortex.cognition.prompt.observation.ObservationSection`. Agency has a parallel one at `io.casehub.agency.perception.observation.affordance.ObservationSection`. Same factory methods — `text(header, content)`, `items(header, emptyMessage, items)` — but different types. The builder uses the neocortex type. `CharacterCognition` converts at the boundary with a `toAgency()` switch expression. Not elegant, but it means zero changes to any caller.

`SocialConfig.Drive` was the other boundary type. The builder works with `DriveSnapshot` — a neocortex record with the same fields (type, intensity, description, context). `CharacterCognition` converts at construction time via `toDriveSnapshots()`. The conversion is a one-liner per drive, and it runs once.

After the extraction: `CharacterCognition` dropped from 985 to 532 lines. `CognitionObservationBuilder` is 370 lines in neocortex. `DriveSnapshot` and `RegulationTracker` (the generic trajectory tracking, minus the manor-specific `classifyBehavior`) are another 50 lines.

I migrated `DriveRenderingUnitTest` as the first proof — 13 deterministic scenarios using the builder directly, no `CharacterCognition`, no `CharacterState`, no `SocialConfig`. It loads drive data from `EvalCharacterLoader.loadGeneric()` and seeds an `InMemoryMindMapStore` with profiles. All 13 scenarios pass with identical results.

The remaining nine tests are harder. They all call `renderCognitiveSections()`, which produces a full prompt including manor-specific sections — tendencies, beliefs, norms, formation memories, attachment. The builder only handles the cognitive sections. To migrate these, `EvalCharacterData` needs to expand beyond drives and formation memories to include all the character facets, and a test observation assembler needs to compose full prompts from that data. That's the next piece of work.

What this opens up: once the eval tests move, the `caps-llm-classifier` module becomes a self-contained test harness for cognitive rendering quality. Drive discrimination, attractor steering, priority signature effects — all testable without spinning up a Quarkus container or importing a single manor type. The observation builder becomes the contract surface between cognitive architecture and application.
