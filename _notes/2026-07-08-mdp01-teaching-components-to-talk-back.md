---
layout: post
title: "Teaching Components to Talk Back"
date: 2026-07-08
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-blocks-ui]
tags: [approval-gate, events, error-handling]
---

The approval-gate component had a quiet blind spot. It sent a PUT to complete a gate decision, checked `response.ok`, and threw the response body away. The `gate.decided` event carried only what the client already knew — `gateId`, `outcome`, `resolution`. Any post-decision server data — trust score deltas, attestation results, governance context — required a separate fetch.

Clinical's SUSAR gate was the sharpest example. After a PI authorises a trial, the server returns governance context: trust score before and after, attestation dimension, gate status. Without that data in the event, clinical had to make a second round-trip. Workable, but unnecessary.

The fix is two changes in `_submitDecision`. First, parse `response.json()` after a successful PUT and include the result as `serverData` in the event payload. A new `GateDecidedPayload` interface types it — `serverData` is `unknown` because different backends return different shapes. If the body can't be parsed (a 204, an empty response, non-JSON), `serverData` is null and the event still fires. The decision succeeded; the extra data is supplementary.

Second, the error path. The component was throwing `HTTP ${status}` on failure — meaningless to the person clicking the button. A 409 that means "gate already completed by another voter" showed up as "HTTP 409". We now parse the error body and extract `body.error` or `body.message`, falling back to the HTTP status when neither exists. Same error rendering, same screen reader announce — just a better message.

One genuine gotcha surfaced during implementation. The idiomatic pattern `await response.json().catch(() => null)` fails when `response.json` is undefined — which happens in test mocks that don't provide a `json` method. The synchronous TypeError from calling `undefined()` fires before any promise is created, so `.catch()` never sees it. The fix is a plain try/catch. Obvious in hindsight, invisible until the test blew up.

The adversarial design review earned its cost. Four rounds, six issues. The reviewer caught a naming collision — `response` in an event payload that already carries `outcome` and `resolution` (both forms of the voter's "response") was confusing. `serverData` is clearer. It also pushed for a typed payload interface to match the existing patterns (`WorkItemSelectedPayload`, `QueueScopeChangedPayload`), tightened the error body narrowing to reject arrays and empty strings, and added `satisfies GateDecidedPayload` as a compile-time guard. All changes that would have shipped without the review, worked fine, and been slightly worse.
