---
layout: post
title: "Four fixes and the orphaned audit entry"
date: 2026-05-29
type: phase-update
entry_type: note
subtype: diary
projects: [CaseHub Work]
tags: [quarkus, cdi, audit]
---

The backlog had 23 open issues. I wanted to clear the small ones — the XS and S
items with no external blockers, sitting unresolved. Four survive the filter.

Three are mechanical. Adding `SIGNAL_RECEIVED` to `WorkEventType` is a single enum
value; the wiring waits on a signal bus that doesn't exist yet, but the vocabulary
should be in place before the first consumer needs it. The `ClaimFirstStrategy` CDI
topology is two annotations and two code changes: both built-in worker selection
strategies were registered as ordinary CDI beans, and every call site worked around
the resulting `AmbiguousResolutionException` individually. Making `ClaimFirstStrategy`
`@Alternative @Priority(0)` fixes the root, but it also requires updating the switch
in `WorkItemAssignmentService.activeStrategy()` and the exclusion filter on
`@Any Instance<WorkerSelectionStrategy>` — all three must land together or the
strategy either activates for everything or activates for nothing. The third is a
test validating the migration customizer against `classpath:db/migration` — a path
no real consumer will configure after the scoped convention landed. The logic is
sound; the test proves the wrong case.

The fourth is more interesting. There's a `CREATE_DENIED` audit entry missing for
the case where the exclusion check rejects a direct assignee at `create()` time. The
problem: the audit entry needs a WorkItem ID, but the exclusion check fires before
the WorkItem has one. `@PrePersist` generates the ID — but `@PrePersist` only runs
when `workItemStore.put(item)` is called, and that call is what the exclusion check
prevents.

The fix is to pre-generate the UUID before the check runs. `@PrePersist` already
guards `if (id == null)`, so pre-setting is safe. On Java 21+, `UUID.randomUUID()`
uses DRBG-backed SecureRandom — sub-microsecond per call, no contention risk before
a write that takes hundreds of milliseconds. The consequence is an audit entry
referencing an entity that never persisted. The `audit_entry.work_item_id` column
has no FK constraint, so the insert works. The audit query endpoint needs to handle
the 404 gracefully — return entries even when the WorkItem is absent, treating them
as evidence of the attempt rather than evidence of the entity.

The alternative — relaxing `work_item_id` to nullable — introduces a discriminator
column problem and muddies what an audit entry means. Pre-generating the ID and
accepting the orphan is the cleaner design.
