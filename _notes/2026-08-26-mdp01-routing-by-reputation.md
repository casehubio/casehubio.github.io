---
layout: post
title: "Teaching Qhorus to Route by Reputation"
date: 2026-08-26
entry_type: note
subtype: diary
projects: [casehubio/qhorus]
tags: [qhorus, routing, eidos, trust, dispatch-pipeline]
series: issue-401-reputation-routing
---

# Teaching Qhorus to Route by Reputation

When a COMMAND targets `role:analyst` instead of a specific agent, nothing happens — the string passes through as a literal. The sending agent has to know which instance handles analytics, leaking orchestration logic into the agent layer.

The eidos project already solved the selection problem. `AgentRegistry` finds agents by capability. `AgentSelector` picks the best one. The scores and match data flow through a clean sealed interface — `Selected`, `NoneQualified`, `Escalated`. We built a bridge in the dispatch pipeline that detects `role:` prefixes and delegates to these SPIs.

The bridge sits between the rate limiter and the obligor trust gate — early enough to resolve the target before commitment tracking opens, late enough that ACL enforcement has already run. Non-role targets bypass it entirely; there's no overhead for the common case.

One design choice worth calling out: the eidos dependency is injected as `Instance<AgentRegistry>` and `Instance<AgentSelector>`. When eidos isn't on the classpath — standalone qhorus without the engine — the `Instance` is unresolvable and routing silently becomes a no-op. No `@DefaultBean` fallback strategy, no error, just pass-through. This keeps qhorus embeddable without forcing consumers to pull in the full agent infrastructure.

Per-channel trust thresholds let operators tighten routing for sensitive channels. A channel with `routingTrustThreshold: 0.7` rejects any candidate below that score, even if the selector would have picked them. The global default is zero — no filtering unless you ask for it.

The ledger records four routing metadata fields on every routed message: original target, selected agent, strategy name, candidate count. This makes routing decisions auditable without separate logging — the existing `list_ledger_entries` query surface picks them up for free.

Claude caught one thing during the review that I'd missed: `RoutingRejectedException` extended `RuntimeException` instead of `IllegalStateException`. The `@WrapBusinessError` annotation on `QhorusMcpTools` only catches `IllegalArgumentException` and `IllegalStateException` — so a routing failure from an MCP tool would have surfaced as a raw exception rather than a structured error response. A one-line fix, but the kind of thing that only shows up when you trace the exception through the full call chain.

The diagnostic tool (`get_routing_candidates`) runs the same registry lookup and selection without dispatching. It shows every matching agent, their trust score from `TrustGateService`, whether they pass the threshold, and which one the selector would pick. Useful for debugging "why did my command go to agent X instead of Y" without having to read ledger entries after the fact.

What this opens up is interesting. Capability-based addressing means agents can publish what they do, not who they are. An orchestrator sends `role:analyst` and the mesh resolves it — today by trust score, eventually by whatever signal blend the deployment configures. The agent that handles the request might be different tomorrow if a higher-scoring one registers. The sender never knows or cares.
