---
title: "Incremental sync lands across CalendarPlatform and DocumentPlatform"
date: 2026-10-01
entry_type: note
subtype: diary
tags: [sync, calendar, document, google-api, spi]
projects: [casehub-connectors]
issue: 129
---

ContactsPlatform shipped with incremental sync back in #125 — `SyncResult<T>`, `SyncRequest`, and `SyncTokenExpiredException` as cross-SPI primitives sitting in `connectors-api`. The pattern worked: null token for full sync, stored token for incremental, HTTP 410 for expired tokens requiring full resync. CalendarPlatform and DocumentPlatform were next in line.

The calendar side maps cleanly onto Google's `events.list` API. Pass `syncToken` on the request, get `nextSyncToken` back in the response. Cancelled events come back with `status: "cancelled"` — we map those to `deletedIds` rather than regular items. One subtlety: when using `syncToken`, you can't set `singleEvents` or time bounds. The API switches from "give me events in this range" to "give me everything that changed."

The document side is different. Google Drive doesn't add sync tokens to `files.list` — it has a separate `changes.list` endpoint. Initial sync calls `changes.getStartPageToken()` to get a baseline token, then lists all files normally. Incremental sync calls `changes.list(pageToken)` and maps each `Change` entry — files with `removed: true` go to `deletedIds`, changed files get mapped to `DocumentSummary`. The token type is `newStartPageToken` rather than `nextSyncToken`, but the consumer-facing contract is identical: `SyncResult<DocumentSummary>` with the same fields.

Both Google implementations follow the `paginating-client-fail-soft` protocol — mid-pagination failures return accumulated results with a WARNING rather than throwing. The `MAX_PAGES` cap (20) prevents infinite loops. These aren't just defensive patterns; they're the difference between a sync that gives you 90% of the data and one that gives you nothing.

I also added `GoogleCredentialResolver` to `calendar-google`, aligning it with the per-user credential pattern from `contacts-google`. The existing `GoogleCalendarPlatform` baked credentials at construction time — one singleton, one user. The new constructor takes a resolver and builds `Calendar` service instances per-user via `buildService(userId)`. The flat SPI methods still use the pre-built singleton service; per-user routing happens when CalendarPlatform eventually gets user-scoped accessors like ContactsPlatform has.

The ref implementations gained monotonic version tracking — same `AtomicLong` + `VersionedRecord` + `deletedVersions` pattern that `contacts-ref` established. Each mutating operation (create, update, delete) increments the version counter. `changedSince(version)` and `deletedSince(version)` filter by that counter. Simple, testable, and it means any consumer can test sync flows without touching a Google API.

Three platforms now share the same sync contract. The fourth — EmailPlatform — is push-based (webhooks/IMAP IDLE), not poll-based, so it stays out of scope. BankPlatform's PSD2 consent lifecycle adds enough complexity that it was explicitly deferred. The infrastructure is there when someone gets to it.
