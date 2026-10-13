---
title: "Selection: from imperative DOM wiring to parameterised datasets"
date: 2026-09-02
author: mdp
tags: [frontend, casehub-pages, selection, parameterised-datasets]
issues: [122, 148]
---

# Selection: from imperative DOM wiring to parameterised datasets

The clinical workbenches had layout scaffolded but no interactive behaviour — clicking a row in the AE or deviation table did nothing. Two issues landed back-to-back today that took the selection story from zero to native in a single session.

## The imperative detour (#122)

I started by enabling `selection: "single"` on both `dataTable()` DSL components and adding a `selection-change` listener in `index.ts`. When a row was selected, the handler extracted cell values, discriminated AE vs deviation by checking for field presence (`grade` → AE, `deviationType` → deviation), and imperatively set `endpoint` properties on every detail component via `document.getElementById`.

It worked. It was also 120 lines of central wiring code with `as any` casts on every DOM lookup, XSS risk from `innerHTML` on the overview tabs (caught in code review — added an `esc()` helper), and a tight coupling between `index.ts` and every component's property API. Each new tab would mean another block in the handler.

## The native mechanism was already there (#148)

Before starting #148, I checked whether the pages selection context APIs — filed as casehubio/casehub-pages#289 and its three children — had actually shipped. They had. All three were closed. The infrastructure I needed was sitting in the runtime:

- `RuntimeContext.selection` — a map of dataset ID to selected row data, updated automatically on every `selection-change` event
- `#{selection.adverse-events.id}` — template syntax in dataset URLs, resolved by the context manager
- Parameterised URL lifecycle — defers fetch until the template resolves, aborts in-flight requests on selection change, re-fetches automatically

The template approach means declaring a dataset like this:

```typescript
restDataset("ae-precedents",
  `/api/trials/${trialId}/adverse-events/#{selection.adverse-events.id}/precedents`)
```

The runtime handles everything — no event listeners, no DOM queries, no imperative property setting. The DSL `dataTable()` with `lookup("ae-precedents")` auto-populates when a row is selected.

For components that manage their own fetching (`commitment-lifecycle`, `ae-grade-history`, `ae-regrade`), I created a `selection-bridge.ts` helper — 23 lines that listens for `selection-change` on `document`, discriminates by dataset, and calls back with the entity ID. Each component registers in its `connectedCallback()` and cleans up in `disconnectedCallback()`. Context flows via `data-trial-id` and `data-source-dataset` attributes set in the workbench DSL.

The net result: deleted 120 lines of imperative handler, the `esc()` XSS helper, and the `ClinicalAuditTrail` component (superseded by a parameterised dataset + DSL table). Added 76 lines across the selection bridge and component listeners. The precedents and audit trail tabs are now pure DSL — no custom components needed.

## What this opens up

The `#{selection.X.Y}` mechanism works for any depth of master-detail nesting. A future DSMB detail view scoped to a selected trial site would be one more `restDataset()` declaration. The pattern also suggests that `<approval-gate>` should listen for selection natively — right now, the gate's `endpoint` and `gate-id` aren't wired to selection at all. That's the next gap.
