---
layout: note
title: "The second adopter"
date: 2026-07-07
projects: [casehub-work]
tags: [platform-convention, domain-model, types-labels]
entry_type: note
---

A platform convention isn't real until the second project adopts it. The first
adopter shaped it; the second one proves it generalises.

CaseDefinition got `types: Set<Path>` and `labels: Set<Path>` a couple of days
ago. The convention says every definable entity carries both — types for
behavioural contracts (routing, dispatch, evaluation), labels for operational
classification (queues, dashboards, analytics). Both use the same `Path` from
`casehub-platform-api`, with ancestor matching via `isAncestorOf()`.

WorkItem was the test. If the convention didn't survive contact with a JPA entity
that has 30+ fields, three persistence backends, and twelve modules consuming it,
it would need rethinking.

It survived — but not without interesting friction. CaseDefinition is an in-memory
model. It holds `Set<Path>` directly. WorkItem is a JPA entity. It needs a join
table, an embeddable, a unique constraint to enforce set semantics at the database
layer, and `LinkedHashSet` to keep insertion order deterministic for serialisation.
The definition layer (WorkItemTemplate) stores types as a JSON text column —
identical to how it already stores label paths. The instance layer (WorkItem) uses
`@ElementCollection`. Same concept, different persistence reality.

The `category` field was the interesting decision. WorkItem had `category` as a
single flat string — `"approval"`, `"review"`. The engine spec explicitly said
categories are subsumed by types. Replacing a single string with `Set<Path>` on a
field that has 49 call sites across twelve modules is the kind of change you only
make when you believe the convention is right. Every caller breaks. Every caller
must think about whether it should be using a specific type path or the full set.
The breakage is the point — it forces every consumer to be explicit.

Path validation needed two entry points. Template instantiation validates via
`Path.parse()` when parsing `typePaths` JSON — malformed paths fail at
instantiation time, not at query time. But direct WorkItem creation via the SPI
bypasses templates entirely. The design review caught this: a second validation
gate in `WorkItemService.create()` closes the bypass. Both paths now reject
malformed paths before they reach the database.

Ancestor matching was the other piece that needed thought. `type("compliance")`
should match WorkItems typed `compliance/audit`. In-memory stores use
`Path.isAncestorOf()` directly. JPA uses `LIKE :type || '/%'` — the path index
makes it fast. MongoDB uses `$regex` with `Pattern.quote()` for safety. Three
backends, three query dialects, same semantics.

The protocol file — formalising "every definable entity carries types and labels"
— is tracked as a garden issue. Once it's written, the next repo that adds a
definable entity has a standing rule to follow rather than a pattern to reverse-
engineer from two examples.
