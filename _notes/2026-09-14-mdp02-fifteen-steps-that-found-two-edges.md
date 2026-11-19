---
title: "Fifteen steps that found two edges"
date: 2026-09-14
author: mdp
entry_type: note
subtype: diary
series: issue-435-yaml-tutorial
projects:
  - casehubio/casehub-pages
tags: [yaml-core, tutorial, module-expander, composition]
---

# Fifteen steps that found two edges

The [previous entry](2026-09-14-mdp01-the-tutorial-that-found-a-language.md) described the design — yaml-core as a universal composition layer, schema composition via `z.intersection()`, a TypeScript port of the Java library. All architecture. This session was supposed to be the easy part: write fifteen tutorial steps and ship.

The tutorial content turned out to be the most effective test suite the expand engine ever had.

## The params gap

Step 8 teaches modules — define a reusable template with parameters, import it with values:

```yaml
modules:
  dashboard:
    parameters:
      label: {type: STRING, required: true}
    sections:
      pages:
        view:
          name: ${params.label} Dashboard
```

Every tutorial solution using `${params.label}` in a module section failed validation. The expand engine reported unresolved variables. I traced it to the ModuleExpander: it merges module sections into the working map *without resolving parameter references*. The Java original resolves `${params.*}` during module expansion; the TypeScript port skipped that step.

The fix was straightforward — create a parameter-scoped resolver in the section merging loop, deferring all other prefixes (`each`, `module`, user-defined variables) so they survive for later pipeline stages. Three new tests confirmed the behavior: params in values, params in keys, deferred prefixes preserved.

What made this interesting wasn't the fix — it was that unit tests hadn't caught it. The existing module expander tests used literal values in sections, never `${params.*}`. The tutorial forced realistic examples through the engine, and realistic examples use parameters.

## The forEach boundary

Steps 10, 13, and 15 originally used LIST parameters inside forEach:

```yaml
forEach:
  as: field
  in: [${params.fields}]
```

Two problems. First, `${` inside a YAML flow sequence is invalid syntax — YAML interprets `{` as the start of a flow mapping. Second, even quoted, the resolved value is a single comma-separated string, not multiple values. The forEach expansion doesn't auto-split.

I redesigned those steps to separate the concerns: modules parameterize structure with STRING values, forEach generates repetition with explicit value lists. They compose orthogonally — a module can produce map-format pages, and forEach can expand components within those pages — but forEach values can't be parameterized through module parameters in the current pipeline.

This is a real limitation, not a tutorial simplification. The pipeline order (modules → variables → forEach) means `${params.*}` is resolved before forEach sees it, but the resolved value is a flat string, not an array. Worth tracking as a follow-up if the pattern comes up in real usage.

## What the guides didn't have

Both the consumer guide and the contributor guide covered thirty-plus packages but didn't mention yaml-core. The doc sync added it to both — the consumer guide gets a YAML Composition section explaining variables, forEach, modules, and conditionals; the contributor guide gets the package listing with entry points and pipeline description. Documentation that doesn't cover new capabilities is documentation that will confuse the next person who reads it.
