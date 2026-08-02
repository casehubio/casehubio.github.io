---
layout: post
title: "Duration Belongs With the Outcome"
date: 2026-08-01
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [sla-calibration, domain-model, cbr]
---

The SLA calibration system already knew how long a PR review took end-to-end — median, min, max across historical precedents. What it couldn't tell you was how long each *capability* took. Code analysis: three minutes. Security review: twelve. That breakdown was sitting in the data, unused.

Each memory fact in the CBR retrieval pipeline carries a `capability` attribute and a `createdAt` timestamp. The case start time is known from the candidate vector. Per-capability duration is just `fact.createdAt - startedAt`. The data was flowing through `enrichOutcomes()` and being thrown away — only the latest timestamp across all capabilities was kept for the total case duration.

The question was where the duration belongs in the model. `CapabilityOutcome` held `outcome` and `detail` — what happened, but not how long it took. A parallel `Map<String, Duration>` on `Precedent` would have worked, but it means two maps with the same keys, two things to keep in sync, and every consumer choosing which to read. Duration is a property of the capability's execution. It belongs with the outcome.

Once `CapabilityOutcome` carries duration, the rest follows. `SlaEstimator` already iterates over capability outcomes per precedent — computing per-capability median/min/max is the same algorithm applied per key. The overall case stats and the per-capability stats share the same shape (median, min, max, sample count), so `DurationStats` falls out as a natural extraction. `SlaEstimate` composes one for the overall and a map of them for the breakdown.

The design review caught something I'd missed: the existing `(capability, scope_path, computed_at DESC)` index can't serve `findLatestCalibration` — that query filters by scope path first, and the index leads with capability. A migration was needed after all. The review also flagged that zero-duration filtering would silently drop fast capabilities like style-check. For overall case duration, zero means noise. For a per-capability measurement, zero is legitimate data — a capability that completes within the same clock tick as case start is real, just fast. The filter was relaxed to exclude only negatives.

Storage was simpler than expected. The `sla_calibration_record` table already has a `capability` column. Today it stores one row per calibration with `capability = "pr-review"`. Per-capability rows use the same table with `capability = "code-analysis"`, `"security-review"`, etc. — same `caseId`, same `computedAt`, different capability key. No new tables, just more rows and a scope-leading index.

There's a known bias: `startedAt` is the feature-vector emission timestamp, not the actual case start. The vector is emitted after `startCase()` returns, so all durations are systematically underestimated by a few milliseconds. For SLA calibration operating at minute-scale precision, this is noise — but it's documented rather than hidden.

The per-capability data is now persisted but not yet consumed. The next step is per-capability SLA overrides — using this breakdown to set different time expectations for different types of review work. That's where the granularity pays off: a security review taking twelve minutes is fine; code analysis taking twelve minutes is a signal something's wrong.
