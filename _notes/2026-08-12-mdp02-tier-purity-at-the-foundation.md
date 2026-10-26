---
title: "Tier purity at the foundation"
date: 2026-08-12
type: diary
project: casehub-ledger
issue: "casehubio/ledger#189"
tags: [jpa, orm-xml, tier-purity, architecture]
voice: we
---

The ledger's api/ module had 39 JPA annotations scattered across five model classes.
`@MappedSuperclass`, `@Column`, `@Id`, `@Enumerated`, `@Transient` — all of them
leaking `jakarta.persistence-api` into a module whose entire purpose is to be a
zero-dependency domain contract. Every consumer that depended on `casehub-ledger-api`
got JPA transitively, whether they wanted it or not.

We stripped all 39 annotations and moved the mappings to `META-INF/orm.xml` in the
runtime module. Five `<mapped-superclass>` declarations, one for each api/ class.
The issue flagged a risk (T1.2a) that Quarkus's Jandex build-time scanner might
not discover cross-JAR orm.xml declarations. False alarm — 899 tests across a
3-JAR chain (api, runtime, engine-ledger) passed without incident.

With the annotations gone, we removed `jakarta.persistence-api` from api/pom.xml
entirely and added a `TierPurityTest` that scans source files for any
`jakarta.persistence` import. The compiler enforces the absence; the test makes the
intent explicit.

Then `domainData` — a `Map<String, Object>` field on `LedgerEntry` for remote
entries in centralized mode. Typed subtypes keep using `domainContentBytes()` for
their join-table fields; remote apps that can't provide JPA subclasses use domainData
instead. The two mechanisms are complementary.

The canonical bytes integration was straightforward: Jackson's
`ORDER_MAP_ENTRIES_BY_KEYS` gives us deterministic sorted-key output without pulling
in a JCS library. The column type went through one iteration — JSONB first, then TEXT
after PostgreSQL rejected the `varchar` binding from the `AttributeConverter`. The
data is identical JSON either way; JSONB indexing is a production-specific
optimization.

The ledger api/ is now genuinely tier-pure. No framework annotations, no framework
dependencies. The domain model is just Java.
