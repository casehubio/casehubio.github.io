---
layout: post
title: "Composition Meets Reality"
date: 2026-08-15
entry_type: note
subtype: diary
projects: [scaffold]
tags: [casehub-pages, orchestration, api-design, data-binding]
series: issue-41-platform-console
---

Continued from [The First Tab That Showed Nothing](2026-08-14-mdp01-first-tab-showed-nothing.md).

The console was feature-complete — ten tabs, all wired, build passing, first-tab bug fixed. Time to seed demo data and verify everything end to end before closing the branch.

Cases rendered perfectly. Six cases in a sortable table with UUIDs, names, statuses, dates. Orchestration showed the same six cases with lifecycle phase badges. So far, good. Then we started clicking.

## The Empty Column

The Orchestration table had a blank first column. Every row showed name, status, namespace, date — but the leading cell was empty. No console error. The column header said "name" but the `caseId` column was just missing.

The cases API returns `caseId`. The orchestration view referenced `col("id")`. Same dataset, wrong field name. The data table silently skipped the unrecognised column and rendered everything else.

That missing column was also breaking the detail panel. The orchestration workbench receives its case reference through `"case-id": "#{row.id}"` — which resolved to nothing because `row.id` didn't exist. The workbench sat there showing "No execution data" regardless of which row was clicked.

Two-character fix: `id` → `caseId` in two places. But the interesting part is what we found when we kept pulling the thread.

## The Workbench That Spoke a Different Language

Even after the field name fix, the orchestration detail panel still showed nothing. Claude traced through the workbench component source and found a deeper mismatch: the view passes `case-id` as a property, but the component's `configure()` method only recognises `executionId`. The property was silently ignored.

But the real gap isn't the property name. The execution monitor builds its SSE URL as `${endpoint}/${executionId}/state`. The engine's actual SSE endpoint is at `/api/v1/cases/{caseId}/stream`. Different path, different data shape entirely. The monitor expects an `ExecutionSnapshot` — agents, routing strategies, execution patterns. The engine emits `CaseStreamEvent` — plan-item transitions and context updates.

My first assumption was that we'd need an adapter layer to bridge these formats. Then we looked in IntelliJ.

## The Data Was Already There

The engine has an `ExecutionSnapshotStore` with full decomposition, DAG plan, and DAG result snapshots. `PlanResource` already serves them at `/api/v1/cases/{caseId}/plan/` with endpoints for the model, the DAG, the decomposition tree, and the execution result. The data the workbench needs — active agents, completed agents, execution state, elapsed time — is all derivable from what `PlanResource` already computes.

No adapter needed. One new `@GET` method in `PlanResource` that composes the existing snapshots into the `ExecutionSnapshot` JSON shape the component expects. The wiring fix in `orchestration.ts` is the other half. The component stays untouched.

## What Else Broke

The full tab-by-tab walkthrough surfaced four more gaps:

The **Queues** tab hits `/queues/health` and `/queues/summary` — neither exists. The backend has per-queue operations (`/{id}/summary`, `/{id}/trend`) but no global aggregates. The data table also expects `pendingCount`, `activeCount`, `completedCount` columns that the list endpoint doesn't return.

**Sessions** — 404. No endpoint at all.

**System > Definitions** — the API returns two case definitions but the sub-tab shows "No items found." Probably the same first-tab data binding issue from the prior entry, but applied to the System tab's internal sub-navigation.

**Work Items** showed zero items, but that's correct — the seeded items have no assignee or candidate groups, so the inbox filters them out. Not a bug, just an empty-queue consequence of the seed script.

## The Composition Gap

The console was built as pure composition — zero new components, just pages DSL wiring blocks-ui workbenches to REST endpoints. That's the right architecture. But composition only works when the component contracts match the API contracts. Several tabs were wired to endpoints or response shapes that don't exist yet. The code compiled, the build passed, the tabs rendered — they just showed empty state or 404s that looked like missing data rather than missing wiring.

This is the kind of thing that only surfaces when you actually run the app with data and click through every tab. The type system can't catch a `col("id")` that should be `col("caseId")` when both are valid strings.
