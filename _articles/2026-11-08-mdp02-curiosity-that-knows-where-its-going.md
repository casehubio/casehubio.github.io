---
layout: post
title: "Curiosity That Knows Where It's Going"
date: 2026-11-08
entry_type: article
subtype: diary
projects: [casehubio/neocortex]
tags: [curiosity, goals, cognitive-architecture, information-gap, bfs, exploration, knowledge-graph]
series: issue-841-cognitive-exploration
---

# Curiosity That Knows Where It's Going

*Previously in this series: [Why Your AI Agent Isn't Curious](2026-11-07-mdp01-why-your-ai-agent-isnt-curious.md) built the information-gap curiosity engine, [When Curiosity Won't Let Go](2026-11-07-mdp02-when-curiosity-wont-let-go.md) added the pathological continuum, and [Three Systems, One Exploration](2026-11-08-mdp01-three-systems-one-exploration.md) coupled curiosity channels to the search budget.*

The curiosity engine from the first post has a problem I glossed over. It finds information gaps — partial-knowledge nodes in the knowledge graph where binary entropy peaks — and ranks them by intensity. An agent pursuing "learn Rust" gets the same curiosity ranking as an agent pursuing "learn to cook," because the ranking doesn't know about goals. A node about Rust ownership and a node about sourdough starters compete on entropy alone.

This is wrong in a way that matters. Curiosity without direction is browsing. Curiosity toward a goal is investigation. The difference isn't intensity — it's allocation. An agent that's curious about everything equally will scatter its attention across whatever happens to sit at moderate confidence. An agent whose curiosity is shaped by its goals will notice that the ownership gap is more interesting *because it's on the path to something it's trying to accomplish.*

## The gap was already almost closed

The machinery was already in place. `InformationGapDetector.detectGaps()` accepts an `activeGoalNodeIds` parameter — a set of goal node IDs that boost gap scores for directly adjacent nodes. But `CuriosityDrive.evaluate()` was passing `null`. Every time. The parameter existed, the boosting logic worked, and nobody was using it.

The original implementation was also too coarse. It checked whether any edge from a gap node touched a goal node directly — one hop only. If yes, the gap got classified as `GOAL_ADJACENT` with a flat 1.5× intensity boost. If no, nothing. Binary: you're next to a goal or you're not.

Two problems with this. First, the `GOAL_ADJACENT` classification replaced the information-theoretic classification. A node that was both at the tip-of-tongue threshold (high confidence, many edges, almost understood) and adjacent to a goal lost its `TIP_OF_TONGUE` identity. The gap type told you it was near a goal. It no longer told you what *kind* of gap it was. Information destroyed.

Second, one hop is not how relevance works in a knowledge graph. The node directly linked to your goal is obviously relevant. But the node two hops away — connected through an intermediate concept — is still relevant, just less so. And three hops out, less again. Relevance decays with distance. A binary yes/no at one hop misses the gradient.

## BFS proximity scoring

The fix uses the same algorithm that already existed elsewhere in the codebase. `GoalRelevanceModulationFactor` modulates memory retrieval by BFS proximity to active goals — a breadth-first search from each entity node, returning a graduated multiplier based on shortest path distance. Distance 1 returns 1.0. Distance 2 returns 0.7. Distance 3 returns 0.4. Beyond 3 hops, 0.0.

I applied the same pattern to gap scoring. `InformationGapDetector` now runs a BFS from each scored node to the nearest active goal, up to 3 hops. The result is a continuous `goalRelevance` field on `InformationGapScore` — not a binary classification, not a flat boost, but a graduated signal that decays with distance.

![BFS goal proximity with graduated relevance](images/bfs-proximity-decay.svg)

The boost formula replaces the old flat multiplier:

```java
if (goalRelevance > 0.0) {
    gapIntensity = Math.min(1.0, gapIntensity * (1 + goalRelevance));
}
```

A gap at distance 1 from a goal gets a 2.0× boost. Distance 2 gets 1.7×. Distance 3 gets 1.4×. All capped at 1.0. The ranking output from `detectGaps()` now reflects goal bias — goal-adjacent gaps rise in the sorted list, and the top-5 truncation in `CuriosityDrive` naturally picks them up.

## GOAL_ADJACENT had to go

Removing the `GOAL_ADJACENT` enum value was a deliberate design choice, not cleanup. The old code had `classifyGap()` return `GOAL_ADJACENT` and skip the information-theoretic classification entirely. A node near a goal that would otherwise be classified as `TIP_OF_TONGUE` — meaning the agent almost understands it, just one piece missing — lost that signal. All the downstream consumer saw was "near a goal."

With `goalRelevance` as a continuous field on the record, the classification can be both. A node is `TIP_OF_TONGUE` with `goalRelevance: 0.7`. The gap type tells you the shape of the gap. The relevance tells you how much the goal cares. Two orthogonal dimensions, both preserved.

![Undirected vs goal-directed curiosity](images/goal-directed-curiosity.svg)

## The supplier pattern

`CuriosityDrive` doesn't depend on `MindMapStore`. It lives in the `cognition` module. It knows about `InformationGapDetector` and `CuriosityChannelComputer` — both from `mindmap-intelligence` — but not about the store that holds the goal subgraph.

So instead of pulling in a store dependency, I added a `Supplier<Set<String>>` to the constructor. The wiring layer — the agency code that assembles the cognitive pipeline — provides a supplier that resolves active goal node IDs from the MindMap's GOAL subgraph. `CuriosityDrive` calls `supplier.get()` during `evaluate()` and passes the result to `detectGaps()`.

```java
public CuriosityDrive(InformationGapDetector gapDetector,
                      CuriosityChannelComputer channelComputer,
                      double deprivationBias,
                      Supplier<Set<String>> activeGoalNodeIds) {
```

A backward-compatible 3-arg constructor defaults to `Set::of` — no goals, no boost, identical behaviour to before. Existing agents see no change. Agents with goal integration get goal-directed curiosity.

## What this means in practice

Consider an agent with the goal "learn Rust." Its knowledge graph has partial-knowledge nodes for ownership, lifetimes, borrowing, trait bounds, and also unrelated nodes for cooking, photography, local history — all at similar confidence levels.

Without goal-directed curiosity, the ranking is pure entropy. Whichever topic happens to have the most edges and the most moderate confidence wins. The agent is equally likely to explore sourdough fermentation as Rust ownership.

With goal-directed curiosity, the BFS from each node finds that ownership is 1 hop from the "learn Rust" goal. Lifetimes is 2 hops away (through ownership). Trait bounds is 3 hops. Cooking is unreachable. The gap scores shift: ownership's intensity doubles, lifetimes gets a 1.7× boost, and cooking stays where it was. The top-5 gaps that feed the curiosity prompt are now dominated by Rust topics.

The prompt rendering reflects this too. When the curiosity channel is in specific focus mode and the top gap is goal-relevant, the agent hears: "Your attention keeps returning to ownership — closely related to an active goal." In diversive mode with multiple goal-relevant gaps: "Several threads worth exploring, some tied to active goals." The LLM gets enough context to orient its exploration.

## Why this matters beyond the implementation

Most agent frameworks that model curiosity treat it as a context-free drive. The agent is curious or it isn't. Intensity varies, but direction doesn't. This is how biological curiosity works in infants — exploratory play without agenda. But it's not how curiosity works in adults pursuing goals. When you're studying for an exam, you don't suddenly become fascinated by an unrelated Wikipedia article with the same probability as the textbook chapter you half-understand. Your goals shape what captures your attention.

The cognitive science backs this up. Loewenstein's original 1994 paper describes curiosity as arising from a gap between what you know and what you *want* to know. The "want" is doing real work in that sentence. It implies a reference point — something you're trying to understand, a goal you're pursuing, a question you're trying to answer. Without that reference point, you're measuring gaps against nothing.

The information-gap model was already right about the shape of curiosity — partial knowledge, entropy peaking at moderate confidence, attention weighted by connectivity. What it was missing was the direction. A gap adjacent to something you care about is more interesting than a gap floating in conceptual space. Not because the gap is different, but because the context around it is.

That's what goal-directed curiosity adds. Not a new model of curiosity, but a lens that focuses the existing model on what the agent is actually trying to do. The same entropy calculation, the same attention weighting, the same four-channel rendering — now shaped by proximity to purpose.
