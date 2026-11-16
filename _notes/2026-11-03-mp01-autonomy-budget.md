---
layout: post
title: "Autonomy Budgets — Why LLM Cost Control Is a Topology Problem"
date: 2026-11-03
entry_type: note
subtype: diary
projects: [casehubio/agency]
tags: [llm, cost-governance, multi-agent, topology, budget]
---

# Autonomy Budgets — Why LLM Cost Control Is a Topology Problem

Multi-agent LLM systems have a cost problem that single-agent wrappers don't. When one agent makes a decision, you control it directly — cap the calls, cap the tokens, done. When a swarm of agents makes independent decisions across a topology, the question stops being "how many calls can this agent make" and becomes "how do we govern a shared, depletable resource across agents with different roles and different importance?"

This isn't a CaseHub-specific problem. Anyone running GPT-4-class models in a multi-agent loop — CrewAI, AutoGen, LangGraph, custom orchestrators — faces the same arithmetic. Five agents making 10 calls each at $0.03 per call is $1.50. Scale that to 50 agents across 100 concurrent case executions and you're burning $150 per minute. The difference between a cost-controlled system and an uncontrolled one isn't a feature — it's whether the product is viable.

I've been designing this for CaseHub's agency layer, and the first-principles answer surprised me. The budget system isn't primarily about cost control. It's about pressure signalling.

## The binary gate is wrong

The existing pattern — `tryAcquire() → boolean` — treats budget as a binary gate. You either get through or you don't. This works for a single agent. For a topology with a coordinator, workers, and coordination actions all drawing from the same pool, binary gating is a blunt instrument. When the pool is 80% consumed, the coordinator's routing decision and a worker's speculative reasoning get the same answer: yes. When it's 100% consumed, they both get: no.

What you actually want is graduated pressure. As the pool depletes, low-priority work should degrade to cheaper models before being denied outright. High-priority coordination work should keep running at full capability until the last drop. The budget system should report *how much pressure the pool is under*, not just whether there's anything left.

![Binary gate vs pressure degradation — same budget, different outcomes](images/budget-degradation-scenario.svg)

The numbers make this concrete. A topology with a 50-call budget running binary gating spends 47 calls at Opus prices, then hits a wall — the coordinator can't even route. The same topology with pressure-based degradation spends 20 calls at Opus, 15 at Haiku (roughly 10x cheaper per token), denies workers while keeping the coordinator alive on reserved capacity for the last 3 calls, and finishes the execution. Same budget, lower cost, better outcome. The coordinator never loses the ability to make decisions.

## Pressure, not permission

The redesigned acquisition returns a `BudgetGrant` — a sealed type with two variants:

```java
public sealed interface BudgetGrant {
    record Granted(BudgetPressure pressure, String correlationId)
        implements BudgetGrant {}
    record Denied(String reason, BudgetPressure pressure)
        implements BudgetGrant {}
}
```

`BudgetPressure` is a single scalar — the worst-case utilisation across calls and tokens — plus the priority tier under which it was computed. A grant at 0.3 pressure means the pool is comfortable. A grant at 0.85 means the routing layer should think about switching to a cheaper model. A denial at 1.0 means the pool is done.

The key insight: the budget system doesn't know about models. It doesn't know about eidos or the manifest. It reports utilisation facts. The routing layer — which already knows about model capabilities, trust scores, and provider health — interprets pressure for model selection. Clean separation. Budget tracks resources; routing makes decisions.

This separation matters because it makes the budget system reusable across any model routing strategy. If you're using OpenAI today and Anthropic tomorrow, nothing in the budget layer changes. If you add a local model tier for low-pressure degradation, nothing in the budget layer changes. The pressure signal is the stable interface.

## Why topology owns priority

I started by reaching for `DecisionScope` as the priority key — it already exists in engine, and FULL_AUTONOMY seemed like a natural marker for "important" actions. Claude pushed back during the design review, and it was right. `DecisionScope` is an access-control lattice. It governs which `DecisionType` values are permitted. A `STIMULUS_DRIVEN` agent responding to a critical signal can be far more budget-important than a `FULL_AUTONOMY` agent doing speculative context mutation. The correlation between "breadth of permitted actions" and "budget priority" is coincidental, not causal.

The right answer: priority is a topology concern. The topology knows which agents are coordinators, supervisors, workers, or peers. It assigns `BudgetPriority` — currently just `RESERVED` and `NORMAL` — when creating each agent's view of the pool.

| Topology | RESERVED | NORMAL |
|----------|----------|--------|
| Centralised | coordinator loops | — |
| Hierarchical | root agent | intermediate + leaf |
| Hive | coordinator | workers |
| Distributed | — | all peers |
| Graduated | — | all agents |

Topologies without a coordinator have no reserved capacity. All budget is shared equally. The reserved partition stays available for coordination actions if the topology adds them later.

This maps cleanly to how real organisations allocate resources. A project manager doesn't get the same budget as an intern, and the difference isn't because managers are "more autonomous" — it's because their decisions affect more people. The topology knows the structure. The topology assigns priority.

## Flat pool, per-agent views

![Budget pool with per-agent views and priority partitioning](images/budget-pressure-flow.svg)

One `AutonomyBudgetPool` per topology execution, with per-agent `AgentBudgetView` instances. The pool tracks two dimensions independently — call count and token count — with pressure as `max(call_utilisation, token_utilisation)`. Views enforce local per-agent limits before drawing from the shared pool, preventing any single agent from starving the others.

The pool has a reserved partition. `NORMAL` callers are denied when the unreserved portion is exhausted. `RESERVED` callers access the full pool. With a default `reserveFraction` of 0.06 on 50 calls, that's 3 reserved calls — enough for critical coordination decisions to survive even when workers have consumed the rest.

Two levels of protection solve two different problems. Per-agent limits prevent the noisy-neighbour problem — one agent making 40 calls out of 50. The reserved partition prevents the starvation problem — workers consuming everything before the coordinator can route. Both are common failure modes in unbudgeted multi-agent systems, and both are invisible until production traffic hits scale.

Dynamic membership falls out naturally. Views are created when agents join, released when they leave. The pool counters are unaffected by view lifecycle — they track total consumption, not per-agent allocation. This composes cleanly with dynamic group management work happening in parallel.

## The enterprise feedback loop

![Enterprise feedback loop — budget to pressure to routing to telemetry](images/budget-feedback-loop.svg)

Every acquisition emits an immutable `BudgetTelemetryEvent.Acquisition` record. Every LLM call completion emits a `BudgetTelemetryEvent.Outcome`. They're paired by correlation ID — the same pattern CaseHub uses for ledger entries and coordination events.

This isn't observability for observability's sake. The paired records tell you: agent X used model Y under Z pressure, consumed N tokens in M milliseconds. Aggregate that across a topology execution and you have a cost profile. Aggregate across an enterprise deployment and you have tuning data.

Was Opus necessary for that routing decision, or would Haiku have sufficed? The telemetry can answer that — and with eidos already routing to different model tiers via the manifest, the path from "this agent consistently produces good outcomes on cheaper models" to "route this agent to cheaper models by default" is a policy update, not a code change.

The trust system feeds the same loop. Agents that earn high trust scores can run on cheaper models even under low pressure — cost optimisation driven by demonstrated competence. Agents with low trust under high pressure get denied rather than degraded, because a bad answer from a cheap model wastes more budget than no answer at all.

For enterprise deployments, this closes the "we're spending too much on LLMs" conversation with data instead of guesswork. The telemetry shows exactly which agents, in which topologies, under which pressure conditions, are consuming the most budget — and whether the outcomes justified the spend.

## Where it lives

Engine is deliberately mechanical — topology, lifecycle, scope enforcement. Budget is about LLM cost control, and LLM execution happens in agency. The entire budget system lives in `agency-core`, package `io.casehub.agency.orchestration.budget`. No engine changes. No eidos SPI changes. Pressure flows through agency's own call path — the decision loop wraps the summariser with a `BudgetAwareSummariser` that acquires budget, delegates, records tokens, and emits telemetry.

This was a conscious choice, and not an obvious one. Agency isn't the only layer that makes LLM calls — neocortex runs models too, for embeddings, retrieval, and memory operations. But neocortex's LLM usage is infrastructure-shaped: fixed cost per operation, predictable, not driven by topology pressure. The budget problem — "five agents competing for a shared pool with different priorities" — is specific to autonomous decision loops. Neocortex doesn't have agents competing. It has a pipeline executing. Different cost profile, different governance model.

Budget could have lived in engine as a general resource governance primitive. The argument against: it would couple the mechanical layer to a concern it doesn't participate in. The argument for agency over a shared module: the pressure abstraction — pool depletion driving model degradation — is meaningful for autonomous agents and meaningless for neocortex's specialised operations. If neocortex needs cost governance, it'll be per-operation rate limiting, not pressure-based pool sharing. Different problem, different solution.

The budget types replace four types that were swarm-specific and binary. The new types are topology-generic and pressure-aware. Pre-release project, no backward compatibility needed. Any topology gets budget for free — you don't need to be running a swarm to benefit from cost governance.

What this opens up: per-agent degradation policies, cost-weighted budgets where cheaper models consume less budget (making degradation self-correcting), and cross-execution budget envelopes at the tenant or case level. None of that is built yet. The pressure infrastructure is the foundation.
