---
layout: post
title: "The Desk That Disciplines"
date: 2026-11-05
entry_type: article
subtype: diary
projects: [casehubio/neocortex]
series: issue-763-cognitive-knowledge-management
tags: [research-workspace, knowledge-management, tool-design, rag, llm-discipline, affordances]
---

# The Desk That Disciplines

*Continues the [knowledge management series](2026-11-02-mdp01-knowledge-that-outlives-the-session.md).*

LLMs are powerful researchers and terrible ones at the same time. They can synthesise across domains, spot patterns in noise, generate hypotheses from sparse evidence. They also satisfice — find the first plausible answer and stop. They confabulate sources when none exist. They lose track of what they have covered and what they have not. They cannot tell when their information is stale.

These are not model limitations that better architectures will fix. They are structural consequences of how LLMs interact with information: everything flows through working memory, nothing persists, and there is no external signal for completeness or freshness. A model with a ten-million-token context window will still satisfice if nothing tells it there are gaps.

The corpus management work I wrote about [last week](2026-11-02-mdp01-knowledge-that-outlives-the-session.md) gives agents a library — curated, long-lived knowledge collections with lifecycle, federation, and provenance. But a library is not the same thing as research. A library stores what you know. Research is the process of building what you know from scratch, iteratively, with uncertainty at every step.

I needed a different abstraction. Not the library — the desk.

## The Desk and the Library

The distinction crystallised during the corpus domain audit. I was stress-testing our corpus abstractions against eight use cases — clinical guidelines, legal precedents, game NPC knowledge, research literature, and four others. The first six worked. The last two — market research and product research — broke everything.

The problem was not technical. The problem was that these two use cases describe a fundamentally different relationship between the agent and its content. A clinical guidelines corpus is batch-loaded by a human, organised once, and searched many times. A market research workspace is built by the agent itself, iteratively, during a research session. The agent decides what to ingest next based on gaps in what it has found so far. The content is provisional. It goes stale in weeks. Later findings derive from and reference earlier ones.

Trying to serve both patterns with the same API is like designing a single interface for a filing cabinet and a whiteboard. They hold different things, for different reasons, with different lifecycles.

![Desk and library architecture](images/workspace-desk-library.svg)

The workspace composes the same shared subsystems — the same byte-level storage, the same vector search, the same result ranking. But the contract it presents to the agent is different. The library gives you `catalogue`, `search`, and `federate`. The desk gives you `store`, `search`, `coverage`, `stale`, and `promote`. Those five operations encode a methodology.

## Discipline Through Affordances

I could have written a research guidelines document. "When conducting research, always track your sources. Always check coverage. Always note reliability." LLMs would follow these instructions approximately, when they remembered, for a while.

Instead I put the discipline in the tools. Not rules the agent must follow — affordances that make disciplined research the path of least resistance.

![Workspace tool affordances](images/workspace-affordances.svg)

**Provenance is enforced at the type level.** The `WorkspaceInput` record has a compact constructor that throws `IllegalArgumentException` if both `sourceUri` and `derivedFrom` are null. An original finding must have a source URL. A derived finding must reference the entries it builds on. There is no third option. The agent cannot store a finding without provenance — not because a validation layer rejects it, but because the type will not construct.

```java
public WorkspaceInput {
    Objects.requireNonNull(content);
    if ((derivedFrom == null || derivedFrom.isEmpty()) && sourceUri == null) {
        throw new IllegalArgumentException(
                "sourceUri is required for non-derived entries (provenance enforcement)");
    }
}
```

This is a design opinion. I could have made `sourceUri` optional and trusted the agent to fill it in. But optional provenance is no provenance — it degrades to empty fields in exactly the cases where traceability matters most.

**Coverage is explicit and gap-aware.** A workspace defines its target categories at creation time — "pricing", "features", "market-share", "customer-sentiment". Every finding the agent stores is tagged with the categories it covers. When the agent calls `coverage()`, it gets back a report showing how many findings exist per category, what their reliability distribution looks like, and which categories have zero entries. The agent cannot claim research is complete without confronting its gaps.

**Freshness is per-category and automatic.** Different content ages at different rates. A competitor's pricing page is stale in a week. A patent filing is stable for a year. The workspace carries a `FreshnessPolicy` that maps categories to maximum ages. When the agent calls `stale()`, it gets back entries with the specific categories that triggered staleness — "this entry is stale in pricing but still fresh in product-features." The agent knows what to refresh and why.

**Enrichment traces analysis to evidence.** When the agent builds analysis on top of raw findings — "Competitor X is 20% cheaper based on these three pricing pages" — it stores the analysis as a new entry with `derivedFrom` references to the source entries. The result is a directed acyclic graph. Any claim can be traced back through the derivation chain to the original sources. No orphaned analysis. No unsupported conclusions floating free of their evidence.

**Reliability is declared, not inferred.** Every entry carries a `ReliabilityTier` — AUTHORITATIVE, OFFICIAL, CURATED, COMMUNITY, or UNKNOWN. The agent assigns this on store. Coverage reports include the reliability distribution per category. A category "covered" by three COMMUNITY entries and zero AUTHORITATIVE ones looks different from one covered by a single AUTHORITATIVE source. The signal is there for any consumer — the agent, a human reviewer, a downstream system — to act on.

## The Consistency Problem

The interesting engineering challenge was the dual-write. Workspace entries live in SQLite (the source of truth for metadata, coverage queries, and freshness checks) and Qdrant (the derived index for semantic search). Two stores, two failure modes, two consistency guarantees.

SQLite writes first. If SQLite fails, nothing happens — no Qdrant write, no partial state. If SQLite succeeds but Qdrant fails, the entry exists in the workspace (coverage tracking works, freshness tracking works) but is invisible to semantic search. The agent gets a warning. A `reconcile()` call re-indexes the missing entry later.

The more dangerous case was the reverse direction. Multiple workspaces share a single tenant-scoped Qdrant collection. When I first designed reconciliation, the natural approach was: list what is in Qdrant, list what is in SQLite, delete orphans, re-index missing entries. The deletion step would have been catastrophic. `EmbeddingIngestor.listDocuments()` returns all documents for the tenant — there is no workspace filter. "Orphans" from Workspace A's perspective are live documents from Workspace B. Deleting them would silently corrupt every other workspace sharing that collection.

Reconciliation is additive-only. It re-indexes entries missing from Qdrant. It never deletes. Vector cleanup happens only during `archiveWorkspace()`, which walks the workspace's own entry IDs from SQLite and deletes them one by one from Qdrant — specific IDs, not orphan detection.

## What the Desk Opens Up

The workspace is a platform capability, not a product feature. Any agent in the casehub ecosystem — competitive intelligence, technical research, product analysis, literature review — gets the same disciplined research tools. The methodology is in the infrastructure, not in the prompt.

The `WorkspacePromotion` SPI is a `@DefaultBean` stub right now. When it has an implementation, mature research graduates from the desk to the library — workspace entries become corpus entries, searchable alongside curated knowledge. Research that starts ephemeral can earn permanence.

The `Content` sealed hierarchy — `TextContent`, `BinaryContent`, `StructuredContent` — was designed for the workspace but belongs in a shared location. The corpus management API has the same problem (its `CorpusEntry.body` is a `String`, which blocks non-text content). When that gets fixed, both systems will use the same content model. I put it in the workspace module for now and marked it for extraction — premature coupling between modules that don't otherwise depend on each other would have been worse than temporary duplication.

The data model carries `agentId` on every entry. Today, one agent per workspace. But the schema supports the multi-agent future — cross-agent search, per-agent coverage views, coordinated research across a team of agents working the same topic. The orchestration is a future problem. The data model is ready now.

I keep coming back to the same design principle: do not constrain agents, shape them. Give them tools whose affordances naturally produce the behaviour you want. A workspace where provenance is enforced by the type system, coverage gaps are visible by default, staleness is surfaced automatically, and every analysis traces to its evidence. The agent does not need to be disciplined. The desk is.
