---
title: When Cases Aren't Trees
type: diary
project: blocks-ui
date: 2026-08-09
---

Every case management system I've seen treats case relationships as a tree. Parent case, child cases, expand, collapse, done. CaseHub's own `entity-tree` does exactly this — select a case, drill into its sub-cases, view the hierarchy. It works.

Until it doesn't.

Consider a fraud investigation. The bank opens case #4012 for a suspicious wire transfer. During investigation, they discover the same account was flagged six months ago — case #3841, now closed. #4012 doesn't contain #3841 and #3841 doesn't contain #4012. They're peers linked by a shared actor. The investigator needs to see this connection. A tree can't show it — by definition, a tree has one parent per node, and these two cases share no ancestor.

It gets worse. The compliance team supersedes #4012 with a broader investigation, #4087. The old case isn't deleted — it has audit trail, evidence, decisions already recorded. It's superseded. #4012 → #4087 is a "this replaced that" link, not a parent-child link. And when the fraud spans three jurisdictions, a coordination case ties together the per-jurisdiction sub-investigations. That's three different relationship types — supersession, coordination, association — none of which fit a tree.

This is the problem the case dependency graph solves. It renders all active cases and their relationships as a force-directed graph — a layout where connected things cluster together and the overall topology reveals itself. Cases that block each other end up close. Supersession chains become visible. Cross-jurisdiction coordination shows the full picture instead of one branch at a time.

The visual metaphor matters. A tree implies hierarchy — someone is the parent, someone is the child. But "case A blocks case B" has no parent. "Case X supersedes case Y" doesn't nest Y inside X. "Case P coordinates cases Q, R, S across three repos" isn't a sub-case relationship — it's a coordination pattern. Force-directed layout treats all these as what they are: connections between peers, with the physics simulation naturally grouping related cases together.

We built this as a generic blocks-ui component — `<blocks-case-dependency-graph>` — because these relationship patterns aren't domain-specific. Insurance claims supersede each other. Clinical trials coordinate across sites. AML investigations link cases across institutions. The relationship types themselves are extensible through a registry: the component ships with parent-child, supersession, and coordination, but any CaseHub application can register types like `blocks`, `relates_to`, or `implements` with their own visual encoding.

What makes this more than a diagram is the interaction model. Click a case node and a detail panel shows what you need: the case description, who's assigned, when it was opened, and — critically — every relationship that case has, with the type and direction. Each relationship target is a link. Click "Fraud #3841" in the relationship list and the graph zooms to centre on that case, the detail panel updates, and you're looking at the prior investigation's context without having navigated away. An investigator traces the fraud from the original wire transfer through the supersession to the broader case, across to the coordination layer, and into each jurisdiction's sub-investigation — all by clicking links in the relationships list. The graph follows their attention.

The toolbar adds edge type filtering. Uncheck "Coordination" and the three jurisdiction links disappear — just the fraud chain and its peer association remain. Check only "Blocks" and you see the single dependency: Jurisdiction A blocking B pending evidence. Each filter combination is a different question about the same data.

The reference point was gastown-viewer-intent's dependency graph, which already renders exactly this kind of multi-type relationship network for development issues. The CaseHub version uses the same approach — D3 force simulation driving SVG directly — but consumes the platform's `GraphModel` from graph-core and uses the status registry for node colouring. Cases show up in their actual status colours. Edge types get distinct line styles and arrow markers. DOT export gives you the graph in a format any external tool can render.

The `entity-tree` still has its place. For drilling into one case's descendants — its workers, sub-cases, gates — a collapsible tree is the right tool. But for answering "what is connected to what, and how?" across the full landscape of active cases, you need a graph.
