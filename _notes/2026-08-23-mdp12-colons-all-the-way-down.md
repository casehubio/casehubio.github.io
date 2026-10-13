---
title: "Colons All the Way Down"
date: 2026-08-23
sequence: 12
author: mdp
entry_type: note
subtype: diary
series: fsitrading
projects:
  - casehubio/fsitrading
tags: [push, websocket, topic-registry, ops-centre, pages, dock-workbench, composition]
---

# Colons All the Way Down

The C4 push topics used slashes. C1-C3 used colons. Both worked — until they didn't.

I'd been looking at #32 as a cleanup task: unify the separators before C5b composes both conventions on the same page. Five minutes of string replacements, maybe a test or two. Then I read the `TopicRegistry` source.

## The Trie That Cares About Punctuation

`TopicRegistry` in casehub-pages-push builds a trie. Every method — insertion, lookup, wildcard matching, validation — splits on `:`. Hardcoded. The `*` and `**` wildcards only work on colon-delimited segments.

A topic like `incidents/summary` is a single trie node. No segments. No wildcard matching. `incidents:*` never finds it because there's no `incidents` segment — just one opaque string with a slash in the middle.

The C4 slash topics had been working because the hostPanel components use SSE internally, not `topicSource`. They never needed wildcard subscriptions. The bug was invisible until C5b needed to subscribe to `incidents:*` for the Ops Centre dashboard — at which point every incident push event would silently vanish.

This isn't a style preference. It's a structural invariant of the platform's push infrastructure. I filed an issue upstream to add slash rejection in `EventBroadcaster.broadcast()` so the next person who types a `/` gets an exception instead of silence.

## The composite() Trap

While designing the Ops Centre's data bindings, the spec review caught something worse.

`composite(fetchSource, topicSource)` is the documented pattern for REST-initial + WebSocket-live data. It loads the REST snapshot, then hands off to the WebSocket source for live updates. The problem: the handoff is a clean disconnect. The WebSocket source starts with an empty accumulator. First push event? Single-row snapshot. All the REST data is gone.

The Trading Desk already uses `composite()` for positions, trust, routing, and deliberations. The bug is latent — masked by infrequent synthetic push events. For the Ops Centre, where incidents can fire in bursts, it would have been immediately visible.

I sidestepped it: all Ops Centre datasets use polled `fetchSource` with a 10-second refresh. Not real-time, but correct. The platform needs to fix `composite()` to seed the live source with the REST snapshot — filed that upstream too.

## Two Pages, One Site

The Ops Centre is the second dock-workbench page. Same DSL, same patterns as the Trading Desk — but composing the C4 panels (case-explorer, work-item-inbox, SLA indicator, approval gate, the full ops stack) into their proper home instead of hiding them as `defaultOpen: false` panels in the trading view.

Getting two pages to coexist required understanding how the pages runtime handles navigation. There's no `site()` builder. `page()` supports nested children, but all children share the path segment `"content"` — the second page overwrites the first in `buildPageIndex`. The fix is wrapping child pages in `tabs()`, which creates distinct navigation slots.

The composition itself is mechanical — 11 panels per page, all DSL or hostPanel. The incident dashboard uses `metricGrid` + `eventTimeline` instead of the custom web component the replan spec assumed. The DSL turned out to be expressive enough. Three blocks-ui components (channel-activity, audit-trail-viewer, preferences-editor) aren't published as npm packages yet, so their hostPanel slots render empty placeholders for now. That's fine for pre-release — the layout and data wiring are correct.

## What This Opens Up

The Trading Desk and Ops Centre are now separate workspaces with independent layout persistence. A trader and an ops engineer can each arrange their panels without stepping on each other. The incident summary endpoints (`/summary/severity`, `/summary/status`) return flat tabular data shaped specifically for the DSL's `metric()` component — no client-side aggregation needed.

C6 (Knowledge & Compliance) is the last vertical slice. CBR similarity panels, compliance grid, GDPR erasure — all of which need to compose into the Ops Centre. The multi-page infrastructure is ready for that. The bigger question is whether the `composite()` fix lands before C6 needs real-time CBR event streams, or whether polling carries us through.
