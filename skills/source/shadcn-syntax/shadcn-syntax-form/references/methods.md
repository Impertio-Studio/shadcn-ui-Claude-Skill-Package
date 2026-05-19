# shadcn Form : Method and API Reference

Complete API surface for the seven shadcn Form primitives, the supporting `useFormField` hook, the react-hook-form integration (`useForm`, `Controller`), the `zodResolver` factory, and the zod schema patterns most relevant to forms. Every signature is verified against the canonical source at `apps/v4/registry/new-york-v4/ui/form.tsx` in the shadcn-ui/ui repository and against the official react-hook-form and zod documentation.

## 1. Form

```tsx
const Form = FormProvider
```

`Form` is a direct re-export of `FormProvider` from react-hook-form. It takes the entire `useForm` return value spread as props and publishes the form context to all descendants.

**Props** : every property returned by `useForm`. The typical invocation is `<Form {...form}>`.

**Required** : ALWAYS spread the `useForm` return value. The descendants (`FormField`, every `useFormContext` consumer) throw when the spread is omitted.

## 2. FormField

```tsx
const FormField = <
  TFieldValues extends FieldValues = FieldValues,
  TName extends FieldPath<TFieldValues> = FieldPath<TFieldValues>,
>(
  props: ControllerProps<TFieldValues, TName>
) => ReactElement
```

`FormField` wraps react-hook-form's `Controller` in a `FormFieldContext.Provider` that publishes the field `name`. Every prop accepted by `Controller` is accepted here.

**Props** :

| Name | Type | Required | Notes |
|------|------|----------|-------|
| `name` | `FieldPath<TFieldValues>` | yes | A path-style key into the schema (e.g. `"email"`, `"address.street"`, `"items.0.name"`). Type-checked against the `useForm` generic. |
| `control` | `Control<TFieldValues>` | yes | `form.control` from `useForm`. |
| `render` | `({ field, fieldState, formState }) => ReactElement` | yes | Render-prop : returns the form item tree. |
| `defaultValue` | field value | no | Overrides the `useForm` defaultValues for this field. Prefer `useForm({ defaultValues })`. |
| `rules` | `RegisterOptions` | no | Inline rh-f rules; redundant when using a resolver. ALWAYS prefer the resolver schema. |
| `shouldUnregister` | `boolean` | no | When `true`, unmounting the field removes its value from form-state. Default `false`. |
| `disabled` | `boolean` | no | When `true`, sets `field.disabled` and removes the field from submission output. |

**render prop signature** :

```ts
({
  field: {
    name: TName
    value: FieldValue
    onChange: (value: FieldValue) => void
    onBlur: () => void
    ref: Ref<unknown>
    disabled?: boolean
  }
  fieldState: {
    invalid: boolean
    isTouched: boolean
    isDirty: boolean
    isValidating: boolean
    error?: FieldError
  }
  formState: FormState<TFieldValues>
}) => ReactElement
```

ALWAYS spread `{...field}` onto native inputs. ALWAYS extract `value` and `onChange` explicitly for Radix-wrapped controls (they expect `onValueChange` or `onCheckedChange`, not `onChange`).

## 3. FormItem

```tsx
function FormItem(props: React.ComponentProps<"div">): JSX.Element
```

`FormItem` renders a `<div data-slot="form-item" className="grid gap-2">` and publishes a unique `id` (from `React.useId()`) via `FormItemContext`. The id is derived into three downstream ids :

- `formItemId` = `${id}-form-item` (used by `FormControl`'s `id` and `FormLabel`'s `htmlFor`)
- `formDescriptionId` = `${id}-form-item-description` (used by `FormDescription`'s `id` and the control's `aria-describedby`)
- `formMessageId` = `${id}-form-item-message` (used by `FormMessage`'s `id` and the control's `aria-describedby` when in error state)

**Props** : every prop accepted by a native `<div>`. ALWAYS pass exactly one `FormItem` per `FormField`.

## 4. FormLabel

```tsx
function FormLabel(
  props: React.ComponentProps<typeof LabelPrimitive.Root>
): JSX.Element
```

Wraps shadcn `Label`. Internally calls `useFormField()` to read `formItemId` and `error`. Sets `htmlFor={formItemId}` and `data-error={!!error}`. The base class string `"data-[error=true]:text-destructive"` colours the label red on error.

**Props** : every prop accepted by Radix `Label` (className, children, etc.). ALWAYS render exactly one `FormLabel` per `FormItem`.

## 5. FormControl

```tsx
function FormControl(
  props: React.ComponentProps<typeof Slot.Root>
): JSX.Element
```

Wraps Radix `Slot.Root`. Calls `useFormField()` and forwards :

- `id={formItemId}` (matches `FormLabel`'s `htmlFor`)
- `aria-describedby={!error ? formDescriptionId : formDescriptionId + " " + formMessageId}` (description always linked; message linked only on error)
- `aria-invalid={!!error}` (sets the data attribute that shadcn destructive styling reads)

**Props** : every prop accepted by Radix `Slot.Root`. Slot forwards everything to its single child.

**Hard requirement** : EXACTLY ONE React child element. Radix `Slot` throws when given multiple siblings or text-only children. ALWAYS wrap exactly one input. NEVER pass two siblings.

## 6. FormDescription

```tsx
function FormDescription(props: React.ComponentProps<"p">): JSX.Element
```

Renders `<p id={formDescriptionId} className="text-sm text-muted-foreground">`. The id is the description id from `FormItemContext`. ALWAYS include when the field needs helper text; the aria wiring is automatic via `FormControl`.

## 7. FormMessage

```tsx
function FormMessage(props: React.ComponentProps<"p">): JSX.Element
```

Renders `<p id={formMessageId} className="text-sm text-destructive">{body}</p>` where `body` is `String(error?.message ?? "")` when there is an error, otherwise `props.children`. Returns `null` when both `body` and `children` are empty.

ALWAYS render `<FormMessage />` exactly once per `FormItem`. NEVER pass children : the component derives its body from the form state. NEVER omit it : without `FormMessage` the zod error never renders.

## 8. useFormField (internal helper, exported)

```tsx
function useFormField(): {
  id: string
  name: string
  formItemId: string
  formDescriptionId: string
  formMessageId: string
  invalid: boolean
  isTouched: boolean
  isDirty: boolean
  isValidating: boolean
  error?: FieldError
}
```

Internal hook used by `FormLabel`, `FormControl`, `FormDescription`, `FormMessage`. Exported for advanced compositions (custom field primitives). ALWAYS call inside a descendant of both `FormField` and `FormItem`; throws otherwise.

## 9. useForm (react-hook-form)

```ts
function useForm<TFieldValues extends FieldValues>(
  props?: UseFormProps<TFieldValues>
): UseFormReturn<TFieldValues>
```

**Options** :

| Option | Type | Default | Notes |
|--------|------|---------|-------|
| `defaultValues` | `Partial<TFieldValues>` or async function | `{}` | Set ONCE at mount. Changing prop has no effect after mount. |
| `values` | `TFieldValues` | undefined | CONTROLLED. Re-resets form when prop changes. |
| `resolver` | `Resolver<TFieldValues>` | undefined | Validation function. Use `zodResolver(schema)`. |
| `mode` | `"onSubmit" \| "onBlur" \| "onChange" \| "onTouched" \| "all"` | `"onSubmit"` | When to validate first. |
| `reValidateMode` | `"onChange" \| "onBlur" \| "onSubmit"` | `"onChange"` | When to re-validate after a field has errored once. |
| `shouldUnregister` | `boolean` | `false` | Unmounted fields removed from state when `true`. |
| `shouldFocusError` | `boolean` | `true` | Focus the first errored field after submit fail. |
| `shouldUseNativeValidation` | `boolean` | `false` | Use browser-native HTML5 validation. |
| `criteriaMode` | `"firstError" \| "all"` | `"firstError"` | Report first error or every error per field. |
| `delayError` | `number` | undefined | Delay (ms) before showing error after change. |
| `disabled` | `boolean` | `false` | Disables every field. |
| `resetOptions` | `KeepStateOptions` | undefined | Defaults for `reset()` called via `values` prop. |

**Return value (UseFormReturn)** :

| Property | Type | Purpose |
|----------|------|---------|
| `control` | `Control<TFieldValues>` | Passed to `FormField`. |
| `register` | `(name, options?) => RegisterReturn` | For native inputs only. |
| `handleSubmit` | `(onValid, onInvalid?) => (event) => Promise<void>` | Wraps the submit handler. |
| `formState` | `FormState<TFieldValues>` | Live state object; see table below. |
| `setValue` | `(name, value, options?) => void` | Programmatic write. |
| `setError` | `(name, error, options?) => void` | Programmatic error (e.g. server-side). |
| `clearErrors` | `(name?) => void` | Clear one or all errors. |
| `getValues` | `(name?) => values` | Read without subscribing. |
| `getFieldState` | `(name, formState?) => FieldState` | Read field state without re-render. |
| `watch` | `(name?, defaultValue?) => value` | Subscribe to changes; re-renders. |
| `reset` | `(values?, options?) => void` | Reset entire form. |
| `resetField` | `(name, options?) => void` | Reset single field. |
| `trigger` | `(name?, options?) => Promise<boolean>` | Programmatically run validation. |
| `unregister` | `(name, options?) => void` | Remove field from state. |

**formState properties** :

| Property | Type | Purpose |
|----------|------|---------|
| `errors` | `FieldErrors<TFieldValues>` | All current errors, keyed by field name. |
| `isValid` | `boolean` | True when no errors and (mode-dependent) validation has run. |
| `isDirty` | `boolean` | True when any value differs from defaultValues. |
| `dirtyFields` | `DirtyFields<TFieldValues>` | Per-field dirty flag. |
| `touchedFields` | `TouchedFields<TFieldValues>` | Per-field touched flag. |
| `isSubmitting` | `boolean` | True while `onValid` Promise is pending. |
| `isSubmitted` | `boolean` | True after first submit attempt (valid or invalid). |
| `isSubmitSuccessful` | `boolean` | True after a successful submit. |
| `submitCount` | `number` | Number of submit attempts. |
| `isValidating` | `boolean` | True during async resolver execution. |
| `isLoading` | `boolean` | True while async `defaultValues` is pending. |
| `disabled` | `boolean` | Mirrors the `disabled` option. |

## 10. handleSubmit signature

```ts
handleSubmit(
  onValid: (values: TFieldValues, event?: BaseSyntheticEvent) => unknown | Promise<unknown>,
  onInvalid?: (errors: FieldErrors<TFieldValues>, event?: BaseSyntheticEvent) => unknown | Promise<unknown>
): (event?: BaseSyntheticEvent) => Promise<void>
```

The returned function calls `event.preventDefault()`, runs the resolver, and invokes either `onValid` (validation success) or `onInvalid` (validation failure). ALWAYS pass the RETURN value to `<form onSubmit={...}>`; never `handleSubmit` itself.

## 11. Controller (react-hook-form, what FormField wraps)

```tsx
function Controller<
  TFieldValues extends FieldValues,
  TName extends FieldPath<TFieldValues>,
>(props: ControllerProps<TFieldValues, TName>): ReactElement
```

`ControllerProps` is identical to the `FormField` prop set above. Use raw `Controller` directly when composing with the newer `Field` primitives (see [shadcn-syntax-field](../shadcn-syntax-field/SKILL.md)) or when not using shadcn `Form` at all.

## 12. zodResolver (factory)

```ts
import { zodResolver } from "@hookform/resolvers/zod"

zodResolver<Schema extends ZodType>(
  schema: Schema,
  schemaOptions?: Partial<ZodParams>,
  factoryOptions?: { mode?: "async" | "sync"; raw?: boolean }
): Resolver<z.infer<Schema>>
```

| Argument | Required | Notes |
|----------|----------|-------|
| `schema` | yes | Any zod schema. ALWAYS define at module scope (see `anti-patterns.md` §3). |
| `schemaOptions` | no | zod parse options (e.g. error map). |
| `factoryOptions.mode` | no | `"async"` forces async parse; default `"sync"` falls back to async when needed. |
| `factoryOptions.raw` | no | When `true`, returns raw zod-parsed values (post-coerce, post-transform). |

The import path is `@hookform/resolvers/zod`. NEVER import from `zod` itself; that package does not export a resolver.

## 13. Common zod schema patterns

```ts
import * as z from "zod"

// strings
z.string()                                // any string
z.string().min(1, "Required.")            // min length with custom message
z.string().max(50)
z.string().email("Invalid email.")        // email regex
z.string().url()                          // URL string
z.string().uuid()
z.string().regex(/^[A-Z]+$/, "Uppercase only.")
z.string().trim()                         // pre-validation trim
z.string().toLowerCase()

// numbers
z.number().int().min(0).max(100)
z.coerce.number()                         // coerce string -> number (for <input type="number">)
z.number().positive()
z.number().nonnegative()

// booleans
z.boolean()
z.literal(true)                           // must be exactly true

// dates
z.date()
z.coerce.date()                           // coerce ISO string -> Date

// enums
z.enum(["pending", "active", "archived"])
z.nativeEnum(MyEnum)

// optional / nullable / default
z.string().optional()                     // string | undefined
z.string().nullable()                     // string | null
z.string().nullish()                      // string | null | undefined
z.string().default("guest")               // fills undefined with default

// arrays
z.array(z.string()).min(1, "Pick at least one.")
z.array(z.object({ id: z.string(), qty: z.number() }))

// objects
z.object({
  username: z.string().min(2),
  email: z.string().email(),
}).strict()                               // forbid unknown keys

// unions and intersections
z.union([z.string(), z.number()])
z.discriminatedUnion("type", [
  z.object({ type: z.literal("a"), a: z.string() }),
  z.object({ type: z.literal("b"), b: z.number() }),
])

// refinements
z.string().refine((v) => v !== "admin", "Reserved name.")
z.object({ pw: z.string(), confirm: z.string() })
  .refine((d) => d.pw === d.confirm, {
    message: "Passwords do not match.",
    path: ["confirm"],                    // attaches the error to the right field
  })

// async refinement
z.string().email().refine(
  async (email) => {
    const res = await fetch(`/api/check?email=${email}`)
    return !(await res.json()).taken
  },
  { message: "Email already taken." }
)

// type inference
const schema = z.object({ ... })
type Values = z.infer<typeof schema>      // pre-transform input type
type Output = z.output<typeof schema>     // post-transform output type
type Input = z.input<typeof schema>       // alias of z.infer
```

ALWAYS attach `path: [...]` inside multi-field `.refine()` to surface the error on the user-visible field (e.g. the confirmation input, not the form root).
