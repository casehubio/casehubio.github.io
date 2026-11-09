---
layout: post
title: "From One Diagram to Two: Extracting DiagramBaseMixin"
date: 2026-08-06
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-blocks-ui]
tags: [casehub-blocks-ui, diagram-editor, lit-mixin, serverless-workflow, typescript]
---

## The refactoring that unlocked swf-diagram

When I designed `casehub-diagram`, everything lived in one 500-line file — undo/redo, render pipeline, persistence, keyboard shortcuts, dirty tracking, error states, the lot. That was fine for a single diagram component. The moment the SWF diagram spec called for a second canvas component with the same orchestration and different domain logic, the choice was duplicate or extract.

We extracted. `DiagramBaseMixin` moves into a new `packages/diagram-core/` package, and `casehub-diagram` shrinks from 500 lines to under 200. Five abstract methods define the subclass contract:

```typescript
protected abstract _adaptYaml(yaml: string): AdapterResult;
protected abstract _applyPropertyEdit(yaml, nodePath, field, value): string;
protected abstract _schemaTypeMap(): Record<string, string>;
protected abstract _paletteTypes(): string[];
protected abstract _emptyTemplate(): string | null;
```

The case diagram fills these in with `toGraph`, `applyPropertyEdit`, its schema map, four palette types, and the empty case YAML template. The SWF diagram fills them with `toSwfGraph`, `applySwfPropertyEdit`, the SWF schema map, an empty palette (no structural editing), and `null` (no document creation).

## TypeScript fought back

The mixin worked at runtime immediately. TypeScript's `composite` mode — required for project references in monorepos — refused to compile it. TS4094: "Property of exported anonymous class type may not be private or protected." Every `@state()` property, every abstract method, every inherited Lit lifecycle method triggered it.

The fix required three TypeScript features composed together: a `declare class` interface for the public surface, a function overload signature returning that interface type, and a cast at the return site. The anonymous class keeps its protected members internally; the exported type exposes only the public contract. It's the standard Lit mixin pattern, but the standard examples don't use `composite` mode, so the error only surfaces in real monorepo setups.

## swf-diagram: self-sufficient by default

`swf-diagram` extends `DiagramBaseMixin(LitElement)` and weighs about 100 lines. One design choice worth noting: the `schema` property defaults to `swfTaskSchema` — a static JSON Schema covering CallTask, SetTask, SwitchTask, RaiseTask, TryTask, and TryCatchTask. The case diagram defaults its schema to `{}` because the CaseDefinition schema is generated externally. SWF's schema is stable enough to ship as a static export, making the component self-sufficient — drop `<swf-diagram yaml="${yaml}">` into a page and property editing works without the host app passing anything.

## Thumbnails without cross-package imports

The interesting constraint for worker thumbnails was dependency direction. The worker stencil lives in `graph-stencil-case`. The SWF thumbnail renderer lives in `graph-stencil-swf`. If the worker stencil imported from the SWF package, every app loading case diagrams would pull in the entire SWF dependency tree — including `@openworkflowspec/sdk`.

A three-line registry solved it. `registerThumbnailRenderer` and `getThumbnailRenderer` live in `graph-stencil-case`. The worker stencil calls `getThumbnailRenderer('swf')` — if nothing is registered, the worker renders without a thumbnail. The hosting application wires them together at init:

```typescript
import { registerThumbnailRenderer } from '@casehubio/graph-stencil-case';
import { createSwfThumbnailRenderer } from '@casehubio/graph-stencil-swf';
registerThumbnailRenderer('swf', createSwfThumbnailRenderer());
```

Apps that don't need SWF support never call this. Workers render fine either way. We captured this as protocol PP-20260806-320d50 — stencil packages must not import from each other.

## What this opens up

The `DiagramBaseMixin` extraction means any new graph domain gets a diagram component for the cost of implementing five methods. Work stencils, IoT device graphs, or any future domain adapter can have a full editor — undo/redo, persistence, keyboard shortcuts, property panel — without duplicating the 400 lines of orchestration that every diagram needs.

The `nodeSizes` parameter that casehub-pages#290 added to `computeElkLayout` means inline expand actually reflows the case diagram around the larger worker node. Without it, the thumbnail grew but the surrounding graph didn't adjust. Small change in the layout engine, large difference in usability.
