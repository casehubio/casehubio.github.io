---
layout: post
title: "The compliance tab and the component that wouldn't bend"
date: 2026-08-23
entry_type: note
subtype: diary
projects: [casehubio/soc]
tags: [soc, compliance, blocks-ui, audit-trail, gdpr, dora, merkle]
series: issue-31-compliance-audit-views
---

The SOC web application has four sidebar tabs. Three were built. The fourth — Compliance & Audit — was a placeholder. Now it's not.

The compliance backend already existed: ledger entries for every investigation step, Merkle-hashed for tamper evidence, PII-sanitised before exposure. DORA response time reports computed from SLA windows. What was missing was the view layer — the part where an auditor actually looks at this data.

I expected the UI to be straightforward composition. casehub-blocks already ships `blocks-compliance-summary` and `blocks-gdpr-erasure-action`. Both turned out to be direct fits — point them at the right REST endpoint, and the data contracts match. The compliance summary takes a list of regulation/requirement/status objects and renders a sortable table with colour-coded badges. The GDPR erasure component handles the full flow: schema-driven form, confirmation dialog with danger styling, receipt display. The platform's `LedgerErasureService` does token-severing under the hood — the identity mapping is deleted, entries remain but become unlinkable. For GDPR Art.17, that's valid erasure.

The audit trail viewer was a different story. `blocks-audit-trail-viewer` exists and does exactly what an audit trail viewer should do — filterable entry table, Merkle verification banner, expandable row detail with attestation display. But it constructs its URLs as `${endpoint}/api/v1/ledger/entries?subjectId=...`. That path is hardcoded in a private method. The `endpoint` property controls the prefix, but the suffix is baked in. Worse, it requires `subjectId` — you can't browse across incidents, which is the whole point of a compliance audit view.

We built a SOC-specific component using the same primitives the platform component uses internally — `pages-table` for the data grid, `renderPropertyTree` for metadata display. Server-side filtering and pagination, step type badges with colour coding, per-entry Merkle proof verification via the existing `/proof/{entryId}` endpoint. Around 300 lines. The real design work was in the compliance summary computation — defining what DORA, SOC2, and NIS2 status badges actually mean in terms of ledger data. DORA maps directly to the existing SLA compliance percentages. SOC2 checks that containment decisions have an authoriser on record. NIS2 measures triage-to-promotion timing against a 30-minute window. Each produces a MET/PARTIAL/GAP/BREACHED badge based on explicit thresholds.

One surprise: the compliance backend had never been integration-tested. The `SocLedgerEntry` JPA entity wasn't registered with the `qhorus` Hibernate persistence unit — the library's package config only lists its own entities, not app subclasses. Unit tests with stubs passed fine. The first `@QuarkusTest` hit `UnknownEntityException: Could not resolve root entity 'SocLedgerEntry'`. The fix was one line in `application.properties`, but the failure mode is genuinely confusing — the entity compiles, the migration runs, the table exists. Hibernate just doesn't see it.

The platform's `CaseInstanceQuery` already handles pagination correctly — `Math.max(0, page)` to clamp negative values before they reach `setFirstResult`. We missed that initially. The JPA spec mandates `IllegalArgumentException` on negative arguments, which means an unvalidated `page=-1` from a client produces a 500 instead of a sensible clamp. Following the platform's own pattern fixed it.

Four sidebar tabs, all wired. The SOC incident response application has a complete UI: incident list with timeline and channels, analyst workbench with SLA indicators, trust dashboards with routing rationale, and compliance audit with Merkle verification and GDPR erasure.
