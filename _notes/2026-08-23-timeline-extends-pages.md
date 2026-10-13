---
title: "BlocksTimeline collapses to 19 lines"
date: 2026-08-23
issue: 123
type: diary
---

The original spec for #123 concluded that PagesEventTimeline's host-pushed data model was incompatible with BlocksTimeline's self-fetch model. That framing justified shared renderers via composition — both components import pure render functions from pages-viz but keep separate component shells. Three batches of work landed that way.

Then the false boundary surfaced. DataSourceMixin is `@casehubio/pages-component` — a pages primitive, not a blocks-ui concept. The "different lifecycle" was an artifact of blocks-ui-core re-exporting pages primitives under its own namespace. #133 removed those re-exports (69 files), and the inheritance path opened up.

PagesEventTimeline gained dual-mode support: self-fetch (endpoint, headers, pagination) alongside the existing host-pushed PagesElement flow. Every capability that was in BlocksTimeline turned out to be generic — self-fetch, pagination, headers, layout switching, render callbacks. The only domain-specific code was `configure()` mapping WorkIdentity to a tenancy header.

BlocksTimeline collapsed from 337 lines to 19. It extends PagesEventTimeline and overrides one method. The strategies (event chronology, state progression, commitment lifecycle, orchestration events) are the domain value — they stayed, importing types directly from pages-viz.

The pages showcase gallery needed three fixes to display the new event-timeline component: parser type registration, displayer type map, and dead code removal from a prior extraction. Each missing registration produced a different silent failure mode — filed as a garden entry for future reference.

Interactive layout switching in the pages gallery is blocked by a stack overflow when data components are nested inside pill/tab sub-pages. Filed as casehubio/casehub-pages#352. The sample shows all three layouts stacked for now.
