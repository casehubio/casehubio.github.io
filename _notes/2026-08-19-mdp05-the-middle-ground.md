---
layout: post
title: "The Middle Ground"
date: 2026-08-19
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [planning, adaptation, reflexion, portfolio-planning, decomposition]
series: issue-927-adaptive-planning-intelligence
---

# The Middle Ground

Continues the [adaptive planning intelligence](2026-08-18-mdp04-when-unknown-means-yes.md) series.

The Phase A foundations — ternary GOAP, failure taxonomy, expectation tracking — gave the engine three measurement axes. Phase B is about using those measurements to make better decisions instead of brute-force ones.

The simplest example: adaptation triggers. `EveryStepTrigger` calls the LLM after every worker completion — ten steps, ten LLM calls, even when the plan is executing perfectly. `OnFailureTrigger` only fires on failure, but misses context drift — a worker can succeed and still produce output that invalidates three downstream steps. The middle ground is measuring how far reality has diverged from the plan and only adapting when that divergence crosses a threshold.

`ProgressGatedTrigger` queries the expectation validation metadata that #928 writes to EventLog entries after every worker completion. `DivergenceScoreComputer` computes a windowed average of those divergence ratios, filtered by adaptation generation (so a freshly adapted plan starts with a clean baseline). If the score exceeds the configured threshold — default 0.3 — adaptation fires. Below that, the plan keeps executing undisturbed.

The per-binding override was an idea borrowed from Embabel's `replanAlways()` and `replanWhen()` decorators. Some bindings inherently produce unpredictable output — an LLM-backed analysis step, a web scraper — and should always trigger replanning regardless of measured divergence. Others are deterministic transformations that never warrant it. `ReplanHint` (ALWAYS, CONDITIONAL, NEVER) on each binding declaration gives the case author that control without touching global thresholds.

The Reflexion work was smaller in scope but addresses a gap that kept bothering me. When a worker fails and the engine reroutes to a different agent, the new agent gets the same input with no context about what went wrong. Reflexion's insight — from Shinn et al.'s NeurIPS 2023 paper — is that a verbal critique stored before retry outperforms scalar reward signals. "Step failed with status FAULTED" is a scalar signal. "Entity resolution failed because the input lacked the required identifier format" is verbal critique that an LLM-backed revision strategy can reason about.

`FailureCritiqueService` generates that critique — LLM-powered for Knowledge failures when a ChatModelProvider is available, falling back to the classification reason string otherwise. The critique stores in `_diagnostics` alongside the existing failure history, flows to rerouted agents via `WorkerContext.failureDiagnoses` (the field was already there from #930), and `ForwardReplanRevision` reads it from the working layer to enrich its prompt. No new threading required — the existing plumbing carried it.

The portfolio strategy is the one I'm most curious to see in production. IPC competition results have shown for years that portfolio planners — try fast classical first, escalate to expensive methods only when needed — consistently outperform single-planner approaches. `PortfolioDecompositionStrategy` runs delegates in sequence with per-delegate time budgets. GOAP gets one second; if it produces a valid plan, that's the answer. If not (insufficient precondition/effect annotations, unsatisfiable goal state), the LLM gets thirty seconds. Each delegate runs on a virtual thread with `Future.get(timeout)` for cancellation.

The default cascade — `["goap", "llm"]` — means well-specified case definitions get classical decomposition in milliseconds with zero token cost. Under-specified ones fall through to the LLM. And if the LLM is down, rate-limited, or slow, GOAP still handles the cases it can. The failure handling is deliberately broad: any exception from a delegate cascades, whether it's `AgentException` (GOAP found no plan), `TimeoutException`, or something unexpected. The portfolio's value proposition is resilience, and swallowing exceptions for cascading serves that.

Batch 3 is where these pieces start composing. Learned action costs from CBR traces (#937) feeds real execution data back into the GOAP planner — actions that historically fail get higher costs, naturally steering the planner toward paths that work. Dynamic decomposition depth (#936) adjusts how deeply the engine decomposes before dispatching. And meta-reasoning (#934) — persist, refine, or concede — is where the failure taxonomy, the divergence score, and the critique all converge into a single decision framework.
