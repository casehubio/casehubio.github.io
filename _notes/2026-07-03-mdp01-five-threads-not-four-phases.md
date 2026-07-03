---
layout: post
title: "Five Threads, Not Four Phases"
date: 2026-07-03
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-work]
tags: [backlog, architecture, cloudevents]
---

The backlog had one item in it. That's not because there's one thing to do — it's because nobody had looked at what was open. Ten issues in GitHub, and the handover said "pick up #152."

I pulled the full list and started grouping. The first pass produced four phases — a sequenced attack order that implied a single initiative. It wasn't. The issues fall into five independent threads with different drivers, different blockers, and no dependency between them:

- **Distributed WorkItems (#92)** — the coherent chain: #172 → #97 → #95. CloudEvent contract enables event mesh, which enables federation.
- **External Integrations (#79)** — blocked on CaseHub and Qhorus stability. #39 (ProvenanceLink) waits behind it.
- **Template evolution** — #180, standalone, deferred from the template work in #170.
- **Project hygiene** — #152, the examples module split. Quick, independent.
- **Ideas** — #237 (structured progress) and #238 (saga compensation). Unrelated to each other.

The interesting one is #172. The issue says "emit+listen SWF 1.0 bridge for casehub-work-flow" — a flow module enhancement. But Quarkus-Flow already has native emit+listen with CloudEvent support. The `ListenWorkflow` example shows it: `listen("waitForStartup", toOne("race.started.v1"))` suspends until a CloudEvent arrives on the messaging channel.

So #172 isn't building new Quarkus-Flow infrastructure. It's wiring casehub-work into infrastructure that already exists. And once you see it that way, the engine's `HumanTaskScheduleHandler` is doing the same thing through different plumbing — Vert.x event bus instead of CloudEvents, `WorkItemCreator` SPI instead of emit, CDI lifecycle events instead of listen. Same two-direction pattern: outbound (create a WorkItem), inbound (human finished, resume the caller).

The CloudEvent contract is the shared piece. Define the event types (`work.item.requested`, `work.item.completed`) in `casehub-work-api`, implement the producer/consumer in `casehub-work-flow` for now, and the engine gets a migration path for free when distributed mode arrives under #92.

#172 should probably be rewritten to reflect this — it's platform infrastructure with a flow-specific first consumer, not a flow module enhancement.

The epic hygiene scan also surfaced accumulated debt: 11 specs and 2 blogs on closed workspace branches that never reached main, plus 10 unstamped project branches. None urgent, but it's the kind of thing that compounds silently.
