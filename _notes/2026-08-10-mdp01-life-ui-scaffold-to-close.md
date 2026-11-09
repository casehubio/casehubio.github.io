---
layout: post
title: "Life UI — scaffold to close"
date: 2026-08-10
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [life-ui, lit, sse, demo-mode, dock-workbench]
---

The Life UI branch started with a question I hadn't answered cleanly: what does a personal life coordination system actually look like when someone opens it? Eight foundation layers of commitment tracking, SLA enforcement, trust routing, and Merkle audit trails — all accessed through REST endpoints nobody had seen rendered.

The answer turned out to be an IntelliJ-style dock workbench. CSS grid with 32px dock bars on left and right, 280px toggle panels, centre content filling the remainder. Three left docks (inbox summary, cases overview, calendar preview), three right docks (family summary, money, comms). The last two are static HTML mockups — placeholders for Open Banking and WhatsApp intake that don't exist yet but show the vision.

Five views landed: Home (the dock workbench), Inbox (split list/detail wired to `/pending-actions`), People (five tabs — details, trust scores, activity timeline, tasks, GDPR erasure), Cases (domain filter, status filter, SSE live refresh), and Journal (analytics dashboard with case stats, SLA compliance bars, trust averages). Twelve panel components total, all Lit 3.x Web Components consuming blocks-ui packages resolved through Vite aliases to Maven SNAPSHOT artifacts.

The SSE integration was the piece that made it feel alive. `LifeEventController` wraps `SSEManager` from pages-data as a Lit Reactive Controller — connect on mount, disconnect on teardown, debounced callbacks so a burst of work-item events doesn't hammer the DOM. The notification badge in the app shell tracks unread count from SSE events and clears when you navigate to Inbox.

Demo mode needed more work than expected. Quinoa's dev server proxy wouldn't forward correctly on macOS — IPv4/IPv6 dual-stack issue that caused the frontend to hang on every API call. The fix was simpler than the debugging: disable Quinoa entirely in demo profile, run Vite standalone on port 5173 proxying to Quarkus on 127.0.0.1:8080. Two terminal windows instead of one, but it works.

The views are wired to real endpoints but several still fetch at the wrong granularity — Cases Tasks tab pulls the entire domain's tasks, not the selected case's tasks. The audit trail tab is a placeholder. The Journal has stats but no decision log. Epic #85 picks up from here: ledger REST endpoints, case-scoped queries, and blocks-ui component integration for the tabs that need real data. Nine issues, four batches, dependency-ordered so the backend enablers land first.
