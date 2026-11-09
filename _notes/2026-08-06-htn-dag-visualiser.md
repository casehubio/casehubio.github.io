---
title: "Visualising what the engine already knows"
date: 2026-08-06
entry_type: note
subtype: diary
projects: [casehub-blocks-ui]
tags: [htn, dag, graph, visualisation, design-review]
status: draft
---

The engine has had a rich HTN planning model for months — `TaskNode` sealed hierarchies, `DagPlan` with dependency edges and join semantics, seven decomposition strategies from static guard-matching to LLM-driven planning to GOAP. Zero UI representation. The data was there, flowing through the runtime, completely invisible.

Today that changed. Five new packages give the platform two distinct views of the same planning model: a recursive tree that shows *how* a compound task decomposes (which strategy, which guard matched, which method was selected), and a DAG graph that shows *what's executing* (dependency edges, node state, dispatch mode).

## The separation that matters

The obvious approach would have been to add a "DAG mode" to the existing case diagram editor. Same component, third mode alongside design and runtime. Wrong answer. casehub-diagram is a YAML editor — it has a YAML adapter, YAML palette, YAML property panel, YAML undo/redo. DagPlan is a completely different data model. Cramming both in creates a god component with two unrelated responsibilities.

The right answer follows the pattern already established by `graph-stencil-case` and `graph-stencil-swf`: parallel domain adapters over the same graph infrastructure. `graph-stencil-htn` converts DagPlan snapshots to GraphModel the same way `graph-stencil-case` converts CaseDefinition YAML. Same pipeline, different input. The components stay focused.

## The status registry pays off

When #109 landed the status registry with 10 built-in domains, it included a `node:` domain with three purpose-specific descriptors — `DISPATCHED` (info, →), `SKIPPED` (neutral, ⏭), `FAILED` (danger, ✗). The design review caught that the original spec routed NodeState through the `task:` domain via an intermediate TaskStatus mapping. Wrong. `Dispatched` means "sent to an agent" — that's `info` with an arrow, not `success` with a play button. The `node:` domain exists precisely for this semantic distinction.

All three review dimensions caught this independently. Convergent findings from independent reviewers are the strongest signal that a design flaw is real.

## The ID space problem

Tree nodes use `LeafTaskSnapshot.id`. DAG nodes use `DagNodeSnapshot.id`. These are different ID spaces — one comes from the task payload, the other from the graph node. If both components emit their native IDs in selection events, clicking a leaf in the tree highlights nothing in the DAG.

The fix: both components emit `taskId` — the task's identity in the engine. `DagNodeSnapshot.taskId` and `LeafTaskSnapshot.id` are the same value. The adapter returns a `taskIdToGraphNodeId` map so the DAG viewer can resolve incoming selection events to graph coordinates. Simple, but every reviewer caught it and every reviewer was right.

## What shipped

- **graph-stencil-htn** — types mirroring the engine's sealed interfaces, DAG adapter with topology detection, dag-node stencil with join indicators (∧ ALL_OF / ∨ ANY_OF), runtime decoration via the `node:` domain
- **blocks-dag-viewer** — read-only graph viewer with toolbar (dispatch mode, staleness timer, summary stats), decoration-only update path that skips ELK layout
- **blocks-decomposition-tree** — ARIA tree with eight strategy badge colours, guard label display, method selection highlighting, render callbacks for extensibility
- **blocks-plan-item-tree** — PlanItemDefinition tree with CompletionSemantics badges and DispatchMode pills
- **blocks-plan-model-dashboard** — card-based CasePlanModel overview: agenda, focus, budget, sub-cases, compound progress bars

Engine REST endpoints to actually serve the data are next (engine#873, slot 90). The UI defines the contract — the engine conforms to it.
