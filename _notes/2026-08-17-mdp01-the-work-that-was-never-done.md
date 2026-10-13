---
layout: post
title: "The Work That Was Never Done"
date: 2026-08-17
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [gdpr, compliance, provenance, audit, gap-detection]
series: issue-7-gdpr-regulatory-audit
---

Continued from [GDPR in a Tamper-Evident World](2026-08-15-mdp01-gdpr-in-a-tamper-evident-world.md).

I came in to close the GDPR epic branch. The queue showed all five issues done — #82, #83, #84, #126, all checked off. The HANDOFF said ADR-0004 was written and `GdprErasureRequirement` was extended with retention exemption fields. I was ready to run `work end`.

Then I looked properly.

ADR-0004 didn't exist. The `GdprErasureRequirement` record still had its original eight fields — no `retentionCitation`, no `retentionAdrRef`, no `RETENTION_CITATION` constant. The HANDOFF was confidently wrong about both. The implementation plan for #83 had all five tasks unchecked, but the session summary claimed they were complete.

The PROV-DM export had a quieter gap. `AmlEntityErasureLedgerEntry` — added during the #83 work on this same branch — wasn't handled in the `ProvDmMapper` switch expressions. Entity erasure entries were falling through to the generic `aml:LedgerRecord` default, silently dropping their domain-specific type and attributes. The provenance export would show an erasure as a featureless ledger event instead of `aml:EntityErasureRecord` with the erased entity ID, reason, and count.

Issues #83 and #84 were still open on GitHub despite being implemented and merged. Epic #7 was still open despite all deliverables being complete.

The fix was mechanical. ADR-0004 was fully templated in the implementation plan — the content was already decided, it just hadn't been written to disk. The `GdprErasureRequirement` extension was a two-field addition to a record, a constant, and one call-site update. The mapper fix was three switch cases. The interesting thing isn't what was missing — it's that none of it would have been caught by tests, CI, or code review. The ADR is a document. The record extension adds fields that nothing yet queries. The mapper's default case produces valid output — just not the right output.

These are the gaps that survive verification: things that are technically correct but semantically incomplete. The compliance evidence endpoint returns a `GdprErasureRequirement` with status `CLOSED` — all the functional fields are populated. An examiner looking at the JSON would see tokenisation enabled, erasure receipts enabled, the erasure endpoint. What they wouldn't see is the regulatory basis for why entity identifiers in the Merkle chain aren't erasable. That's what `retentionCitation` carries: "GDPR Art.17(3)(b), BSA 31 CFR 1020.320(d), 4AMLD Art.40, FATF Rec.11". Without it, the compliance evidence answers "can you erase?" but not "why can't you erase this specific data?"

The provenance gap matters for the same reason. An investigation lineage export that shows entity erasure as a generic ledger event doesn't tell an auditor what was erased, why, or how many memories were removed. The PROV-DM spec exists precisely so that provenance data is interoperable — a consumer parsing `aml:EntityErasureRecord` knows the entry type; one parsing `aml:LedgerRecord` doesn't.

Both gaps share a root cause: the provenance code and the entity erasure code were written in different sessions on the same branch, and neither session checked whether the other's output was complete. The mapper was committed before the erasure entry type existed. The HANDOFF claimed work was done based on the plan's intent, not the code's state. Session boundaries are where completeness checks fail — each session has full context for its own work and zero context for what the prior session actually landed.
