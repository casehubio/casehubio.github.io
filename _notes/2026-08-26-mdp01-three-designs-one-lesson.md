---
layout: post
title: "Three designs, one lesson — check what already exists"
date: 2026-08-26
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [routing, trust, eidos, architecture, platform-coherence]
series: issue-401-reputation-routing
---

I started with a clean-sheet design for reputation-aware routing in qhorus. An agent dispatches a COMMAND with `target: "role:analyst"` and the system picks the best agent for the job based on trust scores. Straightforward requirement. I designed a `CapabilityRouter` with three strategies (HIGHEST_TRUST, ROUND_ROBIN, RANDOM), per-channel configuration, trust score thresholds, and a `@DefaultBean` fallback.

Then I checked the platform repo.

### The first surprise: routing already existed

The platform's `docs/platform/routing.md` documents a four-layer routing architecture that's been built and tested. `AgentRoutingStrategy` SPI in `casehub-api`. `ComposableAgentRoutingStrategy` in the engine that blends trust scores, semantic similarity, LLM reasoning, and CBR evidence with configurable weights. `TrustCandidateClassifier` in the ledger with a four-phase maturity model (BOOTSTRAP, BORDERLINE, QUALIFIED, EXCLUDED). None of this is theoretical — it has implementations, tests, and production callers in `DefaultWorkOrchestrator` and `CaseContextChangedEventHandler`.

My three-strategy enum was a strict subset of what the platform already provided. Worse, it would have diverged over time — a parallel routing system that couldn't benefit from improvements to the platform's trust maturity model.

### The second surprise: eidos already had capability matching

The dependency constraint made the first surprise harder to use directly. Qhorus can't depend on `casehub-api` (the engine API) — that would create a dependency from the communication mesh to the orchestration engine. The architecture is clear: qhorus is independent.

But eidos — the agent identity layer — already had `CapabilityResolver` with vocabulary-aware matching, `AgentRegistry.find(AgentQuery.byCapability())` returning ranked `AgentMatch` results, and `MatchDegree` with semantic matching support. The capability discovery infrastructure was built. What was missing was the trust-scored selection step: given these capability matches, pick the best one using trust scores.

That gap belonged in eidos, not qhorus. We added `AgentSelector` as an SPI in `eidos-api` with `SimpleAgentSelector` as the `@DefaultBean` in eidos-runtime — it filters by `CapabilityHealth`, scores via `TrustScoreSource`, filters by threshold, and picks the highest scorer.

### What qhorus actually needed

The final design is thin. `RoutingBridge` in qhorus's dispatch pipeline detects `role:` prefixed targets, calls `AgentRegistry.find()` to discover capable agents, calls `AgentSelector.select()` to pick the best one, and replaces the target with the resolved instanceId. The message is persisted with the specific agent. The commitment tracks that agent. The ledger records what the original target was, who was selected, and why.

No custom router. No strategy enum. No parallel trust model. The routing infrastructure exists in the platform. The capability matching exists in eidos. Qhorus consumes both through their SPIs — the same pattern as `CredentialResolver` from `casehub-platform-api`.

### The dependency pattern

The insight that made this work: qhorus already depends on `casehub-platform-api` (for `CredentialResolver`, `ActorType`) and `casehub-ledger-api` (for `CommitmentAttestationPolicy`, `TrustScoreSource`). Adding `casehub-eidos-api` follows the same pattern — a thin API module with SPI interfaces, consumed by qhorus, implemented in a runtime module that lives elsewhere. The dependency direction is always downward: qhorus depends on platform APIs, never on engine internals.

When eidos isn't on the classpath, `RoutingBridge` uses `Instance<>` injection. If `AgentRegistry` or `AgentSelector` aren't resolvable, routing is a no-op — `role:analyst` passes through as a literal string, exactly as it did before. Zero overhead, zero breakage.

### The meta-lesson

I designed three iterations of this feature before arriving at the one that required the least new code. The first iteration (custom `CapabilityRouter` with three strategies) would have been roughly 400 lines of routing logic. The second (bridge to platform `AgentRoutingStrategy`) would have been 200 lines of type adaptation. The final version — consuming eidos's existing capability matching and selection — is about 80 lines in `RoutingBridge` plus a data model change.

The reduction wasn't because the problem got simpler. It was because each iteration discovered more of what already existed. The lesson: before designing infrastructure, audit what the platform already provides. Not just in your repo — in the siblings. The capability you're about to build might already be someone else's SPI.
