---
layout: post
title: "Wiring the RAS pipeline — three silent failures"
date: 2026-07-30
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-soc]
tags: [ras, integration, silent-failure, casehub-engine]
series: issue-17-layer-1-alert-ingestion
---

*Part of a series on [#17 — Layer 1: Alert ingestion and case creation pipeline](https://github.com/casehubio/soc/issues/17). Previous: [Platform audit and vertical slice redesign](2026-07-29-platform-audit-vertical-slice.md).*

The Layer 1 goal was straightforward: fire a CloudEvent, have the RAS pipeline detect a critical SIEM alert, create a CaseInstance. The wiring itself was mechanical — add `casehub-ras` and `casehub-ras-persistence-memory` as dependencies, fix a handful of API renames that accumulated while SOC was dormant, write the integration test.

What made it interesting was the silence. Three separate failures produced no exceptions, no warnings, no test failures — just an empty result set where a case should have been.

## The tenant that wasn't there

`RasEngine` observes CloudEvents via `@ObservesAsync` and routes them to the `SituationEvaluator`. But it first checks for a `tenancyid` CloudEvent extension. If missing, it logs a single WARNING and drops the event. No exception, no CDI error — the event is "handled" successfully. Downstream, nothing happens. When the integration test's assertion failed with "expecting actual not to be empty," the natural suspects were situation definitions, ganglion registration, and trigger configuration. The actual cause: a missing three-character extension attribute.

Then a separate tenant issue. `CaseHub.startCase()` reads the tenant from `CurrentPrincipal`, not from the CloudEvent. The integration test was setting `tenancyid` on the event correctly, but `FixedCurrentPrincipal` (from `casehub-platform-testing`) defaults to a fixed UUID. The case was being created — just with the wrong tenant. `findAll("test-tenant")` returned empty because the case was stored under `278776f9-e1b0-46fb-9032-8bddebdcf9ce`.

## The case that rolled back

This was the best one. `DefaultCaseTrigger.fire()` returned a UUID. Case created, right? The `SituationEvaluator` even logged the trigger as successful. But `findAll()` returned nothing.

Claude found the answer buried in the test report's suppressed exception chain. `CaseStartedEventHandler.onCaseStarted()` serializes the case context to JSON via `WritableLayerImpl.asJsonNode()`. The internal `ObjectMapper` doesn't have `jackson-datatype-jsr310` registered. `DefaultCaseTrigger` automatically adds `context.detections()` to the case input — a `List<TimestampedDetection>` with `Instant` fields. Jackson can't serialize `Instant`, throws `InvalidDefinitionException`, the handler catches and suppresses it, and the entire case creation transaction rolls back. The UUID was generated but the case was never persisted.

The fix was `SocCaseInputContributor` — a CDI bean implementing the RAS `CaseInputContributor` SPI. It converts the raw `TimestampedDetection` objects to `Map<String, Object>` with string timestamps, and also extracts structured `alert` data for the case context. `DefaultCaseTrigger.buildInputData()` calls `data.putAll(contributor.contribute(...))`, so putting the same `"detections"` key overrides the raw objects. This is actually the right pattern regardless of the serialization bug — applications should control what enters their case context, not pass raw RAS internals through.

## What the pipeline looks like now

A CloudEvent with `CRITICAL` severity and a `tenancyid` extension enters the CDI event bus. `RasEngine` picks it up, finds the `soc-siem-alert-critical` situation registration, and delegates to `SituationEvaluator`. The `SiemAlertGanglion` (produced as a CDI bean via `SocGanglionProducer` — api/ is pure Java, no CDI annotations) classifies CRITICAL at 0.95 confidence. The threshold chain fires at >= 0.8. `DefaultCaseTrigger` matches the case definition by namespace, name, and version, `SocCaseInputContributor` sanitizes the input, and `CaseHub.startCase()` creates the `CaseInstance` with priority, source, alert data, and detection metadata in the context. The binding guards evaluate — `.alert != null and .iocEnrichment == null` is true — but no workers exist yet to execute them. That's Layer 2.

The version mismatch between the situation YAML (`caseVersion: "1.0"`) and the case definition (`version: "1.0.0"`) was its own silent failure — `DefaultCaseTrigger.findCaseHub()` threw "No CaseHub found" but the exception was caught inside `SituationEvaluator`, logged as SEVERE, and the method returned `false`. Another missing case with no test-level signal.

The platform API drift was expected — SOC was dormant while the platform evolved. `Ganglion.detect()` now returns `DetectionResult` directly instead of `Uni<DetectionResult>` (virtual threads migration). `GateRequired` gained a `QuorumConfig` parameter. `triggerConfig` was renamed to `triggerAction` in the RAS YAML schema. The `soc-brute-force` situation had to be deferred — its `BruteForceDetectorGanglion` doesn't exist yet.

Layer 2 adds the triage workers — six named `Worker` beans (rule + LLM for each of three capabilities), dual bindings per capability in the case YAML, and a blocking `AgentProvider` adapter for LLM workers. The pipeline is ready for them; the binding guards are waiting.
