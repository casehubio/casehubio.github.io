---
layout: post
title: "Wave 3: The Workbench Takes Shape"
date: 2026-08-05
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [frontend, blocks-ui, workbench]
---

# casehub-aml — Wave 3: The Workbench Takes Shape

## Placeholders out, real components in

The AML workbench had three visible gaps — trust scores showing "pending blocks-ui", an officer review section with nothing but a callerRef string, and a compliance view that was just an inbox list. Wave 3 fills all three with actual blocks-ui components, and the UI starts to feel like a real investigation tool rather than a wireframe.

## Trust scores per routing decision

The Routing & Trust tab previously had a placeholder where trust scores should appear. We replaced it with `blocks-trust-score-panel` in compact mode, one per routing decision, laid out in a responsive grid. Each card shows the capability tag, the selected worker name, and the trust score visualisation. A compliance officer looking at a completed investigation can now see at a glance which agents handled which stages and how much the system trusted each one.

The card layout is intentional — trust scores are per-worker, not per-investigation, so a grid of small cards reads better than a single table row. The `compact` mode on the panel component strips it down to just the score arc and value, which is all that's needed at this level of detail.

## Officer review: from callerRef to work-item-detail

The Compliance detail tab had a placeholder div that displayed the raw `callerRef` string and a note saying "work-item-detail integration will be added when work-items endpoint is available." We removed that and dropped in `blocks-work-item-detail`, wired to the work item ID from the compliance evidence response. An officer reviewing a case now sees the actual work item — title, status, priority, permitted outcomes, claim deadline — not a reference string.

## Compliance nav: inbox to workbench

The compliance view was using `work-item-inbox` — a flat list of pending items. We switched to `blocks-work-item-workbench`, which gives a split-pane layout with the list on one side and the selected item's detail on the other. This is a closer match for how compliance review actually works: officers don't just scan a queue, they select an item and work through it, reviewing the SAR narrative, checking the evidence chain, and approving or rejecting.

## The blocks- prefix lesson

Every blocks-ui component registers its custom element with a `blocks-` prefix — `blocks-split-workbench`, `blocks-list-pane`, `blocks-detail-pane`, `blocks-work-item-inbox`. The old tags in the templates were unprefixed (`split-workbench`, `list-pane`), and they worked because the previous version of blocks-ui didn't enforce the prefix. The current version does, and the failure mode is silent — no error, no warning, the element just never upgrades from HTMLElement. We caught it and migrated all the tag names, but it's the kind of thing that could easily cost someone an hour of debugging.

The workbench is still running against mock data — the showcase mock-fetch layer now has realistic work items with proper FinCEN SLA deadlines, permitted outcomes, and callerRef linking back to investigations. The mock layer is getting substantial enough that it functions as a living specification for what the real API responses need to look like.
