---
layout: post
title: "Tomorrow Came Fast"
date: 2026-11-25
entry_type: note
subtype: diary
projects: [casehubio/casehub-mosaic]
tags: [web-components, design-system, css-tokens, diagram, shared-infrastructure]
---

# Tomorrow Came Fast

Yesterday's entry ended with a prediction: "Tomorrow, when the
collapsed-strip sidebar shows type badges..." That tomorrow was today.

The diagram workbench's collapsed strip — the narrow vertical sidebar
that appears when you drill into a sub-diagram — showed only the level
name. The `DrillDownLevel` interface already carried a `diagramType`
field (fixed yesterday), but nothing rendered it. When three levels are
stacked, you see three identical grey strips with vertical text. The
type information exists; the user just can't see it without expanding
each one.

## The dot, not the badge

The strip is 36 pixels wide. The toolbar badge — a pill with text label
and semantic background colour — doesn't fit. We went with a coloured
dot at the top of the strip instead: 8 pixels, variant-coloured using
the same `--pages-*-9` tokens the toolbar badge uses for its
backgrounds. Blue for choreography, purple for HTN, amber for GOAP.
Enough to distinguish types at a glance without trying to read tiny
rotated text.

![Collapsed strip before and after](images/collapsed-strip-type-dot.svg)

The dot carries `role="status"` and `aria-label="Diagram type: htn"` —
the same ARIA pattern as the toolbar badge. A screen reader announces
the type; a sighted user sees the colour. Same information, two
channels.

## Where does the map live?

The more interesting decision was about the `DIAGRAM_TYPE_VARIANT` map
— the lookup from diagram type strings to semantic colour categories
(choreography → info, htn → accent, swf → danger, and so on). It was
defined as a file-scoped `const` inside the toolbar component. The
workbench needs the same map. Duplicate it?

We extracted it to `mosaic-core/diagram-tags.ts`, alongside
`DIAGRAM_TAGS` — the map from diagram types to their custom element tag
names. Both are diagram-type metadata. Both are consumed by multiple
components. Same file, same export.

But that raised a sharper question: should any of this live in pages?

The variant-to-colour CSS — six rules mapping `info`/`accent`/`warning`/
`success`/`danger`/`neutral` to `--pages-*` token pairs — is now
hand-rolled in both the toolbar and the workbench. That pattern is pure
pages infrastructure. It maps semantic categories to pages tokens.
Mosaic should own the domain knowledge (which diagram types belong to
which category); pages should own the rendering (given a category, what
colour is it).

We filed it upstream. The `DIAGRAM_TYPE_VARIANT` map stays in
mosaic-core. The variant badge rendering will move to pages once it
lands. For now, both components carry their own CSS — six lines each,
identical, waiting to be replaced by a single `<pages-variant-badge>`
element.

## Two maps, different shapes

`DIAGRAM_TAGS` and `DIAGRAM_TYPE_VARIANT` live in the same file but
their key sets diverge. `DIAGRAM_TAGS` has `case` and `org`;
`DIAGRAM_TYPE_VARIANT` doesn't — those types don't need a distinctive
colour because the toolbar never shows them as badges. `DIAGRAM_TYPE_VARIANT`
has `llm`, `playbook`, `agent-conf`; `DIAGRAM_TAGS` doesn't — those
aren't renderable as diagrams.

The fallback handles it: any type not in the variant map gets `neutral`.
The divergence is intentional, not a gap. But it's the kind of thing
that becomes a gap if someone adds a new diagram type to one map and
forgets the other. Worth knowing about; not worth over-engineering a
constraint for.