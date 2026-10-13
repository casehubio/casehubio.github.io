---
layout: post
title: "The cache that wasn't a database"
date: 2026-10-04
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
tags: [knowledge-pipeline, sqlite, spatial, cache, architecture]
---

I started this branch expecting to wire up Tile38 — a Redis-protocol geospatial
database — as the cache engine for the knowledge pipeline. The design spec said
so. Twelve adversarial decision reviews said so. Then, five batches into
implementation, the spec turned out to be wrong.

The insight was simple once it arrived: the cache isn't a spatial database. It's
query-result memoization. "Find Italian restaurants within 1km" doesn't need
spatial indexing over entities — it needs to know whether a broader query ("all
restaurants within 5km") already answered it. That's subsumption, not geometry.

Most cache hits come from recognising that a broader result set already exists
and filtering it client-side. A search without a category filter subsumes one
with a filter. A search with a larger radius subsumes one with a smaller radius
at the same centre. These are set-containment checks, not spatial queries. Tile38
is excellent at "what's near this point?" but that question only matters for
entity resolution blocking — a secondary concern, not the primary retrieval path.

Once that reframing landed, the architecture collapsed to SQLite — the same
engine already backing four other stores in neocortex. R\*Tree handles the
spatial subset (entity resolution needs "find cached entities within 200m"
for blocking candidates). Query memoization with a `SubsumptionRule` SPI handles
the rest. Zero external infrastructure. No Docker, no Redis protocol, no Tile38
version constraints.

The `SubsumptionRule` SPI is the interesting piece. Each query domain defines
what "broader than" means. For spatial queries, it's geometric circle
containment — is the queried circle contained within the cached circle? For
category queries, it's filter narrowing — a result without a category filter
subsumes one with a filter. The SPI makes the cache engine domain-agnostic
while each domain brings its own subsumption semantics.

Claude caught a provider attribution bug during code review that I'd have missed.
The orchestrator's `resolveProviderId()` always returned the first provider's ID
regardless of which provider actually returned each Place result. With multiple
providers, every entity would get the same source attribution — meaning cache
entity IDs (SHA-256 of source + externalId) would collide across providers
sharing external IDs. We introduced a `ProviderPlace` record to track the
association through the fetch loop.

The knowledge pipeline now sits between connector SPIs (pure fetch) and MindMap
(durable knowledge). It owns query normalisation, cache memoization with
subsumption, cross-source entity resolution (spatial blocking + domain-specific
matching + three-tier decision with attention pipeline integration), multi-session
research orchestration, field-type TTL decay, and user-triggered promotion to
MindMap with three-stage deduplication.

What interests me looking ahead is subsumption beyond spatial queries. Product
searches have category hierarchies and alias normalisation. Travel route caches
have origin-destination containment. The SPI shape works for all of these — the
domain logic changes, the memoization engine stays the same. Phase 3 will test
whether that generalisation holds.
