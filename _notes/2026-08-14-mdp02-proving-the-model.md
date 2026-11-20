---
title: Proving the Model
date: 2026-08-14
author: mdp
entry_type: note
subtype: diary
tags: [mcp, llm, validation, testing, design]
status: draft
---

Building the hierarchical MCP model was the easy part. The harder question: does it actually work? Can an LLM navigate a two-tier operation catalog and construct correct `casehub_action` calls, including nested record inputs with field expansion?

We wrote integration tests using the platform's own `AgentProvider` SPI — Claude as the LLM, gated by CLI availability, swappable to LangChain4j and Ollama later. Four scenarios: domain selection from tier 0, operation matching from tier 1, complex nested params from record field expansion, and end-to-end dispatch where the LLM-constructed params feed back into `casehub_action` and produce the correct result. All four passed on the first run.

That validates the mechanism. But the test domain has `echo(message)` and `create(TestInput)` — trivially unambiguous. Real domains will have `cancel` in both engine and work, `startCase` versus `resumeCase` versus `restartCase`, optional fields, enums. The comprehension test proves the approach is viable; it doesn't prove it's robust under ambiguity.

The more interesting work was addressing the downsides. Flat MCP tools have JSON Schema in the tool definition — always visible in the system prompt, guiding the LLM before it generates params. The hierarchical model shows the schema once during `casehub_model` navigation. In a long conversation, that schema scrolls out of context. The LLM guesses. It guesses wrong.

The fix was path-based param validation in the dispatcher. The domain + operation path resolves to an `OperationDescriptor` which already carries param metadata — names, types, required flags, field maps for complex types. We validate the incoming params against this metadata before Jackson deserialization touches them. Missing required param? Unknown param name? The error message includes the expected schema:

```
Invalid params for echo: Unknown parameter 'msg'.
Expected: message: String (required)
```

That error IS the schema reinjection. The LLM lost the schema from context, guessed wrong, and the validation error puts the schema back in context. One wasted round-trip, then the retry succeeds. What looked like a fundamental downside — ephemeral model output — becomes a bounded cost.

The deeper insight came from thinking about why flat tool lists break. I'd been framing it as "CaseHub has too many tools." That's half the story. The tool list is shared across ALL connected MCP servers. A developer with filesystem, GitHub, Slack, database, search, and browser tools already has 36 tools before CaseHub shows up. Every tool CaseHub adds to that flat list degrades selection accuracy for every other server's tools too.

Two tools instead of sixty isn't just a CaseHub optimisation. It's being a good citizen in a shared context.

There's one remaining gap: the schema isn't in the tool definition itself. Dynamic tool registration via `tools/list_changed` — a core MCP spec notification — would let us inject the operation catalog into `casehub_action`'s input schema at startup. The LLM would see valid domains and operations in the system prompt, same as flat tools. Filed as #240. Requires investigation into quarkus-mcp-server's programmatic registration API.

Also caught a silent bug in the scanner: `@Subscription` is `io.smallrye.graphql.api.Subscription`, not `org.eclipse.microprofile.graphql.Subscription`. The MicroProfile spec doesn't include subscriptions — SmallRye added it as a vendor extension. String-based annotation checking hid the error completely. The scanner compiled fine and ran fine; it just never matched any `@Subscription` methods.
