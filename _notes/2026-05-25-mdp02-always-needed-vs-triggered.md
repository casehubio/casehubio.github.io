---
layout: post
title: "Always-Needed vs Triggered"
date: 2026-05-25
type: phase-update
entry_type: note
subtype: diary
projects: [CaseHub Work]
---

CLAUDE.md had grown to 51 KB. The warning threshold Claude Code checks at
session start is 40 KB — so every session was loading 11 KB of content that
was never relevant to the task.

The first pass was a line count exercise. We pruned the completed epics table,
merged three overlapping artifact tables into one, stripped integration module
sub-bullets the IDE can look up. Lines dropped from 587 to 528 — a 10%
reduction. The byte count didn't move.

That's the trap. CLAUDE.md files have a bimodal line-length structure: short
structural lines (headings, table rows, bullet markers) and long prose lines —
especially gotcha bullets, which run 200–300 characters each. Removing short
lines leaves the prose dense. `wc -l` lies. `wc -c` tells the truth.

## Timing, not importance

The gotchas section alone was 23 KB — 45% of the file. Fifty-seven bullets
covering extension build quirks, Hibernate reflection behaviour, CDI transaction
semantics, domain-specific API contracts. All loaded every session, regardless
of whether the work touched any of them.

Almost everything in a mature CLAUDE.md is important. The useful distinction
is timing: is this needed before the agent knows what task it's doing, or only
when it encounters a specific situation? A CDI observer gotcha matters enormously
when you're writing a CDI observer. It's dead weight when the session opens an
ADR or adjusts a config value.

We extracted the entire gotchas list to `docs/GOTCHAS.md`, categorised by
domain. The CLAUDE.md replacement is one line:

    Before writing any Java code: read `docs/GOTCHAS.md`.

The agent that knows where to look is as capable as one carrying the list
everywhere.

The same logic applied to Flyway migration conventions and build script
reference. The Project Structure section listed every class in the API and
runtime modules — 80 lines of per-file annotations. `ide_find_class` answers
that question in milliseconds. The listing was paying a context window cost
every session to avoid a search that doesn't even hurt.

Result: 51 KB → 15 KB.

## The cross-repo lesson

The tracking issue for applying this pattern across all casehubio repos went
into casehubio/work initially. That's wrong. A "review CLAUDE.md in every repo"
task is invisible to maintainers of other repos if it lives in one of them.
We moved it to casehubio/parent — the right inbox for platform-wide concerns.
If you want the parent org to act on something, file it where the parent org
looks.
