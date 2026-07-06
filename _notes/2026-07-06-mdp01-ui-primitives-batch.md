---
layout: post
title: "UI Primitives: Approval Gates, SLA Deadlines, and the Theme Bug Nobody Saw Coming"
date: 2026-07-06
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-blocks-ui]
tags: [web-components, lit, theming, shadow-dom, accessibility]
series: issue-13-ui-primitives-batch
---

The queue board redesign left blocks-ui with a solid inbox and detail panel, but no shared primitives for the things every CaseHub app actually needs: deadline indicators, metric dashboards, and approval gates. Four issues — #8, #12, #13, #16 — covered the gap. I decided to batch them on a single branch because the components share patterns and the cleanup items from the epic #4 review would exercise the same files.

Three new components. `<sla-indicator>` renders a countdown that transitions through normal → warning → critical → breached states based on configurable thresholds. Rather than each instance running its own `setInterval`, a `SharedTimerController` in blocks-ui-core provides one module-level timer that starts when the first subscriber connects and stops when the last disconnects. Fifty indicators in a queue view share one interval.

`<kpi-metric-row>` renders a responsive grid of metric cards — value, label, trend arrow, inline SVG sparkline, status border. The sparkline is a pure `<polyline>` with a gradient `<polygon>` fill, no charting library. Cards emit `pages-event` on click for drill-down navigation.

`<approval-gate>` is the most substantial. Clinical trials need PI authorisation gates. AML needs SAR filing decisions. Life needs M-of-N family quorum votes. The engine itself creates oversight gates when agents make high-risk decisions. Each domain had bespoke approval UI. This component replaces all of them with configurable outcomes, an evidence slot, quorum tracking, and SLA deadline integration via the `<sla-indicator>`.

The adversarial design review caught things I'd have shipped as bugs. The spec originally had the approval gate POSTing to a custom `/gates/` endpoint with a `reason` field — the existing `casehub-work` API uses PUT to `/workitems/{id}/complete` with a `resolution` field. The reviewer also flagged that the confirmation dialog's click-outside-to-dismiss behaviour was a safety risk for clinical approvals — an accidental backdrop click during a SUSAR assessment could dismiss a decision the approver thought they'd confirmed. The dialog now has a `persistent` mode that blocks backdrop dismissal while still allowing Escape.

The cleanup pass addressed the remaining items from the epic #4 review: schema-form type unification (the codebase had two near-identical `FieldSchema` types), date/datetime field rendering, `LiveRegionMixin` wired into the inbox and detail components, the batch cancel `window.confirm()` replaced with a styled dialog, hardcoded pixel values replaced with design tokens, `aria-controls` linkage on the detail tabs, and `IntersectionObserver` deferred loading on the queue pill bar.

The showcase verification caught a bug the tests couldn't. The workbench component had its own `.theme-light` and `.theme-dark` CSS classes inside its shadow DOM, declaring hardcoded values for `--blocks-neutral-1` through `--blocks-neutral-12`. These shadow-internal declarations silently overrode the inherited tokens from the document-level theme. Toggling dark mode worked on every other component but not the workbench — no error, no warning, just stale light-mode colours. The fix was removing the shadow-internal theme CSS entirely. Components inherit tokens from the document-level theme; redeclaring them inside shadow DOM creates a parallel system that will always drift.

The same verification found that density compact mode barely changed anything — the token overrides only covered `font-size-base` and `font-size-sm`, leaving headings (`font-size-lg` and above) and larger spacing unchanged.
