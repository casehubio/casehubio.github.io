---
layout: post
title: "When Three Types Cost Eight Modules"
date: 2026-10-30
entry_type: article
subtype: diary
projects: [casehubio/neocortex]
tags: [architecture, dependency-management, annotation, cognitive-architecture]
series: issue-727-observability-api-extraction
---

# When Three Types Cost Eight Modules

A module called `annotation/` needed to observe graph mutations. It used two types — `GraphMutation` and `GraphMutationRecorded` — both records, both pure data, both carrying nothing more than strings, timestamps, and a `Confidence` value.

Those two types lived in `cognitive-observability-core`. To get them, `annotation/` had to depend on that module. And that module depended on `mindmap-intelligence`, which depended on `cognitive-index`, which depended on `mindmap-core`, which depended on `jackson-databind`, which depended on... you get the pattern.

Two records. Eight transitive modules. A full LLM extraction pipeline, a consolidation scheduler, a graph analysis library — all dragged onto the classpath of a module that just wanted to watch for node merges.

![Dependency graph before extraction — annotation/ pulls in the full cognitive-observability-core chain](images/dep-before-extraction.svg)

## The Principle This Violates

CDI events are supposed to decouple producers from consumers. The producer fires a `GraphMutationRecorded` event; any consumer with `@Observes` receives it. No compile-time coupling between the two. Beautiful in theory.

But the event type itself has to come from somewhere. If it lives in a heavy implementation module, the decoupling is illusory. The consumer is structurally free from the producer — but transitively bound to the producer's entire dependency chain. The classpath doesn't know the difference between a type you use and a type you inherit.

The principle: **event types that cross module boundaries belong in the lightest module that can carry their dependencies.** In our tiered architecture, that means a Tier 1 pure-Java module with no CDI, no implementation logic, no runtime behaviour.

## What We Extracted

I brought Claude in for the mechanical work. We created `cognitive-observability-api` — a new Tier 1 module with exactly three types:

- `GraphMutation` — the sealed interface with 13 record subtypes (node added, edge removed, nodes merged, etc.)
- `GraphMutationRecorded` — a CDI event wrapper carrying tenant ID and the mutation
- `FieldChange` — the record used by `NodeUpdated` to describe what changed

The static diff methods that previously lived on `FieldChange` — `diff(MindMapNode, NodeUpdate)` and `diffSnapshot(NodeSnapshot, NodeUpdate)` — moved to a `FieldChanges` utility class in `cognitive-observability-core`. Those methods need `NodeSnapshot`, which stays in core. The record itself has no dependencies beyond `java.lang.Object`.

The new module's pom has three dependencies: `cognitive-api` (for `Confidence`), `mindmap-api` (for `MergeConflict`), and `jackson-annotations` (for `@JsonTypeInfo`). All Tier 1. No implementation modules. No runtime.

![Dependency graph after extraction — annotation/ depends only on the lightweight API](images/dep-after-extraction.svg)

`annotation/` now depends on `cognitive-observability-api` instead of `cognitive-observability-core`. Eight transitive modules gone. Compile time measurably faster. And more importantly: the module's dependency graph now reflects its actual usage — it observes mutations, it doesn't process them.

## Why This Matters Beyond the Diff

The neocortex cognitive stack has grown to around 60 modules. At that scale, dependency discipline isn't about aesthetics — it's about whether new modules can be lightweight. Every module that exists today sets a precedent for modules that come tomorrow. If `annotation/` needs the full observability chain, then `summarisation/` will too. And `knowledge-pipeline/`. And whatever cognitive subsystem comes next.

The tax compounds. Each unnecessary transitive dependency adds build time, increases the risk of version conflicts, and makes it harder to reason about what a module actually does. When I look at a pom and see `mindmap-intelligence` in the dependency tree of a module that never touches extraction, I know the architecture has a leak.

The extraction was four files moved, one utility class created, three poms updated. The kind of change that takes 20 minutes and prevents a year of accumulated technical debt.

## Annotation Re-Extraction: Operational Tooling for a Moving Target

The second piece of work was less structural but equally practical. Our annotation pipeline uses LLM-based extraction: text goes in, entity mentions come out as standoff annotations linking text spans to MindMap nodes. The system already had `removeBySource()` for deleting annotations and the full extraction pipeline for creating them. What it didn't have was a way to re-run extraction across an entire corpus.

This matters because extraction quality isn't static. Better prompts, updated models, refined coreference resolution — every improvement to the extraction pipeline makes existing annotations stale. The text hasn't changed, but our ability to understand it has.

![Bulk re-extraction pipeline — scan, remove, re-extract, annotate](images/reextraction-pipeline.svg)

`BulkReExtractionService` does the obvious thing: scan all memories for a tenant, delete their existing annotations, re-run extraction, write fresh annotations. Cursor-based pagination for large memory sets. Per-memory error isolation so one extraction failure doesn't kill the batch. A `ReExtractionResult` that tells you exactly what happened — how many sources processed, how many annotations created, what failed and why.

It's operational tooling. Not exciting. But the alternative — manually re-extracting each source, or worse, living with annotations that reflect last month's extraction quality while this month's is measurably better — is the kind of invisible degradation that makes knowledge systems quietly less useful over time.

## Documenting the Architecture as It Grows

The neocortex `ARC42STORIES.MD` previously covered layers L1 through L15 — inference, RAG, corpus, memory, CBR. The entire cognitive/mindmap subsystem — around 30 modules spanning the knowledge graph, cognitive pipeline, CAPS personality engine, appraisal, attention, annotation, memory seeding, and schema generation — had no layer definitions at all.

That gap was fine when the cognitive stack was three modules. At 30, it meant the architecture document couldn't answer the question "which tier does `mindmap-intelligence` live in?" — a question that directly affects dependency decisions like the one that motivated this entire session.

We added layers L16 through L28, covering everything from the Tier 1 foundation types (`thing-api`, `cognitive-api`) through the full cognitive pipeline (`cognition-api`, `cognition`). A new Journey J6 captures the cognitive architecture as a coherent narrative arc. The layer × chapter matrix now has a C13 column showing where each of these 30 modules was introduced.

This isn't just documentation hygiene. When someone asks "can `caps-engine` depend on `cognitive-index`?" the answer is in the layer table: L24 depending on L21 — that's a downward dependency, architecturally valid. Without the layers, that question requires tracing the pom dependency chain manually. The architecture document exists so that architectural questions have architectural answers, not Maven archaeology answers.

## The Compound Return

Three types extracted. One utility service built. Thirteen layers documented. None of these changes individually is a large piece of work. But they share a thesis: a growing system needs continuous architectural maintenance, and the maintenance is cheapest when it's done before the debt compounds.

The module extraction prevented eight modules from becoming a permanent fixture in `annotation/`'s classpath. The re-extraction tooling prevented stale annotations from becoming an accepted cost of improving extraction quality. The layer documentation prevented the cognitive subsystem from becoming an architectural black box where dependency questions have no principled answer.

Each one is a bet that future work will be easier because this work was done now. In my experience, that bet pays off.
