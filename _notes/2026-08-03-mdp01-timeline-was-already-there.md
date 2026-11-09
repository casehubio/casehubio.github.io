---
layout: post
title: "The Timeline Was Already There"
date: 2026-08-03
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [blocks-ui, timeline, audit-trail, ledger]
---

The Overview tab in the AML workbench had a placeholder: "Case Timeline (pending blocks-ui `<case-timeline>`)". I'd been carrying it since the workbench UI was first built — a deferred dependency on a component that didn't exist yet in blocks-ui.

Turns out it did. `blocks-timeline` has been in blocks-ui for a while — a pluggable timeline component with a `TimelineStrategy` pattern, vertical/horizontal/compact layouts, category filtering, and expand/collapse for detail. The `eventChronologyStrategy` already maps engine event logs to timeline nodes. What we needed was an AML-specific strategy that understood investigation milestones.

The harder problem was upstream. The audit trail API (`GET /api/investigations/{caseId}/audit-trail`) returns `AuditTrailEntryResponse` records with an `entryType` field mapped from `LedgerEntryType.name()`. Every AML domain entry — case opened, compliance review, SAR officer decision — comes back as `"EVENT"`. The qhorus specialist messages are `"EVENT"`. Engine entries are `"EVENT"`. There's no way to tell what actually happened.

The fix was exposing what JPA already knows. Each ledger entry subclass carries a `@DiscriminatorValue` annotation — `"AML_CASE_OPENED"`, `"QHORUS_MESSAGE"`, `"AML_SAR_OFFICER_REVIEWED"`. We added a `discriminator` field to the response record plus a `domainFields` map that extracts subclass-specific data via `instanceof` dispatch. The annotation read has a null guard so an unrecognised entry returns the class simple name and an empty map — forwards-compatible.

The interesting part of the strategy is qhorus message pairing. A specialist dispatch is a COMMAND message; its completion is a DONE or DECLINE. These arrive as separate ledger entries sharing a `correlationId`. The strategy groups them: COMMAND + DONE becomes "Entity Resolution completed", COMMAND + DECLINE becomes "OSINT Screening declined", unpaired COMMAND becomes "Pattern Analysis in progress". Each milestone node holds the raw entries as expandable detail — click to see the full ledger data underneath.

Different investigation paths produce structurally different timelines. A `HIGH_RISK_JURISDICTION` flag triggers the SAR path: entity resolution, pattern analysis, OSINT screening, SAR drafting, a gate WorkItem for MLRO sign-off, compliance review, case closed. A `STRUCTURING` flag scores below the SAR threshold and exits via `investigation-cleared` — no gate, no compliance review. The timeline handles both without special-casing: it renders whatever entries exist, groups what it can, and passes the rest through as standalone milestones.

The enriched response is backwards-compatible — the Audit tab table ignores the new fields. And any casehub-pages consumer that wants a timeline over the audit trail can now use the same API. The strategy pattern means the AML-specific interpretation lives in the AML app, not in blocks-ui.
