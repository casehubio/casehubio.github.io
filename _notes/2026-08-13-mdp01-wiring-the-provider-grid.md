---
title: "Wiring the Provider Grid"
date: 2026-08-13
author: mdp
entry_type: note
subtype: diary
projects: [casehub-platform]
tags: [agent-infrastructure, multi-provider, openai-sdk, streaming]
series: issue-186-multi-agent-runtime
status: draft
---

The foundation from yesterday — `AgentBackend`, `RoutingAgentProvider`, `SubprocessRuntime` — was deliberately generic. Today was about filling in the grid: four provider modules, each proving a different integration pattern works.

The OpenAI provider was the one I was most curious about. The OpenAI Java SDK exposes a synchronous `StreamResponse<ChatCompletionChunk>` — a blocking Java `Stream` that pulls SSE events from the HTTP connection. CaseHub's agent layer is reactive (Mutiny `Multi`), and the stream subscription can't happen on the Vert.x IO thread. We bridged it with `Multi.createFrom().emitter()` running on the worker pool: the emitter callback opens the `StreamResponse`, iterates the blocking stream, and pumps `AgentEvent` instances into the reactive pipeline. Timeout is a scheduled `close()` on the `StreamResponse` from a daemon thread — same pattern as the Claude provider's subprocess kill.

What earned the native SDK its existence: `prompt_cache_key` is a first-class parameter on `ChatCompletionCreateParams`. It's a routing hint that steers requests to warm GPU cache nodes. The SDK also carries `promptCacheRetention` (deprecated in newer models in favour of `promptCacheOptions.ttl`) and `streamOptions.includeUsage` for token accounting on the final chunk. None of these surface through LangChain4j's abstraction. The event mapper — `OpenAiEventMapper` — translates `ChatCompletionChunk.Choice.Delta` content, tool calls, and usage into the platform's `AgentEvent` types. Straightforward, but the SDK types are Kotlin-generated with `Optional` everywhere, which makes the Java interop verbose.

The CLI providers were the opposite experience. Codex and Gemini CLI are structurally identical: spawn the binary via `AgentRuntime`, read stdout through a `BufferedReader`, emit each line as a `TextDelta`. The only meaningful differences are the CLI argument conventions. The `AgentRuntime` abstraction — which felt like it might be YAGNI when I designed it — immediately justified itself. Both providers inject `AgentRuntime` and call `spawn()` with a `AgentRuntimeConfig`. Neither knows or cares that `SubprocessRuntime` uses `ProcessBuilder` underneath. A `KubernetesRuntime` that spawns pods would slot in without touching a line of provider code.

The Gemini API provider follows the same structural pattern as OpenAI — CDI constructor, test constructor with stream factory, semaphore, timeout scheduler. The production path stubs out pending integration testing with a real API key. The cachedContent integration — Gemini's explicit caching where the API forbids combining tools with cached context in the same request — is the hard part and will need careful request structuring. The module compiles, the tests verify the behavioural contract, and the production path is ready to wire when we have API credentials to test against.

The final picture is cleaner than I expected. Six `AgentBackend` implementations discovered by CDI `Instance<AgentBackend>`, dispatched by a router that reads a string key from the config record. The `GatedAgentProvider` decorator wraps the router transparently — rate limiting applies uniformly without any backend knowing about it. No deployment scenario produces a CDI error: zero backends means `NoOpAgentProvider` activates; any backend on the classpath brings the router transitively.

The part I didn't anticipate: the test constructor pattern scales well across providers. Every backend has the same three-constructor shape — CDI injection, no-arg proxy, stream factory bypass. The unit tests don't touch real APIs or spawn real processes. Integration tests (gated by `@EnabledIfEnvironmentVariable`) cover the production paths when credentials are available. Each new provider module was roughly thirty minutes from POM to green tests, because the pattern was already proven.
