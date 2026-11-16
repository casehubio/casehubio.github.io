---
layout: post
title: "Why Removing an Edge Is Free"
date: 2026-10-26
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
tags: [bayesian-inference, junction-tree, algorithms, inference-bayes]
series: issue-681-dynamic-structure-adaptation
---

# Why Removing an Edge Is Free

The Bayesian junction tree algorithm has an elegant asymmetry hiding inside it. I found it while designing dynamic structure adaptation for the BJT engine — the ability to add and remove edges and variables at runtime without rebuilding the entire inference network.

The insight: removing a causal edge never invalidates a junction tree. Adding one almost always does.

When you build a junction tree from a Bayesian network, the first step is moralization — connecting all co-parents. If A and B are both parents of C, moralization adds an undirected edge between A and B. Triangulation then adds fill-in edges to make the graph chordal, and the resulting maximal cliques form the junction tree nodes.

Now remove the edge A → C. A is no longer a parent of C, so the moral edge between A and B (if it existed solely because of the co-parent relationship) is no longer needed. But here's the thing: the existing cliques already contain those variables. They're supersets of what's needed. A junction tree with oversized cliques is still a valid junction tree — the running intersection property holds because removing an edge from the DAG can only remove entries from the moral graph, never add them.

So edge removal is a constant-time structural operation: remove the edge from the DAG, swap in the child's new CPT (without the removed parent dimension), re-initialize the affected clique potentials, and propagate. No re-triangulation. No rebuild. The tree is suboptimal — cliques are larger than necessary — but numerically correct.

Edge addition is the opposite. A new edge A → B means B has a new parent, which means new moral edges between A and B's other parents. These new undirected edges might require variables to appear together in a clique that doesn't currently contain them. The triangulation might need new fill-in edges. The junction tree structure might need entirely different cliques. There's no way around it — you need to rebuild.

This asymmetry led to a three-tier dispatch design. Tier 1: the existing leaf operations from #627 (append a 2-variable clique, O(1)). Tier 2: edge removal (incremental, O(cliques)). Tier 3: everything else — full rebuild via BayesNetworkBuilder, with evidence snapshot and restore so the caller's likelihood observations survive across the reconstruction. A batch API wraps multiple Tier 3 operations into a single rebuild.

The rebuild path turned out to be the cleanest part of the design. Snapshot all evidence by variable name, reconstruct the network from surviving definitions via the existing builder, restore evidence on variables that still exist. Variable IDs change across rebuilds but all lookups are by name, so it works. The `ListGraphStore` stays append-only — no surgery on the graph implementation needed.

I'd expected this issue to require local re-triangulation — the algorithmically hard path where you splice new cliques into an existing junction tree while maintaining the running intersection property. That's what the literature calls research-grade. But the categorized approach sidesteps it: edge removal gets the meaningful optimisation for free, and everything else pays the rebuild cost, which for the network sizes we deal with (tens to low hundreds of variables) is negligible compared to the propagation cost that follows.
