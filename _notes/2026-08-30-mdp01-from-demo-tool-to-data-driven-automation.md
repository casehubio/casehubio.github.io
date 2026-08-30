---
layout: post
title: "From demo tool to data-driven automation"
date: 2026-08-30
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [scenario-engine, automation, compilation, aria-targeting, script-library, gallery]
---

# From demo tool to data-driven automation

Yesterday's session designed the script library. Today we built it — all six batches, from the compilation pipeline through to interactive gallery examples you can click through in the browser. But the interesting story isn't that we shipped it. It's what we learned about the gap between "scripts exist" and "scripts are browsable, runnable, and composable."

## The compilation pipeline and the CSV problem

The core of the automation platform is `ScenarioCompiler.compile(yaml, callerParams)`. It wires `casehub-platform-yaml-core`'s shared primitives — `VariableResolver`, `ForEachExpander`, `CsvParser`, `Truthiness` — into a pipeline that takes scenario YAML and produces a flat list of expanded, resolved steps.

The interesting design problem was CSV data sources. A scenario can declare inline CSV with typed columns, iterate over rows, and use column values in conditionals:

```yaml
dataSources:
  team-members:
    type: csv
    data: |
      name:String,email:String,admin:Boolean
      Alice,alice@example.com,true
      Bob,bob@example.com,false
      Carol,carol@example.com,true

steps:
  - label: "Grant admin access for ${each.member.name}"
    forEach:
      source: team-members
      as: member
    when: "${each.member.admin}"
    commands:
      - action: click
        target: {role: button, name: "Grant Admin"}
```

The `ForEachExpander` is generic — it stamps elements per iteration value and evaluates `when` conditionals. But it only knows about simple string values. CSV rows have typed columns, and the `when` condition needs to see column values like `${each.member.admin}` to decide whether to include a step.

The first attempt tried to shoehorn CSV through the expander by converting data source names to iteration groups and adding row context in the `ForEachAdapter.stamp()` method. It broke: the expander evaluates `when` *before* calling `stamp`, so the row context wasn't available when the conditional needed it.

The fix was to accept that CSV forEach is scenario-specific logic that doesn't belong in the generic expander. `expandCsvForEach` runs first as a pre-pass: it iterates rows manually, builds a resolver with both `withEachContext` (for the stamp key) and `withEachRowContext` (for column access), evaluates `when` per row, resolves all command values, and produces flat steps with no forEach or when left. The `ForEachExpander` then handles only simple iteration groups — the clean case it was designed for.

## The index field that crossed the stack

Table population needs positional targeting. You can't assume table rows have accessible names — a `<tr>` doesn't naturally have `aria-label="Row 0"`. The YAML surface for this is clean:

```yaml
  - label: "Populate user table"
    forEach:
      source: team-members
      as: member
    commands:
      - action: fill
        target:
          role: row
          index: "${each.index}"
          within: {role: table, name: Users}
        value: "${each.member.name}"
```

But `{role: row, index: ${each.index}}` had to flow through five layers:

1. **Java record**: `AriaTarget(role, name, index, within)` — `name` became nullable
2. **Parser**: `HierarchicalParser.parseAriaTarget()` reads `index` from YAML
3. **Compiler**: `resolveAriaTarget()` resolves `${each.index}` in the index field
4. **TypeScript type**: `AriaTarget` interface gains optional `index?: string`
5. **Tree walker**: `resolveTarget()` picks the Nth matching element when index is present

Each layer had its own concern. The Java side stores index as a `String` (not `Integer`) because the parser sees `"${each.index}"` before compilation resolves it. The TypeScript side parses the string to int at match time. The compiler adds the iteration counter to `withEachContext(Map.of(as, rowKey, "index", String.valueOf(i)))` — one line, but it unlocks the whole pattern.

## Call graphs and acyclic enforcement

Script composability — one script calling another via `action: call` — gives automations a module system:

```yaml
steps:
  - label: "Onboard new hire"
    action: call
    script: create-user
    params:
      name: "${params.employeeName}"
      role: engineer

  - label: "Set up workspace"
    action: call
    script: provision-workspace
    params:
      user: "${params.employeeName}"
```

The callee's steps are inlined at the call site with name prefixing — `create-user/Fill name`, `create-user/Click submit`. The callee inherits the caller's resolved context, and each call can pass its own params.

Composability needed cycle detection. `CallGraphValidator` does DFS with path tracking. When it finds a cycle, the error message includes the full path: `root → A → B → root`. The validator takes a `Function<String, Optional<ScriptRef>>` resolver, keeping it decoupled from the registry. The acyclic check runs before inlining — a cycle would cause infinite recursion if inlined first.

## The descriptor model — browsing before compiling

A script library needs to show what's available without compiling every script. `ScriptDescriptorExtractor` solves this by parsing just the YAML header — name, description, typed parameters, CaseHub labels, free-form tags — and extracting ARIA targets from the first step. Full compilation happens only when you run a script. Browsing is a parse operation, not an execution one.

`ScriptDescriptor` carries the metadata plus a `ScriptProvenance` — bundled (shipped with the distribution), uploaded (pasted by users), or external (fetched from a remote registry). The provenance matters because it determines trust: bundled scripts can run without confirmation, uploaded scripts might need review.

`ScriptRegistry` aggregates multiple `ScriptSource` implementations: `BundledScriptSource` scans the classpath, `UploadedScriptSource` holds user-submitted scripts in memory, and `ExternalRegistrySource` fetches a JSON manifest from an HTTP endpoint. Name collisions across sources are rejected at registration — if two sources provide a script called `onboard-user`, the second one fails loudly rather than silently overwriting.

The REST layer is two endpoints. `GET /scenario/library` returns all descriptors with optional label and tag query filters. `GET /scenario/library/{name}` returns a single descriptor with its full YAML for compilation. The filtering happens server-side in the registry — the controller doesn't need to fetch everything and filter locally.

## The readiness probe — the server can't see the DOM

The natural instinct is to put `ready: boolean` on the server's descriptor. But the server has no DOM. It can't call `findByRole()`. It doesn't know whether the form the script targets is actually rendered.

The fix flips the responsibility. `ScriptDescriptorExtractor` pulls ARIA targets from the first step at parse time and includes them in the descriptor. The client receives targets like `{role: textbox, name: "Customer name"}` and runs `probeReadiness(targets)` locally — calling `resolveTarget` on each and returning `ready`, `not-ready`, or `unknown`. Green, amber, red — without a single server round trip. In the gallery, with the form elements actually in the DOM, the probes resolve live as the page loads.

## The library view

`PagesLibraryView` is a Lit web component embedded in the scenario controller as a toggleable panel. It has two modes: server-connected (fetches from `GET /scenario/library` on mount) and standalone (scripts passed via the `.scripts` property). The standalone mode made the gallery examples possible — the companion script sets scripts on the element and listens for `script-selected` events.

The Run button is where the library connects back to the engine. Clicking Run on a script dispatches it through the existing `ScenarioConnectionController` — the same controller that handles play/pause/step for manually loaded scenarios. The library adds a browse-and-run flow on top of the existing execution infrastructure without duplicating any of it.

Theme adaptation took longer than I expected. The library view needed to work in both the controller's dark glassmorphic card and the gallery's light dashboard wrapper. Three rounds of fixes replaced hardcoded colours with design tokens — `var(--pages-neutral-N)` for text and borders, `var(--pages-surface-N)` for backgrounds. The toggle button that opens the library panel was the last holdout: its icon was invisible in dark mode until we switched to `currentColor` inheriting from the parent's text colour.

## Gallery examples — seeing it work

The examples gallery runs companion `.ts` files through a `stripTs` function that removes TypeScript syntax, then executes the result via `new Function()`. This means no imports, no type annotations, no `interface` declarations — every companion script is ES5-compatible JavaScript with a `.ts` extension.

Six examples cover the range, but three stand out.

**Table population** puts the index-based ARIA targeting to work. A CSV data source defines 5 users with name, email, and admin columns. The scenario iterates rows, targeting `{role: row, index: ${each.index}}` to fill each table row in sequence. You watch the table fill cell by cell — each row appearing as the iterator advances. It's the first example where data-driven iteration is visible, not just described.

**Composable workflow** demonstrates call-graph inlining. A parent script calls three child scripts — "Create user", "Set permissions", "Send welcome email" — each with their own parameters. The controller's outline shows the full inlined tree with section highlighting: the parent's steps interleaved with the children's, each prefixed with the callee's name. The readiness probes run on the parent's entry targets, so the green/amber/red indicators reflect whether the parent's first step can execute, not the children's.

**Scenario controller** is the real `<pages-scenario-controller>` component running standalone. A mock state pump simulates the push wire, cycling through step completions at configurable speed. The outline highlights advance, the transport controls respond, and the progress bar fills — all driven by the same Lit component that runs in production, just with synthetic state instead of a WebSocket.

The real `PagesLibraryView` component works in the gallery because it's bundled via esbuild into `dist/controller.js` and imported by the gallery's webpack entry point. The scenario controller example creates the component programmatically — setting `eventTarget` before DOM insertion so the connection controller picks it up in `firstUpdated`.
