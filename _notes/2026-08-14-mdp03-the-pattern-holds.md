---
title: The Pattern Holds
date: 2026-08-14
author: Mark Proctor
type: diary
tags: [graphql, mcp, dual-mode, pattern]
issue: 405
---

Three domains down, two to go. Engine, work, and the typed client all have their
GraphQL + MCP modules built and tested. The pattern that emerged from the engine
work — `@GraphQLApi` plus `@McpDomain` on thin resolver adapters, auto-discovered
by the platform's `GraphQLModelScanner` at startup — held without modification
across all three.

That's the thing about good abstractions: the second and third consumers should
be boring. The engine module was the interesting one — it established the resolver
shape, the DTO convention (`from()` factories on records), the CDI event bridge
for subscriptions, and the `ModelEnricher` contract for MCP domain summaries. The
work module is structurally identical. Same annotations, same delegation pattern,
same test shape. The only differences are the domain types flowing through.

The MCP side is the part I didn't expect to be this clean. Two fixed tools —
`casehub_model` for navigating the operation catalog, `casehub_action` for
dispatching — and everything else is data. The `GraphQLModelScanner` walks CDI
beans at startup, finds `@McpDomain`-annotated resolvers, reflectively extracts
`@Query` and `@Mutation` methods, and builds an operation catalog. Adding a new
domain to MCP is literally adding an annotation to a class that already exists
for GraphQL.

Ledger and qhorus are next. If the pattern holds — and I expect it to — the
interesting work shifts to the scaffold module: wiring all four domains into a
single CaseHub server and proving it works end-to-end.
