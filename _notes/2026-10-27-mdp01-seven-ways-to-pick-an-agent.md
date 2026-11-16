---
layout: post
title: "Seven Ways to Pick an Agent"
date: 2026-10-27
entry_type: note
subtype: diary
projects: [casehubio/casehub-mosaic]
tags: [typescript, web-components, agency, routing, visualization]
---

# Seven Ways to Pick an Agent

CaseHub's orchestration framework has seven routing strategies — first-match, round-robin, sequential, select-all, llm-selected, cbr-agent, disposition-aware — each with fundamentally different selection semantics. A first-match evaluates a guard predicate down a candidate list and stops at the first hit. A round-robin cycles through agents in sequence. A select-all fans out to everyone simultaneously. An llm-selected hands the decision to a language model. These aren't variations on a theme; they're different mechanisms.

The mosaic codebase already had a runtime view for routing — `routing-rationale` shows trust scores, workload factors, candidate phases, and the final selection after it happens. What it didn't have was a structural view showing what routing strategy is *configured* and how it *works*. The supervisor-view rendered routing as a single teal badge with the type name. The saga and decomposition views showed it as a metadata card. Choreography ignored it entirely.

I wanted a component that made the selection mechanism visible. A round-robin should look like a cycle. A first-match should look like a filter. A select-all should look like a fan-out. A uniform table — the easy path — would obscure the very thing that makes each strategy distinct.

The design has two layers. The component itself (`agency-routing-strategy-view`) is a property-driven Lit element that accepts a `RoutingSpec`, an agent list, optional governance config, and an optional runtime status map. It doesn't use DataSourceMixin — the composing view provides everything. The rendering layer is a registry: a `Map<string, RoutingRendererFn>` where each strategy type maps to a pure function that returns a `TemplateResult`. All seven built-in strategies are pre-registered. Unknown types fall back to a generic badge-plus-grid layout.

The registry exists because `RoutingSpec` is extensible — unlike `PatternCategory` (a closed union with `default: never` guards in the dispatch), routing strategies can be added by downstream apps without forking mosaic. The `registerRoutingRenderer` function mirrors the existing `registerStatus` pattern from mosaic-core's StatusBadge. A downstream app adds `registerRoutingRenderer('custom-priority', myRenderer)` and the component renders it without modification.

Each renderer is a standalone function — no class, no mixin, no inheritance. The first-match renderer shows a guard expression badge above a numbered candidate list with arrows. Round-robin shows a cycling indicator with position counters. Sequential shows a numbered chain with a "one-shot" badge. Select-all renders a central "dispatch all" node with fan-out arrows. LLM-selected shows a decision node with model and prompt badges. CBR-agent shows a case-based reasoning node with a similarity threshold. Disposition-aware renders horizontal weight bars across personality axes — openness, conscientiousness, analytical, risk-tolerance — using the pink social colour rather than the peach orchestration default, because disposition scoring bridges the social and orchestration domains.

The governance tie-in was a deliberate scope expansion. `GovernanceSpec` has `trustRouting` (threshold, borderline margin, blend factor) and `cbrRouting` (success/failure/gate weights) — config that directly shapes agent selection but lives outside `RoutingSpec`. Showing routing strategy without the policy that governs it gives an incomplete picture. When the component receives governance config, it renders a "Routing Policy" section below the strategy layout with the same visual vocabulary as routing-rationale: threshold bars, blend factor badges, weight distributions. The reader sees intent and policy in one place.

The runtime overlay is intentionally lightweight: a `Record<string, RoutingAgentStatus>` mapping agent names to `'selected' | 'skipped' | 'not-evaluated' | 'excluded'`. Agent cards get colored left borders — green for selected, amber for skipped, grey for not-evaluated, red for excluded. No scores, no reasons, no detail. That's routing-rationale's job. The two components are complementary: routing-strategy-view shows what's configured and how it selects; routing-rationale shows what actually happened and why.

The integration replaced inline routing rendering across all four orchestration views. Supervisor-view dropped its bespoke `.routing-badge` CSS class and hand-rolled badge span. Decomposition and saga views dropped their metadata cards. Choreography gained routing display where it previously had none. Each view passes `spec.routing`, `spec.agents`, and `spec.governance` as properties — the same three-line composition pattern everywhere.
