---
layout: post
title: "Boring Is Good — Ledger GraphQL Follows the Pattern"
date: 2026-08-14
entry_type: note
subtype: diary
projects: [casehub-platform]
tags: [graphql, mcp, ledger, trust, dual-mode]
series: issue-405-dual-mode-platform
---

The ledger-graphql module went in today. Third domain — after engine and work — following the same `@GraphQLApi` + `@McpDomain` thin-resolver pattern. Seven queries, two mutations, a `ModelEnricher`, twelve unit tests. The interesting thing about it was how little was interesting.

The engine module established the shape. Work confirmed it. Ledger replicated it without friction. Same resolver structure, same DTO convention (records with `from()` factories), same CDI wiring. The only domain-specific part was the `trustRoutingProfile` composite query — global score, capability score, decision count, and quality dimensions in one call, replacing what would otherwise be four separate trust queries. That's the kind of thing GraphQL is supposed to make easy, and it did.

One SPI gap surfaced. The issue called for `appendLedgerEntry` with `domainData` — the freeform key-value payload for centralized-mode entries where remote apps can't provide typed JPA subclasses. But `AuditRecord` (the write-path input type) didn't carry `domainData`. It stopped at `metadata`. I extended the record with a `domainData` component and a `withDomainData()` fluent method, wired it through `DefaultLedgerAppender`, and updated every caller. Java records make this mechanical but non-trivial — adding a component means touching every factory method, every `with*` method, and every direct constructor call in tests. The blast radius is compile-time-visible, which is the saving grace.

Also corrected the "Your MCP Server Has Too Many Tools" article in the scaffold repo. It was written before implementation and described a `ModelProvider` SPI and a `@PlatformService` code generator — neither of which shipped. The actual mechanism — `@McpDomain` + `GraphQLModelScanner` scanning existing GraphQL annotations — is simpler and better. Two of the article's three reasons for promoting operations to dedicated tools turned out to be non-issues: schema validation is handled by the dispatcher, and operation definitions cache after the first hop. Updated the article and pushed.

Qhorus is next. If the pattern holds a fourth time — and I expect it to — the interesting work shifts to the scaffold module: wiring all four domains into a single server and proving end-to-end.
