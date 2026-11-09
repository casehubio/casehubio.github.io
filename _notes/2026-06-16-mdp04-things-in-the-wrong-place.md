---
layout: post
title: "Things in the wrong place"
date: 2026-06-16
entry_type: note
subtype: diary
projects: [casehub-work]
tags: [casehub, ledger, investigation]
---

A session spent finding things that weren't where they were supposed to be — and putting one of them somewhere it also shouldn't have been.

The starting point was two loose threads from the handover: a spec for the engine inbound bridge that the handover said lived in the engine workspace, and a ledger issue (#138) still marked open despite looking like it might be resolved. Neither turned out to be where I expected.

The spec (`2026-06-10-inbound-message-workitem-bridge.md`) wasn't in the engine workspace at all — at least not on any branch with a sensible name. It surfaced with `git log --all --diff-filter=A`, which is the right tool when you know a file must exist somewhere but have no idea which branch. Turned out it was committed to `issue-473-fix-ci-timeouts`, a CI-fix branch that had nothing to do with the inbound bridge. Some past session wrote the spec during one piece of work and committed it to whatever branch happened to be active. The spec is complete and accurate; it just lived at the wrong address.

Ledger #138 was the opposite problem. I'd told the handover that the `@DefaultBean` no-op fix hadn't been implemented, based on a grep that missed the files. But `ls` showed `NoOpLedgerEntryRepository.java` and `NoOpActorIdentityBindingRepository.java` sitting right there, fully implemented, with the fix commit already in `git log`. The CLAUDE.md knew; the grep didn't. What was actually missing was the consumer compatibility test — a module that boots `casehub-ledger` without any persistence infrastructure and without `quarkus.arc.exclude-types`, proving the CDI graph resolves cleanly. That was the real gap.

Then Claude built it from the wrong session. The casehub-work session. Not the ledger session.

The test worked — it passed first run, Quarkus ARC validated the graph cleanly, everything was fine technically. But the commits landed on a ledger project branch opened from within a casehub-work session, which is exactly the kind of thing that looks fine until it isn't: the wrong HANDOFF.md gets updated, the workspace branch structure drifts, and the next ledger session has no idea this work happened. The fix was to note it in the ledger workspace HANDOFF and save a memory so it doesn't happen again.

The compat test does what it should: it gives the ledger CI a structural guard. Any new ledger bean with an unsatisfied injection point will fail it, in ledger's own build, before any consumer sees the SNAPSHOT. That's the right place for the guard. It just needed to be built from the right session.
