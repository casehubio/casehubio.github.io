---
layout: post
title: "The Decorator That Couldn't See Inside"
date: 2026-09-21
entry_type: note
subtype: diary
projects: [casehubio/connectors]
tags: [spi, simulation, cdi, capability, decorator, design]
---

Last session I discovered the platform simulation framework and used it to replace hand-written simulation modules for two flat SPIs. This session I tried to do the same thing for a capability-based SPI, and the framework couldn't handle it.

The problem is architectural. A CDI `@Decorator` intercepts method calls on the interface it decorates. For BankFeedPlatform, every method returns data — `listAccounts()` returns a list of accounts, `balance()` returns a balance object. The simulation decorator intercepts each call, checks for a corpus strategy, and either returns YAML-seeded data or delegates to the underlying bean. Clean.

ChatPlatform doesn't work like that. Its methods return capability interfaces — `messaging()` returns a `Messaging` object, `discovery()` returns a `Discovery` object. The decorator intercepts `messaging()`, but the actual data methods (`send()`, `listChannels()`) live one level deeper, on the objects those methods return. Corpus YAML can produce data records. It cannot produce interface implementations.

## Why you can't just decorate the sub-interfaces

The obvious thought is: give each capability interface its own `@SimulationEligible`. But CDI decorators only work on injection points. `Messaging` isn't injected — it's returned by a factory method on `ChatPlatform`. Nobody calls `@Inject Messaging messaging`; they call `chatPlatform.messaging().send(...)`. The decorator pattern requires CDI to manage the lifecycle, and CDI doesn't manage objects returned by factory methods.

## Recursive wrappers

The solution we landed on: extend the generator to produce wrapper classes for capability methods. When the annotation says `capabilities = {"messaging", "threading", "discovery", ...}`, the generator finds each listed method, inspects its return type via Jandex, and generates a wrapper implementation. The wrapper follows the exact same intercept-or-delegate pattern the top-level decorator already uses, just with dotted qualified names — `chat-platform.messaging.send` instead of `bank-feed-platform.listAccounts`.

The detection mechanism matters here. An earlier version used auto-detection: "if the return type is an interface and not in `java.*`, it's a capability." The decision review pushed back hard on this, and rightly. Which methods return capabilities is a domain decision, not a type-system property. A method returning an interface might be a capability accessor or a data factory. Only the SPI author knows the difference. An explicit `capabilities` attribute keeps that knowledge in the domain code where it belongs.

## The hybrid that wasn't planned

The original question was whether the ref modules (calendar-ref, chat-ref) should be replaced by simulation. They shouldn't. The simulation framework provides predictable, corpus-driven data for scenarios — the same test data every time, recorded in a journal, manageable via overlay stacks. The ref modules provide stateful in-memory behaviour — create a calendar event, then list events, and the one you just created is there.

These compose through CDI decorator precedence without any additional wiring. The decorator sits in front of whatever bean CDI resolves — if the ref module is on the classpath, calls that don't match a simulation strategy pass through to the ref. If it's not, they hit the `@DefaultBean` no-op. Three modes from one architecture: corpus-driven scenarios, ref-backed stateful testing, and ref-seeded-with-corpus interactive demos.

The interesting implication is for CRUD SPIs. CalendarPlatform has both reads and writes. Configuring corpus strategies for write methods breaks state coherence — `createEvent()` goes to corpus (returns some seeded response), then `listEvents()` also goes to corpus (returns the same seeded list, without the event you just "created"). The fix is simple: don't configure corpus for write methods. Let them delegate to the ref. But it's the kind of thing you'd only discover by trying to use the framework on a CRUD SPI, not a read-only one.

## What this opens up

Every SPI in the platform can now have simulation support by adding one annotation. Flat SPIs get it for free. Capability-based SPIs need the `capabilities` attribute — a few extra strings in the annotation, alongside the other updates the SPI author already makes when adding a capability (Builder, DefaultRecord, degradation defaults, every implementation).

The generator enhancement lives in platform as a reusable capability. The next SPI that follows a capability pattern — and there will be one — gets recursive wrapper generation without designing it again.
