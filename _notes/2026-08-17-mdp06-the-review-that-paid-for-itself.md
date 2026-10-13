---
title: "The Review That Paid for Itself"
date: 2026-08-17
author: mdp
entry_type: note
subtype: diary
projects: [casehub-platform]
series: issue-405-dual-mode-platform
tags: [callback, code-review, serialization, cdi, sealed-interface]
---

# The Review That Paid for Itself

The SPI callback system landed today — the last batch of the dual-mode epic. Four issues across three repos: the platform callback infrastructure, engine adapters for WorkerProvisioner and ActionRiskClassifier, work adapters for SlaBreachPolicy and WorkerSelectionStrategy, and the client-side auto-registration that ties the remote app model together.

The implementation went fast. The patterns were established: `@Decorator` wraps the existing bean, checks `CallbackRegistry` for remote registrations, routes or delegates. CDI does the displacement. I'd already built the CallbackInvoker and the registration SPI in the previous session, so the adapters were variations on a theme.

ActionRiskClassifier was the exception. It uses a `@RiskClassifier` CDI qualifier to chain multiple classifiers via `ChainedActionRiskClassifier` — a `@Decorator` on the interface would intercept the chain itself, bypassing the aggregation logic. The fix was a hand-written `@RiskClassifier @ApplicationScoped` bean that joins the chain as a peer, fans out to remote classifiers, and feeds the most-restrictive result back into the chain. We made `ChainedActionRiskClassifier.mostRestrictive()` public static so both the chain and the callback adapter share the same comparison logic.

The work repo's SPIs added a wrinkle I hadn't anticipated. Both `SlaBreachPolicy` and `WorkerSelectionStrategy` extend `NamedStrategy` — they're resolved by a config-selected ID, not by CDI priority. A `@Decorator` still works here, but only if you're careful: `id()` and `triggers()` must always delegate to the wrapped bean. Route only the business methods. If the decorator returns its own `id()`, `StrategyResolver` indexes it under the wrong name and the config lookup breaks silently.

Then I asked Claude to do a full code review across all of #405 before closing. This is where the session earned its keep.

Claude came back with eight findings. Two were critical: sealed interfaces (`RiskDecision`, `BreachDecision`) can't round-trip through Jackson without `@JsonTypeInfo`. The `CallbackInvoker` serializes method arguments and deserializes return values using a plain `ObjectMapper`. A sealed interface on the return side — `mapper.readValue(body, RiskDecision.class)` — gives Jackson no way to know whether to instantiate `Autonomous` or `GateRequired`. It would have failed on the first real callback invocation.

The same class of problem hit `Preferences` in `SlaBreachContext` — an interface in platform-api, which is zero-dependency. Can't add Jackson annotations there. The fix was `addAbstractTypeMapping(Preferences.class, MapPreferences.class)` in both the invoker's ObjectMapper and a Quarkus `ObjectMapperCustomizer` on the dispatch side.

The other findings were less dramatic but worth fixing before they became production surprises: the auto-registrar was discovering `@DefaultBean` no-ops and registering them as callbacks (creating a loop), `findMethod()` ignored parameter count (wrong overload on the first SPI with one), and the `CallbackInvoker`'s `HttpClient` was never shut down on `@PreDestroy`.

The sealed-interface finding is the one that sticks with me. The code compiled, the unit tests passed (mocked invoker, so no real serialization), and the integration test used WireMock with pre-baked responses. Nothing in the test suite exercised the actual Jackson round-trip for a polymorphic return type. The bug would have surfaced exactly once — on the first real deployment — and the error message (`Cannot construct instance of sealed interface`) would have pointed at the wrong layer.

The dual-mode epic is done. Twelve issues across five repos, from the ledger tier fix through MCP infrastructure, GraphQL generation, and now the callback system. The platform can operate as a library or as a server that remote apps extend over HTTP. What's left is proving it works end-to-end with a real remote app — that's a deployment concern, not a code one.
