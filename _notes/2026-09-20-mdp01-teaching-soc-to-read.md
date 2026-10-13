---
title: "Teaching SOC to read its own threat library"
entry_type: note
subtype: diary
series: issue-51-rag-investigation-enrichment
projects:
  - casehubio/soc
tags: [rag, attck, neocortex, investigation-pipeline, threat-intelligence]
author: mdp
date: 2026-09-20
published: false
---

# Teaching SOC to read its own threat library

SOC's investigation pipeline has always known *what* ATT&CK technique an alert maps to — `AttckLookupTable` does a fast prefix-and-IOC-type lookup and returns a technique ID and a confidence score. But it never knew *why* that technique matters, what the detection guidance says, which threat groups favour it, or what mitigations are recommended. The structural data existed in the MITRE STIX bundle, but nothing extracted it, nothing indexed it, and nothing queried it during investigations.

This branch changes that in two layers.

## Layer 1: Structural backbone

I started with the STIX 2.1 enterprise bundle — ~600 techniques, ~140 threat groups, ~400 mitigations, ~15,000 relationships. We built a pure-Java parser (`AttckStixParser`) that reads the JSON and filters deprecated/revoked objects during parsing, producing typed records for each STIX entity.

At application startup, `AttckIngestionService` takes the parsed bundle and populates two representations inside neocortex:

**A MindMap subgraph** — techniques, groups, mitigations, malware, and tools as nodes with typed edges (`uses`, `mitigates`, `subtechnique-of`). Each node's MITRE ID is registered as an alias, so `resolveNode("T1003")` returns the OS Credential Dumping node directly. This gives you graph traversal: "what groups use this technique?", "what mitigates it?", "what are its sub-techniques?"

**A RAG corpus** — technique descriptions, detection guidance, group TTPs, and mitigation advice ingested as prose chunks with metadata. A natural language query like "credential dumping Active Directory" returns semantically relevant technique descriptions with their MITRE IDs attached.

The two representations complement each other. The graph answers structural questions. The prose answers "explain this to me" questions. A retrieved chunk carries a `mitreId` in its metadata that resolves to a MindMap node — so you can start from prose and traverse the graph, or start from the graph and read the prose.

The version-stamping design was the interesting part. The ingestion service creates a root node with an `attck-version` property as the *last* data-writing step. Any subgraph without a valid version stamp — null `rootNodeId`, missing root node, missing version property — is treated as incomplete and erased before re-ingestion. This gives you crash recovery without transactions: if the app crashes mid-ingestion, the next startup detects the incomplete state and re-ingests cleanly.

`RuleAttckMappingWorker` was already producing technique IDs from the static lookup table. We wired `AttckEnrichmentService` into its output path — each technique now comes back with related threat groups, applicable mitigations, and sub-techniques from the MindMap graph. The static table handles the fast deterministic path; the graph provides the context that makes it useful.

## Layer 2: Investigation-time retrieval

With the corpus populated, the next piece was wiring retrieval into the investigation pipeline. A new `rag-retrieval` capability fires after `attck-mapping` — at that point, the case context has the alert data, IOC enrichment, and ATT&CK technique names, which is the richest signal for constructing a retrieval query.

`SocRagRetrieveService` concatenates alert rule names, ATT&CK technique names, and IOC types into a natural language query, then hands it to neocortex's `CaseContextRetriever`. The results land at `.ragEnrichment` in the case context — alongside `.retrievedIncidents` from CBR and `.attckMapping` from the structural index. The analyst review now surfaces all three: past similar incidents, structural threat relationships, and relevant prose.

The service design isolates query construction from retrieval mechanics. `buildQueryText()` is the SOC-specific part — it knows which fields to extract from a SOC alert. Everything else (calling `CaseRetriever`, mapping results, handling errors, multi-corpus iteration) lives in `CaseContextRetriever` in neocortex. When AML or clinical need investigation-time RAG retrieval, they implement their own query builder and inject the same retrieval infrastructure.

That extraction was a mid-design decision. I'd originally planned to ship the generic retrieval inline and extract later. But the boundary was so clean — ~40 lines of genuinely generic code with a single domain-specific method — that deferring it would just mean someone copying the same boilerplate when the next app needed it.

## What this opens up

The internal knowledge layer (#58) is next — post-mortems, analyst notes, and runbooks ingested as garden entries. The retrieval infrastructure is already there; the question is how to structure incident narratives as prose that retrieves well. Past incidents retrieved via CBR give you structural similarity (same alert type, same IOC pattern). Prose retrieval gives you semantic similarity (similar investigation narrative, similar resolution approach). The combination should surface "we saw something like this six months ago" from both angles.

There's also a broader question about neocortex's MindMap search capabilities. Not all MindMap nodes are prose-rich enough for RAG embedding — group aliases, mitigation names, tool labels are short strings, not documents. Filed as neocortex#370: keyword/BM25 search over node content as a complement to embedding-based retrieval. If that lands, the dual-ingestion pattern (MindMap nodes + separate RAG ingest) becomes unnecessary for future subgraphs.
