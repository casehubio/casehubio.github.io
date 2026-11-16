---
layout: post
title: "Porting the Good Parts"
date: 2026-10-01
entry_type: note
subtype: diary
projects: [casehubio/blocks-ui]
tags: [web-components, lit, avatar, workbench, incremental]
series: issue-219-workbench-incremental-improvements
---

# Porting the Good Parts

The unified personality selector (#218) was a rewrite that lost too much. Avatar-step's centralised filter pipeline — the `_applyFilters` chokepoint that every interaction flows through — is elegant and battle-tested. Rebuilding it in a monolithic workbench was a regression, not a simplification.

The decision: keep the two-tab workbench from #217. Port the four best ideas from #218 as incremental additions to the existing components. No structural changes, no rewrite. Each feature slots into the existing architecture.

## The collection bar belongs to the workbench

Avatar-step had its own collection bar. The workbench needed one visible on both tabs. The pattern that fell out: the workbench owns `_collection` state, passes it down as a property. When avatar-step receives an external `collection`, it hides its own bar and syncs the value to its internal `_collection` via `willUpdate`. Agent-catalog gets the same property and passes it through to every `agent-avatar` element.

The tricky part was Vite's module resolution. In a yarn workspace monorepo, `@casehubio/agent-avatar-2d` resolves through a symlink to `packages/agent-avatar-2d/dist/index.js`. But the demo HTML imports from `src/index.ts`. Two different module instances, two separate collection registries. Collections registered in one are invisible to the other — no error, just a silently empty bar. I ended up importing `listCollections` from the relative source path in the workbench to guarantee a single registry instance at dev time.

## Grid tiers that communicate

The #222 profession ghosting was the most visually impactful change. When you select a profession pill, archetypes not mapped to that profession drop to 20% opacity and scale down. But ghosting alone wasn't enough — the user correctly pointed out that role-matched archetypes need positive differentiation, not just "not ghosted."

Three tiers emerged:
- **Selected** — blue background + border
- **Role-matched** — accent border + bold blue label (other archetypes in the same profession/role)
- **Ghosted** — 20% opacity, scaled down (everything else)

CSS cascade order matters here. All the tier classes have equal specificity (two class selectors each), so source order determines the winner. Declaring them in ascending priority — `strong` then `incompatible` then `profession-ghosted` then `role-match` then `selected` — means the last applicable rule wins without specificity hacks.

## The profile panel grows up

The detail header ported from #218 transforms the profile panel from a picker into an identity card. When a template is selected, the panel shows the avatar, name, archetype key, description, preferred alias, a personality summary, and role badges for every profession/role the archetype maps to. The workbench computes `archetypeRoles` via `getRolesForArchetype` and passes it down alongside `templateName`, `templateDesc`, and `templateAlias`.

The architecture stayed clean: properties down, events up. No child component reaches into a sibling. The workbench is the only thing that knows what's selected — children just render what they're told.

## What this approach proved

Four features, four commits, zero structural changes to avatar-step's filter pipeline. The existing `_applyFilters` chokepoint handled profession ghosting without modification — `_professionArchetypes()` and `_roleArchetypes()` are pure computed sets that feed into the grid's `classMap`. The two-tab workbench absorbed collection switching, group toggling, and template detail pass-through without growing complex.

The #218 branch was worth building even though we threw most of it away. It proved which features mattered and provided reference implementations to port from. The mistake would have been shipping the rewrite. The value was in the exploration.
