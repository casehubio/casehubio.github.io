---
layout: post
title: "casehub-life — The Two Modes"
date: 2026-06-28
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [openclaw, provisioner, sentinel, architecture]
series: issue-37-layer7-worker-provisioner
---

## What I was trying to achieve: complete Layer 7 — real OpenClaw integration with the full skill ecosystem

casehub-life had 32 AgentExec workers calling OpenClaw's `/hooks/agent` endpoint for single-shot LLM tasks — book appointments, get quotes, record decisions. They worked. But they were scattered: 62 string literals across 7 CaseHub classes defining agent identity, with 7 boilerplate descriptor methods copy-pasting the same 18-argument constructor. And the bigger question — how do agents monitor things persistently, not just respond when asked — was still open.

Two issues on one branch: life#46 (consolidate agent identity) and life#37 (wire the WorkerProvisioner for heartbeat monitoring).

## What I believed going in: that heartbeat mode would use OpenClaw's provisioner infrastructure

The original issue described a straightforward wiring job — un-exclude the openclaw-casehub CDI beans, activate the reactive provisioner, let agents operate autonomously via channels. The assumption was that OpenClaw's existing provisioner infrastructure (agent registry, channel backend, worker status listener) was designed for exactly this. I expected to plug it in, configure capabilities, and have persistent agents.

## The identity problem was the easy one

life#46 took a morning. The 62 scattered string literals — agent IDs like `"openclaw:health-agent@1"` in descriptor constructors, `"health-agent"` in 32 `forAgent()` calls, `"GB"` hardcoded 7 times as jurisdiction — collapsed into a `LifeAgent` enum with 4 constants and a `LifeAgentDescriptorFactory` CDI bean. The factory injects `tenancyId` and `jurisdiction` from config, so 7 CaseHubs no longer need those injections at all.

The interesting decision was putting `descriptor()` on a factory rather than the enum. The enum carries pure identity data. The factory owns config-to-descriptor construction — tenancyId, jurisdiction, and eventually vocabulary URIs and capabilities when those become populated. The enum stays a data holder. The factory becomes the extension point.

The review caught briefing drift I'd missed: three CaseHubs sharing the HEALTH agent had divergent briefings ("Health domain booking and follow-up agent" vs "Health domain agent"). The consolidation forced a choice — "Health domain coordination agent" covers all three accurately.

## Three verified findings that broke the original architecture

life#37 started as "wire the provisioner" and ended as "design a completely different sentinel architecture." Three findings, each verified through engine bytecode analysis, forced the pivots.

**The bridge drops STATUS.** I'd designed heartbeat result delivery through Qhorus channels — the sentinel posts a STATUS message, the `QhorusMessageSignalBridge` routes it to case context, bindings react. Clean, standard, uses existing infrastructure. Wrong. The bridge's `isCommitmentResolving()` whitelist accepts only RESPONSE, DONE, DECLINE, and FAILURE. STATUS is silently dropped. No error, no warning. Every heartbeat result would have vanished.

The fix was to bypass the channel stack entirely: `CaseHubRuntime.signal(caseId, "sentinelReport", result)` — a public engine API already used by `LifeCaseService.startCase()`. Direct, typed, no intermediary.

**The registry overwrites silently.** `OpenClawAgentRegistry.register()` uses `put()`, not `putIfAbsent()`. If two health-domain cases run concurrently (an appointment cycle and a care coordination case), the second registration silently drops the first. Case 1's sentinel loses its agent association — the heartbeat keeps firing, but results can't route back.

The fix was to not use the openclaw registry at all. `LifeSentinelRegistry` keys by `(caseId, capabilityName)` — multiple cases with the same agent type coexist without conflict.

**The engine never calls terminate().** The `ReactiveWorkerProvisioner` SPI defines `terminate(workerId, tenancyId)`. No call site exists in the engine runtime — verified through bytecode analysis. The method is defined. The no-op implementations exist. Nobody invokes them. Provisioned resources are never cleaned up by the engine.

The fix: `LifeProvisionerCleanupObserver` watches `CaseLifecycleEvent` for terminal states (CaseCompleted, CaseFaulted, CaseCancelled) and calls `terminateAllForCase()`. The engine should own this lifecycle — filed as a follow-on issue.

## What the sentinel architecture actually looks like

The engine's execution flow handles mode selection naturally. A binding fires. If an inline worker matches the capability, AgentExec runs it synchronously — single LLM call, structured result, case context updated directly. If no inline worker matches, the engine falls through to `tryProvision()`. The provisioner registers the sentinel and schedules a heartbeat.

Each heartbeat tick is the same proven path the 32 existing workers use: `Agent.execute()` via `DirectCallBridge` to `/hooks/agent`, structured response via response schema, result delivered via `CaseHubRuntime.signal()`. The sentinel gets fresh case context each tick via `CaseHubRuntime.query()` — contractor name, job details, current status — and uses OpenClaw's skills for independent real-world verification.

Seven sentinel capabilities, one per case plan. The contractor sentinel monitors job progress every 4 hours. The anomaly sentinel scans transactions daily. The follow-up sentinel checks prescription collection and referral bookings every 12 hours. Each has a per-domain response schema — `ContractorSentinelReport` with progress percentage and status, `AnomalySentinelReport` with anomaly list and severity, `FollowUpSentinelReport` with pending actions and days overdue.

The re-firing problem was the subtlest design challenge. The sentinel binding condition (`.contractorRequest != null`) stays true for the case's lifetime. Every context change re-evaluates it, and the engine calls `tryProvision()` each time. The idempotency guard in `LifeSentinelRegistry.isProvisioned()` turns all repeat calls into O(1) no-ops. The first call provisions; every subsequent call returns `ProvisionResult.empty()` immediately.

## Where this leaves us

Zero new CDI bean activations. Zero openclaw-casehub beans un-excluded. The sentinel architecture reuses the DirectCallBridge and `LifeOpenClawChatModelFactory` already proven by 32 AgentExec workers. The only new dependencies are `CaseHubRuntime` (already used by `LifeCaseService`) and Quartz (already available via `casehub-engine-scheduler-quartz`). The sentinel infrastructure lives entirely in life-owned code.

Layer 7 is complete. The final review flagged missing license headers on new files — a mechanical fix for the next session. The architectural shape is settled: AgentExec for request/response, provisioner for persistent monitoring, same agent infrastructure underneath both.
