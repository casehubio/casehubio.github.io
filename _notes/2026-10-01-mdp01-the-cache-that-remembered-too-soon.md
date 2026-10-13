---
layout: post
title: "The cache that remembered too soon"
date: 2026-10-01
entry_type: note
subtype: diary
projects: [casehubio/engine]
tags: [cbr, caching, bug-fix, feature-extraction]
series: issue-1124-cbr-lifetime-retrieval-timing
---

CBR retrieval with `CASE_LIFETIME` timing is meant to run once and cache the result — query the case store on first access, then return the same experiences for the life of the case. The problem: "first access" can fire before the case context has meaningful data.

When a `CaseContextChangedEvent` triggers rule evaluation, `CbrRetrievalService` extracts features from the WORKING layer via JQ expressions. If only some of those expressions resolve — because the working layer hasn't been populated yet — the feature vector is partial but non-empty. A partial vector still runs the query, gets low-similarity matches, and the result is cached. Every subsequent evaluation cycle returns that cached result, even after the missing data arrives.

The fix is to track completeness alongside extraction. `extractJqFeatures` now compares how many JQ expressions resolved against how many were configured. If they don't match, the result is returned but not cached — the next cycle gets another shot with more data. Once all features resolve, the result is cached normally.

It's a one-line gate change in `retrieveInternal`: cache only when `extraction.complete()` is true. The supporting change is a small `FeatureExtractionResult` record that carries both the feature map and the completeness flag.

What makes this worth noting: the existing code handled the empty case correctly. If *no* features resolve, it returns `CbrRetrievalResult.empty()` before reaching the cache. The bug only bit when features were *partially* populated — enough to produce results, not enough to produce good ones. The partial result slipped through the empty check and got locked in for the case lifetime.
