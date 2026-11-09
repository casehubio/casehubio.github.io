---
layout: note
title: "The tenancy that kept leaking"
date: 2026-06-11
project: casehub-work
tags: [multi-tenancy, test-infrastructure, platform-architecture]
---

I came into this session expecting to close two issues and move on. #261 was
twelve test classes failing because `tenancyId` was null on direct Panache
persist. #234 was routing inbound messages to WorkItem creation. Both filed as
S/Low. Neither was.

The twelve tests from #261 were down to four by the time I picked them up —
the previous session's commits had fixed most of them. The four survivors
weren't tenancyId nulls at all. They were test ordering contamination:
`MutableCurrentPrincipal`, an `@ApplicationScoped` mutable bean, was leaking
tenant state between test classes. A tenancy test sets `tenancyId = TENANT_B`,
the bean survives to the next class, and the next class's queries silently
return empty results. Not errors — empty. You'd never know unless you ran the
full suite in the right order.

The fix was a `QuarkusTestBeforeEachCallback` registered via ServiceLoader —
infrastructure-level reset that fires before every test method without
per-class wiring. One class, one registration file, problem structurally
eliminated.

Then the cascade started. Every module that depends on runtime needs
`quarkus.scheduler.start-mode=halted`, not `enabled=false`. The names sound
interchangeable. They're not. `enabled=false` removes the Quartz `Scheduler`
CDI bean entirely — fine until something injects it for programmatic
scheduling. Eight modules had the wrong setting.

Then the examples. Then the queues-examples. Then the dashboard. Then the AI
unit tests constructing `InMemoryWorkItemStore` outside CDI with no
`CurrentPrincipal`. Then the postgres-broadcaster SSE tests filtering by
tenant with mismatched IDs. Then a Merkle hash mismatch from `canonicalBytes`
including `tenancyId` before the entry had it set. Each fix revealed the next,
and each time I pushed to CI without verifying locally first, I lost five
minutes waiting to discover what I could have found in seconds.

That pattern — fix one thing, push, wait, find the next — was the real
lesson. I built a `fix-ci` skill to formalise the right workflow: reproduce
locally, root-cause exhaustively, verify every failing test in isolation,
full local build, one push. The skill exists because I didn't follow the
process I knew was correct.

#234 turned out to be a design question, not an implementation task. The
original issue said to put an `@ObservesAsync InboundMessage` observer in
casehub-work. But casehub-work and casehub-qhorus are Foundation-tier peers —
neither depends on the other. The bridge between them belongs in casehub-engine,
which already aggregates both. We spec'd a new `casehub-engine-inbound` module
that observes Qhorus `MessageReceivedEvent` (not raw `InboundMessage`) and
creates WorkItems through the existing `WorkItemService`. Two dependencies:
`casehub-qhorus-api` + `casehub-work-core`. The spec lives in the engine
workspace; two issues filed there; #234 closed as "design moved to engine."

CI is green. All fifteen modules pass locally. Three garden entries submitted
for the gotchas that cost the most time. The session ran long, but the codebase
is cleaner for it — every module now handles tenancy correctly, and the test
infrastructure prevents the contamination class of bugs from recurring.
