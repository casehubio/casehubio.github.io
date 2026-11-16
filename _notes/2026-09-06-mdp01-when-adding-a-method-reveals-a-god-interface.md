---
layout: post
title: "When Adding a Method Reveals a God Interface"
date: 2026-09-06
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
tags: [cbr, interface-segregation, decorator, refactoring]
series: issue-284-cbr-bulk-supersession
---

# When Adding a Method Reveals a God Interface

The feature request was straightforward: add a bulk supersession API to `CbrCaseMemoryStore` so fsitrading's regime change detection could invalidate all cases from a prior market regime in one call. The existing `supersede(caseId, ...)` works for individual cases but requires the caller to enumerate every affected case ID — which means a full client-side scan.

I started proposing flat methods on the existing interface. Four new methods, same pattern as the existing ones. Then I stopped and looked at what `CbrCaseMemoryStore` had become: 14 methods mixing storage, retrieval, lifecycle management, and admin operations. Adding a filter-based bulk supersession — which combines querying (a retrieval concern) with state mutation (a lifecycle concern) — would make the mixing undeniable.

The right move was to decompose before adding. We extracted four role-based sub-interfaces: `CbrCaseStore` (write, erase, schema), `CbrCaseRetriever` (similarity search, scan), `CbrCaseLifecycle` (supersession, reinstatement, outcomes), and `CbrCaseAdmin` (tenant discovery, retention). `CbrCaseMemoryStore` becomes the composite extending all four. Backends still implement the composite. Decorators still wrap it. But consumers can now inject only what they need — fsitrading's regime handler injects `CbrCaseLifecycle`, not the 18-method composite.

The decorator refactoring surfaced a subtlety I hadn't anticipated. CDI §12.1 says abstract decorator classes work — the container auto-delegates unimplemented methods. So we made all 8 decorators abstract and deleted their forwarding stubs. Clean. Until the unit tests broke: they instantiate decorators directly with `new`, outside the CDI container. No container means no auto-delegation, which means `new FooDecorator(...) {}` creates an anonymous subclass that must implement every abstract method from the entire interface.

The fix was the classic delegation pattern: a `DelegatingCbrCaseMemoryStore` base class that centralises all forwarding. Decorators extend it and only override what they intercept. Tests work without CDI because the base provides the delegation. And when the bulk API adds 5 new methods, the forwarding stubs go in one place instead of 8.

The CbrFeatureValidator also needed attention. The regime use case requires filtering by a scalar `Categorical` field (`regime = "TRENDING"`), but the validator only accepted `Contains` filters on `CategoricalList` fields. The Qdrant backend already handled scalar categoricals correctly via `matchKeyword` — the validator was more restrictive than the backends it was supposed to protect. One condition change, and the InMemory and JPA `matchesSingleFilter` methods needed the same fix.

Batch 1 is landed. The ISP decomposition, delegating base class, decorator conversion, and validator fix are all committed and green. The actual bulk API — `supersedeMatching`, `reinstateMatching`, `findCaseIds`, and the CDI event notifications — comes next.
