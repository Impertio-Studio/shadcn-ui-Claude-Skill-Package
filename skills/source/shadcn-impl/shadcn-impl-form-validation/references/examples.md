# shadcn-impl-form-validation: examples

Seven complete, copy-pasteable examples. Each one targets a specific end-to-end pattern.

## 1. Minimal sign-in form

The smallest working shape: email + password, zod resolver, typed submit, loading state on the button.

```tsx
"use client"

import { zodResolver } from "@hookform/resolvers/zod"
import { useForm } from "react-hook-form"
import * as z from "zod"

import { Button } from "@/components/ui/button"
import {
  Form, FormControl, FormField, FormItem, FormLabel, FormMessage,
} from "@/components/ui/form"
import { Input } from "@/components/ui/input"

const signInSchema = z.object({
  email: z.string().min(1, "Required.").email("Invalid email."),
  password: z.string().min(8, "Min 8 characters."),
})

type SignInValues = z.infer<typeof signInSchema>

export function SignInForm() {
  const form = useForm<SignInValues>({
    resolver: zodResolver(signInSchema),
    defaultValues: { email: "", password: "" },
  })

  async function onSubmit(values: SignInValues) {
    await new Promise((r) => setTimeout(r, 800))
    console.log(values)
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

## 2. Sign-up with confirm-password (.refine cross-field)

Cross-field validation attaches the error to a specific field via `path`. Without `path`, the error goes to the form root and no `<FormMessage />` renders it.

```tsx
const signUpSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8, "Min 8 characters."),
  confirmPassword: z.string().min(1, "Required."),
}).refine((data) => data.password === data.confirmPassword, {
  message: "Passwords do not match.",
  path: ["confirmPassword"],
})

type SignUpValues = z.infer<typeof signUpSchema>

export function SignUpForm() {
  const form = useForm<SignUpValues>({
    resolver: zodResolver(signUpSchema),
    defaultValues: { email: "", password: "", confirmPassword: "" },
    mode: "onTouched",
  })

  async function onSubmit(values: SignUpValues) {
    const res = await fetch("/api/sign-up", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(values),
    })
    if (!res.ok) {
      const data = await res.json()
      for (const [k, v] of Object.entries<string>(data.fieldErrors ?? {})) {
        form.setError(k as keyof SignUpValues, { type: "server", message: v })
      }
      return
    }
    form.reset()
  }

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-6">
        <FormField control={form.control} name="email" render={({ field }) => (
          <FormItem>
            <FormLabel>Email</FormLabel>
            <FormControl><Input type="email" {...field} /></FormControl>
            <FormMessage />
          </FormItem>
        )} />
        <FormField control={form.control} name="password" render={({ field }) => (
          <FormItem>
            <FormLabel>Password</FormLabel>
            <FormControl><Input type="password" autoComplete="new-password" {...field} /></FormControl>
            <FormMessage />
          </FormItem>
        )} />
        <FormField control={form.control} name="confirmPassword" render={({ field }) => (
          <FormItem>
            <FormLabel>Confirm password</FormLabel>
            <FormControl><Input type="password" autoComplete="new-password" {...field} /></FormControl>
            <FormMessage />
          </FormItem>
        )} />
        <Button type="submit" disabled={form.formState.isSubmitting}>Create account</Button>
      </form>
    </Form>
  )
}
```

## 3. Async email-availability with AbortController + debounce

The naive `.refine(async ...)` fires a request per keystroke and stale responses can flicker the error state. Wrap the network call so each new validation aborts the prior one.

```tsx
// utils/check-email.ts
let inflight: AbortController | null = null

export async function checkEmailAvailable(email: string): Promise<boolean> {
  inflight?.abort()
  const ac = new AbortController()
  inflight = ac
  try {
    const res = await fetch(`/api/check-email?email=${encodeURIComponent(email)}`, { signal: ac.signal })
    if (!res.ok) return true                 // fail-open: do not block on transient errors
    const { taken } = await res.json()
    return !taken
  } catch (e) {
    if ((e as Error).name === "AbortError") return true
    return true
  } finally {
    if (inflight === ac) inflight = null
  }
}
```

```tsx
const schema = z.object({
  email: z.string().email().refine(checkEmailAvailable, { message: "Email already in use." }),
})

type Values = z.infer<typeof schema>

export function EmailAvailability() {
  const form = useForm<Values>({
    resolver: zodResolver(schema),
    defaultValues: { email: "" },
    mode: "onBlur",
    reValidateMode: "onBlur",
    delayError: 300,
  })

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(async (v) => console.log(v))}>
        <FormField control={form.control} name="email" render={({ field }) => (
          <FormItem>
            <FormLabel>Email</FormLabel>
            <FormControl><Input type="email" {...field} /></FormControl>
            <FormDescription>
              {form.formState.isValidating ? "Checking..." : "We will not share your email."}
            </FormDescription>
            <FormMessage />
          </FormItem>
        )} />
        <Button type="submit" disabled={form.formState.isSubmitting || form.formState.isValidating}>
          Continue
        </Button>
      </form>
    </Form>
  )
}
```

`mode: "onBlur"` + `delayError: 300` keeps the async check off the hot keystroke path; the request fires once when the field loses focus.

## 4. Server-error setError after API call

The server is the source of truth for "username already taken", "rate limit exceeded", "invalid coupon". Map the response back to specific fields via `setError`.

```tsx
type ApiError = { fieldErrors?: Record<string, string>; formError?: string }

async function onSubmit(values: SignUpValues) {
  form.clearErrors("root.serverError")           // wipe any prior form-level error
  try {
    const res = await fetch("/api/sign-up", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(values),
    })
    if (!res.ok) {
      const data: ApiError = await res.json()
      if (data.formError) {
        form.setError("root.serverError", { type: "server", message: data.formError })
      }
      for (const [name, msg] of Object.entries(data.fieldErrors ?? {})) {
        form.setError(name as keyof SignUpValues, { type: "server", message: msg })
      }
      return
    }
    form.reset()
    toast.success("Account created.")
  } catch (e) {
    form.setError("root.serverError", { type: "network", message: "Network error. Try again." })
  }
}

// Reading the form-level error in the UI:
{form.formState.errors.root?.serverError && (
  <p role="alert" className="text-sm text-destructive">
    {form.formState.errors.root.serverError.message}
  </p>
)}
```

## 5. File upload via Controller + FileList

File inputs MUST go through Controller. The destructured `value` is dropped (file inputs have read-only `value`); `onChange` forwards `e.target.files` (FileList, not File).

```tsx
const MAX_BYTES = 5 * 1024 * 1024
const ACCEPTED = ["application/pdf"]

const resumeSchema = z.object({
  fullName: z.string().min(1),
  resume: z.instanceof(FileList)
    .refine((f) => f.length === 1, "Upload one file.")
    .refine((f) => f[0]?.size <= MAX_BYTES, "Max 5 MB.")
    .refine((f) => ACCEPTED.includes(f[0]?.type), "PDF only."),
})

type ResumeValues = z.infer<typeof resumeSchema>

export function ResumeForm() {
  const form = useForm<ResumeValues>({
    resolver: zodResolver(resumeSchema),
    defaultValues: { fullName: "", resume: undefined as unknown as FileList },
  })

  async function onSubmit(values: ResumeValues) {
    const fd = new FormData()
    fd.append("fullName", values.fullName)
    fd.append("resume", values.resume[0])
    const res = await fetch("/api/resume", { method: "POST", body: fd })
    if (res.ok) form.reset()
  }

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-6">
        <FormField control={form.control} name="fullName" render={({ field }) => (
          <FormItem>
            <FormLabel>Full name</FormLabel>
            <FormControl><Input {...field} /></FormControl>
            <FormMessage />
          </FormItem>
        )} />

        <FormField control={form.control} name="resume" render={({ field: { onChange, value: _ignored, ...rest } }) => (
          <FormItem>
            <FormLabel>Resume (PDF, max 5 MB)</FormLabel>
            <FormControl>
              <Input
                type="file"
                accept="application/pdf"
                {...rest}
                onChange={(e) => onChange(e.target.files)}
              />
            </FormControl>
            <FormMessage />
          </FormItem>
        )} />

        <Button type="submit" disabled={form.formState.isSubmitting}>Upload</Button>
      </form>
    </Form>
  )
}
```

The `defaultValues.resume` is intentionally `undefined as unknown as FileList`: a file input cannot be initialised with a value. The cast satisfies the inferred type without triggering the controlled/uncontrolled warning, because the underlying DOM input never receives a `value` prop.

## 6. Multi-step wizard via FormProvider + useFormContext

A single `useForm` instance hosts every step. Steps render conditionally; form state is preserved across renders because `FormProvider` is the stable ancestor.

```tsx
"use client"

import { useState } from "react"
import { zodResolver } from "@hookform/resolvers/zod"
import { FormProvider, useForm, useFormContext } from "react-hook-form"
import * as z from "zod"

import { Button } from "@/components/ui/button"
import { Form, FormControl, FormField, FormItem, FormLabel, FormMessage } from "@/components/ui/form"
import { Input } from "@/components/ui/input"

const wizardSchema = z.object({
  step1: z.object({ name: z.string().min(1), email: z.string().email() }),
  step2: z.object({ address: z.string().min(1), city: z.string().min(1) }),
  step3: z.object({ agree: z.literal(true, { errorMap: () => ({ message: "You must agree." }) }) }),
})

type WizardValues = z.infer<typeof wizardSchema>

export function Wizard() {
  const [step, setStep] = useState<0 | 1 | 2>(0)
  const form = useForm<WizardValues>({
    resolver: zodResolver(wizardSchema),
    defaultValues: {
      step1: { name: "", email: "" },
      step2: { address: "", city: "" },
      step3: { agree: false as unknown as true },
    },
    mode: "onChange",
  })

  async function next() {
    const key = (`step${step + 1}` as const) as keyof WizardValues
    const ok = await form.trigger(key)
    if (ok) setStep((s) => Math.min(s + 1, 2) as 0 | 1 | 2)
  }

  async function onSubmit(values: WizardValues) {
    console.log("submit", values)
    form.reset()
    setStep(0)
  }

  const StepComponent = [Step1, Step2, Step3][step]

  return (
    <FormProvider {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-6">
        <StepComponent />
        <div className="flex gap-2">
          {step > 0 && <Button type="button" variant="outline" onClick={() => setStep((s) => Math.max(0, s - 1) as 0 | 1 | 2)}>Back</Button>}
          {step < 2 ? (
            <Button type="button" onClick={next}>Next</Button>
          ) : (
            <Button type="submit" disabled={form.formState.isSubmitting}>Submit</Button>
          )}
        </div>
      </form>
    </FormProvider>
  )
}

function Step1() {
  const form = useFormContext<WizardValues>()
  return (
    <div className="space-y-4">
      <FormField control={form.control} name="step1.name" render={({ field }) => (
        <FormItem>
          <FormLabel>Name</FormLabel>
          <FormControl><Input {...field} /></FormControl>
          <FormMessage />
        </FormItem>
      )} />
      <FormField control={form.control} name="step1.email" render={({ field }) => (
        <FormItem>
          <FormLabel>Email</FormLabel>
          <FormControl><Input type="email" {...field} /></FormControl>
          <FormMessage />
        </FormItem>
      )} />
    </div>
  )
}

function Step2() {
  const form = useFormContext<WizardValues>()
  return (
    <div className="space-y-4">
      <FormField control={form.control} name="step2.address" render={({ field }) => (
        <FormItem>
          <FormLabel>Address</FormLabel>
          <FormControl><Input {...field} /></FormControl>
          <FormMessage />
        </FormItem>
      )} />
      <FormField control={form.control} name="step2.city" render={({ field }) => (
        <FormItem>
          <FormLabel>City</FormLabel>
          <FormControl><Input {...field} /></FormControl>
          <FormMessage />
        </FormItem>
      )} />
    </div>
  )
}

function Step3() {
  const form = useFormContext<WizardValues>()
  return (
    <FormField control={form.control} name="step3.agree" render={({ field }) => (
      <FormItem className="flex items-center gap-2">
        <FormControl>
          <input type="checkbox" checked={field.value as unknown as boolean} onChange={(e) => field.onChange(e.target.checked)} />
        </FormControl>
        <FormLabel>I agree to the terms.</FormLabel>
        <FormMessage />
      </FormItem>
    )} />
  )
}
```

`form.trigger("step1")` validates ONLY the current step's group of fields. Without it, `next()` would advance even when the current step is invalid.

## 7. Loading state on submit button

The canonical loading-button shape combines `isSubmitting`, a `<Loader2>` spinner, and a label swap.

```tsx
import { Loader2 } from "lucide-react"
import { Button } from "@/components/ui/button"

<Button type="submit" disabled={form.formState.isSubmitting}>
  {form.formState.isSubmitting ? (
    <>
      <Loader2 className="size-4 animate-spin" aria-hidden="true" />
      Saving...
    </>
  ) : (
    "Save"
  )}
</Button>
```

For forms that combine async validation with submit, also gate on `isValidating`:

```tsx
<Button
  type="submit"
  disabled={form.formState.isSubmitting || form.formState.isValidating}
  aria-busy={form.formState.isSubmitting}
>
  {form.formState.isSubmitting && <Loader2 className="size-4 animate-spin" />}
  {form.formState.isSubmitting ? "Saving..." : "Save"}
</Button>
```

NEVER gate the button on `!form.formState.isValid` while async validation is in flight: the button visibly flickers as `isValid` toggles between re-validations.

## Sources

- https://ui.shadcn.com/docs/components/radix/form
- https://ui.shadcn.com/docs/forms/react-hook-form
- https://react-hook-form.com/docs/useform
- https://react-hook-form.com/docs/useformcontext
- https://react-hook-form.com/docs/useform/seterror
- https://react-hook-form.com/docs/useform/trigger
- https://zod.dev/?id=refine

Verified 2026-05-19.
