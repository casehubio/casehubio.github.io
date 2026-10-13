---
layout: post
title: "When the planner can't see ahead"
date: 2026-08-22
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [goap, planning, integration-testing, dispatch-pipeline]
---

# When the planner can't see ahead
The GOAP planning strategies had comprehensive unit tests — mock the `CaseDefinition`, mock the `CaseContext`, assert the planner returns the right binding. What they didn't have was a single test exercising the real dispatch pipeline. A case starts, context changes fire, Quartz picks up the job, the worker runs, output merges into the context, and the next planning cycle evaluates from the new world state. The unit tests proved the planner's logic. They said nothing about whether the planner actually gets called with the right inputs when embedded in the engine.

Writing those integration tests surfaced something I hadn't thought about.

## The partial-eligibility problem

GOAP plans by searching for a path from the current world state to a goal state using the available actions. The engine determines "available" by intersecting GOAP actions with currently *eligible* bindings — those whose trigger conditions are satisfied right now.

This creates a tension. If I design a case where each binding's trigger depends on the previous step's output (`.analysisResult == true` gates the assess binding, `.riskAssessment == true` gates the report binding), then at case start only the first binding is eligible. GOAP sees one action, can't reach the goal with just that action, and returns an empty plan. Nothing fires. The case stalls.

Sequential strategy doesn't have this problem — it just returns the first pending binding regardless of goal reachability. GOAP needs to see enough of the action space to find a path.

The working pattern is all bindings sharing a broad trigger (`.trigger == true`) while GOAP preconditions and effects encode the dependency ordering. All bindings are eligible from the start; GOAP selects which one to fire based on precondition satisfaction in the current world state. After the first worker completes and its output changes the world state, the next planning cycle picks the next step in the chain — because the preconditions that were unsatisfied before are now met.

This is actually the right design. Trigger conditions gate *whether the engine evaluates a binding at all*. GOAP preconditions gate *whether an action is applicable in the planner's search*. Conflating the two — using trigger conditions as GOAP guards — creates the partial-eligibility gap.

## Declared effects vs runtime output

A related subtlety: GOAP plans from declared effects, not from runtime output. If a worker's `GoapAction` declares effects `{analysisResult: true}` but the worker function also produces `{lowRisk: true}` at runtime, the planner doesn't know about `lowRisk` when computing the initial plan. The planner picks the path that doesn't require `lowRisk` — even if a cheaper alternative exists that does.

This is correct for *initial* planning. But since both GOAP and Adaptive strategies replan from the current world state on each dispatch cycle, the runtime-produced keys become visible on the next cycle. The planner sees `lowRisk=true` in the world state and picks the cheaper `fastTrack` action that requires it. The plan adapts to reality — just one step behind.

The Adaptive strategy test exercises exactly this: `analyse` declares `{analysisResult: true}` as its effect but also produces `lowRisk: true` at runtime. The initial plan routes through the expensive `assess` path. After `analyse` completes and the replan fires, `fastTrack` (cost 0.5) beats `assess` (cost 3.0). The execution order is `analyse → fastTrack → report`.

## What GOAP actually filters

One test I'm happy with: the decoy binding. Four bindings, all eligible, but one produces `{irrelevantData: true}` — effects that contribute nothing toward the goal. GOAP's A* search never reaches that action because it doesn't appear on any path to the goal state. The decoy worker never fires, despite being eligible by trigger condition, having the cheapest cost (0.1), and having a PlanItem created in PENDING state.

This is GOAP earning its keep. Sequential strategy would fire the decoy. Choreography would fire everything. GOAP fires only what contributes to the goal.
