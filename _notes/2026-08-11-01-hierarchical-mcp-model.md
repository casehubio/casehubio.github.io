---
title: "Your MCP Server Has Too Many Tools"
date: 2026-08-11
type: article
subtype: explanation
tags: [mcp, architecture, ai-agents, quarkus]
---

# Your MCP Server Has Too Many Tools

Every MCP tool you register costs context. The tool name, description, parameter schema, and type definitions all land in the agent's prompt before it reasons about your request. For a server with ten tools this is fine. For a platform with fifty operations across six domains, you have burned thousands of tokens before the agent has even started thinking.

I hit this building CaseHub. The platform has a case engine, a work-item inbox, an audit ledger, a communication mesh, agent identity, and desired-state management. Each domain has 8-15 operations. Expose them all as individual MCP tools and the agent stares at a wall of 60+ tool descriptions, most irrelevant to the task at hand. Tool selection accuracy drops. Context is consumed by descriptions the agent will never call.

The fix is straightforward: stop encoding operations as tools. Encode them as data.

## The pattern: model + action

Instead of N tools (one per operation), you register two:

```java
@McpServer("casehub")
@ApplicationScoped
public class CaseHubMcpTools {

    @Inject ModelRegistry registry;
    @Inject ReflectiveOperationDispatcher dispatcher;

    @McpTool(description = "Navigate CaseHub capabilities")
    public String casehub_model(@Nullable String domain) {
        if (domain == null) {
            return toJson(registry.getDomains());
        }
        return toJson(registry.getDomain(domain).orElseThrow());
    }

    @McpTool(description = "Execute a CaseHub operation")
    public String casehub_action(String domain, String operation,
                                 String params) {
        return toJson(dispatcher.dispatch(domain, operation, params));
    }
}
```

Two tools. The entire platform surface is accessible through them.

## Two hops to anything

The agent discovers what is available through the model, not through tool schemas:

**Hop 1** — what domains exist:
```json
// casehub_model()
{
  "domains": {
    "engine": { "summary": "Case lifecycle and coordination", "queries": 4, "mutations": 8 },
    "work":   { "summary": "Work items, inbox, delegation", "queries": 4, "mutations": 10 },
    "ledger": { "summary": "Audit trail and trust scores", "queries": 7, "mutations": 2 },
    "qhorus": { "summary": "Channels, messages, presence", "queries": 5, "mutations": 8 }
  }
}
```

Small payload. The agent reads this and knows which domain to drill into.

**Hop 2** — what operations a domain offers:
```json
// casehub_model("engine")
{
  "domain": "engine",
  "queries": [
    {
      "name": "cases",
      "description": "List cases with optional filtering and pagination",
      "params": [
        { "name": "filter", "type": "CaseFilterInput", "required": false,
          "fields": { "status": "String", "namespace": "String" } },
        { "name": "page", "type": "PageInput", "required": false,
          "fields": { "offset": "Integer", "limit": "Integer" } }
      ],
      "returns": "CasePage"
    }
  ],
  "mutations": [
    {
      "name": "startCase",
      "description": "Start a new case from a registered definition",
      "params": [
        { "name": "input", "type": "StartCaseInput", "required": true,
          "fields": { "namespace": "String", "name": "String", "context": "JSON" } }
      ],
      "returns": "CaseInstance"
    }
    // ...
  ]
}
```

The agent now has full parameter detail for the domain it cares about — including nested field descriptions for complex types, so it knows exactly what to pass. It calls `casehub_action("engine", "startCase", "{\"input\": {\"namespace\": \"aml\", \"name\": \"sar-investigation\"}}")` and gets the result.

Two hops. Two tools. The agent's context contains only the domain it is working with, not the full platform catalog. After the first two hops, the operation catalog is cached in the agent's conversation — subsequent calls to the same domain cost nothing.

## Why flat tool lists fail at scale

Consider a coordinator agent managing a case that involves creating work items, querying trust scores, and sending channel messages. With flat tools:

```
Available tools (62):
  engine_startCase, engine_suspendCase, engine_resumeCase, engine_cancelCase,
  engine_signalCase, engine_listCases, engine_getCaseById, engine_getCaseContext,
  engine_getCaseDefinitions, engine_getCasePlanItems, engine_getCaseGoals,
  engine_getCaseEvents,
  work_createWorkItem, work_claimWorkItem, work_startWorkItem,
  work_completeWorkItem, work_rejectWorkItem, work_delegateWorkItem,
  work_suspendWorkItem, work_resumeWorkItem, work_cancelWorkItem,
  work_escalateWorkItem, work_getWorkItems, work_getWorkItemById,
  work_getInbox, work_getInboxSummary,
  ledger_appendEntry, ledger_getEntries, ledger_getEntry,
  ledger_getTrustScore, ledger_getCapabilityScore, ledger_getRoutingProfile,
  ledger_verifyMerkle, ledger_createAttestation,
  qhorus_createChannel, qhorus_deleteChannel, qhorus_pauseChannel,
  // ... 28 more
```

Every one of those tool definitions — name, description, parameters, types — is in the agent's context window for every turn. The agent working on a trust score query is paying the context cost of channel management and work item delegation tools it will never call.

With the hierarchical model, the same agent sees:

```
Available tools (2):
  casehub_model(domain?)
  casehub_action(domain, operation, params)
```

It calls `casehub_model()`, sees four domains, calls `casehub_model("ledger")`, gets the ledger operations, calls `casehub_action("ledger", "trustRoutingProfile", "{\"actorId\": \"agent-1\", \"capabilityTag\": \"sar-filing\"}")`. Three calls, but only the ledger operations consumed context. The other 54 operations were never loaded.

## Discovery from GraphQL annotations

The early design called for a `ModelProvider` SPI — hand-written providers, one per domain, each declaring its operations. That was replaced before implementation started by something better: the operations already exist.

GraphQL resolvers classify operations as `@Query` or `@Mutation`. They carry `@Description` annotations. Their method signatures define the parameter types. MicroProfile GraphQL annotations already describe exactly what the MCP model needs.

One annotation groups resolver classes into domains:

```java
@McpDomain("engine")
@GraphQLApi
@ApplicationScoped
public class CaseQueryResolver {

    @Query
    @Description("List cases with optional filtering and pagination")
    public CasePage cases(CaseFilterInput filter, PageInput page) {
        // ...
    }
}
```

A CDI startup bean — `GraphQLModelScanner` — scans all `@McpDomain`-annotated classes, reads their `@Query`/`@Mutation`/`@Subscription` methods, reflects over parameter types to expand complex fields, and builds the operation catalog. The result is cached in a `ModelRegistry` (`ConcurrentHashMap`) and served on every `casehub_model` call.

Multiple resolver classes with the same domain merge. If `CaseQueryResolver` and `CaseMutationResolver` both declare `@McpDomain("engine")`, their operations appear together under the `engine` domain. The agent sees one domain, not two classes.

`ModelEnricher` is the optional dynamic layer. Domains provide CDI beans annotated with `@McpDomain` that contribute live state:

```java
@McpDomain("engine")
@ApplicationScoped
public class EngineModelEnricher implements ModelEnricher {

    @Override
    public String summary() {
        return "Case lifecycle engine — start, suspend, resume, cancel cases.";
    }

    @Override
    public Map<String, Object> state() {
        return Map.of("registeredDefinitions", registry.allDefinitions().size());
    }
}
```

An agent calling `casehub_model()` sees not just operations but live counts and status. Static operation metadata is cached at startup; dynamic state is fresh on every call.

## Zero drift by construction

The early design also proposed a Quarkus build-time code generator — `@PlatformService` annotations on service interfaces, generating REST, GraphQL, MCP, and client code from a single source. I killed this before writing a line. The generator would cost 3000-5000 lines of Jandex scanning and bytecode generation to replace maybe 800 lines of hand-written adapters. Negative ROI for five service interfaces.

What killed it wasn't the complexity alone — it was that the inputs already exist. The MCP model piggybacks on GraphQL. Add a `@Query` method to a resolver, annotate the resolver class with `@McpDomain`, and the operation appears in both the GraphQL schema and the MCP model. Zero drift because there is one source of truth, not two that a generator must keep in sync.

The dispatch path was the security-sensitive part. `casehub_action` calls resolver methods via reflection. Three constraints: only methods registered during the startup scan are invocable — the scanner filters at startup, not at call time. Resolver beans are obtained as CDI proxies via `CDI.current().select()` so interceptors fire. Parameter validation happens at dispatch time — unknown parameters are rejected, required parameters are checked, and the error includes the expected signature with types. Jackson with `JavaTimeModule` handles parameter deserialization, aligned with SmallRye GraphQL's behaviour.

## Different apps, different model trees

The model tree reflects what is actually deployed, not a static catalog. This is important when the same platform serves different applications.

A financial compliance app that embeds engine, work, and ledger:
```json
// casehub_model()
{ "engine": { "queries": 4, "mutations": 8 },
  "work": { "queries": 4, "mutations": 10 },
  "ledger": { "queries": 7, "mutations": 2 },
  "aml": { "queries": 3, "mutations": 2 } }
```

A chat application that embeds only the communication mesh:
```json
// casehub_model()
{ "qhorus": { "queries": 5, "mutations": 8 },
  "chat": { "queries": 2, "mutations": 1 } }
```

Same two MCP tools. Different model trees. The resolvers are CDI beans — only the ones on the classpath contribute. No configuration, no registration. Add a module to the classpath and its operations appear in the model.

## When to add a dedicated tool

The two-tool pattern handles more than you might expect. Two early concerns — parameter validation and repeated dispatch overhead — turned out to be non-issues:

**Schema validation works at dispatch time.** The dispatcher validates parameters against the operation's reflected signature. Unknown parameters are rejected. Required parameters are checked. The error message includes the full expected signature with types and field names. This is functionally equivalent to MCP-level schema validation, just one layer deeper.

**Definitions cache after the first hop.** Once the agent calls `casehub_model("engine")`, the full operation catalog — names, descriptions, parameter types with nested fields — is in its conversation context. Subsequent `casehub_action` calls reference cached knowledge, not a blind `Map<String, Object>`. The cost is two hops at the start of a domain interaction, not on every call.

That leaves one legitimate reason to promote an operation to a dedicated MCP tool: when the operation has distinct semantics that an LLM selects better from the tool name than from the operation catalog. If the agent consistently struggles to find the right operation string, a named tool removes ambiguity. But start with two tools and add dedicated ones only when you have evidence — not because the operation is important.

## Adapting the pattern

The CaseHub implementation uses Quarkus, CDI, and SmallRye MCP server. The pattern itself is framework-agnostic:

1. **Annotate existing API classes** with a domain grouping marker
2. **Build two tools** — model (returns the catalog) and action (dispatches operations)
3. **Scan at startup** — discover operations from whatever metadata your framework already provides (annotations, decorators, type hints)
4. **Cache the model** — domain list and operation details served from memory, dynamic state refreshed per call
5. **Keep hops to two** — domain list, then domain detail. More depth adds round-trips without meaningful filtering

The point is not the specific technology. The point is separating capability discovery from capability execution. Tools handle execution. Data handles discovery. The agent reads the catalog, picks what it needs, and calls the generic executor. The tool count stays fixed as the platform grows.
