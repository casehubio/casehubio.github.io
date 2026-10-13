---
layout: post
title: "The Diagram That Saves Itself"
date: 2026-08-26
entry_type: note
subtype: diary
projects: [casehubio/blocks-ui]
tags: [diagram, export, svg, png, html-to-image, react-flow]
---

# The Diagram That Saves Itself

I wanted Claude sessions working on CaseHub apps to be able to generate diagrams for documentation. Paste YAML, get an SVG. The kind of thing that should take fifteen minutes.

The approach was straightforward — React Flow's own docs point at `html-to-image` for capturing the viewport as SVG or PNG. We compute the node bounding box, derive the viewport transform, and call `toSvg` or `toPng`. The interesting part was doing this from Lit rather than React. React Flow's utilities like `getNodesBounds` are tied to the React store context, so we reimplemented the bounds and viewport math as pure functions. About 30 lines of geometry that any React Flow consumer outside React would need.

The foreignObject question came up early. `html-to-image`'s `toSvg` doesn't produce native SVG elements — it wraps the rendered HTML in a `<foreignObject>` inside an SVG container. For documentation and presentations rendered in a browser, this is pixel-perfect. For GitHub markdown, it's blank — GitHub strips `foreignObject` for security. PNG at 2x pixel ratio turned out to be the reliable choice for markdown embedding, with SVG as a bonus for browser-rendered contexts.

Then the download didn't work. We had the data URL, the anchor element, the `click()` call — and nothing happened. No error, no warning, just silence. Claude spotted it: the anchor was never appended to the DOM. Most browsers require the `<a>` element to participate in the document's navigation context for `.click()` to trigger a download. A detached element fires the event but the download manager ignores it. This is the kind of thing that appears in every tutorial and Stack Overflow answer, works intermittently across browsers, and fails silently when it doesn't.

We also switched from data URLs to Blob URLs for the download payload. Large exports — a complex case diagram with SWF thumbnails is over 500KB — can hit browser-internal size limits on data URL navigation. The `fetch(dataUrl) → blob → URL.createObjectURL()` pipeline handles any size and revokes the reference after the click.

The export lives in `diagram-core` so both `casehub-diagram` and `swf-diagram` get it for free. The toolbar buttons dispatch a `toolbar-export` event, the mixin handles it, and the consuming components just wire the event — same pattern as the existing save button. A standalone `export.html` page ships in the examples build output: YAML editor with syntax highlighting on the left, live diagram on the right, export buttons in the toolbar.

The YAML syntax highlighter is inline — a transparent textarea over a highlighted `<pre>`, about 20 lines of regex. It belongs in pages as a proper `pages-code-editor` component, and that's now filed as casehub-pages#372. The component would need scroll synchronisation, cursor-aware highlighting, and hooks for external tool integration — the kind of thing that earns its own session.
