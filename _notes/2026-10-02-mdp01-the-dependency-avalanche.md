---
layout: post
title: "The dependency avalanche"
date: 2026-10-02
entry_type: note
subtype: diary
projects: [casehubio/soc]
tags: [dependencies, snapshot-drift, cdi, quarkustest, resilience-testing]
series: issue-60-quarkustest-attck-ingestion
---

I came in to write @QuarkusTest integration tests for the ATT&CK ingestion
pipeline. qhorus#438 — the CDI deployment error that had blocked this work for
weeks — was closed. Clean path forward, or so it seemed.

The first `mvn clean` exposed the real situation. The `-U` flag pulled fresh
SNAPSHOT dependencies, and the ledger module had been split from a monolith into
`casehub-ledger-api`, `casehub-ledger-jpa-common`, and `casehub-ledger-core`.
Every import path changed. `JpaLedgerEntry` moved from
`io.casehub.ledger.runtime.model.jpa` to `io.casehub.ledger.jpa`. The trust
score repository moved. `WorkerDecisionEntry` and `CaseLedgerEntryRepository`
were removed entirely — no replacement API visible.

The engine API also changed: `CaseInstanceRepository.findByUuid()` went from
returning `CaseInstance` to `Optional<CaseInstance>`. Five files across three
packages broke on the same pattern.

We fixed 20 compilation errors across 10 files. The import renames were
mechanical. The `Optional` wrapping was a find-and-replace. The interesting
decision was what to do about `WorkerDecisionEntry` — the class that
`SocTrustService.getRoutingRationale()` and `SocAttestationService.processOutcome()`
depended on is gone from the new ledger API with no obvious replacement. I
stubbed both methods with log warnings and filed #62 to track the full
restoration once the ledger team documents the new query surface.

With the build green, I tried @QuarkusTest. The Jandex fix from ledger#215
cleared the entity indexing error, but revealed 84 CDI deployment problems —
all neocortex RAG config beans (`RagConfig`, `IngestionConfig`,
`CorpusStorageConfig`) unresolvable. We added test defaults in
`application.properties`, a Jandex index directive, and `@DefaultBean`
alternatives for `CaseContextRetriever` and `DedupIngestionConfig`. That
brought it down to 71 — all `CurrentPrincipal` ambiguity from the platform's
test fixture lacking CDI priority annotations. Filed parent#520.

Each fix revealed the next layer. That's the nature of CDI wiring in a
multi-module Quarkus application — you don't see the full dependency graph
until augmentation runs, and augmentation stops at the first error.

The resilience tests landed as unit tests with `InMemoryMindMapStore` and
`InMemoryEmbeddingIngestor`. Seven scenarios from the design spec: crash
recovery with null root nodes, RAG failure isolation (MindMap survives when
the embedding store throws), deleteCorpus failure preventing duplicate
ingestion, capability guard for NoOp stores, disabled config, version-stamped
re-ingestion, and the error boundary that keeps the app starting even when
ingestion throws. All pass. When parent#520 lands, upgrading these to
@QuarkusTest is a one-annotation change — the test logic stays identical.
