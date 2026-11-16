---
layout: post
title: "The Conscientiousness Gap"
date: 2026-10-28
entry_type: note
subtype: diary
projects: [casehubio/casehub-mosaic, casehubio/eidos]
tags: [personality, vocabulary, eidos, manifest, architecture]
series: issue-291-personality-vocab-manifest
---

# The Conscientiousness Gap

The blocks-ui `avatar-step` component has personality data scattered across four TypeScript files — signal weights, tooltips, adjective phrases, tendency rules. The same data lives authoritatively in eidos Java enums. We'd been planning to replace the TS copies with imports from a generated manifest, but a completeness audit found five gaps in what eidos actually exported. The interesting one was the fifth.

## Auditing the manifest

The manifest (`personality-vocab.json`, generated at build time from Java vocab enums) had framework values, archetype compatibility, and categorical axis mappings. It was missing numeric signal weights — the [-1, +1] values that `deriveDispositions()` averages across frameworks to produce blended personality scores. Also missing: behavioral tendency rules (22 threshold-based conditions), detailed tooltip paragraphs, short adjective phrases for summary text, and disposition axis descriptions.

Four of these were straightforward additions. The `VocabularyTerm` interface already had extension points — `defaultProfile()` for Jungian function weights, `responseStyleGuidance()` for LLM prompt text. Adding `axisSignalWeight(DispositionAxis)`, `detailedDescription()`, and `adjective()` followed the same pattern. Each framework enum overrides the methods with per-value data. The generator conditionally emits the new fields. Mechanical work, but a lot of data to transcribe — 52 framework terms across 6 enums, each with 5 axis weights and a paragraph of tooltip text.

## The gap that wasn't data

`BigFiveTerm` had 8 constants. The Big Five model has 5 dimensions. That's 10 poles, not 8.

The missing two were conscientiousness — high and low. Eidos had a separate `ConscientiousnessTerm` vocabulary with 12 categorical axis values ("strict", "principled", "flexible"), and the original `BigFiveTerm` explicitly said in its metadata: "Conscientiousness axis terms already exist in the Conscientiousness vocabulary; this vocabulary provides the remaining four factors."

The reasoning looked plausible until you asked what each vocabulary actually does.

`BigFiveTerm` is a personality framework input. A user picks "I score high on Conscientiousness" in a profile editor. The system blends that selection's signal weight (+0.7 on the ruleFollowing axis) with their MBTI, Enneagram, and other framework selections to produce a disposition score.

`ConscientiousnessTerm` is an axis value vocabulary. The system resolves framework selections into categorical axis positions — "this combination maps to 'strict' on the rule-following axis." It's an output of the resolution process, not an input.

![Two abstractions, one name](images/vocab-abstraction-layers.svg)

Same domain name. Different level of abstraction. The first is what a user selects; the second is what the system derives. Neither replaces the other.

## The options that weren't options

I looked at three ways to handle the gap on the consumer side:

1. **Hardcode the two C weights in TypeScript.** The manifest would be the source of truth for everything except two numbers. Defeats the entire purpose of the migration.

2. **Derive C weights from the categorical axis values.** `ConscientiousnessTerm` has terms like "strict" and "flexible", but these are categories — not numeric signals. Mapping categories to [-1, +1] requires a translation table, which is just hardcoding wearing a trench coat.

3. **Add `CONSCIENTIOUSNESS_HIGH` and `CONSCIENTIOUSNESS_LOW` to `BigFiveTerm`.** Two new enum constants with signal weights, descriptions, and axis mappings. The manifest exports them like every other pole. The consumer reads all 10 Big Five poles uniformly. No special cases.

Option 3 is the only one that doesn't leave a seam. If the manifest is the single source of truth, it needs to contain all the truth.

The fix was 38 lines of Java — two enum constants, their `axisExactMatch()` overrides, signal weights, and detailed descriptions. The `@VocabularyMetadata` description updated from "provides the remaining four factors" to "High/Low poles for all five dimensions."

## What this opened up

The consumer spec is written and ready for the next session. With the manifest now complete — 10 BigFive poles, signal weights on all 52 framework terms, 22 behavioral tendency rules, tooltip paragraphs, adjective phrases — the blocks-ui side can replace its four data files with a single codegen import. No hardcoded values survive.

The more general observation: when two vocabularies share a domain name, check whether they serve different abstraction layers. "Conscientiousness already exists" was true at the word level and wrong at the architecture level. The axis-value vocabulary and the personality-framework vocabulary both deal with conscientiousness. They're not the same thing.
