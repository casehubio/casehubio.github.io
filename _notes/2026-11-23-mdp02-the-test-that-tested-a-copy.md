---
layout: post
title: "The Test That Tested a Copy"
date: 2026-11-23
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
tags: [testing, test-design, cdi, integration-testing, code-review]
series: issue-1205-coping-bias-assertion
---

# The Test That Tested a Copy

A test that duplicates production logic is a bet against yourself. You're betting that you'll remember to update both copies when the logic changes — and that the test, which exists specifically to catch bugs in that logic, will somehow catch bugs in its own duplicate of the same logic.

`SituationalSnapshotAccessorProducerTest` had a `buildAccessor()` helper — thirty lines of code that mapped `PersonalityState` fields into a `SituationalPersonalitySnapshot`. It was a line-for-line copy of the producer method it was testing. Six scenarios exercised this copy: intensities, winners, facet shifts, published counts, absent state, null winners. All green. All testing the helper, not the producer.

The integration test from #1193 did the same six scenarios plus one, but wired through the real `SituationalSnapshotAccessorProducer` via a mocked `Instance<PersonalityStateService>`. Same assertions. Same state transitions. Different plumbing — the kind that actually catches drift.

The original test existed because the producer uses CDI field injection, and the initial author didn't want CDI in a unit test. The mocked `Instance` approach solves that — it's still a unit test, it's still fast, and it exercises the actual code path. Once that test exists, the duplicate-logic version isn't a safety net. It's a liability: if the producer's mapping changes, the helper needs to change too, and the test will pass either way because it's testing its own assumptions.

Delete the copy. Keep the one that tests the real thing.
