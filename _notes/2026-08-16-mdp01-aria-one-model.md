---
layout: post
title: "ARIA as the Only Interaction Model"
date: 2026-08-16
entry_type: note
subtype: diary
projects: [casehub-pages]
tags: [aria, accessibility, automation, mcp, design]
---

Every UI platform eventually faces the same question: how do automated tools find components? The industry's default answer is a parallel attribute system — `data-testid`, `data-automation`, `data-cy` — bolted on after the fact, maintained separately from the real interface, and invisible to users who actually need the UI to be navigable.

I've been working on CaseHub's pages platform for long enough to know where this leads. You end up with two models of your UI: the one screen readers use, and the one your test harness uses. They drift. The test attribute says the button is "submit-btn". The screen reader sees "Submit". A refactor renames the component and updates one model but not the other. The test passes. The accessibility breaks. Nobody notices because the people who would notice aren't running your CI pipeline.

The thesis behind the work we just landed on pages is simple: these are the same problem, and they should have the same solution. ARIA attributes — `role`, `aria-label`, `aria-expanded`, `aria-disabled` — are already the canonical way to describe what a UI component *is* and what state it's *in*. A screen reader navigates by role and name. An automated scenario should too. If the automation can't find a component, a screen reader can't either. One root cause, one fix.

## The contract

We defined four types in `pages-primitives` — `AriaRole`, `AriaInteractive`, `AriaTarget`, `AriaState` — that form the contract every interactive component must satisfy. `AriaInteractive` is minimal: a role and an accessible name. That's the compile-time layer. The CI layer is axe-core validating rendered output against WAI-ARIA 1.2 required properties. Two enforcement points, zero runtime overhead.

I wanted to avoid adding a new mixin or controller pattern. The existing a11y code in pages-primitives uses four mixins — `FocusTrapMixin`, `RovingTabindexMixin`, `LiveRegionMixin`, `KeyboardShortcutMixin` — all handling *behavioural* accessibility: focus management, keyboard navigation, screen reader announcements. The ARIA attribute contract is orthogonal. Components set their attributes directly through Lit's built-in reflection. No new pattern to learn. An early design had an `AriaController` that would validate attributes at render time, but it created architectural incoherence — four existing a11y primitives are mixins, and adding a fifth as a controller splits the concern across two patterns without justification.

## The navigation API

The interesting part isn't the contract — it's what the contract enables. We built `@casehubio/pages-aria` with three capabilities: a tree walker, a command executor, and a scenario runner.

The walker traverses the composed DOM tree — shadow DOM aware, using the same composed tree traversal patterns that `FocusTrapMixin.collectFocusable()` uses. It finds elements by role and accessible name: `findByRole('button', 'Submit')`. It computes accessible names from `aria-label`, `aria-labelledby`, and text content, in that priority order. It reads ARIA state attributes and returns them as typed objects.

The executor maps ARIA-semantic commands to DOM operations. `click({ role: 'button', name: 'Submit' })` resolves the target through the walker, enforces unique-match (throws if ambiguous), and dispatches the event. `fill`, `select`, `expand`, `collapse`, `assertState`, `waitFor` — each command operates on the ARIA tree, not the DOM tree. The target is always `{ role, name }`, optionally scoped with `within: { role, name }` for disambiguation. Two Delete buttons in a table? `{ role: 'button', name: 'Delete', within: { role: 'row', name: 'Case #42' } }`.

This is the exact vocabulary an MCP server needs. When an LLM drives the UI through MCP tools, it doesn't know CSS selectors or DOM structure. But it can read ARIA snapshots — a compact semantic view of every interactive element on the page — and issue commands in terms of roles and names. `aria_click`, `aria_fill`, `aria_find` become natural tool definitions that map directly to the executor API. The same walker that finds elements for a scenario YAML file finds them for an LLM that's trying to fill in a form.

## Why this matters for MCP

The standard approach to giving an LLM control over a UI is Playwright — full DOM access, CSS selectors, coordinate-based clicking. It works. But it's the wrong abstraction layer. The LLM doesn't think in DOM nodes. It thinks in "there's a submit button" and "there's a text field labelled Customer Name". That's ARIA. That's what screen readers see. That's what the semantic model already describes.

An MCP server built on the ARIA executor offers tools at the right level of abstraction:

- `aria_snapshot` returns the full ARIA tree — every interactive element with its role, name, and state. The LLM gets a compact, semantically meaningful view of the page without parsing HTML.
- `aria_click`, `aria_fill`, `aria_select` operate on `{ role, name }` targets. No selectors to construct, no coordinates to calculate.
- `aria_assert` and `aria_wait` verify state transitions using the same ARIA attributes the UI exposes to assistive technology.

The key insight is that none of this requires a separate automation layer. The ARIA tree IS the automation API. If a component is properly labelled for a screen reader, it's automatically addressable by an LLM. If it's not addressable, it's also not accessible — and that's a bug in both senses.

## The scenario format

We also built a declarative YAML format for scenario automation that uses the same vocabulary:

```yaml
scenario: helpdesk-intake
steps:
  - fill:
      role: textbox
      name: "Customer Name"
      value: "Alice Chen"
  - select:
      role: combobox
      name: "Priority"
      value: "High"
  - click:
      role: button
      name: "Submit"
  - wait:
      role: button
      name: "Submit"
      state: { aria-busy: false }
      timeout: 5000
```

Every step is a verifiable assertion about the UI's ARIA contract. If `fill` can't find a textbox with `aria-label="Customer Name"`, the ARIA is wrong — not the scenario. The YAML runs client-side through the executor with no backend needed. A JSON Schema defines the format, and the same schema will generate Java POJOs for server-side scenario orchestration that mixes API calls with UI commands.

## What we shipped

Eight commits across two sessions. The contract types in pages-primitives. The walker, executor, and scenario runner in a new `@casehubio/pages-aria` package. Widget remediation for seven interactive components — `PagesButton`, `PagesStatusDot`, `PagesBadge`, `PagesGridTable`, `PagesAlert`, `PagesMetric`, `PagesSelector`. ECharts `aria.enabled` on all chart components through the `PagesChartElement` base class, plus `aria-label` propagation on form inputs. axe-core CI validation in the Playwright test suite. And a `CommandResult` variant on the Java push protocol's sealed `PushRequest` interface, extending pages-push for the bidirectional command/response that server-side scenario orchestration needs.

The remaining work — blocks-ui widget remediation for roughly 40 components, the MCP server itself, the Java POJO command model — is scoped and planned but belongs in separate sessions. The foundation is in place. Every pages component that a scenario step or MCP tool might need to address now has the ARIA attributes to be found by role and name.

The thing I keep coming back to is how natural this feels once you commit to it. ARIA isn't a constraint you impose on your UI for compliance. It's the semantic description of what your UI actually does. Once that description is correct, everything that needs to understand the UI — screen readers, scenario tools, LLMs — gets it for free. The alternative is maintaining parallel models that inevitably diverge. I've seen enough `data-testid` drift in my career to know which approach wins.
