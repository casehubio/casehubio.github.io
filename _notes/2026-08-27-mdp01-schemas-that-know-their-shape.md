---
layout: post
title: "Schemas That Know Their Shape"
date: 2026-08-27
entry_type: note
subtype: diary
projects: [casehubio/blocks-ui]
tags: [diagram, json-schema, property-palette, discriminated-unions]
---

# Schemas That Know Their Shape

Every diagram node in CaseHub now carries its own property schema — a JSON Schema descriptor that tells the palette exactly what to render, how to group it, and when to show it.

The old approach had each diagram component maintaining a `_schemaTypeMap()` method that mapped node types to `$defs` keys in a monolithic schema object. It worked for SWF's six task types, but the case diagram has five node types with radically different shapes — a worker alone has seven function types, each with its own nested config. A milestone has six fields. A worker agent has a model provider with five branches, each with temperature, maxTokens, topP. Mapping all of that through `$defs` lookups would have produced an unreadable schema file and a brittle type map.

We replaced the whole mechanism with a per-node-type registry. A simple `Map<string, JSONSchema>` in diagram-core, two functions: `registerPropertySchema` and `getPropertySchema`. Each stencil package registers its schemas during `registerCaseStencils()` / `registerSwfStencils()` / `registerHtnStencils()` — right alongside visual stencil registration. One call site, one registration pattern, three packages.

The interesting design question was how to represent discriminated unions in JSON Schema for a property palette. A worker's function type switches between agent, flow, a2a, mcp, and sequence — each with completely different fields. The binding's trigger switches between contextChange, cloudEvent, schedule, and scopeActivated. Standard JSON Schema `oneOf` gives you the branching, but the palette needs to know which field controls the branch selection. We added `x-discriminator` as a schema extension — it names the field whose value selects the active `oneOf` branch. The palette renders a type selector dropdown for that field and swaps the visible sub-schema when it changes.

```json
{
  "x-discriminator": "_type",
  "oneOf": [
    { "properties": { "_type": { "const": "agent" }, "systemPrompt": { ... }, "model": { ... } } },
    { "properties": { "_type": { "const": "mcp" }, "transport": { ... } } },
    { "properties": { "_type": { "const": "a2a" }, "endpoint": { ... } } }
  ]
}
```

The worker schema nests two levels deep — the agent's model has its own `x-discriminator` on provider (openai, anthropic, ollama, mistralAi, googleAiGemini), and the MCP transport has one on stdio vs http. The palette handles this recursively without any special case code.

Five custom editor stubs sit alongside the schemas for fields where a text input is wrong: `blocks-prompt-editor` for system prompts (textarea with future pop-out dialog), `blocks-env-map-editor` for KEY=VALUE environment variables, `blocks-sequence-editor` for ordered worker lists, `blocks-swf-link` for the read-only "edit via SWF drill-down" link, and `blocks-json-editor` for opaque JSON display. Each follows the same contract — `value` property, `value-changed` event — so the palette creates them via `document.createElement(tagName)` and doesn't care which one it gets.

The grouping convention across all schemas uses six named groups with fixed ordering: Identity (0), Configuration (10), Target (15), Function (18), Behaviour (20), Advanced (30). Advanced fields carry `x-visibility: 'advanced'` so the palette can hide them behind a toggle. A milestone has two groups. A worker has five. The palette doesn't need to know the difference — it reads the annotations and renders accordingly.

The HTN schemas are the odd ones out — they describe runtime execution snapshots from the engine's REST API, not YAML definitions. DAG nodes and plan items are display-only, with `readOnly: true` on most fields. The property palette shows them but doesn't allow editing. This is the right separation: the case YAML is the editable source of truth, the HTN snapshot is a read-only view of what the engine produced from it.

Next is wiring these schemas into the actual pages palette components — `pages-property-palette` and `pages-diagram-palette` both landed in pages and are waiting to be consumed. That's where the schemas stop being data structures and start driving a real editing UX.
