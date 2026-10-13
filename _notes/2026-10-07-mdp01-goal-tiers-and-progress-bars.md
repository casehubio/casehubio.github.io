---
layout: post
title: "Goal tiers and progress bars — making the goal graph visible"
date: 2026-10-07
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
tags: [goals, mindmap, consolidation, visualization]
series: issue-467-goal-tier-property
---

# Goal tiers and progress bars — making the goal graph visible

The cognitive workbench needs to render goals at different sizes and positions based on what kind of goal they are. A standing life orientation ("be healthier") isn't the same as a concrete next action ("call the doctor"), and the visualizer needs to know which is which without guessing from the horizon field.

We added a `goal-tier` property — THEMATIC, STRATEGIC, or TACTICAL — derived from the existing horizon. Aspirational goals are thematic (standing orientations). Medium and long-horizon goals are strategic (scoped initiatives). Immediate and short-horizon goals are tactical (concrete actions). The derivation is a `GoalTier` enum in mindmap-api with a `fromHorizon()` switch — simple, but it needed to be set consistently at every creation site: experience-derived goals in `GoalRecognitionPhase`, decomposed sub-goals in `GoalResolutionPhase.expand()`, and drive-proposed goals in `DriveGoalBridgeParticipant`.

The interesting design question was progress computation. When a goal decomposes into sub-goals via `decomposes-into` edges, the visualization wants a progress bar — "3 of 5 sub-goals completed (60%)." But the goal graph is recursive. A strategic goal might decompose into tactical sub-goals, and one of those tactical goals might itself have sub-steps.

I went with recursive averaging rather than flat counting. A leaf goal's effective progress is 1.0 if completed, 0.0 if not. A parent's progress is the mean of its children's effective progress. This propagates smoothly upward — if a mid-level goal has two children, one completed and one half-done, it reports 0.75, and its parent sees that fractional progress rather than counting it as zero because it isn't "completed" yet.

The computation runs as the last step of `GoalResolutionPhase.run()`, after revise and sync have settled all status changes. It reads a fresh node snapshot, builds a parent→children map from `decomposes-into` edges, and walks the graph bottom-up with memoization. A visited set per recursion path guards against cycles — unlikely since `detectAndBreakCycles` runs earlier, but defensive code is free in a consolidation phase that runs once per tick.

This is two of the five Batch 1 issues for the cognitive workbench epic. The remaining piece before visualization can start is the explicit tier for thematic/strategic/tactical presentation — now landed, the UI team has what it needs.
