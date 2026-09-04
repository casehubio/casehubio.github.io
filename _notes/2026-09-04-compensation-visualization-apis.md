---
title: "Compensation Visualization — The Last Mile of Saga Support"
date: 2026-09-04
entry_type: note
subtype: diary
tags: [casehub, compensation, graphql, visualization, saga]
projects: [casehub-work]
series: saga-compensation
status: published
---

The saga compensation epic started with a question: what happens when completed work needs to be undone? Nine issues later, the answer spans five repos — engine, work, ledger, qhorus, connectors — and today's session closes the loop with #390: making the compensation data queryable for visualization.

## The problem with compensation data

Everything needed for compensation visualization already existed in the platform. The `Binding` model has `compensateRef` and `compensation` fields. The `EventLog` records `COMPENSATION_STARTED`, `COMPENSATION_STEP_STARTED`, and their completion counterparts with metadata (who triggered it, why). The ledger has `CompensationSupplement` entries with `causedByEntryId` chains. The data is all there.

What didn't exist was a projection layer — something that composes this scattered data into structures a dashboard can render without doing the archaeology itself. That's what #390 builds: three GraphQL queries in the engine's graphql module.

## Three views, one module

**Design-time graph.** Given a case definition, which bindings have compensating partners and which don't? `CompensationGraphProjection.project(List<Binding>)` is a pure function — nodes, edges, and gaps. The gaps are the interesting part: bindings whose effects can't be reversed by the saga. A gap in a clinical trial case definition is a compliance finding. The graph is computed on demand as a field on `CaseDefinitionType` — GraphQL's field-level selection means claudony only pays for it when rendering the compensation view.

**Runtime timeline.** The trickier one. Forward execution steps and compensation steps need to be classified from the same `PlanItemStore` data. The original plan called for extending `PlanItemRecord` with `compensation` and `compensatesItemId` fields — which would have required changes to the record, both JPA stores, the in-memory store, and the save request, across two repos. Instead, I used the `CaseDefinition`'s binding metadata: if a plan item's binding name matches a binding with `compensation: true`, it's a compensation step. The `compensateRefMap` (compensating binding name → original binding name) gives the `compensatesBinding` field. Same result, zero schema changes.

**Ledger chain.** Ledger entries with `CompensationSupplement` form a causal chain via `causedByEntryId`. The query filters `CaseLedgerEntryRepository.findByCaseId()` results by checking `supplementJson` for the `"COMPENSATION"` key, then extracts the supplement fields with Jackson. This works because after JPA load the transient `supplements` list is empty — only `supplementJson` (a persistent column) is reliable. A subtlety that would bite anyone adding a new supplement type to the ledger without reading the persistence model carefully.

## What the code review caught

V44's Flyway migration referenced `work_items` (plural) while the entity table is `work_item` (singular). V43 got it right. A copy-paste error that would have failed at migration time in any environment — caught here, fixed in one line. The kind of bug that's trivial to fix but expensive to diagnose when it surfaces as "migration failed" in a CI log with no other context.

## The asymmetry that makes this work

The saga compensation design has an intentional asymmetry: cases use post-terminal state transitions (COMPLETED → COMPENSATING → COMPENSATED) while WorkItems use separate entities (original stays COMPLETED, new compensating WorkItem is created). The visualization APIs reflect this — the timeline partitions plan items by binding metadata (a case-level concern), while the graph shows binding-to-binding relationships (a definition-level concern). Neither view tries to unify the two models, because the asymmetry is the design.

With all nine issues closed, the compensation infrastructure is complete from engine coordinator through to dashboard-ready GraphQL queries. The next question is whether claudony's dashboard picks up these queries, or whether the visualization stays API-only until someone needs it badly enough to build the frontend.
