---
title: "Opening the Agent Layer"
date: 2026-08-12
author: mdp
entry_type: note
subtype: diary
tags: [agent-infrastructure, multi-provider, architecture, spi-design]
series: issue-186-multi-agent-runtime
status: draft
---

CaseHub's agent infrastructure has been Claude-only since day one. `ClaudeAgentProvider` wraps the Claude CLI subprocess, `ChatModelAgentProvider` bridges LangChain4j for everything else, and a CDI priority ladder picks the winner. It works. But the competitive analysis against Gas City kept flagging the same line: 9 CLI agents, 7 runtime providers, and we're sitting on one native implementation.

I wanted to understand whether expanding the agent layer was genuinely needed or just a checkbox exercise. The question that matters: does each provider's native SDK offer something LangChain4j can't express? That's why `ClaudeAgentProvider` exists in the first place — the Claude CLI manages `cache_control` breakpoints at the API level, and LangChain4j has no way to express that.

The research turned up two concrete findings. OpenAI's GPT-5.6 models introduced `prompt_cache_key` — a routing hint that steers requests to warm cache nodes — and `prompt_cache_retention` for 24-hour extended TTL. Neither parameter exists in LangChain4j. Gemini has a harder problem: its API forbids passing `tools` or `system_instruction` in the same request as `cachedContent`. LangChain4j's standard tool binding pattern breaks explicit caching entirely. A native provider can structure requests to separate cached context from tool declarations.

Beyond API providers, both Codex and Gemini now ship full CLI agents — ReAct loops, MCP support, subagent spawning, sandboxed execution. Wrapping them as subprocess providers gives CaseHub the same agentic capabilities (file editing, code analysis, multi-step tool use) that the Claude provider already delivers.

The design centres on three new SPIs. `AgentBackend` is the implementor-facing interface — same `invoke()` and `openSession()` as `AgentProvider`, plus a `key()` that identifies the backend ("claude", "openai", "codex", "gemini", "gemini-cli"). `RoutingAgentProvider` implements `AgentProvider`, discovers all `AgentBackend` beans via CDI `Instance`, and dispatches by the `model` field on `AgentSessionConfig`. Callers don't change — they still inject `AgentProvider`. The third SPI, `AgentRuntime`, abstracts process execution for CLI providers. `SubprocessRuntime` wraps `ProcessBuilder` today; a future `KubernetesRuntime` would spawn pods without touching provider code.

The interesting architectural choice was keeping `AgentBackend` and `AgentProvider` as separate interfaces with identical method signatures rather than having one extend the other. The duplication is deliberate — `AgentProvider` is for callers, `AgentBackend` is for implementations, and the router bridges them. Extending would mean every backend is also an `AgentProvider`, and `Instance<AgentProvider>` would find them all alongside the router, recreating the same ambiguity problem `ChatModelAgentProvider` already solves with `Instance<ChatModel>` filtering. Separate types, clean roles.

I got through the foundation today: the three SPIs, `SubprocessRuntime` with tests against real echo/cat/sleep processes, `RoutingAgentProvider` with routing-by-key and catch-all fallback to LangChain4j, and both existing providers refactored from `AgentProvider` to `AgentBackend`. Five commits, all tests green across the full platform. The four new provider modules (agent-openai, agent-codex, agent-gemini, agent-gemini-cli) and integration remain.

The part I'm most curious about is how the CLI providers will handle output parsing. Claude's SDK gives us a `messages()` stream with typed content blocks. Codex and Gemini don't have equivalent Java SDKs — the providers will parse subprocess stdout directly, mapping JSON-line output to `AgentEvent` streams. That's where the AgentRuntime abstraction earns its keep: the provider handles protocol, the runtime handles process lifecycle, and they compose rather than coupling.
