---
layout: post
title: "The start() that should never have been called"
date: 2026-05-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-connectors]
tags: [java, quarkus, webhooks, cdi, design]
---

casehub-connectors has always been outbound-only — send a message to Slack, Teams,
SMS, WhatsApp. The library routes to the right connector by id and that's it. I added
inbound this week: receiving messages from the same platforms via webhook.

The first design question was whether to have one `InboundConnector` interface covering
both pull-based transports (IMAP polling) and webhook-based ones (Slack Events API,
Teams, WhatsApp, Twilio). A unified interface would have felt symmetric. It was also
wrong.

Webhook connectors have no lifecycle. They don't start or stop — the JAX-RS endpoint
just exists and receives POST requests. Pull connectors do have a lifecycle: `start(sink)`
is called at Quarkus startup, the connector begins polling, and `stop()` is called at
shutdown. Putting `start()` and `stop()` on a unified interface and then making webhook
connectors implement them as final no-ops means `InboundConnectorService.onStart()` would
call a no-op on every webhook connector at startup. The interface would be misleading to
anyone reading it. So there are two types: `InboundConnector` for pull,
`WebhookInboundConnector` standalone for webhook. CDI discovers them through separate
`@All List<>` injection points.

The router maps each result type to an HTTP response. `Unauthorized` was the interesting
one. My first instinct was to return 401 — the request failed authentication, 401 is what
401 means. But all four platforms (Slack, Twilio, WhatsApp, Teams) retry on non-2xx.
Slack retries for up to 30 days with exponential backoff. So a bad signature on a POST
becomes a retry storm if you return 401. The correct response for a POST is 200 with
a security log entry — the platform never knows you rejected it; the ops team sees it
in the logs.

GET is different. WhatsApp's subscription verification is a human-triggered one-time
action in Meta's admin console. If the verify token is wrong, a 200 response with a
body that doesn't match the expected challenge just silently fails the subscription —
the admin sees a green 200 but no webhooks arrive. They need the 403. So: POST
`Unauthorized` → 200, GET `Unauthorized` → 403. The router checks `request.method()`
in the `Unauthorized` case.

Slack has one more wrinkle. Before the Events API goes live, Slack sends a
`url_verification` POST with a challenge value. You respond with the challenge, Slack
confirms the URL is reachable, and only then can you configure the signing secret.
That means `url_verification` has to be handled *before* the blank-secret guard — if
you check the secret first and return `Ignored()` when it's not configured, initial
setup breaks every time. The check order in `SlackInboundConnector` is: URL
verification, then blank-secret, then replay prevention, then HMAC.

I brought Claude in for two rounds of code review — one on the spec, one on the
implementation. The spec review was thorough: 14 findings, three of them critical.
The most important was that the spec never mentioned constant-time HMAC comparison.
`String.equals()` is timing-attackable — an attacker can measure response latency to
recover the expected HMAC byte by byte. The fix is `MessageDigest.isEqual()`. The
implementation review caught the replay window boundary: the spec said "more than 5
minutes" but the code used `>` at exactly 300 seconds, which lets 300-second-old
requests through. Should be `>=`.

We ended up with four concrete webhook connectors — Slack (HMAC-SHA256), Teams Outgoing
Webhooks (HMAC-SHA256 with a base64-decoded key), WhatsApp Business (HMAC-SHA256 + GET
challenge), and Twilio SMS (HMAC-SHA1 over the full request URL and sorted form params,
because Twilio chose SHA-1 and it's not configurable). One new Maven module, one JAX-RS
router, and a CDI event bus that both pull and webhook paths route through.
