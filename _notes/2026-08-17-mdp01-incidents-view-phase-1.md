---
layout: post
title: "The SOC gets a face"
date: 2026-08-17
entry_type: note
subtype: diary
projects: [casehub-soc]
tags: [soc, web-ui, quinoa, pages, blocks-ui, lit, sse]
series: issue-28-incidents-view
---

Until now, casehub-soc has been an engine with no dashboard. Alerts arrive, workers investigate, containment recommendations get generated — but the only way to see any of it is through REST calls and log output. We gave it a face.

The Incidents view is the first real screen of the SOC web application. Two-column layout: incident list on the left with KPIs and an alert heatmap below, investigation timeline on the right with tabbed detail panels for channels, ATT&CK mapping, and IOCs. The skeleton sidebar from Phase 0 now has a working first tab.

## The layout choice that mattered

The detail pane could have been a vertical scroll of every panel stacked on top of each other. We went with timeline-always-visible plus tabs for the rest. The reasoning: timeline is the spine of an investigation. An analyst working an incident needs to see the chronological flow of alert → enrichment → ATT&CK mapping → containment at all times. The ATT&CK matrix and IOC panel are reference lookups — you consult them, you don't watch them. Tabs keep the view compact without hiding what matters most.

Cross-component selection uses a hybrid approach: URL hash holds the selected incident ID for deep-linking, while pages events give instant in-page reactivity. An analyst can share a link to a specific incident, and browser back/forward works.

## Three SOC components, built for promotion

The ATT&CK matrix, IOC panel, and alert heatmap are new Lit elements sitting in the SOC webui. They follow blocks-ui conventions — `DataSourceMixin` for data binding, `pages-event` emission for user interactions, no SOC-specific imports in the component itself. When they're ready to promote to blocks-ui, it's a file move and a `package.json`, not a refactor.

The ATT&CK matrix renders a 14-column grid matching the MITRE tactic taxonomy, with technique cells colour-coded by confidence score. The heatmap does source × time with intensity driven by alert count. Both emit click events that could filter the incident list — the wiring is ready even though the filter receiver isn't built yet.

## The garden saved us time

Five garden entries directly informed this work. Two were data-binding gotchas we would have hit and debugged: `restSource` needs `dataPath: "items"` for paginated responses, and the first tab in a pages `sidebar()` layout shows "No data" unless you explicitly call `site.navigate()` after `loadSite()`. Two more shaped the push architecture: `EventBroadcaster` fails on `Instant` fields without a custom `JsonWriter` that registers `JavaTimeModule`, and the topic registry uses colon separators, not slashes.

The fifth — composable Lit reactive controllers — we filed for later. It matters when multiple views share the same push connection, which becomes relevant in Phase 2.

## What the push layer revealed

Adding `casehub-pages-push-runtime` as a dependency surfaced a CDI gap: `PushProducers` provides `@DefaultBean` for `EventStore` and `JsonWriter` but not for `SessionSender`. The asymmetry is surprising — you expect the module to be self-contained. The fix is a no-op `SessionSender` producer in the consuming app. We captured this as a garden entry for the next app that hits it.

The push service itself is straightforward. `SocIncidentPushService` observes `CaseLifecycleEvent` and `SocIncidentStatusChangedEvent`, broadcasts to `soc:incidents` and `soc:kpis`. List views use `restSource` with `triggerUrl` for push-triggered refresh. Detail views will use `EventStreamController` for granular append — but that wiring waits for real channel data from Qhorus.

## What opens up

The channels endpoint returns empty — it's a stub waiting for Qhorus channel integration to deliver worker speech acts. Once that's wired, the `blocks-channel-activity` component renders the investigation in real time: COMMAND/DONE messages flowing as the triage pipeline executes.

The remaining sidebar tabs are still placeholders. Phase 2 adds the analyst workbench — work items, SLA indicators, approval gates. Phase 3 is trust dashboards with CBR similarity panels. Each phase fills a tab without touching the others.
