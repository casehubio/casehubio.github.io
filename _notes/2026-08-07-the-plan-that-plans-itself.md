---
title: "The Plan That Plans Itself"
date: 2026-08-07
author: Mark Proctor
projects: [casehub-engine]
tags: [hierarchical-planning, goal-decomposition, htn, llm, architecture]
entry_type: note
subtype: diary
status: draft
---

# The Plan That Plans Itself

The DecompositionStrategy SPI has been sitting in engine-api since blocks#60 — a full type hierarchy with `TaskNode`, `CompoundTask`, `LeafTask`, guard-gated methods, the works. Wired into `EngineStrategyResolver`. Referenced from `CaseDefinition.decompositionStrategy`. And nothing called it. Zero production implementations. A perfectly shaped hole waiting for someone to fill it.

Today we filled it.

## What the SPI already had — and what it didn't

The HTN decomposition infrastructure was promoted from blocks during the unified execution model work. `DecompositionStrategy<T>` takes a `CompoundTask` and returns a `DagPlan<LeafTask<T>>`. The types are clean. The strategy resolver picks them up by ID. But there was no bridge from agent goals to decomposition — no code that said "this agent has a goal called `comprehensive-analysis` targeting capabilities `data-gathering`, `analysis`, and `reporting`; figure out the order."

That bridge is `GoalDecomposer`. At case start, it collects active goals from each agent's descriptor, builds a `CompoundTask` per goal, calls the decomposition strategy, and materializes the result as compound `PlanItemDefinition`s. The existing dispatch machinery — `CompoundLifecycleEvaluator`, `CompoundStrategyDispatcher`, CHOREOGRAPHED dispatch — handles the rest. No new dispatch infrastructure needed.

## The module boundary surprise

The plan said GoalDecomposer should live in the runtime module alongside `CaseStartedEventHandler`. It can't. Runtime doesn't depend on planning — the dependency goes the other way. This isn't obvious from the package names, and I only discovered it when IntelliJ reported "package does not exist" for every planning import.

The fix follows a pattern already established in the codebase: define an SPI interface in `common/spi/` (where both modules can see it), implement it in planning as `DefaultGoalDecomposer`, and inject via `Instance<GoalDecomposer>` in the runtime handler. `WorkOrchestrator` uses the same approach. The `Instance<>` guard means the decomposer is transparent no-op when the planning module isn't on the classpath.

## What the LLM actually decides

The `LlmDecompositionStrategy` has a narrow job: given a goal description and a list of available capabilities, determine the ordering. It doesn't invent capabilities (must reference existing ones), doesn't select workers (routing handles that), doesn't set input projections (bindings handle that). The prompt is specific: "produce an ordered sequence of sub-steps, each referencing exactly one capability by name."

In v1, plans must be linear chains. Parallel branches — where the DagPlan can express `A → [B, C] → D` — are rejected at validation time. CHOREOGRAPHED dispatch is strictly sequential, and wiring parallel-aware dispatch is future work. This was a deliberate scope cut surfaced by the design review.

## Six things the review caught

The light design review (coherence + structure + robustness + cross-cutting) converged on six issues that all three dimensions independently identified:

1. **CHOREOGRAPHED can't express parallelism** — the DagPlan supports parallel branches but the dispatch mode doesn't. v1 restricts to sequential.
2. **Empty DagPlan is impossible** — `DagPlan.fromNodes()` rejects empty node lists. The spec said "return empty plan" as an error path. Check must happen before construction.
3. **`addCompound()` was redundant** — `DefaultCasePlanModel` already has `registerDefinition()` which handles compound registration recursively.
4. **Recovery was underspecified** — CasePlanModel is in-memory. After JVM restart, the compound parent/child tree is lost. Solution: rebuild from PlanItem metadata + idempotency guard.
5. **No timeout on LLM calls** — case start could hang. Added per-goal timeout with graceful degradation.
6. **Overlapping compound scopes** — two goals referencing the same binding. Resolved by goal priority.

Strong convergence across independent reviewers. The cross-cutting pass connected the dots — the recovery fix paths from the three dimensions were contradictory, and the cross-cutting reviewer identified which one was actually correct.

## The gap that's harder to ignore

After implementation, I asked a question that's been nagging: how much of the recent feature surface is covered by examples a developer could learn from? The honest answer is none. Composable routing signals, JPAF personality, reflection, memory retrieval, goal abandonment, lifecycle scopes, action risk classification, CBR, humanTask routing, and now hierarchical planning — all documented as infrastructure internals in CLAUDE.md, none demonstrated in a worked example showing a real domain problem.

Filed #877 with a full gap analysis. Ten feature areas, every one with zero worked examples. The consumer guide lists SPIs and types but never answers "how do I build an agent that learns from experience and plans its work?" That question should have a concrete answer before we add more features.
