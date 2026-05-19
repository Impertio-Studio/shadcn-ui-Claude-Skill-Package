---
name: shadcn-syntax-field
description: >
  Use when building forms in shadcn ui evergreen-2026 and the new Field
  primitive family is the recommended path, when composing Field, FieldLabel,
  FieldDescription, FieldError, FieldGroup, FieldSet, FieldLegend,
  FieldContent, FieldSeparator, or FieldTitle, when wiring accessibility
  attributes (htmlFor, id, aria-invalid, aria-describedby) by hand around a
  controlled input, when integrating shadcn inputs with react-hook-form
  Controller WITHOUT the higher-level Form / FormField wrappers, when
  integrating with TanStack Form, when choosing between the older Form
  composition path and the newer Field path, or when an error message fails
  to render under a Field-wrapped input.
  Prevents the common Field failures : omitting FieldLabel htmlFor so the
  label does not focus the control, rendering FieldError outside its parent
  Field so data-invalid styling never applies, mixing Field with FormField
  in the same field tree, forgetting aria-invalid on the inner input so
  screen readers do not announce the error, and using Field as a generic
  layout wrapper for non-form content.
  Covers the ten exported Field primitives and their exact composition tree,
  the data-invalid pattern that drives destructive colour tokens, the
  Field-plus-Controller pattern that is the canonical react-hook-form
  integration in 2026, the Field-plus-TanStack-Form pattern, the standalone
  Field pattern without any form-state library, the FieldGroup container
  queries that switch orientation responsively, the FieldSet plus FieldLegend
  pattern for grouped controls, and the decision tree between Field (new
  flexible path) and the older Form composition (legacy quick-scaffold path).
  This skill covers the Field primitive layer only. The Form composition
  layer that wraps Field with react-hook-form context is documented in
  shadcn-syntax-form. Both paths coexist in the registry.
  Keywords: shadcn field, FieldLabel, FieldDescription, FieldError,
  FieldGroup, FieldSet, FieldLegend, FieldContent, FieldSeparator, FieldTitle,
  a11y form primitive, accessibility form primitive, form 2026 path,
  new shadcn form path, field vs form, Field versus FormField,
  Controller with Field, react-hook-form Controller, TanStack Form Field,
  aria-describedby, aria-invalid, data-invalid, htmlFor, label not focusing,
  error not showing under input, how do I build a form without FormField,
  how to use shadcn ui Field, orientation horizontal field, responsive field,
  Standard Schema errors, zod field errors, valibot field errors.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui : Field primitive (a11y form foundation, 2026)

The `Field` family is the accessibility foundation introduced in shadcn ui 2026. It provides the same label/description/error wiring as the older `Form` composition but WITHOUT coupling to `react-hook-form`. Every claim in this skill traces to the canonical source at `apps/v4/registry/new-york-v4/ui/field.tsx` in `shadcn-ui/ui` and to the official docs at `https://ui.shadcn.com/docs/components/radix/field` and `https://ui.shadcn.com/docs/forms/react-hook-form`.

ALWAYS read `shadcn-core-architecture` first if the copy-not-install paradigm is unclear. ALWAYS read `shadcn-syntax-form` when working on an existing codebase that already uses the older `<Form>` / `<FormField>` composition; the two layers coexist but should not be mixed inside one field tree.

## Quick Reference

### Minimal Field + react-hook-form Controller (canonical 2026 path)

```tsx
"use client"

import { zodResolver } from "@hookform/resolvers/zod"
import { Controller, useForm } from "react-hook-form"
import * as z from "zod"

import { Button } from "@/components/ui/button"
import {
  Field,
  FieldDescription,
  FieldError,
  FieldLabel,
} from "@/components/ui/field"
import { Input } from "@/components/ui/input"

const formSchema = z.object({
  title: z.string().min(5, "At least 5 characters.").max(32),
})

type FormValues = z.infer<typeof formSchema>

export function BugReportForm() {
  const form = useForm<FormValues>({
    resolver: zodResolver(formSchema),
    defaultValues: { title: "" },
  })

  function onSubmit(values: FormValues) {
    console.log(values)
  }

  return (
    <form onSubmit={form.handleSubmit(onSubmit)}>
      <Controller
        name="title"
        control={form.control}
        render={({ field, fieldState }) => (
          <Field data-invalid={fieldState.invalid}>
            <FieldLabel htmlFor={field.name}>Bug title</FieldLabel>
            <Input
              {...field}
              id={field.name}
              aria-invalid={fieldState.invalid}
              placeholder="Login button not working on mobile"
              autoComplete="off"
            />
            <FieldDescription>Concise summary of the bug.</FieldDescription>
            {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
          </Field>
        )}
      />
      <Button type="submit">Submit</Button>
    </form>
  )
}
```

### Ten-primitive map

| Primitive | Renders | Role |
|-----------|---------|------|
| `Field` | `<div role="group" data-slot="field" data-orientation="...">` | Container per logical input. `orientation="vertical"` (default), `"horizontal"`, or `"responsive"`. `data-invalid` drives destructive colour tokens. |
| `FieldLabel` | shadcn `<Label>` with `data-slot="field-label"` | Visible label. ALWAYS set `htmlFor` to match the inner control's `id`. |
| `FieldDescription` | `<p data-slot="field-description">` | Helper text below or beside the control. |
| `FieldError` | `<div role="alert" data-slot="field-error">` | Renders error message(s). Accepts `errors={[...]}` array (Standard Schema issues) OR `children`. Returns `null` when both are empty. |
| `FieldGroup` | `<div data-slot="field-group" class="@container/field-group">` | Stacks multiple `Field` siblings. Establishes a container query (`@container/field-group`) that lets `orientation="responsive"` switch at `@md`. |
| `FieldSet` | `<fieldset data-slot="field-set">` | Semantic grouping of related controls (e.g. address block, checkbox group). |
| `FieldLegend` | `<legend data-slot="field-legend" data-variant="legend|label">` | Heading for a `FieldSet`. `variant="label"` for nested fieldsets. |
| `FieldContent` | `<div data-slot="field-content">` | Flex-column wrapper grouping a label, descriptions, and a non-input visual (used in selectable card patterns). |
| `FieldSeparator` | `<div data-slot="field-separator">` containing `<Separator>` and optional inline content | Visual divider between sibling Fields inside a `FieldGroup`. |
| `FieldTitle` | `<div data-slot="field-label">` | Title text inside a `FieldContent` (NOT a `<label>` element; use when no control needs `htmlFor`). |

NEVER add a primitive that is not in this list. The ten above are the entire export surface of `field.tsx`.

## The composition tree

ALWAYS compose primitives in this tree :

```
<FieldSet>                            (optional ; only when grouping related fields)
  <FieldLegend>...</FieldLegend>
  <FieldDescription>...</FieldDescription>
  <FieldGroup>                        (optional ; required for orientation="responsive")
    <Field data-invalid={...}>
      <FieldLabel htmlFor={inputId}>...</FieldLabel>
      <Input id={inputId} aria-invalid={...} />
      <FieldDescription>...</FieldDescription>
      <FieldError errors={[...]} />
    </Field>
    <FieldSeparator>or</FieldSeparator>
    <Field>...</Field>
  </FieldGroup>
</FieldSet>
```

ALWAYS keep `FieldError` as a direct child of its `Field` parent. NEVER place a `FieldError` outside the `Field` it describes : the `data-invalid` styling on the parent `Field` is scoped via `group/field` and a sibling-out-of-tree breaks the cascade.

ALWAYS provide `htmlFor` on `FieldLabel` matching an `id` on the inner control. Field does NOT auto-generate ids (unlike the older `FormItem`); the explicit pairing is your responsibility. The canonical docs example uses `htmlFor={field.name}` plus `id={field.name}` from react-hook-form's `Controller`.

## The data-invalid pattern

The `Field` wrapper exposes its invalid state via the `data-invalid` attribute. Internally `fieldVariants` (the cva object in `field.tsx`) declares :

```ts
"group/field flex w-full gap-3 data-[invalid=true]:text-destructive"
```

That single class lights up the destructive colour token on every descendant that opts in via the `group/field` selector. ALWAYS set `data-invalid={fieldState.invalid}` (RHF) or `data-invalid={isInvalid}` (TanStack) on `Field`. ALWAYS ALSO set `aria-invalid={fieldState.invalid}` on the inner input so assistive tech announces the error : `data-invalid` is presentational, `aria-invalid` is the accessibility signal.

NEVER omit `aria-invalid` on the input. The `Field` wrapper does NOT propagate it down (no Slot.Root involved); the input must carry the attribute itself.

## Orientation

`Field` has a `orientation` prop with three values, driven by `cva` :

| Value | Layout | Use when |
|-------|--------|----------|
| `"vertical"` (default) | `flex-col` ; label above control, full-width | Standard mobile-first forms ; vast majority of cases. |
| `"horizontal"` | `flex-row items-center` ; label flex-auto, control on the right | Checkbox / Switch / Radio rows where label-then-control is the natural reading order. |
| `"responsive"` | `flex-col` on small, switches to `flex-row` at `@md/field-group` | Settings screens where vertical on mobile and horizontal on desktop is the spec. REQUIRES a `FieldGroup` ancestor : the container-query class `@md/field-group:...` is keyed to that ancestor's `@container/field-group`. |

ALWAYS wrap `Field orientation="responsive"` in a `FieldGroup`. Without the group ancestor the container query has no host and the orientation never switches.

## FieldError : Standard Schema integration

`FieldError` accepts an `errors` array whose items have the shape `{ message?: string }`. This matches the issue shape emitted by every Standard Schema validator (Zod, Valibot, ArkType). The primitive :

1. De-duplicates by `message`.
2. Renders a single string when one unique message remains.
3. Renders a `<ul>` of `<li>` messages when more than one.
4. Returns `null` when `errors` is empty AND `children` is empty.

```tsx
// Single error (typical RHF case)
<FieldError errors={[fieldState.error]} />

// Multiple errors (typical TanStack case)
<FieldError errors={field.state.meta.errors} />

// Custom message (overrides the array)
<FieldError>Something custom went wrong.</FieldError>
```

ALWAYS pass the error array directly; do NOT pre-stringify. The primitive needs the `{ message }` shape to de-duplicate and to skip empty entries. NEVER pass `errors={[fieldState.error.message]}` (plain strings) : the de-dup map keys on `.message` and bare strings produce undefined keys.

## Field vs Form composition : decision tree

```
Are you starting a new form in 2026?
├── YES -> use Field (this skill) + Controller (RHF) OR Field + form.Field (TanStack)
└── NO (existing code already imports from @/components/ui/form)
    ├── Field + FormField in the SAME tree? -> NEVER. Pick one path per form.
    ├── Existing form works? -> keep it on Form composition (shadcn-syntax-form)
    └── Adding a new form alongside? -> Field is fine ; the two coexist at file level
```

| Aspect | Field (new) | Form composition (legacy) |
|--------|-------------|---------------------------|
| Form library coupling | None ; works with RHF, TanStack Form, or no library | Hard-coupled to `react-hook-form` via `FormProvider` + `Controller` |
| a11y wiring | Manual `htmlFor` + `id` + `aria-invalid` + `aria-describedby` | Automatic via `useId()` + `Slot.Root` in `FormControl` |
| Boilerplate per field | ~7 lines (Controller render-prop) | ~6 lines (FormField render-prop) |
| Flexibility | Full control over markup, can wrap non-input visuals | Constrained to the seven-primitive tree |
| TanStack Form support | First-class | Not supported : the layer assumes RHF context |
| When to pick | New code, TanStack Form, mixed component libraries, custom markup | Legacy code, quick scaffolds, RHF-only projects |

ALWAYS pick ONE path per form. NEVER mix `Field` and `FormField` inside the same `<form>` tree : the two systems do not share id contexts and aria wiring breaks.

## Coexistence

Both `field.tsx` and `form.tsx` remain in the registry. They live in separate files (`@/components/ui/field` and `@/components/ui/form`) and can be installed independently :

```bash
npx shadcn@latest add field        # installs Field family only
npx shadcn@latest add form         # installs Form composition only
npx shadcn@latest add field form   # installs both ; pick path per form
```

ALWAYS install whichever path the existing codebase already uses to avoid mixed conventions. NEVER delete `form.tsx` from a codebase mid-migration : doing so breaks every legacy form simultaneously.

## RSC and 'use client'

The Field primitives use `useMemo` (in `FieldError`) and require client rendering when wired to a form library (`Controller` and `form.Field` both rely on React context). ALWAYS mark every file that composes Field with a form library as `"use client"`. The `field.tsx` file itself starts with `"use client"` at the top of the source.

NEVER place a `Controller` (or `form.Field`) inside an RSC : the context provider is in a client component and cannot cross the boundary.

## Companion Skills

- [shadcn-syntax-form](../shadcn-syntax-form/SKILL.md) : the older `Form` / `FormField` / `FormItem` composition layer (RHF-coupled).
- [shadcn-impl-form-validation](../../shadcn-impl/shadcn-impl-form-validation/SKILL.md) : end-to-end form recipe including server-action submit and async refine.
- [shadcn-errors-form-state](../../shadcn-errors/shadcn-errors-form-state/SKILL.md) : top failure modes for both Field and Form paths.
- [shadcn-syntax-variant-cva](../shadcn-syntax-variant-cva/SKILL.md) : `cn()` and the cva variant model used by `fieldVariants`.
- [shadcn-syntax-button](../shadcn-syntax-button/SKILL.md) : submit button variants and `disabled={form.formState.isSubmitting}` pattern.
- [shadcn-syntax-selectors](../shadcn-syntax-selectors/SKILL.md) : Select / Combobox binding via Controller inside a Field.

## Reference files

- [references/methods.md](references/methods.md) : complete API signatures for all ten Field primitives.
- [references/examples.md](references/examples.md) : Field + Controller (RHF), Field + TanStack Form, Field standalone, FieldSet + FieldLegend + FieldGroup composition, Field with Select / Checkbox controls.
- [references/anti-patterns.md](references/anti-patterns.md) : five canonical anti-patterns with WHY each fails and the fix.

## Sources

All claims trace to URLs in `SOURCES.md` :

- https://ui.shadcn.com/docs/components/radix/field (Field primitive canonical docs)
- https://ui.shadcn.com/docs/forms/react-hook-form (Field + Controller canonical example)
- https://ui.shadcn.com/docs/forms/tanstack-form (Field + TanStack Form canonical example)
- https://ui.shadcn.com/docs/components/radix/form (Form composition layer ; sibling skill)
- https://react-hook-form.com/docs/usecontroller/controller (Controller API)
- https://tanstack.com/form/latest (TanStack Form API)
- https://github.com/shadcn-ui/ui : `apps/v4/registry/new-york-v4/ui/field.tsx` (canonical source)

Verified 2026-05-19.
