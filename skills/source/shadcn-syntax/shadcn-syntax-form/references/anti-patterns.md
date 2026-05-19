# shadcn Form : Anti-patterns

Seven failure modes that account for the majority of shadcn `Form` bugs reported in `react-hook-form` and `shadcn-ui/ui` issues. Each entry documents the symptom, the wrong pattern, WHY it fails, and the fix.

## 1. Using register on a non-native component

**Symptom** : the user picks a value in a `Select`, `Checkbox`, `RadioGroup`, `Switch`, or `Slider`, but the value never appears in the submit payload. No console warning. The field is silently absent.

**Wrong** :

```tsx
<Select {...form.register("language")}>
  <SelectTrigger>...</SelectTrigger>
  <SelectContent>...</SelectContent>
</Select>
```

**WHY it fails** : `form.register(name)` returns `{ name, onChange, onBlur, ref }`. The `ref` is forwarded to the DOM `<input>` element. Radix `Select` is NOT an `<input>` : it has no native `ref`. The `onChange` event from a native element never fires because the trigger does not emit `change` events. The Select communicates via `onValueChange`, which `register` never wires. The form-state slot for `language` therefore stays at its `defaultValue` forever.

**Fix** : use `FormField` (which uses `Controller` internally) and bind `value` + `onValueChange` explicitly :

```tsx
<FormField
  control={form.control}
  name="language"
  render={({ field }) => (
    <Select value={field.value} onValueChange={field.onChange} name={field.name}>
      <FormControl>
        <SelectTrigger>
          <SelectValue placeholder="Select" />
        </SelectTrigger>
      </FormControl>
      <SelectContent>...</SelectContent>
    </Select>
  )}
/>
```

The same rule applies to every Radix-wrapped shadcn control : `Checkbox` uses `checked` + `onCheckedChange`, `Switch` uses `checked` + `onCheckedChange`, `RadioGroup` uses `value` + `onValueChange`, `Slider` uses `value` + `onValueChange`, `ToggleGroup` uses `value` + `onValueChange`.

ALWAYS prefer the FormField + render-prop path for any non-native input. NEVER use `register` for a control that does not accept a native `ref`.

## 2. Forgetting FormMessage

**Symptom** : zod schema declares `z.string().min(2, "At least 2 chars.")`. The user submits with one character. The submit handler does not run (validation correctly fails), but no error message appears under the input. The user sees a frozen submit button and no feedback.

**Wrong** :

```tsx
<FormField
  control={form.control}
  name="username"
  render={({ field }) => (
    <FormItem>
      <FormLabel>Username</FormLabel>
      <FormControl>
        <Input {...field} />
      </FormControl>
      {/* FormMessage missing here */}
    </FormItem>
  )}
/>
```

**WHY it fails** : `FormMessage` is the ONLY primitive that renders `error?.message`. `FormLabel` colours its text red on error (via `data-[error=true]:text-destructive`) but does not render any error string. `FormControl` sets `aria-invalid="true"` on the input but emits no visible text. Omitting `FormMessage` therefore produces an invisible, unannounced error : screen-readers may pick up `aria-invalid` but sighted users see nothing.

**Fix** : ALWAYS render `<FormMessage />` exactly once per `FormItem`. Pass NO children : the component derives its body from form-state.

```tsx
<FormItem>
  <FormLabel>Username</FormLabel>
  <FormControl>
    <Input {...field} />
  </FormControl>
  <FormMessage />              {/* renders error.message automatically */}
</FormItem>
```

## 3. Defining the schema inside the component body

**Symptom** : forms feel sluggish on input. React DevTools shows the entire form tree re-rendering on every keystroke. Memoisation of child components does not help. `formState.isValidating` flickers true on each render.

**Wrong** :

```tsx
export function MyForm() {
  const formSchema = z.object({                        // re-created every render
    username: z.string().min(2),
  })
  const form = useForm({ resolver: zodResolver(formSchema) })  // resolver also re-created
  // ...
}
```

**WHY it fails** : `z.object({...})` returns a brand-new schema instance on every render. `zodResolver(schema)` then returns a brand-new resolver function. react-hook-form's internal `useEffect` watches the `resolver` reference and re-mounts validation listeners on every change. Every change-event therefore goes through a fresh listener pipeline, child components subscribed via `useFormContext` re-render, and any memoised `FormField` render-prop closure invalidates.

**Fix** : define the schema OUTSIDE the component (module scope) :

```tsx
const formSchema = z.object({                          // created ONCE
  username: z.string().min(2),
})
type Values = z.infer<typeof formSchema>

export function MyForm() {
  const form = useForm<Values>({
    resolver: zodResolver(formSchema),                 // stable reference
    defaultValues: { username: "" },
  })
  // ...
}
```

When the schema MUST depend on props or runtime values, wrap the construction in `React.useMemo` :

```tsx
export function MyForm({ minLength }: { minLength: number }) {
  const formSchema = React.useMemo(
    () => z.object({ username: z.string().min(minLength) }),
    [minLength]
  )
  const resolver = React.useMemo(() => zodResolver(formSchema), [formSchema])
  const form = useForm({ resolver, defaultValues: { username: "" } })
}
```

ALWAYS define the schema at module scope. NEVER inline `z.object()` inside the component body.

## 4. Setting defaultValues to undefined

**Symptom** : the React DevTools console shows "A component is changing an uncontrolled input to be controlled. Input elements should not switch from uncontrolled to controlled (or vice versa)." The warning fires on the first keystroke into a field.

**Wrong** :

```tsx
const form = useForm<Values>({
  resolver: zodResolver(schema),
  defaultValues: { email: "", username: undefined, age: undefined },
})
```

**WHY it fails** : react-hook-form treats `undefined` as "no value", which causes `field.value` to be `undefined` on first render. The underlying `<Input value={undefined}>` is uncontrolled. On the first user keystroke `field.onChange("a")` writes `"a"` into form-state, `field.value` becomes `"a"`, and the input switches to controlled. React detects the transition and warns. For boolean controls (Checkbox, Switch) the same transition triggers when the user clicks.

**Fix** : ALWAYS initialise EVERY field declared in the schema, even when the "real" initial value is empty. Match the type of the field :

| Field type | Initial value |
|------------|---------------|
| `z.string()` | `""` |
| `z.number()` | `0` or a sensible default (NEVER `undefined`) |
| `z.boolean()` | `false` |
| `z.array(...)` | `[]` |
| `z.object({...})` | `{}` with every nested field initialised |
| `z.date()` | `new Date()` or `null` paired with `z.date().nullable()` |
| `z.enum([...])` | `"" as ZodEnum` cast, or one of the enum members |

```tsx
defaultValues: { email: "", username: "", age: 0, agree: false, tags: [] }
```

When the initial value is unknown until data loads, ALWAYS use the `values` prop AND a sentinel `defaultValues` ; see [examples.md §7](examples.md).

## 5. Not wrapping form children in <Form {...form}>

**Symptom** : the page throws on first render with "useFormContext must be used within a FormProvider" or "Cannot read properties of undefined (reading 'control')". The form never mounts.

**Wrong** :

```tsx
const form = useForm<Values>({ ... })
return (
  <form onSubmit={form.handleSubmit(onSubmit)}>          // missing Form wrapper
    <FormField                                            // throws here
      control={form.control}
      name="email"
      render={({ field }) => <Input {...field} />}
    />
  </form>
)
```

**WHY it fails** : `FormField`'s render-prop subtree calls `useFormContext()` indirectly via `useFormField()` inside `FormLabel`, `FormControl`, `FormDescription`, and `FormMessage`. `useFormContext` returns `null` when no `FormProvider` ancestor exists. The first access on `null.getFieldState` (inside `useFormField`) throws.

**Fix** : ALWAYS wrap the entire form in `<Form {...form}>` and spread the entire `useForm` return value :

```tsx
<Form {...form}>
  <form onSubmit={form.handleSubmit(onSubmit)}>
    <FormField ... />
  </form>
</Form>
```

The `Form` component is a re-export of `FormProvider`. Spreading `{...form}` publishes `control`, `register`, `handleSubmit`, `formState`, and every other return value as context.

## 6. Calling handleSubmit without passing it as the form onSubmit prop

**Symptom** : clicking the submit button does nothing. No console error. No network request. `form.formState.submitCount` stays at 0. The browser does not even reload the page.

**Wrong** :

```tsx
<Button onClick={() => form.handleSubmit(onSubmit)}>Submit</Button>
```

OR

```tsx
<Button onClick={onSubmit}>Submit</Button>
```

**WHY it fails** : `form.handleSubmit(onSubmit)` RETURNS an event handler; it does NOT invoke `onSubmit`. The first wrong pattern creates a handler on every click but never calls it. The second wrong pattern bypasses validation entirely : `onSubmit` is called with no arguments (or with the click event), the resolver never runs, and the function probably crashes when it tries to read `values.email` from a `MouseEvent`.

**Fix** : pass the RETURN of `handleSubmit(onValid)` to the native `<form onSubmit={...}>` prop. Trigger the submit via a `<button type="submit">` inside that form :

```tsx
<form onSubmit={form.handleSubmit(onValid, onInvalid)}>
  ...
  <Button type="submit" disabled={form.formState.isSubmitting}>Submit</Button>
</form>
```

To submit programmatically (e.g. from a custom modal action), call `form.handleSubmit(onValid)()` :

```tsx
<Button onClick={() => form.handleSubmit(onValid)()}>Submit</Button>
```

Note the trailing `()` : the first call returns the handler, the second invokes it. ALWAYS prefer the `type="submit"` button path; the trailing-call pattern is for cases where the submit is decoupled from a `<form>` element.

## 7. Async refine without await on the consumer side (race condition)

**Symptom** : on a fast typist, an async refinement (e.g. "email already taken") shows the wrong result. The user changes the email, the previous fetch resolves AFTER the new one, and the stale `taken=true` overwrites the fresh `taken=false`. The submit button stays disabled even though the current value is valid.

**Wrong** :

```ts
z.string().email().refine(
  (email) => {
    fetch(`/api/check-email?email=${email}`)              // not awaited
      .then((res) => res.json())
      .then((data) => data.taken === false)               // never returned
    return true                                            // always passes locally
  },
  { message: "Email already taken." }
)
```

OR (a similar race) :

```ts
z.string().email().refine(
  async (email) => {
    const res = await fetch(`/api/check-email?email=${email}`)
    const { taken } = await res.json()
    return !taken                                          // no abort: stale fetch can win
  },
  { message: "Email already taken." }
)
```

**WHY it fails** :

The first variant returns synchronously (`return true`) and fires a fetch in the background whose result is never observed; the refinement always passes. The second variant correctly returns a Promise but does NOT abort the previous fetch when the user types again. If fetch #1 (for `"alice@"`) is slow and fetch #2 (for `"alice@x.com"`) is fast, `react-hook-form` resolves with the result of fetch #2, then fetch #1 resolves and overwrites the cached error. Subsequent renders surface the stale value.

**Fix** : await every async branch AND attach an `AbortController` (or a request-id token) to discard stale results. The simplest robust pattern caches results per input value and combines with `mode: "onBlur"` so each refine fires at most once per blur :

```ts
const cache = new Map<string, boolean>()
const inflight = new Map<string, Promise<boolean>>()

async function isEmailTaken(email: string): Promise<boolean> {
  if (cache.has(email)) return cache.get(email)!
  if (inflight.has(email)) return inflight.get(email)!
  const p = (async () => {
    const res = await fetch(`/api/check-email?email=${encodeURIComponent(email)}`)
    const { taken } = (await res.json()) as { taken: boolean }
    cache.set(email, taken)
    inflight.delete(email)
    return taken
  })()
  inflight.set(email, p)
  return p
}

export const schema = z.object({
  email: z.string().email().refine(
    async (email) => !(await isEmailTaken(email)),
    { message: "Email already taken." }
  ),
})
```

```tsx
const form = useForm<Values>({
  resolver: zodResolver(schema),
  mode: "onBlur",                          // fire async refine after blur, not on every keystroke
  defaultValues: { email: "" },
})
```

ALWAYS await every async branch inside `refine`. ALWAYS use `mode: "onBlur"` (or `"onTouched"`) for async refines so the network does not fire on every keystroke. ALWAYS cache or dedupe in-flight requests when the same value can be queried twice.
