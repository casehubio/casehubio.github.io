---
layout: post
title: "Two Tools, Not Sixty"
date: 2026-08-20
entry_type: note
subtype: diary
projects: [casehub-platform]
tags: [mcp, tool-schema, json-schema, quarkus-mcp-server]
---

The hierarchical MCP model uses two tools — `casehub_model` for navigation, `casehub_action` for dispatch. That design holds. But until now, `casehub_action` had a static schema: three string parameters, no guidance about what values are valid. An LLM calling it had to first query `casehub_model`, read the response, hold that in context, and construct the right call. When the model output scrolled out of context in a long conversation, it guessed — and got it wrong. Validation errors included the expected schema so the LLM could self-correct on retry, but that's reactive. The schema should be proactive: always visible in the tool definition, always in the system prompt.

The fix is straightforward in concept. After `GraphQLModelScanner` discovers all the `@McpDomain` resolvers at startup, build a JSON Schema that encodes the operation catalog directly into `casehub_action`'s tool definition. Domain names become a `string` enum. Operations get listed per domain. The LLM sees valid values before generating parameters, not after getting them wrong.

Two schema modes, selectable by config (`casehub.mcp.schema-mode`). `simple` puts the operation catalog in the `description` field — works with any client, no JSON Schema sophistication required. `rich` uses `if-then` conditional schemas: pick domain `"engine"`, and the `operation` field constrains to `["cases", "startCase", "signalCase", ...]`. More precise, but only useful if the client actually parses conditional JSON Schema. Default is `simple`.

The interesting part was the registration mechanism. `casehub_action` was a `@Tool`-annotated method — static, schema auto-generated from Java parameter types. To inject a dynamic schema, we needed `ToolManager.newTool()`, the programmatic registration API in quarkus-mcp-server 1.11.1. The API is clean: `setInputSchema(Object)` accepts a `Map<String, Object>` that becomes the tool's JSON Schema. `register()` fires `notifications/tools/list_changed` automatically. One call after the scan completes, and every connected client sees the updated tool definition.

We hit one unexpected wall. The original `@Tool` method lived on a class annotated `@McpServer("casehub")` — a named server. The natural move was `setServerName("casehub")` on the programmatic tool. It compiled fine. At startup: `IllegalStateException: Invalid server name: casehub`. The named server exists, the config is present, `@Tool` methods on the same server work — but the ToolManager's runtime validation uses a different server name registry than the annotation processor. Programmatic tools can only register on the default server. The fix is to omit `setServerName()` entirely; in single-server deployments, all tools share the same HTTP endpoint regardless of logical server grouping.

The architecture splits cleanly. `GraphQLModelScanner` fires a `ModelScanComplete` CDI event after scanning. `DynamicToolRegistrar` observes it, builds the schema via `McpSchemaBuilder` (a pure function — no CDI, easy to unit test), and registers the tool. `CaseHubMcpTools` keeps `casehub_model` as a `@Tool` annotation. The dispatch logic in `ReflectiveOperationDispatcher` is unchanged — the handler in the registrar delegates to it the same way the old `@Tool` method did.

What this opens up is runtime schema updates. When a domain registers at runtime (not just at startup), the registrar can rebuild the schema and re-register. The ToolManager API supports `removeTool()` for programmatic tools, so the pattern would be: remove, rebuild, re-register. That's future work — but the architecture doesn't need to change to support it.
