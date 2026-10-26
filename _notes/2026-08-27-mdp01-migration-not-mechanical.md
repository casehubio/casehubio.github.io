---
title: "The Migration That Wasn't Mechanical"
date: 2026-08-27
entry_type: note
subtype: diary
projects: [casehubio/devtown]
tags: [graphql, quarkus, smallrye, sealed-interface, api-design]
series: issue-421-worker-reasoning-traces
---

# The Migration That Wasn't Mechanical

Continued from [The Reasoning Gap](2026-08-22-mdp13-the-reasoning-gap.md).

We'd shipped the reasoning traces across three repos — worker, engine, devtown. The final cleanup was migrating devtown's MCP tools from the old `@Tool` annotation pattern to `@McpDomain` resolvers backed by SmallRye GraphQL. Twenty-two methods, mostly one-line delegations. Split them into three resolver classes by domain concern — governance queries, governance mutations, memory queries. Delete the old monolith. Migrate the tests. Mechanical.

The test migration was mechanical. The GraphQL schema was not.

## SmallRye sees everything Jandex touches

The `@McpDomain` annotation composes with `@GraphQLApi`. That means SmallRye GraphQL walks every method signature — return types, parameter types, and their transitive fields — to build a schema. If a return type lives in a module that isn't in the Jandex index, the schema can't resolve it: `type Precedent not found in schema`.

The obvious fix: add the module to `quarkus.index-dependency`. And this is where it got interesting.

The domain module contains sealed interfaces:

```java
public sealed interface MergeOutcome {
    record Success(String mergeSha) implements MergeOutcome {}
    record Failure(String reason) implements MergeOutcome {}
}
```

SmallRye maps these as GraphQL object types. A sealed interface has no instance fields — only `permits` clauses and inner records. An object type with zero fields violates the GraphQL spec. The schema validation fails with `"Object" must define one or more fields` — and the error doesn't name which type is the problem.

I spent time binary-searching through the domain module's types before realising the sealed interfaces were the culprit. There were six of them: `MergeOutcome`, `RevertOutcome`, `BatchBranchOutcome`, `CoordinatedMergeResult`, `ReviewerOutcome`, `PrLifecycleEvent`. None of them are referenced in any resolver method signature. They're just *present in the indexed module*, and SmallRye discovers them anyway.

## The right fix was already in front of me

Every other mutation in the resolver already returned a local record type — `RetryResult`, `EnqueueResult`, `RerouteResult`. The `reportIncident` method was the exception: it returned `IncidentFeedbackResult` from the domain module. The query resolver had a similar issue with `ReviewDetail` containing `PrPayload` from the review module.

The fix was to wrap these in local records:

```java
public record IncidentReport(UUID caseId, int attestationsWritten,
                             List<FlaggedAgentEntry> flaggedAgents) {}
```

Map from the domain type in the method body, return the local type in the schema. No module indexing needed. The GraphQL schema stays self-contained in the app module.

This is actually better API design than what I was trying to do. Indexing external modules couples the public schema to internal domain model evolution. A sealed interface added to the domain module for internal pattern matching shouldn't break the API layer. The wrapper records are a natural boundary — the same kind of boundary that REST DTOs provide, just applied to GraphQL.

The "mechanical" migration exposed a design principle I should have seen from the start: the schema is a public contract, and public contracts shouldn't leak internal types.
