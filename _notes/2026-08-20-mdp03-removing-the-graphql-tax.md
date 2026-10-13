---
layout: post
title: "Removing the GraphQL Tax"
date: 2026-08-20
entry_type: note
subtype: diary
projects: [casehub-platform]
tags: [mcp, graphql, cdi, annotations, platform-api]
---

# Removing the GraphQL Tax

CaseHub's MCP integration works through `GraphQLModelScanner` — a CDI startup observer that discovers `@McpDomain` resolvers, builds an operation catalog, and registers it as the `casehub_action` tool. The scanner was written to piggyback on the GraphQL codegen pipeline: it expects `@GraphQLApi` on the resolver class and `@Query`/`@Mutation` on the methods.

This works fine for repos that actually want a GraphQL endpoint. But for apps that only want MCP domain registration — like the helpdesk example — it forces three unnecessary dependencies: `casehub-platform-graphql-generator` (annotation processor), `quarkus-smallrye-graphql` (full GraphQL runtime), and the Jandex Maven plugin (the processor reads Jandex indexes). That's a lot of ceremony for registering a handful of operations.

The platform API already had the right annotations. `@PlatformQuery` and `@PlatformMutation` have been in `platform-api` since the early MCP work — they're used by the codegen processor to mark methods on SPI interfaces. The scanner just never looked at them.

The fix is a second pass in `scan()`. After the existing `@GraphQLApi` loop, iterate the same CDI beans again and check their implemented interfaces for `@McpDomain`. For any match, scan the interface's declared methods for `@PlatformQuery`/`@PlatformMutation` and register them as operations. The `resolverClass` is the bean class (the impl), not the interface — so `Method.invoke` works through normal inheritance. If the same domain was already registered via the GraphQL path, skip it — backward compat is a `containsKey` check.

The refactoring was minimal. The existing `buildOperation` method extracted into `buildOperationFromMethod` that takes an explicit description string instead of reading `@Description`. The old method delegates to the new one. Everything else — `buildParams`, `readParamName`, `expandFields` — works unchanged on interface methods.

After this, an app registering an MCP domain needs only `casehub-platform-api` (already present) and `casehub-platform-mcp`. No codegen, no GraphQL runtime, no Jandex plugin.
