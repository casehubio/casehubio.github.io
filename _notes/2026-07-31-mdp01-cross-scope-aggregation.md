---
layout: post
title: "Cross-Scope Aggregation: When Sites Stop Being Independent"
date: 2026-07-31
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-clinical]
tags: [cbr, dsmb, aggregation, safety-signals, ledger]
series: issue-120-cbr-multi-scope-dsmb
---

*Part of a series on [#120 — CBR Phase 7](https://github.com/casehubio/clinical/issues/120). Previous: [Multi-Scope CBR: The Wiring Session](2026-07-30-mdp01-multi-scope-cbr-wiring.md).*

## Cross-Scope Aggregation: When Sites Stop Being Independent

The previous session wired scope-tagged storage and scope-aware retrieval — individual CBR cases now carry their position in the trial/site/patient hierarchy. This session builds the thing that hierarchy exists to serve: trial-level safety signal detection from site-level data.

The core question is straightforward. A single site reporting Grade 3 hepatotoxicity is a site-level concern. Three of five sites reporting it is a DSMB-level signal. The difference isn't the individual events — it's the pattern across sites that no site-level agent can see.

`TrialSafetyAggregationJob` runs periodically, scanning each active trial's sites for two signal types. Grade threshold fires when a configurable fraction of sites exceed a high-grade AE rate — the default is 3+ sites above 10% Grade 3+ rate. Cross-site cluster fires when the same event type appears at 3+ sites regardless of grade. Both are deliberately simple thresholds; the value isn't in the detection algorithm, it's in the fact that detection happens at all and feeds into the CBR and audit trail.

Each detected signal gets three things: a trial-scope CBR case in the new `clinical-trial-safety` domain (so future DSMB reviews can retrieve prior signal patterns), a `TrialSafetySignal` entity tracking when the signal was first detected and whether it's been resolved, and a `DsmbSafetySignalLedgerEntry` in the tamper-evident audit trail with an EU AI Act Art.12 compliance supplement. The signal also fires a CDI event for downstream consumers — DSMB WorkItem creation being the obvious next step, deferred to a follow-up issue.

The interesting design choice is signal lifecycle. Signals aren't one-shot alerts — they persist across aggregation runs. If the same signal is detected again, the existing record is updated (affected site count may change, last-detected timestamp advances). If a previously-detected signal drops below threshold, it's marked resolved. This means the `trial_safety_signal` table becomes a running record of the trial's safety posture over time, not just a log of point-in-time detections.

## The Worktree Gotcha

I hit a tooling issue worth documenting. IntelliJ MCP's `ide_create_file` targets the `project_path` parameter, not the session's working directory. I was working in a git worktree (`worktrees/53/clinical`) but `project_path` pointed to the main repo (`/clinical`). Seven new Java files and five edits all landed silently in the wrong repo. No error, no warning — IntelliJ even compiled clean because it could see the files in the main repo's tree. The only signal was `git status` in the worktree showing nothing.

The fix is to call `ide_open_project` with the worktree path before any file operations. Submitted this to the garden as GE-20260731-e74510.

## What This Opens Up

The aggregation job is the last piece of the cross-scope CBR story. Patient-level AE cases feed site-level patterns which feed trial-level signals — the hierarchy is wired end to end. The deferred piece is hooking `DsmbSafetySignalEvent` to WorkItem creation, giving the DSMB a concrete task to review when a signal fires. That's a natural Layer 6-style integration, not CBR work.

The broader pattern — periodic batch detection producing both CBR cases and audit entries — could extend to other aggregate signals: enrollment velocity anomalies, protocol deviation clustering, consent withdrawal patterns. The schema and job structure are deliberately generic enough to replicate.
