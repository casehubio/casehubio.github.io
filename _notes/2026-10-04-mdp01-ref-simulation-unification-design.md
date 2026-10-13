---
title: "Two Layers, One System: Designing the Ref/Simulation Unification"
date: 2026-10-04
author: mdp
entry_type: note
subtype: diary
projects: [casehubio/connectors]
series: issue-138-design-unify-ref-simulation
tags: [simulation, ref-implementations, architecture, design]
status: draft
---

The connectors repo has nine ref implementations and a simulation framework that solve overlapping problems. Ref implementations are stateful in-memory services — you can create a contact, list contacts, and the created one appears. The simulation framework wraps SPIs with strategy-driven responses — corpus-backed, configurable, realistic. Both exist. Neither knows about the other.

The overlap has been obvious for a while. A ref's search operation returns hardcoded London restaurants regardless of what you searched for. A simulation strategy can return contextually relevant results but can't track state across calls. The question that triggered [#138](https://github.com/casehubio/connectors/issues/138) was straightforward: can we make them work as layers of a single system instead of independent alternatives?

## The Interesting Decision

The design brainstorm surfaced a genuine tension. I initially saw simulation playing a broad role — more operations delegated, more capability coverage. Claude pushed back from first principles: every capability moved to simulation is one that stops working without simulation config. The nine refs currently work standalone with zero configuration. That's a real feature — [#137](https://github.com/casehubio/connectors/issues/137)'s QuarkusTest CDI wiring test demonstrates it. If you hollow out the ref to make simulation primary, every consumer pays a setup tax.

The resolution: ref stays complete standalone. Simulation enhances, doesn't replace. The ref's naive search (substring match, first-N results) is structurally correct — it returns the right shapes, the right types. When you need realistic responses, simulation strategies provide them. When you don't care about realism (most tests), the ref handles everything.

This split maps cleanly onto the capability classification that fell out of the design. Operations divide into deterministic (CRUD, lifecycle — ref is authoritative) and interpretive (search, recommendations, scoring — ref works but is naive). ChatPlatform, BankPlatform, and CalendarPlatform are fully deterministic. LocationPlatform is fully interpretive — all four capabilities need interpretation. The rest are mixed.

## What the Reviews Caught

Claude ran adversarial decision review against the nine captured decisions and found a real problem: we'd proposed a "unified corpus" where ref seed data IS simulation corpus data, stored as `InvocationRecord<I,O>` pairs. The reviewer pointed out that `InvocationRecord` is an immutable request-response mapping. You can't express "after `addToCart` twice, `viewCart` returns both items" in that format — mutable state sequences don't fit. The revised decision: shared seed *format* (one set of YAML files), separate runtime *models* (ref populates its domain state, simulation populates InvocationRecords).

The post-spec review caught factual errors I'd missed. The `DataRealism` enum — which the spec proposed as the mechanism for distinguishing naive from realistic responses — has different values than what the spec used. It's `GARBAGE, PLACEHOLDER, STRUCTURALLY_VALID, DOMAIN_PLAUSIBLE, RECORDED_REAL`, not `REALISTIC` and `SYNTHETIC`. And it's in `simulation-api`, not `platform-api`. And it's currently unused — a dead enum, defined but never referenced. The spec was designing against an API that didn't exist in the form we described. Each of these required spec corrections before the design could be credible.

The `paginate()` signature was wrong too — the spec proposed `List<T> paginate(List<T>, int, int)` when the actual contract is cursor-based: `Page<T> paginate(List<T>, PageRequest)`. The CDI wiring characterisation had the majority and minority patterns inverted. Getting the details right matters more for a spec like this than for code — the spec is the thing three separate implementation phases will build against.

## Phase 1 — Normalisation

With the design solid, Phase 1 normalised all nine refs to consistent patterns: extracted the duplicated `paginate()` into a shared `PaginationHelper` in `connectors-api`, standardised `supports()` from chained `==` to `Set.of()` constants, migrated three backends from direct construction to `@DefaultBean @ApplicationScoped`, and fixed `DocumentPlatform`'s missing `capabilities` attribute on `@SimulationEligible`.

One thing bit us during the CDI migration. The plan called for moving `seed()` from the constructor to `@PostConstruct`. Clean in theory — CDI manages the lifecycle. But the tests create backends with `new InMemoryContactsBackend()` directly, bypassing CDI. `@PostConstruct` doesn't fire outside CDI, so every test that constructs a backend directly gets an empty, unseeded instance. The fix: keep the constructor calling `seed()`, which works in both CDI and non-CDI contexts. CDI calls the no-arg constructor, which calls `seed()`. Tests call the no-arg constructor, which calls `seed()`. Sometimes the obvious migration path breaks the secondary use case.

## What's Next

Phase 2a (platform repo) adds DataRealism tagging to the simulation decorator generator — replacing the existing `boolean simulated` with proper quality-spectrum levels. Phase 2b (connectors repo) extracts hardcoded seed data into YAML files and ships default simulation corpus files for interpretive capabilities. Both are separate issues, blocked on each other in sequence.

The broader question — [#140](https://github.com/casehubio/connectors/issues/140), filed as a side note during the design session — is whether production search should lean on specialised NLP resources (synonym dictionaries, taxonomies, phonetic matching) rather than treating search as purely an LLM-interpretation problem. That's a platform architecture question, not a simulation one, but it was surfaced by thinking carefully about what "interpretive" actually means.
