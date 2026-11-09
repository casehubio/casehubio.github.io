---
layout: post
title: "When the Dock Workbench Gives You Three Equal Columns"
date: 2026-08-08
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [life-ui, layout, pages-ui, testing, css-grid]
series: issue-81-life-ui-design
---

The Life UI dashboard uses a dock workbench layout — side panels on left and right, centre content fills the rest. IntelliJ-style. The pages-ui library ships a `dockWorkbench()` DSL function that builds exactly this, and we'd been using it since the first scaffold.

It looked terrible on a wide screen. Three equal columns, each getting roughly a third of the viewport. The inbox panel — which should be a narrow 280px dock — was 640 pixels wide, mostly white space. Same on the right with the family summary.

I dug into the pages-ui source to understand why. The `dockWorkbench()` function internally calls `split("horizontal", [leftZone, centre, rightZone])` with only `minSizes` passed — no initial size ratios. The `split()` component defaults to equal distribution. The `DockWorkbenchConfig` TypeScript interface exposes `minSize` per panel but nothing else — no `defaultSize`, no `initialRatio`. The equal-split behaviour is invisible from the API surface.

The spec had anticipated this. Section 3.1 documented a CSS grid fallback for exactly this case. We replaced the dock workbench with a simple grid: `grid-template-columns: auto 1fr auto`. Dock bars are 32px vertical tab strips; open panels are 280px fixed width; the centre fills remaining space. The toggle behaviour — click a dock tab to show/hide its panel — is just Lit `@state()` and conditional rendering. Simpler than the abstraction it replaced.

The rest of the session was wiring depth into the views that were scaffolded last time. The cases view gained tabs (Overview, Tasks, Audit), a domain filter dropdown, and SSE live refresh. The Tasks tab fetches pending actions filtered by the case's domain — not a direct case-to-task relationship, but enough to show relevant work items in the demo. The people view went from three tabs to five: the activity tab now fetches from `/external-actors/{id}/activity`, the tasks tab from `/external-actors/{id}/tasks`, and there's a GDPR erasure tab with the "erase personal data" button wired to the real `DELETE` endpoint.

We also hardened the demo data — seven cases now instead of five, including completed and failed cases with historical timestamps, plus five completed work items so the activity timelines have content to show.

The session's other surprise was the test suite. Four tests were intermittently failing with `OptimisticLockException` — but the root cause wasn't locking. It was cross-class data leakage in H2. Test class A creates a work item with a past deadline and doesn't clean it up. Test class B calls `checkExpired()`, which finds all expired items including the leaked one. The Quartz scheduler also fires for expired items, so when the test manually calls `expireItem()`, both code paths update the same entity version — optimistic lock. The symptom pointed at locking; the fix was `@BeforeEach` cleanup and letting the Quartz scheduler handle expiry via Awaitility instead of racing with it.
