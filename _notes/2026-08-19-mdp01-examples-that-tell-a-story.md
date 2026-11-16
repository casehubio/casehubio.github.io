---
layout: post
title: "Examples that tell a story"
date: 2026-08-19
entry_type: note
subtype: diary
projects: [eidos]
tags: [annotations, examples, capability-matrix, agent-identity]
series: issue-141-annotation-examples
---

Annotation-driven identity needs examples that aren't test fixtures wearing a trenchcoat. The two examples from the annotations module — `DocumentAnalyst` and `LegalAnalystAgent` — covered the basics, but between them they exercised maybe half the annotation surface. `provider`, `modelFamily`, `version`, `vocabulary`, `dispositionProfile`, `styleProfile`, `delegation`, `PRIVATE` visibility — all untested in examples.

The fix was replacing those two with six, each grounded in a domain where the annotation capabilities they demonstrate are load-bearing rather than decorative.

`MedicalScribeAgent` is the compliance showcase — explicit `id` and `name` overrides (for audit-trail stability), `jurisdiction: "US"`, `dataHandlingPolicy: "hipaa-compliant"`, `version: "2.1.0"`. It carries a `PRIVATE` goal (`detect-safety-signals`) that the supervising clinician's system sees but the scribe's A2A card omits — because you don't want external agents routing work based on your internal safety monitoring.

`ChildCompanionBot` makes the same visibility mechanism tangible from a different angle. Its `PRIVATE` constraint (`session-limits` — enforce time caps set by parents) and `PRIVATE` goal (`escalate-distress` — detect signs of distress and alert the guardian) have genuine meaning. A child interacting with the bot shouldn't see "I will escalate distress signals to your parent" in the agent card. That's the whole point of `Visibility.PRIVATE`.

`CreativeDirectorAgent` carries all four vocabulary URIs — `vocabulary`, `slotVocabulary`, `dispositionVocabulary`, `styleVocabulary` — plus `dispositionProfile` (Jungian functions), `styleProfile` (sarcasm dimensions), and `delegation: true`. It's personality-first, no individual disposition axes, everything driven by the cognitive profile.

`TutorAgent` demonstrates that axes and `dispositionProfile` coexist — `socialOrient: "supportive"` alongside `dispositionProfile: {"INTROVERTED_SENSING", "EXTRAVERTED_FEELING"}`. Its `learning-outcomes` goal maps to two capabilities (`explanation` and `assessment`), showing multi-capability goal mapping.

`CodeReviewAgent` is the common case — what most developers will actually write. Identity, all five disposition axes, three capabilities, `provider` and `modelFamily` set.

`CustomerSupportTriage` is the bare minimum — `@Identity` only, no `@Disposition`, no capabilities. Auto-derived id from class name. The test verifies that `disposition()` returns `null` and capabilities/goals/constraints are all empty.

The `CAPABILITY-MATRIX.md` maps all 36 annotation capabilities to which example demonstrates each one and which deployment test verifies it. Modelled directly on engine's capability matrix from the engine-annotations work.
