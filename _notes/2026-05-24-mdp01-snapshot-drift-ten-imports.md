---
layout: post
title: "SNAPSHOT Drift: Ten Imports, Two Lessons"
date: 2026-05-24
entry_type: note
subtype: diary
projects: [casehub-work]
tags: [quarkus, maven, flyway, git]
---

The CI failure that kicked off this session was a compile error in `casehub-work-ledger`: `cannot find symbol: class ActorType, location: package io.casehub.ledger.api.model`. The cause took thirty seconds to find — `casehub-ledger#88` had moved `ActorType` and `ActorTypeResolver` from that package to `io.casehub.platform.api.identity`. Both types are shared across multiple repos and belong in the platform module. Our DTOs and audit service still had the old import.

The fix should have been: grep the whole codebase for `io.casehub.ledger.api.model.ActorType`, fix everything at once, run all tests, push once. What I actually did was fix the ledger module, push, watch CI fail in examples, fix examples, push, watch CI fail again on a transient Testcontainers timeout. Claude grepped for the old package path after the first round and found the remaining two files immediately. Eight imports in the ledger module, two in examples — ten total across three pushes when one would have done. When a package moves, grep is the blast radius tool. CI is not.

The second break was subtler. `casehub-ledger#95` had moved the base Flyway migrations from `classpath:db/migration` to `classpath:db/ledger/migration`. Every consumer's test `application.properties` needed both paths, otherwise V2001+ migrations fail at startup with `Table "LEDGER_ENTRY" not found`. The error looks like a migration ordering problem. It's actually a classpath gap — the base tables (`ledger_entry`, `ledger_attestation`) were never created because Flyway didn't scan the right location.

```properties
quarkus.flyway.locations=db/migration,db/ledger/migration
```

What made this hard to catch locally: Maven had already resolved and cached the old SNAPSHOT. Tests passed. CI pulled a fresh copy and hit it immediately. Classic SNAPSHOT drift scenario — the local cache confirms a state that no longer exists upstream.

There was one other mistake worth recording. While updating `PLATFORM.md` in the parent repo, we were on `issue-60-sync-all-repo-lists`, not main. I ran `git push upstream main`. It appeared to succeed — `393ca03..d500aa3 main -> main`. What it pushed was local main, not the current branch. Git did exactly what I asked. The SHA range in the output was the tell: it showed where local main was, not where the current branch was. The fix was a cherry-pick to local main. It's in the garden now.

Full test suite green locally. CI confirmed clean on the rerun.
