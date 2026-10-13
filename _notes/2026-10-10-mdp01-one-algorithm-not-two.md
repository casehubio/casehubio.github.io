---
layout: post
title: "One Algorithm, Not Two"
date: 2026-10-10
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [editor, refactoring, milkdown, prosemirror]
series: issue-539-progressive-insert-consolidation
---

# One Algorithm, Not Two

The playbook system had a nice text insertion effect — characters first, then words, then progressively larger chunks, so text appears to accelerate onto the page. The problem: it existed as two independent implementations. `command-executor.ts` had `progressiveInsert` for rich editors. `playbook-handler.ts` had `progressiveFill` for form inputs. Same algorithm, different consumers, diverging over time.

I extracted the core into `pages-editor-core` as two functions. `progressiveEmit` is the low-level version — takes a callback, calls it with each chunk at the right timing. `progressiveInsert` wraps that for `EditableText` consumers. Both existing callers now import from the shared module.

The interesting part came when we tried it in the Milkdown editor. Every chunk rendered as its own paragraph — three words, new paragraph, three words, new paragraph. The Milkdown bridge's `insertText()` runs each chunk through the markdown parser, which wraps plain text in `<p>` blocks. Correct behaviour for the parser; wrong behaviour for character-by-character insertion.

The fix was `insertRawText` — an optional method on the `EditableText` interface that bypasses markdown parsing and uses ProseMirror's `tr.insertText()` directly. `progressiveInsert` prefers it when available, falls back to `insertText` for editors that don't implement it. The Milkdown bridge implements it; the CodeMirror bridge doesn't need to because CodeMirror's `insertText` doesn't parse markdown.

The API layering ended up clean: `progressiveEmit` (callback) → `progressiveInsert` (typed interface) → `insertRawText` (parser bypass). Each level serves a distinct consumer. Any new editor that implements `EditableText` gets progressive insertion for free.
