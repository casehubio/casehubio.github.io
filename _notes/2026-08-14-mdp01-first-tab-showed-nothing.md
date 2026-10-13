---
layout: post
title: "The First Tab That Showed Nothing"
date: 2026-08-14
entry_type: note
subtype: diary
projects: [scaffold]
tags: [casehub-pages, debugging, frontend, data-binding]
series: issue-41-platform-console
---

The scaffold console has ten tabs. Nine of them worked. The first one — Cases — showed an empty data table while the Orchestration tab, pulling from the exact same dataset, rendered six cases without complaint.

The obvious suspect was the data layer. Wrong endpoint? Missing `dataPath`? Column names that didn't match the response shape? We checked all of them. The REST source was fetching correctly — the data was arriving. The table just wouldn't render it.

Claude traced the problem through the pages framework source. In `interactive.ts`, `renderInitialSlot()` makes the first tab visible by calling `applyOneVisible()`. It looks correct — the tab appears, the components mount. But it never dispatches `pages-slot-change`. That event is what populates the `activeSlots` map in `site.ts`, which `computeCurrentPage()` depends on to wire up data bindings. Without it, the page context is empty and the data table has nothing to bind to.

The `swap()` function — called when a user clicks a tab — does fire the event. That's why every other tab worked. The first tab got the visual activation but skipped the data lifecycle.

The fix is one line:

```typescript
const site = await loadSite(container, app);
if (!location.hash) {
  site.navigate("Cases");
}
```

`site.navigate()` is the intended programmatic API for tab switching. It goes through `swap()`, fires the event, and the data bindings initialise. The alternative — patching `renderInitialSlot()` in the framework — would fix it at the source, but that's a casehub-pages release. The app-level workaround is immediate and applies to any pages app with data-bound components on the first tab.

This is the kind of bug where the symptom points in the wrong direction. "No data" looks like a fetch problem. The data was there the whole time — it was the tab activation lifecycle that was incomplete. The clue was that clicking away and clicking back made the data appear, but you only see that clue once you know about `pages-slot-change`.
