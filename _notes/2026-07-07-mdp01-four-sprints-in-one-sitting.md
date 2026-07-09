---
title: Four Sprints in One Sitting
date: 2026-07-07
author: mdp
tags: [data-table, web-components, typescript, refactoring]
entry_type: note
subtype: diary
type: phase-update
projects: [casehub-blocks-ui]
series: issue-30-tree-expandable-rows
---

*Part of a series on the data-table build. Previous: [The Platform Table](2026-07-06-mdp02-the-platform-table.md).*

Yesterday we built the data-table from scratch. Today we closed four issues against it in a single session — the kind of velocity that only happens when the underlying architecture is clean enough to extend without fighting.

## Type debt from the prototype sprint

The first issue was cleanup: six `as any` casts scattered across three production files. They accumulated during the consolidation sprint, where we were moving fast and the API response shapes weren't nailed down yet.

The interesting one was in `work-item-detail.ts`. The `WorkItemRelation` interface had `readonly` fields, but `_fetchRelatedItemDetails` needed to enrich relations with titles and statuses after fetching them from the API. The original code cast through `any` to mutate the readonly fields in place — a pattern that works at runtime but defeats the purpose of the type system.

The fix was to return new objects instead of mutating: `_fetchRelatedItemDetails` now returns `WorkItemRelation[]` with the enriched fields spread in. No mutation, no cast, and the `readonly` constraint actually means something.

## Three features the architecture absorbed

**Multi-column sort** was the one I expected to be fiddly. The existing single-column sort used two properties — `sortColumnId` and `sortDirection`. Multi-column needs a stack. I kept backward compatibility by turning those properties into getters backed by a `_sortStack: SortEntry[]` array. Normal clicks replace the stack; Shift+click appends. The sort indicator now shows priority numbers (1, 2, 3) when multiple columns are active. The `createMultiComparator` chains column comparators — first non-zero result wins.

**CSV export** was a standalone module: `tableToCsv` iterates rows × visible columns, `downloadCsv` creates a blob URL, `copyToClipboard` writes to the clipboard API. Proper CSV escaping for commas, quotes, and newlines. No coupling to the component — it's a pure function that takes rows and column definitions.

**Tree/expandable rows** needed more thought. The `flattenTree` function takes root rows, a `getChildren` accessor, and an expanded-ID set, then produces a flat array with depth metadata. The component renders expand/collapse toggles on the first visible column with CSS indentation at 20px per depth level. The tricky part was threading tree metadata through the existing `_visibleRows` pipeline without breaking the return type — I used a parallel `Map<unknown, TreeRow>` that the cell renderer consults.

## What made this possible

The data-table's `ColumnDef<R>` design carries most of the weight. Sort needs `getValue` and `compare`. CSV export needs `getValue` and `label`. Tree needs the component to accept a `getChildren` prop — the column definitions don't change at all. Each feature plugged into the existing data model without modifying it.

The virtual scroll engine being a pure function helped too. Tree rows and multi-column sort both change the row set before it reaches the scroll window calculation — no special cases needed in the renderer.
