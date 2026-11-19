---
layout: post
title: "One Name, Two Problems"
date: 2026-11-04
entry_type: article
subtype: diary
projects: [casehubio/neocortex]
tags: [entity-references, tokenisation, knowledge-representation, cognitive-architecture, identity]
series: issue-808-entity-identity-props
---

# One Name, Two Problems

I wrote [previously](2026-11-03-mdp03-names-are-not-identity.md) about separating entity identity from surface text — inline tokens like `{{bowie:full:David Bowie}}` that carry stable identity through the system while names change around them. That solved the rename problem. But it assumed every entity is a single identity.

David Bowie is not a single identity.

Ziggy Stardust had different relationships, different artistic output, different emotional associations than Bowie-as-himself. The Thin White Duke was cold, European, musically minimal. These aren't nicknames — they're semantically distinct personas that share a base person but carry their own graph state. An NPC operating under a cover identity is the same pattern. A historical figure known by different names in different life periods is the same pattern.

The entity token system handled one form of name multiplicity: aliases. "David Robert Jones" and "Davy Jones" are different strings that refer to the same David Bowie. The tokeniser scans text, finds "David Robert Jones", and emits `{{bowie:full:David Robert Jones}}`. Same entity ID, same node in the knowledge graph, same memories and edges. Aliases are a lookup problem, and the Aho-Corasick trie already solves it.

Personas are different. Ziggy performed at Hammersmith Odeon — that's an edge on the Ziggy persona, not on the Bowie base entity. The Duke era had its own emotional valence. A cognitive system that flattens all of these into one identity node loses the ability to reason about which persona was active when.

## The Test

Claude asked a question that clarified the design: are personas the same as nicknames?

No. The distinction is simple and practical. Does this name carry its own edges, memories, and temporal validity? If yes, it's a persona — it needs its own node in the knowledge graph. If no, it's a nickname — it belongs in the aliases set on the existing entity.

"Friducha" for Frida Kahlo is a nickname. She didn't have different relationships when people called her Friducha in letters. One node, one set of edges, one identity — the name is just a surface form.

"Ziggy Stardust" for David Bowie is a persona. Ziggy had the Spiders from Mars, a specific time period (1972-1973), a specific emotional profile (flamboyant, alien, doomed). Those facts need somewhere to live in the graph, and that somewhere can't be the same node as Bowie-the-person.

![Nickname vs persona distinction](images/nickname-vs-persona.svg)

Two mechanisms for two problems. Nicknames go in `aliases` on `EntityIdentity`. Personas get their own MindMap nodes connected to the base entity via `persona-of` edges.

## Dot-Notation Tokens

The token format already allows dots in entity IDs — the regex is `[^:}]+`. A persona token looks like:

```
{{bowie.ziggy:full:Ziggy Stardust}} performed at Hammersmith Odeon.
{{bowie.duke:preferredShort:The Duke}} era was colder, more European.
{{bowie:canonicalFirst:David}} composed Heroes.
```

The dot separates the base entity ID from the persona qualifier. `TokenFormat` doesn't need to change — it returns `"bowie.ziggy"` as the entity ID, which is semantically correct. The dot-splitting happens in `EntityRenderer`, where semantic interpretation belongs.

![Dot-notation resolution flow](images/persona-resolution-flow.svg)

The renderer splits on the first dot, looks up the base entity, finds the alter ego in its `alterEgos` map, and resolves the variant against the persona's identity. If anything in the chain is missing — unknown base entity, unknown qualifier, unknown variant — it falls back to the surface form. Safe degradation, same as non-dot tokens.

## The Graph Layer

Each persona gets its own MindMap node in the same subgraph as its base entity. A persona of a person is still a person — it lives in the PERSON subgraph, carries the same identity property keys (`canonicalFull`, `canonicalFirst`, `pronounsSubject`), and can have its own traits, edges, temporal validity, and memories.

The `persona-of` edge type is registered via `MindMapVocabulary` with aliases `alter-ego` and `stage-persona`. Edge direction is persona → base entity. Given a base entity, you find its personas by looking for incoming `persona-of` edges.

`EntityIdentityCache` traverses these edges eagerly when building the tenant cache. For each base entity, it finds persona nodes, reads their identity properties, and builds alter ego `EntityIdentity` instances with compound IDs. The cache result for "bowie" includes both the base identity and a `Map<String, EntityIdentity>` of alter egos keyed by qualifier. When the `SurfaceFormTrie` is built from these identities, "Ziggy Stardust" is registered alongside "David Bowie" — the tokeniser handles both transparently.

## Transparent Aggregation

The last piece is `CognitiveProfile`. When the cognitive system resolves entity knowledge for Bowie — memories, edges, affect trajectory — it should include Ziggy's memories and edges without the caller knowing about personas. Memories stored against the Ziggy node ID appear in Bowie's `EntityKnowledge` result. Edges from Ziggy to other entities appear in Bowie's edge list.

This is the right default. A caller asking "what do I know about Bowie" should get the complete picture. The persona mechanism is an implementation detail of identity representation, not something every consumer of `CognitiveProfile` needs to understand.

## Why This Matters Beyond This Project

Any system that represents entities in a knowledge graph will eventually hit this problem. Character databases for fiction. Historical knowledge bases where a figure operated under multiple identities. NPC systems where agents maintain cover identities. CRM systems where a person has different professional personas across organisations.

The two-mechanism approach — aliases for surface form variants, graph nodes for semantic identities — is general. The specific implementation (dot-notation tokens, persona-of edges, eager cache traversal, transparent CognitiveProfile aggregation) is one realisation. The design decision is the reusable part: recognise that name multiplicity has two distinct causes, and handle them with distinct mechanisms at the appropriate layer.

The alternative — treating everything as a name variant, or treating everything as a separate entity — fails in predictable ways. Flatten personas into aliases and you lose the ability to attach persona-specific facts. Elevate nicknames into separate nodes and you fragment an entity's knowledge across nodes that don't need separation. The test ("does this name carry its own graph state?") draws the line where it belongs.
