---
layout: post
title: "The Ref That Connects the Diagrams"
date: 2026-08-30
entry_type: note
subtype: diary
projects: [casehubio/engine]
tags: [yaml, design-time, drill-down, dsl-parity]
---

# The Ref That Connects the Diagrams

CaseHub has four diagram types — Case, Serverless Workflow, HTN, DAG — and a worker node in any of them can reference a definition in any other. A Case worker might embed a SWF, which dispatches back to another Case, which decomposes into an HTN. The drill-down graph crosses format boundaries naturally.

Until today, those references were opaque. A worker named `research-analyst` was a string. The UI knew the name but had no way to follow it — no path to the definition, no way to render the next diagram in the chain.

The fix is a single field: `definitionRef`. A relative file path or an inline `#name` reference, carried on any YAML node that points to another definition:

```yaml
workers:
  - name: triage-bot
    capabilities: [triage]
    definitionRef: cases/triage.yaml

  - name: investigation-flow
    capabilities: [investigate]
    definitionRef: '#investigation'

definitions:
  investigation:
    do:
      - collect-evidence:
          call: casehub:dispatch
          with:
            capability: evidence-collection
            definitionRef: cases/evidence.yaml
```

The engine stores it as-is — no file validation, no resolution. The UI follows the ref at design time, detects the diagram type from YAML structure (`workers:` + `bindings:` = Case, `do:` = SWF, `decomposition:` = HTN, `nodes:` = DAG), and renders the right diagram in a drill-down pane.

What makes this interesting is that it works cross-format by convention, not by type system. The `definitionRef` field appears on Case workers, on SWF `casehub:dispatch` steps (via the `with:` block), and will appear on HTN leaf tasks and DAG nodes once those get YAML schemas. For third-party SWF steps, `metadata.definitionRef` uses the standard SWF extension point. One convention, universal drill-down — any diagram node that carries `definitionRef` is navigable.

## The bigger realisation

The implementation itself was straightforward — a nullable string on `Worker`, parsing in `WorkerDeserializer`, threading through `CaseDefinitionPostProcessor` rebuilds, a `definitions:` block on `CaseDefinition`. But the conversation that led to it surfaced something more important.

I started thinking about this as a runtime problem — snapshot types, REST endpoints, the execution workbench. Claude explored `DagNodeSnapshot`, `LeafTaskSnapshot`, `PlanResource`, the whole snapshot infrastructure. I pulled it back: this is design time. We don't even have runtime yet for these diagram types.

That reframe clarified the scope, but it also surfaced a question I hadn't explicitly asked: why should YAML and Java be treated as peer representations of the same models?

The answer is three things that compound:

1. **Modeling-first design.** YAML definitions are the natural entry point for designing case behavior before writing code. Full YAML coverage means the entire design surface is accessible without a Java toolchain.

2. **Non-technical accessibility.** Domain experts and analysts can author and review case definitions directly. YAML's readability makes this practical where Java DSL does not.

3. **Interactive demo tutorials.** The pages platform builds slide-based demos that walk through definitions. YAML definitions render directly in these tutorials, enabling teach-by-example flows. This requires every execution model to be expressible in YAML.

This isn't a convenience feature. It's a platform principle. I updated the DSL Style Guide with a "YAML/Java Parity Principle" section and strengthened epic #978 with this rationale. Every casehub repo's CLAUDE.md now references it.

The three pathways — pure YAML, pure Java, hybrid — are a family, not a hierarchy. YAML is not a subset of Java. When we add a new builder field, the YAML surface ships in the same issue. When we design a new execution model, the YAML shape is designed alongside the Java DSL — not as follow-on work.

`definitionRef` is the first field designed with this principle already in place. It exists purely for design-time navigation — something YAML enables that Java DSL doesn't need (Java has the type system and IDE navigation instead). That's the kind of divergence the parity principle handles well: each pathway can have features the other doesn't, as long as every *model* is expressible in both.
