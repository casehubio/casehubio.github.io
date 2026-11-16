---
layout: post
title: "Four Views, One Pipeline"
date: 2026-10-29
entry_type: note
subtype: diary
projects: [casehubio/casehub-mosaic]
tags: [mosaic, web-components, visualization, agency, graph-rendering]
---

# Four Views, One Pipeline

CaseHub's agency layer has a lot of moving parts — agents, patterns, desired state graphs, deployment topologies — and until now the UI coverage was uneven. Some patterns had views; others fell through to "not yet supported." The infrastructure views — what's deployed, what's converging, what's healthy — didn't exist at all. Building them required a rendering pipeline that could handle DAGs, compound nodes, and status decorations without each component reinventing the layout engine.

## The Graph Pipeline

The rendering pipeline has four stages. A domain adapter transforms a snapshot (desired state, topology, or anything with nodes and edges) into a `GraphModel`. ELK computes the layout. A transform step converts the positioned graph into ReactFlow nodes and edges, merging in `NodeDecoration` overlays. The canvas renders the result.

![Graph rendering pipeline: domain adapter → ELK layout → ReactFlow transform → canvas](images/graph-pipeline.svg)

The key abstraction is `NodeDecoration` — a bag of optional visual properties: `badge` (icon + colour + pulse), `border` (style + colour), `pills` (text + colour array), and `tooltip`. Every domain expresses status through the same decoration vocabulary. A converged desired-state node gets a green solid border. A drifted one gets amber plus a ⚠ badge. A deploying topology service gets a blue border and a strategy badge like `🐤 Canary`. The rendering pipeline doesn't know what these mean — it just places them.

The desired-state graph was the first consumer. `DesiredStateSnapshot` carries nodes with six statuses (CONVERGED, PENDING, DRIFTED, FAULTED, PROVISIONING, ABSENT), `dependsOn` edges for backward resolution, and `forEachSource` edges for forward expansion. The adapter maps these to graph nodes and two edge types. Decorations are a separate pure function — given a snapshot, produce a Map from node ID to `NodeDecoration`. Adapter tests are pure (no DOM); component tests mock the layout engine.

```typescript
export function desiredStateToGraph(snapshot: DesiredStateSnapshot): DesiredStateAdapterResult {
  const nodes: GraphNode[] = snapshot.nodes.map(n => ({
    id: `${PREFIX}${n.id}`,
    type: 'desired-state',
    properties: { name: n.name, status: n.status, desired: n.desired, actual: n.actual },
  }));
  // ... dependsOn and forEach edges
  return { model: { nodes, edges } };
}
```

The topology-viewer was the second consumer and the harder test. It needed compound nodes — services grouped inside cluster containers — plus deployment strategy badges, replica pills, and image tooltips. `ClusterGroup` nodes get `parentId` references so ELK renders them as containers. The topology adapter uses the same pipeline but produces a richer graph model. Once that worked, we had confidence the pipeline could handle any domain.

## Human Work in Agent Pipelines

Not everything in the agency layer is automated. Work items are tasks that need a human — review a document, approve a gate, verify an extraction. The `<human-work-item-card>` renders one of these in agent-viewer: teal left border (the human colour, distinct from the blue agent hexagons), a 👤 icon, priority badge, status indicator, and an embedded `<agency-sla-indicator>` for deadline countdown. When the SLA breaches, the card pulses.

![Agent step vs human work item card — distinct visual identity in coordination pipelines](images/human-work-item-card.svg)

The card consumes `WorkItemResponse` from mosaic-core — twelve statuses from PENDING through ESCALATED to OBSOLETE, four priority levels. It's a small component, but it fills a gap: until now, human-in-the-loop steps in a coordination pipeline had no visual representation. An agent-delegate step shows a hexagon; a human work item step now shows a teal card with a deadline.

## Situation-Triggered Mutations

A topology isn't static. Volatility spikes, maintenance windows, scaling events — each one triggers coordinated mutations across the service graph. Scale up the risk assessors, reroute payment traffic to a fallback, spin down non-essential workers. The interesting design question: how do you visualise mutations without losing the baseline topology?

We rejected side-by-side comparison (doubles the rendering cost, eats horizontal space) and toggle mode (loses simultaneous context). What worked: overlay annotations on the existing graph. A `Situation` is a named scenario with a condition expression and a list of mutations. Each mutation targets a node by ID and specifies an action — scale-up, scale-down, add, remove, or reroute. The topology-viewer renders the baseline graph unchanged; when a situation is active, mutation badges appear on affected nodes.

![Situation pill bar with mutation badges on a topology graph](images/situation-pill-bar.svg)

The pill bar is a `radiogroup` — Baseline is always first, one situation active at a time. `topologyToDecorations` gained an optional second parameter: pass an active situation and it overlays mutation badges on top of the existing status decorations. A RUNNING node with a scale-up mutation shows both its green border and the ↑ badge. No layout recomputation — mutations only change decorations, not the graph structure.

Five actions, five badge icons, five Catppuccin colours. The situation lives on `TopologySnapshot` as an optional array — rendered when present, ignored when absent. Zero breaking changes.

## The Dashboard Problem

With topology, desired-state, agents, and patterns all rendering individually, the missing piece was a single view that answers: where is this project? How complete is each layer?

I chose a generic model over a fixed four-card layout. A `LayerSummary` has an id, label, icon, total, completed, health status, and optional metrics. The dashboard accepts any array of layers — four today, potentially more as the platform grows. Each card shows a progress bar, a health badge (healthy/degraded/incomplete), and optional metric pills.

![Project dashboard with layer cards and completeness matrix](images/project-dashboard.svg)

`overallHealth` is a pure one-liner: scan the layers, worst health wins. If any layer is incomplete, the project is incomplete. Card clicks emit a `pages-event` with the layer id — the host app handles drill-down. The dashboard stays thin and decoupled from every layer's detail view.

The completeness matrix complements the dashboard with a different lens. Instead of "how much is done," it asks "what stage is each layer at?" Four signal columns — exists, configured, tested, deployed — with status indicators per layer. A row with ● ● ◐ ○ tells a story that a progress bar can't: the agents exist and are configured, testing is partial, nothing is deployed yet. An optional next-action column points to what's most useful to do next.

## Focus Mode

The coordination pipeline view renders every step in an agent pipeline — REST calls, GraphQL queries, MCP tool invocations, agent delegations. For a KYC pipeline with four agents, three extractors, and a risk assessor, that's a lot of plumbing. The LLM decision points — where agents actually reason — are buried in the noise.

Focus mode is a single boolean property. When active, non-agent-delegate steps dim to 25% opacity. No layout changes, no hidden elements, no mode switching. The LLM decision points stand out because they're the only things at full opacity. A `🔍 LLM Logic` badge appears in the pipeline header.

![Focus mode dimming non-LLM steps in a coordination pipeline](images/focus-mode.svg)

The implementation is one check in `renderStepCard`:

```typescript
const isDimmed = this.focusMode && step.action !== 'agent-delegate';
```

The `ACTION_ICONS` map already discriminated agent-delegate from rest/graphql/mcp-call — focus mode just leveraged that existing classification.

## The Decoration Pattern

What's interesting looking back across these components is how much `NodeDecoration` absorbs. Status borders, drift badges, deployment strategy pills, convergence tooltips, provisioning pulses — all from the desired-state and topology work. Mutation overlays stacked on top without modification. The same badge/border/pills/tooltip structure that shows "this node is DRIFTED" also shows "this node will scale up when volatility spikes."

The project dashboard and completeness matrix don't use graph decorations, but they follow the same principle: typed status data in, visual indicators out, host app handles interaction. A `LayerHealth` maps to badge colours the same way a `DesiredStateNodeStatus` maps to border colours. The rendering vocabulary is converging even across components that share no code.

Two issues remain in the epic — a YAML case definition diagram and gallery demo data from real project definitions. Both are cross-module work. The graph pipeline is ready for them.
