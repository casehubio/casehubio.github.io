---
layout: post
title: "One Field, Twenty Files"
date: 2026-08-19
entry_type: article
subtype: diary
projects: [casehub-qhorus]
tags: [a2a, message-model, propagation, records]
series: issue-396-a2a-interop-audit
---

A2A interoperability needs qhorus messages to carry structured data — not just text content but JSON payloads representing DataParts, tool outputs, artifact metadata. The design decision was straightforward: add a nullable `payload` field to `Message`, positioned after `content`. String type, not JsonNode, keeping the API module free of Jackson. JSONB in PostgreSQL validates the JSON on write.

The field itself was trivial. The propagation was not.

## Records all the way down

`Message` is a record. `MessageDispatch` is a record. `MessageView`, `NormalisedMessage`, `OutboundMessage`, `MessageReceivedEvent` — all records. Adding a field to `Message` means adding it to every carrier type that reproduces the message shape. Each carrier has its own constructor, and each constructor is called from production code and test code with positional arguments.

The production sites were manageable — maybe forty call sites across `MessageService`, `ChannelGateway`, `DeliveryBatchExecutor`, `QhorusEntityMapper`, and the MCP tools. The test sites were the problem. `sendMessage()` alone — the 13-parameter `@Tool` method — had over a hundred call sites across twenty test files. Every one needed a `null` inserted at position five.

## Regex at scale, and why it almost worked

The previous session had started the propagation with IntelliJ's regex replacement and got most of the production code right. But the regex approach had three failure modes that only showed up at the tail:

**Whitespace variants.** A search for `, corrId,` (one space) misses `,  corrId,` (two spaces) and `,   corrId,` (three spaces). Test files align arguments for readability — the spacing isn't consistent. A second pass with `\s{2,}` caught the stragglers.

**Null content.** When the content argument is `null` rather than a string literal, a regex anchored on the last `"` in the call captures the type argument (`"event"`) instead of the content. The regex silently matches the wrong position and produces a valid-looking but semantically incorrect replacement.

**Already-correct calls.** The trickiest failure. Some test files had been partially fixed by the previous session. A replace-all for `, corrId,` → `, null, corrId,` also matches within `, null, corrId,` — turning a correct 13-argument call into a broken 14-argument call. The fix compiled but put `corrId` at the `Long inReplyTo` position. The type mismatch was silent when all the trailing arguments were null.

We caught all three by iterating: run the build, read the errors, fix the pattern, repeat. The final pass caught a `@PersistenceUnit("qhorus")` annotation on the new `ExternalAgentBindingEntity` — Quarkus 3.32 doesn't support persistence unit assignment at the class level on entities. The entity's package was already in the `.packages` config; the annotation was unnecessary and blocking the entire test suite.

## What the payload field opens up

With `payload` threaded through every carrier type, the A2A bridge can map DataParts to structured JSON without losing type information. The field is deliberately excluded from normative governance — watchdog Jaccard similarity, commitment tracking, and protocol enforcement all operate on `content` only. Payload is what the message *carries*; content is what it *says*. The ledger records both for audit completeness, but only content participates in the normative layer.

Next is the inbound bridge refactor — restructuring `A2AResource` from REST-style endpoints to JSON-RPC 2.0 dispatch, replacing the inline record types with the new protocol module, and wiring up `tasks/cancel`.
