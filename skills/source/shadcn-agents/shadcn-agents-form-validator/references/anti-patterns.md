# Form Validator : Anti-Patterns

Six canonical silent-failure anti-patterns. Each has a WRONG snippet, a RIGHT snippet, the WHY, and the validator checkpoint that catches it.

## AP-1 : zod path typo, FormField name silently un-validates

### WRONG

```tsx
const formSchema = z.object({ email: z.string().email() })

const form = useForm({ resolver: zodResolver(formSchema), defaultValues: { email: "" } })

<FormField
  control={form.control}
  name="emai"          // typo : missing the final "l"
  render={({ field }) => (
    <FormItem>
      <FormLabel>Email</FormLabel>
      <FormControl><Input {...field} /></FormControl>
      <FormMessage />
    </FormItem>
  )}
/>
```

### RIGHT

```tsx
<FormField
  control={form.control}
  name="email"         // matches the schema path exactly
  render={({ field }) => (
    <FormItem>
      <FormLabel>Email</FormLabel>
      <FormControl><Input {...field} /></FormControl>
      <FormMessage />
    </FormItem>
  )}
/>
```

### WHY

The `name` prop on FormField is the key under which react-hook-form tracks the field's value, dirty-state, and error. The zod schema enforces validation on `email`. With `name="emai"`, RHF tracks `emai` and submits `{ emai: "...", email: undefined }`. zod sees `email: undefined`, fires "Required", but the FormMessage is on the FormField named `emai` (not `email`). The error never reaches the UI ; the user types an email, sees no error, hits submit, and the form submits invalid data. There is no console warning. There is no TypeScript error if the form is typed as `any` or if the zod schema is inferred only at the resolver.

ALWAYS spell the FormField `name` exactly as it appears in the zod schema. NEVER trust visual inspection : Levenshtein 1 typos (one extra letter, one missing letter, transposition) are invisible at a glance.

**Validator checkpoint** : [4] Name prop exact match.

## AP-2 : Missing FormMessage, errors fire silently

### WRONG

```tsx
<FormField
  control={form.control}
  name="email"
  render={({ field }) => (
    <FormItem>
      <FormLabel>Email</FormLabel>
      <FormControl><Input {...field} /></FormControl>
      {/* no FormMessage : zod errors are tracked in formState but never rendered */}
    </FormItem>
  )}
/>
```

### RIGHT

```tsx
<FormField
  control={form.control}
  name="email"
  render={({ field }) => (
    <FormItem>
      <FormLabel>Email</FormLabel>
      <FormControl><Input {...field} /></FormControl>
      <FormMessage />
    </FormItem>
  )}
/>
```

### WHY

FormMessage reads `formState.errors[name]?.message` and renders it. Without FormMessage, validation runs (zodResolver is wired, the error is in `formState.errors`), but the user sees no feedback. The form refuses to submit (handleSubmit blocks invalid forms), so the user clicks submit, nothing happens, no visible error : the form appears frozen. This is one of the top three "my form is broken" bug reports.

ALWAYS include exactly one `<FormMessage />` in every FormItem. NEVER assume the input's native validation popup covers it : zod messages do not surface to native popups, they surface to FormMessage.

**Validator checkpoint** : [3] FormMessage presence per field.

## AP-3 : register on a Radix Select, silent fail

### WRONG

```tsx
<FormField
  control={form.control}
  name="role"
  render={({ field }) => (
    <FormItem>
      <FormLabel>Role</FormLabel>
      <FormControl>
        <Select {...register("role")}>
          <SelectTrigger><SelectValue /></SelectTrigger>
          <SelectContent>
            <SelectItem value="admin">Admin</SelectItem>
            <SelectItem value="user">User</SelectItem>
          </SelectContent>
        </Select>
      </FormControl>
      <FormMessage />
    </FormItem>
  )}
/>
```

### RIGHT

```tsx
<FormField
  control={form.control}
  name="role"
  render={({ field }) => (
    <FormItem>
      <FormLabel>Role</FormLabel>
      <FormControl>
        <Select value={field.value} onValueChange={field.onChange}>
          <SelectTrigger><SelectValue /></SelectTrigger>
          <SelectContent>
            <SelectItem value="admin">Admin</SelectItem>
            <SelectItem value="user">User</SelectItem>
          </SelectContent>
        </Select>
      </FormControl>
      <FormMessage />
    </FormItem>
  )}
/>
```

### WHY

`register("role")` returns `{ name, onChange, onBlur, ref }` and assumes the bound control fires a native DOM `onChange`. Radix Select does NOT fire native `onChange` : it fires `onValueChange` with the new value as the argument. `register` never receives an update, so the form value for `role` stays at the defaultValue (or undefined). The Select visually changes (Radix manages its own internal state) but the form state does not. At submit, zod sees the stale or undefined value and fires "Required". The user has clearly picked a role on screen ; they see "Required" anyway ; the form looks haunted.

The same applies to Checkbox (`onCheckedChange`), Switch (`onCheckedChange`), RadioGroup (`onValueChange`), Combobox (custom `onChange`), and DatePicker / Calendar (`onSelect`). ALWAYS bind Radix controls through the `field` render-prop with the exact callback name the control fires. NEVER use `register` on any control that does not have a native HTML element underneath.

**Validator checkpoint** : [2] Controller-vs-register correctness.

## AP-4 : Partial defaultValues, uncontrolled-to-controlled warning

### WRONG

```tsx
const formSchema = z.object({
  username: z.string(),
  agree: z.boolean(),
  tags: z.array(z.string()),
})

useForm<z.infer<typeof formSchema>>({
  resolver: zodResolver(formSchema),
  defaultValues: {
    username: "",
    // missing : agree, tags
  },
})
```

### RIGHT

```tsx
useForm<z.infer<typeof formSchema>>({
  resolver: zodResolver(formSchema),
  defaultValues: {
    username: "",
    agree: false,
    tags: [],
  },
})
```

### WHY

When a FormField renders an input bound to `field.value` and `field.value` is `undefined`, React treats the input as uncontrolled. The first keystroke (or first click on a Checkbox) sets `field.value` to a defined value : React promotes the input to controlled. This triggers the famous warning : "A component is changing an uncontrolled input to be controlled. This is likely caused by the value changing from undefined to a defined value." Beyond the warning, the input loses its cursor position on the transition and the Checkbox renders `checked={undefined}` which collapses to false silently in some Radix versions.

For each zod type the safe initial value is documented in `methods.md` §5. ALWAYS specify every schema key in defaultValues. NEVER rely on "I will set it later via `setValue`" : the warning fires at first render, before any setValue can land.

**Validator checkpoint** : [5] defaultValues completeness.

## AP-5 : onSubmit not wrapped in handleSubmit

### WRONG

```tsx
const onSubmit = (values: any) => { /* server call */ }

return (
  <Form {...form}>
    <form onSubmit={onSubmit}>
      {/* ... FormField etc ... */}
    </form>
  </Form>
)
```

### RIGHT

```tsx
const onSubmit = (values: FormValues) => { /* server call */ }

return (
  <Form {...form}>
    <form onSubmit={form.handleSubmit(onSubmit)}>
      {/* ... FormField etc ... */}
    </form>
  </Form>
)
```

### WHY

`form.handleSubmit(onValid, onInvalid?)` is the gate that runs zodResolver before calling `onValid` with the validated and parsed values. Skipping handleSubmit bypasses validation entirely : `onSubmit` receives a raw `SyntheticEvent` (not validated values), the zod schema is never consulted, and the form submits whatever raw state RHF currently holds (including undefined fields, partial values, invalid email shapes, all of it). The user has a "valid-looking" form, the developer has a passing TypeScript compile (the onSubmit signature has `any`), and the server gets garbage. This is the most expensive of the silent failures because the data reaches the database.

The optional second argument `onInvalid` is called when the validation FAILS, with the formState errors. This is the documented hook for showing a top-level error banner (Alert) on submit-failure.

ALWAYS wrap : `onSubmit={form.handleSubmit(onSubmit)}`. NEVER pass `onSubmit` directly. NEVER pass `form.handleSubmit(onSubmit())` (immediate invocation : that calls onSubmit at render-time and passes the return value as the handler).

**Validator checkpoint** : [6] handleSubmit wrap.

## AP-6 : FormProvider nested, two competing contexts

### WRONG

```tsx
import { FormProvider } from "react-hook-form"

return (
  <Form {...form}>
    <FormProvider {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)}>
        <FormField .../>
      </form>
    </FormProvider>
  </Form>
)
```

OR

```tsx
return (
  <Form {...form}>
    <form onSubmit={form.handleSubmit(onSubmit)}>
      <Form {...form}>            {/* second <Form> nested */}
        <FormField .../>
      </Form>
    </form>
  </Form>
)
```

### RIGHT

```tsx
return (
  <Form {...form}>
    <form onSubmit={form.handleSubmit(onSubmit)}>
      <FormField .../>
    </form>
  </Form>
)
```

### WHY

The shadcn `<Form>` component IS a FormProvider : its implementation is `<FormProvider {...props}>{children}</FormProvider>` plus the shadcn FormField context. Nesting a second `<Form>` or wrapping with a manual `<FormProvider>` creates two react-hook-form contexts under the same subtree. Symptoms : FormField inside the inner provider reads from a different `formState`, errors may render in one context but not the other, the FormControl's `aria-describedby` chain points to ids from one context while the FormMessage reads from the other. The form "works" intermittently and the bug is the hardest to diagnose because the React DevTools shows two FormProvider entries that look identical.

ALWAYS use exactly one `<Form>` per logical form. NEVER import `FormProvider` from `react-hook-form` directly when you have a shadcn `<Form>` available : the shadcn `<Form>` already covers it. The only legitimate use of multi-Form is two SEPARATE forms on the same page, each with their own `useForm()` instance and their own `<Form>` wrapper, at sibling subtrees (not nested).

**Validator checkpoint** : [8] No nested FormProvider.

## Summary Table

| AP | Failure mode | Visible signal | Checkpoint |
|----|--------------|----------------|------------|
| AP-1 | name prop typo | Validation never fires for the field, but no error shown either | [4] Name prop exact match |
| AP-2 | FormMessage missing | Validation fires, form refuses submit, no visible error | [3] FormMessage presence |
| AP-3 | register on Radix control | Control visually changes, form value stays at default | [2] Controller-vs-register |
| AP-4 | Partial defaultValues | Uncontrolled-to-controlled React warning on first interaction | [5] defaultValues completeness |
| AP-5 | onSubmit not wrapped | Validation skipped, invalid data submits | [6] handleSubmit wrap |
| AP-6 | FormProvider nested | Intermittent error rendering, double aria-describedby | [8] No nested FormProvider |

ALWAYS run all six against any draft form, not just the one that "looks suspicious". Real broken forms ship 3+ anti-patterns simultaneously, and the user only reports the symptom of one. NEVER stop at the first match : continue through all eight checkpoints.
