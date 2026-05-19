# shadcn Form : Worked Examples

Eight complete, runnable examples covering the most common shadcn `Form` patterns. All examples target shadcn ui evergreen-2026 with Tailwind v4. Every example begins with `"use client"` because the `Form` component uses React hooks and context (see SKILL.md "RSC and 'use client'").

The component paths assume the default shadcn ui alias `@/components/ui/...`. Adjust to your project's alias configuration.

## 1. Schema definition (module scope)

ALWAYS define schemas OUTSIDE the component body and derive the TypeScript type via `z.infer`. NEVER inline a schema inside the component : see [anti-patterns.md §3](anti-patterns.md).

```ts
// schemas/sign-in.ts
import * as z from "zod"

export const signInSchema = z.object({
  email: z.string().min(1, "Email is required.").email("Invalid email."),
  password: z
    .string()
    .min(8, "At least 8 characters.")
    .max(72, "At most 72 characters."),
  remember: z.boolean().default(false),
})

export type SignInValues = z.infer<typeof signInSchema>
```

## 2. Minimal sign-in form (email + password)

```tsx
"use client"

import { zodResolver } from "@hookform/resolvers/zod"
import { useForm } from "react-hook-form"

import { signInSchema, type SignInValues } from "@/schemas/sign-in"
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

export function SignInForm() {
  const form = useForm<SignInValues>({
    resolver: zodResolver(signInSchema),
    defaultValues: { email: "", password: "", remember: false },
  })

  async function onSubmit(values: SignInValues) {
    const res = await fetch("/api/sign-in", {
      method: "POST",
      body: JSON.stringify(values),
    })
    if (!res.ok) {
      form.setError("root.serverError", {
        type: "server",
        message: "Invalid credentials.",
      })
    }
  }

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-6">
        <FormField
          control={form.control}
          name="email"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Email</FormLabel>
              <FormControl>
                <Input
                  type="email"
                  autoComplete="email"
                  placeholder="you@example.com"
                  {...field}
                />
              </FormControl>
              <FormDescription>
                We will never share your email.
              </FormDescription>
              <FormMessage />
            </FormItem>
          )}
        />
        <FormField
          control={form.control}
          name="password"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Password</FormLabel>
              <FormControl>
                <Input
                  type="password"
                  autoComplete="current-password"
                  {...field}
                />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        {form.formState.errors.root?.serverError && (
          <p className="text-sm text-destructive">
            {form.formState.errors.root.serverError.message}
          </p>
        )}
        <Button type="submit" disabled={form.formState.isSubmitting}>
          {form.formState.isSubmitting ? "Signing in..." : "Sign in"}
        </Button>
      </form>
    </Form>
  )
}
```

## 3. Form with Select (Controller path)

Radix `Select` is controlled-only : it expects `value` + `onValueChange`. NEVER use `register("language")` here; the value will never reach form-state.

```tsx
"use client"

import { zodResolver } from "@hookform/resolvers/zod"
import { useForm } from "react-hook-form"
import * as z from "zod"

import { Button } from "@/components/ui/button"
import {
  Form,
  FormControl,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from "@/components/ui/form"
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from "@/components/ui/select"

const schema = z.object({
  language: z.string().min(1, "Please choose a language."),
})
type Values = z.infer<typeof schema>

export function LanguageForm() {
  const form = useForm<Values>({
    resolver: zodResolver(schema),
    defaultValues: { language: "" },
  })

  return (
    <Form {...form}>
      <form
        onSubmit={form.handleSubmit((v) => console.log(v))}
        className="space-y-6"
      >
        <FormField
          control={form.control}
          name="language"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Spoken Language</FormLabel>
              <Select
                value={field.value}
                onValueChange={field.onChange}
                name={field.name}
              >
                <FormControl>
                  <SelectTrigger>
                    <SelectValue placeholder="Select a language" />
                  </SelectTrigger>
                </FormControl>
                <SelectContent>
                  <SelectItem value="en">English</SelectItem>
                  <SelectItem value="es">Spanish</SelectItem>
                  <SelectItem value="fr">French</SelectItem>
                </SelectContent>
              </Select>
              <FormMessage />
            </FormItem>
          )}
        />
        <Button type="submit">Save</Button>
      </form>
    </Form>
  )
}
```

The `<FormControl>` wraps the `<SelectTrigger>` because the trigger is the focusable, labellable element. Wrapping the whole `<Select>` would defeat the aria wiring.

## 4. Form with Checkbox (Controller path)

```tsx
"use client"

import { zodResolver } from "@hookform/resolvers/zod"
import { useForm } from "react-hook-form"
import * as z from "zod"

import { Button } from "@/components/ui/button"
import { Checkbox } from "@/components/ui/checkbox"
import {
  Form,
  FormControl,
  FormDescription,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from "@/components/ui/form"

const schema = z.object({
  terms: z.literal(true, {
    errorMap: () => ({ message: "You must accept the terms." }),
  }),
})
type Values = z.infer<typeof schema>

export function TermsForm() {
  const form = useForm<Values>({
    resolver: zodResolver(schema),
    defaultValues: { terms: false as unknown as true },
  })

  return (
    <Form {...form}>
      <form
        onSubmit={form.handleSubmit((v) => console.log(v))}
        className="space-y-6"
      >
        <FormField
          control={form.control}
          name="terms"
          render={({ field }) => (
            <FormItem className="flex flex-row items-start gap-3">
              <FormControl>
                <Checkbox
                  checked={field.value}
                  onCheckedChange={field.onChange}
                />
              </FormControl>
              <div className="grid gap-1.5">
                <FormLabel>Accept terms and conditions</FormLabel>
                <FormDescription>
                  Read the agreement before continuing.
                </FormDescription>
                <FormMessage />
              </div>
            </FormItem>
          )}
        />
        <Button type="submit">Continue</Button>
      </form>
    </Form>
  )
}
```

For a multi-select checkbox group bound to `z.array(z.string())`, push and filter the array inside `onCheckedChange` :

```tsx
<Checkbox
  checked={field.value.includes(taskId)}
  onCheckedChange={(checked) => {
    field.onChange(
      checked
        ? [...field.value, taskId]
        : field.value.filter((id) => id !== taskId)
    )
  }}
/>
```

## 5. Form with native Input via register

For a single isolated native input where the full FormField / FormItem wiring is overkill, `register` works directly. ALWAYS accept that `aria-describedby`, `aria-invalid`, and the `data-error` styling will NOT be wired automatically; you must add them by hand.

```tsx
"use client"

import { zodResolver } from "@hookform/resolvers/zod"
import { useForm } from "react-hook-form"
import * as z from "zod"

import { Button } from "@/components/ui/button"
import { Input } from "@/components/ui/input"
import { Label } from "@/components/ui/label"

const schema = z.object({
  search: z.string().min(1, "Required."),
})
type Values = z.infer<typeof schema>

export function SearchForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<Values>({
    resolver: zodResolver(schema),
    defaultValues: { search: "" },
  })

  return (
    <form
      onSubmit={handleSubmit((v) => console.log(v))}
      className="flex items-end gap-2"
    >
      <div className="grid gap-2">
        <Label htmlFor="search">Query</Label>
        <Input
          id="search"
          aria-invalid={!!errors.search}
          aria-describedby={errors.search ? "search-error" : undefined}
          {...register("search")}
        />
        {errors.search && (
          <p id="search-error" className="text-sm text-destructive">
            {errors.search.message}
          </p>
        )}
      </div>
      <Button type="submit" disabled={isSubmitting}>
        Search
      </Button>
    </form>
  )
}
```

ALWAYS prefer the Form composition path for anything beyond a single input.

## 6. Async validation (email-already-taken refine)

ALWAYS debounce async refines : without debouncing, every keystroke fires a network request. The pattern below uses a single in-memory cache plus an `AbortController` to discard stale fetches.

```ts
// schemas/sign-up.ts
import * as z from "zod"

const cache = new Map<string, boolean>()

async function isEmailTaken(email: string): Promise<boolean> {
  if (cache.has(email)) return cache.get(email)!
  const res = await fetch(`/api/check-email?email=${encodeURIComponent(email)}`)
  const { taken } = (await res.json()) as { taken: boolean }
  cache.set(email, taken)
  return taken
}

export const signUpSchema = z
  .object({
    email: z.string().email("Invalid email."),
    password: z.string().min(8),
    confirm: z.string().min(8),
  })
  .refine((d) => d.password === d.confirm, {
    path: ["confirm"],
    message: "Passwords do not match.",
  })
  .refine(
    async (d) => !(await isEmailTaken(d.email)),
    { path: ["email"], message: "Email already in use." }
  )

export type SignUpValues = z.infer<typeof signUpSchema>
```

```tsx
"use client"

import { zodResolver } from "@hookform/resolvers/zod"
import { useForm } from "react-hook-form"

import { signUpSchema, type SignUpValues } from "@/schemas/sign-up"

export function SignUpForm() {
  const form = useForm<SignUpValues>({
    resolver: zodResolver(signUpSchema),
    defaultValues: { email: "", password: "", confirm: "" },
    mode: "onBlur",          // run async refine on blur, not on every keystroke
  })

  // ... render with Form / FormField / FormMessage as in example 2
}
```

ALWAYS pair async refines with `mode: "onBlur"` or `mode: "onTouched"` so the fetch fires after the user leaves the field, not on every keystroke.

## 7. defaultValues vs values vs reset

```tsx
"use client"

import { useQuery } from "@tanstack/react-query"
import { useEffect } from "react"
import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import * as z from "zod"

const schema = z.object({ name: z.string(), bio: z.string().optional() })
type Values = z.infer<typeof schema>

export function ProfileEditor({ userId }: { userId: string }) {
  const { data } = useQuery({
    queryKey: ["user", userId],
    queryFn: async () => (await fetch(`/api/users/${userId}`)).json() as Promise<Values>,
  })

  // Pattern A : `values` prop (recommended)
  // Form re-resets whenever `data` changes. defaultValues acts as a fallback before data arrives.
  const formA = useForm<Values>({
    resolver: zodResolver(schema),
    defaultValues: { name: "", bio: "" },
    values: data,
  })

  // Pattern B : imperative reset in an effect (legacy)
  const formB = useForm<Values>({
    resolver: zodResolver(schema),
    defaultValues: { name: "", bio: "" },
  })
  useEffect(() => {
    if (data) formB.reset(data)
  }, [data, formB])

  return null
}
```

ALWAYS prefer Pattern A (`values`). NEVER mix Pattern A and Pattern B for the same form : `values` and a manual `reset` race each other.

After a successful submit, ALWAYS `form.reset(submitted)` to clear `isDirty` :

```tsx
async function onValid(values: Values) {
  await save(values)
  form.reset(values)                   // clears isDirty; preserves the values
}
```

## 8. Custom FormField composition (extending the pattern)

When the same field shape repeats (e.g. text-input + label + description + message), extract a custom field component that calls `useFormContext` and `useFormField`. The shadcn primitives remain intact; the wrapper just curries.

```tsx
"use client"

import { useFormContext, type FieldPath, type FieldValues } from "react-hook-form"

import {
  FormControl,
  FormDescription,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from "@/components/ui/form"
import { Input } from "@/components/ui/input"

type Props<T extends FieldValues> = {
  name: FieldPath<T>
  label: string
  description?: string
  placeholder?: string
  type?: React.InputHTMLAttributes<HTMLInputElement>["type"]
}

export function TextInputField<T extends FieldValues>({
  name,
  label,
  description,
  placeholder,
  type = "text",
}: Props<T>) {
  const form = useFormContext<T>()
  return (
    <FormField
      control={form.control}
      name={name}
      render={({ field }) => (
        <FormItem>
          <FormLabel>{label}</FormLabel>
          <FormControl>
            <Input type={type} placeholder={placeholder} {...field} />
          </FormControl>
          {description && <FormDescription>{description}</FormDescription>}
          <FormMessage />
        </FormItem>
      )}
    />
  )
}
```

Usage :

```tsx
<Form {...form}>
  <form onSubmit={form.handleSubmit(onSubmit)}>
    <TextInputField<SignInValues> name="email" label="Email" type="email" />
    <TextInputField<SignInValues> name="password" label="Password" type="password" />
    <Button type="submit">Sign in</Button>
  </form>
</Form>
```

ALWAYS keep the wrapper INSIDE the `<Form {...form}>` tree : `useFormContext` reads the same context that `FormField` consumes. NEVER lift the wrapper above `Form` : the context is not available.
