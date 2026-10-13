---
layout: post
title: "Two agents for every job"
date: 2026-07-30
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-soc]
tags: [dual-agent, workers, routing, soc]
series: issue-18-layer2-triage-workers
---

*Part of a series on [#18 — Layer 2: triage workers](https://github.com/casehubio/soc/issues/18). Previous: [Wiring the RAS pipeline](2026-07-30-mdp02-wiring-the-ras-pipeline.md).*

A Security Operations Center runs on a simple loop: alerts come in, someone decides what they mean, someone else decides what to do about them. The "someone" used to be a human analyst staring at a SIEM dashboard at 3am. Increasingly it's a mix of automated rules and AI — but the handoff between those two has been handled badly almost everywhere I've looked.

The typical pattern is either-or. You deploy rule-based detection and get speed plus predictability. Or you deploy an LLM agent and get reasoning plus natural language output. Switching between them is a configuration change or a deployment decision. The system runs one or the other, never both at once, and never with any feedback loop between them.

We took a different approach with casehub-soc. Both worker types are registered for every capability from day one. The system doesn't choose between them at deployment time — it chooses at dispatch time, per incident, based on accumulated trust.

## Three capabilities, six workers

The SOC investigation pipeline has three stages. Each stage has a rule-based and an LLM implementation:

**IOC enrichment** extracts indicators of compromise from alert data. The rule-based worker runs regex patterns against the alert payload — IPs, file hashes, domains, CVEs. The LLM worker sends the same data to Claude and gets back a reasoned analysis of what the indicators mean and how they relate.

**ATT&CK mapping** takes those indicators and maps them to the MITRE ATT&CK framework. The rule-based worker uses a lookup table: `credential-harvesting + EMAIL → T1566 (Phishing)`. The LLM worker reasons about the attack narrative and constructs the full chain.

**Containment recommendation** decides what to do. The rule-based worker applies a decision matrix: severity + tactic → action. CRITICAL + CREDENTIAL_ACCESS → revoke credentials. The LLM worker considers the full investigation context and explains its reasoning.

Both worker types produce output that conforms to the same contract. Downstream stages don't know or care which type produced the data.

## Routing is the interesting part

This only works because of how CaseHub dispatches workers. When a capability binding fires, the engine finds all workers registered for that capability and asks the `ComposableAgentRoutingStrategy` to pick one. The strategy blends signals from multiple providers — trust scores, workload, prior experience — and selects a single winner.

In Bootstrap mode (no trust history), all signals are neutral and the rule-based worker wins by convention. This is deliberate. Rule-based workers are deterministic, fast, and need no external API calls. They're the reliable baseline.

As incidents resolve and analysts attest to investigation quality, trust scores diverge. The LLM worker starts getting selected for cases where its reasoning has proven more thorough. The rule-based worker keeps handling the straightforward cases where speed matters more than depth.

The routing strategy doesn't just select — it also excludes. A worker whose trust score drops below threshold gets filtered out entirely. If an LLM worker starts producing unreliable ATT&CK mappings, it stops getting dispatched for that capability. The system self-corrects without anyone changing a configuration file.

## What this means for operators

The SOC operator doesn't think about any of this. They see an incident, they see the investigation findings, they make a decision. Whether the IOC enrichment came from regex matching or Claude's analysis is metadata, not workflow.

But for the SOC manager watching resolution quality over time, the routing architecture produces something useful: a system that gets better at matching the right tool to the right incident, based on evidence rather than assumptions about which approach is "better."

The cold-start problem is real — LLM workers can't build trust if they never fire, and they don't fire in Bootstrap. That's a Layer 4 problem. The foundation for solving it is here: two workers per capability, a routing strategy that can differentiate them, and a trust model that accumulates evidence from analyst attestations.

The containment recommendation is where this gets consequential. When a worker recommends isolating a host or revoking credentials, the engine gates that action through a risk classifier. High-risk containment requires human approval before execution. The same gate applies regardless of which worker made the recommendation — rule-based or LLM. The accountability model doesn't bend for AI.
