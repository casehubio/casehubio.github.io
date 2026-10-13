---
layout: post
title: "Why hostPanel Props Were Never Resolved"
date: 2026-08-18
entry_type: note
subtype: diary
projects: [casehub-pages]
tags: [runtime, template-resolution, context-wiring, configurable-panel]
series: issue-322-hostpanel-template-resolve
---

The pages runtime has a template system — `#{params.caseId}`, `#{selection.cases.status}`, `#{filter.ward}` — that resolves context values into strings wherever they appear. Title components resolve them. HTML content resolves them. Markdown resolves them. Dataset URLs resolve them. Action bodies resolve them.

hostPanel props didn't.

A `hostPanel` call like `hostPanel("workbench", { executionId: "#{selection.cases.caseId}" })` passed the literal string `"#{selection.cases.caseId}"` through to `configure()`. The workbench connected to `/api/cases/#{selection.cases.caseId}/state` — the literal template, not a UUID.

The fix follows the same ContextManager consumer pattern that every other template-aware component uses. But hostPanel props are different from title or HTML content in two ways that shaped the design.

**Props are programmatic, not visual.** Title text with an unresolved template shows as empty — harmless. Panel props with an unresolved template trigger invalid network requests, broken WebSocket connections, error states that flash before real data arrives. So where title/html resolve immediately (empty string is a valid visual state), hostPanel defers `configure()` until all template vars are non-empty. Same `allTemplateVarsResolved()` gate the data pipeline uses for parameterised URLs.

**Props come in sets, not singles.** A title has one template string. A hostPanel might have three — endpoint, caseId, mode. The existing ContextConsumer pattern evaluates each template entry independently and fires its `apply` callback when the value changes. With three entries, that's three separate `configure()` calls per context change, each with a partially-updated props object. A panel tearing down and rebuilding a WebSocket connection on each call would churn through two invalid intermediate states before landing on the correct one.

We added a `postEvaluate(changed: boolean)` hook to `ContextConsumer`. It fires once after all template entries in a consumer have been evaluated. The `changed` flag tells the consumer whether anything actually changed — no spurious reconfiguration on unrelated context updates. The host panel's `postEvaluate` resolves all props in one pass and calls `configure()` once with the complete set.

The `#{row.xxx}` syntax from the original issue title doesn't work through this mechanism, and that's architecturally correct. `row` on `RuntimeContext` is a per-instance local context — `createRowContext(base, rowData)` produces a new context with row set, used inside `PagesDataTable` for per-row expression evaluation. The ContextManager is a shared singleton; putting row on it would corrupt every other consumer. Master-detail panels use `#{selection.datasetId.field}` instead — the selection namespace is global and reactive.

One subtlety the design review caught: when a host panel has both template props and a `lookup` (data pipeline binding), the data request must also be deferred. A panel that receives dataset rows before it's been configured can't process them — it doesn't know which endpoint to connect to or which columns to render. The `pages-data-request` event now fires alongside the first `configure()`, not at mount time.

The ConfigurablePanel contract's timing guarantee had to soften. The JSDoc used to say `configure()` is called before `connectedCallback()`. That's still true for panels without template props. With templates, `connectedCallback` fires first — the panel must have a valid default state from its constructor. The "Re-configuration" clause already required handling re-entry; the change is that the first call may come after connection rather than before.
