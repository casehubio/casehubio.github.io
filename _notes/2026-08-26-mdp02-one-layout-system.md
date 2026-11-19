---
title: "One Layout System"
date: 2026-08-26
author: mdp
entry_type: note
subtype: diary
series: issue-334-schema-form-dsl
projects:
  - casehubio/casehub-pages
tags: [architecture, forms, layout, web-components]
---

# One Layout System

Schema-form had a layout problem. It generates fields from a JSON schema — useful — but renders them in a single column internally. You can't wrap its children in `columns()` or `grid()` because the component produces them itself. The alternative — dropping schema-form and composing individual `textInput()` / `dropdown()` calls — works for layout but loses the validation, create mode, and submit wiring that makes schema-form worth using.

The first instinct was to add `columns` and `columnSpan` props to schema-form. That creates a parallel layout system. The whole platform uses `columns()`, `rows()`, `grid()` for layout. Form fields should compose with those like everything else.

## The separation

The answer was to pull form management apart from field generation. `formScope` provides the scope — schema context, field registration, validation, value collection, submit wiring — as a container that wraps any children. Including layout primitives.

```typescript
formScope({ schema, validateOnBlur: true },
  columns([6, 6],
    [schemaFields({ schema, fields: ["name", "grade", "value"] })],
    [schemaFields({ schema, fields: ["active", "startDate", "notes"] })],
  ),
  submitButton({ label: "Create" }),
)
```

`schemaFields()` is PagesSchemaForm in a `fieldsOnly` mode — it generates the fields but delegates validation and submit to its ancestor formScope. Individual inputs work too:

```typescript
formScope({ schema, validateOnBlur: true },
  columns([4, 4, 4],
    [textInput({ field: "name" })],
    [numberInput({ field: "value" })],
    [dropdown({ field: "grade", options: {...} })],
  ),
  submitButton({ label: "Submit" }),
)
```

One layout system. Three tiers: `schemaForm()` for the simple case (unchanged), `formScope` + `schemaFields` for auto-generated fields with custom layout, `formScope` + individual inputs for full control.

## The sibling problem

The spec review caught something I hadn't considered. My initial design had formScope as a web component extending PagesElement. That doesn't work because of how `render.ts` structures the DOM.

When `renderNode` processes a component, it creates a wrapper `div`, fires `onNode` (which creates the viz element), then renders children into slot containers. The viz element and slot containers end up as siblings inside the wrapper — not parent-child. Events from children in slot containers never bubble through the viz element. They share a parent, not an ancestor-descendant relationship.

No existing `DATA_COMPONENT_TYPE` has runtime-rendered children. They're all leaf components.

The fix: formScope isn't a web component at all. It's a container type managed by the activation callback. The callback adds event listeners directly on the wrapper `el` — which IS the ancestor of all children. Fields register themselves via `pages-field-register` events that bubble up through any depth of layout containers. This follows the precedent of `title`, `html`, and `markdown` handlers that work without viz elements.

## The queue

This session cleared four issues. Three were mechanical — `schemaForm()` and `actionButton()` DSL builders, `mutableRestSource` re-export. `actionButton()` turned out to already exist as a web component; it just needed the type registry and DSL wiring. The fourth — formScope — was the real work: brainstorming, spec, decision review, implementation plan, seven tasks across four packages.

The shared utilities that fell out (`validateField`, `readFieldValue`, `setFieldError`, `STANDALONE_TYPES`) now live in pages-component where both PagesSchemaForm and FormScopeState can use them. That consolidation eliminated three separate definitions of the same field-type set.
