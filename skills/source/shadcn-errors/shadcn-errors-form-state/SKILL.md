---
name: shadcn-errors-form-state
description: >
  Use when a shadcn Form built on react-hook-form silently submits with missing
  values, when a Radix Select or Checkbox or RadioGroup or Switch never updates
  form state, when FormMessage refuses to render the zod error, when the
  console logs "A component is changing an uncontrolled input to be controlled"
  on first keystroke, when the entire form re-renders on every character typed,
  when an async zod refine never blocks submission, when handleSubmit fires
  with no validation, when formState.isDirty or formState.isValid stays stuck
  at false, when reset() does not clear errors, when a file input loses its
  FileList after the first render, when a server validation response should
  light up specific field errors, when defaultValues is partially specified
  and some fields jump from uncontrolled to controlled, or when a typo in the
  zod schema field path causes a field to never validate without warning.
  Prevents the six recurring react-hook-form failures inside shadcn Form
  composition : using register on a non-native control (Select, Checkbox,
  RadioGroup, Switch) so its value never enters form state, calling form.watch
  inside the component body so every keystroke re-renders the entire form,
  passing defaultValues with undefined fields so React fires the controlled
  versus uncontrolled warning on first interaction, nesting FormField outside
  the Form provider so the inner context lookup returns null, defining the zod
  schema with a field path that does not match the FormField name string, and
  passing only one argument to handleSubmit so invalid submissions vanish
  silently with no onInvalid handler.
  Covers the Controller-versus-register decision matrix, the watch-versus-
  useWatch re-render trap, the defaultValues-versus-values-versus-reset triad
  and when each is correct, the FormMessage context-binding requirement, the
  resolver schema-path-must-match-FormField-name rule, the formState
  subscription model and which keys must be read to subscribe, the
  handleSubmit(onValid, onInvalid) two-argument signature, the async refine
  pattern with debounce and AbortController, and the file-input Controller
  pattern with manual FileList handling.
  Keywords: form state error, Controller vs register, react-hook-form watch
  re-render, FormMessage not showing, defaultValues undefined, controlled
  uncontrolled warning, zod refine async, form silent submit, form errors not
  displaying, register on shadcn Select, Select onValueChange not firing,
  Checkbox onCheckedChange ignored, useWatch vs watch, useFormContext returned
  null, FormField outside Form, schema path mismatch, handleSubmit onInvalid,
  isValid stuck false, isDirty never true, reset does not clear errors,
  setError server error, file input react-hook-form, FileList disappears,
  formState subscription, isValidating, useFormState, async validation,
  email already taken, debounce refine, zodResolver typo, my form does not
  submit, my form submits empty values, why is my Select not validated.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui Errors: Form State and react-hook-form Integration

When a shadcn Form silently submits with missing values, when a Select never validates, or when FormMessage refuses to show the zod error, the cause is almost always one of six things : (1) `register` was used on a non-native control whose value never reaches `form.control`, (2) `form.watch` was called inline in the component body and re-rendered the whole subtree on every keystroke, (3) `defaultValues` was incomplete so a field flipped from uncontrolled to controlled on first interaction, (4) `FormField` was rendered outside the `<Form>` provider so the context returned null, (5) the zod schema path does not match the `name` string in `FormField` (typo), or (6) `handleSubmit` was called with only the success handler so invalid submissions vanished silently. This skill enumerates the deterministic rules and the verified API shapes so the diagnosis takes seconds.

Every claim traces to the shadcn Form recipe at `https://ui.shadcn.com/docs/forms/react-hook-form` and to the react-hook-form API at `https://react-hook-form.com/docs/useform`.

## Quick Reference

### Controller vs register matrix (memorize this)

For each input type, exactly one of `register` or `Controller` (via `FormField`) is correct. Picking the wrong one is the single most common cause of "my form submits empty values".

| Input type | Use | Reason |
|---|---|---|
| Native `<input>` (text, email, number, url, password) | `register` OR `Controller`/`FormField` | Both work ; `FormField` adds aria wiring automatically |
| Native `<textarea>` | `register` OR `Controller`/`FormField` | Same as text input |
| Native `<select>` (HTML element) | `register` OR `Controller`/`FormField` | Native select fires `change` event ; `register` listens for it |
| Native `<input type="checkbox">` (single boolean) | `register` | `register` uses the native `checked` property |
| Native `<input type="radio">` | `register` | Same as native checkbox |
| Native `<input type="file">` | `Controller`/`FormField` | `register` returns `{value, onChange}` but you need the `FileList` from `e.target.files`, NOT the string value |
| shadcn `Select` (Radix-wrapped) | `Controller`/`FormField` | Radix Select uses `onValueChange`, not `onChange` ; `register` listens for the wrong event |
| shadcn `Checkbox` (Radix-wrapped) | `Controller`/`FormField` | Radix Checkbox uses `onCheckedChange`, not `onChange` |
| shadcn `RadioGroup` (Radix-wrapped) | `Controller`/`FormField` | Radix RadioGroup uses `onValueChange` |
| shadcn `Switch` (Radix-wrapped) | `Controller`/`FormField` | Radix Switch uses `onCheckedChange` |
| shadcn `Slider` | `Controller`/`FormField` | Radix Slider value is `number[]` and uses `onValueChange` |
| shadcn `Combobox` / cmdk `Command` | `Controller`/`FormField` | Custom value model, no DOM `change` event |
| shadcn `Calendar` / `DatePicker` | `Controller`/`FormField` | `react-day-picker` value is a `Date` object via `onSelect` |
| Any third-party controlled component | `Controller`/`FormField` | Default rule : if it does not fire native `change`, use `Controller` |

The rule of thumb : if the prop is `onValueChange`, `onCheckedChange`, `onSelect`, or any non-`onChange` handler, `register` will silently fail and the field stays empty. ALWAYS use `FormField` (or raw `Controller`) for any non-native input.

### The watch over-rendering trap

`form.watch()` called inside the component body subscribes the WHOLE component (and every child) to re-render on every form value change. For a 10-field form, every keystroke re-renders all 10 fields. The fix is `useWatch({ control, name })`, which subscribes a single hook scope.

```tsx
// WRONG : every keystroke re-renders this component and every child.
function MyForm() {
  const form = useForm<FormData>({ ... })
  const username = form.watch("username")     // SUBSCRIPTION at component root
  return <Form {...form}>...</Form>
}

// RIGHT : useWatch subscribes only this hook's scope ; isolate it in a child.
function UsernameMirror({ control }: { control: Control<FormData> }) {
  const username = useWatch({ control, name: "username" })
  return <p>Hello, {username}</p>
}
```

A second valid option is `form.watch((values) => { ... })` with a callback. The callback form does NOT trigger re-renders : it fires as a side effect for logging, autosave, or analytics.

### defaultValues vs values vs reset

These three APIs all change the form's stored values. Picking the wrong one breaks either the initial render, the editing flow, or both.

| API | When to use | Behavior |
|---|---|---|
| `useForm({ defaultValues })` | Initial values are known at mount time | Set ONCE on mount ; subsequent prop changes do NOT update the form |
| `useForm({ values })` | Initial values arrive async (server fetch) and may change | Re-sets the form every time `values` reference changes ; resets dirty state |
| `form.reset(newValues)` | Imperative reset after submit or cancel | Replaces values + clears errors + clears dirty/touched ; call from event handler |
| `form.reset(undefined, { keepErrors: true })` | Reset values but keep validation errors | Selective reset via `keepValues`, `keepErrors`, `keepDirty`, `keepTouched` flags |

CRITICAL : `defaultValues` MUST specify every field that will become controlled. Omitting a field (or setting it to `undefined`) makes the field start as uncontrolled, then flip to controlled on first keystroke, which fires React's "switched from uncontrolled to controlled" warning AND can lose the first character.

```tsx
// WRONG : email is undefined ; first keystroke flips controlled state.
useForm<{ name: string; email: string }>({ defaultValues: { name: "" } })

// RIGHT : every field has an explicit initial value.
useForm<{ name: string; email: string }>({ defaultValues: { name: "", email: "" } })
```

For nullable database fields, use `""` for strings, `false` for booleans, `null` only if the input handles `null`, and `[]` for arrays. NEVER use `undefined`.

### FormMessage context-binding

`FormMessage` is NOT a generic message component. It reads the field's error message from a `FormField` context. Rendering it outside a `FormField` makes it inert.

```tsx
// WRONG : FormMessage has no FormField context ; it renders nothing.
<Form {...form}>
  <FormMessage />                          // empty, always
  <FormField name="email" .../>
</Form>

// RIGHT : FormMessage lives INSIDE the FormField render prop.
<FormField
  control={form.control}
  name="email"
  render={({ field }) => (
    <FormItem>
      <FormControl><Input {...field} /></FormControl>
      <FormMessage />                      // reads context.error.message
    </FormItem>
  )}
/>
```

A second cause of FormMessage staying empty : the zod schema validates, but the field path in the schema does not match the `name` prop on `FormField`. The validation runs against `email` but the input is named `userEmail`. See the next section.

### Resolver schema-vs-field-path match

`zodResolver(schema)` produces an `errors` object whose keys are the schema paths. `FormField name="X"` reads `errors.X`. A typo on either side causes the field to never display an error AND never block submission.

```tsx
// WRONG : schema uses "email", FormField uses "userEmail" ; validation runs but errors land in errors.email, FormField looks at errors.userEmail.
const schema = z.object({ email: z.email() })
<FormField name="userEmail" ... />        // never validated, never errors

// RIGHT : schema path and FormField name match exactly.
const schema = z.object({ email: z.email() })
<FormField name="email" ... />
```

The TypeScript inference via `z.infer<typeof schema>` only protects the `defaultValues` shape ; it does NOT type-check the `name` string against schema keys. The `name` prop accepts any string at runtime. Use exact-match identifiers ; a one-character typo is a silent un-validation.

### formState subscription model

`formState` is a Proxy. Reading a key (`formState.isDirty`) subscribes the component to re-render when that key changes. Keys that are NEVER read do NOT cause re-renders, which is why a hidden state appears stuck.

```tsx
// WRONG : isDirty is never read, so the component never re-renders when it flips.
const { formState } = useForm<FormData>()
useEffect(() => { console.log(formState.isDirty) }, [])   // logs once, value stale

// RIGHT : destructure to subscribe.
const { formState: { isDirty } } = useForm<FormData>()
// isDirty re-renders the component when it changes.
```

Subscribable keys : `isDirty`, `dirtyFields`, `touchedFields`, `isSubmitted`, `isSubmitting`, `isSubmitSuccessful`, `submitCount`, `isValid`, `isValidating`, `errors`. `isValid` ONLY updates when `mode` is `"onChange"`, `"onTouched"`, or `"onBlur"` (it stays `true` in the default `"onSubmit"` mode until first submit). For "submit button stays disabled until valid" UX, set `mode: "onChange"` on `useForm`.

### handleSubmit(onValid, onInvalid)

`handleSubmit` accepts TWO arguments. The second is called when validation fails. Omitting it means invalid submissions silently do nothing : the form stays put, errors render via FormMessage, but no toast, no scroll-to-error, no logging.

```tsx
// WRONG : invalid submits vanish silently. Hard to debug.
<form onSubmit={form.handleSubmit(onSubmit)}>

// RIGHT : second arg fires when zod rejects.
<form onSubmit={form.handleSubmit(onSubmit, (errors) => {
  toast.error("Please fix the highlighted fields")
  // scroll to first error, log to analytics, etc.
})}>
```

## Decision Tree : "My form does not behave"

1. Did the value reach the submit handler ? Open `onSubmit` and log the argument.
   - Field missing or empty : the input is not wired into form state. Go to step 2.
   - Field present and correct : validation may be the issue. Go to step 4.
2. Inspect the input wiring. Is it a native `<input>` or a shadcn / Radix component ?
   - Native input registered with `register("field")` : check the `name` matches the schema ; otherwise see step 4.
   - shadcn `Select` / `Checkbox` / `RadioGroup` / `Switch` wired with `register` : WRONG. Switch to `FormField` + `Controller`. See Pattern A in references/examples.md.
   - Wired with `FormField` but no value : check that `Form {...form}` spreads the form context above ; otherwise the `useFormContext` returns null.
3. Does the console show "A component is changing an uncontrolled input to be controlled" ?
   - YES : the field is missing from `defaultValues`. Add every field with an explicit initial value. See Pattern C.
   - NO : continue.
4. Does FormMessage render the error ? Click submit on an empty required field.
   - No FormMessage anywhere : check that `<FormMessage />` is inside a `<FormField render={({ field }) => ...}>` block.
   - FormMessage present but empty : the schema path does not match `FormField name`. Diff the strings character-by-character. See Pattern B.
5. Did validation run at all ? Read `form.formState.isValidating` and `form.formState.errors`.
   - `errors` is empty AND submit handler ran : validation passed (which may be wrong if the schema is incomplete).
   - `errors` is empty AND submit handler did NOT run : `handleSubmit` is wired but `onSubmit` was probably called without `handleSubmit` wrapping. See Pattern D.
   - `errors` is non-empty AND submit silently did nothing : you need the `onInvalid` second argument. See `handleSubmit(onValid, onInvalid)` above.
6. Is the form re-rendering visibly on every keystroke ? Mount a counter in a child.
   - YES : `form.watch()` is called in the component body. Replace with `useWatch({ control, name })` in a child, or use the callback form `watch((v) => ...)`.

## Companion Skills

- `shadcn-syntax-form` (Batch 3) : the seven Form primitives, canonical useForm + zodResolver setup, accessibility wiring. READ FIRST when designing a new form.
- `shadcn-impl-form-validation` (Batch 9) : end-to-end validation recipes, zod schema patterns, server-side validation handoff, async refine with debounce. READ for production form wiring.
- `shadcn-errors-radix-controlled` (this batch) : half-open controlled state on Radix primitives (Dialog, Sheet, Drawer, Popover) which can also surface as "form does not submit" when the dialog containing the form never closes.
- `shadcn-syntax-field` (Batch 3) : the newer Field family that decouples accessibility wiring from react-hook-form ; the path forward for TanStack Form users.

## Reference Files

- `references/methods.md` : the full `useForm` signature, the `Controller` render-prop API, the `formState` Proxy subscription rules, and the async zod refine pattern with AbortController.
- `references/examples.md` : WRONG-vs-RIGHT snippets for the six recurring traps, including the optimal `useWatch` placement, the explicit `defaultValues` shape, the async refine with debounce, and the `setError` server-error mapping pattern.
- `references/anti-patterns.md` : seven verified anti-patterns with root cause and deterministic fix : register on Select, inline watch, partial defaultValues, FormField outside Form, schema path mismatch, missing onInvalid, and file input with register.

## Verified Sources

- https://ui.shadcn.com/docs/forms/react-hook-form (canonical Form recipe, Field-based primitives, integration examples for Select / Checkbox / RadioGroup)
- https://ui.shadcn.com/docs/components/radix/form (Form layer summary, FormField wiring via Controller)
- https://react-hook-form.com/docs/useform (useForm options : mode, resolver, defaultValues, values, resetOptions ; formState subscription proxy)
- https://react-hook-form.com/docs/usecontroller/controller (Controller render-prop signature)
- https://react-hook-form.com/docs/useform/handlesubmit (onValid, onInvalid signature)
- https://react-hook-form.com/docs/useform/setvalue (setError, setValue, reset interaction)
- https://react-hook-form.com/docs/usewatch (useWatch vs watch subscription model)
- https://zod.dev (refine, superRefine, async refine semantics)

Last verified: 2026-05-19.
