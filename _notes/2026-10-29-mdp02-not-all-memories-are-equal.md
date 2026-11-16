---
layout: post
title: "Not All Memories Are Equal — Importance Scoring for Agent Retrieval"
date: 2026-10-29
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
tags: [memory, retrieval, generative-agents, modulation, sc2, cognitive-architecture]
---

# Not All Memories Are Equal — Importance Scoring for Agent Retrieval

An SC2 agent that remembers "I built a supply depot" and "enemy rushed with 6-pool" equally well is an agent that will die to the rush. Both memories are recent. Both are semantically relevant to "what happened this game." But one of them is a routine action and the other is a critical threat. Without a way to distinguish the two, retrieval returns whatever the vector similarity and timestamp happen to favour. Sometimes that's the rush. Sometimes it's the supply depot. The agent's survival becomes a coin flip decided by embedding geometry.

Park et al.'s Generative Agents paper established the fix: tri-score retrieval. Every memory gets three scores — recency, relevance, and importance — and retrieval multiplies them together before ranking. Recency handles temporal decay. Relevance handles semantic similarity. Importance is the third axis — a permanent rating of how significant a memory is, independent of when it happened or what you're currently thinking about.

![Tri-score retrieval — recency × importance × relevance](images/tri-score-retrieval.svg)

Neocortex already had recency decay and relevance. The `ModulationFactor<T>` SPI and `RetrievalModulator` provide a multiplicative composition pipeline — each factor returns a `double`, and the composite score is the product of all factors. Four factors existed: `recencyDecay` (exponential half-life), `confidenceWeight`, `moodCongruence` (PAD-distance alignment), and `GoalRelevanceModulationFactor` (BFS graph proximity to active goals in the MindMap). The architecture was ready. The missing piece was a way to assign importance.

## The Timing Problem

The obvious implementation — "call an LLM to rate importance every time you store a memory" — runs into the timing constraint. An SC2 agent stores memories every turn. A synchronous LLM call per memory adds 100–200ms to every store operation. For a real-time game agent, that's unacceptable.

The `CaseEnrichmentStep` SPI provides synchronous pre-store enrichment — it intercepts `MemoryInput` before it reaches the store. That's the wrong boundary for an LLM call. The store path needs to stay fast: store the memory, return the ID, let the caller continue.

The answer is async post-store enrichment via `enrichAttributes()`. Store the memory immediately with no importance score. Buffer up a batch of recent memories. When the batch reaches a count threshold (default 8) or a time deadline (default 5 seconds), flush the batch to an `ImportanceScorer` in a single LLM call. Write the scores back via `enrichAttributes()`.

The cost: a brief window where memories exist without importance scores. Retrieval during that window falls back to the default (1.0 — no penalty). In practice, the window is a few seconds at most. The agent's next retrieval cycle finds the scores already in place.

## Architecture

![Importance scoring pipeline — store, score, retrieve](images/importance-scoring-pipeline.svg)

Four components, clean separation between the write path and the read path.

**`ImportanceScorer`** is the SPI — a `@FunctionalInterface` in `memory-api`. It takes a batch of `ScoringEntry` records (memoryId, text, domain, tenantId) and returns a `Map<String, Integer>` of memoryId → importance (1–10). Missing entries in the return map mean "could not score" — the observer skips enrichment for those memories. The default bean is a no-op that returns an empty map. Zero overhead when no scorer is configured.

**`LlmImportanceScorer`** is the LLM-backed implementation. A single prompt asks the model to rate each memory on a 1–10 scale. The response is a JSON array of `{index, score}` pairs. Parsing uses regex pattern matching rather than a JSON library — this handles partial failures gracefully. If the LLM produces valid scores for 6 out of 8 memories, those 6 get enriched and the other 2 are skipped. No all-or-nothing failure mode.

**`ImportanceScoringObserver`** is the CDI glue. It observes `ExperienceRecorded` events — the same CDI event that `RelationshipProcessor` and `SubThoughtExtractionObserver` already watch. The buffer is a `ConcurrentLinkedQueue`. The flush trigger is dual: count threshold checked inline on each event, time deadline via a `ScheduledExecutorService` daemon thread. The `@PreDestroy` shutdown method drains the buffer — no memories are silently lost on container shutdown.

**`ImportanceModulationFactor`** is the retrieval-time reader. It pulls the `"importance"` attribute from `Memory.attributes()`, parses the integer, and divides by 10.0 to produce a multiplier in the range 0.1–1.0. A score of 1 (trivial) produces a 0.1 multiplier — demoting the memory 10× relative to a score of 10 (critical). Missing attributes return 1.0, so memories stored before this feature was added are not penalised.

## Why String Attributes, Not a First-Class Field

The `Memory` record has first-class typed fields for confidence, PAD emotional dimensions, and timestamps. Importance could have been another first-class field — a `Double` on the record, with a `ModulationProfile` accessor and compile-time type safety.

I chose not to do that. Adding a field to `Memory` and `MemoryInput` means touching every store implementation — in-memory, SQLite, JPA, Mem0, Graphiti — plus Flyway migrations for the SQL-backed stores. That's a lot of mechanical work for a feature that hasn't proven its value yet. The `enrichAttributes()` API works specifically with `Map<String, String>`, and the string parsing overhead at retrieval time is one `Integer.parseInt` call per memory. Negligible.

If importance proves its worth — if it measurably improves retrieval quality in SC2 games — promoting it to a typed field is a straightforward migration. But I'd rather ship the feature and learn from it than build the perfect data model for a hypothesis.

## What This Means for the Agent

The difference is concrete. Two memories, same game, same agent:

- "Built another supply depot" → importance 2 → multiplier 0.2
- "Enemy rushed with 6-pool — critical threat" → importance 9 → multiplier 0.9

Before importance scoring, these memories competed on recency and vector similarity alone. If the embedding space happened to place "supply depot" closer to the current query, the agent would retrieve the routine action over the existential threat. After importance scoring, the 6-pool rush has a 4.5× retrieval advantage. It would need to be substantially less recent or less semantically relevant to lose to the supply depot.

That 4.5× gap is what separates an agent that remembers what matters from one that treats every experience as equally memorable. The same mechanism applies beyond games — any domain where some memories are more consequential than others, which is most domains.

## Salience Is Not Importance

The cognitive architecture already has a salience system: `SalienceStrategy` in `cognition-api`, implementing Klinger's Current Concerns model. Perceptual salience filters observations through the agent's active drives, goals, and concerns during cognitive ticks. The same observation has different salience depending on the agent's current motivational state.

Importance is different. It's a permanent, context-free rating assigned once at storage time. The 6-pool rush is important regardless of whether the agent is currently focused on macro, micro, or scouting. Salience gates what enters cognition in the moment. Importance modulates what gets retrieved later. Different timescales, different mechanisms, complementary effects.

An agent with both is an agent that notices the right things in real time (salience) and remembers the right things across time (importance). Neither alone is sufficient.