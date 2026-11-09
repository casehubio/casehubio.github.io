---
layout: post
title: "Two Ways to Reach Slack"
date: 2026-05-26
type: phase-update
entry_type: note
subtype: diary
projects: [connectors]
---

The `Connector` SPI in casehub-connectors is simple by design: implement `id()`
and `send()`, register as a CDI bean. But leaving the routing pattern to callers
means every consumer writes the same `stream().filter().findFirst()` loop — and
every caller defines its own behaviour when the id doesn't exist. Some log a
warning. Some silently drop. One caller throwing and another swallowing the same
error is the kind of divergence that surfaces as an inconsistent platform.

Claude and I worked through this and landed on `ConnectorService`: an
`@ApplicationScoped` bean that indexes all registered connectors at startup,
routes `send()` by id, and throws `IllegalArgumentException` with the available
ids when the channel is unknown. Duplicate ids cause startup failure. There's
a `supports()` method for guarding before send and `ids()` for enumeration.

The injection side was worth a moment's thought. The natural CDI approach for
"give me all beans of this type" is `Instance<T>`. It works, but `Instance<T>`
is a CDI interface with a long list of abstract methods — stubbing it in a plain
JUnit test is awkward. We used `@All List<Connector>` from ArC instead. ArC
populates the list at startup; in tests you pass `List.of(...)` to the constructor
directly:

```java
ConnectorService(@All List<Connector> connectors) {
    this.registry = connectors.stream()
        .collect(Collectors.toMap(Connector::id, Function.identity(), ...));
}
```

Six unit tests, no container, no mock framework. The CDI detail stays inside
the library; callers inject `ConnectorService` and know nothing about how it's
wired.

---

I wanted to think through what casehub-openclaw meant for this library. OpenClaw
delivers to Slack, Teams, WhatsApp — the same channels. Does connectors stay the
canonical delivery layer, or does openclaw subsume it?

The answer is no, and the reason is architectural. OpenClaw's channel integrations
— Baileys for WhatsApp, Bolt for Slack, grammY for Telegram — live inside the
Gateway, a long-lived Node.js daemon that is the only process that opens those
channel sessions. Everything routes through a scheduling queue and the agent
runtime. There is no path to call the Slack adapter directly from outside the
Gateway; you send a prompt to the agent, the agent decides what to do, and the
Gateway handles delivery.

That means two things for the platform.

First, direct mechanical access. When casehub-engine fires an escalation or
casehub-work hits an SLA breach, it needs to send a Slack message — now, from
a Java call, with no reasoning involved. There is no prompt to write. There is
no agent to invoke. casehub-connectors does this with a single HTTP POST to a
webhook URL. Getting OpenClaw to do the same thing would mean running a Gateway,
maintaining an agent session, and routing through an LLM that has nothing useful
to contribute to "POST this JSON to this URL."

Second, setup overhead. A service that only needs email alerts on SLA breaches
does not need a Node.js daemon managing persistent WhatsApp and Telegram sessions.
casehub-connectors adds a CDI dependency. That is the entirety of the setup.
OpenClaw's strengths — the skill ecosystem, the heartbeat model, the
conversational agent — are exactly what you don't need for a platform
notification.

The two layers genuinely complement each other. OpenClaw is where an agent
reasons about what to say and how to reach a person. casehub-connectors is where
the platform sends what it has already decided to send. `ConnectorService` is
the entry point for the latter.
