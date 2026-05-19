# Anti-Patterns : shadcn Form + react-hook-form

Seven verified failure modes observed in shadcn projects and react-hook-form GitHub issues, each with root-cause analysis and the deterministic fix. All references verified at `https://ui.shadcn.com/docs/forms/react-hook-form` and `https://react-hook-form.com/docs/useform` (2026-05-19).

## AP-1 : register on a Radix-wrapped control

```tsx
// SYMPTOM : form submits with the field value as undefined (or empty string).
// No error, no warning. The Select looks like it works on screen.
<Select {...form.register("flavor")}>
  <SelectTrigger><SelectValue /></SelectTrigger>
  <SelectContent>...</SelectContent>
</Select>
```

**Root cause** : `register("flavor")` returns `{ name, ref, onChange, onBlur }` where `onChange` is a native DOM-change-event handler. Radix `Select` does NOT fire native `change`. It fires `onValueChange(value: string)`. The two prop names do not collide ; Radix simply ignores the spread `onChange` and registers's `onChange` never fires. Form state stays empty.

**Fix** : wrap with `FormField` + `Controller` and bind `onValueChange={field.onChange}` + `value={field.value}` explicitly. Same fix for `Checkbox` (`onCheckedChange`), `Switch` (`onCheckedChange`), `RadioGroup` (`onValueChange`), `Slider` (`onValueChange`), `Combobox`, and any third-party controlled component.

```tsx
<FormField
  control={form.control}
  name="flavor"
  render={({ field }) => (
    <FormItem>
      <Select onValueChange={field.onChange} value={field.value}>
        <FormControl><SelectTrigger><SelectValue /></SelectTrigger></FormControl>
        <SelectContent>...</SelectContent>
      </Select>
      <FormMessage />
    </FormItem>
  )}
/>
```

**Detection rule** : if a control's event prop is anything other than `onChange` (specifically : `onValueChange`, `onCheckedChange`, `onSelect`, `onDayClick`, `onPressedChange`), `register` will silently fail. Always use `FormField` for those.

## AP-2 : form.watch in the component body

```tsx
// SYMPTOM : the form feels sluggish on slow devices. React DevTools shows every
// FormField re-rendering on every keystroke in any field.
function MyForm() {
  const form = useForm<FormData>({ defaultValues })
  const values = form.watch()                          // subscribes the WHOLE component
  return <Form {...form}>...</Form>
}
```

**Root cause** : `form.watch()` (no args) subscribes the calling component to ALL field changes. The calling component is the form root, so every child re-renders. A 20-field form re-renders 20 FormFields on every keystroke.

**Fix (read for render)** : extract the consumer into a child and use `useWatch({ control, name })`. The hook subscribes only its own scope.

```tsx
function NameMirror({ control }: { control: Control<FormData> }) {
  const name = useWatch({ control, name: "name" })
  return <span>{name}</span>
}
```

**Fix (side effect, no render)** : use the callback form ; it does NOT subscribe to re-renders.

```tsx
useEffect(() => {
  const sub = form.watch((values, { name, type }) => {
    if (type === "change") autosave(values)
  })
  return () => sub.unsubscribe()
}, [form])
```

**Detection rule** : if a variable is assigned from `form.watch(...)` and lives in the form-root component body, the form will re-render on every keystroke. Move the watch into a child component, or convert to the callback form.

## AP-3 : Partial defaultValues (undefined fields)

```tsx
// SYMPTOM : console warning "A component is changing an uncontrolled input to be
// controlled" on first keystroke. Some users report losing the first character.
useForm<{ name: string; email: string }>({
  defaultValues: { name: "" },                         // email is undefined
})
```

**Root cause** : `defaultValues` is processed once at mount. A field NOT listed (or listed as `undefined`) starts as uncontrolled because `field.value` is `undefined`. The first keystroke writes a string, which makes `field.value` defined ; React detects the controlled-state flip and warns. The first character may also be lost depending on input type and React version.

**Fix** : every field that will become controlled MUST have an explicit initial value in `defaultValues` matching its TypeScript type. String fields : `""`. Boolean fields : `false`. Number fields : `0` or a sensible default. Array fields : `[]`. Object fields : the empty shape. NEVER use `undefined`.

```tsx
useForm<{ name: string; email: string; subscribed: boolean; tags: string[] }>({
  defaultValues: { name: "", email: "", subscribed: false, tags: [] },
})
```

For async-loaded initial data (edit-record flow), use `values` instead of `defaultValues`. `values` is reactive : every reference change resets the form. Combine with a loading state to render `null` until data arrives :

```tsx
const { data } = useQuery(...)
const form = useForm<FormData>({ values: data ?? { /* empty shape */ } })
if (!data) return <Skeleton />
```

## AP-4 : FormField rendered outside the Form provider

```tsx
// SYMPTOM : runtime error : "Cannot read properties of null (reading 'getFieldState')"
// OR : the field renders but submitting does nothing and FormMessage is empty.
function Wrapper() {
  return (
    <>
      <FormField name="email" .../>                    // no <Form> ancestor
    </>
  )
}
```

**Root cause** : `FormField` is a `Controller` wrapped in a context publisher. It needs `useFormContext()` to return the form's `control`. Without `<Form {...form}>` (which is `FormProvider` from react-hook-form) as an ancestor, the context returns `null` and the field cannot read or write state.

**Fix** : every `FormField` MUST live inside a `<Form {...form}>` ancestor that spreads the `useForm` return value. The HTML `<form>` element is separate and OPTIONAL ; the React `<Form>` component is what publishes the context.

```tsx
<Form {...form}>                                       {/* publishes context */}
  <form onSubmit={form.handleSubmit(onSubmit)}>        {/* HTML element */}
    <FormField name="email" .../>                      {/* reads context */}
  </form>
</Form>
```

**Detection rule** : if you see `useFormContext returned null` or any error from `getFieldState`, walk up the JSX tree until you find `<Form {...form}>`. If absent, add it.

## AP-5 : Zod schema path mismatch with FormField name

```tsx
// SYMPTOM : the field never shows an error. The form submits invalid data.
// No console warning, no TypeScript error.
const schema = z.object({
  emailAddress: z.email(),                             // schema path : "emailAddress"
})
<FormField name="email" .../>                           // FormField name : "email"
```

**Root cause** : zod validates the `emailAddress` key on the form data ; errors land in `formState.errors.emailAddress`. `FormField name="email"` reads `formState.errors.email`, which stays `undefined`. The field is never validated AND never reports an error. The form submits regardless of the schema.

`z.infer<typeof schema>` types the form data correctly (`{ emailAddress: string }`), but `FormField`'s `name` prop accepts any string at runtime, so a typo bypasses TypeScript.

**Fix** : use exact-string identifiers and consider extracting them as constants :

```tsx
const FIELDS = { email: "email", password: "password" } as const
const schema = z.object({
  [FIELDS.email]: z.email(),
  [FIELDS.password]: z.string().min(8),
})
<FormField name={FIELDS.email} .../>
```

Or define the schema first and use `keyof z.infer<typeof schema>` as the prop type for a typed wrapper :

```tsx
type FormData = z.infer<typeof schema>
type Field = keyof FormData
function TypedField({ name, ... }: { name: Field; ... }) { return <FormField name={name} ... /> }
```

**Detection rule** : if `formState.errors` is consistently empty after a known-invalid submission, log `Object.keys(form.formState.errors)` and compare against the rendered `FormField name` strings. Any mismatch is a typo.

## AP-6 : Missing onInvalid callback (silent submission failure)

```tsx
// SYMPTOM : clicking Submit on an invalid form appears to do nothing.
// FormMessage renders the errors but no toast fires, no analytics event, no scroll.
<form onSubmit={form.handleSubmit(onSubmit)}>
```

**Root cause** : `handleSubmit` accepts TWO arguments : `onValid` and `onInvalid`. When validation fails, `onValid` is NOT called. If `onInvalid` is omitted, no callback fires at all. The user gets only the per-field FormMessage feedback, which on a long form may be off-screen.

**Fix** : always pass an `onInvalid` handler that signals the failure (toast, scroll, log).

```tsx
<form
  onSubmit={form.handleSubmit(
    async (data) => {
      await api.submit(data)
    },
    (errors) => {
      // errors is FieldErrors<TFieldValues>
      const firstField = Object.keys(errors)[0]
      toast.error(`Please fix : ${firstField}`)
      // optional : scroll to first error
      const el = document.querySelector(`[name="${firstField}"]`)
      el?.scrollIntoView({ behavior: "smooth", block: "center" })
    }
  )}
>
```

**Detection rule** : if a user reports "clicking submit does nothing", check whether `handleSubmit` has a second argument. If not, add one with at least a toast.

## AP-7 : File input with register (lost FileList)

```tsx
// SYMPTOM : form.getValues("avatar") returns the empty string.
// onSubmit receives data.avatar as "" instead of a FileList.
<input type="file" {...form.register("avatar")} />
```

**Root cause** : `register` exposes a `value`/`onChange` pair that mirrors the DOM input's `value` property. For `type="file"`, the DOM `value` is the file path string (often `""` for security), NOT the `FileList`. The `FileList` lives on `e.target.files`, which `register`'s default `onChange` does NOT read.

**Fix** : use `Controller`/`FormField` and forward `e.target.files` to `field.onChange` :

```tsx
<FormField
  control={form.control}
  name="avatar"
  render={({ field: { onChange, onBlur, name, ref } }) => (
    <FormItem>
      <FormControl>
        <Input
          type="file"
          name={name}
          ref={ref}
          onBlur={onBlur}
          onChange={(e) => onChange(e.target.files)}   // pass FileList
        />
      </FormControl>
      <FormMessage />
    </FormItem>
  )}
/>
```

**Schema side** : the zod validator MUST match the value type (`FileList`, not `string`).

```tsx
const schema = z.object({
  avatar: z.instanceof(FileList).refine((f) => f.length === 1, "Pick one file"),
})
```

Note : `field.value` is NOT spread into the input for `type="file"` (DOM security blocks programmatic file-input values). Only `name`, `ref`, `onBlur`, and the controlled `onChange` are forwarded.

**Detection rule** : if a file input lives in a react-hook-form Form and `onSubmit` receives the field as a string (not a FileList), the input is using `register`. Convert to `FormField` + `Controller` + `e.target.files`.
