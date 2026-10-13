---
title: "Three Tiers of Response"
date: 2026-09-28
author: mdp
entry_type: note
subtype: diary
projects: [casehubio/fsitrading]
series: issue-50-c13-trading-yaml-playbooks
tags: [yaml-core, playbooks, cbr, design, orchestration]
---

The trading desk has three orchestration layers now. Case definitions detect and dispatch — reactive choreography that fires bindings when context changes. Playbooks coordinate the response — imperative step sequences with deadlines, scatter-gather, state machines. Prose resolution documents surface guidance when neither automation nor known playbooks apply.

I'd been circling this architecture for weeks without naming it cleanly. The vertical-slice work through C1-C7 built the detection layer (Market Pulse, RAS situation awareness) and the response agents (13 incident response agents, strategy arena), but the coordination between "something happened" and "here's what to do about it" was implicit. Hard-coded agent dispatch in Java classes. When I wanted to add a new response pattern — say, a flash crash response that halts orders, assesses exposure across all instruments, closes positions sequentially through a risk gate, then monitors until recovery — the only path was writing a new Java orchestrator class.

yaml-core changes that. The step catalog landed last week in platform — 172 production classes, six invoke types, a 12-position decorator chain. A step definition file declares what actions exist; a playbook file declares how to compose them. Drop a YAML file on the classpath and the step catalog picks it up. No recompilation for new actions, no new Java classes for new coordination patterns.

The design question that consumed most of this session: how do playbooks relate to case definitions? Cases use engine choreography — bindings fire on context changes. Playbooks use yaml-core's imperative step model — sequential execution with control flow. Two different orchestration paradigms. I considered three options: playbooks replace case definitions (one model), playbooks are standalone (independent of cases), or playbooks complement cases (two layers, each doing what it's best at).

The complement model won. A case definition detects a situation via its bindings and dispatches to a worker. The worker's `do:` block calls `casehub:step-file`, which resolves to the playbook's yaml-core step file. The playbook runs its step sequence with access to the case context via `${context.*}` variables. On completion, outputs merge back into the case context. The case's CBR observer records the outcome automatically — no new wiring needed in fsitrading.

What I like about this layering: each model handles what it's good at. Case definitions are excellent at reactive detection — "when this context changes and these conditions hold, fire this binding." Terrible at "do these five things in sequence with a 120-second deadline and scatter-gather to three agents." Playbooks are the opposite — excellent at imperative coordination, but they don't watch for events. Put them together: the case watches, the playbook acts.

The design review surfaced 74 issues across four dimensions. The significant ones: MCP tool names didn't match the platform's actual `@McpDomain` convention (I'd used fictitious names), the SPI interfaces needed to be consumer contracts designed for the step definition inputs/outputs rather than extracted from existing implementation signatures, the flash crash playbook had a race condition where concurrent position closes would read stale risk state, and the CBR observers were hardcoded to the overnight-incident case type — four of five playbooks would have been invisible to the feedback loop.

The flash crash race condition is worth noting. The original design used `forEach` with `concurrency: 3` — three parallel position closes while three concurrent risk assessments read the portfolio. The problem: each close mutates portfolio exposure, so the risk assessment denominator changes under your feet. The fix is sequential. Slower, but the risk decisions are consistent. In a flash crash, correct is more important than fast.

One surprise at the end: the app module doesn't compile. Not because of anything in this epic — the neocortex CBR API was refactored upstream, and 11 classes in fsitrading reference types that no longer exist. `CbrCaseMemoryStore` became `CbrRecordStore`, `ResolvedCase` and `FeatureVectorCbrCase` disappeared, and several retrieval types were removed. Filed as #63. The SPI interfaces and their tests are clean (api module compiles and 141 tests pass), but any app module work is blocked until the CBR API migration is done.

The implementation plan has 12 tasks across 6 batches. One complete (SPI interfaces with `@SimulationEligible`), one partial (OrderSemaphoreService — the reference-counted order-halt mechanism). The YAML-only work (step definitions, playbooks, shared modules, corpus files) is the bulk of the epic and doesn't depend on the CBR fix. The fix gates the Java adapter implementations and the CBR observer generalisation.

What this opens up: once the playbooks are authored and the engine's `StepFileCallableDispatcher` lands, the trading desk has a library of coordinated response patterns that learn from outcomes. A flash crash at 3 AM fires the flash-crash-response playbook, which halts orders, assesses risk, closes exposed positions through a gate, monitors recovery, and — via CBR — uses last month's flash crash response to inform tonight's triage. The prose documents handle what playbooks can't: counterparty negotiations, regulatory inquiries, unprecedented events. Three tiers, each for a different kind of problem.
