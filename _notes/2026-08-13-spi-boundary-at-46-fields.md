---
layout: note
title: "SPI boundary at 46 fields"
date: 2026-08-13
project: casehub-work
tags: [spi, architecture, records, quarkus]
entry_type: note
subtype: diary
---

The WorkItemStore SPI extraction landed today. The core move: pull the store interface and its return type out of the runtime module into `api/`, so consumers like `persistence-memory` can depend on the query SPI without dragging in JPA entities, Hibernate, and a mandatory datasource.

The interesting design question was what the SPI boundary type should look like. The JPA entity has 46 fields. All the existing SPI types in `api/` are immutable -- `WorkItemRef` (record), `WorkItemSummary` (record), `WorkItemCreateRequest` (final class with builder). So the answer was a 46-field Java record with a builder and `toBuilder()` for copy-and-modify.

The `version` field stays on the entity -- it leaks OCC semantics that don't belong in the SPI contract. The mapper handles bidirectional conversion at the persistence boundary: `toDomain()` produces the immutable snapshot, `updateEntity()` copies domain fields onto the managed entity (preserving version), and Hibernate's `@Version` check fires on flush. Same pattern for MongoDB with version-checked `replaceOne`.

The migration touched 60+ files across examples, integration tests, and platform modules. Most of it was mechanical -- change `WorkItemEntity` to `WorkItem`, add `()` to field access -- but the IntelliJ workspace made it adversarial. A rename refactoring on `WorkItem` in the work repo propagated across six peer repos (aml, clinical, engine, iot, devtown, life), incorrectly changing every `.id()` call on unrelated record types to `.id`. `JsonTypeInfo.Id.NAME` became `JsonTypeInfo.id.NAME`. The fix was surgical: discard the collateral damage, keep the legitimate import updates, commit per-repo.

The progress REST API documentation closes the other half of the branch -- 18 endpoints covering CRUD, state machines, rollback, step sequences, and SSE streaming. That was the straightforward part.

What this opens up: `persistence-memory` now depends only on `api/`, which means Claudony can embed WorkItem querying without CDI deployment errors. The engine-adapter and qhorus bridge already worked through `api/` -- this brings the in-memory store to the same level.
