---
layout: post
title: "The 35 chapters DESIGN.md didn't know about"
date: 2026-06-04
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-work]
tags: [arc42stories, documentation, architecture, casehub]
---

I went in thinking this was a documentation restructuring job. DESIGN.md had 27 numbered phases; the plan was to map those phases to ARC42STORIES.MD chapters, write the layer entries from code, delete the old files, done. Probably a day.

The first thing that stopped me was the chapter count. Going through the git history systematically rather than just reading the phase log, we found 35 distinct deliverable chapters — eight more than DESIGN.md tracked. The 2026-04-20 session alone had delivered WorkItemTemplate, WorkItemNote, WorkItemRelation graph, SSE live stream, recurring schedules, Micrometer metrics, and bulk inbox operations. All real features. None of them in any numbered phase. An entire session's worth of production code with no DESIGN.md entry — because the DESIGN.md was written to describe a plan, and the plan kept getting richer than the document tracking it.

The archaeology forced a lesson: when I'm trying to reconstitute what was built, the source of truth is the git log, not the documentation written alongside it.

The second thing worth noting was the foundation tier / application tier split. The CaseHub Profile for ARC42STORIES.MD has two sections — one for harness apps, one for foundation modules. The harness app section includes a prescribed layer taxonomy (domain baseline → casehub-work → casehub-qhorus → casehub-ledger → casehub-engine). I had to be explicit that this taxonomy describes how harness *consumers* integrate casehub-work's layers, not how casehub-work documents its own internals. A foundation module defines its own layer taxonomy. casehub-work ended up with L1 Domain Baseline / L2 REST API / L3 Lifecycle Engine / L4 Label System / L5 Ledger Integration / L6 Distribution / L7 Optional Modules — none of which are in the harness profile.

The layer entries were where the code-first discipline mattered most. We ran `find` before writing every Key files section, and it caught real errors. LabelVocabulary and LabelDefinition live in the `runtime` module, not `queues` — the queues module depends on runtime, not the other way around. FilterChain turned out to be an inverse index (filterId → Set of workItemIds), not a cascade-delete structure from QueueView as the name suggests. The JQ evaluator class is `JqConditionEvaluator`, which injects `JQEvaluator` from `casehub-platform-expression` — a completely different class from what I'd assumed. `LedgerEventCapture` is in `ledger/service/`, not the root ledger package. `ExpiryCleanupJob` uses `@Scheduled(every=...)` not a cron expression.

Five corrections in seven layer entries. Every one caught because we looked at actual source before writing rather than templating from what the names implied.

Three-check quality gate ran at the end: issue #39 still open, all Key file classes verified to exist, all CDI annotations matched production. DESIGN.md and ARCHITECTURE.md deleted. 1861 lines in the final document.

The thing I keep thinking about: DESIGN.md was always described as the "implementation tracker," but what it was really tracking was the plan — the intended phases. The gap between plan and what actually shipped grew silently over eight weeks. The architecture document is now the codebase's actual history, not its intended one.
