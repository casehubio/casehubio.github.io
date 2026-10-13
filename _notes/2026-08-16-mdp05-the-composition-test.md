---
layout: post
title: "The Composition Test"
date: 2026-08-16
entry_type: note
subtype: diary
projects: [casehub-platform]
tags: [graphql, mcp, scaffold, composition, persistence]
series: issue-405-dual-mode-platform
---

*Continues from [Boring Is Good](2026-08-14-mdp04-boring-is-good.md).*

Qhorus followed the pattern. Fourth domain, same shape: `@GraphQLApi` + `@McpDomain("qhorus")`, thin resolvers over SPIs, `from()` factory DTOs. Four queries (channels, channel, channelMessages, commitments), five mutations, two subscriptions (channelActivity via `MessageReceivedEvent`, channelPresence via a new `PresenceChangedEvent` CDI event added to the runtime). One design choice worth noting: qhorus-graphql depends on `casehub-qhorus-api` only — not the runtime. The resolvers inject `ChannelReader`, `ConsumerMessaging`, `CommitmentReader` — all SPIs from the api module. No JPA stores, no Hibernate entities, no persistence topology leaking through.

That turned out to matter.

The scaffold — the unified CaseHub server that composes all four domains — is where the pattern got its real test. Adding the dependency was trivial. SmallRye auto-discovers `@GraphQLApi` beans from Jandex indexes. The `casehub_model` MCP tool scans `@McpDomain` annotations at startup and builds the navigable operation catalog. In theory, add the jar, get the schema.

In practice, the CDI context refused to start. Twenty-five unsatisfied `EntityManager` dependencies. Every JPA repository from engine and work — beans I hadn't touched and hadn't added — suddenly couldn't find their `@Default EntityManager`.

I chased this for a while in the wrong direction. Assumed the scaffold was reactive-only (it isn't — it has both `quarkus-hibernate-orm` and `quarkus-hibernate-reactive-panache`). Assumed stale Maven artifacts from the cross-repo rebuild. Tried removing the new graphql dependencies entirely. Full clean rebuild of all repos. Same twenty-five failures every time. The existing `CaseGraphQLIT` tests — the ones that had been passing before this session — were also broken.

The actual cause was persistence unit package routing. Quarkus auto-assigns entities to the default persistence unit when it's the only one. But qhorus declares a named PU (`quarkus.hibernate-orm.qhorus.datasource=qhorus` with explicit `packages`). Once any named PU exists with package declarations, Quarkus stops auto-assigning entities to the default PU. The engine and work entities — which had been silently auto-assigned — suddenly had nowhere to go. No default `EntityManager` produced. The error message gives no hint: "Unsatisfied dependency for type EntityManager and qualifiers [@Default]" reads like Hibernate ORM isn't configured at all.

The fix is one line of configuration:

```properties
quarkus.hibernate-orm.packages=io.casehub.persistence.jpa,io.casehub.work.runtime,io.casehub.platform.acl.jpa
```

Declare what the default PU owns. After that, the second issue was a SmallRye GraphQL schema validation failure: `"Object" must define one or more fields`. The ledger-graphql DTOs had `Map<String, Object>` fields for `domainData`. SmallRye tries to introspect `Object` as a GraphQL type — which has no fields — and rejects the schema. Switched to the `Json` custom scalar from platform-graphql.

Once both were fixed, the composed schema came up clean. Engine types, qhorus types, platform types — all coexisting in one `/graphql` endpoint. The MCP hierarchical model lists all four domains with enricher summaries and live state. `casehub_action` dispatches to engine resolvers and returns structured results.

The api-only dependency principle proved its worth here. Qhorus-graphql brings zero persistence baggage into scaffold because it depends only on SPIs. The engine and work graphql modules still depend on their runtimes — they import internal model classes and service implementations rather than api interfaces. That's the next refactor: extract those service interfaces to api modules so all four graphql modules are persistence-topology-neutral. The qhorus module is the proof that the pattern works.

Four domains, two fixed tools, one navigable catalog. The scaffold serves a composed GraphQL schema and a hierarchical MCP model from the same annotated resolvers. The dual-mode vision from the epic is now running end-to-end.
