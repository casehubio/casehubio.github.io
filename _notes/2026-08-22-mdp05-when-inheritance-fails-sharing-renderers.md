---
layout: post
title: "When Inheritance Fails: Sharing Renderers Without Sharing Lifecycles"
date: 2026-08-22
entry_type: note
subtype: diary
projects: [casehub-blocks-ui]
tags: [composition, web-components, pages-viz, timeline, architecture]
series: issue-123-timeline-extends-pages
---

The plan was straightforward: `BlocksTimeline` should extend `PagesEventTimeline` from pages-viz. Both components render the same three timeline layouts (vertical, horizontal, compact). Both consume the same node type. The rendering code is nearly identical. Classic inheritance case.

Except it isn't.

`PagesEventTimeline` extends `PagesElement`, which gates `renderContent()` behind a triple condition: `!!this.props && !this.controller.loading && !!this.controller.dataSet`. That gate exists for good reason — it's the correct lifecycle for dashboard-embedded components where a host pushes data through `DataSourceController`. But `BlocksTimeline` consumers set `endpoint` and let `DataSourceMixin` fetch. They never touch `props`. They never push `dataSet`. The component would show a loading skeleton forever.

I initially missed this. The `data` property on `PagesEventTimeline` looked like a bypass — set data directly, skip the controller. Claude's decision review caught it: `data` is only consumed inside `renderContent()`, which never fires without the gate conditions being met. The "bypass" doesn't bypass the render dispatch. To make inheritance work, we'd need to override `render()` entirely — about 125 lines of bridge code to work around a base class whose lifecycle assumptions don't match.

The right answer turned out to be the one I'd initially dismissed: composition. Extract the three renderers as pure functions in pages-viz — nodes in, HTML out, no lifecycle, no state. Both `PagesEventTimeline` and `BlocksTimeline` import and call them with their own callbacks. Each component keeps its own data lifecycle. The rendering code lives in one place.

The refactoring also forced a layering cleanup. The current renderers have CSS classes like `.timeline-node.CASE .node-dot` and `.event-dot.WORKER` — CaseHub domain categories baked into what's supposed to be a generic rendering layer. Moving renderers to pages-viz made this violation visible: pages shouldn't know about `CASE`, `WORKER`, or `TIMER`. The shared renderers now use status-based styling only (`status-completed`, `status-active`, etc.). Category-specific visual differentiation moves to strategy `renderNode` callbacks with inline styles, which is where protocol PP-20260713-8ea1af says it should have been all along.

We built the pages-viz side this session — four shared render functions (vertical, horizontal, compact, filter bar) and refactored `PagesEventTimeline` to use them. PagesEventTimeline gained horizontal and compact layout support in the process, which it didn't have before. The blocks-ui side (dropping local renderers, adopting pages types, updating strategies) is next session's work. The examples need attention too — pages should show all three layout variations with the shared renderers, and blocks-ui should demonstrate the composition and layering: domain strategies providing category-specific rendering via callbacks on top of the status-based generic layer.

The broader lesson: when two components share rendering but have different data lifecycles, the rendering layer is what gets shared — not the class hierarchy. The lifecycle IS the value each component provides to its context.
