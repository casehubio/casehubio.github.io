---
layout: post
title: "From Tool Calls to Browsable State"
date: 2026-08-20
entry_type: note
subtype: diary
projects: [casehub-platform]
tags: [mcp, resources, subscription, notification, quarkus-mcp-server, architecture]
---

# From Tool Calls to Browsable State

Most MCP integrations start with tools. You expose operations — query this, mutate that — and let the LLM call them. CaseHub started there too. Two tools, not sixty: `casehub_model` for navigating the domain catalog, `casehub_action` for dispatching operations. A hierarchical model that keeps the tool surface small while scaling the operation set underneath.

But tools are request-response. You ask a question, you get an answer. There's a whole category of data that doesn't fit that pattern — data that exists independently of any query, changes over time, and is interesting precisely because it changes. An IoT device's current temperature. A case's active work items. A domain's operational state. This is read-only, potentially subscribable data. In MCP protocol terms: resources.

The MCP spec draws a clear line. Tools are for actions with side effects. Resources are for data that clients can browse, read, and optionally subscribe to for change notifications. The protocol even has the plumbing: `resources/list`, `resources/read`, `resources/subscribe`, and `notifications/resources/updated`. quarkus-mcp-server 1.11.1 implements all of it — `ResourceManager`, `ResourceTemplateManager`, `ResourceInfo.sendUpdateAndForget()`. The primitives exist. The question was how to wire them through the platform so domain repos can contribute resources without depending on quarkus internals.

## The SPI design

The pattern follows what already works for tools. Domain repos don't call `ToolManager` directly — `GraphQLModelScanner` discovers `@McpDomain` resolvers and `DynamicToolRegistrar` handles the programmatic registration. Resources need the same separation: a platform SPI that domains call, and a bridge that handles the quarkus-mcp-server wiring.

`McpResourceRegistry` is that SPI. It lives in `platform-api` — pure Java, zero dependencies, available to every domain repo. The API uses a builder pattern that mirrors the quarkus upstream:

```java
resourceRegistry.newResource(McpResourceDescriptor.of(
        "casehub-domain-index",
        "casehub://domain-index",
        "application/json",
        "All domains with summaries and operation counts"))
    .handler(request -> {
        String json = formatDomainIndex();
        return McpResourceContent.of(request.uri(), json);
    })
    .register();
```

The descriptor is a sealed hierarchy — `StaticResourceDescriptor` for fixed URIs, `TemplateResourceDescriptor` for parameterised ones like `casehub://domains/{domain}`. The sealed type gives exhaustive switch expressions in the bridge, which matters because static and template resources route to completely different quarkus managers.

Registration returns a `McpResourceHandle`. For subscribable resources, `handle.notifyUpdate(uri)` pushes `notifications/resources/updated` to all subscribers — one method call, no protocol knowledge needed. For decoupled architectures where the data producer lives in a different CDI bean from the registrar, there's also a `McpResourceUpdated` CDI event that the bridge observes and relays.

## What clients actually see

The first consumer is the domain catalog itself. `casehub_model` already serves this data as a tool, but it's architecturally a resource — read-only metadata about available domains, operations, and runtime state. Now it's exposed both ways:

- **`casehub://domain-index`** — a static resource listing all domains with summaries and operation counts. Same data as `casehub_model` with no arguments, but browsable in the MCP resource panel rather than requiring a tool call.

- **`casehub://domains/{domain}`** — a template resource with completions. Type `cas` and the client auto-completes to `cases`. Read it and you get the full operation catalog, parameters, return types, event channels. Same depth as `casehub_model` tier 1, but you're browsing a data tree, not interrogating a tool.

The completions matter. MCP template completions let clients suggest valid variable values — so a client exploring `casehub://domains/{domain}` gets offered `cases`, `work`, `ledger` without having to guess. The bridge registers these via `ResourceTemplateCompletionManager`, with prefix filtering: type `w` and you get `work`, not the full list.

## Where subscription gets interesting

Static resources with `subscribable=true` support the full subscribe-notify cycle. A client subscribes to a resource URI, and when the domain calls `handle.notifyUpdate()`, the platform pushes `notifications/resources/updated` to every subscriber. The notification doesn't carry the new data — it tells the client the resource changed, and the client re-reads if it cares. This is the right model for high-frequency sources like IoT device state, where you want to notify about changes but let the client decide whether and when to fetch.

One limitation worth knowing: quarkus-mcp-server 1.11.1 only supports subscription on statically registered resources. Template-resolved URIs — the ones with variables like `iot://devices/123/state` — can be read but not subscribed to. The `subscribe()` path checks a different internal map than the `read()` path. The bridge guards against this: registering a template with `subscribable=true` throws `IllegalArgumentException` rather than silently advertising a capability that won't work. The workaround for now is to register individual static resources per subscribable entity. Not ideal for thousands of devices, but it's an upstream gap, not an architectural one.

## The three-layer MCP architecture

Pulling back, the platform's MCP story now has three layers:

**Tools** — request-response operations. `casehub_action` dispatches mutations and queries with a dynamic JSON Schema that encodes the domain catalog. `casehub_model` navigates the catalog interactively. Two tools serving an arbitrarily large operation surface.

**Resources** — browsable, read-only data. Domain catalog metadata, runtime state, eventually IoT device state and case timelines. Clients discover what's available via `resources/list`, browse with `resources/read`, subscribe with `resources/subscribe`.

**Notifications** — server push. When subscribable resource data changes, `notifications/resources/updated` tells clients to re-read. The push travels over the same transport the client connected with — SSE or streamable HTTP.

Each layer has a different interaction pattern, a different consumer need, and a different SPI in the platform. But they share infrastructure: the same `@McpServer("casehub")` scope, the same `ModelScanComplete` lifecycle event, the same CDI event patterns for extensibility. The domain catalog is the proof — the same data is available as a tool response, a static resource, and a template resource, each serving a different client interaction model.

The IoT team is next. `iot://devices/{deviceId}/state` as subscribable resources, with `StateChangeEvent` driving notifications. That work is in a separate repo and needs the template subscription gap resolved upstream. But the infrastructure is ready — registration, content serving, notification relay, and completions all work end to end.
