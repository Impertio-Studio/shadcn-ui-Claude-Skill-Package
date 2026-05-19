---
name: shadcn-impl-form-validation
description: >
  Use when building an end-to-end form workflow in shadcn ui that combines the
  Form composition primitives with react-hook-form and zod, when wiring the
  full pipeline from zod schema to typed submit handler to server-error
  display, when adding async validation (email availability, username taken,
  coupon check), when surfacing API errors back to a specific field via
  setError, when toggling a submit button between idle, submitting, success,
  and disabled states, when resetting a form after successful submit to
  clear stale state, when building a multi-step wizard that preserves form
  state across pages, or when wiring a file upload control into the
  react-hook-form value tree.
  Prevents the common end-to-end form failures: defining the zod schema
  inside the component so the resolver re-creates every render, mixing
  schema validation with manual setError on the same field so the two
  systems race and the user sees flickering error text, omitting
  form.handleSubmit on the form element so the browser submits natively
  and reloads the page with no validation at all, never calling form.reset
  after a successful submit so the form retains the just-submitted data
  and the user accidentally re-submits, reading form.formState.isLoading
  instead of form.formState.isSubmitting (isLoading reflects async
  defaultValues loading, not submit state), leaving a defaultValues key
  undefined so the first user keystroke triggers a controlled-to-uncontrolled
  warning, rendering FormDescription outside FormItem so the aria-describedby
  wiring never finds the description id, and binding a File input with
  register so the FileList is lost on validation re-run.
  Covers the five-step end-to-end workflow (schema -> useForm -> compose ->
  submit -> error-handling), the Controller-versus-register decision matrix
  per shadcn input, the async refine pattern with debounce, the
  setError-after-server-error pattern that maps API errors back onto
  specific fields, the isSubmitting submit-button-disabled pattern, the
  form.reset on successful submit pattern, the multi-step wizard pattern
  via a single FormProvider plus useFormContext across pages, the file
  upload via Controller plus FileList plus zod refine pattern, and the
  exact onValid plus onInvalid handleSubmit signature.
  This skill orchestrates the react-hook-form path. The Field-plus-Controller
  path covered in shadcn-syntax-field is the recommended 2026 baseline for
  new projects; see shadcn-syntax-form for the primitive composition tree.
  Keywords: shadcn form workflow, zod form, form validation end-to-end,
  async form validation, server error setError, form reset, isSubmitting,
  multi-step form, file upload form, how do I validate a form,
  react-hook-form zodResolver, zodResolver hookform, z.infer schema,
  email availability check, coupon code validation, debounced async refine,
  setError manual, mapServerErrorsToFields, form.reset after submit,
  controlled to uncontrolled warning, form does not submit,
  submit button stays enabled, submit button spinner,
  multi-step wizard form, wizard form state, FormProvider wizard,
  useFormContext step, file input react-hook-form, FileList Controller,
  zod refine async, zod superRefine, form.formState.isSubmitting,
  form.formState.isValidating, FieldErrors, onValid onInvalid,
  api error to field, server validation react-hook-form,
  why does my form reload the page, why is my submit handler not called.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui: end-to-end Form workflow (zod + react-hook-form)

This skill is the IMPLEMENTATION recipe that orchestrates the primitives covered in `shadcn-syntax-form`, `shadcn-syntax-field`, and `shadcn-syntax-selectors`. It teaches the five-step workflow from zod schema definition to typed submit handler to server-error display.

ALWAYS read `shadcn-syntax-form` first if the seven Form primitives are unclear. ALWAYS read `shadcn-syntax-selectors` first if binding Select, Checkbox, RadioGroup, or Switch to Controller is unclear.

## Quick Reference

### Canonical end-to-end form

```tsx
"use client"

import { zodResolver } from "@hookform/resolvers/zod"
import { useForm } from "react-hook-form"
import * as z from "zod"

import { Button } from "@/components/ui/button"
import {
  Form, FormControl, FormDescription, FormField,
  FormItem, FormLabel, FormMessage,
} from "@/components/ui/form"
import { Input } from "@/components/ui/input"
import { toast } from "sonner"

const signInSchema = z.object({
  email: z.string().min(1, "Required.").email("Invalid email."),
  password: z.string().min(8, "At least 8 characters."),
})

type SignInValues = z.infer<typeof signInSchema>

export function SignInForm() {
  const form = useForm<SignInValues>({
    resolver: zodResolver(signInSchema),
    defaultValues: { email: "", password: "" },
  })

  async function onSubmit(values: SignInValues) {
    const res = await fetch("/api/sign-in", {
      method: "POST",
      body: JSON.stringify(values),
    })
    if (!res.ok) {
      const { fieldErrors } = await res.json()
      for (const [name, message] of Object.entries(fieldErrors ?? {})) {
        form.setError(name as keyof SignInValues, { type: "server", message: String(message) })
      }
      return
    }
    toast.success("Signed in.")
    form.reset()
  }

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-6">
        <FormField control={form.control} name="email" render={({ field }) => (
          <FormItem>
            <FormLabel>Email</FormLabel>
            <FormControl><Input type="email" autoComplete="email" {...field} /></FormControl>
            <FormMessage />
          </FormItem>
        )} />
        <FormField control={form.control} name="password" render={({ field }) => (
          <FormItem>
            <FormLabel>Password</FormLabel>
            <FormControl><Input type="password" autoComplete="current-password" {...field} /></FormControl>
            <FormMessage />
          </FormItem>
        )} />
        <Button type="submit" disabled={form.formState.isSubmitting}>
          {form.formState.isSubmitting ? "Signing in..." : "Sign in"}
        </Button>
      </form>
    </Form>
  )
}
```

### Controller vs register decision matrix

| shadcn input | Path | Field binding pattern |
|--------------|------|------------------------|
| `Input`, `Textarea` | Controller (via FormField) | `<Input {...field} />` |
| `Select` | Controller | `<Select value={field.value} onValueChange={field.onChange}>` |
| `Checkbox` | Controller | `<Checkbox checked={field.value} onCheckedChange={field.onChange}>` |
| `RadioGroup` | Controller | `<RadioGroup value={field.value} onValueChange={field.onChange}>` |
| `Switch` | Controller | `<Switch checked={field.value} onCheckedChange={field.onChange}>` |
| `Slider` | Controller | `<Slider value={field.value} onValueChange={field.onChange}>` |
| `InputOTP` | Controller | `<InputOTP value={field.value} onChange={field.onChange}>` |
| Native `<input type="file">` | Controller (NEVER register) | `<input onChange={(e) => field.onChange(e.target.files)}>` |
| Native `<input type="text">` | register permitted | `<input {...form.register("name")}>` (loses aria wiring) |

NEVER bind a Radix-wrapped shadcn control with `register`. NEVER bind a file input with `register` because the FileList resets on every re-render.

## The five-step end-to-end workflow

### Step 1: define the zod schema (module scope)

ALWAYS define the schema OUTSIDE the component body and infer the TypeScript type via `z.infer`. NEVER inline the schema inside the component: the schema object would be re-created every render, `zodResolver(schema)` would also re-create, and react-hook-form's memoised resolver reference would invalidate every render.

```tsx
const signUpSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
  confirmPassword: z.string(),
}).refine((data) => data.password === data.confirmPassword, {
  message: "Passwords do not match.",
  path: ["confirmPassword"],
})

type SignUpValues = z.infer<typeof signUpSchema>
```

ALWAYS use `.refine` (not `.superRefine`) for single-field or cross-field rules with a single message. ALWAYS set `path: [<field-name>]` so the error attaches to a specific field; without `path`, the error attaches to the form root and `<FormMessage />` will not render it.

### Step 2: instantiate useForm with zodResolver

ALWAYS pass `resolver: zodResolver(schema)` and explicit `defaultValues` for every field declared in the schema. ALWAYS pass the inferred type as the generic to `useForm<SignUpValues>` so `form.control`, `form.setValue`, and the `onSubmit` data argument are all type-safe.

```tsx
const form = useForm<SignUpValues>({
  resolver: zodResolver(signUpSchema),
  defaultValues: { email: "", password: "", confirmPassword: "" },
  mode: "onSubmit",     // when to validate: onSubmit | onBlur | onChange | onTouched | all
  reValidateMode: "onChange",
})
```

ALWAYS provide a default for every schema key (`""` for strings, `false` for booleans, `0` or null for numbers, `[]` for arrays, `null` for files). NEVER leave a default `undefined`: react-hook-form treats `undefined` as uncontrolled and React warns on the first keystroke.

### Step 3: compose Form primitives

ALWAYS spread `{...form}` on `<Form>`, ALWAYS pass `onSubmit={form.handleSubmit(onValid)}` to the inner `<form>` element, ALWAYS wrap every field in `<FormField control={form.control} name="<schema-key>" render={({ field }) => (...)}>`, ALWAYS render exactly one `<FormItem>` per field, and ALWAYS include `<FormMessage />` even when no description is present. See `shadcn-syntax-form` for the seven-primitive composition tree.

### Step 4: handle submit (onValid + onInvalid)

`form.handleSubmit(onValid, onInvalid?)` returns a single event-handler function. ALWAYS pass the RESULT of calling `handleSubmit`, never `handleSubmit` itself.

```tsx
async function onValid(values: SignUpValues) {
  // values are zod-parsed and type-safe.
  // ... call API, handle server errors, reset on success.
}

function onInvalid(errors: FieldErrors<SignUpValues>) {
  // optional: runs only when client-side validation fails.
}

<form onSubmit={form.handleSubmit(onValid, onInvalid)}>...</form>
```

While `onValid` returns a pending Promise, `form.formState.isSubmitting` is `true`. ALWAYS disable the submit button via `disabled={form.formState.isSubmitting}` so the user cannot double-submit. NEVER read `form.formState.isLoading` for this purpose: `isLoading` reflects async `defaultValues` loading, not submit progress.

### Step 5: error handling (server + reset)

ALWAYS map server-returned field errors back via `form.setError(fieldName, { type: "server", message })` AFTER the API call resolves. ALWAYS call `form.reset()` after a successful submit so the form clears stale data (unless the form is an edit-existing-record flow where the user expects to keep the values).

```tsx
async function onSubmit(values: SignUpValues) {
  const res = await fetch("/api/sign-up", { method: "POST", body: JSON.stringify(values) })
  if (!res.ok) {
    const { fieldErrors, formError } = await res.json()
    if (formError) {
      form.setError("root.serverError", { type: "server", message: formError })
    }
    for (const [name, message] of Object.entries(fieldErrors ?? {})) {
      form.setError(name as keyof SignUpValues, { type: "server", message: String(message) })
    }
    return
  }
  toast.success("Account created.")
  form.reset()
}
```

ALWAYS use `"root.<key>"` (e.g. `"root.serverError"`) for non-field-specific errors. The error is readable from `form.formState.errors.root?.serverError?.message` and survives until the next `form.clearErrors("root.serverError")` or `form.reset()`.

## Async validation pattern

zod supports async refinement via `.refine(async ...)` and `.superRefine(async ...)`. `form.formState.isValidating` is `true` while at least one async check is in flight. Use this for email availability, username taken, coupon code checks.

ALWAYS debounce the network call (300 to 500 ms) so each keystroke does not fire a request. ALWAYS handle the stale-response case by comparing the awaited value against the current schema input value (or by aborting prior requests via `AbortController`).

```tsx
const signUpSchema = z.object({
  email: z.string().email().refine(
    async (email) => {
      const res = await fetch(`/api/check-email?email=${encodeURIComponent(email)}`)
      const { taken } = await res.json()
      return !taken
    },
    { message: "Email already in use." }
  ),
})
```

ALWAYS `await` every async check inside `.refine`. NEVER fire-and-forget an async call from `.refine`: the resolver expects a boolean (or boolean Promise) and a stale resolution after the user has moved on can flicker the error state. See `references/examples.md` for the AbortController + debounce pattern.

## Server-side error mapping (setError)

Server validation errors that the client-side schema cannot catch (e.g. "email already taken", "invalid coupon code", "rate-limited") MUST be mapped back to specific fields via `form.setError`. ALWAYS use `type: "server"` so the error can be distinguished from client-side validation errors. ALWAYS clear server errors before re-submitting (`form.clearErrors()` runs implicitly via `handleSubmit`, but explicit `clearErrors` is safer for partial resubmits).

```tsx
// Server returns: { fieldErrors: { email: "Email already taken." }, formError: null }
for (const [name, message] of Object.entries(response.fieldErrors)) {
  form.setError(name as keyof FormValues, { type: "server", message: String(message) })
}
```

NEVER mix client-side zod validation and manual `setError` on the same field within the same render cycle: the next user keystroke triggers a re-validation that wipes the manual error. ALWAYS rely on the zod schema for everything the client can check and use `setError` ONLY for errors the server discovers.

## Loading state on submit button

ALWAYS read `form.formState.isSubmitting` (not `isLoading`, not `isValidating`) to drive the submit button's disabled state.

```tsx
<Button type="submit" disabled={form.formState.isSubmitting || !form.formState.isValid}>
  {form.formState.isSubmitting ? (
    <><Loader2 className="size-4 animate-spin" /> Saving...</>
  ) : "Save"}
</Button>
```

| Flag | Meaning |
|------|---------|
| `isSubmitting` | `onValid` returned a Promise that is still pending. |
| `isValidating` | A zod async `.refine` (or any async validator) is running. |
| `isLoading` | Async `defaultValues` (a Promise passed to `useForm`'s `defaultValues`) is still loading. |
| `isSubmitSuccessful` | The last `onValid` call resolved without throwing. |
| `isValid` | The form passes the resolver. |

ALWAYS gate the submit button on `isSubmitting` alone unless the schema is cheap to validate; gating on `!isValid` while async validation is in flight visibly flickers the button.

## File upload pattern

File inputs MUST be bound via Controller (not register) because react-hook-form copies the FileList by reference and a re-render can wipe the value. ALWAYS store the `FileList` in form state; validate via `.refine` for file count, size, and MIME type.

```tsx
const schema = z.object({
  resume: z.instanceof(FileList)
    .refine((files) => files.length === 1, "Upload one file.")
    .refine((files) => files[0]?.size <= 5 * 1024 * 1024, "Max 5 MB.")
    .refine((files) => ["application/pdf"].includes(files[0]?.type), "PDF only."),
})
```

```tsx
<FormField control={form.control} name="resume" render={({ field: { onChange, value, ...rest } }) => (
  <FormItem>
    <FormLabel>Resume</FormLabel>
    <FormControl>
      <Input type="file" accept="application/pdf" {...rest}
        onChange={(e) => onChange(e.target.files)} />
    </FormControl>
    <FormMessage />
  </FormItem>
)} />
```

ALWAYS destructure `value` away from the spread (a file input cannot have a controlled `value` prop in React). ALWAYS forward `e.target.files` (FileList), not `e.target.files[0]` (single File). For multipart upload to the server, build a `FormData` object in `onSubmit`.

## Multi-step wizard pattern

Preserve form state across pages by wrapping every step in a single `<FormProvider>` (the same context that `<Form>` provides) and reading via `useFormContext` in each step component. The wizard renders ONE step at a time but the form state is shared.

```tsx
import { FormProvider, useFormContext, useForm } from "react-hook-form"

const wizardSchema = z.object({
  step1: z.object({ name: z.string().min(1), email: z.string().email() }),
  step2: z.object({ address: z.string().min(1), city: z.string().min(1) }),
  step3: z.object({ agree: z.literal(true) }),
})
type WizardValues = z.infer<typeof wizardSchema>

export function Wizard() {
  const form = useForm<WizardValues>({
    resolver: zodResolver(wizardSchema),
    defaultValues: { step1: { name: "", email: "" }, step2: { address: "", city: "" }, step3: { agree: false } },
    mode: "onChange",
  })
  const [step, setStep] = useState(0)
  const steps = [Step1, Step2, Step3]
  const StepComponent = steps[step]

  async function next() {
    const stepKey = `step${step + 1}` as keyof WizardValues
    const ok = await form.trigger(stepKey)
    if (ok) setStep((s) => s + 1)
  }

  return (
    <FormProvider {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)}>
        <StepComponent />
        {step < steps.length - 1 ? (
          <Button type="button" onClick={next}>Next</Button>
        ) : (
          <Button type="submit" disabled={form.formState.isSubmitting}>Submit</Button>
        )}
      </form>
    </FormProvider>
  )
}

function Step1() {
  const form = useFormContext<WizardValues>()
  return <FormField control={form.control} name="step1.name" render={...} />
}
```

ALWAYS call `form.trigger(stepKey)` before advancing to validate only the current step's fields. NEVER unmount form state between steps: react-hook-form preserves values across renders, but unmounting a `<FormProvider>` clears state. The `FormProvider` MUST be a stable ancestor of every step.

## Companion Skills

- [shadcn-syntax-form](../../shadcn-syntax/shadcn-syntax-form/SKILL.md) (Batch 3): the seven Form primitives and their composition tree.
- [shadcn-syntax-field](../../shadcn-syntax/shadcn-syntax-field/SKILL.md) (Batch 4): the new 2026 Field primitive family that decouples a11y wiring from react-hook-form.
- [shadcn-syntax-selectors](../../shadcn-syntax/shadcn-syntax-selectors/SKILL.md) (Batch 5): binding Select, Combobox, Command palette via Controller.
- [shadcn-errors-form-state](../../shadcn-errors/shadcn-errors-form-state/SKILL.md) (Batch 12): top failure modes (Controller vs register confusion, watch over-rendering, async stale state).
- [shadcn-syntax-button](../../shadcn-syntax/shadcn-syntax-button/SKILL.md): submit button variants and `disabled={form.formState.isSubmitting}` pattern.

## Reference files

- [references/methods.md](references/methods.md): full API signatures (useForm options, zodResolver, handleSubmit, setError, clearErrors, reset, trigger, formState fields, Controller props, common zod async patterns).
- [references/examples.md](references/examples.md): minimal sign-in, sign-up with confirm-password, async email-availability with AbortController, server-error setError, file upload with FileList, multi-step wizard with FormProvider, isSubmitting submit button.
- [references/anti-patterns.md](references/anti-patterns.md): six canonical end-to-end anti-patterns with WHY each fails and the fix.

## Sources

All claims trace to URLs in `SOURCES.md`:

- https://ui.shadcn.com/docs/components/radix/form (Form composition canonical)
- https://ui.shadcn.com/docs/forms/react-hook-form (current Field-based react-hook-form pattern)
- https://react-hook-form.com/docs/useform (useForm signature, formState fields)
- https://react-hook-form.com/docs/useform/seterror (setError contract)
- https://react-hook-form.com/docs/useform/handlesubmit (handleSubmit contract)
- https://react-hook-form.com/docs/useform/reset (reset contract)
- https://react-hook-form.com/docs/useformcontext (multi-step wizard pattern)
- https://zod.dev (zod schema basics)
- https://zod.dev/?id=refine (refine + async refine)
- https://github.com/react-hook-form/resolvers (zodResolver import path)

Verified 2026-05-19.
