---
layout: post
title: "The Indirection That Disappeared"
date: 2026-11-04
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
tags: [entity-references, tokenisation, architecture, cognitive-architecture, design-patterns]
series: issue-799-inline-entity-tokenisation
---

# The Indirection That Disappeared

*Continues from [The Text That Knows Who It Means](2026-11-04-mdp01-the-text-that-knows-who-it-means.md) — which built inline entity tokenisation from scratch. This entry retires the thing it replaced.*

---

Every indirection in a system carries a lifecycle tax. A foreign key means a cascade rule. An offset-based index means a re-index operation. A parallel data structure means a synchronisation problem at every write, every merge, every delete.

The standoff annotation infrastructure in neocortex had five modules, thirty source files, and approximately three thousand lines of code whose only purpose was to maintain a parallel mapping between text and the entities mentioned in it. The mapping existed because the text itself didn't know who it was talking about. A memory like "Frida walked to the park with Diego" was stored as-is — literal names, no structure, no link to the knowledge graph. The annotation index provided that link: character offsets pointing from positions in the text to entity node IDs.

When I built inline entity tokenisation, that same memory becomes `{{p1:full:Frida}} walked to the park with {{p2:full:Diego}}`. The entity references live in the text. The parallel mapping no longer has a reason to exist.

This entry is about what happened when I removed it.

## The Consumers That Got Simpler

Three components in the cognitive pipeline queried the annotation index at runtime. Each had a different query pattern. Each got simpler when the query disappeared.

![Annotation index query vs inline token scan](images/annotation-vs-token-query.svg)

**EntityOverlapModulationFactor** boosts retrieval scores for memories that share entities with the current cognitive context. The old version took three constructor parameters — an `EntityAnnotationIndex`, a tenant ID, and a set of context entity IDs. On every memory evaluation, it called `annotationIndex.distinctEntitiesInSource("memory", memoryId, tenantId)` — a SQL query against a twelve-column table, returning a set of entity node IDs. It computed the overlap, multiplied by a boost factor.

With inline tokens, the constructor takes one parameter: the set of context entity IDs. The query becomes `TokenFormat.extractEntityIds(memory.text())` — a regex scan over text already in the JVM's memory, returning the same set. No store, no tenant, no infrastructure. The entity IDs are in the text.

**EntityAwareSalienceStrategy** boosted salience for observations that mention entities from the agent's active concerns. The old version called `annotationIndex.findBySource("memory", memoryId, tenantId)` for each concern, extracted mention texts, lowercased them, and checked if any appeared in the current observation. A SQL round-trip per concern, per tick.

With inline tokens, `TokenFormat.parse(concern.text())` extracts the surface forms directly. Same case-insensitive match, same result, no index.

**LlmReflectionSynthesizer** built an "Entity Context" section for LLM prompts — which entities appear across a set of experience memories, and how many times. The old version did two nested queries per memory: `distinctEntitiesInSource()` to get entity IDs, then `findByEntity()` to get mention texts. A cross-join over the annotation index, memory by memory.

Claude and I replaced this with a single loop: for each memory, `TokenFormat.parse(memory.text())` yields the surface forms and entity IDs directly. One regex scan per memory, no index joins, no tenant parameter. The prompt section looks identical to the LLM.

## The Dead Weight

Alongside the three live consumers, three other classes carried `EntityAnnotationIndex` as a constructor field — injected, stored, never read.

`BeliefRevisionPhase` accepted an `Instance<EntityAnnotationIndex>` in its constructor, resolved it eagerly, and assigned it to a field. No method in the class ever referenced the field. `MentalModelOrchestrator` had the same pattern — a nullable field with a package-private accessor that nothing called. `ExtractionRequestedObserver` had a live injection that wrote annotations after LLM extraction, but the consumer of those annotations (the reverse-query path) was already being replaced.

Three fields, three constructor parameters, three CDI resolution attempts at startup — all doing nothing. They accumulated over time as the annotation infrastructure was wired optimistically into components that might need it someday. That day never came.

## The Retirement

The five annotation modules collectively contained:

- `EntityAnnotationIndex` — the SPI with `addAnnotation`, `findBySource`, `findByEntity`, `distinctEntitiesInSource`, `removeBySource`, twelve-column `EntityAnnotation` record
- Two implementations — `InMemoryEntityAnnotationIndex` (tests) and `SqliteEntityAnnotationIndex` (production)
- A contract test suite with eighteen tests
- Five CDI lifecycle observers — `AnchorBiographyPreprocessor`, `MemoryAnnotationObserver`, `EntityResolutionService`, `MergeRepointObserver`, `MemoryEntityErasedObserver`
- `BulkReExtractionService` — a batch re-annotation tool for retroactive annotation of stored memories

![Surface area before and after](images/indirection-retirement.svg)

All of it replaced by two modules — `entity-token` (pure Java, zero dependencies) and `entity-token-cdi` (CDI lifecycle wiring). The entity reference index that replaced the annotation index has four columns instead of twelve. The character offsets are gone because nobody needs them — the token itself identifies the entity.

The final commit deleted forty-one files and three thousand one hundred thirty-five lines. The build passed. The tests passed. No consumer lost functionality.

## The Pattern: Embedded References Beat External Indexes

The annotation index was a reasonable design when it was built. Entity references needed to be tracked. A separate index is the standard approach — it's how NLP annotation tools work, how linguistic corpora are structured, how standoff markup has worked since the 1990s.

But standoff markup was designed for a world where the primary text is sacred — you can't modify the source document, so you overlay annotations externally. In a system where you control the text at write time, that constraint doesn't apply. You can put the reference in the text itself.

The practical consequences compound:

**Rename costs nothing.** A stored token `{{p1:full:Jane Doe}}` doesn't change when the entity is renamed. The render-time cache resolves to the current canonical name. No stored text is updated, no index is re-keyed.

**Merge is surgical.** The entity reference index identifies exactly which sources contain tokens for the absorbed entity. String prefix replacement — `{{old_id:` → `{{new_id:` — is deterministic and has zero false positives because entity IDs are unique.

**Erasure is clean.** Replace each `{{erased_id:variant:surface_form}}` with the embedded surface form. The text stays readable. The entity reference disappears. GDPR Article 17 compliance without destroying prose.

**Search is exact.** "Find all memories mentioning Mexico City" is not a semantic similarity search that might miss "CDMX" or "Ciudad de México." It's a prefix match on `{{place_001` — exhaustive, instant, embedding-free.

Every one of these operations required a synchronisation step with the annotation index. Every synchronisation step was a place where the index could drift out of sync with the text. Every drift was a bug that surfaced under edge conditions — concurrent writes, failed transactions, partial merges.

The pattern generalises. If you have a parallel data structure whose sole purpose is tracking relationships that could be encoded in the data itself — consider whether the indirection is earning its keep. Wikipedia discovered this decades ago: wikilinks are inline, not a separate database of page-to-page relationships. The link IS the text. The parallel structure was never needed.

## What This Opens Up

With entity references in the text, entity-aware retrieval becomes exact rather than approximate. The cognitive pipeline no longer depends on the annotation index being in sync — there is no sync, because there is no separate structure to sync. The `Personable` trait now exposes identity fields — pronouns, gender, name parts, preferred forms — through typed accessors that map directly to node properties, which means prompt rendering can address entities by the correct pronoun without a separate lookup.

The deeper implication is about profile portability. When you transplant a cognitive profile into a different context — say, the same character in a different story, with different entity names — the tokens remain stable. `{{p1:full:Jane Doe}}` becomes `{{p1:full:Maria Garcia}}` through a cache configuration change, not a data migration. The entity identity is an indirection that earns its keep.

Not all indirections are bad. But an indirection that exists only because the text couldn't describe itself was always a workaround, not a design.
