---
title: "Examples That Tell a Story"
date: 2026-10-13
author: mdp
entry_type: note
subtype: diary
series: issue-541-reorganize-scenario-examples
projects: [casehubio/casehub-pages]
tags: [examples, narrative-design, scenario-playbooks, developer-experience]
---

The scenario examples gallery had grown to 17 samples with about 74 sub-examples. Good coverage — every playbook feature had a demo. But opening the gallery felt like opening a dictionary: alphabetically sorted, no connecting thread, each sample a standalone island of capability.

Worse, four samples demonstrated the same ARIA form-filling pattern (Form Automation, Composable Workflow, Parameterized Script, Table Population) with slightly different wrapping. "Channel" appeared in two samples meaning completely different things. "Barrier" showed up in three places. A user browsing "Coordination" and then "Step Workflows" would see different barriers with no indication of how they related.

The problem wasn't the features — it was the organising principle. The samples were arranged by implementation module, not by how someone learns a system.

## Programs have a learning arc

I started from a first-principles question: what IS a playbook? It's a program. And programs have a natural learning arc that every PL tutorial follows:

1. What can it do? (actions)
2. How do you control it? (flow control, branching)
3. How do you run things simultaneously? (concurrency)
4. How do things coordinate? (synchronisation)
5. How do you compose bigger from smaller? (modules, triggers)

Each sentence presupposes the previous one. You can't understand coordination without concurrency, can't use concurrency without flow control, can't control flow without knowing what actions exist. That arc became the gallery structure — five acts, 13 samples, each building on the last.

## The renames carry meaning

Old names like "Invoke Bindings" and "Coordination Primitives" described the implementation. New names describe the question the user is asking:

- **UI Automation** — "what can I automate?"
- **External Calls** — "how do I connect to things?"
- **Step Modifiers** — "how do I control individual steps?"
- **Control Flow** — "how do I structure decisions?"
- **Synchronisation** — "how do branches talk to each other?"
- **Shared State** — "how do I track progress across branches?"

The naming shift is small but the effect is real — someone scanning the sidebar reads a progression, not an index.

## Merges that expose bugs

Consolidating four UI automation samples into one with a picker-based layout was straightforward. But the real value came from running every sub-example in the browser afterward.

Claude hit "Run" on the Synchronisation > Barrier example and got silence — no error, no output, just "idle." The `parseScenario` function had been renamed to `parsePlaybook` in the runtime, but four sample files still called the old name. The guard clause `if (!parseScenario) return` silently exited. No exception, no console warning. Four samples had been quietly broken with no visible signal.

That led to a second discovery: `signal:` and `await:` steps inside `concurrent:` branches were never pre-extracted. The parser's pre-extraction pass only ran at the top level — nested branches bypassed it entirely and hit the Walker, which rejected `await` as a decorator key. The fix was a recursive pre-extraction via mutual recursion between `preExtract` and `resolveStepArray`, gated on whether the catalog was available.

Then `mutex:` — not in the Walker's RESERVED_KEYS. And `wait:` — same. And `createStepRunner` passing `{ scope } as never` as the execution context, missing the `resolver` field that the `signal` decorator needs.

Four pre-existing bugs, all silent, all exposed only because someone systematically clicked Run on every sub-example. The reorganisation was the excuse; the verification was the value.

## What changed

17 samples became 13. About 74 sub-examples became 67. Zero capabilities lost. The `generate-samples.js` build script gained a `SAMPLE_ORDER` map so the narrative ordering survives regeneration — without it, the build sorts alphabetically and the story dissolves.

The gallery now reads top-to-bottom as a progression: start by automating a form, then control how steps execute, then run things in parallel, then coordinate them, then compose reusable modules, then manage everything with tooling. Each sample assumes the previous ones. That's not a feature list — it's a curriculum.
