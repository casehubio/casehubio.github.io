---
title: "The compliance layer that almost ate its own data"
date: 2026-08-11
author: mdp
type: diary
tags: [compliance, ledger, merkle, dora, pii, jpa, audit]
issue: 24
epic: 21
status: published
---

The architecture document says six `JpaLedgerEntry` subclasses — one per investigation step. `AlertTriageLedgerEntry` with typed severity fields and NOT NULL constraints. `ContainmentDecisionLedgerEntry` with a schema-enforced `approverId`. The compliance argument for this is real: SOC2 Type II auditors prefer structural guarantees over application-level promises. A database column that can't be null is a stronger statement than code that always fills it in.

We built one entity instead.

`SocLedgerEntry` has two typed columns: `incidentId` and `stepType`. Everything else — alert severity, approver identity, containment action, resolution outcome — lives in the `metadata` JSON field. The platform's `canonicalBytes()` already includes `metadata` in the Merkle hash, so tamper evidence isn't weakened. The `CaseLedgerEntry` in engine-ledger follows exactly this pattern: one entity, `commandType` and `eventType` as string discriminators, all lifecycle events through a single join table.

The trade-off is explicit: we lose schema-level NOT NULL enforcement on compliance-critical fields. The `SocLedgerEntryWriter` compensates with application-level validation — a metadata field map per step type, checked before `LedgerEntryRepository.save()` runs. If `approverId` is missing from a `CONTAINMENT_DECISION` entry, the writer throws `IllegalStateException` and the entry never enters the Merkle chain. Not as clean as a database constraint, but cheaper to evolve in a pre-release application where the step types are still stabilising.

The decision review pushed back hard on this. The reviewer pointed out that the platform has *multiple* `JpaLedgerEntry` subclasses (`CaseLedgerEntry`, `WorkerDecisionEntry`, `MessageLedgerEntry`), each with typed domain fields — the pattern isn't "one class per application" but "one class per domain concern." Fair point. But SOC investigation steps are phases of the same concern, not independent domain concepts. `CaseLedgerEntry` doesn't split into `CaseStartedLedgerEntry` and `CaseSuspendedLedgerEntry` — it uses string fields to differentiate. I kept the single-entity approach.

The plan review caught something more dangerous. The compliance service's `incidentTimeline()` method queries ledger entries, sanitises the `metadata` field (stripping IPs and emails), and returns them. The original implementation mutated the `metadata` field directly on the JPA entity objects. Those entities came from a `@Transactional` repository method, which means the Hibernate persistence context was still open. On transaction commit, dirty checking would flush the sanitised values back to the database — permanently replacing `10.0.0.1` with `[REDACTED-IP]` in the audit trail. The compliance endpoint would have silently destroyed the data it existed to protect.

The fix: return detached copies. Create a new `SocLedgerEntry` instance for each result, copy all fields, sanitise only the copy's metadata. The original managed entity is never touched. A test confirms it: assert the returned object is `isNotSameAs` the original, and assert the original's metadata is unchanged.

The PII sanitiser itself is fail-closed — if regex processing fails for any reason, it returns `[SANITISATION_FAILED]` instead of the original data. An incorrect compliance report is worse than an unavailable one. This reversed the initial design, which was fail-open for availability.

The write path splits into two observers. `SocResolutionLedgerObserver` implements the `CaseOutcomeObserver` SPI — the same pattern as `SocAttestationService` from the trust layer. It fires at case completion and writes the `INCIDENT_RESOLVED` entry. `SocIncidentLedgerObserver` observes `CaseLifecycleEvent` directly via CDI async, extracting `incidentStatus` from the case context snapshot to create `ALERT_TRIAGE` and `INCIDENT_PROMOTED` entries. Both share a `SocLedgerEntryWriter` that owns sequence number assignment, metadata validation, and the `LedgerEntryRepository.save()` call.

The DORA response time report aggregates across these entries: group by incident, compute durations between `ALERT_TRIAGE.occurredAt` and `INCIDENT_RESOLVED.occurredAt`, aggregate by priority (extracted from the triage entry's `assignedSeverity` metadata field), and compare against SLA windows from `SocPreferences`. The SLA windows — 15 minutes for P1, 1 hour for P2, 4 hours for P3, 24 hours for P4 — use the platform's `PreferenceKey<DurationPreference>` API, so they're configurable per tenant without code changes.

Epic 3 is complete. Trust attestations feed the Bayesian routing model. CBR retains resolved incidents and retrieves similar ones for future triage. The compliance layer provides Merkle-verified audit trails and DORA metrics. The three layers compose: an incident resolves, the attestation observer scores the workers, the CBR observer stores the case, and the compliance observer records the ledger entry — all firing independently from the same `CaseOutcomeEvent`.
