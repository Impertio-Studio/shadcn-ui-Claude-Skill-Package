---
name: shadcn-syntax-form
description: >
  Use when building forms with shadcn ui and react-hook-form plus zod, when
  composing the seven Form primitives (Form, FormField, FormItem, FormLabel,
  FormControl, FormDescription, FormMessage), when wiring zodResolver to
  useForm, when deciding between Controller and register for a given input
  type, when validation errors fail to render under a field, when an input
  value never reaches the submit handler, or when an input throws a "switched
  from uncontrolled to controlled" warning.
  Prevents the common form failures : using register on a non-native control
  (Select, Checkbox, RadioGroup, Switch) so its value never enters
  form-state, forgetting FormMessage so zod errors silently never render,
  defining the zod schema inside the component body so it re-creates every
  render and kills memoised resolver references, calling handleSubmit without
  wiring it to the form's onSubmit prop so validation never runs, omitting
  the outer Form spread so FormField cannot reach the form context, and
  setting defaultValues to undefined so the input jumps from uncontrolled to
  controlled on first user keystroke.
  Covers the seven Form primitives and their accessibility wiring, the
  schema-first useForm + zodResolver setup, the z.infer-typeof pattern for
  TypeScript inference, the Controller-versus-register decision tree, the
  defaultValues-versus-values-versus-reset triad, the handleSubmit
  onValid-onInvalid signature, async refine validation, the
  'use client' requirement under React Server Components, and a
  forward-pointer to the newer Field primitive that sits beneath Form
  composition.
  This skill covers the react-hook-form path only. TanStack Form is listed
  as the alternative in the shadcn ui docs and is out of scope here.
  Keywords: shadcn form, react-hook-form, zod, zodResolver, FormField,
  Controller, FormMessage, FormControl, FormLabel, FormItem, FormDescription,
  useForm, schema-first form, z.infer, form validation not showing,
  validation error not appearing, error message not rendered,
  value not in submit data, input shows no error, form does not submit,
  how do I validate a form, how do I show form errors, how do I bind
  a Select to react-hook-form, how do I bind a Checkbox to react-hook-form,
  switched from uncontrolled to controlled, defaultValues, values, reset,
  handleSubmit, FormProvider, register vs Controller, async validation,
  refine, email already taken, async refine, use client form,
  RSC form, server components form, zod resolver, hookform resolvers,
  z.object, z.string, z.email, form context missing, useFormContext error.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui : Form Composition (react-hook-form + zod)

The shadcn Form layer is a thin set of seven composition primitives wrapping `react-hook-form` and wiring accessibility attributes automatically. Every claim in this skill traces to the canonical source at `apps/v4/registry/new-york-v4/ui/form.tsx` in the shadcn-ui/ui repository and to the verbatim API of `react-hook-form` and `zod`.

ALWAYS read `shadcn-core-architecture` first if the copy-not-install paradigm is unclear. ALWAYS read `shadcn-syntax-variant-cva` if `cn()` and the variant model are unclear.

## Quick Reference

### Minimal form snippet

```tsx
"use client"

import { zodResolver } from "@hookform/resolvers/zod"
import { useForm } from "react-hook-form"
import * as z from "zod"

import { Button } from "@/components/ui/button"
import {
  Form,
  FormControl,
  FormDescription,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from "@/components/ui/form"
import { Input } from "@/components/ui/input"

const formSchema = z.object({
  username: z.string().min(2, "At least 2 characters.").max(50),
})

type FormValues = z.infer<typeof formSchema>

export function ProfileForm() {
  const form = useForm<FormValues>({
    resolver: zodResolver(formSchema),
    defaultValues: { username: "" },
  })

  function onSubmit(values: FormValues) {
    console.log(values)
  }

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-6">
        <FormField
          control={form.control}
          name="username"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Username</FormLabel>
              <FormControl>
                <Input placeholder="shadcn" {...field} />
              </FormControl>
              <FormDescription>Your public display name.</FormDescription>
              <FormMessage />
            </FormItem>
          )}
        />
        <Button type="submit">Submit</Button>
      </form>
    </Form>
  )
}
```

### Seven-primitive map

| Primitive | Role | a11y wiring |
|-----------|------|-------------|
| `Form` | Re-export of `FormProvider` from react-hook-form. Spreads the `useForm` return value as context. | Provides form context; required ancestor of every `FormField`. |
| `FormField` | Wraps react-hook-form's `Controller`. Adds a context that publishes `name` to the descendants. | Bridges schema field name to rendered control. |
| `FormItem` | Grid layout container. Creates a unique `React.useId()` and publishes it as `formItemId`, `formDescriptionId`, `formMessageId`. | Ties label / control / description / message together via shared id. |
| `FormLabel` | Wraps shadcn `Label`. Reads `formItemId` from context and sets `htmlFor`. Adds `data-error` when the field has an error. | `htmlFor` -> control id; `data-error` enables destructive colour token. |
| `FormControl` | Renders Radix `Slot.Root`. Forwards props to its single child (the actual input). | Sets `id`, `aria-describedby` (description + message ids when error, description-only otherwise), `aria-invalid` (`true` when error). |
| `FormDescription` | Helper-text `<p>` with the description id. | Linked via `aria-describedby` on the control. |
| `FormMessage` | Error-text `<p>` with the message id. Renders the current zod error message when the field has one. | Linked via `aria-describedby` on the control only when error is present. |

ALWAYS use all seven primitives together. NEVER omit `FormMessage` : it is the only primitive that renders the zod error.

## The seven Form primitives

ALWAYS compose primitives in this exact tree :

```
<Form {...form}>                                 // FormProvider context
  <form onSubmit={form.handleSubmit(onSubmit)}>  // native <form> element
    <FormField                                   // Controller wrapper
      control={form.control}
      name="<schema-key>"
      render={({ field }) => (
        <FormItem>                               // grid layout + id context
          <FormLabel>...</FormLabel>             // htmlFor wired to control
          <FormControl>                          // Slot wrapping the input
            <Input {...field} />                 // or Select, Checkbox, etc.
          </FormControl>
          <FormDescription>...</FormDescription> // optional helper text
          <FormMessage />                        // zod error renders here
        </FormItem>
      )}
    />
  </form>
</Form>
```

### Why each primitive exists

1. **Form** : `FormProvider` re-exported under a shorter name. Spreading `{...form}` publishes `control`, `register`, `handleSubmit`, `formState`, `setValue`, `reset`, `watch`, `getValues`, and `trigger` to every descendant via React context. NEVER omit the spread : `FormField` calls `useFormContext` and throws when context is missing.
2. **FormField** : a generic React component that internally renders `react-hook-form`'s `Controller` and wraps it in a `FormFieldContext.Provider` that publishes `name`. The `render` prop receives `{ field, fieldState, formState }` from `Controller`. ALWAYS pass `control={form.control}` and `name="<schema-key>"`. The `name` MUST match a key in the zod schema; mismatches are runtime-silent.
3. **FormItem** : a `<div className="grid gap-2">` that calls `React.useId()` once per item and publishes `id`, `formItemId`, `formDescriptionId`, `formMessageId` via `FormItemContext`. ALWAYS render exactly one `FormItem` per `FormField`. NEVER share an `id` across fields by reusing a single `FormItem`.
4. **FormLabel** : wraps shadcn `Label`. Internally calls `useFormField()` to read `formItemId` (sets `htmlFor`) and `error` (sets `data-error="true"`, which the class string `data-[error=true]:text-destructive` colours red).
5. **FormControl** : renders Radix `Slot.Root`. It takes exactly ONE child and forwards `id={formItemId}`, `aria-describedby={!error ? formDescriptionId : formDescriptionId + " " + formMessageId}`, and `aria-invalid={!!error}` to that child. ALWAYS pass a single React element child. NEVER nest two siblings under `FormControl` : Slot throws.
6. **FormDescription** : renders `<p id={formDescriptionId}>`. The id is consumed by `FormControl`'s `aria-describedby`. ALWAYS include a description when the field needs explanation; the wiring is automatic.
7. **FormMessage** : renders `<p id={formMessageId}>{error?.message}</p>`. Returns `null` when there is no error AND no `children`. ALWAYS include `<FormMessage />` even when no description is present; otherwise the zod error never renders.

ALWAYS leave the body of `FormMessage` empty (`<FormMessage />`); the component reads `error?.message` itself. NEVER hand-roll an error `<p>` outside the primitive : `aria-describedby` will not wire to it.

## Schema-first pattern

ALWAYS define the zod schema OUTSIDE the component, derive the TypeScript type via `z.infer`, and pass both to `useForm`. NEVER inline the schema inside the component body : the schema object would be re-created on every render, `zodResolver(schema)` would also re-create, and react-hook-form's memoised resolver reference would invalidate every render. See `references/anti-patterns.md` §3 for the failure mode.

```tsx
// Module-scope (good : created once)
const formSchema = z.object({
  email: z.string().min(1, "Required.").email("Invalid email."),
  age: z.coerce.number().int().min(18, "Must be 18+."),
  agree: z.boolean().refine((v) => v === true, "You must agree."),
})

type FormValues = z.infer<typeof formSchema>

export function MyForm() {
  const form = useForm<FormValues>({
    resolver: zodResolver(formSchema),
    defaultValues: { email: "", age: 18, agree: false },
  })
  // ...
}
```

The `<FormValues>` generic on `useForm` propagates through `form.control`, `field.value`, the `onSubmit` data argument, and `form.setValue`'s name+value pair. ALWAYS pass the generic explicitly; TypeScript will refuse mismatched `name=""` strings at compile time.

## Controller vs register decision

ALWAYS use `FormField` (which uses Controller internally) for any control that does NOT accept a native `ref`. Native HTML inputs (`<input>`, `<textarea>`, `<select>`) accept `ref` and can be wired with `form.register("name")`. Radix-wrapped shadcn controls do NOT accept native `ref` : they use controlled `value` + `onValueChange` (or `checked` + `onCheckedChange`). Using `register` on these returns nothing useful and the value never enters form-state.

| Control | Path | Why |
|---------|------|-----|
| shadcn `Input` | `<FormControl><Input {...field} /></FormControl>` via FormField | native `<input>` under the hood but spreading `field` covers everything |
| shadcn `Textarea` | `<FormControl><Textarea {...field} /></FormControl>` | native `<textarea>` |
| shadcn `Select` | FormField + render with `Select value={field.value} onValueChange={field.onChange}` | controlled, no native ref |
| shadcn `Checkbox` | FormField + render with `Checkbox checked={field.value} onCheckedChange={field.onChange}` | controlled, no native ref |
| shadcn `RadioGroup` | FormField + render with `RadioGroup value={field.value} onValueChange={field.onChange}` | controlled, no native ref |
| shadcn `Switch` | FormField + render with `Switch checked={field.value} onCheckedChange={field.onChange}` | controlled, no native ref |
| shadcn `Slider` | FormField + render with `Slider value={field.value} onValueChange={field.onChange}` | controlled, no native ref |
| Native `<input>` | `register("name")` is permitted, but loses auto-wired `aria-describedby`, `aria-invalid`, and `data-error` | bypasses FormControl wiring |

NEVER use `register` for Select, Checkbox, RadioGroup, Switch, Slider, Toggle, ToggleGroup, or any other Radix-wrapped control. NEVER use `register` for components shipped as controlled-only.

## defaultValues vs values vs reset

| Option | Behaviour | Use when |
|--------|-----------|----------|
| `defaultValues` | Set ONCE at mount. Internal form-state initialised to this object. Changing the prop later has NO effect. | Static initial values known at render time. |
| `values` | CONTROLLED. Whenever the prop changes, the form re-initialises to the new object (preserves dirty fields per `resetOptions`). | Editing an existing record where data arrives async (after `fetch`). |
| `form.reset(newValues, options?)` | Imperative reset from inside an effect or callback. | After successful submit, after user clicks "Cancel", after async data lands. |

ALWAYS provide `defaultValues` for every field declared in the schema. NEVER leave a field absent or `undefined` : react-hook-form treats `undefined` as uncontrolled and React will warn "A component is changing an uncontrolled input to be controlled" on the first keystroke. For booleans use `false`; for strings use `""`; for numbers use `0` or a sensible sentinel; for arrays use `[]`.

ALWAYS use `values` (not `defaultValues`) when the initial data arrives after first render (e.g. `useQuery` for editing a record). Mixing the two is supported but `values` wins on prop change.

```tsx
// Pattern : editing an existing record
const { data } = useQuery({ queryKey: ["user", id], queryFn: getUser })
const form = useForm<FormValues>({
  resolver: zodResolver(formSchema),
  values: data,                              // re-resets when data arrives
  defaultValues: { name: "", email: "" },    // fallback before data loads
})
```

## Submit handling

`form.handleSubmit(onValid, onInvalid?)` returns a single event-handler function suitable for the native `<form onSubmit={...}>` prop. ALWAYS pass the RESULT of calling `handleSubmit`, never `handleSubmit` itself.

```tsx
<form onSubmit={form.handleSubmit(onValid, onInvalid)}>...</form>

function onValid(values: FormValues) {
  // values are already zod-parsed and type-safe
}

function onInvalid(errors: FieldErrors<FormValues>) {
  // optional : runs only when validation fails
  console.warn(errors)
}
```

`handleSubmit` :
1. Prevents the native form submit (calls `event.preventDefault()` internally).
2. Runs `zodResolver` (or any other resolver) against the current values.
3. On success, calls `onValid(values, event)`.
4. On failure, calls `onInvalid(errors, event)` if provided; updates `formState.errors` either way.
5. Sets `formState.isSubmitting` to `true` while `onValid` returns a Promise; back to `false` on settle.

ALWAYS check `form.formState.isSubmitting` to disable the submit button during async work. NEVER call `onValid` outside `handleSubmit` (e.g. `<Button onClick={() => onValid(form.getValues())}>`) : validation will not run.

## Async validation

zod supports async refinement via `.refine(async (v) => ...)` and `.superRefine`. `react-hook-form` reflects pending status via `formState.isValidating`. Examples : checking whether an email is already taken, validating a coupon code against an API.

```tsx
const formSchema = z.object({
  email: z
    .string()
    .email()
    .refine(
      async (email) => {
        const res = await fetch(`/api/check-email?email=${email}`)
        const { taken } = await res.json()
        return !taken
      },
      { message: "Email already in use." }
    ),
})
```

ALWAYS await every async check inside the refine callback. NEVER fire-and-forget an async call from refine : the resolver expects a boolean (or boolean Promise) return; a stale fetch resolves after the user has moved on. See `references/examples.md` for a debounced async-validation pattern.

## RSC and 'use client'

The shadcn `Form` component uses React hooks (`useId`, `useContext`, `useFormContext`, `useFormState`). ALWAYS mark every file that imports from `@/components/ui/form` with `"use client"` at the top, even when wrapping the form in a server component shell. Without the directive Next.js (App Router) and any other RSC framework throws "useContext is not a function" on the server.

ALWAYS render the server-component shell as the form's PARENT and keep the entire form in a single client component. NEVER split `<Form>` and `<FormField>` across a server-client boundary : the React context cannot cross it.

## Forward-pointer : the Field primitive (2026)

shadcn ui 2026 introduces a separate `Field` family (`Field`, `FieldLabel`, `FieldDescription`, `FieldError`, `FieldGroup`, `FieldSet`, `FieldLegend`, `FieldContent`, `FieldSeparator`, `FieldTitle`) that provides the same accessibility wiring without coupling to react-hook-form. The official docs now show the canonical react-hook-form pattern composing raw `Controller` with `Field` primitives. The `Form` composition documented in this skill remains in the registry and is fully supported; `Field` is the recommended new path for new code AND for TanStack Form integrations.

ALWAYS read [shadcn-syntax-field](../shadcn-syntax-field/SKILL.md) (Batch 4) when starting a new project. For existing projects that already use `Form / FormField / FormItem`, this skill remains the canonical reference; the two systems work side by side.

## Companion Skills

- [shadcn-syntax-field](../shadcn-syntax-field/SKILL.md) : Field/FieldLabel/FieldDescription/FieldError primitives (the newer 2026 path).
- [shadcn-impl-form-validation](../../shadcn-impl/shadcn-impl-form-validation/SKILL.md) : end-to-end recipe (schema, layout, server action submit, error display, async refine).
- [shadcn-errors-form-state](../../shadcn-errors/shadcn-errors-form-state/SKILL.md) : top failure modes (Controller vs register confusion, watch over-rendering, async stale state).
- [shadcn-syntax-button](../shadcn-syntax-button/SKILL.md) : submit button variants, `disabled={form.formState.isSubmitting}` pattern.
- [shadcn-syntax-selectors](../shadcn-syntax-selectors/SKILL.md) : Select / Combobox / Native Select binding via Controller.
- [shadcn-syntax-input-otp](../shadcn-syntax-input-otp/SKILL.md) : OTP input binding via Controller.

## Reference files

- [references/methods.md](references/methods.md) : complete API signatures (Form, FormField, FormItem, FormLabel, FormControl, FormDescription, FormMessage, useFormField, useForm options, Controller props, zodResolver, common zod schema patterns).
- [references/examples.md](references/examples.md) : sign-in form, Select + Controller, Checkbox + Controller, native Input via register, async refine, defaultValues vs values, custom FormField composition.
- [references/anti-patterns.md](references/anti-patterns.md) : seven canonical anti-patterns with WHY each fails and the fix.

## Sources

All claims trace to URLs in `SOURCES.md` :

- https://ui.shadcn.com/docs/components/radix/form (Form composition canonical)
- https://ui.shadcn.com/docs/forms/react-hook-form (current Field-based pattern)
- https://react-hook-form.com/docs/useform (useForm signature)
- https://react-hook-form.com/docs/usecontroller/controller (Controller pattern)
- https://zod.dev (zod schema basics)
- https://github.com/react-hook-form/resolvers (zodResolver import path)
- https://github.com/shadcn-ui/ui : `apps/v4/registry/new-york-v4/ui/form.tsx` (canonical source)
- https://github.com/shadcn-ui/ui : `apps/v4/registry/new-york-v4/examples/form-rhf-*.tsx` (canonical examples)

Verified 2026-05-19.
