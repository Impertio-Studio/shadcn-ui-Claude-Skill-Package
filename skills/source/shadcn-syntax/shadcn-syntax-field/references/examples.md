# shadcn ui : Field primitive examples

All examples verified against `https://ui.shadcn.com/docs/forms/react-hook-form`, `https://ui.shadcn.com/docs/forms/tanstack-form`, and `apps/v4/registry/new-york-v4/ui/field.tsx` (2026-05-19).

## Example 1 : Field + react-hook-form Controller (canonical 2026 path)

The canonical example from the official docs. Shows the recommended new-code pattern.

```tsx
"use client"

import * as React from "react"
import { zodResolver } from "@hookform/resolvers/zod"
import { Controller, useForm } from "react-hook-form"
import * as z from "zod"

import { Button } from "@/components/ui/button"
import {
  Field,
  FieldDescription,
  FieldError,
  FieldGroup,
  FieldLabel,
} from "@/components/ui/field"
import { Input } from "@/components/ui/input"
import { Textarea } from "@/components/ui/textarea"

const formSchema = z.object({
  title: z
    .string()
    .min(5, "Bug title must be at least 5 characters.")
    .max(32, "Bug title must be at most 32 characters."),
  description: z
    .string()
    .min(20, "Description must be at least 20 characters.")
    .max(100, "Description must be at most 100 characters."),
})

type FormValues = z.infer<typeof formSchema>

export function BugReportForm() {
  const form = useForm<FormValues>({
    resolver: zodResolver(formSchema),
    defaultValues: {
      title: "",
      description: "",
    },
  })

  function onSubmit(data: FormValues) {
    console.log(data)
  }

  return (
    <form onSubmit={form.handleSubmit(onSubmit)}>
      <FieldGroup>
        <Controller
          name="title"
          control={form.control}
          render={({ field, fieldState }) => (
            <Field data-invalid={fieldState.invalid}>
              <FieldLabel htmlFor={field.name}>Bug title</FieldLabel>
              <Input
                {...field}
                id={field.name}
                aria-invalid={fieldState.invalid}
                placeholder="Login button not working on mobile"
                autoComplete="off"
              />
              <FieldDescription>
                Provide a concise title for your bug report.
              </FieldDescription>
              {fieldState.invalid && (
                <FieldError errors={[fieldState.error]} />
              )}
            </Field>
          )}
        />

        <Controller
          name="description"
          control={form.control}
          render={({ field, fieldState }) => (
            <Field data-invalid={fieldState.invalid}>
              <FieldLabel htmlFor={field.name}>Description</FieldLabel>
              <Textarea
                {...field}
                id={field.name}
                aria-invalid={fieldState.invalid}
                placeholder="Reproduction steps, expected vs actual..."
                rows={5}
              />
              {fieldState.invalid && (
                <FieldError errors={[fieldState.error]} />
              )}
            </Field>
          )}
        />

        <Button type="submit" disabled={form.formState.isSubmitting}>
          Submit
        </Button>
      </FieldGroup>
    </form>
  )
}
```

Why this works :
- `Controller` provides `field` (value + onChange + name + ref) and `fieldState` (invalid + error + isTouched).
- `Field data-invalid` lights up destructive colour tokens via `data-[invalid=true]:text-destructive`.
- `Input id={field.name}` plus `FieldLabel htmlFor={field.name}` wires label-to-control focus.
- `aria-invalid` on the input announces the error to screen readers.
- `FieldError errors={[fieldState.error]}` renders the zod message and returns `null` when there is no error.

## Example 2 : Field + TanStack Form

The canonical example from the official docs for the alternative form library.

```tsx
"use client"

import * as React from "react"
import { useForm } from "@tanstack/react-form"
import { toast } from "sonner"
import * as z from "zod"

import { Button } from "@/components/ui/button"
import {
  Field,
  FieldDescription,
  FieldError,
  FieldGroup,
  FieldLabel,
} from "@/components/ui/field"
import { Input } from "@/components/ui/input"

const formSchema = z.object({
  title: z
    .string()
    .min(5, "Bug title must be at least 5 characters.")
    .max(32),
  description: z.string().min(20).max(100),
})

export function BugReportForm() {
  const form = useForm({
    defaultValues: {
      title: "",
      description: "",
    },
    validators: {
      onSubmit: formSchema,
    },
    onSubmit: async ({ value }) => {
      toast.success("Form submitted")
      console.log(value)
    },
  })

  return (
    <form
      onSubmit={(e) => {
        e.preventDefault()
        form.handleSubmit()
      }}
    >
      <FieldGroup>
        <form.Field
          name="title"
          children={(field) => {
            const isInvalid =
              field.state.meta.isTouched && !field.state.meta.isValid
            return (
              <Field data-invalid={isInvalid}>
                <FieldLabel htmlFor={field.name}>Bug title</FieldLabel>
                <Input
                  id={field.name}
                  name={field.name}
                  value={field.state.value}
                  onBlur={field.handleBlur}
                  onChange={(e) => field.handleChange(e.target.value)}
                  aria-invalid={isInvalid}
                  placeholder="Login button not working on mobile"
                  autoComplete="off"
                />
                <FieldDescription>
                  Provide a concise title for your bug report.
                </FieldDescription>
                {isInvalid && (
                  <FieldError errors={field.state.meta.errors} />
                )}
              </Field>
            )
          }}
        />

        <Button type="submit">Submit</Button>
      </FieldGroup>
    </form>
  )
}
```

Why this works :
- TanStack Form's `form.Field` provides `field.state.value`, `field.handleChange`, `field.handleBlur`, and `field.state.meta.errors` (Standard Schema array).
- `FieldError errors={field.state.meta.errors}` accepts the meta-errors array directly : the primitive de-dups by `message`.
- Invalid signal is derived as `isTouched && !isValid` to avoid showing errors before the user interacts.

## Example 3 : Field standalone (no form library)

For a one-field form (search bar, single setting toggle), no form library is needed.

```tsx
"use client"

import * as React from "react"

import {
  Field,
  FieldDescription,
  FieldError,
  FieldLabel,
} from "@/components/ui/field"
import { Input } from "@/components/ui/input"

export function SearchField() {
  const [value, setValue] = React.useState("")
  const [touched, setTouched] = React.useState(false)

  const error =
    touched && value.length > 0 && value.length < 3
      ? "Search query must be at least 3 characters."
      : undefined

  return (
    <Field data-invalid={!!error}>
      <FieldLabel htmlFor="search">Search</FieldLabel>
      <Input
        id="search"
        value={value}
        onChange={(e) => setValue(e.target.value)}
        onBlur={() => setTouched(true)}
        aria-invalid={!!error}
        placeholder="Type at least 3 characters..."
      />
      <FieldDescription>Searches title, description, and tags.</FieldDescription>
      {error && <FieldError errors={[{ message: error }]} />}
    </Field>
  )
}
```

Why this works :
- No `Controller`, no `form.Field` : plain `useState` drives `value`.
- The `FieldError` `errors` prop accepts any array of `{ message }` shapes, including ad-hoc ones.
- The `data-invalid` + `aria-invalid` pair is wired manually but mirrors the form-library pattern exactly.

## Example 4 : FieldSet + FieldLegend + FieldGroup composition

Grouped fields with a legend (e.g. an address block).

```tsx
"use client"

import { Controller, useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import * as z from "zod"

import {
  Field,
  FieldDescription,
  FieldError,
  FieldGroup,
  FieldLabel,
  FieldLegend,
  FieldSet,
} from "@/components/ui/field"
import { Input } from "@/components/ui/input"
import { Button } from "@/components/ui/button"

const formSchema = z.object({
  street: z.string().min(1, "Street required."),
  city: z.string().min(1, "City required."),
  postalCode: z.string().regex(/^\d{4}\s?[A-Z]{2}$/i, "Invalid NL postal code."),
})

type FormValues = z.infer<typeof formSchema>

export function AddressForm() {
  const form = useForm<FormValues>({
    resolver: zodResolver(formSchema),
    defaultValues: { street: "", city: "", postalCode: "" },
  })

  return (
    <form onSubmit={form.handleSubmit((v) => console.log(v))}>
      <FieldSet>
        <FieldLegend>Shipping address</FieldLegend>
        <FieldDescription>
          Where should we send your order?
        </FieldDescription>
        <FieldGroup>
          <Controller
            name="street"
            control={form.control}
            render={({ field, fieldState }) => (
              <Field data-invalid={fieldState.invalid}>
                <FieldLabel htmlFor={field.name}>Street</FieldLabel>
                <Input {...field} id={field.name} aria-invalid={fieldState.invalid} />
                {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
              </Field>
            )}
          />
          <Controller
            name="city"
            control={form.control}
            render={({ field, fieldState }) => (
              <Field data-invalid={fieldState.invalid}>
                <FieldLabel htmlFor={field.name}>City</FieldLabel>
                <Input {...field} id={field.name} aria-invalid={fieldState.invalid} />
                {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
              </Field>
            )}
          />
          <Controller
            name="postalCode"
            control={form.control}
            render={({ field, fieldState }) => (
              <Field data-invalid={fieldState.invalid} orientation="responsive">
                <FieldLabel htmlFor={field.name}>Postal code</FieldLabel>
                <Input {...field} id={field.name} aria-invalid={fieldState.invalid} />
                {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
              </Field>
            )}
          />
        </FieldGroup>
        <Button type="submit">Save address</Button>
      </FieldSet>
    </form>
  )
}
```

Why this works :
- `FieldSet` provides semantic grouping via the native `<fieldset>` element.
- `FieldLegend` (default `variant="legend"`) renders the group heading at `text-base`.
- `FieldGroup` establishes `@container/field-group`, enabling the `postalCode` field's `orientation="responsive"` to switch to horizontal at `@md`.

## Example 5 : Field with shadcn Select via Controller

A controlled non-native input (Radix Select) inside a Field.

```tsx
"use client"

import { Controller, useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import * as z from "zod"

import {
  Field,
  FieldDescription,
  FieldError,
  FieldLabel,
} from "@/components/ui/field"
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from "@/components/ui/select"
import { Button } from "@/components/ui/button"

const formSchema = z.object({
  priority: z.enum(["low", "medium", "high", "critical"], {
    message: "Pick a priority.",
  }),
})

type FormValues = z.infer<typeof formSchema>

export function PrioritySelectField() {
  const form = useForm<FormValues>({
    resolver: zodResolver(formSchema),
    defaultValues: { priority: "medium" },
  })

  return (
    <form onSubmit={form.handleSubmit((v) => console.log(v))}>
      <Controller
        name="priority"
        control={form.control}
        render={({ field, fieldState }) => (
          <Field data-invalid={fieldState.invalid}>
            <FieldLabel htmlFor={field.name}>Priority</FieldLabel>
            <Select value={field.value} onValueChange={field.onChange}>
              <SelectTrigger
                id={field.name}
                aria-invalid={fieldState.invalid}
              >
                <SelectValue placeholder="Pick a priority" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="low">Low</SelectItem>
                <SelectItem value="medium">Medium</SelectItem>
                <SelectItem value="high">High</SelectItem>
                <SelectItem value="critical">Critical</SelectItem>
              </SelectContent>
            </Select>
            <FieldDescription>
              Critical issues page the on-call engineer immediately.
            </FieldDescription>
            {fieldState.invalid && (
              <FieldError errors={[fieldState.error]} />
            )}
          </Field>
        )}
      />
      <Button type="submit">Save</Button>
    </form>
  )
}
```

Why this works :
- Radix Select is controlled-only (no native ref) ; `Controller` is the correct integration.
- `Select value={field.value} onValueChange={field.onChange}` binds the controlled pair.
- `SelectTrigger id={field.name}` matches `FieldLabel htmlFor` for click-to-focus.
- `aria-invalid` lives on the `SelectTrigger` (the focusable element).
