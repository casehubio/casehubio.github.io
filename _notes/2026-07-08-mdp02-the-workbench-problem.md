---
title: "The Workbench Problem"
date: 2026-07-08
entry_type: note
subtype: diary
tags: [blocks-ui, composition, web-components, aml]
---

# The Workbench Problem

AML built a case workbench. blocks-ui already had a work-item workbench. Same layout — split-pane, draggable divider, responsive collapse, keyboard shortcuts, selection coordination. Different children.

The overlap happened because `<work-item-workbench>` hardcodes its children. It renders `<work-item-inbox>` on the left and `<work-item-detail>` on the right. There's no way to swap them out. AML needed a simpler list (just a data-table with a filter) and a pluggable tab panel (domains register their own panels). So they built their own.

The fix was obvious once you see it: slots.

## Slots fix everything

A workbench is a layout shell. The children are the domain's concern. `<split-workbench>` accepts `slot="list"` and `slot="detail"` and knows nothing about what goes in them. Selection coordination flows through `pages-event` on `document` — the workbench listens for `{topic}:selected` / `{topic}:deselected` purely for responsive panel switching on mobile. The children talk to each other through events, not through the parent.

AML's usage reads exactly as you'd expect:

```html
<split-workbench selection-topic="case" title="AML Investigations">
  <list-pane slot="list" selection-topic="case" endpoint="/api/investigations"
    .columns=${investigationColumns}>
  </list-pane>
  <detail-pane slot="detail" selection-topic="case"
    .tabs=${investigationTabs}>
  </detail-pane>
</split-workbench>
```

Three properties on the workbench. Everything else is the children's business.

## The tab registry question

AML had a module-level singleton for registering tabs: `registerCaseTab({ id, label, tagName, order })`. Imperative, global, import-order-dependent. Two workbenches on one page would share tabs.

We replaced it with a property array. The page definition — which knows the domain — passes the tab list directly:

```typescript
.tabs=${[
  { id: 'overview', label: 'Overview', tagName: 'aml-investigation-overview' },
  { id: 'findings', label: 'Findings', tagName: 'aml-findings-panel' },
]}
```

The detail pane doesn't pull from global state. It renders what it's given. Declarative, testable, independently configurable.

## The item contract

Each tab panel needs the selected item's data. We went back and forth on this — fixed property names (`itemId` + `itemData`) vs configurable mapping vs `configure()` calls. The answer was simpler than any of those: one generic property called `item`.

The detail pane receives the selection event payload and sets `.item` on the active tab element. It never inspects or maps the payload. Each domain's tab panel destructures what it needs:

```typescript
@property({ attribute: false }) item: any;
get caseId(): string { return this.item?.caseId ?? ''; }
```

The detail pane is a dumb pipe. The tab panels know their domain. Nobody in the middle needs to understand the shape.

## What the review caught

We ran the spec through an adversarial design review — six rounds. The first round surfaced twelve issues. Event topic naming was the most consequential: blocks-ui used dots (`work-item.selected`) but the pages-data `matchesTopic()` protocol splits on colons. AML already used colons. We migrated all event topics to colons as a prerequisite.

The review also caught underspecified responsive behaviour — the spec said "single-panel mode on mobile" without specifying the mechanism. We added `[has-selection]` host attribute driven by selection events, CSS container queries for the breakpoint, and a back button rendered in the workbench's shadow DOM (not injected into slotted content).

## The pipeline question

While inspecting the pages codebase for the migration, I reviewed the new `DataSource` abstraction — `restSource()`, `sseSource()`, `simulated()`, the whole pipeline. The architecture is sound but several gaps make blocks-ui component authoring harder than necessary. `DataReceiver` is missing `loading` state, the SSE paths haven't converged, and `restSource()` requires full `ResolverContext` infrastructure that standalone components can't easily provide.

We filed casehub-pages#145 with seven improvements prioritised. The components we built use `DataEndpointMixin` today — when the unified `DataSourceMixin` ships, it's a one-line mixin swap per component. No public API change.

## End state

blocks-ui gained three generic components. `work-item-workbench` lost 400 lines and became a thin wrapper. AML deleted 535 lines of local workbench code and consumes the generic versions. OpenClaw, Clinical, and DevTown can adopt the same components for their workbenches without building their own.
