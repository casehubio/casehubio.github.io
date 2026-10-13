---
title: Two Tools, Not Sixty
date: 2026-08-14
author: mdp
entry_type: note
subtype: diary
tags: [mcp, graphql, platform, design]
status: draft
---

CaseHub has a tool proliferation problem. Qhorus alone exposes 37 MCP tools. Add connectors, IoT, openclaw, and drafthouse, and a unified CaseHub server puts 60+ tool definitions in front of an agent. Context wasted on descriptions it won't use. Tool selection accuracy degrades with scale.

The answer turned out to be two fixed tools: `casehub_model` and `casehub_action`. The agent calls `casehub_model()` to see what domains exist. Calls `casehub_model(domain="engine")` to see what operations the engine offers. Calls `casehub_action(domain="engine", operation="startCase", params={...})` to execute. Two tools instead of sixty. The Trellis pattern, applied to CaseHub.

The interesting design question was where the operation catalog comes from. The issue originally described a Quarkus build-time code generator — `@PlatformService` annotations on service interfaces, generating REST, GraphQL, MCP, and client code from a single source. I challenged this hard. The generator would cost 3000-5000 lines of Jandex scanning and bytecode generation to replace maybe 800 lines of hand-written adapters. Negative ROI for five service interfaces.

What killed the code generator wasn't the complexity alone — it was that the inputs already exist. GraphQL resolvers classify operations as `@Query` or `@Mutation`. They carry `@Description` annotations. Their method signatures define the parameter types. MicroProfile GraphQL annotations already describe exactly what the MCP model needs: what operations exist, what type each is, what parameters each takes, and what each does.

So the MCP model piggybacks on GraphQL. One annotation — `@McpDomain("engine")` — groups resolver classes into domains. A CDI startup bean scans all `@McpDomain`-annotated classes, reads their `@Query`/`@Mutation` methods, and builds the model automatically. Add a `@Query` method to a resolver, it appears in both the GraphQL schema and the MCP model. Zero drift by construction.

The dispatch path was the security-sensitive part. `casehub_action` needs to call resolver methods via reflection. Three constraints: only annotation-registered methods are invocable (the scanner filters at startup, not at call time). Resolver beans are obtained as CDI proxies via `CDI.current().select()` so interceptors fire. Jackson with `JavaTimeModule` handles parameter deserialization, aligned with SmallRye GraphQL's behaviour.

`ModelEnricher` is the optional layer — domains provide CDI beans annotated with `@McpDomain` that contribute live state. An agent calling `casehub_model()` sees not just operations but case counts and pending items. The enricher uses the same `@McpDomain` annotation as the resolvers — one identity mechanism, no string matching that could silently drift.

The per-domain GraphQL resolvers don't exist yet — they're tracked by separate issues (#892, #347, #190, #394). The infrastructure validates against test resolvers. When real resolvers arrive, they appear in the model automatically.

Also committed the ledger #189 work from the previous session — JPA annotations stripped from api/, orm.xml mappings in runtime/, domainData extension field. That was sitting uncommitted across the session boundary.
