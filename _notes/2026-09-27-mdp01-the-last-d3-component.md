---
layout: post
title: "The Last d3 Component"
date: 2026-09-27
entry_type: note
subtype: diary
projects: [casehubio/blocks-ui]
tags: [graph, migration, d3, pages, elk, cleanup]
---

# The Last d3 Component

The blocks-ui graph stack has been migrating toward pages infrastructure for months — each diagram component trading its bespoke rendering for `DiagramBaseMixin`, `pages-graph-canvas`, and ELK layout. After today, every graph component in the repo uses the same stack.

The session started with a queue of thirteen cleanup issues left over from the LSP schema work — dead code, debug scaffolding, a timer leak in the IntelliJ plugin, broken TypeScript exports. Mechanical fixes, most of them. The interesting one was a bug in `moveSwfTask`: when you drag a node to a new position in an SWF workflow, only the specific edge you dropped onto got its `then:` reference updated. Other tasks pointing to the moved node via `then:` were left dangling, producing a broken flow graph. The fix needed two rewiring passes — one at the removal site (redirect everything that pointed to the moved task), one at the insertion site (redirect the edge source) — plus updating the moved task's own `then:` to its new successor. Three operations where the original code only did one.

With the queue drained, I ran a full codebase audit. The question behind it: have we actually finished migrating the graph code, or are there stragglers? The audit confirmed that all four stencil packages — SWF, Case, HTN, Org — contain exactly what they should: adapters, YAML editors, stencil definitions, schemas, and edit policies. No generic infrastructure hiding in domain packages.

Except `case-dependency-graph`. A 900-line component with its own d3-force physics engine, d3-selection SVG renderer, d3-zoom pan handler, and d3-drag node interaction. Built before the pages graph stack existed, never migrated.

The migration turned out simpler than expected. The key discovery: pages already had `computeElkLayout(model, { algorithm: 'force' })` wired — ELK supports force-directed layout natively. No new pages work needed. We replaced the d3 simulation with a single ELK call, swapped the SVG rendering for `pages-graph-canvas`, and dropped four runtime dependencies and four type packages. The component went from 302 lines to 170, and the entire d3 subsystem — `force-layout.ts`, `graph-renderer.ts`, `SimNode`/`SimLink` types — was deleted. 502 lines removed net.

The trade-off: the live physics simulation is gone. d3-force gave you drag-to-reposition with spring-back — drag a node and the whole graph rebalances around it. ELK force layout is a one-shot computation. ReactFlow still lets you drag nodes, but without the physics. For a dependency viewer that's mostly read-only, that's an acceptable trade. If interactive force layout matters later, the d3 code is in git history.

The org stencil package still has one infrastructure straggler: `radial-layout.ts`, a 115-line standalone hub-and-spoke algorithm that doesn't use any org-specific types. That should migrate to `graph-renderer` as a peer of the existing layout algorithms — filed as casehub-pages#473.

Every graph component in blocks-ui now runs on the same rendering and layout stack. The stencil packages are pure domain config. That was the goal.
