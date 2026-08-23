---
layout: post
title: "Goal-oriented planning in case engines"
date: 2026-08-23
entry_type: article
subtype: diary
projects: [casehub-engine]
tags: [goap, planning, adaptive, case-management, agent-orchestration]
---

Most case engines dispatch workers by trigger condition. A context change fires, the engine evaluates which bindings match, and every matching binding gets dispatched. The order is either declaration order or "everything at once." This works for simple cases. It falls apart when workers have dependencies — when the output of one worker is the input of the next, and firing them in the wrong order wastes work or produces wrong results.

GOAP — Goal-Oriented Action Planning — solves this by planning backward from the desired outcome. Originally from game AI (Jeff Orkin's work on F.E.A.R.), the idea is straightforward: declare what each action requires (preconditions) and what it produces (effects), declare the goal state, and let an A* search find the cheapest sequence of actions that gets from here to there.

We brought GOAP into the casehub engine as a planning strategy — a pluggable component that sits between "which bindings are eligible?" and "which ones actually fire this cycle?" The engine evaluates triggers, builds the eligible set, and hands it to the strategy. GOAP plans with those actions and returns only the next step.

## How the planner decides

Each GOAP action declares three things: preconditions (what must be true for this action to apply), effects (what becomes true after), and cost (how expensive it is). The planner builds a world state from the case context — keys present in the working layer are TRUE, precondition keys not yet present are FALSE — and searches for the cheapest path to a state where all goal conditions are satisfied.

A concrete example. Three workers: `analyse` (no preconditions, produces `analysisResult`), `assess` (requires `analysisResult`, produces `riskAssessment`), and `report` (requires `riskAssessment`, produces `report`). Goal: `report = true`. The planner finds the only viable chain: `analyse → assess → report`. The workers can be declared in any order — `report` first, `analyse` last — and GOAP still dispatches `analyse` first because it's the only action applicable in the initial state.

This is the difference from sequential strategy. Sequential returns the first pending binding. GOAP returns the first binding on the cheapest path to the goal. If there were two routes to `riskAssessment` — a thorough assessment at cost 3.0 and a fast-track check at cost 0.5 — GOAP picks the cheap one. Sequential picks whichever appears first in the definition.

## Adaptive replanning

The Adaptive strategy extends GOAP with per-step replanning. After each worker completes and its output merges into the case context, the strategy replans from the new world state. The plan adapts to what actually happened, not what was predicted.

This matters when workers produce output beyond their declared effects. A worker's `GoapAction` might declare `{analysisResult: true}` as its effect, but the worker function itself also writes `{lowRisk: true}` to the context at runtime. The planner didn't know about `lowRisk` when it computed the initial plan. But after the worker completes, the replanning cycle sees `lowRisk = true` in the world state. If a `fastTrack` action exists that requires `lowRisk` and costs 0.5 instead of the 3.0 standard assessment, the replanner picks it.

The plan was one step behind reality. The replan caught up.

## What GOAP filters out

The most interesting property is what GOAP *doesn't* dispatch. Add a fourth worker — a decoy that produces `{irrelevantData: true}`, costs almost nothing (0.1), and is eligible by trigger condition. Sequential strategy would fire it. Choreography would fire everything. GOAP never touches it, because `irrelevantData` doesn't appear on any path to `{report: true}`. The A* search never reaches the decoy action. It's invisible to the goal.

This is GOAP earning its keep in a multi-agent system. In a case with twenty available workers, only the subset that contributes to the current goal gets dispatched. The rest sit idle — eligible but not useful right now.

## Two layers of gating

Building the integration tests surfaced a design constraint worth naming. The engine has two independent gating mechanisms, and they serve different purposes:

**Trigger conditions** gate whether the engine evaluates a binding at all. They're event-driven: "fire when `.transaction` appears in the context." A binding whose trigger isn't satisfied doesn't enter the eligible set, period.

**GOAP preconditions** gate whether an action is applicable in the planner's search. They're state-based: "this action requires `analysisResult = true` in the world state to be applicable."

Conflating them — using trigger conditions to enforce dependency ordering between GOAP actions — creates a partial-eligibility problem. If only the first action's trigger is satisfied, GOAP sees one action, can't reach the goal, and returns an empty plan. The case stalls.

The working pattern: all GOAP bindings share a broad trigger (they're all eligible from the start), and the precondition/effect declarations handle the ordering. GOAP sees the full action space and plans the path. After each step completes and changes the world state, the next cycle's planner picks the next step — because preconditions that were unsatisfied before are now met.

Triggers gate *whether*. Preconditions gate *when*. Keep them separate.
