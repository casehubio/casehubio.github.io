---
layout: post
title: "The Case Definition Is the Diagram"
date: 2026-10-30
entry_type: note
subtype: diary
projects: [casehubio/casehub-mosaic]
tags: [mosaic, diagrams, case-definition, yaml, agent-orchestration]
---

# The Case Definition Is the Diagram

A case definition in CaseHub is a YAML file that describes everything an agent system does. Which agents exist, what capabilities they have, what triggers them, how they coordinate, what happens when something fails. For a simple case — two agents, one binding, sequential execution — the YAML is readable. For a real one — eight workers across four function types, twelve bindings with conditional triggers, parallel extraction lanes feeding into a risk assessor with match-case routing — the YAML is correct but opaque. You can't see the topology.

This is the core argument for visual case editing: YAML is the source of truth, but topology is what you reason about. When someone asks "what happens when the identity extraction fails?" the answer is a path through the graph, not a line number in a YAML file. When someone asks "which agents can the risk assessor reach?" the answer is a set of edges, not a grep through `capabilities:` arrays.

## The Drill-Down Model

A case definition is a hierarchy of concerns, and the diagram reflects this. Each level is a different diagram type. Clicking a node drills into the next level.

![Case definition drill-down: overview → execution model → worker detail → binding detail](images/case-drill-down-levels.svg)

**Level 0** is the case overview — always available regardless of execution model. Workers as rectangles, capabilities as ovals, edges showing which worker handles which capability, trigger labels on bindings. An execution model badge tells you whether this is a choreography, HTN planner, GOAP state machine, or something else. This level answers: what does this case definition contain?

**Level 1** changes shape depending on the execution model. A choreography renders as a reactive flow — bindings as nodes, data-flow edges derived from produced keys and trigger conditions. An HTN renders as a decomposition tree — compound tasks, guarded methods, leaf tasks. A GOAP would render as a state machine — world states as nodes, actions as edges. Each model has fundamentally different topology, so each needs its own diagram type. The drill-down workbench manages this — it maintains a stack of levels, shows collapsed strips for previous levels, and dispatches to the right diagram component based on what you drilled into.

**Levels 2 and 3** are worker detail and binding detail. Click a worker to see its function type (agent, react, a2a, MCP, SWF), execution policy, capability mapping, and data projections. Click a binding to see its trigger condition, target, outcome policy, and recovery config.

## YAML Is the Source of Truth — the Diagram Is a Projection

The diagram is not a separate artifact. It's a live projection of the YAML, and edits flow in both directions.

![Bidirectional sync between YAML source and visual diagram via yamlPaths](images/yaml-to-diagram.svg)

The case adapter produces a `GraphModel` from the YAML — nodes, edges, stencil types. But it also produces a `yamlPaths` map: every node ID maps to the YAML path it was derived from. This is the backbone of bidirectional sync. Click a node in the diagram, and the system knows exactly which YAML block to scroll to. Place your cursor in a YAML block, and the system knows which diagram node to highlight.

Edits are CST-preserving. When you change a property in the diagram's palette — rename a worker, change a model, add a capability — the edit is applied to the YAML's concrete syntax tree, not by serialising a new document. Comments survive. Formatting survives. The YAML you wrote by hand still looks like the YAML you wrote by hand, with one property changed.

This matters because YAML case definitions live in version control. They're reviewed in pull requests. If every diagram edit rewrote the entire file, the diffs would be unreadable. CST-preserving edits produce minimal diffs — exactly what changed, nothing else.

## Why Execution Models Need Different Diagrams

This is the part that's easy to get wrong. A naive approach renders every case definition the same way — workers as boxes, bindings as arrows, done. It works for simple cases and collapses for complex ones, because different execution models have fundamentally different topologies.

A **choreography** is a reactive system. There's no central controller. Bindings fire when data arrives — a document classifier produces `classifiedDocs`, which triggers the identity extractor, which produces `identityResult`, which triggers the risk assessor. The interesting structure is the data-flow chain, not the worker list. The right diagram is a reactive flow where bindings are nodes and data dependencies are edges.

An **HTN** (Hierarchical Task Network) is a decomposition tree. A compound task decomposes into methods, each method has guards and children, children can be compound tasks that decompose further. The interesting structure is the recursive breakdown — which decomposition was chosen, which guards evaluated to true, what the leaf tasks actually do. The right diagram is a tree.

A **GOAP** (Goal-Oriented Action Planning) planner works in state space. Actions have preconditions and effects. The planner searches for a sequence of actions that gets from the current world state to the goal state. The interesting structure is the state machine — which states exist, which actions transition between them, what the planner chose. The right diagram is a state-space graph.

Rendering all of these as "boxes and arrows" loses precisely the information that makes each model useful. The drill-down workbench dispatches to the right diagram type based on the execution model badge — choreography gets a reactive flow, HTN gets a tree, SWF gets a workflow diagram. The Level 0 case overview is the shared entry point; Level 1 is where model-specific topology appears.

## What This Means for Agent Orchestration

The case definition is the central artifact in CaseHub. It's where the system architect declares what agents do, how they coordinate, and what happens when things go wrong. Making this artifact visually editable — with proper drill-down, bidirectional YAML sync, and execution-model-aware rendering — is what makes it practical to build complex agent systems.

Without visual editing, the ceiling is low. A developer comfortable with YAML can build a five-worker choreography. A fifteen-worker system with HTN decomposition, GOAP planning for resource allocation, and SWF workflows for the mechanical steps? Nobody holds that in their head from YAML alone. The diagram makes the topology legible, and legibility is what lets you reason about correctness, identify missing bindings, trace data-flow chains, and spot coordination gaps before they become runtime failures.

The infrastructure is largely built. The case adapter, the stencils, the drill-down workbench, the CST-preserving editor, the property palette with function-type discriminators — all working. What remains is filling in the execution-model-specific diagrams (choreography reactive flow, GOAP state machine) and wiring up the YAML ↔ editor scroll sync. The hard architecture is done; the remaining work is domain-specific rendering.
