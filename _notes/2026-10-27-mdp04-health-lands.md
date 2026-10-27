---
layout: post
title: "The Eleventh SPI — Health Lands in Connectors"
date: 2026-10-27
entry_type: note
subtype: diary
projects: [casehubio/connectors]
tags: [spi, health, nhs, architecture, platform]
---

# The Eleventh SPI — Health Lands in Connectors

The connectors repo now has eleven platform SPIs. The newest — HealthPlatform — covers provider search, appointment booking, and medical records. It's the first SPI where the domain model carries real regulatory weight: NHS ODS codes for provider identity, clinical urgency levels on referrals, a sealed record hierarchy that forces exhaustive handling of lab results, referrals, and prescriptions.

## Why Three Capabilities, Not Two

The obvious split was "find a provider" and "do patient stuff." But NHS APIs don't work that way. Provider directories (think NHS Service Search), appointment booking (e-Referral Service), and clinical records (GP Connect) are separate services with separate access models. A directory service might exist without any booking capability. A records provider might not know about appointments at all.

So HealthPlatform gets three nested capability interfaces — `ProviderSearch`, `AppointmentBooking`, `MedicalRecords` — each independently declared via `supports()`. This matches the TravelPlatform pattern (four capabilities for transport and accommodation) rather than CalendarPlatform's flat interface, because partial implementations are a real scenario.

The interesting scoping decision: provider search is public (no userId), while booking and records are user-scoped. A directory lookup doesn't need patient context. An appointment does. This follows the same split TravelPlatform uses — `transportSearch()` vs `transportBooking(userId)`.

## The Sealed Record Hierarchy

Medical records needed a type system, not a generic bag of attributes. A lab result has a test name, a reference range, and a status (pending, final, corrected). A referral has an urgency level (routine, urgent, two-week-wait, emergency) and a destination specialty. A prescription tracks repeats remaining and the next due date.

```java
sealed interface MedicalRecord
    permits LabResult, Referral, Prescription {
    String recordId();
    LocalDate date();
    String providerName();
}
```

The sealed hierarchy forces exhaustive pattern matching in callers. When a life worker handles a medical record, the compiler ensures it handles all three types — no silent fall-through when a fourth type is added later.

The alternative was a generic `Map<String, Object>` attributes bag. That would have been simpler to implement but pushed all the type knowledge into stringly-typed lookups in every consumer. The life workers need to act on specific record types — prescription renewals have different logic from referral status checks — so the type system earns its place.

## ProviderId and Coordinates

Two model decisions that ripple further than they look.

`ProviderId(String scheme, String value)` replaces a bare string ID. The `scheme` field — `"ods"` for NHS ODS codes, `"npi"` for US NPI numbers — means the SPI doesn't hardcode NHS assumptions into its contract. The ref implementation seeds with `scheme="ods"` and realistic London GP surgeries. A non-UK implementation would use a different scheme without touching the interface.

`Coordinates` got extracted from `location-spi` to `connectors-api` as a shared primitive. Both HealthPlatform's `Provider.location` and LocationPlatform's `Place.location` now share the same type, so workers composing the two SPIs don't need to convert between structurally-identical-but-type-incompatible coordinate records. The extraction touched sixteen files across four modules — mechanical, but the kind of cross-cutting cleanup that only happens when a new consumer forces the issue.

## What This Opens Up

Three casehub-life workers — `book-appointment`, `find-alternative`, `provide-care` — were blocked on health-specific APIs. They had CalendarPlatform for scheduling but no way to search providers by specialty and location, no way to interact with booking systems, no way to read medical records. That's now unblocked.

The appointment status lifecycle (REQUESTED → CONFIRMED → COMPLETED/CANCELLED/NO_SHOW) with CDI events follows the PaymentStatusChanged pattern from bank-spi. The ref implementation confirms immediately; a real NHS integration would return REQUESTED and transition asynchronously. The life workers don't need to know which — they observe the CDI event either way.
