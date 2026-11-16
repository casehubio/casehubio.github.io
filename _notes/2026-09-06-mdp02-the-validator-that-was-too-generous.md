---
layout: post
title: "The Validator That Was Too Generous"
date: 2026-09-06
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
tags: [cbr, validation, supersession, bulk-api]
series: issue-284-cbr-bulk-supersession
---

# The Validator That Was Too Generous

The ISP decomposition landed cleanly last session. What remained was the actual bulk API — the thing fsitrading needs for regime change detection. Five new SPI methods, CDI event notifications, and implementations across three backends (InMemory, JPA, Qdrant).

Implementation was mechanical. `supersedeMatching` finds cases by filter, marks them superseded. `reinstateMatching` reverses it. `supersedeAll` and `reinstateAll` take explicit ID lists. `findCaseIds` is the general-purpose query backing all of them. Each backend translates the same `CbrFilter` predicates to its native language — CopyOnWriteArrayList iteration for InMemory, JPQL for JPA, Qdrant scroll with `CbrQueryTranslator.applyStructuralFilters` for the vector store.

The return type change was the most consequential design call. `supersede` and `reinstate` returned `void` — callers had no way to know if the case existed, was already superseded, or actually transitioned state. Boolean returns let the notification decorator fire events only on real state transitions, preventing spurious `CbrCasesSuperseded` events for non-existent case IDs.

The interesting thing happened during code review. The ISP decomposition included a validator change — broadening `requireCategoricalList` to accept scalar `Categorical` fields, so `Contains("TRENDING")` would work on a single-valued regime field. The change renamed the method to `requireCategorical` and applied it to all five filter routing paths: `Contains`, `ContainsAll`, `ContainsAny`, `NotContains`, `NotContainsAny`.

Only `Contains` and `ContainsAny` got corresponding runtime support in `matchesSingleFilter`. The other three still only handled `StringListVal`. So `NotContains("X")` on a scalar `Categorical` with value `"Y"` would pass validation — the validator says it's legal — then silently return `false` at runtime because `StringVal` doesn't match the `StringListVal` pattern. The case gets excluded regardless of its actual value.

The semantics helped clarify the fix. `Contains` on a scalar means "field equals value" — sensible. `ContainsAny` means "field is one of these values" — also sensible. But `ContainsAll(["X", "Y"])` on a scalar can never be true (a single value can't contain two). And `NotContains` on a scalar is really "field is not X" — a valid operation, but one with different semantics than the list version. Rather than adding fragile runtime handling for edge cases, we narrowed the validator back: `Contains` and `ContainsAny` accept both field types; `ContainsAll`, `NotContains`, and `NotContainsAny` require `CategoricalList`.

The bug was invisible to testing because no test exercised `NotContains` on a scalar `Categorical`. It would have surfaced the first time a consumer tried it — silently wrong results, no error.
