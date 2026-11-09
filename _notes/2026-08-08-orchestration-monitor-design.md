---
title: "Status Registries, Decision Chains, and the Art of Not Building New Components"
entry_type: note
subtype: diary
projects: [casehub-blocks-ui]
tags: [web-components, orchestration, status-registry, timeline, design]
date: 2026-08-08
status: draft
---

The orchestration framework has eighty source files defining execution states, pattern types, agent results, failure policies, and an EU AI Act Art.12 compliance audit chain. None of it had UI representation. I needed to fix that without creating a parallel universe of components.

## Two Data Flows, One Domain

The first thing that became clear: orchestration monitoring is two fundamentally different problems wearing the same label. Live execution state — what's happening right now — is SSE-pushed, ephemeral, changes every second. The historical audit trail — what happened and why — is REST-pulled, immutable, append-only ledger entries. Treating them as one component would mean fighting the data flow at every turn.

So: separate components. An execution monitor for the live stream. A timeline view for the audit chain. A workbench shell that composes them. Three things that each do one thing cleanly.

## Reuse Over Invention

The temptation with a domain this rich is to build domain-specific badges, domain-specific timelines, domain-specific everything. But blocks-ui already has the primitives.

The status registry — originally built for case states, task states, work items — turns out to handle orchestration states perfectly. Seven execution states, four agent result statuses, and eight pattern types all went in as new domains. `StatusBadge` renders them with zero code changes. The registry is a lookup table, not a state machine, so patterns — which are static metadata, not dynamic status — work just as naturally as execution states that change every second.

The timeline strategy pattern was the other key reuse. blocks-timeline already supports pluggable strategies — event chronology, state progression, commitment lifecycle. Adding an orchestration-events strategy meant writing `toNodes` (which maps audit events to timeline nodes) and `renderDetail` (which renders domain-specific detail per event type). The timeline component handles layout, pagination, filtering, expansion — all for free.

The net result: nineteen status registry entries, one timeline strategy, and two new components. Not seven components and a new badge system.

## The Garden Entry That Saved a Wrong Turn

Claude searched the knowledge garden during design and surfaced a gotcha I would have hit mid-implementation: `EventStreamController` — despite its name — is WebSocket-based, not SSE. The actual SSE primitive is `SSEManager` from pages-data, imported from a completely different path. The naming is misleading enough that session-detail is the only component in the codebase that uses it correctly. Without that garden entry, I'd have wired up EventStreamController, watched it fail silently against an SSE endpoint, and spent an hour figuring out why.

## What the Review Caught — and What It Got Wrong

Design review found a real gap: the typed event payloads were disconnected from the audit event interface. The spec had `payload: unknown` — meaning the timeline strategy would need runtime type casts at every switch branch. Making it a discriminated union keyed on a `type` field lets TypeScript narrow the payload type at each case. Simple fix, but the kind of thing that becomes painful to retrofit once tests are written against the wrong shape.

What the review got wrong: three independent reviewers flagged SSEManager, StatusBadge, and split-workbench as "phantom infrastructure that does not exist." All three exist — they just live in pages-data or blocks-ui-core, not in the source tree the reviewers were given. A reminder that review scope determines review quality. Reviewers without access to dependencies will confidently report that your dependencies don't exist.

## What This Opens Up

The orchestration types are in blocks-ui-core now, available to any component. audit-trail-viewer can render orchestration-specific payloads using the typed interfaces. kpi-metric-row can display execution metrics using the pattern and execution domains for badge rendering. The status registry approach means any future domain — workflow engines, IoT device states, whatever — follows the same pattern: register the domain, and StatusBadge handles the rest.
