---
title: The Platform Table
date: 2026-07-06
author: mdp
entry_type: note
subtype: diary
projects: [casehub-blocks-ui]
tags: [data-table, web-components, css-grid, virtual-scroll, design-review]
issue: 22
---

Every CaseHub app needs tables. Until today, every CaseHub app that needed a table would have had to build its own — or use the one baked into pages-viz, which is coupled to the pages-runtime data pipeline and has no virtual scrolling.

The work-item-inbox had its own hand-rolled virtual scroller: fixed 72px rows, a translateY approach, a buffer of 10, and an activation threshold at 50 items. It worked but it was hardcoded — no pagination, no column configuration, no reuse path.

## What we built

`<pages-data-table>` is a standalone Lit Web Component with no dependency on pages-runtime. It receives data via properties and reports interactions via events — the same pattern as filter-chips.

The data model is `ColumnDef<R>` — a generic row type with column definitions that carry value extractors and optional render functions. The table doesn't know about TypedDataSet or WorkItemResponse. It calls `getValue(row)` to extract a value and `render(value, row)` to display it. When pages-runtime wants to use this table, it constructs ColumnDefs from its YAML config — ~5 lines of bridge code, not a compatibility layer.

Three display modes share the same rendering engine: auto (static for small datasets, virtual scroll for large), paginated (client-side slicing or server-side with totalRows), and scroll (virtual rendering with load-more for infinite scroll). The virtual scroll engine is a pure function — no DOM dependency, trivially testable.

CSS Grid per row instead of `<table>` elements. This enables virtual scroll via absolute positioning (impossible with `<tr>`), and keeps the door open for row and column spanning later.

Row styling uses CSS `::part()` — the table exposes each row as `part="row priority-urgent"`, and consumers style from outside the shadow boundary. This replaced an earlier `getRowClass` design that the adversarial review caught — CSS classes can't cross shadow DOM boundaries.

## The design review

The spec went through a 5-round adversarial review before implementation. Two independent Claude sessions — one reviewer, one implementor — iterated until convergence. 19 issues surfaced:

- RovingTabindexMixin is 1D only — the table needs 2D grid navigation (reviewer caught that neither blocks-ui-core nor pages-primitives supports row × column coordinate mapping)
- Shadow DOM blocks getRowClass CSS classes (changed to `::part()`)
- Auto mode had no performance guard (now auto-activates virtual scroll above 50 rows)
- Select-all semantics were unspecified per display mode (now explicit: page-only for server-side pagination, full-array for scroll)
- getRowKey upgraded from console.warn to hard Error (index-based identity causes data corruption with virtual scroll + sort)

All 19 issues were resolved. 17 verified by the reviewer, 2 accepted (three-state sort cycle as intentional improvement, pages- element prefix for promotion path).

## The implementation

10 tasks via subagent-driven development. The pure functions (scroll engine, sort comparators) went first. Then the component built up incrementally: core rendering → pagination → scroll → selection → sorting/visibility → keyboard/ARIA. The inbox refactored last, dropping ~170 lines.

The whole-branch review found 4 more critical issues: select-all only selected visible rows in scroll mode, ARIA sort used invalid values, tabindex roving broke after scrolling, and the inbox claim shortcut targeted removed elements. All fixed.

Then the browser caught what tests couldn't: the host element missing `height: 100%` (broke scroll and pagination), no sort direction arrows, and the column picker misaligning the header grid. JSDOM has no layout engine — `clientHeight` is always 0. A gotcha worth remembering.

## What this enables

Every CaseHub app that needs a table now has one. The inbox already uses it. When the component is stable, it promotes to pages-primitives and replaces the old PagesTable in pages-viz — tracked as #27. The pages YAML examples migrate at that point, not before.

Six deferred items are filed: spanning (#26), pages migration (#27), multi-column sort (#28), text filter (#29), tree/expandable rows (#30), CSV export (#31).
