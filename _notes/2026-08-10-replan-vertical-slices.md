---
date: 2026-08-10
title: "Replanning fsitrading: from layers to vertical slices"
type: diary
tags: [replan, architecture, platform, blocks, pages]
---

The CaseHub platform has changed substantially since fsitrading was first scaffolded in June. blocks now has a full agentic orchestration framework with eight composable patterns. blocks-ui ships 31 domain components. pages just landed IntelliJ-style dock workbenches with drag-and-drop tool windows and floating panels. The old chapter-per-layer plan was designed for a platform that no longer exists.

The replan replaces it with seven vertical slices. Each chapter delivers a working end-to-end trading scenario — backend orchestration, platform pattern integration, and UI panels composed into dock-workbench layouts. The priority order is deliberate: showcase the platform first, then domain fidelity, then layer coverage.

The most distinctive design decision is using blocks' multi-level summarisation framework as the market analysis architecture itself. Five temporal levels — raw ticks through 1-minute bars, 5-minute trends, hourly regime detection, and session narrative — each produced by a SummarisationRunner chain. Strategy agents observe at the granularity matching their time horizon. Market makers see ticks. Portfolio rebalancers see narratives. The analysis emerges from the level hierarchy.

C0 (foundation fixes) landed today: @Transactional for dual-datasource atomicity, casehub-blocks dependency with CDI routing strategy exclusion, ARC42STORIES rewrite, and the full epic structure for C1-C6. Two garden entries captured along the way — H2 XA enlistment and blocks CDI ambiguity.

Next: C1 Strategy Arena — the orchestration, routing, and accountability backbone.
