---
entry_type: note
subtype: diary
title: "Property palette: schema-driven editing from first principles"
date: 2026-08-26
author: mdp
projects: [casehub-pages]
tags: [pages-property-palette, pages-ui-components, lit, web-components, json-schema]
---

# Property palette: schema-driven editing from first principles

The property palette started as a migration target — blocks-ui's `renderPropertyForm` is 155 lines of inline HTML that renders JSON Schema as form fields. No grouping, no rich editors, no validation beyond required/min/max. The goal was to extract this into a reusable pages component. What we actually built was significantly broader.

## The three-validator problem

We found three independent `validateField()` functions scattered across the platform: one in `pages-viz/schema-types.ts`, one in `blocks-ui/diagram-core/form/validation.ts`, and the old `pages-form` package (absorbed into pages-viz months ago) had a third. Each covered a different subset of JSON Schema constraints. None handled `exclusiveMinimum`, `exclusiveMaximum`, `multipleOf`, `minItems`, or `maxItems`.

The fix was extracting a canonical `validateField()` into `@casehubio/pages-ui-components/validation` — one function, one location, all consumers import from there. We also consolidated `FieldSchema` as a canonical type in `pages-ui-components/types`, complete with the `x-${string}` template literal index signature for arbitrary JSON Schema extensions. The pages-viz copies now re-export from the shared location.

## Six controls and a tag collision

The palette needs rich editors that `pages-ui-components` didn't have: number input, date picker, datetime picker, color swatch, range slider, tag/chip editor. Building these exposed a subtle problem: `pages-viz` already registered `pages-number-input` and `pages-date-picker` as `PagesFormInput` subclasses — data-pipeline-coupled components with a completely different API (`props`, `dataSet`, `emitFieldChange()`). Our new standalone controls use the same `pages-number-input` tag but follow the standard form control contract (`value`, `label`, `error`, `change` event).

Both use the `if (!customElements.get('pages-number-input'))` guard, so whichever loads first silently wins. If both are imported on the same page, one definition is discarded and all property bindings on the losing class fail with no error.

We resolved this by removing the pages-viz versions entirely and updating `PagesSchemaForm` to use the standalone controls via its `STANDALONE_TYPES` set. A breaking migration, but a clean one — the data-pipeline-coupled versions served no purpose the standalone controls couldn't fulfil.

The color swatch pairs a native `<input type="color">` popup with a hex text input that validates on blur. The slider pairs `<input type="range">` with a plain `<input type="number">` — deliberately not a `<pages-number-input>` custom element, to avoid transitive side-effect registration when importing the slider sub-path. The tag editor is the most complex: chips with remove buttons, Enter-to-add, `maxItems`/`uniqueItems` constraints, and a live region that announces additions and removals for screen readers.

## One element, not four

The initial design proposed composed web components: `<pages-property-palette>` containing `<pages-property-group>` containing `<pages-property-field>` containing the editor. The decision review caught the problem: four nested shadow roots for a single text input. Each boundary blocks CSS inheritance, requires `composed: true` on events, and makes DOM inspection harder. The intermediate elements had no independent use case — nobody would use `<pages-property-group>` outside the palette.

We revised to a single `<pages-property-palette>` Lit element with internal render methods (`renderGroup`, `renderField`, `renderNestedObject`). Groups are native `<details>` elements. Fields delegate directly to form controls. One shadow boundary total.

## The resolver pattern

The palette resolves JSON Schema definitions to editors through a static function: `resolveEditor(schema) → EditorDescriptor`. The descriptor is either a tag name (`{ kind: 'tag', tag: 'pages-input' }`) or a render function (`{ kind: 'render', render: (ctx) => html`...` }`). The dual-mode descriptor accommodates both component-based and function-based editors without forcing one model.

Custom resolvers are first-chance overrides. The palette's `resolver` property accepts a function that returns a descriptor or `undefined`; undefined falls through to the built-in default. This is how blocks-ui will wire its trigger editor — a domain-specific `oneOf` editor that renders radio buttons for trigger types with nested sub-forms.

## What's next

The blocks-ui migration is deferred — `diagram-properties.ts` becomes a thin wrapper over `<pages-property-palette>`, but it needs the palette package published to `.casehub-packages` first. That's tracked as a follow-up issue. The `PagesSchemaForm` migration (#375) is also pending — it'll embed the palette internally, eliminating the resolver divergence between `mapFieldToComponentType()` and `resolveEditor()`. The duration editor (ISO 8601 multi-field UI) is tracked as #374.

The palette's API surface is stable: `source` (the selection SPI), `resolver` (custom editor override), `paletteId` (localStorage persistence key). Any pages consumer — diagrams, form builders, config editors — gets schema-driven property editing with grouping, validation, and rich editors for free.
