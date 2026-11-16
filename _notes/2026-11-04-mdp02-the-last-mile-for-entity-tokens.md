---
layout: post
title: "The Last Mile for Entity Tokens"
date: 2026-11-04
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
tags: [entity-references, tokenisation, cognitive-architecture, prompt-rendering, decorator-pattern]
---

# The Last Mile for Entity Tokens

I wrote [previously](2026-11-03-mdp03-names-are-not-identity.md) about why inline entity tokens — `{{p1:full:Frida}}` — replaced standoff annotations. The design was sound: stable identity tokens in stored text, dynamic name resolution at read time. But after building the tokeniser, the entity record, the Aho-Corasick surface-form scanner, and the CDI lifecycle wiring, two gaps remained. Nothing in the prompt pipeline resolved tokens before they reached the LLM. And nothing in the biography pipeline wrote the identity properties that the resolver needs.

These are the last two pieces of a pipeline that starts with biography YAML and ends with an LLM seeing natural language. Without both, the system has all the infrastructure for entity-aware cognition but none of the behaviour.

## The Prompt Rendering Gap

The cognitive pipeline produces 20+ prompt sections per tick. Mood, drives, narrative, mental models, goals, attention signals — each section pulls from memory stores, formats text, and hands it to the prompt assembler. Any section that includes memory text potentially carries entity tokens. Those tokens were passing straight through to the LLM as literal `{{p1:full:Frida}}` strings.

An LLM encountering `{{p1:full:Frida}} painted a self-portrait` has no idea what that syntax means. It might try to parse it. It might ignore it. It might hallucinate an interpretation. None of those outcomes is acceptable for a system that depends on the LLM understanding the semantic content of its own cognitive state.

The obvious fix — make each prompt section call `EntityRenderer.render()` on its output — would scatter rendering logic across 20+ classes. Miss one and tokens leak through silently. The failure mode is insidious: the LLM still generates a response, it just generates it from corrupted input.

A single decorator solves this. `EntityRenderingSection` wraps any `CognitionPromptRenderer` delegate. When `render()` is called, it delegates to the original section, checks the output for `{{` tokens, and resolves them:

```java
public @Nullable String render(CognitionRenderContext context) {
    String rendered = delegate.render(context);
    if (rendered == null || !rendered.contains("{{")) {
        return rendered;
    }
    if (identityLookup != null) {
        return EntityRenderer.render(rendered,
                entityId -> identityLookup.apply(entityId, context.tenantId()),
                RenderMode.PROSE);
    }
    return TokenFormat.stripTokens(rendered);
}
```

No prompt section needs to know entity tokens exist. The `contains("{{")` check short-circuits sections that don't carry entity text — mood scores, drive intensities, attention signals pass through with zero overhead.

CognitionCore already had the extension point: `chainSectionCustomizer()`, a composable operator that transforms the section list. Tier filtering uses it to suppress low-priority sections under high arousal. Entity rendering chains into the same mechanism — one call wraps every section, present and future.

## The Identity Property Gap

But resolving tokens requires identity records, and identity records require properties on MindMap nodes. `EntityIdentityCache` reads properties like `canonicalFirst`, `pronounsSubject`, `preferredShort` from nodes to build the `EntityIdentity` that the renderer uses. The `Personable` trait provides typed accessors for the same keys. The reader infrastructure was complete.

The writer was empty. `EntityEnrichmentHandler` — the biography importer that writes drive baselines, CAPS patterns, and behavioural attractors onto entity nodes — had no awareness of identity fields. Neither did any other biography handler. The `EntityEnrichmentEntry` YAML record had fields for psychological data but nothing for "what is this entity's first name?"

The cache would dutifully scan every node's properties, find no identity keys, and return an `EntityIdentity` with nothing but the node's `name()` as `canonicalFull`. One surface form per entity. No pronouns. No preferred forms. No first/last decomposition. The tokeniser's surface form trie was nearly empty, and the renderer fell back to surface forms for everything.

![Entity identity pipeline](images/identity-property-pipeline.svg)

The fix is an `IdentityFields` inner record on `EntityEnrichmentEntry`:

```yaml
entity_enrichment:
  - id: enrich-frida
    entity_ref: Frida Kahlo
    identity:
      canonical_first: Frida
      canonical_last: Kahlo
      pronouns_subject: she
      pronouns_object: her
      pronouns_possessive: her
      pronouns_reflexive: herself
      preferred_short: Frida
      preferred_long: Frida Kahlo de Rivera
      aliases: [Friducha, "La Heroína del Dolor"]
    drive_baselines:
      curiosity: { baseline: 0.85, justification: "..." }
```

`IdentityFields.toProperties()` converts non-null fields to the property map keys that `EntityIdentityCache` reads — `canonicalFirst`, `pronounsSubject`, `preferredShort`, etc. The handler writes them in the same `store.updateNode()` call that writes drive baselines. Three lines of handler code.

## Why Both Pieces Together

Neither piece works alone. The renderer without identity properties falls back to surface forms — technically functional but semantically impoverished. Identity properties without the renderer sit unused on nodes while raw tokens reach the LLM.

Together, the pipeline is:

**Biography YAML** declares identity → **EntityEnrichmentHandler** writes properties onto MindMap nodes → **EntityIdentityCache** reads properties into `EntityIdentity` records → **EntityRenderingSection** resolves tokens in prompt sections → **LLM** sees natural language with current canonical names.

Each link is independently testable. Each degrades gracefully when the next link is absent. The renderer falls back to `TokenFormat.stripTokens()` when no cache is available — the LLM still sees readable prose, just with the original surface forms instead of resolved canonical names.

## What This Actually Means for AI Systems

Most AI agent frameworks treat entity references as a solved problem because they never separate identity from surface form in the first place. The agent's prompt says "Alice told Bob about the project" and that string is the entire representation. It works until Alice changes her name, or two Alices appear, or the same agent profile is reused in a context where Alice is called something else.

Entity tokenisation with dynamic rendering means the stored representation carries stable identity while the rendered text is always current. Rename an entity and every prompt section across every cognitive tick resolves the new name — no migration, no backfill, no find-and-replace across thousands of memories. Merge two entities that turned out to be the same person, and the token swap is a single operation on stored text.

The less obvious benefit is for the tokeniser itself. With identity properties populated, `EntityIdentityCache` builds surface form tables with every variant — full name, first name, preferred short, aliases. The Aho-Corasick scanner recognises "Frida", "Kahlo", "Friducha", "La Heroína del Dolor" as the same entity. Without identity properties, it only knows "Frida Kahlo" — miss the first name alone in text and the tokeniser can't link it.

The infrastructure existed for months. What was missing was the wiring — the three lines in the handler that write identity, and the decorator that resolves it before the LLM sees it.
