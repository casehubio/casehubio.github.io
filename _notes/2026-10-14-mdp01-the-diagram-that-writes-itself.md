---
layout: post
title: "The Diagram That Writes Itself"
date: 2026-10-14
entry_type: note
subtype: diary
projects: [casehubio/blocks-ui]
tags: [web-components, data-bridge, org-diagram, lit]
---

# The Diagram That Writes Itself

The relationship editor had two views — a grouped table for editing, an arc diagram for visualising direct connections. Both consume the same flat data: a roster of agents and an array of relationships. The org-diagram component, which already renders rich interactive org charts with avatars, professions, and escalation chains, consumes something entirely different: a YAML string describing an organisational hierarchy with nested units, members, and typed relationships.

The interesting design question was how to bridge between these two data shapes without duplicating rendering logic or wrapping the diagram in a way that defeats its own composition pipeline.

The answer turned out to be a single pure function: `toOrgYaml`. It takes the roster, groups agents by their `unitId` into org units, attaches the relationships, and serialises to the YAML string the diagram already knows how to consume. Agents without a unit land in a synthetic "unassigned" group. The function lives alongside `toOrgGraph` in the stencil-org adapter package — both are org-domain data transformations, one going each direction.

What I find satisfying about this is how little the relationship editor needs to know about the diagram. It passes the YAML string and an agents descriptor record — the same API the diagram uses everywhere else. The editor doesn't import any rendering code, doesn't register stencils, doesn't configure layouts. It just writes a `<blocks-org-diagram readonly>` tag in its template and the consumer's page handles the element registration. The diagram is a black box that happens to render beautifully.

The more interesting choice was what the diagram should show during editing. The table view already marks pending additions in green and pending removals with strikethrough — that's the diff view. We made the diagram show the merged state instead: confirmed relationships plus pending additions minus pending removals. The diagram is the "what it will look like" preview. The table is the "what's changing" diff. Each view has a clear role, and neither tries to do both.

We also wired click-to-focus: clicking an agent node in the diagram sets it as the focal agent, which re-groups the table around that agent's relationships and re-centres the arc view. The org-diagram already emits selection events via its `selectionTopic` property — we just listen and extract the agent ID from the node ID format. The bidirectional flow feels natural: see an agent in the diagram, click them, edit their relationships in the table.

The table rows got richer too. With the `agents` descriptor record already flowing in for the diagram, the table rows can use it to show avatar thumbnails and profession badges alongside agent names. A 24px `<agent-avatar>` with `size="xs"` drops into a flex row without fuss — the same pattern the org stencils use internally.

The data bridge pattern is worth noting beyond this specific case. When you have two web components that consume fundamentally different data shapes but represent the same domain, a pure adapter function in the domain's stencil package keeps the transformation logic in one place and lets both components stay ignorant of each other. The relationship editor doesn't know how the diagram renders. The diagram doesn't know where its YAML came from. The bridge is the only piece that understands both shapes, and it has no rendering concerns of its own — just data in, data out.
