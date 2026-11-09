---
layout: post
title: "The Dashboard Comes Alive"
date: 2026-08-05
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [life-ui, dashboard, dock-workbench, demo-mode, lit, vite]
---

The spec called for two tiers. Tier 1: real platform data surfaced through a dock workbench layout — tasks, cases, contacts, trust, commitments, SLA compliance. Tier 2: mock panels showing the executive assistant vision — email extraction, financial tracking, family summaries from ambient data. We built both in one session.

## What it looks like

The dashboard is an IntelliJ-style dock workbench. Left docks: inbox, cases, calendar. Right docks: family, money, comms. Centre: a vertical stack of morning briefing, KPI strip, action items, and active cases grouped by domain. The briefing says "Good morning — 2 items need your attention today" and shows exactly which two. The KPI strip shows 4 active cases, 6 pending actions, 6 due today. The action items are urgency-sorted — amber borders for due-soon, neutral for normal. Every piece of this is real data from the backend.

The dock workbench uses the pages-runtime's `loadSite()` with `hostPanel()` references that resolve via a panel registry. Each panel type maps to a custom element through `registerPanel()`. The resolution chain is non-obvious — I traced it through three separate modules (`builders.js`, `panel-registry.js`, `activation.js`) before understanding how the pieces connected. The components must be both imported (so `customElements.define()` runs) and registered (so `lookupPanel()` can find them) before `loadSite()` renders the tree.

## Five views, all live

Beyond the dashboard home, we built inbox (split list/detail with urgency filtering), people (external actor list with search, trust tabs), cases (domain-grouped with status filters), and journal (stacked analytics — overview stats, cases-by-type table, SLA compliance bars with colour coding). The app shell gained a toolbar: notification bell, theme toggle between `casehub-light` and `casehub-dark`, and a user identity chip.

## The demo mode saga

Getting the backend to serve real data in demo mode uncovered a cascade of pre-existing problems. Flyway migrations used PostgreSQL-specific syntax — partial unique indexes with `WHERE` clauses, multi-column `ALTER TABLE`, PL/pgSQL `DO $$` blocks — none of which H2 supports. The `work_item_template` schema had drifted: `category` column gone, `tenancy_id` and `version` now required. The `ExternalActor` enum changed from role-based values (`CONTRACTOR`, `DOCTOR`) to type-based (`EXTERNAL_HUMAN`, `AI_AGENT`). Two different modules both had a V2000 migration, causing Flyway to refuse to start.

We fixed the individual migrations, then stepped back and took a different approach entirely: Hibernate `drop-and-create` with `sql-load-script` for demo mode, bypassing Flyway completely. This creates all tables from JPA entity annotations and loads a single `import-demo.sql` with the household data — 5 external actors, 5 case trackers, 8 work items across all domains.

Then Quinoa's HTTP forwarding hung silently. Server started fine — "started in 4.2s" — but every request to port 8080 hung indefinitely. No error, no timeout. Claude traced it to an IPv4/IPv6 mismatch: on macOS, Vite binds to `::1` only, Quinoa's Vert.x HTTP client tries to forward but the connection pipeline hangs. The workaround: disable Quinoa, run Vite standalone with its own proxy config pointing to `127.0.0.1:8080`. Two processes instead of one, but both start instantly and the dashboard renders with live data.

Then auth. With OIDC disabled, `@RolesAllowed` still returns 403. The `permit-all` HTTP auth permission uses `/*` which only matches single-level paths — `/dashboard` passes, `/dashboard/briefing` doesn't. The fix: `quarkus.security.auth.enabled-in-dev-mode=false`, which works regardless of the active profile because `mvn quarkus:dev` always runs in DEV launch mode. And finally, the visibility policy. `CurrentPrincipal.groups()` returns empty without OIDC, so the case query filters everything out — HTTP 200 with zero results, no error. A `DemoCurrentPrincipal` with `@IfBuildProfile("demo")` that returns household-admin groups resolved it.

Each fix revealed the next failure. But the end state is clean: the dashboard renders all five views with live data from a household that exercises all eight domains.

## Where this is heading

The mock panels show the direction. The comms panel has email and WhatsApp extraction examples — "Bob's Plumbing: Thursday 2pm → extracted: appointment → updated: contractor case → trust: confirmed commitment." The money panel shows household spend broken down by family member. These are static HTML now, but they demonstrate the structural advantage: every extracted obligation flows through the same accountability pipeline as a manually created task. The system discovers, researches, recommends, and waits for human approval — tracked end to end.
