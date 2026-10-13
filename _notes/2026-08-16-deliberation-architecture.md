---
title: "Debate as Architecture: How Conversation Protocol Becomes Trade Deliberation"
date: 2026-08-16
type: diary
project: casehub-fsitrading
tags: [blocks, conversation-protocol, convergence, debate-pattern, qhorus, commitments]
---

The blocks conversation protocol was designed for structured multi-agent dialogue — points raised, agreed, disputed, resolved. Epistemic common ground tracks what's established fact versus contested claim. Convergence detection decides when to stop talking and act.

I'd been thinking about C3 as "wire up the debate pattern and add some channel handlers." That turned out to be backwards. The interesting design question isn't how to run a debate — it's how the conversation protocol's epistemic machinery connects to the agentic execution framework.

## The Bridge Problem

The debate pattern (`Patterns.debate()`) orchestrates agent turns. The conversation protocol (`ConversationProjection` → `CommonGroundAnalyser` → `ConvergenceAnalyser`) tracks epistemic state. These are two independent systems. The debate pattern knows about rounds and termination. The conversation protocol knows about established facts and disputed points. Neither knows about the other.

`ConvergenceTermination` is the bridge — it takes a `Function<T, ConversationState>` state extractor, an `EpistemicRule`, and a `ConvergencePolicy`, and maps the conversation protocol's analysis into the debate pattern's termination decision. CONSENSUS maps to Complete, DEADLOCK to Escalate. That much was in the spec from the start.

What wasn't obvious: where does the `ConversationState` live during a debate? The projection needs to see every channel message. The termination condition needs to read the projected state. The `ChoreographedDriver` needs to wake up when new messages arrive. Three concerns, all needing the same data.

## ChannelObserver: The Type I Didn't Know Existed

I started by filing three issues on blocks for the missing adapters: a `ChannelAgentInvoker` to handle `AgentRef.ChannelAgent` in the debate loop, a `QhorusEventSource` to adapt channel messages into blocks `EventSource` events, and auto-wiring between agent invocation and projection state updates.

All three were already implemented. `ChannelObserver<S>` — shipped in blocks#97, sitting in the SNAPSHOT I hadn't updated — implements `MessageObserver` (qhorus dispatch), `EventSource` (ChoreographedDriver wake-up), and holds the projected state via `AtomicReference`. One type, three interfaces. Plus convenience methods for building `TerminationCondition`s directly from the projected state.

The composition becomes clean:

```java
var observer = new FsiDeliberationStateObserver(
    new FsiConversationProjection(), channelName);

var termination = new ConvergenceTermination<>(
    ctx -> observer.currentState(),
    epistemicRule, convergencePolicy,
    recentWindow, confidenceThreshold,
    Set.of(CONSENSUS, DEADLOCK, DIMINISHING_RETURNS));

Patterns.<DeliberationContext>debate()
    .debaters(channelAgents)
    .convergence(termination)
    .backend(choreographed(invoker, serialize(), observer))
    .execute(context);
```

The observer IS the event source. The debate pattern wakes when channel messages arrive, the projection updates the state, and the termination condition reads from that state. No manual wiring needed.

## Five Convergence Outcomes, Not Three

The replan spec listed three outcomes: CONSENSUS, DEADLOCK, DIMINISHING_RETURNS. The design review found two more hiding in the interaction between `ConvergencePolicies.composite()` and `MaxIterationsTermination`.

PROGRESSING — the debate was actively productive when the round cap fired. Agents were still contributing new points, not repeating themselves. Executing a trade here would be premature. Route to human escalation.

CONVERGING — a subtler case. The composite policy picks the highest-confidence signal, with severity as tiebreaker. When the established ratio crosses 70% but a recent status change keeps the structural policy returning CONVERGING (severity 3) instead of letting the common-ground policy's CONSENSUS (severity 1) through, the debate looks like it's almost there but the tiebreaker suppresses the consensus signal. If the round cap fires during this window, the outcome depends on whether the established ratio actually meets the consensus threshold — if yes, the consensus was real but suppressed; if no, the debate was genuinely still converging.

This is the kind of interaction effect you only find by reading the actual `ConvergencePolicies.composite()` implementation. The severity tiebreaker serves its purpose (preventing low-confidence CONSENSUS from beating high-confidence DEADLOCK), but it creates a suppression window that the spec needs to handle explicitly.

## What the Concurrency Guard Taught Me

The first draft used a `ConcurrentHashMap` for the one-deliberation-per-instrument guard. The design review pointed out this doesn't survive a server restart — a crash mid-deliberation permanently blocks the instrument.

C1's arena had the same problem and solved it with a database-level guard: a partial unique index on `status = 'IN_PROGRESS'`. Multiple completed records per instrument are fine; only one in-progress is allowed. A startup recovery sweep marks orphaned IN_PROGRESS records as FAILED.

I should have checked C1 first. The pattern was right there.

## The ChannelAgentHandler Surprise

I'd designed three sub-task handlers — correlation check, volume analysis, news check — with the first two as "rule-based computation" and the third as "LLM-powered." Clean separation.

The structure review read the `ChannelAgentDispatcher.dispatch()` bytecode and found that every handler unconditionally goes through the agent provider. There's no way to skip the LLM call. The interface is LLM-first by design.

The fix: computational handlers compute their raw data in `prepareTask()` and embed it in the LLM prompt. The LLM interprets the pre-computed statistics rather than calculating them. Different from what I planned, but arguably better — LLM-interpreted findings produce natural language that integrates coherently with the debate.
