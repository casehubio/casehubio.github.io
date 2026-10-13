---
title: "The Contract Before the Caller"
date: 2026-08-27
entry_type: note
subtype: diary
author: mdp
projects: casehub-qhorus
tags: [compliance, judgment, ledger, telemetry, merkle]
issue: 413
---

# The Contract Before the Caller

Engine #998 will add judgment provenance events — YIELDED, RESPONDED, VERIFIED, ESCALATED — but it hasn't landed yet. The question was whether to wait for the engine to define those events, or to define the contract from the consumption side and let the engine implement against it.

I chose consumption-first. The compliance module knows what data it needs for EU AI Act reports. The engine knows how to dispatch EVENTs through qhorus channels. Defining the telemetry field names, column types, and query patterns here means engine #998 has a target to hit rather than a blank canvas.

## What the contract looks like

Four `JudgmentEventKinds` constants in `casehub-qhorus-api` — `YIELDED`, `RESPONDED`, `VERIFIED`, `ESCALATED`. Both the engine and the compliance module reference the same class. A typo in either side breaks at compile time, not at "why are my reports empty" time. This is a small thing that eliminates an entire class of integration failure.

The four constants map to `toolName` values on EVENT messages. We added four dedicated columns to `MessageLedgerEntry` — `judgment_id`, `judgment_type`, `verification_outcome`, `evidence_quality` — extracted from telemetry JSON at write time, queryable in SQL. The alternative was a raw `telemetry_json TEXT` column with in-memory parsing, but the decision review caught this: for three known fields, dedicated columns are strictly better. SQL-filterable, type-safe, schema-documented.

## The Merkle backward-compat trick

The hardest part was getting the new columns into `domainContentBytes()` without breaking existing Merkle chains. The method uses `String.join("|", field1, field2, ..., field14)` — adding four more fields changes the hash for every existing entry.

We solved it with a tagged suffix: `|J:` followed by the four judgment fields, pipe-separated. The suffix is only appended when any judgment field is non-null. Existing entries — all judgment fields null — produce the exact same hash as before. New judgment entries get a distinct, collision-free encoding. The `J:` tag prevents any field value from accidentally producing the same byte sequence through a different field combination.

## Two reports, not two wrappers

The decision review pushed back on whether JUDGMENT_ATTRIBUTION and JUDGMENT_FULFILLMENT were genuinely distinct from the existing Attribution and Obligation reports. They share infrastructure — both use `CausalGraphService`, trust scores, Merkle roots. But they serve different entry points and aggregation patterns.

JUDGMENT_ATTRIBUTION takes a `judgmentId` and builds a judgment lifecycle timeline (YIELDED→RESPONDED→VERIFIED/ESCALATED) alongside the message causal graph. The timeline and the graph are complementary views — one tracks the judgment lifecycle, the other tracks the underlying COMMAND/RESPONSE message flow.

JUDGMENT_FULFILLMENT aggregates by judgment type and by caller — acceptance rates, escalation rates, evidence quality scores. The query uses SQL aggregation (`GROUP BY judgment_type, tool_name, verification_outcome`), not bulk loading. Pending judgments follow the `ObligationReport.stillOpen` pattern: all currently open, unbounded by the time window, because a judgment that's been waiting for three weeks matters more than one opened yesterday.

## What surprised me

The `HtmlReportRenderer.esc()` method was a no-op. The method existed, the test existed, the test was failing — but the failure was masked because the escape targets and replacements were identical strings. `replace("&", "&")` instead of `replace("&", "&amp;")`. An XSS vulnerability hiding in plain sight in a compliance reporting module. We fixed it as part of the renderer work.

The JPA JOINED inheritance index issue was more subtle. `CREATE INDEX ON message_ledger_entry(tenancy_id, tool_name)` fails because `tenancy_id` lives on the parent `ledger_entry` table — JOINED inheritance splits columns across tables, but JPQL hides this behind a unified entity model. You write `WHERE e.tenancyId = :tid` and forget that the column isn't physically where you think it is.

## What this opens up

When engine #998 lands and starts dispatching judgment EVENTs, the compliance reports light up with no additional qhorus work. The contract is defined. The columns exist. The queries are tested with synthetic data.

The per-caller fulfillment breakdown could feed back into trust scores — a caller with a 95% acceptance rate across 200 judgments has earned trust that the system should recognise. That's the governed yield vision from epic #410: judgment quality feeding reputation, reputation feeding routing, routing feeding judgment assignment. Each layer composes.
