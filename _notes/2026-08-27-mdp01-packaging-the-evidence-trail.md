---
layout: post
title: "Packaging the Evidence Trail"
date: 2026-08-27
entry_type: note
subtype: diary
projects: [casehubio/qhorus]
tags: [compliance, eu-ai-act, reporting, prov-dm, graphql]
series: issue-402-compliance-evidence-export
---

# Packaging the Evidence Trail

Qhorus has had the raw audit data since the ledger landed — every message, every commitment state transition, every attestation verdict, every enforcement action. What it didn't have was a way to hand that data to an auditor in a format they could use. Article 12 of the EU AI Act requires deployers to maintain operational logs. We had the logs. We didn't have the export.

The compliance-report module packages five report types from data that already exists across the ledger, commitment store, causal graph service, and trust score engine. No new data collection — just composition and formatting of what Qhorus already tracks.

## Five reports, one optional module

Each report serves a different audit question:

| Report | Question it answers |
|--------|-------------------|
| **Attribution** | Who did what, in what order, and how trustworthy were they? |
| **Obligation** | Are agents fulfilling their commitments? |
| **Violation** | What got blocked, and why? |
| **Trust History** | How has an agent's reputation changed over time? |
| **Provenance** | Can I get a W3C-standard causal chain? |

Attribution was the most interesting to build. It starts with `CausalGraphService.buildGraph()` — which already gives you the cross-channel delegation tree for a correlation ID — then enriches each node with the trust score at that point, the attestation verdict on the ledger entry, and any `ComplianceSupplement` data from `LedgerComplianceReportService`. The result is a complete decision chain where every step has its evidence attached.

The trust score enrichment has a known limitation: until the ledger gets a snapshot table (tracked as ledger#203), we can only report the agent's *current* trust score, not the score at the time of the action. The field is nullable and `schemaVersion` signals which mode is active. Good enough for initial compliance needs — historical scores are a separate piece of infrastructure.

## The PROV-DM mapping

Provenance reports produce W3C PROV-JSON-LD, mapping Qhorus concepts to the PROV vocabulary. Agents become `prov:Agent` with `ledger:actor/` IRIs. Messages become `prov:Activity` with `ledger:activity/` IRIs. The IRI namespaces are shared with `LedgerProvSerializer` in the ledger module — same actor, same IRI, regardless of which export path produces it. HANDOFF messages map to `prov:Delegation`. Causal edges map to `prov:wasDerivedFrom`.

I chose to keep `ProvJsonLdMapper` as a static utility rather than a CDI bean. It takes a `CausalGraph` and returns a `Map<String, Object>`. No state, no dependencies, no lifecycle — the kind of thing that's easier to test and reason about as a pure function.

## Triple exposure and the dead branch

Reports are exposed three ways: REST with content-negotiation (JSON, CSV with RFC 4180 escaping, HTML with print CSS), GraphQL via `@McpDomain("qhorus")` resolvers, and MCP tools generated automatically from the GraphQL schema. The REST surface handles format rendering; the GraphQL surface handles structured queries and gets MCP for free.

Code review caught a copy-paste error in the REST layer — `getStoredReport()` had an `if/else` where both branches returned identical JSON responses. The content negotiation for stored reports was supposed to re-render from the stored JSON, but the implementation just returned the raw JSON regardless of the Accept header. Collapsed to a single return. Also caught missing null defaults on `from`/`to` query parameters — the GraphQL resolver defaulted to 30 days; the REST endpoints would NPE.

## What the enforcement gap reveals

The violation report has a structural limitation worth noting. `EnforcementExecutor` dispatches EVENT messages with telemetry containing `enforcement_action`, `violations`, `violation_sources`, `blocked_sender`, and `enforcement_mode`. But `LedgerWriteService.populateTelemetry()` only extracts predefined columns — `tool_name`, `duration_ms`, `token_count`, and a few others. The enforcement-specific keys are silently discarded.

This means the violation report can count enforcement events (by querying ledger entries with sender `system:enforcement`) but can't detail *what* was violated or *which* enforcement mode was active. The count is useful for aggregate compliance metrics. The detail would be useful for incident investigation. Fixing it properly means either extending the ledger's telemetry column set or storing enforcement telemetry in a separate structure — a decision for a future branch.

## The module shape

`compliance-report/` follows the optional module pattern established by `webhook-observer/` and `notification-bridge/`: classpath-presence activation, Jandex discovery for `@GraphQLApi` classes, entities in the qhorus named persistence unit. SPI types (`CompliancePostureProvider`, `CompliancePosture`, `PostureEntry`, `PostureStatus`) live in `api/` — the obligation report uses them to attach external compliance posture data from casehub-ops when it's on the classpath, falling back to `CompliancePosture.EMPTY` via the `@DefaultBean` when it isn't.

Scheduled generation uses `DigestSchedule.isFlushDue()` from the platform — the same mechanism that drives notification digests. Each schedule is a JPA entity with a JSON-serialised schedule expression. The scheduler sweeps hourly with per-schedule error isolation, so one broken schedule doesn't block the others.
