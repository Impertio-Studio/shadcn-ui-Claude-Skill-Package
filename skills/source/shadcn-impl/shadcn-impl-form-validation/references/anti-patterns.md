# shadcn-impl-form-validation: anti-patterns

Six canonical end-to-end form failures. Each entry: WHAT is wrong, WHY it fails, the FIX.

## 1. Mixing zod validation and manual setError on the same field (race condition)

**Wrong**:

```tsx
const schema = z.object({
  email: z.string().email("Invalid email."),
})

async function onSubmit(values: FormValues) {
  if (await isEmailTaken(values.email)) {
    form.setError("email", { type: "server", message: "Email taken." })
    return
  }
  // ...
}

// And on every keystroke the zod resolver re-runs and wipes the "Email taken." error.
```

**Why it fails**: react-hook-form runs the resolver on every change after `mode: "onChange"` / `reValidateMode: "onChange"`. The resolver returns ONLY the zod errors. The manual `setError` on `email` is overwritten as soon as the user types another character. The user sees the error flicker on and off.

**Fix**: either move the check into a zod `.refine(async ...)` so the resolver owns the rule, OR shift the manual error to a different "owner" (`root.serverError` or a sibling read-only field) that the resolver never touches.

```tsx
// Option A: own the rule in the schema
const schema = z.object({
  email: z.string().email().refine(
    async (v) => !(await isEmailTaken(v)),
    { message: "Email taken." }
  ),
})

// Option B: surface the error at root.serverError, not on the field
form.setError("root.serverError", { type: "server", message: "Email taken." })
```

## 2. Forgetting form.handleSubmit so the browser submits natively

**Wrong**:

```tsx
<form onSubmit={onSubmit}>
  {/* fields */}
  <Button type="submit">Submit</Button>
</form>
```

**Why it fails**: passing `onSubmit` directly (or omitting `onSubmit` entirely) sends the form to the browser's default submit handler. The browser serialises field values into a GET query string, navigates to the current URL with the query appended, and the page reloads. NO zod validation runs, NO `formState.errors` populate, the user just sees the page flash.

A second variant of the same bug:

```tsx
<form onSubmit={form.handleSubmit}>           // passes the function reference
```

This calls `handleSubmit(submitEvent)` and treats the event object as `onValid`. The form does not reload (because `handleSubmit` calls `preventDefault` internally) but the actual submit logic never runs.

**Fix**: ALWAYS wrap your handler:

```tsx
<form onSubmit={form.handleSubmit(onValid)}>
```

`onValid` receives the zod-parsed, type-safe data. Pass the RESULT of calling `handleSubmit`, never the bare reference.

## 3. Not resetting the form after successful submit (stale state re-submit)

**Wrong**:

```tsx
async function onSubmit(values: SignUpValues) {
  await api.signUp(values)
  toast.success("Account created.")
  // form retains email, password, confirmPassword filled in
}
```

**Why it fails**: the form fields still hold the just-submitted data. The submit button is enabled again (because `isSubmitting` flipped back to `false`). A quick double-click, a refresh, or a back-button-then-resubmit creates a duplicate account. Worse for editable lists: the user may not realise they are editing the OLD record's values when they think they are creating a new one.

**Fix**: ALWAYS reset on success.

```tsx
async function onSubmit(values: SignUpValues) {
  await api.signUp(values)
  toast.success("Account created.")
  form.reset()                         // back to defaultValues
}
```

For edit-existing-record flows where the user expects values to remain (e.g. profile editor), call `form.reset(values)` with the canonical server response so `isDirty` flips back to `false` while the visible values stay.

EXCEPTION: in a wizard, do NOT reset between steps. Reset only after the final submit.

## 4. Reading form.formState.isLoading instead of isSubmitting

**Wrong**:

```tsx
<Button type="submit" disabled={form.formState.isLoading}>
  {form.formState.isLoading ? "Saving..." : "Save"}
</Button>
```

**Why it fails**: `form.formState.isLoading` reflects the loading state of an ASYNC `defaultValues` Promise (a `useForm({ defaultValues: fetchUser() })` pattern). It is `true` ONLY while the initial values are loading at mount and is `false` for the entire rest of the form's lifecycle. The submit button never goes into a loading state, the user double-clicks, and two API calls fire.

**Fix**: read `isSubmitting`.

```tsx
<Button type="submit" disabled={form.formState.isSubmitting}>
  {form.formState.isSubmitting ? "Saving..." : "Save"}
</Button>
```

Quick reference:

| Flag | When `true` |
|------|-------------|
| `isSubmitting` | `onValid` Promise is pending. THE one for submit buttons. |
| `isValidating` | A `.refine(async ...)` is running. Drive an inline field spinner. |
| `isLoading` | Async `defaultValues` Promise is pending. Only during first mount. |

## 5. defaultValues undefined causing controlled-to-uncontrolled warning

**Wrong**:

```tsx
const schema = z.object({ name: z.string(), age: z.number() })

const form = useForm<z.infer<typeof schema>>({
  resolver: zodResolver(schema),
  defaultValues: { name: "" },                  // age missing
})
```

**Why it fails**: react-hook-form treats the missing `age` as `undefined`. The shadcn `<Input>` renders with `value={undefined}` (uncontrolled). The first user keystroke flips it to controlled (`value={"5"}`), and React logs the warning:

```
Warning: A component is changing an uncontrolled input to be controlled.
This is likely caused by the value changing from undefined to a defined value, which should not happen.
```

The form still works, but the warning is a sign that the value was not present at mount.

**Fix**: ALWAYS provide a default for every schema key. Match the type:

| Schema type | Sensible default |
|-------------|------------------|
| `z.string()` | `""` |
| `z.number()` | `0` (or `null` with `z.number().nullable()`) |
| `z.boolean()` | `false` |
| `z.array(...)` | `[]` |
| `z.object(...)` | `{ ...nested defaults }` |
| `z.literal(true)` | `false as unknown as true` (the user toggles to `true` to satisfy) |
| `z.instanceof(FileList)` | `undefined as unknown as FileList` (the input never receives a `value` prop, so no warning fires) |

```tsx
const form = useForm<z.infer<typeof schema>>({
  resolver: zodResolver(schema),
  defaultValues: { name: "", age: 0 },          // every key present
})
```

## 6. FormDescription nested outside FormItem (or outside FormField)

**Wrong**:

```tsx
<FormField control={form.control} name="email" render={({ field }) => (
  <>
    <FormDescription>We will not share your email.</FormDescription>
    <FormItem>
      <FormLabel>Email</FormLabel>
      <FormControl><Input {...field} /></FormControl>
      <FormMessage />
    </FormItem>
  </>
)} />
```

A second variant:

```tsx
<FormField control={form.control} name="email" render={({ field }) => (
  <FormItem>
    <FormLabel>Email</FormLabel>
    <FormControl><Input {...field} /></FormControl>
    <FormMessage />
  </FormItem>
)} />
<FormDescription>We will not share your email.</FormDescription>  // outside FormField
```

**Why it fails**: `FormDescription` calls `useFormField()` to read `formDescriptionId` from `FormItemContext`. The context provider is `FormItem`, and `FormItem` is rendered by `FormField`'s `render` prop. When `FormDescription` is rendered outside `FormItem`, `useFormField()` throws "useFormField should be used within <FormField>". When it is rendered outside `FormField` entirely, the same throw fires from the outer FormFieldContext check.

Even if you suppress the throw (e.g. by using a generic `<p>` instead), the description's `id` is never wired into `FormControl`'s `aria-describedby`. Screen readers skip the helper text.

**Fix**: ALWAYS place `FormDescription` INSIDE the same `FormItem` that contains the field, between `FormControl` and `FormMessage`.

```tsx
<FormField control={form.control} name="email" render={({ field }) => (
  <FormItem>
    <FormLabel>Email</FormLabel>
    <FormControl><Input {...field} /></FormControl>
    <FormDescription>We will not share your email.</FormDescription>
    <FormMessage />
  </FormItem>
)} />
```

The canonical order inside `<FormItem>` is: `FormLabel`, `FormControl`, `FormDescription`, `FormMessage`. Any other order works at runtime, but this order matches the visual + a11y reading order.

## Sources

- https://ui.shadcn.com/docs/components/radix/form
- https://react-hook-form.com/docs/useform/handlesubmit
- https://react-hook-form.com/docs/useform/seterror
- https://react-hook-form.com/docs/useform/reset
- https://react-hook-form.com/docs/useform/formstate
- https://github.com/shadcn-ui/ui (apps/v4/registry/new-york-v4/ui/form.tsx)
- https://react.dev/reference/react-dom/components/input#controlling-an-input-with-a-state-variable

Verified 2026-05-19.
