---
layout: post
title: "When 30 Terminals Need to Share 16 GPU Contexts"
date: 2026-08-06
type: phase-update
entry_type: note
subtype: diary
projects: [trellis]
tags: [workspace, dockview, xterm, webgl, persistence, electron]
---

I've been building a workspace view for Trellis — a floating-frame terminal workbench where each frame holds tabbed terminals pointing at different repos. The use case is a developer managing ~30 repositories across a multi-repo organisation, with 5-6 logical groupings of related repos running simultaneously. The single-panel model we had before forced linear navigation: look at one terminal, switch, look at another, switch back. The workspace view lets you see and type into multiple terminals at once, arranged however you want.

The interesting engineering problems weren't where I expected.

## The WebGL context budget

Chrome allows 16 WebGL contexts per GPU process. Electron shares one GPU process across all windows. With 30+ terminals, each backed by xterm.js, you blow through that budget on the first frame arrangement.

The solution is a three-tier renderer model. The focused terminal (active tab in the focused frame) gets WebGL — full GPU acceleration. Visible terminals in other frames get the Canvas renderer — CPU-rendered but lighter. Hidden tabs (background tabs within a frame) dispose their renderer entirely. The buffer survives because xterm.js separates the Terminal instance from the renderer — data keeps flowing into the buffer even with no renderer attached. When you switch back, the scrollback is intact.

The budget coordination crosses windows. Each BrowserWindow's renderer process tracks its local WebGL count, but acquisition and release go through the Electron main process via IPC. When a window requests a WebGL context and the global count is at 16, the main process finds the least-recently-focused WebGL terminal across all windows and sends a `webgl:demote` message to its owning window. That window downgrades to Canvas and replies with `webgl:release`, freeing the slot. A pending-acquire queue handles the case where the freed slot should go to a different waiter than the one that triggered the demotion.

I extracted the tier determination into a pure module — `workspace-renderer-tiers.ts`. Given a frame-tab map and the focused frame ID, it produces a `Map<string, 'webgl' | 'canvas' | 'none'>`. A separate `computeTransitions` function diffs two tier maps and returns the minimal set of changes. All testable without xterm.js, Dockview, or Electron in the loop.

## Discovering createTabComponent

The tab hover flyout — showing repo metadata, agent state, and terminal output when you hover a tab for 300ms — seemed straightforward until I needed to attach mouseenter/mouseleave handlers to Dockview's tab elements. Dockview renders its own tab bar; you don't own those DOM elements.

The obvious approach is MutationObserver: watch the tab container, detect new `.dv-tab` elements, attach listeners. It works, but it's fragile — tied to Dockview's internal DOM structure, needs cleanup on every group change, misses tabs created during rapid operations.

The right answer was buried in Dockview's `DockviewFrameworkOptions`: a `createTabComponent` factory. You return an `ITabRenderer` — an object with an `element` (your custom HTMLElement), `init(params)`, and optional `update`/`dispose`. Dockview wraps your element in its tab infrastructure, handling click-to-select and drag-to-rearrange. You just render the label and handle hover.

```typescript
createTabComponent: (options) => {
  const tabEl = document.createElement('div');
  let hoverTimer: ReturnType<typeof setTimeout> | null = null;

  tabEl.addEventListener('mouseenter', () => {
    hoverTimer = setTimeout(() => {
      this._showTabFlyout(options.id, tabEl);
    }, 300);
  });

  tabEl.addEventListener('mouseleave', () => {
    if (hoverTimer) { clearTimeout(hoverTimer); hoverTimer = null; }
    this._hideTabFlyout();
  });

  return {
    element: tabEl,
    init(params) { tabEl.textContent = params.title; },
    dispose() { if (hoverTimer) clearTimeout(hoverTimer); },
  };
},
```

Clean, no observers, survives Dockview DOM changes. The factory receives `options.id` (the panel's unique ID — our terminal name) and `params.title` in `init`. That's all you need.

## Persistence across two worlds

Trellis runs as an Electron app with a Quarkus sidecar. In Electron mode, layout persistence goes through IPC to electron-store — the main process aggregates per-window layouts and writes them atomically. But Trellis also works in browser mode: point a browser at the sidecar's port and you get the same UI without Electron.

In browser mode, `window.trellis` is undefined. Every persistence call — save layout, load groups, restore on startup — silently no-ops. Layouts are lost on reload.

The fix was a REST fallback. The sidecar gained two endpoints: `GET/PUT /api/workspace/layout` and `GET/PUT /api/workspace/groups`, backed by JSON files under `.trellis/` in the workspace root. The frontend detects `_browserMode = !window.trellis` at init and routes persistence calls accordingly. Group CRUD (save, update, delete) goes through `_loadGroupsData`/`_saveGroupsData` helpers that check for the IPC bridge and fall back to REST. No abstraction layer, no strategy pattern — just an if/else in each helper method.

In Electron mode, the main process is the single writer, coordinating across windows. In browser mode, there's one window, so the frontend wraps its own layout in `{ windows: [layout] }` and PUTs directly. Same data format, different transport.

## Pure logic extraction

Dockview doesn't run in happy-dom (vitest's test environment). No ResizeObserver, no real layout, no GPU. The mock is minimal — a constructor that captures options, an `addPanel` that returns a fake group.

This meant every piece of logic that touches Dockview needs to be tested through the component's public API against the mock. That works for integration tests, but it's slow and imprecise for algorithmic logic.

We extracted four pure modules: `workspace-zorder.ts` (z-index counter management and compaction), `workspace-spatial-nav.ts` (directional frame navigation via center-point half-plane geometry), `workspace-organisers.ts` (grid/stack/sidebar layout functions), and `workspace-renderer-tiers.ts` (tier determination and transition diffing). Each is a set of pure functions with no DOM, no Dockview, no Lit — just inputs and outputs.

The z-order module, for instance, exports `bringToFront(counter, isPinned)` and `compactFrames(frames)`. The workspace view calls them and applies the result to DOM elements. The tests verify counter arithmetic, tier boundaries (normal [1, 9999], pinned [10001, 20000]), and compaction triggers — none of which need a browser.

The open question is how far this pattern extends. The detach/reattach flow, for example, involves frame serialization, IPC coordination, and save-inhibit timing — all of which have testable logic currently embedded in async methods with side effects. There's probably another extraction or two waiting.
