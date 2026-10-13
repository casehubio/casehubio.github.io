---
title: "The supplement that breaks the chain"
date: 2026-08-14
author: mdp
entry_type: note
subtype: diary
tags: [gdpr, art22, compliance-supplement, merkle, ledger, design-review]
projects: [casehub-aml]
status: draft
---

# The supplement that breaks the chain

GDPR Art.22 says if you make automated decisions that significantly affect people,
you document what the algorithm did and why. In AML, that's the triage decision —
the moment the system decides whether someone gets investigated further or cleared.

The platform already has `ComplianceSupplement` — a metadata attachment on ledger
entries that records algorithm reference, confidence score, rationale, contestation
URI, and whether human override is available. The clinical-trial harness uses it
for EU AI Act Art.12 compliance. AML doesn't use it at all.

The implementation looked straightforward: factory class for the supplement, attach
it to the triage entry, add an Art.22 section to the compliance evidence report.
The clinical pattern is clean — static factory methods, `entry.attach(supplement)`
before save, done.

Two things weren't straightforward.

**The sanitiser that shouldn't be global.** The epic mentioned a
`DecisionContextSanitiser` SPI for PII redaction. We designed one — then the
decision review pointed out that `ContentSanitiser` already exists in
`casehub-ledger-runtime` with the exact same signature. Parameter even named
`decisionContextJson`. The entire foundation SPI work evaporated.

But the plan review caught a subtler problem: making AML's IBAN-redacting sanitiser
a CDI bean that displaces the platform's `PassThroughContentSanitiser` would affect
every domain sharing the ledger. A future clinical-trial supplement with a legitimate
IBAN in its context would get silently redacted. The fix: plain class, instantiated
locally, no CDI. Domain-specific PII rules stay domain-scoped.

**The observer that corrupts the chain.** The spec review said Art.22 logic shouldn't
be coupled to the CBR observer — different concerns, different error boundaries,
different packages. Fair point. So we designed a separate `CaseOutcomeObserver` that
queries the profile entry back, attaches the supplement, and re-saves.

The plan review killed it. `LedgerEntry.canonicalBytes()` includes `supplementJson`.
The digest computed at the first save doesn't match the content after supplement
attachment. The Merkle chain quietly corrupts — no error at write time, only
discovered when `verify()` runs and reports chain inconsistency.

The fix is obvious once you see it: attach the supplement inside the existing
observer, before the single `repository.save()` call. One write, one digest,
chain intact. The factory and sanitiser remain separate classes — the concern
separation is in the components, not in the observer boundary.

The spec's own global constraint said it: "attach() must be called before save()."
We wrote the constraint, then designed around it, and the review caught us.

That's why reviews exist.
