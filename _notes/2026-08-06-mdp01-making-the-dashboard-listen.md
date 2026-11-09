---
layout: post
title: "Making the Dashboard Listen"
date: 2026-08-06
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [life-ui, sse, real-time, notification-badge, trust-scores, demo-mode]
series: issue-81-life-ui-design
---

*Part of a series on [#81 — Life UI design](https://github.com/casehubio/life/issues/81). Previous: [The Dashboard Comes Alive](2026-08-05-mdp01-dashboard-comes-alive.md).*

A household dashboard that only shows you the state of affairs when you load the page isn't a dashboard — it's a report. The difference matters when a contractor confirms a Thursday appointment at 2pm and you're looking at Monday's snapshot. Or when an SLA breach fires on a GP follow-up that's already three days overdue and the number next to "Due Today" still says zero.

The previous session built the static version: morning briefing, KPI strip, action items, five views, dock panels. All rendering real platform data. This session wired the real-time layer — SSE events push state changes to every panel the moment they happen, a notification badge counts what's arrived since you last looked, and the trust scores that were missing from the KPI strip now show 82% across five external actors.

## Why SSE matters for life automation

CaseHub's accountability properties — SLA enforcement, commitment tracking, watchdog escalation — generate events continuously. A contractor misses an ETA and a watchdog alert fires. A health appointment SLA breaches and an escalation event propagates. A financial decision requires oversight approval and a gate opens. Each of these changes the state of the dashboard: pending action counts, SLA compliance percentages, active case status.

Without real-time updates, a household admin checking the dashboard at 9am sees accurate data. By 9:15, after three work items were created by automated case workers, the data is stale. The morning briefing says "2 items need your attention" when the real number is 5. The gap between what the dashboard shows and what's actually happening is exactly the gap that casehub-life is designed to close — the foundation provides formal tracking, but the UI has to keep up.

The SSE infrastructure was already built on the backend — `LifeEventBroadcaster` publishes CDI events from casehub-work and casehub-engine through a multiplexed `/events/stream` endpoint. The missing piece was the frontend: connecting that stream to the Lit components that render the data.

## Two SSE clients, only one does SSE

The pages platform has two event-streaming abstractions. `EventStreamController` in `@casehubio/blocks-ui-core` is a Lit Reactive Controller with lifecycle management — exactly what you'd reach for. The name says "event stream," it auto-connects on mount, auto-disconnects on unmount. But it wraps `EventStream` → `EventConnection` → `new WebSocket()`. It's a WebSocket client with an SSE-sounding name.

The actual SSE utility is `SSEManager` in `@casehubio/pages-data/dist/sse/`, using native `EventSource` with connection pooling, exponential backoff reconnection, and RAF-batched dispatch. This is what life-ui needs — the backend serves standard SSE, not a WebSocket topic protocol.

The naming collision is worth knowing about because it's the kind of thing you discover after wiring the wrong client to your endpoint and wondering why nothing arrives. The two abstractions serve different architectural patterns: `EventStream` is for structured topic-based push (where the server manages subscriptions and sequence numbers over WebSocket), `SSEManager` is for standard HTTP SSE streams.

## The debounce problem you don't see coming

The natural pattern for SSE-driven refreshes is simple: event arrives, re-fetch data from REST. The SSE event is a signal, not the data — each component re-queries its own endpoint to get the current state. This keeps REST as the single source of truth and avoids the complexity of maintaining client-side state from event payloads.

The problem appears when ten events arrive in the same animation frame. A case engine starting a batch run emits `CASE_STARTED` for each case. SSEManager's RAF batching delivers them efficiently — but then each handler fires a `fetch()`. With four dashboard panels subscribed, that's forty concurrent REST requests in under 16 milliseconds. The server handles it, but it's pure waste — the second through tenth fetches return the same data as the first.

The fix belongs in the controller, not in each component. `LifeEventController` debounces the `onEvent` callback at 300ms — events within that window coalesce into a single refresh. The unread count increments immediately (the notification badge responds to every event), but the expensive REST call waits for the burst to settle. This is a meaningful distinction: the badge tells you something happened right now, the data refresh tells you what the world looks like after the dust settles.

## Trust: the number that earns its place

The KPI strip shows five metrics: Active Cases, SLA Compliance, Pending Actions, Due Today, and Trust Average. Four of those are operational — counts and percentages that change with work item lifecycle events. Trust Average is different. It's a Bayesian reputation score computed from attestation records in the tamper-evident ledger, aggregated across every external actor the household interacts with.

When the dashboard shows 82%, that's the mean global trust score across Bob's Plumbing (0.72 — decent work, sometimes late), Dr. Patel (0.91 — reliably accurate and proactive), Harris & Co Solicitors (0.84 — dependable but their quotes drift), Oakwood Primary (0.88 — communicates well), and Maria Santos the carer (0.76 — good but needs monitoring on deadlines). Each score has per-dimension breakdowns: `deadline-reliability`, `cost-accuracy`, `factual-accuracy`, `proactive-alerting`. Bob scores 0.58 on deadline reliability but 0.85 on cost accuracy — his quotes are accurate, he just doesn't always show up when he says he will.

This matters because the trust score drives the platform's agent routing. When a case worker needs to assign a contractor task, the trust-weighted routing strategy uses these scores to decide whether to route directly, add monitoring, or require oversight approval. A low `deadline-reliability` score on Bob triggers a watchdog sentinel to follow up automatically. The dashboard KPI makes this invisible infrastructure visible — the household admin sees the number and can decide whether 0.58 on deadline reliability is acceptable for the next job, or whether it's time to find a different plumber.

## Quarkus security: permit-all doesn't

Running the demo from a packaged jar instead of `quarkus:dev` produced 403 Forbidden everywhere. The demo config disables OIDC and sets a permit-all HTTP permission. The expectation: all endpoints accessible without authentication.

Quarkus security has two independent enforcement layers. `permit` at the HTTP layer allows the request through Vert.x routing. `@RolesAllowed` on the resource method is a CDI interceptor that checks `SecurityIdentity.getRoles()` — a completely separate check that requires an actual authenticated identity with the declared roles. With OIDC disabled, there's no authentication mechanism to create one. The `auth.enabled-in-dev-mode=false` property disables both layers, but only in `quarkus:dev` mode. There's no production equivalent.

The fix is a one-class `DemoIdentityProvider` — an `IdentityProvider<AnonymousAuthenticationRequest>` with `@IfBuildProfile("demo")` that creates a `QuarkusSecurityIdentity` with household-admin roles for every anonymous request. Profile-scoped, invisible in production, and the only way to make jar-mode demos work without OIDC.

The broader point: the gap between "permit all requests" and "permit all requests with the roles they need" is non-obvious in Quarkus. HTTP permissions and CDI security annotations are orthogonal systems that happen to both say "security" on the tin.

## Where this is heading

The dock panel expand/collapse still doesn't work — the toggle mechanism in pages-runtime looks correct in code but doesn't produce visible results. That needs browser dev tools, not a code audit.

The more interesting open question is what happens when SSE meets real case execution. The demo dashboard runs on static seed data — no engine, no case workers, no events flowing. The SSE wiring is in place, the debounce handles bursts, the notification badge counts arrivals. But the first time a care coordination case runs with four agents executing in parallel, pushing work item updates and case lifecycle events through the SSE stream while the dashboard is open — that's when we'll find out whether the 300ms debounce window is right, whether the per-component type filtering is granular enough, and whether the REST re-fetch pattern holds up under real load.
