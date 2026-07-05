---
layout: post
title: "Where Does the CloudEvent Stop?"
date: 2026-07-05
entry_type: note
subtype: diary
projects: [casehub-work]
tags: [cloudevents, architecture, event-driven, casehub-work]
series: issue-172-cloudevent-workitem-bridge
---

*Part of a series on [#172 — CloudEvent WorkItem Bridge](https://github.com/casehubio/work/issues/172). Previous: [Six Issues, One Branch](2026-07-04-mdp02-six-issues-one-branch.md).*

## Where Does the CloudEvent Stop?

I'd been staring at issue #172 for a while. It was written around the SWF 1.0 emit/listen pattern — a workflow fires a CloudEvent requesting human work, listens for a CloudEvent saying it's done. casehub-work already had the outbound half: `WorkCloudEventAdapter` converts lifecycle events to CloudEvents. The inbound half was missing.

The issue assumed the two event systems would connect naturally — SWF SDK emit fires a CloudEvent, CDI observer catches it, WorkItem gets created. They don't. The SWF SDK has its own in-memory event broker (`InMemoryEvents`), completely disconnected from CDI. quarkus-flow's messaging module bridges to SmallRye reactive messaging channels, not to CDI events. There is no CDI↔SWF bridge.

That rabbit hole cost me time before I realised the issue was simpler than the title suggested. There's no quarkus-flow in this. The SWF emit/listen is the *design pattern* — the architectural shape — not the implementation target. The scope is just casehub-work: receive a CloudEvent, create a WorkItem.

Once I reframed it, the real design question surfaced: where does the CloudEvent stop and the POJO begin?

Within casehub-work, nine internal observers consume `WorkItemLifecycleEvent` directly — typed fields, no JSON parsing, no deserialization. The CloudEvent is the wire format at the system boundary. Inside casehub-work, everything is typed POJOs. The outbound adapter sits at the edge converting POJOs to CloudEvents. The inbound adapter mirrors that: CloudEvent comes in, raw domain data passes through opaquely as a string to the WorkItem's `payload` field, and the template provides all the structural fields. Zero deserialization of domain content.

That "opaque by default" principle turned out to be the most important design decision. The caller's domain data — whatever it is — flows through untouched. If a domain wants typed access, the template's existing `inputDataSchema` (JSON Schema) validates at creation time. Consumers deserialize on demand: `objectMapper.readValue(payload, MyType.class)`. One line. No forced schema. No marshal/unmarshal chains.

The design review drove the spec hard. The original design had a `payloadType` field for Java class-based validation — the reviewer correctly pointed out that `inputDataSchema` already does this with JSON Schema, language-agnostically. The original had no idempotency guard — the reviewer added `findByCallerRef` plus a partial unique index for the TOCTOU race in clustered deployments. The H2 partial index limitation (H2 doesn't support `CREATE INDEX ... WHERE`) led to a Java-based Flyway migration that checks the database type and skips gracefully on non-PostgreSQL.

The most interesting thing that fell out of the design was the event pattern discipline. I'd been checking whether CDI events were being misused as request mechanisms. Within casehub-work, every observer is genuinely a notification — "this happened, react if you care." But `HumanTaskScheduleEvent` in casehub-engine is a request disguised as an event. The engine needs a WorkItem created, but instead of calling `WorkItemCreator.create()` directly, it fires an event that a handler observes. That's a request — it should be a direct call. CloudEvents at the boundary, direct calls internally, events for notifications only.

HumanTask is also in the wrong repo. It's a Work domain concept — "a human needs to do something" — currently living in casehub-engine because that's where the trigger comes from. But proximity to the trigger isn't domain ownership. I filed #290 to relocate it.
