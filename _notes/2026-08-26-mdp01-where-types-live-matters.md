---
layout: post
title: "Where Types Live Matters"
date: 2026-08-26
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [typescript, dsl, architecture]
---

# Where Types Live Matters

Adding `schemaForm()` to the DSL looked like a one-liner — and the builder
function itself is. But the type it needs, `SchemaFormProps`, lived in
`pages-viz`, and the builders live in `pages-ui`. The dependency graph runs
`pages-ui → pages-component → pages-data`. There's no edge from `pages-ui`
to `pages-viz`.

So the "one-liner" required moving `FieldSchema` and `SchemaFormProps` out
of `pages-viz` into `pages-component`'s model layer, where every other
props type already lives. The pattern was already there — `TextInputProps`,
`CheckboxProps`, `DropdownProps` all sit in `pages-component/src/model/form-input-types.ts`.
`SchemaFormProps` was the exception, not the rule.

The builder itself:

```typescript
export function schemaForm(props: SchemaFormProps): TypedComponent<"schema-form"> {
  return freeze({ type: "schema-form" as const, props: { ...props } });
}
```

One line of real logic. Nine files of plumbing to wire it in: type registry,
type guard, barrel exports, import updates in `pages-viz` to consume from
`pages-component` instead of defining locally.

The implicit rule this surfaces: if a type appears in the `ComponentTypeRegistry`,
it must live in `pages-component`. The registry is the contract between the
model layer and everything above it. Types that bypass it create import
dead-ends the next time someone needs them upstream.
