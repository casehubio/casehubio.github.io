---
layout: post
title: "The Connector That Was Hiding in the Corpus"
date: 2026-11-09
entry_type: note
subtype: diary
projects: [casehubio/neocortex, casehubio/connectors]
tags: [architecture, connectors, spi, refactoring, web-capture, module-extraction, platform-design]
series: issue-895-web-corpus-production-readiness
---

# The Connector That Was Hiding in the Corpus

Last week I built a web capture pipeline inside neocortex's corpus-management module. HTML parsing with readability4j, positional markdown output, image extraction via Jsoup, PDF and DOCX parsing through Tika, an intervention queue for paywalled content. The code worked, the tests passed, and the whole thing sat inside `corpus-management-parsers` and `corpus-management/impl` like it belonged there.

It didn't.

## The tell was the import list

Twenty-seven Java source files. I went through their imports looking for corpus-specific dependencies. `CorpusEntryStore`? No. `CorpusRef`? No. `WebCaptureMetadata`? Not in the parsers, not in the image processor, not in the intervention queue. The HTML parser depends on readability4j and flexmark. The image processor depends on Jsoup and an `ImageDescriber` SPI. The intervention queue depends on `ConcurrentHashMap` and a `Duration`.

None of them know what a corpus is.

This is the architectural smell that's easy to miss in the moment of building: code that works gets committed where it's convenient, not where it belongs. The web capture pipeline landed in corpus-management because that's where the MCP tools lived. But fetch-parse-extract-describe is a platform capability. Corpus management is one consumer.

## Connectors already solved this problem

The casehubio connectors repo has eleven platform SPIs: chat, calendar, bank, email, document, contacts, location, commerce, travel, project, health. Each follows the same three-module pattern:

- `<domain>-spi` — the SPI interface with capability sub-interfaces
- `<domain>-ref` — an in-memory reference implementation for tests
- `<domain>-<provider>` — real implementations

A `WebPlatform` SPI is the same shape. Fetch a URL. Parse HTML to markdown. Extract and describe images. Parse PDFs and Word documents. Queue URLs that need human intervention to access. Five capabilities, each independently supportable, each usable by any consumer on the platform.

![WebPlatform capability structure](images/web-platform-capabilities.svg)

The moment I saw it as a connector, the question became mechanical: which files move, what stays, and what breaks.

## What moved, what stayed

![Before and after architecture](images/web-platform-before-after.svg)

Everything generic moved to connectors. Everything corpus-specific stayed in neocortex.

The SPIs — `ImageDescriber`, `DocumentParser`, `LayoutClassifier` — moved to `web-spi`. The value types — `ParseResult` with its positional layout data, `PanelDefinition`, `CapturedImage`, `InterventionRequest` — moved alongside them. The implementations — `HtmlDocumentParser`, `PdfDocumentParser`, `WebImageProcessor`, `InterventionQueue`, all the Tika-backed parsers — moved to the `web` module.

What stayed: `WebCaptureService` (the bridge that deduplicates URLs and stores entries in the corpus), `WebCaptureMetadata` (corpus metadata constants), `WebCacheCurationService` (TTL-based purging), and `StructuralChunker` (corpus-specific chunking that consumes the positional markdown output). These are the corpus concerns. They depend on `CorpusEntryStore`. They belong where they are.

The split was clean because the dependencies pointed one way. The parsers never imported corpus types. The corpus bridge imported parser output types. That's the dependency arrow that tells you where the boundary should be.

## The mechanical migration

Claude and I opened a single IntelliJ workspace spanning both repos — neocortex and connectors side by side, one Maven reactor. IntelliJ's `ide_move_file` handled the cross-module moves: file goes from `io.casehub.neocortex.corpus.management.parsers` to `io.casehub.connectors.web`, package declaration updates, and the IDE tracks what references need fixing.

Twenty-seven source files and eleven test files crossed the repo boundary. IntelliJ updated some imports automatically; others needed manual fixing where the destination module wasn't in the source path at move time. A Python script handled the bulk import rewriting — find every `io.casehub.neocortex.corpus.management.parsers.X` reference, replace with `io.casehub.connectors.web.X` or `io.casehub.connectors.web.model.X` depending on whether it was an implementation or a value type.

The dependency chain surfaced one surprise: `PanelDefinition`, a simple coordinate record for document layout panels, lived in `corpus-management-api` — the API tier. Layout classification types referenced it. Document parsers used it. The structural chunker consumed it. It looked like a corpus type because it was filed with corpus types. But it's a document parsing concern — it describes where panels sit on a PDF page. It moved to `web-spi/model` where it belongs, and the chunker now imports it from there.

## What connectors as boundaries enable

The practical consequence: any module on the platform can now read web content. Corpus management is no longer the gatekeeper.

The knowledge pipeline — which already uses `LocationPlatform` from connectors for place search — can use `WebPlatform` for research session content acquisition. The RAG pipeline can use it for document ingestion. Any agent that needs to read a webpage, handle a paywall, or extract images from HTML just injects `WebPlatform`.

And the `ImageDescriber` SPI lives in `web-spi`, with `LlmImageDescriber` in neocortex providing the vision-model implementation via `AgentProvider`. The SPI is generic; the LLM-backed implementation is domain-specific. Same pattern as every other connector — the SPI says what, the provider says how.

The reference implementation (`web-ref`) gives tests an in-memory `WebPlatform` with configurable URL-to-HTML responses, zero network calls, zero Docker containers. The same pattern `document-ref` and `location-ref` use.

## The thing about premature placement

I built the web capture code fast and in the right place for the first consumer. That's fine — you don't design platform infrastructure before you have a working use case. But the second you notice zero domain dependencies in 27 files, that's the signal. The code is telling you it doesn't belong to its consumer.

Connectors in this architecture aren't just "API wrappers for external services." They're capability boundaries. A connector says: this capability exists, here is its SPI, here are its types, and any consumer can use it without pulling in the first consumer's dependencies. The eleven existing connectors established the pattern. WebPlatform is the twelfth.

The branch still has work ahead — Playwright session transfer for authenticated content, an end-to-end smoke test, and an agent integration guide. But the architectural lift is done: web reading is a platform capability, and corpus management is just one of its consumers.
