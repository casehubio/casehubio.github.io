---
layout: post
title: "The Text That Knows Who It Means"
date: 2026-11-04
entry_type: article
subtype: diary
projects: [casehubio/neocortex]
tags: [entity-references, tokenisation, aho-corasick, performance, cognitive-architecture]
series: issue-799-inline-entity-tokenisation
---

# The Text That Knows Who It Means

*Continues from [Names Are Not Identity](2026-11-03-mdp03-names-are-not-identity.md) — where I argued that inline tokens beat standoff annotations. This entry is about building the thing.*

---

The previous entry made the case for replacing standoff annotations with inline entity tokens. The argument was conceptual: tokens are simpler, self-describing, and eliminate the coordination problem of maintaining a parallel offset index. All true. But arguments are cheap. The interesting questions only surface when you build it.

The token format is settled — `{{entity_id:variant:surface_form}}`. The entity record decomposes names into parts. The variant taxonomy maps surface forms to specific name components. What the previous entry didn't cover is the performance architecture, the concurrency model, or the integration strategy. Those are where the real design decisions live.

## Scanning Text for Entity Names at Write Speed

The tokeniser runs on every text write — every memory recorded, every biography entry imported, every conversation segment created. It needs to scan arbitrary prose against hundreds of entity surface forms and replace matches with tokens. This is the hot path.

The naive approach, which the old `MemoryAnnotationObserver` used, builds a list of all entity names sorted by length (longest first), then does `String.indexOf()` for each candidate against the input text. O(n × m) where n is the text length and m is the candidate count. For a hundred entities with three surface forms each, that's three hundred sequential substring searches per text write.

I replaced this with an Aho-Corasick automaton — the multi-pattern string matching algorithm used in tools like `grep -f`. You build a trie from all the patterns (surface forms), compute failure links via BFS, then scan the text in a single pass. Every character is examined exactly once. O(n) regardless of how many patterns you're matching against.

![Write path — tokenisation at store time](images/tokenisation-write-path.svg)

The trie is built from entity identity data stored as properties on MindMap nodes. Each entity contributes its canonical name, name parts (first, last, title), preferred forms, and aliases. The `surfaceForms()` method on `EntityIdentity` collects all variants, deduplicates them, and feeds them to the trie builder.

Three implementation details that matter:

**Case-insensitive matching with original-casing preservation.** The trie is built from lowercased surface forms and the input text is lowercased during scanning. But the token's `surface_form` field preserves the original casing from the source text. So "JANE walked home" tokenises to `{{p1:canonicalFirst:JANE}} walked home` — the trie matched on "jane", but the stored surface form is "JANE" as written. This is important because the surface form is the fallback when the renderer can't find the entity.

**Word-boundary post-filtering.** Aho-Corasick finds substring matches. "Jane" inside "Janeway" is a valid substring match. "Jo" inside "job" is another. Every match gets a word-boundary check: the character before the match start must be whitespace, punctuation, or string boundary, and the character after the match end must be the same. Two character comparisons per match, O(1) cost.

**Copy-on-write trie replacement.** When entity properties change, the trie must be rebuilt. Rather than locking, a new trie is built and atomically swapped into a `ConcurrentHashMap`. In-flight tokenisations use the old trie and complete consistently. A 200ms coalescing debounce prevents N rebuilds during bulk imports — multiple entity changes within the window coalesce into a single rebuild.

## Rendering at the Speed of Reading

The read path is simpler than the write path, and it needs to be — rendering happens on every text display, every prompt composition, every memory retrieval. The cognitive pipeline renders dozens of texts per agent turn.

![Read path — rendering at display time](images/tokenisation-read-path.svg)

The renderer parses tokens from stored text, looks up each `entity_id` in an in-memory identity cache, resolves the `variant` to the current name part, and substitutes. Most renders are no-ops — the surface form in the token matches the current canonical name. The entity was not renamed since tokenisation. The cache check is O(1), and when it confirms no change, the renderer skips the substitution entirely.

The identity cache is a `ConcurrentHashMap` per tenant, holding about 200 bytes per entity. For a tenant with a thousand entities — a generous estimate for most contexts — that's 200KB. Populated lazily on first access from MindMap node properties, invalidated per-entity via `GraphMutationRecorded` CDI events.

The fallback contract is important: when the cache has no entry for a token's entity_id, the renderer outputs the embedded surface form. This handles post-merge race conditions (the absorbed node was deleted before the async rewrite completed), cache cold starts, and orphaned tokens. The surface form is always safe because it was the original text before tokenisation.

## Three Operations, Three Strategies

The entity lifecycle is where inline tokens earn their keep. Standoff annotations require a parallel index update for every lifecycle event. Inline tokens encode the reference in the text itself, so each lifecycle operation has a natural strategy.

![Entity lifecycle — three operations, three strategies](images/tokenisation-lifecycle.svg)

**Rename** costs nothing. The stored token `{{p1:full:Jane Doe}}` doesn't change. The identity cache updates to reflect the new canonical name. Next time the renderer runs, it resolves to the new name. No text writes, no index updates, no migration. The surface form "Jane Doe" becomes a historical artifact — what the entity was called when the text was written.

**Merge** is a targeted rewrite. The `entity_references` index tells you exactly which sources contain tokens for the absorbed entity. For a tenant with ten thousand memories where fifty mention the merged entity, only those fifty are loaded and rewritten. A string prefix replacement — `{{old_id:` → `{{new_id:` — deterministic, no regex, no false positives because entity IDs are unique.

**Erasure** (GDPR Art.17) strips tokens for the erased entity, replacing each `{{erased_id:variant:surface_form}}` with its embedded surface form. The text stays readable — "Jane Doe walked to the park" — but the structured entity reference is gone. The entity is forgotten without destroying the prose.

## What the Read-Side Index Replaced

Retiring standoff annotations left a gap. The annotation index provided reverse queries: "which memories mention entity X?", "which entities appear in this source?", "how often is each entity mentioned?" The cognitive pipeline depends on these for retrieval scoring and salience computation.

A simple four-column table — `entity_id`, `source_type`, `source_id`, `tenant_id` — replaces the twelve-column annotation table. Populated as a side effect of tokenisation at write time, maintained by lifecycle handlers at merge and erasure. The queries are the same, the join is simpler, and the index has no character offsets to keep in sync.

The two consumers that hit this index on every cognitive tick — `EntityOverlapModulationFactor` (boosts retrieval scores for memories sharing entities with the current context) and `EntityAwareSalienceStrategy` (boosts salience for entity-relevant observations) — become calls to `findEntitiesBySource()` instead of `distinctEntitiesInSource()`. Same semantics, simpler implementation, no annotation infrastructure.

## Where Identity Lives

I considered three places for entity identity data: a separate `EntityRecord` SPI with its own store, trait interfaces on `Thing`, or properties on `MindMapNode`. Properties won because identity is just another aspect of the node — no new store to maintain, no new CDI lifecycle to wire. The node already has `name()`, `properties()`, `traits()`, and `NodeRef` for external references.

Property keys use camelCase — `canonicalFirst`, `pronounsSubject`, `preferredShort` — because `ThingProxyHandler.invoke()` uses `method.getName()` directly as the property key lookup. A trait interface method `canonicalFirst()` maps to `node.property("canonicalFirst")` with no transformation. The existing `Personable` trait gets extended with identity methods; no new interface needed.

This means the rendering cache, the trie, and the entity identity model are all derived views of data that already lives in the knowledge graph. No new persistence layer, no sync problem, no migration. When a node's identity properties change, a `GraphMutationRecorded` event invalidates the cache and schedules a trie rebuild.

## The Integration Points

The tokeniser hooks into the write path at two levels:

**Memory recording** uses `CaseEnrichmentStep` — the existing SPI for transforming `MemoryInput` before storage. The `TokenisingEnrichmentStep` runs at priority 100 (after other enrichments), tokenises the text, and returns a new `MemoryInput` with the tokenised text. The rest of the pipeline — storage, CDI events, retrieval — sees tokenised text transparently.

**ConversationBridge and biography import** call the tokeniser explicitly before creating nodes. These paths create MindMap nodes, not memories, so they don't go through `CaseMemoryStore`. The tokeniser is a static utility — `EntityTokeniser.tokenise(text, trie)` — no CDI dependency in the Tier 1 module.

Entity references are recorded post-storage. For memories, an `EntityReferenceRecordingObserver` listens to `ExperienceRecorded` events, scans the event's description text against the trie, and records the entity_ids it finds. For nodes, the caller records references immediately after `addNode()` returns the node ID.

## What Comes Next

The foundation is in place — two modules, seventy-two tests, all green. What remains is the consumer migration (rewiring cognitive pipeline consumers from the annotation index to the entity reference index and inline token scanning), extending the `Personable` trait with identity properties, integrating the tokeniser into biography import, and retiring the five annotation modules.

The migration will be interesting. Thirteen consumers depend on the annotation infrastructure in various ways — some for write-side annotation creation, some for read-side entity queries, some for lifecycle sync. Two of them turned out to have dead annotation index injections — constructor fields with zero callers anywhere in the codebase. The spec review caught both.

The bigger question is what entity tokenisation enables downstream. When every text reference to an entity carries a stable identity token, entity-aware retrieval becomes exact rather than approximate. Semantic search for "memories about Mexico City" no longer depends on embedding similarity between "Mexico City" and "CDMX" — it's a prefix match on `{{place_001`. Profile reuse — transplanting a cognitive profile into a different context where entities have different names — becomes a cache configuration, not a data migration.

The text knows who it means. That's a different kind of system than one that hopes the embeddings are close enough.
