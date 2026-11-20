---
title: "The controllers were already there"
date: 2026-09-28
author: mdp
entry_type: note
subtype: diary
tags: [rest-spring-generator, spring, code-generation, platform]
series: spring-deployment
status: draft
---

# The controllers were already there

I spent the first part of this session designing a full generator migration — extending `graphql-spring-generator` to support `@PlatformWebhook`, porting WEBHOOK from the APT's internal scanner to the shared Jandex scanner, creating a new `notification-dispatch-spring` module, moving `@McpDomain` annotations from Quarkus resources to core POJOs. The design review flagged five issues that would have caused compile failures. The revised spec was clean.

Then, while researching the implementation, Claude found that `platform-spring` already generates Spring controllers for all three resources via `rest-spring-generator`. The controllers exist. They compile. They run. They just return the wrong HTTP status code.

The bug: when a Quarkus resource method returns `jakarta.ws.rs.core.Response`, the generator throws away the delegate's return value and always generates `ResponseEntity.noContent().build()`. A `CallbackDispatchResource` that returns `DispatchResult(200, body)` becomes a Spring controller that returns 204 with no body. Silent, deterministic, wrong.

The fix was three files. `RestMethodDescriptor` gets two new fields: `delegateReturnType` (the type the core POJO actually returns) and `statusBearing` (does it have an `int status()` method?). The scanner resolves the delegate's return type via Jandex when the resource returns `Response`. The writer generates `ResponseEntity.status(result.status()).body(result)` for status-bearing types. Everything else falls through unchanged.

The scope collapsed from M to S. The original design — extending the shared scanner, adding WEBHOOK as an operation type, migrating three modules — remains valid work for when consumer repos need webhook Spring support. But for this issue, the right answer was to fix what was already there rather than replace it with something more principled.

The design review earned its keep. Five HIGH findings caught annotation API misreads, wrong parameter types, and a `contextParamResolution` mechanism that fundamentally couldn't handle what I was asking it to do. All of that was moot after the pivot — but it would have cost real debugging time if I'd gone straight to implementation on the original plan.
