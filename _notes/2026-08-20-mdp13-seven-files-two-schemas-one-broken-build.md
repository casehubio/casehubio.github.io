---
title: "Seven files, two schemas, one broken build"
date: 2026-08-20
author: Mark Proctor
entry_type: note
subtype: diary
tags: [flyway, migration, upstream-api, casehub-work, snapshot]
project: casehub-devtown
issue: 189
---

devtown had accumulated seven incremental Flyway migrations for a database that doesn't exist yet. No production deployment, no live data to migrate — just a growing chain of `ALTER TABLE` and `CREATE INDEX` scripts that evolved the schema over months of development. Platform#226 flagged it: collapse them while collapsing is free.

The consolidation itself was straightforward. Read all seven, mentally apply them in order, write the final state as two initial schemas: `V1` for domain tables (merge queue entry, batch, SLA calibration), `V2500` for ledger subclass tables (merge decision, contributor outcome). Two files instead of seven. V2500 rather than V2000 because the engine-ledger already owns V2000 on the shared Flyway locations path — a collision we discovered when H2 threw `Found more than one migration with version 2000` at test startup.

The real work was everything else. When we ran `mvn test` to verify, the build wouldn't compile. Not because of the migrations — those were fine — but because upstream SNAPSHOT dependencies had drifted while nobody was looking.

`casehub-work` had moved `WorkItemLifecycleEvent` from `io.casehub.work.runtime.event` to `io.casehub.work.api`. The `WorkItem` class changed from a mutable JPA entity to an immutable Java record, which meant every `workItem.id` became `workItem.id()` and every `new WorkItemEntity()` became `WorkItem.builder().id(...).build()`. `MessageReceivedEvent` in qhorus grew two new constructor parameters (`actorType` and `topic`). The engine added `CallbackActionRiskClassifier` with the `@RiskClassifier` qualifier, creating a CDI ambiguity with devtown's own classifier. And the connectors module started requiring Signal, Twilio, and WhatsApp config properties that devtown doesn't use.

Each fix was mechanical. But they cascaded — fixing the compilation revealed test compilation errors, fixing those revealed CDI deployment errors, fixing those revealed missing config properties that appeared one at a time because Quarkus validates config properties lazily during startup. Fix one, run, hit the next.

The ledger had the most interesting bug. `JpaLedgerEntryRepository.findPeerAttestationsByAttestorIds()` queried `a.tenancyId` — a field that doesn't exist on `LedgerAttestation`. The entity joins to `LedgerEntry` for tenancy filtering (every named query on the entity does `JOIN LedgerEntry e ON a.ledgerEntryId = e.id` and filters on `e.tenancyId`), but this programmatic query skipped the join. Same bug in `findPeerAttestationPairCounts`. Two queries, same mistake. We fixed both in the ledger source, but then hit a second gotcha: `mvn install` in the ledger repo installed to a slot-local `.m2` directory (via `.mvn/maven.config`), not the global `~/.m2` that devtown resolves from. The fix was invisible until we passed `-Dmaven.repo.local=~/.m2/repository` explicitly.

The build went from uncompilable to 610 tests passing, zero failures. The migration consolidation was the stated goal; the upstream alignment was the price of admission for verifying it.

What this surfaces is a recurring tension in the casehub multi-repo setup: SNAPSHOT dependencies drift silently. Nothing breaks until someone runs the full test suite — which can be weeks after the upstream change landed. The individual API changes are each reasonable (record migration, constructor evolution, new CDI beans), but they compound. A project that hasn't been touched in a fortnight can accumulate half a dozen breaking changes from its transitive dependency tree, none of which are visible in its own git log.
