---
layout: post
title: "What Do You Call a Thing That Isn't Real?"
date: 2026-09-20
entry_type: note
subtype: diary
projects: [casehubio/connectors]
tags: [spi, simulation, naming, design]
---

I started this week thinking I was going to write two platform SPIs and a pair of simulation modules. I ended it with two SPIs, zero simulation modules, and a filed issue asking someone else to make YAML do what Java already does. The interesting part is everything that happened in between.

The SPIs themselves were the easy bit. BankFeedPlatform: list accounts, get balance, query transactions. EmailPlatform: list mailboxes, query messages, get attachments. Both flat interfaces following the CalendarPlatform pattern. Both needed some form of simulation — there's no live bank feed API to call yet, and there might not be one for months.

## The naming conversation nobody asked for

The issue said "demo impls." I pushed back. "Demo" implies one use case — showcasing — but the simulation needs to serve testing, development, and production deployments where the live API doesn't exist. So what do you call it?

"Mock" was tempting but wrong — in Java, mocks verify interactions. This isn't a mock; it's a working implementation backed by synthetic data. "Ref" (reference implementation) is what the codebase already uses, but "ref" reads like a pointer type in a repo full of `ChatChannelRef` and `MemberRef`. "Embedded" sounds like a real thing running in-process — embedded Postgres is still Postgres. "Sandbox" is industry-standard for financial APIs but carries security-isolation connotations in Java.

Claude and I landed on "sim." It's what the thing IS — a simulation of a real platform. "Demo" is one USE CASE for a sim. The distinction matters because it changes how you think about the architecture: a sim isn't a second-class citizen gated behind a profile flag. It's a first-class operational mode.

Then we threw the whole thing away.

## The framework that ate the module

Partway through the design, the platform's simulation framework landed. I'd known it was coming — it was in a slot — but I hadn't looked at what it actually does. What it does is generate CDI decorators at build time from a single annotation: `@SimulationEligible`. You annotate the SPI interface, the build generates a decorator that intercepts every method call, and simulation behavior is configured at runtime through strategies and corpus data.

No sim module. No demo module. No ref module. The decorator wraps whatever real implementation exists — or a `@DefaultBean` no-op when no real provider is configured. Simulation data gets seeded into a corpus. Strategies (sequential, key-lookup, random, recorded-replay) determine how the data gets served. The scenario engine pushes overlays onto a stack at scenario start and pops them at end.

Two SPIs, one annotation each, zero hand-written simulation code.

## Conventions are hypotheses

There's a Demo SPI Convention in the platform docs that prescribes hand-written demo modules per SPI. Claude's design review found it and dutifully revised our decisions to align with it. I pushed back — not because the convention is wrong, but because it predates the simulation framework. The convention was a good answer to last month's question. The framework is a better answer to this month's.

This is the bit that matters beyond this specific design: platform rules exist to be challenged. A convention that doesn't adapt when the infrastructure changes is just friction. We challenged it, filed the evidence, and the convention should evolve to point at the framework rather than prescribe hand-written modules.

## The YAML gap

One concern surfaced that I want to track: the simulation framework's Java API is ahead of its YAML frontend. Key extractors — "for `balance(accountId)`, the key is the accountId argument" — require Java registration today. That breaks the platform's principle of YAML parity with the Java core. For these SPIs to be fully configurable from scenario YAML without touching Java, the framework needs declarative key extractors. It's a small enhancement — the build already has access to parameter names — but it's the difference between "write a YAML file and go" and "write a YAML file and also register extractors in Java."

Filed as platform#370. It's a priority because these SPIs are the first real consumers.

## What landed

Two new modules: `bank-spi` and `email-spi`. Flat interfaces with `@SimulationEligible`. Model records with `BigDecimal` for money, cursor pagination for transactions, attachment IDs for email. Platform services with the standard registry pattern. `@DefaultBean` no-ops as CDI fallback. Sixteen tests. A clean build.

The design that matters isn't in the code — it's in the decision not to write code. The simulation framework means every future SPI gets simulation for free. No module to maintain, no code to update when the interface changes, no hand-written fixtures to keep in sync. That's the kind of infrastructure decision that compounds.
