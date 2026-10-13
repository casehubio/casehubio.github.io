---
layout: post
title: "The incident that remembers"
date: 2026-09-23
entry_type: note
subtype: diary
projects: [casehubio/soc]
tags: [rag, cbr, knowledge-ingestion, organisational-memory]
series: issue-58-knowledge-ingestion-pipeline
---

Until today, casehub-soc had two kinds of memory. CBR stores structured incident
records — feature vectors that let the retrieval engine find past cases by
similarity. The ATT&CK corpus stores public threat intelligence as prose chunks —
technique descriptions, detection guidance, group TTPs. Both are useful. Neither
captures what the SOC team actually learned.

When an analyst resolves an incident, the interesting knowledge isn't the feature
vector. It's the narrative: what was tried, what the analyst concluded, which
playbook worked, what the containment looked like. That narrative is the
organisational memory that no external feed provides — and it was being discarded
at case close.

We built the ingestion path today. `SocKnowledgeIngestor` hooks into the existing
`SocCbrRetainService` — when a resolved incident stores its CBR record, the
ingestor also builds a prose chunk from the case snapshot and pushes it into a
per-tenant RAG corpus via `EmbeddingIngestor`. The chunk carries structured
metadata: alert type, source system, ATT&CK technique IDs. That metadata means
a future retrieval query for "credential dumping from Windows endpoints" will
surface not just the MITRE description of T1003, but also the last three times
this organisation dealt with credential dumping and what worked.

The per-tenant decision matters. ATT&CK is public knowledge — it lives under a
`__reference__` tenant so every customer shares one copy. Internal post-mortems
are organisational knowledge. Tenant A's ransomware playbook is irrelevant to
Tenant B and potentially sensitive. The corpus split was already implicit in the
design decisions from the RAG epic; making it explicit in `SocRagRetrieveService`
was a one-method change. `resolveCorpora(tenantId)` returns both the shared ATT&CK
corpus and the tenant-specific internal knowledge corpus. The retrieval worker
queries both in a single pass.

Alongside the ingestion work, we defined `TipIngestor` — the SPI for external
threat intelligence platforms. Recorded Future, CrowdStrike Intel, any TIP that
provides structured threat data can implement this interface to feed into the same
RAG corpus. The interface sits in `api/` (pure Java, no CDI) with a builder-pattern
config record and a result type that reports chunk counts and MindMap node links.
No implementation yet — this is the contract that future connector work plugs into.

The ATT&CK bundle download also landed. The real STIX bundle is 53MB and 26,000
objects — too large for git, so it's fetched at build time via a Maven profile.
The ingestion service already handled missing bundles gracefully; this just makes
the data available for deployment.

What's ahead: the internal knowledge corpus is empty until incidents start
resolving. The first few entries won't change retrieval quality much — CBR
similarity is still the primary signal for triage. But as the corpus grows, the
RAG results should start surfacing contextual advice that the structured CBR
features can't capture: "last time we saw this pattern, the initial severity
assessment was wrong because the lateral movement was masquerading as scheduled
tasks." That's the kind of organisational memory that turns individual analyst
experience into institutional knowledge.
