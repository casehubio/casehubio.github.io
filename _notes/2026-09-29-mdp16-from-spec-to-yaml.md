---
title: "From Spec to YAML"
date: 2026-09-29
author: mdp
entry_type: note
subtype: diary
projects: [casehubio/fsitrading]
series: issue-50-c13-trading-yaml-playbooks
tags: [yaml-core, playbooks, simulation, cbr, implementation]
---

# From Spec to YAML

Continues from [Three Tiers of Response](2026-09-28-mdp15-three-tiers-of-response.md).

The design spec said `DemoSpi` — the simulation framework's connector interface that apps implement to inject scenario events. Sensible name, clean contract. I went looking for it in the pages codebase. Not there. Not in any jar. Not in any index. The spec was referencing an interface that doesn't exist yet.

This is the gap between designing against a platform and building against one. The design session had the right concept — a simulation connector that accepts temporal profile events and dispatches them into the market pipeline — but pinned it to an interface that was still on someone's backlog. The fix was straightforward: build `FsiSimulationConnector` as a standalone CDI bean with a `dispatch(action, data)` method and a `target()` identifier. When `DemoSpi` eventually ships in pages, the migration is a one-line `implements` clause. No architectural change, just a contract formalisation.

What I found more interesting was the simulation infrastructure that *does* exist. The platform's `SimulationDecoratorProcessor` generates CDI decorators at compile time from `@SimulationEligible` annotations. The generated code intercepts SPI calls and, when a simulation strategy is registered, returns corpus data instead of calling the real implementation. The corpus configuration schema supports five strategy types — key-lookup, sequential, random, recorded-replay, nearest-match — each with its own resolution semantics. The temporal profile system layers on top: timed event sequences with speed multipliers and loop support. All driven by a single `simulation.yaml`.

So the simulation layer for fsitrading became two things. Layer 1: five corpus YAML files — canned responses for each `@SimulationEligible` SPI (market data snapshots by instrument, risk assessments by instrument-scenario pair, order fills in a sequential cycle, strategy evaluations by instrument-regime pair, agent responses by prompt category). Layer 2: four temporal profiles that script market sessions — normal trading day with U-shaped volume, flash crash with liquidity withdrawal and recovery, gradual regime shift from trending to mean-reverting, overnight gap with morning recovery. Each profile fires events through `FsiSimulationConnector` into the l0Bus and CDI event system.

The CBR generalisation went cleaner than I expected. The existing observers were hardcoded to `CASE_TYPE = "overnight-incident"` — four of five playbook case types would have been invisible to the feedback loop. I extracted an `FsiCbrFeatureExtractor` SPI interface in api/ with `caseType()` and `extractFromSnapshot()`, built a CDI registry that discovers all implementations via `Instance<T>`, and created five per-case-type extractors. The observers now delegate to the registry and silently skip unknown case types. The original `FsiFeatureExtractor` got the SPI interface and a `caseType()` method without renaming — the plan wanted `OvernightIncidentFeatureExtractor`, but the existing name is used in enough places that the rename would have been pure churn.

The playbooks themselves were the largest batch but the most mechanical. Each one is a state machine or coordination pattern expressed in yaml-core notation — YAML that validates structurally against the step catalog without needing the engine runtime. Flash crash response: DETECTED → RESPONDING → MONITORING → RESOLVED with a 120-second deadline that falls back to ESCALATED. Strategy evaluation cycle: forEach instruments in watchlist, parallel evaluate-strategy with quorum consensus, risk gate, submit-order. Overnight incident: SLA-driven with per-severity deadlines and graceful triage degradation. Risk escalation: CRITICAL/HIGH match with semaphore lifecycle. Market regime shift: per-regime parameter adaptation with convergence monitoring.

Two shared modules extract the patterns that appear across multiple playbooks. `risk-gate` composes assess-risk → trigger-risk-gate → notify-risk-desk-on-rejection. `parallel-assessment` dispatches the same agent step in parallel with different context strings for multi-perspective situational assessment. Every playbook that needs a risk gate imports the module instead of reimplementing the three-step sequence.

The resolution documents are the third tier — prose guidance for situations where no automated playbook applies. Counterparty defaults, regulatory inquiries, unprecedented events, system failures, margin calls. Each document follows the `ResolutionGuideInput` contract: problem description, solution approach, ordered action steps with automation hints, and feature maps for CBR similarity retrieval. `FsiResolutionGuideAdapter` implements `CorpusSourceAdapter` and discovers all five from the classpath.

The full C13 epic is now structurally complete — 12 tasks across 6 batches, every step definition resolving, every playbook parsing, every corpus file loading. What remains is engine work: `StepFileCallableDispatcher` for case→playbook runtime dispatch, `PlaybookStateMachineExecutor` for state machine execution, and step-level CBR outcome recording. That's cross-repo work tracked against casehubio/engine. fsitrading delivers validated YAML; the engine delivers the runtime that executes it.
