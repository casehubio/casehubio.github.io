---
layout: post
title: "The timestamp that lied on every API call"
date: 2026-08-18
entry_type: note
subtype: diary
projects: [casehub-soc]
tags: [soc, rest-api, data-integrity, platform-gap]
---

The SOC incidents list endpoint had a `createdAt` field that returned `Instant.now()`. Every time you called the API, every incident had a different creation timestamp. It was a placeholder from the initial wiring — `SocIncidentResource.toSummary()` needed a timestamp and didn't have one, so someone reached for `Instant.now()` and moved on.

The obvious fix was to use `CaseInstance.getCreatedAt()`. Except that method didn't exist.

`CaseInstance` — the core domain object for every case in the platform — had no creation timestamp. The only timestamp available was `CaseMetaModel.getCreatedAt()`, which records when the case *definition* was deployed, not when the instance was created. A case definition is shared across all instances of that type. Every incident-investigation case points to the same meta model, so `CaseMetaModel.getCreatedAt()` returns the same value for all of them — the date the SOC module was deployed, not the date any particular incident was reported.

The engine's own `CaseInstanceResponse.from()` was using this value and labelling it "Case creation timestamp." The same misleading pattern, built into the platform.

I filed engine#919 to add a real `createdAt` field to `CaseInstance` — field, JPA column, set at case start time. It landed the same day. With that in place, the SOC fix was a one-line change: `Instant.now()` became `ci.getCreatedAt()`.

The test that caught it cross-checks the API response against the repository value directly — not just stability across calls. If someone later introduces another placeholder, the test fails because the values don't match, not because they changed between requests.

A small observation: platforms accumulate these gaps silently. `CaseMetaModel.getCreatedAt()` was "close enough" that nobody noticed the engine's own response DTO was returning the wrong thing. The SOC needed a real per-instance timestamp, so the gap surfaced. The fix was trivial once the field existed — the hard part was recognising it didn't.
