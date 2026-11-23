---
layout: post
title: "The Eleven That Hid Behind One"
date: 2026-11-23
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
tags: [code-style, imports, code-review, consistency]
series: issue-1205-coping-bias-assertion
---

# The Eleven That Hid Behind One

Continued from [The Test That Tested a Copy](2026-11-23-mdp02-the-test-that-tested-a-copy.md).

The previous session standardised `Duration` imports across four test files. Code review of that change flagged something in `InnerLifeOrchestratorTest`: after the fix, `Duration` was properly imported — but `Optional`, `Instant`, `Map`, `DriveAxis`, `DriveIntensity`, `SessionPipeline`, `Arrays`, `Uni`, `AtomicBoolean`, and `CognitionPromptRenderer` were all still fully qualified in the method bodies. Eleven types hiding behind the one that got fixed.

The file compiled. The tests passed. Nothing was wrong. But the import block told one story — "this file uses proper imports" — while the method bodies told another. The fixed `Duration` line made it worse: now the file looked *partially intentional*, as though somebody had deliberately chosen to import `Duration` but write `java.util.Optional.of(driveProfile)` inline. That reads like a conscious style choice, not an oversight.

This is the problem with partial style fixes. A file that is consistently messy is easy to read as "needs cleanup." A file that is half-fixed looks like it was reviewed and left that way on purpose. The inconsistency creates a false signal about intent.

The fix was mechanical — add ten imports, replace every fully qualified reference, verify one stray `java.time.Duration` that persisted despite being imported. The interesting part isn't the fix. It's the discovery pattern: fixing one thing made the remaining eleven visible in a way they hadn't been before. Code review of a narrow change is often the first time anyone looks at the surrounding code with fresh eyes.
