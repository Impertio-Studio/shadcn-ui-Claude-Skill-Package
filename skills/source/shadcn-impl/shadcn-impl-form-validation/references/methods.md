# shadcn-impl-form-validation: methods reference

API signatures and option contracts for the end-to-end workflow. Verified against `react-hook-form` v7.x docs and `zod` v3.x docs (2026-05-19).

## useForm signature

```ts
import { useForm, type UseFormProps, type UseFormReturn } from "react-hook-form"

function useForm<TFieldValues = FieldValues, TContext = any, TTransformedValues = TFieldValues>(
  props?: UseFormProps<TFieldValues, TContext, TTransformedValues>
): UseFormReturn<TFieldValues, TContext, TTransformedValues>
```

### UseFormProps fields used in this skill

| Option | Type | Purpose |
|--------|------|---------|
| `resolver` | `Resolver<TFieldValues>` | External validation function. Pass `zodResolver(schema)` from `@hookform/resolvers/zod`. |
| `defaultValues` | `DefaultValues<TFieldValues>` | Set ONCE at mount. Must include every schema key. Async load supported (pass a Promise; `formState.isLoading` flips while pending). |
| `values` | `TFieldValues` | CONTROLLED. Re-resets the form on each prop change. Use when initial data arrives async after first render. |
| `mode` | `"onSubmit" \| "onBlur" \| "onChange" \| "onTouched" \| "all"` | When the resolver runs BEFORE the first submit. Default `"onSubmit"`. |
| `reValidateMode` | `"onSubmit" \| "onBlur" \| "onChange"` | When the resolver re-runs AFTER the first submit. Default `"onChange"`. |
| `shouldFocusError` | `boolean` | Default `true`. When `onValid` is skipped due to errors, focus the first invalid field. |
| `criteriaMode` | `"firstError" \| "all"` | Whether to collect every error per field or stop at the first. Default `"firstError"`. |
| `delayError` | `number` | Debounce error rendering by N ms after a typing burst. |
| `resetOptions` | `KeepStateOptions` | What survives a `values`-prop-driven reset (keepDirty, keepTouched, keepErrors, ...). |

### UseFormReturn fields used in this skill

| Field | Type | Purpose |
|-------|------|---------|
| `control` | `Control<TFieldValues>` | Passed to `<FormField control={...}>` or to `<Controller control={...}>`. |
| `register` | `(name) => UseFormRegisterReturn` | For native inputs only. Returns `{ name, onChange, onBlur, ref }`. |
| `handleSubmit` | `(onValid, onInvalid?) => (e) => Promise<void>` | Wrap as `onSubmit={form.handleSubmit(onValid)}`. |
| `setError` | `(name, error, opts?) => void` | Manually set a field error. See below. |
| `clearErrors` | `(name?) => void` | Clear a single field, an array of fields, or all errors when no arg. |
| `setValue` | `(name, value, opts?) => void` | Programmatic update. `opts.shouldValidate`, `opts.shouldDirty`, `opts.shouldTouch`. |
| `getValues` | `() => TFieldValues` | Read current values without subscribing. |
| `watch` | `(name?) => value` | Subscribe to a field; component re-renders on change. Use sparingly. |
| `reset` | `(values?, options?) => void` | Reset to defaultValues (no arg) or to provided values. |
| `trigger` | `(name?) => Promise<boolean>` | Imperatively run validation on a field, an array, or the whole form. |
| `formState` | `FormState<TFieldValues>` | See breakdown below. |

## zodResolver

```ts
import { zodResolver } from "@hookform/resolvers/zod"

function zodResolver<TSchema extends ZodSchema>(
  schema: TSchema,
  schemaOptions?: ParseParams,
  resolverOptions?: { mode?: "async" | "sync"; raw?: boolean }
): Resolver<z.infer<TSchema>>
```

ALWAYS instantiate the schema OUTSIDE the component and pass the SAME schema reference on every render. Re-creating the schema or wrapping it freshly in `zodResolver` invalidates react-hook-form's memoised resolver.

`resolverOptions.mode = "async"` is the default and is required for any `.refine(async ...)` schema. Setting `mode: "sync"` silently skips async refines.

## handleSubmit contract

```ts
form.handleSubmit(
  onValid: (data: TFieldValues, event?: BaseSyntheticEvent) => unknown | Promise<unknown>,
  onInvalid?: (errors: FieldErrors<TFieldValues>, event?: BaseSyntheticEvent) => unknown | Promise<unknown>
): (e?: BaseSyntheticEvent) => Promise<void>
```

Behaviour, in order:

1. Calls `event.preventDefault()` internally (the native form does NOT submit).
2. Runs the resolver against current values.
3. On success: sets `formState.isSubmitting = true`, calls `onValid(data, event)`, awaits the returned Promise, sets `formState.isSubmitting = false`, sets `formState.isSubmitSuccessful = true` if no throw.
4. On failure: sets `formState.errors`, calls `onInvalid(errors, event)` if provided, focuses the first invalid field if `shouldFocusError` is `true`.

ALWAYS pass the RESULT of calling `handleSubmit` to the form's `onSubmit` prop. NEVER pass the bare reference: `<form onSubmit={form.handleSubmit}>` will fire `handleSubmit(submitEvent)` and treat the event object as `onValid`.

## setError contract

```ts
form.setError(
  name: FieldPath<TFieldValues> | "root" | `root.${string}`,
  error: { type: string; message?: string; types?: MultipleFieldErrors },
  options?: { shouldFocus?: boolean }
): void
```

- `name`: a field path from the schema, or `"root"`, or `"root.<key>"` for form-level errors.
- `error.type`: an identifier for the error source. ALWAYS use `"server"` for server-returned errors so `formState.errors.fieldName?.type === "server"` can drive conditional UI.
- `error.message`: the human-readable message rendered by `<FormMessage />`.
- `options.shouldFocus`: when `true`, focuses the field (only for real schema fields, not for `root.*`).

Server errors persist until either:
1. `form.clearErrors(name)` is called, OR
2. `form.reset()` is called, OR
3. The user edits the field AND `reValidateMode` triggers re-validation (which the zod resolver runs and overwrites the manual error).

## clearErrors signature

```ts
form.clearErrors(name?: FieldPath | FieldPath[] | "root" | `root.${string}`): void
```

No argument clears every error. Pass `"root"` or `"root.serverError"` to clear a form-level error. Pass an array to clear multiple fields.

## reset signature

```ts
form.reset(
  values?: TFieldValues | ((prev: TFieldValues) => TFieldValues),
  keepStateOptions?: KeepStateOptions
): void
```

Default behaviour with no args resets to the original `defaultValues`. Pass `values` to reset to a specific shape (useful after successful submit when the API returns canonical server values).

KeepStateOptions:

| Option | Default | Effect when `true` |
|--------|---------|---------------------|
| `keepErrors` | `false` | Preserves error map (rarely useful). |
| `keepDirty` | `false` | Preserves the `isDirty` flag and `dirtyFields` map. |
| `keepDirtyValues` | `false` | Resets defaultValues but preserves user-edited values. |
| `keepValues` | `false` | Preserves values; resets only meta-state (dirty, touched, errors). |
| `keepDefaultValues` | `false` | Preserves the previous defaultValues reference. |
| `keepIsSubmitted` | `false` | Preserves `isSubmitted`. |
| `keepTouched` | `false` | Preserves the touched map. |
| `keepIsValid` | `false` | Preserves `isValid`. |
| `keepSubmitCount` | `false` | Preserves `submitCount`. |

After a successful submit, the canonical idiom is `form.reset()` (no args) which clears values, errors, and meta-state.

## formState API

```ts
form.formState: {
  isDirty: boolean              // any field differs from defaultValues
  dirtyFields: FieldNamesMarkedBoolean
  touchedFields: FieldNamesMarkedBoolean
  isSubmitted: boolean          // handleSubmit was called at least once
  isSubmitSuccessful: boolean   // last onValid resolved without throwing
  isSubmitting: boolean         // onValid Promise is currently pending
  isLoading: boolean            // async defaultValues Promise still loading
  isValid: boolean              // resolver currently passes
  isValidating: boolean         // resolver is currently running (async refine in flight)
  submitCount: number
  defaultValues: Partial<TFieldValues>
  disabled: boolean             // form-level disabled flag
  errors: FieldErrors<TFieldValues>
}
```

ALWAYS read `isSubmitting` to drive submit-button disabled state during the network call. ALWAYS read `isValidating` to drive an inline spinner next to a field with `.refine(async ...)`. NEVER conflate `isLoading` with `isSubmitting`: `isLoading` is true ONLY while a Promise passed to `defaultValues` is pending.

## trigger signature

```ts
form.trigger(
  name?: FieldPath | FieldPath[],
  options?: { shouldFocus?: boolean }
): Promise<boolean>
```

Returns `true` if the named field(s) pass validation. Used in multi-step wizards: call `await form.trigger("step1")` before advancing to step 2.

ALWAYS use a string path (`"step1.email"`) or an array of paths for partial validation. NEVER omit `name` in a wizard (`form.trigger()` validates the WHOLE schema and may reject step 2 fields that the user has not yet seen).

## Controller props (when not using FormField)

```ts
<Controller
  control={form.control}
  name="<schema-key>"
  defaultValue=""
  rules={{}}                         // optional non-resolver rules
  shouldUnregister={false}
  disabled={false}
  render={({ field, fieldState, formState }) => ...}
/>
```

`field` exposes `{ name, value, onChange, onBlur, ref, disabled }`. `fieldState` exposes `{ invalid, isTouched, isDirty, error }`. `formState` is the whole form state.

ALWAYS spread `field` onto the inner input when the input accepts `value` + `onChange` (Radix-wrapped shadcn controls). NEVER spread `field` onto a native file input: the `value` prop on file inputs is read-only.

## zod async refine patterns

```ts
// Single-field async refine
z.string().email().refine(
  async (email) => {
    const res = await fetch(`/api/check-email?email=${encodeURIComponent(email)}`)
    const { taken } = await res.json()
    return !taken
  },
  { message: "Email already in use." }
)

// Cross-field refine (path attaches the error to one field)
z.object({ password: z.string(), confirm: z.string() })
  .refine((data) => data.password === data.confirm, {
    message: "Passwords do not match.",
    path: ["confirm"],
  })

// superRefine for multi-error reporting
z.object({ password: z.string() }).superRefine(({ password }, ctx) => {
  if (!/[A-Z]/.test(password)) ctx.addIssue({ code: z.ZodIssueCode.custom, message: "Need uppercase.", path: ["password"] })
  if (!/[0-9]/.test(password)) ctx.addIssue({ code: z.ZodIssueCode.custom, message: "Need digit.", path: ["password"] })
})
```

ALWAYS set `criteriaMode: "all"` on `useForm` when using `superRefine` with multiple `addIssue` calls; otherwise only the first error per field renders.

## FileList validation pattern

```ts
const MAX_BYTES = 5 * 1024 * 1024
const ACCEPTED = ["application/pdf", "image/png", "image/jpeg"]

z.object({
  upload: z.instanceof(FileList)
    .refine((files) => files.length === 1, "Pick one file.")
    .refine((files) => files[0]?.size <= MAX_BYTES, "Max 5 MB.")
    .refine((files) => ACCEPTED.includes(files[0]?.type), "PDF, PNG, or JPEG."),
})
```

ALWAYS guard `files[0]?.size` (optional chaining); a `length: 0` FileList has no element 0 and a non-optional access throws inside refine.

## FormData submission

```ts
async function onSubmit(values: FormValues) {
  const fd = new FormData()
  for (const [key, value] of Object.entries(values)) {
    if (value instanceof FileList) {
      Array.from(value).forEach((file) => fd.append(key, file))
    } else {
      fd.append(key, value as string)
    }
  }
  await fetch("/api/submit", { method: "POST", body: fd })
}
```

NEVER set the `Content-Type` header manually when sending FormData; `fetch` will set it to `multipart/form-data; boundary=...` automatically.

## Sources

- https://react-hook-form.com/docs/useform
- https://react-hook-form.com/docs/useform/handlesubmit
- https://react-hook-form.com/docs/useform/seterror
- https://react-hook-form.com/docs/useform/clearerrors
- https://react-hook-form.com/docs/useform/reset
- https://react-hook-form.com/docs/useform/trigger
- https://react-hook-form.com/docs/useform/formstate
- https://react-hook-form.com/docs/usecontroller/controller
- https://github.com/react-hook-form/resolvers
- https://zod.dev
- https://zod.dev/?id=refine
- https://zod.dev/?id=superrefine

Verified 2026-05-19.
