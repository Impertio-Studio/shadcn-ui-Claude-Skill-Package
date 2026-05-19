# Examples : WRONG vs RIGHT for shadcn Form + react-hook-form

Every snippet is verified against `https://ui.shadcn.com/docs/forms/react-hook-form` and `https://react-hook-form.com/docs/useform` (last verified 2026-05-19). All snippets assume `"use client"` is at the file head and that the shadcn `Form`, `FormField`, `FormItem`, `FormLabel`, `FormControl`, `FormDescription`, `FormMessage` primitives are imported from `@/components/ui/form`.

## Example 1 : register on shadcn Select (silent fail) vs Controller via FormField

The single most common form bug : `register` listens for the native `change` event, but Radix Select fires `onValueChange`. The form submits with `flavor: undefined`.

```tsx
// WRONG : register listens for DOM change ; Radix Select fires onValueChange.
// The form submits with flavor: undefined ; no error, no warning.
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from "@/components/ui/select"

function FlavorForm() {
  const form = useForm<{ flavor: string }>({
    resolver: zodResolver(z.object({ flavor: z.string().min(1) })),
    defaultValues: { flavor: "" },
  })
  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit((data) => console.log(data))}>
        <Select {...form.register("flavor")}>          // BUG : Select ignores register's onChange
          <SelectTrigger><SelectValue placeholder="Pick one" /></SelectTrigger>
          <SelectContent>
            <SelectItem value="vanilla">Vanilla</SelectItem>
            <SelectItem value="chocolate">Chocolate</SelectItem>
          </SelectContent>
        </Select>
        <Button type="submit">Submit</Button>
      </form>
    </Form>
  )
}
```

```tsx
// RIGHT : FormField wraps Controller, which exposes field.onChange that maps to onValueChange.
function FlavorForm() {
  const form = useForm<{ flavor: string }>({
    resolver: zodResolver(z.object({ flavor: z.string().min(1) })),
    defaultValues: { flavor: "" },
  })
  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit((data) => console.log(data))}>
        <FormField
          control={form.control}
          name="flavor"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Flavor</FormLabel>
              <Select onValueChange={field.onChange} value={field.value}>
                <FormControl>
                  <SelectTrigger><SelectValue placeholder="Pick one" /></SelectTrigger>
                </FormControl>
                <SelectContent>
                  <SelectItem value="vanilla">Vanilla</SelectItem>
                  <SelectItem value="chocolate">Chocolate</SelectItem>
                </SelectContent>
              </Select>
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

The same pattern applies verbatim to `Checkbox` (use `onCheckedChange={field.onChange}` and `checked={field.value}`), `Switch` (same), `RadioGroup` (`onValueChange`, `value`), and `Slider` (`onValueChange`, `value`, value is `number[]`).

## Example 2 : form.watch in component body (full re-render) vs useWatch in isolated child

```tsx
// WRONG : every keystroke in any field re-renders the entire form.
// React DevTools will show every FormField re-rendering on every character typed.
function ProfileForm() {
  const form = useForm<{ name: string; email: string; bio: string }>({
    defaultValues: { name: "", email: "", bio: "" },
  })
  const name = form.watch("name")                     // SUBSCRIPTION at form root
  return (
    <Form {...form}>
      <form>
        <FormField control={form.control} name="name" render={({ field }) => (
          <FormItem><FormControl><Input {...field} /></FormControl><FormMessage /></FormItem>
        )} />
        <FormField control={form.control} name="email" render={({ field }) => (
          <FormItem><FormControl><Input {...field} /></FormControl><FormMessage /></FormItem>
        )} />
        <FormField control={form.control} name="bio" render={({ field }) => (
          <FormItem><FormControl><Textarea {...field} /></FormControl><FormMessage /></FormItem>
        )} />
        <p>Hello, {name}</p>                          // value source for the watch above
      </form>
    </Form>
  )
}
```

```tsx
// RIGHT : useWatch lives inside its own child component ; only that child re-renders.
function NameGreeting({ control }: { control: Control<{ name: string }> }) {
  const name = useWatch({ control, name: "name" })
  return <p>Hello, {name}</p>
}

function ProfileForm() {
  const form = useForm<{ name: string; email: string; bio: string }>({
    defaultValues: { name: "", email: "", bio: "" },
  })
  return (
    <Form {...form}>
      <form>
        {/* same FormFields */}
        <NameGreeting control={form.control} />       // only THIS component re-renders on "name" change
      </form>
    </Form>
  )
}
```

For autosave or analytics that do NOT render a value, use the callback form ; it does not trigger any re-render :

```tsx
useEffect(() => {
  const subscription = form.watch((values, { name, type }) => {
    if (type === "change") debouncedSave(values)
  })
  return () => subscription.unsubscribe()
}, [form])
```

## Example 3 : Partial defaultValues (controlled/uncontrolled warning) vs every field explicit

```tsx
// WRONG : email is undefined ; first keystroke flips controlled state and React warns.
// Some browsers silently lose the first character because the input was uncontrolled at type-time.
function SignupForm() {
  const form = useForm<{ name: string; email: string; age: number; subscribed: boolean }>({
    defaultValues: { name: "" },                       // email, age, subscribed are undefined
  })
  return ( /* fields ... */ )
}
```

Console output on first keystroke into the email field :

```
Warning : A component is changing an uncontrolled input to be controlled. This is likely caused by the value changing from undefined to a defined value, which should not happen.
```

```tsx
// RIGHT : every field has an explicit initial value matching its type.
function SignupForm() {
  const form = useForm<{ name: string; email: string; age: number; subscribed: boolean }>({
    defaultValues: {
      name: "",
      email: "",
      age: 0,                                          // or null if input handles null
      subscribed: false,
    },
  })
  return ( /* fields ... */ )
}
```

For arrays of dynamic field groups (used with `useFieldArray`), the initial value MUST be an array of fully-shaped objects, never `undefined` :

```tsx
useForm<{ tags: { value: string }[] }>({
  defaultValues: { tags: [{ value: "" }] },            // start with one empty tag
})
```

## Example 4 : Async zod refine with debounce-via-onBlur

```tsx
const schema = z.object({
  username: z
    .string()
    .min(3, "At least 3 characters")
    .refine(
      async (value) => {
        if (value.length < 3) return true              // skip until min length met
        const res = await fetch(`/api/check-username?u=${encodeURIComponent(value)}`)
        const { available } = await res.json()
        return available
      },
      { message: "Username already taken" }
    ),
})

type FormData = z.infer<typeof schema>

function UsernameForm() {
  const form = useForm<FormData>({
    resolver: zodResolver(schema),
    mode: "onBlur",                                    // run async refine only on blur
    defaultValues: { username: "" },
  })

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit((data) => console.log(data))}>
        <FormField
          control={form.control}
          name="username"
          render={({ field, fieldState }) => (
            <FormItem>
              <FormLabel>Username</FormLabel>
              <FormControl>
                <Input {...field} />
              </FormControl>
              {fieldState.isValidating && <FormDescription>Checking ...</FormDescription>}
              <FormMessage />
            </FormItem>
          )}
        />
        <Button type="submit" disabled={form.formState.isSubmitting || form.formState.isValidating}>
          Submit
        </Button>
      </form>
    </Form>
  )
}
```

For aggressive abort-on-keystroke behaviour, use `superRefine` with an `AbortController` ; see `methods.md` for the full pattern.

## Example 5 : setError for server-side validation handoff

```tsx
type FormData = z.infer<typeof schema>

function CreateAccountForm() {
  const form = useForm<FormData>({
    resolver: zodResolver(schema),
    defaultValues: { email: "", password: "" },
  })

  async function onSubmit(data: FormData) {
    const res = await fetch("/api/account", {
      method: "POST",
      body: JSON.stringify(data),
    })
    if (!res.ok) {
      const body = await res.json()
      // body.fieldErrors : { email: "Already registered", password: "Too common" }
      for (const [field, message] of Object.entries(body.fieldErrors ?? {})) {
        form.setError(field as keyof FormData, { type: "server", message: String(message) }, { shouldFocus: true })
      }
      // body.formError : a form-wide error not tied to one field
      if (body.formError) {
        form.setError("root.serverError", { type: "server", message: body.formError })
      }
      return
    }
    toast.success("Account created")
  }

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit, (errors) => {
        // onInvalid : fired when zod validation rejects.
        const first = Object.keys(errors)[0]
        toast.error(`Please fix : ${first}`)
      })}>
        {/* FormField for email + password ... */}
        {form.formState.errors.root?.serverError && (
          <p className="text-sm text-destructive">
            {form.formState.errors.root.serverError.message}
          </p>
        )}
        <Button type="submit">Create account</Button>
      </form>
    </Form>
  )
}
```

The literal `"root"` path is reserved for form-wide errors. Use `` `root.${string}` `` for multiple named root errors (e.g., `root.serverError`, `root.rateLimitError`). These render through `formState.errors.root.<name>` and survive `reset()` only when `keepErrors: true` is passed.

## Example 6 : File input with Controller

```tsx
// WRONG : register on file input loses the FileList ; field.value is a string.
function AvatarForm() {
  const form = useForm<{ avatar: FileList }>({ defaultValues: { avatar: undefined as any } })
  return (
    <input type="file" {...form.register("avatar")} />  // value is the empty string
  )
}
```

```tsx
// RIGHT : Controller exposes onChange ; pass e.target.files (the FileList).
const schema = z.object({
  avatar: z.instanceof(FileList).refine((files) => files.length === 1, "Pick one file"),
})

function AvatarForm() {
  const form = useForm<z.infer<typeof schema>>({
    resolver: zodResolver(schema),
    defaultValues: { avatar: undefined as unknown as FileList },
  })
  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit((data) => uploadAvatar(data.avatar[0]))}>
        <FormField
          control={form.control}
          name="avatar"
          render={({ field: { onChange, onBlur, name, ref } }) => (
            <FormItem>
              <FormLabel>Avatar</FormLabel>
              <FormControl>
                <Input
                  type="file"
                  name={name}
                  ref={ref}
                  onBlur={onBlur}
                  onChange={(e) => onChange(e.target.files)}    // pass FileList, not the string value
                />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        <Button type="submit">Upload</Button>
      </form>
    </Form>
  )
}
```

Note : `field.value` is NOT spread into the input for `type="file"` (the DOM blocks programmatic file-input values). Only `name`, `ref`, `onBlur`, and the controlled `onChange` are forwarded. The selected `FileList` lives in form state and reaches `onSubmit` via the resolver-validated `data.avatar`.
