---
title: "The Reviewer Was Wrong — and Right"
date: 2026-08-26
author: mdp
entry_type: note
subtype: diary
projects: [casehubio/blocks-ui]
series: issue-137-channel-activity-wrapper
tags: [channel-activity, wrapper, design-review, svg-export, adoption]
---

Quarkmind's evaluation of blocks-ui came back with a list of reasons not to adopt. Most of them were wrong — but one was accidentally useful.

The headline claim: `<blocks-channel-activity>` doesn't exist. They'd looked for the obvious element name, not found it, and concluded the component family was too fragmented to use. Fair enough — they were right that we didn't have a convenience wrapper. Every other component family (trust, session, orchestration, conversation) ships both composable primitives and a workbench-level composition. Channel-activity was the holdout.

The rest of the claims fell apart under fact-checking. "Expects SSE via PushController, not WebSocket" — PushController is transport-agnostic, two methods, zero transport code. "Split-workbench slots are for scrollable lists only" — the slots accept any children, no type checking. "Detail-pane shows 'Select an item' unconditionally" — configurable property, not hardcoded. Each one read like someone had formed a mental model of "CRUD dashboard framework" and then pattern-matched every component against it.

But the missing wrapper was real. I filed #137 and we built `<blocks-channel-activity>` in one session — split-workbench layout with nav, feed, input, topic-bar, and a collapsible tabbed sidebar for members, tasks, artifacts, and correlations. The interesting design question was controller ownership. My initial instinct was to accept pre-created controllers as properties from the host — clean separation, host retains control.

Claude's design review caught why that wouldn't work. Lit's ReactiveControllers bind to their constructor host via `addController(this)` and call `host.requestUpdate()` on that host — not on whatever component happens to be reading the controller's data. Passing controllers between components means the receiving component never re-renders. The fix: the wrapper accepts a `PushController` (still transport-agnostic) and creates domain controllers internally, binding itself as the ReactiveControllerHost. The host retains transport control; the wrapper owns the reactive lifecycle.

The review also caught property accuracy problems — my spec listed properties that didn't exist (`messageCounts`), types that were wrong (`CommitmentState` vs `CommitmentRecord`), and missed about ten existing properties on channel-feed and channel-input. Each one would have been a silent bug at integration time. The gap between "I've read the promotion spec" and "I've verified the actual source" was wider than I expected.

Separately, we fixed the diagram export. The SVG download had been silently failing — not because of any complex rendering issue, but because a Vite config alias used `path.resolve()` with a trailing slash that `resolve()` strips. `react/jsx-runtime` was resolving to `reactjsx-runtime`. Once that was fixed and exports worked, the files were 1.2MB. All of that was inline styles — `html-to-image` copies every CSS property from `getComputedStyle()` onto every element, including defaults like `accent-color: auto`. A whitelist of ~60 essential properties brought it down to 83KB with identical visual output.

The quarkmind situation is worth watching. Their reviewer described a "fundamentally different UI pattern" between blocks-ui and a real-time 3D game visualiser. That framing is wrong — PushController doesn't care what's producing the data, and split-workbench doesn't care what's in its slots. But the 70% max-width cap on the list panel is real, and if quarkmind needs a wider left panel for their Three.js canvas, they'd compose from the primitives rather than using the wrapper. That's exactly the design — convenience wrapper for the common case, primitives for everything else.
