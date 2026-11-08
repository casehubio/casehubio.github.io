---
layout: post
title: "Closed Type Systems Deserve Closed Dispatch"
date: 2026-10-26
entry_type: note
subtype: diary
projects: [casehubio/casehub-mosaic]
tags: [typescript, web-components, architecture, agency]
series: issue-277-agency-pattern-views
---

# Closed Type Systems Deserve Closed Dispatch

CaseHub's agency framework has seven capability categories — orchestration, deliberation, perception, communication, governance, optimisation, social — plus playbook patterns inherited from the step-orchestration layer. Each category has typed sub-variants: orchestration has supervisor, choreography, saga, decomposition; deliberation has voting, debate, negotiation, coalition, normative. Twenty-two agency types in total, all expressed as a closed TypeScript discriminated union.

The question was how agent-viewer should dispatch to the right visualization for each type. I started with a registry — a `Map<string, string>` mapping `"category:type"` keys to custom element tag names, following the `registerStatus` pattern already in mosaic-core. Views would self-register at import time. Agent-viewer would look up the key and dynamically render whatever was registered.

A design review tore that apart. The reviewer pointed out that `registerStatus` maps data lookups (`(domain, state) → descriptor`), not component dispatch. More importantly, `PatternCategory` is a closed union — every type is known at compile time. A runtime registry trades compile-time exhaustiveness for runtime flexibility that nobody asked for. If someone adds a new type to the union and forgets the view, a registry silently falls back to an empty state. A switch with `default: never` guards produces a compiler error. For a closed type system, that's the better contract.

The same review challenged per-category packages. I'd proposed `packages/pattern-views-orchestration/`, `packages/pattern-views-deliberation/`, and so on — seven new packages, some containing a single view. The rationale claimed this "mirrors the type system hierarchy," but the types themselves don't follow that structure — five of seven category interfaces live in a single `base.ts` file. A package with one source file and its own `package.json`, `tsconfig.json`, and build pipeline is overhead that doesn't earn its keep. We moved everything into subdirectories within agent-viewer instead: `src/views/orchestration/`, `src/views/deliberation/`, and so on.

The third revision was more interesting. I'd argued for no shared base — YAGNI, let each view extend LitElement directly. The reviewer's counter: when you're about to build twenty views with identical `:host` CSS, identical ARIA setup, and identical property contracts, that's not a premature abstraction. That's a known pattern. We ended up with a thin `PatternViewMixin` that adds the `spec` property, sets `role="region"` and `aria-label` in `connectedCallback`, and provides a shared `renderEmptyState`. Light enough that it doesn't constrain, substantial enough that twenty views won't diverge on boilerplate.

The implementation was straightforward after the design settled. We migrated voting-view and debate-view to the new convention — moved to `src/views/deliberation/`, renamed elements to `agency-deliberation-voting-view` and `agency-deliberation-debate-view`, switched from the old `pattern` property to the uniform `spec`. Built `agency-base-pattern-view` — a single parameterized component that handles six types (governance/oversight, governance/attestation, governance/trust, communication/channel-execution, communication/channel-summary, social/reflection) whose category interfaces have no fields beyond `BasePatternCategory`. One component, one test file, one gallery section. When a type later gains its own fields, it graduates from the base view to a dedicated component — a one-line dispatch change.

The gallery restructuring surfaced an assumption I hadn't examined. The pattern views are internal sub-components — they're rendered by agent-viewer's dispatch, never used directly by applications. The original gallery showed them as peers of agent-viewer, which implied they were standalone components. We restructured: agent-viewer is now section 2 with four data configurations (debate, voting, governance/oversight, playbook), showing it as the composition shell that apps actually embed. Pattern views are grouped below as "Internal Sub-Components," clearly labelled as what agent-viewer dispatches.

The switch statement is sixty-odd lines and handles all twenty-two agency types plus playbook. Every inner switch has a `default: never` guard. Adding `'supervisor': return html`<agency-orchestration-supervisor-view .spec=${spec}>`` is the template for Wave 1 — the infrastructure is in place, the contract is clear, and the compiler enforces it.
