---
title: The Reasoning Gap
date: 2026-08-22
entry_type: note
subtype: diary
tags: [reasoning, memory, agent-architecture, casehub, foundation]
---

# The Reasoning Gap

A security agent reviews a pull request. It checks for cryptographic misuse, authentication boundary crossings, PII exposure. Finds nothing. Returns `APPROVED`. The architecture agent runs next. It also checks for authentication boundary crossings — because it has no idea the security agent already did that. The style agent runs after that. It doesn't know the PR is low-risk. Nobody told it.

Six agents, six independent assessments, zero shared reasoning. Each starts cold. The case accumulates outcomes — approved, approved, approved — but throws away the thinking that produced them. The most valuable signal any agent generates is its chain of reasoning, and the system discards it at the boundary.

I've been calling this the reasoning gap, and I think it's the single largest blind spot in how we build multi-agent systems today.

## The problem is structural, not incidental

This isn't a CaseHub-specific issue. Every orchestration system I've seen — workflow engines, agent frameworks, multi-model pipelines — treats agent outputs as typed results. The agent returns a value. The orchestrator routes on that value. The value is a decision. The reasoning behind the decision is not part of the contract.

That makes sense if agents are pure functions. It stops making sense the moment agents operate in a shared context where one agent's analysis would change another agent's behaviour. And in any case management system — whether that's PR review, medical triage, insurance claims, or legal discovery — that's exactly the situation. Specialists reason about overlapping concerns. Their reasoning is complementary. Discarding it forces redundant work.

The gap compounds with case complexity. Two parallel workers, negligible waste. Six, noticeable. Twenty — and some adaptive case management scenarios have dozens — the system is doing orders-of-magnitude more work than it would if reasoning were shared.

## Why not just put it in the output?

The obvious answer is: have the agent include its reasoning in the output map. A key called `_reasoning` or a structured field in the response. This works, but it conflates two fundamentally different things.

Output is data that the case acts on — a review verdict, a risk score, a merge decision. It flows through output mappings, gets projected into the case context, triggers binding conditions. Reasoning is metadata about *how* that data was produced. It doesn't drive case logic. It informs other agents and, critically, it informs humans investigating what happened after the fact.

When reasoning lives in the output, it gets projected alongside the verdict. JQ expressions that extract `securityReview.outcome` now also carry reasoning text into the case context, inflating it. Or the output mapping strips it, and it's gone. Neither outcome is right. Reasoning needs its own channel.

## The design that fell out

We placed reasoning on `WorkerResult` — the wrapper that workers actually return — rather than on each variant of the outcome type. A worker attaches reasoning explicitly:

```java
return WorkerResult.of(output).withReasoning(
    "No cryptographic operations found. Auth boundaries checked — "
    + "PR modifies data layer only, no controller changes. "
    + "Input validation present on all new endpoints.");
```

The `withReasoning()` copy method avoids type-erasure collisions that would have made factory method overloads ambiguous. More importantly, it makes reasoning attachment a deliberate choice. Workers that have nothing to say don't say anything — reasoning defaults to null and no storage happens.

The interesting design decision was entity scoping. CaseHub's memory system (`CaseMemoryStore`) organises memories by entity ID and domain. The existing pattern for agent memories is entity = worker name — each agent sees its own history. For reasoning, that's exactly wrong. The whole point is cross-agent visibility: worker B needs to see worker A's reasoning.

We scoped reasoning entities to the case: `entityId = "case:<caseId>"`. One query retrieves every worker's reasoning for the current case. Worker identity lives in the memory attributes, not the entity key. The retrieval configuration declares which memory domains are case-scoped versus agent-scoped, and the retriever handles both with separate budgets — a worker's own experience memories aren't displaced by the case's collective reasoning.

This scoping distinction — agent-scoped for personal history, case-scoped for shared context — maps cleanly onto how human teams work. A surgeon has their own clinical experience. But during a specific procedure, what the anaesthetist observed and what the radiologist noted are shared context for the case, not personal history of the surgeon. The memory system should reflect that boundary.

## Failure reasoning is at least as valuable as success reasoning

One thing I didn't expect to care about this much: declined and failed reasoning traces. A worker that returns `DECLINED` with reasoning like "this PR requires Kubernetes deployment knowledge and I'm scoped to application-layer review — the dependency graph crosses into infrastructure" is giving the routing system a precise signal about capability boundaries. The next time a similar PR arrives, that reasoning — stored and retrievable — tells the router not to waste a dispatch.

Failed reasoning is even more pointed. "I attempted to validate the SQL migration but the schema references a table that doesn't exist in the test database" tells the next worker, or the human investigating the failure, exactly where the gap is. Without the reasoning, the failure is a black box: `FAILED`, retry or escalate.

We made reasoning storage fire before the outcome fork in the completion handler. Success, failure, decline, gate classification — reasoning is captured regardless of which path the outcome takes. This was a deliberate choice: outcome-independent storage means the system treats all reasoning as equally worth preserving.

## What this actually opens up

The immediate value is obvious: agents in a case share context. The architecture reviewer knows what the security reviewer checked. The style reviewer knows the PR is low-risk. Less redundant analysis, more focused investigation.

But the deeper value is in what becomes queryable once reasoning is stored as structured memory.

**Trust calibration.** Current trust scoring operates on outcomes alone — did the agent succeed or fail? Reasoning adds a second axis: the quality of the agent's analysis. An agent that approves a PR with "no issues found" and one that approves with a detailed justification covering five specific risk vectors are producing the same outcome but demonstrating very different capabilities. Reasoning quality is a trust signal that outcomes can't express.

**Post-incident forensics.** When a merged PR causes a production incident, the audit trail currently shows which agents approved and what their outcomes were. With reasoning traces, it shows *what they checked and what they missed*. The security reviewer approved because it found no cryptographic operations — but the PR introduced a timing side-channel in a comparison function. The reasoning trace makes the gap visible. This is the difference between "the reviewer approved" and understanding why the approval was wrong.

**Cross-case pattern detection.** Reasoning stored at the case level is queryable across cases through application-specific MCP tools. When security reviewers across a hundred PRs all check for crypto misuse but none check for timing attacks, that pattern is invisible in outcomes — all APPROVED — but clear in reasoning traces. The system can learn its own blind spots.

The reasoning gap isn't about making agents talk to each other. It's about treating the system's collective analysis as a first-class resource — queryable, auditable, and available for learning. Workers that share reasoning stop being isolated functions and become collaborators with shared context. That's a different kind of system entirely.
