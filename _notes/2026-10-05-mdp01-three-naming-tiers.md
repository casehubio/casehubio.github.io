---
layout: post
title: "Three naming tiers for the playbook system"
date: 2026-10-05
entry_type: note
subtype: diary
projects: [casehubio/platform]
tags: [naming, playbook, yaml, state-machine]
series: issue-520-playbook-naming
---

# Three naming tiers for the playbook system

The playbook naming unification (epic #520) reached the point where the remaining
platform-side renames needed judgment calls, not just find-and-replace. The session
surfaced a naming model I hadn't thought through in advance — three tiers, each
with a different relationship to the old "Scenario" terminology.

**Tier 1: the top-level construct.** A scenario is now a playbook. This was decided
in #510 and is already landed — `PlaybookFrontMatter`, `PlaybookParser`,
`PlaybookDocument`, the `.playbook.yaml` file extension. No ambiguity here. The
thing a user writes and the system executes is a playbook.

**Tier 2: internal execution components.** `StepWalker`, `StructuralStepEvaluator`,
`StepCatalog` (now `PluginRegistry`). The issue had these marked for rename —
`StepWalker → Walker`, `StructuralStepEvaluator → StructuralEvaluator`. I pushed
back. Playbooks execute steps. A walker that walks steps is naturally a `StepWalker`.
Removing the "Step" prefix strips domain specificity for no naming benefit. The
unification targeted the top-level identity ("Step YAML" as a format name), not the
internal vocabulary. "Step" as a concept within playbooks stays.

**Tier 3: the state machine DSL.** `ScenarioParser`, `ScenarioDefinition`,
`ScenarioCompiler`, `ScenarioValidator`, `CompiledScenario`. These parse and
compile the `states:` block — states, transitions, events, deadlines. The epic
branch had renamed them to `Playbook*`, but that collides with `PlaybookParser`
(which handles front matter) and doesn't describe what the content actually is.
We renamed to `StateMachine*` instead — `StateMachineParser`, `StateMachineDefinition`,
`StateMachineCompiler`, `StateMachineValidator`, `CompiledStateMachine`. Package
moved from `io.casehub.yaml.step.scenario` to `io.casehub.yaml.step.statemachine`.

The model is clean: Playbook is what the user writes. Steps are what the playbook
executes. State machine is the specific execution topology for playbooks that use
states and transitions. `ExecutionScope` stays as a runtime orchestration primitive —
that's consistent with the epic's explicit scoping.

The terminology sweep (#519) was two files. "CaseHub YAML" → "CaseHub Playbook YAML"
in the language guide title and the diagram editor spec. The spec doc annotation
issue (#518) turned out to have no valid work — the annotations assumed Step* renames
that shouldn't happen.
