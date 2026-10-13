---
layout: post
title: "Moving Schema Shorthands Out of the Post-Processor"
date: 2026-09-07
entry_type: note
subtype: diary
projects: [casehubio/engine]
tags: [schema, generator, victools, refactoring]
series: issue-1067-shorthand-module-migration
---

# Moving Schema Shorthands Out of the Post-Processor

The engine's `SchemaPostProcessor` was 1728 lines of raw JSON-building code — a mix of structural fixes, property renames, and `oneOf` shorthand patterns. The shorthands (scalar-or-object schemas like `AdaptationConfig`) didn't belong there. They're a schema generation concern, not a post-processing one.

Platform delivered `ShorthandModule` (platform#280), which intercepts during victools schema generation rather than patching the output afterward. Three patterns migrated: `AdaptationConfig` moved to a `ShorthandDefinition` registration, `ExpressionOrOverride` turned out to be redundant (the `ExpressionEvaluatorModule` was already generating the correct schema — the post-processor was silently overwriting it with an identical copy), and four trigger sub-type schemas moved into `TriggerModule` via `createDefinitionReference()`.

The `createDefinitionReference` call is the interesting part. The post-processor was building trigger sub-type schemas (`CloudEventTrigger`, `ScheduleTrigger`, etc.) as standalone defs, while the `TriggerModule` already generated the parent `Trigger` discriminated union referencing them by name. The defs existed only because the post-processor created them — victools never traversed the sub-types because `TriggerModule`'s `CustomDefinitionProvider` handled the parent type before victools reached the children. `createDefinitionReference()` forces victools to traverse the type and generate its def, closing the loop: the module that defines the structure also ensures the referenced types exist.

The committed `CaseDefinition.yaml` shifts slightly — `adaptation:` becomes a `$ref` to a named `AdaptationConfig` def instead of an inline `oneOf`, and the four trigger sub-types appear as top-level defs. `SchemaDriftTest` catches any regression — it compares the generator output against the committed schema on every build.

One thing this surfaced: the engine's full build doesn't compile cleanly on this branch, but the failures are pre-existing `ScoredCbrCase` type inference errors in the runtime test module — unrelated to the generator work. In a pre-release monorepo with 50+ modules, verifying "my changes don't break anything" means scoping to the modules I actually touched, not waiting for someone else's upstream fix to land.
