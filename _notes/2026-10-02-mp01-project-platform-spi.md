---
layout: post
title: "ProjectPlatform — giving agents hands for task management"
date: 2026-10-02
entry_type: note
subtype: diary
projects: [casehubio/connectors]
tags: [spi, project-management, github, mcp, avatar]
series: issue-124-project-platform-spi
---

The seventh platform SPI — `ProjectPlatform` — drops into the casehub connectors library. Issues, labels, milestones, comments, and project boards, backed by GitHub's REST and GraphQL APIs.

The interesting question wasn't *how* to build it — the pattern is established across six existing SPIs, from ChatPlatform through to ContactsPlatform. The interesting question was *why*.

I'm building toward a world where the avatar and neocortex agents have real hands. Today they can send messages, read email, manage calendars, query bank accounts, search documents, sync contacts. What they can't do is manage work — create issues, track milestones, organise boards. Every time the avatar needs to act on a task, it shells out to `gh` CLI like a human at a terminal. That's fine for me running Claude Code interactively. It's not fine for a runtime agent making decisions about what to prioritise next.

The ProjectPlatform SPI fills that gap. Five capability sub-interfaces — `Issues`, `Labels`, `Milestones`, `Comments`, `Boards` — each user-scoped, each taking an `OwnerRepo` parameter. The repo-scoping is unique to this SPI; none of the other six platforms need it. GitHub's API is inherently repo-scoped, and one platform instance should serve any repo the authenticated user can access.

The GitHub provider splits into two modules: `github-client` (the shared HTTP client, using `HttpHelper.CLIENT` per protocol) and `project-github` (the SPI wiring with `GitHubCredentialResolver` for per-user token resolution). The client handles both REST for the first four capabilities and GraphQL POST for Boards — GitHub Projects v2 killed the classic REST API in April 2025, so GraphQL is the only option. My initial instinct was to treat this as a complication. It isn't. GraphQL is just an HTTP POST with a JSON body. Same client, same auth header, different payload shape.

One thing Claude caught during code review: the `Issue` record had `Objects.requireNonNull(title)` in its compact constructor, but `close()` and `reopen()` need to send a partial update with only the `state` field — no title. The original code passed `"unused"` as the title, which would have silently renamed every issue it closed on GitHub. Removing the null constraint and documenting partial-update semantics fixed it. The tension — records serving as both read and write models — is a known Java design friction, but it's the kind of thing that produces bugs when you don't think about it explicitly.

The `project-ref` module ships with pre-loaded test data: five issues across two milestones, three labels, comments, and a board with three columns. Same pattern as every other ref implementation. The avatar can test project management flows against this without hitting real GitHub, and the `@SimulationEligible` annotation means the platform's recursive wrapper generation works out of the box.

Forty-three tests. Full thirty-four-module build green. The MCP dispatch is wired — `ConnectorProjectApi` with twenty-four endpoints, `connectorsReport` updated to report project capabilities alongside the other seven platforms.

What this opens up: the avatar can now manage tasks as part of daily goal tracking. Create an issue from a life goal, attach it to a milestone representing a quarterly objective, move it across a board as work progresses, close it when done. The platform API family — chat, calendar, contacts, documents, bank, and now project — gives the agent structured access to everything it needs to act, not just observe.
