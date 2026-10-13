---
title: "Teaching the Harness to Read Code"
date: 2026-09-23
author: mdp
entry_type: note
subtype: diary
series: issue-203-llm-reviewer-agents
projects: [casehubio/devtown]
tags: [llm-agents, code-review, structured-findings, priority-displacement]
---

Until now, devtown's reviewer agents were stubs. The whole coordination layer worked — CasePlanModel dispatch, 4-tier failure cascade, trust scoring, qhorus speech-act messaging — but the thing that actually reads the code was returning canned strings. A security reviewer that always finds "rate-limiting absent on /payment" regardless of what you changed.

This session wired real LLM-powered code analysis into all six review capabilities.

## The displacement problem

The obvious CDI pattern — `@DefaultBean` on stubs, `@ApplicationScoped` on LLM agents — breaks when you have six stubs sharing the same `ReviewerAgent` bean type. One non-`@DefaultBean` agent suppresses *all* `@DefaultBean` instances, not just the one with the matching capability. Deploy an LLM security reviewer and you lose the architecture, style, test-coverage, and performance stubs entirely.

The fix is a `ReviewerAgentRegistry` that indexes agents by capability and selects by explicit `priority()`. Stubs return 0, LLM agents return 1. Deploy two LLM agents and the other four capabilities continue using stubs. No CDI magic, no bean type collisions, no partial deployment breakage.

## Structured findings, not free text

`ReviewerOutcome.Completed` changed from `List<String>` to `List<ReviewFinding>`. Each finding carries severity, category, file path, line range, message, and a confidence score clamped to [0.0, 1.0]. The clamping matters — LLMs return percentage values (0.95 vs 95) unpredictably, and unclamped confidence would corrupt trust calibration models downstream.

File path validation strips hallucinated paths after deserialization. The LLM sees the diff, but it may reference files that exist in the repo and weren't changed in the PR. Those get silently filtered against the actual diff file set before findings enter the trust pipeline.

## Batching and resilience

A 500-file PR doesn't go to the LLM in one shot. `LlmReviewerAgent` batches files by estimated token budget (chars/4, conservative) with a hard cap at 10 batches per agent per review. Failed batches don't discard findings from successful ones — each batch is independent. Only when every batch fails does the agent return `Failed`.

The code-analysis classifier got its own hardening: on classification failure, it now assumes worst case — `securitySensitive=true`, `architectureCrossing=true` — so all downstream review capabilities fire. The initial implementation optimistically returned "no security issues" on LLM failure, which would silently skip security review when classification breaks.

## What this opens up

The trust scoring pipeline now has real data to work with. Each `ReviewFinding` becomes a data point for attestation — human gate reviewers confirming or refuting LLM findings produces the signal that drives trust-weighted routing. The stubs produced identical findings every time, which gave trust scoring nothing to learn from.

The next question is calibration: do the stated confidence scores actually predict true-positive rates? Over many reviews, comparing confidence distributions against human confirmation rates will show whether the LLM is systematically over- or under-confident per capability. That's where the per-finding granularity starts earning its keep.
