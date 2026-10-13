---
layout: post
title: "The Viewer That Was Already There"
date: 2026-09-03
entry_type: note
subtype: diary
projects: [casehubio/blocks-ui]
tags: [diagram, case-flow, composition, architecture]
series: issue-150-case-flow-viewer
---

# The Viewer That Was Already There

The issue asked for a read-only case flow viewer — something any CaseHub app could drop in to show investigation flows with runtime state, without dragging in the full `casehub-diagram` editor and its 900 lines of palette, property panel, and YAML editing machinery.

The first instinct was to build a new component with its own data contract — a `CaseFlowResponse` type with flat nodes, edges, and parallel groups. A separate rendering path. Claude drafted that approach and I stopped it.

The rendering pipeline already exists. `toGraph()` turns CaseDefinition YAML into a `GraphModel`. `toDecorations()` maps `CaseRuntimeState` into badges. `computeElkLayout()` does the layout. `pages-graph-canvas` renders it. `registerCaseStencils()` registers the worker, binding, milestone, goal, and subcase stencils. `casehub-diagram` composes all of these — but so can anything else. That's the point of having them as separate, importable functions.

The viewer is 140 lines. It extends `DiagramBaseMixin` in readonly mode, overrides `_adaptYaml()` to call `toGraph()`, overrides `_decorations()` to call `toDecorations()`, and gets src fetch, ELK layout, error handling, and SVG/PNG export for free. No code was copied from `casehub-diagram`. No code was extracted. The viewer and the editor are independent compositions of the same shared units.

## The trust score boundary

The interesting design question was how to render trust scores on worker nodes. `NodeDecoration` — the runtime overlay type from `graph-core` — had `badge`, `border`, `overlay`, and `tooltip`. No mechanism for supplementary labels.

The quick path was to inject trust scores into `GraphNode.properties` during the adapter call — smuggling runtime data into the definition model. Claude proposed this as the recommended approach. I pushed back: `GraphNode.properties` carries definition data parsed from YAML. `NodeDecoration` carries runtime visual state. Conflating them is a workaround that breaks the boundary between what a case *is* and what's happening to it *right now*.

The proper fix was extending `NodeDecoration` upstream with a `pills` array — a generic mechanism for domain-agnostic supplementary labels. Trust scores, execution times, SLA deadlines, cost indicators — all use the same channel. One upstream change in `graph-core`, and every stencil renderer can display pills without any stencil-specific code. That became casehub-pages#404 — a small change with wide reuse.

## Parallel groups without model mutation

The original proposal inserted synthetic compound nodes into the `GraphModel` after `toGraph()` returned — mutating the graph to force ELK into side-by-side layout. It worked, but it coupled the viewer to internal graph structure assumptions and blurred whether parallel groups are a structural concept or a layout concern.

They're a layout concern. ELK supports partitioning natively — assign each parallel group's nodes a partition index, activate partitioning, and the layout engine places them side-by-side. No synthetic nodes, no model mutation. The viewer's `_layoutOptions()` override maps `runtimeState.parallelGroups` to partition indices. The graph model stays clean.

## What it opens up

Two protocols came out of this work. The first — diagram viewers compose shared pipeline pieces independently, never extract or duplicate from editors — codifies something that should have been written down earlier. The second — runtime visual state belongs in `NodeDecoration`, not `GraphNode.properties` — draws a boundary that will matter as more domain-specific overlays land.

The `pills` array on `NodeDecoration` is the most reusable piece. Any viewer, any stencil, any domain can now show supplementary labels on graph nodes through a single, typed channel. Trust scores were just the first consumer.
