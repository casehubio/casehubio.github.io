---
layout: post
title: "The module that was never just for testing"
date: 2026-06-07
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-work]
tags: [persistence, cdi, thread-safety, module-structure]
---

Five in-memory store implementations have lived in `casehub-work-testing` since
the project's first weeks. The name said test artifact. The CDI annotation —
`@Alternative @Priority(1)` — said Tier 2 override, the same slot as MongoDB.
Neither was right.

The stores are legitimate persistence backends. Someone evaluating CaseHub Work
without PostgreSQL should be able to add a single dependency and get a working
system — data lost on restart, acceptable for demos and local evaluation. A
module called `testing` forces that person to add a test artifact to their
production classpath. That's wrong semantics, and it was hiding a real bug.

The bug: `InMemoryWorkItemStore` and `MongoWorkItemStore` were both
`@Alternative @Priority(1)`. Any deployment with both modules on the classpath
would get `AmbiguousResolutionException`. Nobody had hit it because nobody had
tried both together — but the collision was there, waiting.

The fix required thinking about what CDI priority actually means in a platform
with pluggable persistence. The platform protocol defined three tiers: Tier 0
(`@DefaultBean`, no-op fallback), Tier 1 (`@ApplicationScoped`, JPA), Tier 2
(`@Alternative @Priority(1)`, MongoDB). The in-memory stores don't fit any of
these. They need to beat everything — JPA and MongoDB — when present. That's a
fourth tier.

I set the new tier at `@Priority(100)` instead of the minimum `@Priority(10)`.
The gap matters: Redis at Priority(50) or SQLite at Priority(25) can slot in
later without renumbering. One-time decision that costs nothing now and saves a
migration later.

There's a semantic inversion worth noting. Tiers 0 through 2 follow a natural
progression — higher priority means more capable backend. Tier 3 breaks that.
The in-memory store is the *least* capable backend (no durability) with the
*highest* CDI priority. It wins not because it's better, but because adding it
to the classpath is a deliberate override — you want ephemeral storage, and it
must displace whatever production backend is already wired in.

The thread-safety work was straightforward but satisfying. Four of five stores
used `LinkedHashMap` and were documented as "Not thread-safe — designed for
single-threaded test use only." Production deployment means concurrent requests.
`ConcurrentHashMap` was the right choice — its weakly consistent iteration gives
READ COMMITTED semantics, matching what JPA stores provide on PostgreSQL.
`synchronized` would have given SERIALIZABLE, which is stricter than the database
stores. Semantic consistency across backends matters more than maximum safety
within one.

The `InMemoryAuditEntryStore` needed more than a data structure swap. The old flat
`ArrayList` forced O(n) scan on every `findByWorkItemId()` call. I restructured
it to `ConcurrentHashMap<UUID, CopyOnWriteArrayList<AuditEntry>>` keyed by
workItemId. `findByWorkItemId()` is now O(1) lookup plus a sort of the per-item
list (typically 3–10 entries). The copy-on-write cost is bounded per work item,
not per store. `query()` still does full iteration — `AuditQuery` has no
`workItemId` field, so there's nothing to key on.

One gotcha from the thread-safety work: `store.getOrDefault(workItemId, List.of())`
looks like the natural empty-case pattern, but it fails when the map's value type
is `CopyOnWriteArrayList<AuditEntry>`. Java's type inference resolves `List.of()`
as `List<Object>`, which doesn't unify with `CopyOnWriteArrayList<AuditEntry>`.
The compile error points at a lambda field access downstream, not at the
`getOrDefault` call. An explicit null check and early return was the clean fix.

The review also surfaced that the SPI Javadoc on `WorkItemStore` and
`AuditEntryStore` had been wrong since creation. Both documented the tier ladder
as `@DefaultBean (mock/in-memory) < @ApplicationScoped (JPA) < @Alternative
@Priority(1) (NoSQL)` — but the in-memory stores were never `@DefaultBean`. They
were `@Priority(1)` from day one. The Javadoc described the intended architecture;
the implementation never matched. Correcting this across all five SPIs was part
of the same change.

The branch landed as four commits after squash: spec and plan, module extraction
(git mv + package rename + CDI annotation), thread-safe data structures, and
consumer migration with documentation. Five issues were filed during the work —
MongoDB completeness (#253), ephemeral deployment integration test (#254), minor
review findings (#255), parent repo deep-dive sync (parent#195), and protocol
update for Tier 3 (garden#2). The module is done; the ecosystem ripples will take
a few more sessions.
