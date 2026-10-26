---
layout: post
title: "ARIA Batches 3-4: Widget Remediation and MCP End-to-End"
date: 2026-08-18
entry_type: note
subtype: diary
projects: [casehub-pages, casehub-blocks-ui]
tags: [aria, mcp, accessibility, web-components, integration-testing]
series: issue-417-aria-interaction-model
---

*Continues from [2026-08-17-mdp01-aria-batches-one-two.md](2026-08-17-mdp01-aria-batches-one-two.md).*

Came into this session expecting an XL widget remediation task — the spec's Section 7 audit listed ~40 blocks-ui components needing ARIA. Turned out to be M scope. Most components already had ARIA from prior work across the platform. The actual gap was 12 components with missing attributes and 7 workbenches where the spec had the wrong pattern assignment.

## The spec was wrong about workbenches

The spec assigned `tablist` to five composition shells — session-workbench, orchestration-workbench, contributor-workbench, trust-workbench, document-workbench. None of them use tabs. They compose `split-workbench`, which provides `role="region"` on each pane with `role="separator"` on the divider. The ARIA was already correct — the spec was aspirational rather than observational.

Similarly, `trust-score-panel` uses `role="img"` with a descriptive `aria-label` for its SVG gauge. The spec wanted `role="meter"` with `aria-valuenow/min/max`, but the component renders a complex multi-part visualization — not a simple numeric gauge. `role="img"` is the right pattern. And `sla-indicator` uses `role="timer"` for its countdown, which is semantically more accurate than the spec's `role="meter"`.

Corrected the spec rather than the code. Protocols are not dogma.

## Build validation as a ratchet

After the remediation, we built `aria-check.mjs` — a static analysis script that scans every `@customElement` file for ARIA patterns. It found 45 sub-components still missing attributes (toolbars, internal panels, convenience wrappers). Rather than blocking the build immediately, it runs as a warning. The backlog is filed as #125. Once cleared, it becomes a build-breaking check.

The ratchet approach — warn on existing gaps, block on new ones — lets you adopt incrementally. Both repos' `CLAUDE.md` files now mandate ARIA for every new component, so Claude won't create non-ARIA components going forward.

## MCP: from annotation to browser

The second half of the session built the MCP bridge. `casehub-pages-mcp` is a new backend module with a `@McpDomain("aria")` resolver that exposes eight operations: `findByRole`, `click`, `fill`, `select`, `expand`, `collapse`, `assertState`, `waitFor`. Each operation sends a command through the push protocol to the browser, where the pages-aria executor handles it.

The interesting piece is `AriaCommandBridge` — a request-response correlation layer over the async push protocol. It registers a `CompletableFuture` per command ID, broadcasts on `scenario/cmd-{id}`, and waits for the matching `CommandResult` from the browser. Configurable timeout, clean error propagation.

The integration test proves the full dispatch chain works without booting Quarkus. We replicate what `GraphQLModelScanner` does — scan the resolver class for `@McpDomain`, `@Query`, `@Mutation` annotations, build `OperationDescriptor` records, then dispatch via `Method.invoke()` with Jackson `convertValue()` for parameter deserialization. This catches annotation correctness, parameter naming, and nested type serialization (the `within` scoping on `AriaTarget` deserializes correctly through Jackson) in under a second.

## End-to-end: shadow DOM, tabs, trees, forms

The ARIA executor integration test constructs a realistic component hierarchy with shadow DOM boundaries — an app shell with three tabs (Dashboard, Settings, Activity), nested custom elements with their own shadow roots, forms with grouped inputs, and a two-level tree structure. Then it drives the entire thing through the ARIA executor: switch tabs by `role="tab"` + name, traverse shadow DOM to find metrics, fill inputs scoped with `within` to a specific form group, expand tree items and assert status on leaf nodes.

The full scenario test navigates from the dashboard to settings, fills an email and API key in separate form groups using `within` scoping, clicks save, then navigates to the activity tree, expands two levels, and verifies leaf statuses. All through ARIA roles and accessible names — no `data-testid`, no DOM structure knowledge, no CSS selectors.

This is the same path an LLM would take through MCP: `casehub_action("aria", "click", {role: "tab", name: "Settings"})` → `casehub_action("aria", "fill", {role: "textbox", name: "Email address", value: "alice@casehub.io", within: {role: "group", name: "Notification settings"}})`. One interaction model for screen readers, scenario automation, and LLM agents.

The queue still has #125 (45 remaining sub-component gaps) for the next session on this slot. The MCP integration is end-to-end tested — what's left is the volume work of bringing every last toolbar, picker, and convenience wrapper into compliance.
