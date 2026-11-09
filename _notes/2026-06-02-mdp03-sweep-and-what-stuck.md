---
layout: post
title: "The Sweep and What Stuck"
date: 2026-06-02
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-work]
tags: [workitems, exclusion, jpa, value-types]
---

The `issue-235-sxs-sweep` branch was a batch of seven small-to-medium items. Six shipped. One (#234 — route inbound messages to WorkItem creation) came off the list: blocked on `casehub-connectors-core` not being published and no resolved design for routing or speech-act classification. It lives in epic #79 now, which is the right home.

Of the six that did ship, two were mechanical migrations. `LabelDefinition.path` moved from `String` to the platform `Path` type, which required a `PathAttributeConverter` to keep the database column unchanged. The converter gets `@Converter(autoApply = false)` — if you use `autoApply = true` for a value type that also appears in DTOs and REST response records, you get unexpected conversions outside the JPA context. Once you know it, obvious; before you know it, confusing.

The other migration was the defaultPayload deep-merge — JSON merge semantics for `WorkItemTemplate.defaultPayload` when a `payloadOverride` is provided at instantiation time. Straightforward once specified.

The more interesting work was `excludedGroups`. Templates already supported `excludedUsers` (comma-separated actor IDs); this extends that to group names. The key decision: when does group membership get resolved? At WorkItem creation or at each claim attempt?

We went with snapshot at creation. Groups are resolved once via `GroupMembershipProvider.membersOf()` at instantiation and merged into `excludedUsers`. This keeps the claim path free of external I/O — `GroupMembershipProvider` is an SPI that production deployments can back with LDAP or SCIM2, which means latency and failure surface you don't want on every claim. The tradeoff is that a group membership change after WorkItem creation isn't reflected. For short-lived WorkItems, that's acceptable.

The expansion runs through a single `TemplateExpander` component. There are three places WorkItems get instantiated from templates — `WorkItemTemplateService.instantiate()`, `MultiInstanceSpawnService.createGroup()`, and `WorkItemSpawnService.spawn()` — and all three needed the same expansion logic. Centralising it in one injected component was the obvious call.

One infrastructure detail: `GroupMembershipProvider` has no production implementation on the classpath — `MockGroupMembershipProvider` is in `casehub-platform`, which is test-scope only. Without something, CDI startup fails in any deployment that doesn't supply its own provider. So `NoOpGroupMembershipProvider` goes into the runtime module as a `@DefaultBean`, returning empty sets. Production overrides it with `@Alternative @Priority(1)`. This pattern appears in a few SPIs now and is worth knowing.

The `ExclusionPolicy` example module got `ExpiringExclusionPolicy` — time-window conflict-of-interest exclusion encoding `userId:YYYY-MM-DD` in the `excludedUsers` field. An actor is excluded until the date passes, permanently excluded if there's no date, and excluded with a warning if the date is unparseable. It's the reference implementation for anyone writing a custom policy.
