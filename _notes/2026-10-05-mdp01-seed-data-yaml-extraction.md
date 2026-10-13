---
layout: post
title: "From Hardcoded Java to YAML — Externalising Nine Ref Implementations"
date: 2026-10-05
entry_type: note
subtype: diary
projects: [casehubio/connectors]
tags: [ref, simulation, yaml, jackson, seed-data]
series: issue-143-seed-data-yaml-simulation-corpus
---

# From Hardcoded Java to YAML — Externalising Nine Ref Implementations

The ref implementations in casehub-connectors have always been self-seeding — each `InMemory*Backend` hardcodes its test data in Java, directly in `seed()` methods or static fields. For standalone testing this works fine. For the simulation framework, it's a dead end.

The simulation layer needs seed data it can read — corpus files that strategies can replay, domain objects that both ref and simulation can consume from the same source. Hardcoded Java methods can't serve both purposes. Phase 2b of the ref/simulation unification extracts all that data to YAML.

## The shape of the extraction

Seven of the nine ref modules have seed data. Chat and calendar start empty — nothing to extract. The other seven follow the same refactoring:

A `SeedLoader` class per module reads YAML from `src/main/resources/seed/` via Jackson's `jackson-dataformat-yaml`. The existing `seed()` method (or `withTestData()` factory, in project-ref's case) delegates to the loader instead of building objects inline. The constructor still calls `seed()` — that was a deliberate design decision from Phase 1. `@PostConstruct` would break non-CDI test usage, and tests like `new InMemoryCommerceBackend()` need to keep working without a container.

The domain records are all Java records — Jackson deserialises YAML to them directly. No intermediate DTOs, no mapping code. Where the YAML structure doesn't match the domain record exactly (contacts need `LabelledValue` wrapping, locations need `Place` + `PlaceDetail` splitting from a unified seed format), a small seed-specific record bridges the gap.

## What caught us

Location-ref's `Review` record uses `timeMillis`. Commerce-ref's `ProductReview` uses `timestampMs`. Both are epoch millisecond timestamps. The YAML was written with `timestampMs` everywhere — compiled fine, failed at runtime with Jackson's `UnrecognizedPropertyException`. The error message is clear once you see it, but the wasted build cycle is the real cost. Cross-module record naming inconsistencies in domain models are the kind of trap that catches you exactly once.

Bank-ref was the interesting structural case. It's the only module that used `private static final` fields instead of a `seed()` method — three immutable `List.of()` and `Map.of()` constants. The refactoring changed these to instance fields populated from YAML. The branch audit caught that Jackson's deserialised `ArrayList` is mutable where the original `List.of()` was not — wrapping in `List.copyOf()` preserved the original contract.

## Simulation corpus files

The second half of the work: InvocationRecord-format YAML for every interpretive capability across six SPI modules. These are the operations where the ref's naive implementation (substring match, first-N-items) isn't realistic enough for simulation — search relevance, recommendation scoring, geocoding, directions. The corpus files provide realistic input/output pairs that simulation strategies can replay.

The format follows the established pattern from chat-spi and bank-spi. Each corpus file is keyed by SPI method name, with entries carrying a tenancy ID, input parameters, and expected output. Commerce got the richest coverage — three corpus files for ProductSearch and ProductDetails. Location needed five files to cover PlaceSearch, PlaceDetails, Geocoding, and Directions.

## What this enables

With seed data externalised and corpus files shipped, the simulation decorator has everything it needs to tag `DataRealism` levels on responses. A strategy-resolved search result from the corpus gets `DOMAIN_PLAUSIBLE`. A ref fallthrough for the same search gets `STRUCTURALLY_VALID`. A deterministic cart operation stays unmarked — it's authoritative, not simulated. Agent frameworks can now distinguish "this search result is realistic" from "this is just substring matching against eight products" and adjust their behaviour accordingly.
