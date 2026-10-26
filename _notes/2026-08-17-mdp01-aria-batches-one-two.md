---
layout: post
title: "ARIA Batches 1-2: From Decisions to Wire Protocol"
date: 2026-08-17
entry_type: note
subtype: diary
projects: [casehub-pages]
tags: [aria, adr, protocol, java, push]
series: issue-417-aria-interaction-model
---

*Continues from [2026-08-16-mdp02-slot-tooling-aria-queue.md](2026-08-16-mdp02-slot-tooling-aria-queue.md).*

The queue from last session had three batches. This session cleared the first two — conventions and command infrastructure — leaving the big one (widget remediation) for next time.

Batch 1 was codification. The ARIA design spec already captured every decision — the unified interaction model, contract-as-types plus axe-core CI, the command vocabulary, the push protocol extension. ADR 0002 and a platform protocol distil those decisions into standing artifacts that outlast the branch. The ADR argues the case: one targeting system instead of two, so automation bugs are accessibility bugs. The protocol gives the enforcement rules — pattern table by component type, the two-layer check (compile-time interface, CI axe-core), and an explicit prohibition on `data-testid` as a parallel identity system.

Batch 2 had two pieces that connect across languages. The Java side needed records for the scenario command model — `Scenario`, `ScenarioStep`, `AriaTarget` — and a YAML parser. I put these in a new `casehub-pages-scenario` backend module rather than adding YAML parsing dependencies to the lean `push` module. The records use the flattened representation from the design spec: action, target, value, state, timeout. The YAML itself uses a discriminated union — action name as the key, target object as the value — so the parser does the shape conversion. Both sides validate against the same JSON Schema that already exists in `pages-aria/scenario`.

The TypeScript side needed the push channel handler — the `@casehubio/pages-aria/server` entry point that the `package.json` already declared but had no implementation. Claude built the `ScenarioHandler`: subscribe to `scenario/*` topics, receive command events, execute them through the ARIA executor, send `CommandResult` back over the `EventConnection`. The interesting discovery was a timing issue in the event dispatch. The first implementation used an `async` function as the event handler, which wraps every return path — including synchronous throws — in a promise. Tests checking results synchronously after `dispatchEvent()` saw nothing, because the `.then()` and `.catch()` callbacks hadn't fired yet. The fix: a non-async function returning `void | Promise<void>`. Synchronous commands (click, fill, assert) dispatch their results immediately via try/catch. Only `waitFor`, which genuinely polls, returns a Promise.

The queue now sits at Batch 3 — ARIA widget remediation across ~40 blocks-ui components. That's a different kind of work: systematic, repetitive, guided by the audit table in the design spec. The protocol from Batch 1 defines the target state for each component pattern. The command model from Batch 2 will eventually exercise those patterns through scenarios. The three batches form a dependency chain: you can't remediate without knowing the rules, and you can't test remediation without the command infrastructure.
