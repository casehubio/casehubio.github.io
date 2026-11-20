---
title: "Step Catalog and Block Control Flow"
date: 2026-09-27
author: mdp
entry_type: note
subtype: diary
projects: [casehubio/platform]
series: issue-445-agent-invoke-handler-wiring
tags: [yaml-core, step-catalog, control-flow, dsl-design]
---

The step catalog and block control flow branch landed today — 17 issues across four sessions, touching yaml-core, yaml-jackson, yaml-step-runtime, and platform. The branch adds two things that were missing from the orchestration language: a way to add steps dynamically, and a way to group them structurally.

## The step catalog

Until this branch, step types were hard-coded. If you wanted a new action in a `steps:` list, you modified StepWalker. Now steps come from a catalog that discovers actions at startup from four sources: annotation-processed `@StepPlugin` records on the classpath, MCP tools, filesystem scripts with companion schema files, and YAML step definition files. Each source contributes typed parameter definitions and an execution binding — the catalog merges them by priority, and StepWalker resolves YAML keys against whatever the catalog contains.

The interesting design decision was the invoke binding model. Every step definition carries an `InvokeBinding` that says how it executes: call an MCP tool, spawn a script, invoke an AI agent, make a REST call, run a process. The binding is a sealed interface with six variants. A step definition file can declare `invoke: { agent: { descriptor: kyc-validator, model: claude-sonnet-5 } }` and the runtime knows how to dispatch it without the step author writing any Java.

Script auto-discovery turned out cleaner than I expected. Drop a `.py` file and a `.schema.yaml` companion in a configured directory, and the step appears in the catalog. Runtime inferred from extension — `.py` gets `python3`, `.js` gets `node`. The security model underneath is a process executor with a configurable allow-list of command patterns (EXACT, PREFIX, GLOB matching). Empty config means unrestricted; once you set it, it's deny-all-except.

## Block control flow

The second half of the branch addresses a gap that's been bothering me since the decorator evaluation order landed. Control flow constructs — `loop`, `forEach`, `if`, `retry`, `timeout` — only worked as decorators on single steps. If you needed to loop three steps together, the only option was extracting a module. For two or three steps used in one place, that's excessive.

The fix is `block:` — a structural keyword that groups N steps into a single compound step. Decorators stack on the block exactly as they do on a single step. One keyword, one line of overhead, and suddenly decorator composition stays flat instead of forcing module extraction.

With `block:` established, `if/then/else` and `match/cases` followed naturally as structural step types alongside `parallel:`. All four occupy position 10 in the decorator evaluation order — the decorator stack wraps any step type identically. The uniform model means `timeout` wrapping a `match/cases` works the same way `timeout` wrapping a plugin action does.

The `match/cases` design borrows the syntax and structure of pattern matching from Java 21+ and Rust, but not their type-safety guarantees. YAML `match:` operates on runtime expression values — exhaustiveness checking is a parse-time lint (warn when no `default:` case), not a sealed-type proof.

## The vocabulary split

The rename from `when` to `if` in the imperative step vocabulary was the smallest change with the largest surface area. ~20 Java references, ~66 TypeScript references, a JSON Schema file, and the YAML language guide. The motivation: `when` is the wrong word for an imperative guard. In rules engines — Drools, OPA — `when` is reactive: it watches for a pattern to match. In a sequential step list, `if` is correct: it checks a condition at this point in execution and moves on. The vocabulary split is intentional — step-layer `if` (imperative) vs engine-layer `when` (reactive). When you see the keyword, you know which world you're in.

## What it opens up

The step catalog means the orchestration language is now extensible without touching the parser. Drop a plugin, a script, or an MCP tool and it appears in YAML. The structural types mean complex orchestration — conditional dispatch, pattern-matched event routing, parallel execution with timeout — can be expressed inline without module extraction. The YAML language guide now documents all of this: ten sections covering variables through step plugins, with a decorator evaluation order table that makes the composition model explicit.

The natural next step is runtime execution of the structural types. The parse layer resolves them; nothing executes them yet. That's engine-repo work — `BlockStep` needs a sequential executor, `IfElseStep` needs condition evaluation, `MatchStep` needs pattern matching against live scrutinee values with `${match}` variable scoping. The types are ready; the runtime isn't.
