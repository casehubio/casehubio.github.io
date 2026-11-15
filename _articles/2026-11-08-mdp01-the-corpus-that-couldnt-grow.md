---
layout: post
title: "The Corpus That Couldn't Grow"
date: 2026-11-08
entry_type: article
subtype: diary
projects: [casehubio/neocortex]
tags: [architecture, corpus, llm-agents, api-design, mcp, knowledge-management]
series: issue-763-cognitive-knowledge-management
---

# The Corpus That Couldn't Grow

Our corpus read path is genuinely good. Hybrid search with dense, sparse, and BM25 legs. Adaptive sizing that extends when results are thin and trims when they're noisy. Federation across corpora. Version history. Provenance tracking. A self-improving feedback loop. An agent looking for knowledge finds a polished, ergonomic surface.

The write path was a different story. To store a finding, an agent had to construct a `CorpusEntry` manually — pipe-delimited metadata strings, no lifecycle, no session continuity, no way to ask "what did I write?" The corpus was a library with excellent checkout and no returns desk.

This is a common asymmetry in knowledge systems and it's worth understanding why it happens.

## Why read paths win

Read paths get built first because they're the demo path. You show the search working, stakeholders see results, the feature ships. Write paths get deferred because "the data is already there" — imported from files, populated by a batch job, seeded from a source of truth somewhere else.

For a static knowledge base that works. For an agent-driven system it doesn't. An agent that can search but not contribute is a consumer. A corpus that only gets searched is a snapshot — frozen at the moment someone last loaded it. The knowledge it contains decays from the moment it stops growing.

The practical consequence: agents avoid writing to the corpus because the affordance isn't there. They'll search it naturally — MCP makes that easy. But storing a finding requires five lines of construction code that the agent has to figure out from the API shape. No agent does that voluntarily.

![Corpus read/write path symmetry](svg/corpus-write-path-symmetry.svg)

## Making write natural

The fix was four tools that bring the write side up to the read side's ergonomics.

**Batch findings** solved the chatty-roundtrip problem. A research session that discovers ten things shouldn't need ten separate calls. `recordFindings` takes a list, returns IDs for successes, and isolates failures — finding 3 having a blank topic doesn't block findings 1, 2, and 4 through 10.

**Lifecycle states** gave entries a temporal dimension. Before this, entries were binary — present or deleted. Now they can be drafts (provisional, excluded from default search), published (confident, searchable), or archived (superseded, still queryable on demand). The implementation detail that made this backward-compatible is worth noting: search uses `PayloadFilter.Not(In("status", ["draft", "archived"]))` rather than `Eq("status", "published")`. In Qdrant's filter semantics, `must_not` only excludes points where the field *matches* — points where the field doesn't exist pass through. Every entry that predates the status field continues to appear in search results without a migration.

**Research sessions** addressed the cold-start problem. Without session continuity, every conversation starts from zero. The agent doesn't know what it investigated last time, what it found, or what questions remain open. `startSession` creates a named research context. `recordFinding` accepts a `sessionId` to link findings to that context. `resumeSession` returns the session state including everything discovered so far. The next agent picks up warm instead of cold.

**Contributions** closed the write-side observability gap. `ReadTracker` already showed which entries an agent consumed. `contributions` shows what it produced — created, modified, archived, filtered by time range and optional session. This is the handover primitive: the next agent can see exactly what the previous one contributed.

## The sessionId-as-metadata pattern

One design choice is worth calling out because it shows up in other contexts. Research sessions link to findings through a metadata convention — `sessionId` stored as a field in the entry's metadata map. No schema change, no new table, no foreign key.

This works because the metadata map is already the extension point. Every entry has one. The search infrastructure already indexes it. The only cost is a string field per entry, and null when no session is involved.

The tradeoff is queryability. A dedicated join table would give SQL-native lookups. The metadata approach requires either an in-memory index (what we built — a `ConcurrentHashMap` that tracks session→finding links at runtime) or a vector-store filter query. The in-memory index is a first-iteration pragmatism: it doesn't survive restarts, but the sessionId *is* persisted in the entry metadata, so a future implementation can rebuild the index from the store.

I prefer this kind of layered commitment. Start with the cheapest thing that works. The metadata is durable. The in-memory index is fast. If the system proves the feature matters, promote the index to SQLite. If it doesn't, you've paid for a string field per entry instead of a migration.

## What a growing corpus changes

The deeper point isn't about API ergonomics. It's about what happens when a corpus starts growing from use.

A static corpus is a reference — valuable but inert. It answers questions from a fixed point in time. An agent-fed corpus is a living knowledge base. Every research session deposits findings. Every finding links to provenance. Every session builds on what previous sessions discovered. The corpus becomes the institutional memory of the system's investigations.

This changes how you think about quality. A static corpus is curated — someone reviews entries before they go in. An agent-fed corpus is filtered — entries arrive as drafts, get promoted when confirmed, and get archived when superseded. The lifecycle states aren't workflow decoration; they're the quality mechanism. The corpus self-curates through use.

Two issues remain in the queue: inline annotations (so agents can comment on existing entries without creating new ones) and entry relationships (supersedes, updates, contradicts). Those complete the write surface. Once they land, the corpus has the same structural richness for writes that it's always had for reads.
