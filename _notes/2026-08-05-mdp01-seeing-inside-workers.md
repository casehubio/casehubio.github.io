---
layout: post
title: "Seeing Inside Workers: SWF Diagrams for CaseHub"
date: 2026-08-05
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-blocks-ui]
tags: [casehub-blocks-ui, serverless-workflow, diagram-editor, openworkflowspec, graph-stencils]
---

## The view that was missing

CaseHub's visual diagram editor renders case definitions as directed graphs — bindings, workers, milestones, goals, connected by capability dispatch and subcase edges. You can see the structure of a case at a glance: which workers handle which capabilities, how bindings route work, where milestones gate progress.

But workers with `do:` blocks — the ones that implement Serverless Workflow steps internally — showed up as opaque boxes. A worker labelled "ocr-worker" might contain an HTTP call, a data transform, a conditional branch, and an error handler. From the case diagram, all you saw was the label and a list of capabilities. The implementation was invisible.

The `@openworkflowspec/sdk` npm package changed that. It ships a `buildFlatGraph()` function that takes a parsed SWF Workflow and produces a typed node/edge graph — call nodes, switch nodes, set nodes, raise nodes, try/catch containers, start/end boundaries. The graph construction that would have taken weeks to build from the SWF spec is one function call.

## Three levels of detail

The design I landed on is progressive disclosure — three ways to see what's inside a worker, each giving more detail than the last.

**Thumbnails** render directly on the worker node in the case diagram. A miniaturised SVG showing the flow structure of the `do:` block — positioned rectangles and connecting lines, non-interactive, scaled to fit a 180×100px container. The same adapter and layout engine produce the data; the thumbnail just projects it as lightweight SVG instead of a full React Flow canvas. If a worker has a `do:` block, you see its shape without clicking anything.

**Inline expand** grows the worker node to show a larger, readable version of the same SVG. Still on the case diagram canvas, still non-interactive — but big enough to read step labels and trace the flow. ELK relays the surrounding graph to accommodate the expanded node.

**Full open** fires a `diagram:worker-drill-down` event. The hosting app decides what to do with it — open an `swf-diagram` component in a new panel, overlay, tab, whatever fits the application context. `swf-diagram` is a first-class canvas, the SWF equivalent of `casehub-diagram`. Same persistence backend, same property editing, same undo/redo — it just speaks a different domain language.

## The dual-walk integrity check

The adapter needs two things from the same YAML: a graph (nodes and edges for rendering) and a path map (YAML positions for property editing). The SDK provides the graph via `buildFlatGraph()`. The path map requires walking the YAML CST ourselves, because property edits need to target exact positions in the source YAML — CST-preserving mutations that keep formatting and comments intact.

Both walks must agree on node identity. The SDK generates path-based node IDs: `/do/0/fetchData`, `/do/1/transformResult`, `/do/0/tryBlock/try/0/innerStep`. The YAML walker replicates this path construction by descending the CST in the same order.

After both walks complete, an integrity assertion checks that every task node in the SDK graph has a corresponding YAML path. If they diverge — which would mean an SDK upgrade changed ID generation — the adapter returns a `degraded` flag instead of throwing. The graph still renders (layout and stencils don't need YAML paths), but property editing is disabled with a warning banner. The alternative — silently targeting wrong YAML locations on property edit — would corrupt the user's workflow definition.

```typescript
const taskSdkNodes = allSdkNodes.filter(n => !SYNTHETIC_TYPES.has(n.type));

if (yamlPaths.size !== taskSdkNodes.length) {
  degraded = { reason: `yamlPaths/model mismatch: ${yamlPaths.size} paths for ${taskSdkNodes.length} task nodes` };
}
```

The `SYNTHETIC_TYPES` filter is important. The SDK generates structural nodes — `start`, `end`, `entry`, `exit`, `root`, `try`, `catch` — that have no corresponding YAML task items. They're graph-construction artefacts, not editable content. Filtering them out before the integrity check prevents false mismatches.

## Stencil-per-step-type

Each SWF step type gets its own visual treatment. A `call` node shows a globe icon for HTTP, a plug for gRPC, brackets for function calls, an arrow for `casehub:dispatch`. A `switch` node shows its case count. A `raise` node has a red accent band and the error title. Boundary markers (`start`, `end`, `entry`, `exit`) render as rounded pills in green and grey.

Every SWF stencil type is prefixed with `swf-` — `swf-call`, `swf-switch`, `swf-set` — to avoid collisions in the shared stencil registry. Case stencils remain unprefixed as the founding domain. A `swf-generic` fallback handles any SDK node type that doesn't have dedicated artwork yet.

All type routing happens through a single `SWF_KNOWN_TYPES` constant that both the adapter and the registration function reference. Adding a new dedicated stencil means adding one entry to that set — a single source of truth that keeps mapping and registration in sync.

## What's next

The adapter and stencils are built. The larger piece ahead is extracting a `DiagramBaseMixin` from `casehub-diagram` — the orchestration logic (undo/redo, render pipeline, persistence, keyboard shortcuts) that both the case diagram and the new SWF diagram need. It's the highest-risk task because it refactors working code, but the existing test suite is comprehensive enough to validate the extraction. Once the mixin exists, `swf-diagram` is a thin subclass that plugs in the SWF adapter and stencils.

The deeper question this work surfaces: how far does the stencil-per-domain pattern extend? CaseHub already has case definitions and now SWF workflows. The engine also has planned actions, commitment lifecycles, and trust policies — each with its own YAML schema and visual structure. The adapter/stencil/diagram layering we're building for SWF is the template for all of them. Getting the mixin extraction right matters beyond this one component.
