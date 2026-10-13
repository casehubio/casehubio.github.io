---
title: "The Last Repo and the BOM That Broke Everything"
author: Mark Proctor
date: 2026-09-14
tags: [architecture, maven, spring-boot, quarkus, dual-framework, connectors]
status: draft
entry_type: note
subtype: diary
series: dual-framework-extraction
---

# The Last Repo and the BOM That Broke Everything

Connectors was the last repo in the dual-framework extraction queue. Eight repos, four batches, and the pattern had been refined enough that the extraction itself was mechanical. What wasn't mechanical was the naming problem — and the BOM interaction that nearly derailed the finish line.

## The naming collision

The connectors repo already had a module called `core` with the artifact ID `casehub-connectors-core`. The established extraction pattern names the framework-neutral module `<repo>-core`. When "core" is already taken, you can't follow the pattern literally.

The solution was `connectors-api` — a new zero-dependency module containing all the SPIs, records, and stripped service POJOs. The existing `core` module kept its artifact name (zero change for Quarkus consumers) and became thin CDI wiring: a `ConnectorsBeans` class with `@Produces` methods delegating to the POJOs. It's the same pattern as every other extraction, just with a different name for the neutral layer.

## Seventeen modules, one pattern

The connectors repo has seventeen modules — Slack, Discord, Teams, WhatsApp, email, IRC, calendar, notification bridge, MCP tools, GraphQL. Most had only one or two CDI-coupled classes. The extraction was the same move repeated: strip `@ApplicationScoped` and `@ConfigProperty` field injection, convert to constructor parameters, add a `*Beans` class for CDI wiring.

Category C files — MCP `@Tool` annotations, `@ObservesAsync` event bridges, JAX-RS endpoints — stayed as-is. Their framework coupling isn't incidental; it's their purpose.

## The Groovy surprise

The Spring auto-configuration module compiled fine. All twenty modules compiled fine. Then the Quarkus tests broke.

Three `@QuarkusTest` GET-handler tests failed with a `NullPointerException` deep inside Groovy's `ClosureMetaClass.invokeOnDelegationObject`. POST tests in the same suite passed. The stack trace mentioned nothing about Spring, Maven, or dependency versions — it looked like a Groovy/Java 26 compatibility bug.

The cause was the Spring Boot BOM. I'd imported it in the connectors parent pom alongside the Quarkus BOM, following the approach that had worked in other repos. But Spring Boot 4.1.0 manages a Groovy version that breaks rest-assured's closure-based GET dispatch on Java 26. The Quarkus BOM doesn't manage Groovy at all, so "Quarkus BOM first" — the fix that works for JUnit version conflicts — doesn't help here. Spring's Groovy version wins uncontested.

The fix is architectural, not ordering: the Spring Boot BOM belongs only in the module that actually needs Spring dependencies. Parent-level imports leak version management into modules that never asked for it.

## The pattern for multi-framework Maven projects

This sharpens a rule that was forming across the extraction work. Each repo that added a `-spring` module imported the Spring Boot BOM differently — engine and qhorus put it in the child repo's parent pom (after the Quarkus BOM), blocks put it in the parent pom. The connectors failure proves that parent-level import is wrong even when the Quarkus BOM comes first. The only safe scope is the Spring module's own pom.

The reason is subtle: Maven's "first wins" rule only applies when both BOMs manage the same artifact. When Spring manages something Quarkus doesn't — Groovy, in this case — there's no conflict to resolve. Spring's version just wins silently, affecting every module in the reactor.

## Eight repos, one epic

The dual-framework extraction is done. Eight repos extracted across four batches: engine, work, ledger, eidos, neocortex, qhorus, blocks, connectors. Each has a framework-neutral core module, a Quarkus wiring layer, and a Spring auto-configuration module. The `spring-generator` Maven plugin handles the roughly eighty percent of mappings that are mechanical; the remaining twenty percent are hand-written where CDI patterns don't have a direct Spring equivalent.

What started as "can casehub run on Spring Boot?" turned into a systematic separation of business logic from framework wiring across the entire platform. The business logic is genuinely framework-neutral now — pure Java POJOs with constructor injection. The framework choice is a deployment decision, not an architectural one.
