---
layout: post
title: "The Audit That Got Everything Wrong"
date: 2026-08-19
entry_type: note
subtype: diary
projects: [casehub-neocortex]
tags: [cdi, quarkus, audit, flyway]
---

A CDI audit flagged five classes in neocortex for unnecessary CDI annotations. The criterion: zero injections. The class doesn't `@Inject` anything, so CDI must be unnecessary. Five out of five were wrong.

The audit checked one direction — does this class consume beans? — and missed the other — is this class a bean others consume? CDI serves both roles. A class with zero injections can still be a legitimate CDI bean if it exists to be discovered, injected, or displaced by the container.

Three of the five had active CDI consumers. `InMemoryQueryExpander` is `@Inject`ed in a `@QuarkusTest` that verifies the decorator wiring. `YamlFrontmatterExtractor` is a `@DefaultBean` that `CorpusBindingProducer` injects via CDI constructor injection — remove the annotation and there's no fallback `MetadataExtractor` when no alternative is on the classpath. `TemplateQueryExpander` has an `@Inject` constructor taking `ExpansionConfig` — the audit said "one injection" but still flagged it for removal.

The other two — `InMemoryRelevanceEvaluator` and `InMemoryCursorStore` — are `@Alternative @Priority(1)` test stubs in `rag-testing`, a published module. No `@QuarkusTest` in this repo currently injects them, so by the audit's logic they're unnecessary. But the CDI annotations are their API contract. Downstream consumers add `rag-testing` to the test classpath expecting these beans to displace production implementations. Removing the annotations is a breaking change for consumers who don't appear in this repo's reference search.

The "zero injections" heuristic is reasonable-sounding. It breaks on published test-support modules, `@DefaultBean` fallbacks, and `@IfBuildProperty`-gated beans. The lesson isn't that automated audits are useless — it's that the right criterion is "zero CDI interactions," checking both directions.

On the mechanical side, neocortex had five incremental Flyway migrations in `memory-cbr-jpa` (V1 through V5) despite having no production database. Each migration added columns from the CBR feature development: outcome tracking, supersession, scope, reinstated_at. With no deployed schema to migrate from, the incremental history serves no purpose. Consolidated them into a single `V1__initial_schema.sql` with the final 23-column schema, and normalised `memory-jpa`'s orphaned `V1000` to `V1` to match every other module.
