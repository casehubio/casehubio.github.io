---
layout: post
title: "Four buttons and a countdown"
date: 2026-08-20
entry_type: note
subtype: diary
projects: [casehub-soc]
tags: [soc, web-ui, workbench, approval-gate, escalation, human-in-the-loop, blocks-ui]
---

The entire SOC investigation pipeline — alert ingestion, IOC enrichment, ATT&CK mapping, threat assessment — exists to produce evidence for a single moment: an analyst pressing one of four buttons.

Containment is consequential. Isolating a host, blocking an IP range, revoking credentials — these actions are sometimes irreversible, and compliance frameworks (SOC2, DORA, NIS2) require demonstrable human authorization before they happen. The Analyst Workbench is where that authorization lives.

## The decision that matters

The workbench is a two-pane layout: inbox on the left, detail on the right. Selecting a work item loads the live investigation — IOC count, ATT&CK techniques mapped, SLA countdown — and presents an approval gate:

```typescript
const SOC_TRIAGE_OUTCOMES = [
  { key: "CONFIRM_SEVERITY", label: "Confirm Severity", variant: "success" },
  { key: "DOWNGRADE",        label: "Downgrade",        variant: "neutral" },
  { key: "ESCALATE",         label: "Escalate",         variant: "neutral" },
  { key: "FALSE_POSITIVE",   label: "False Positive",   variant: "danger" },
];
```

Four outcomes. The approval gate — a platform component, not SOC-specific code — wraps the decision in prompt, evidence, and confirmation. Selecting an outcome calls `PUT /workitems/{id}/complete`, the output mapping writes the result into case context, and the lifecycle binding picks it up.

The SLA indicator ticking down next to the gate is deliberate. Compliance doesn't just require that a human decided — it requires that it happened within a defined window.

## When nobody shows up

Most SIEM tools treat an untouched alert as a missing data point. It sits in a queue, ages out, maybe gets swept into a "stale" bucket. Nobody decided anything. The system records nothing.

Here, the SLA breach policy fires and the WorkItem transitions to ESCALATED. Then a CDI async observer writes one line into the case context:

```java
ci.getCaseContext().set("analystDecision", "escalated");
```

The goal binding treats this like any other decision. Inaction becomes a recorded event with a timestamp, an actor, and a consequence: the next tier gets the case. The audit trail has no gap.

"Nobody looked at it" is the most common failure mode in SOC operations. The platform's WorkItem lifecycle already handles SLA breach from earlier work. Bridging ESCALATED status to case context is one observer, forty-five lines. The architecture was ready for this — I just hadn't connected the pieces until the workbench made the gap visible.

## Notes went to the wrong store

The original design stored analyst investigation notes in CaseContext — the same key-value map that holds IOC enrichment and ATT&CK mappings. The mismatch surfaced during design review: machine-generated data is structured, schema-driven, and consumed by bindings. Human prose is sequential, timestamped, and authored. CaseContext bindings deserialize the entire map on every access. An unbounded list of analyst notes growing inside it is a performance trap.

Qhorus channels already solve this. Sequential messages with authorship, timestamps, ACLs, and ledger writes — that's what investigation notes are. Notes go as INFORM speech acts on the incident's observe channel. The channel-activity component from Phase 1 renders them. No new endpoint, no new data model. The right answer was already in the platform; the first instinct just reached for the wrong store.

## Five components, two endpoints

The workbench is almost entirely composed from platform parts. `blocks-work-item-inbox` for the task list. `blocks-sla-indicator` for the countdown. `blocks-approval-gate` for the decision. `blocks-notification-inbox` for the bell. All backed by `casehub-work-rest` and `casehub-platform-notifications` — added as Maven dependencies, JAX-RS resources auto-register.

The SOC adds exactly two endpoints: `SocIocSubmissionResource` for manual IOC submission, and the escalation handler. Everything else is wiring.

This is the first real test of the platform-as-composition thesis. The workbench landed in one commit — five blocks-ui components, two SOC endpoints, and the investigation context from Phase 1. Whether the platform delivers on its promise of application-by-assembly depends on what happens when someone actually tries it. So far, the answer is yes.
