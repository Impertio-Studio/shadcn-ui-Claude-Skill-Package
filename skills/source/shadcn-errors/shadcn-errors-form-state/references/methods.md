# Methods : react-hook-form APIs Relevant to shadcn Form Errors

Every signature here is verified against `https://react-hook-form.com/docs/useform` and the shadcn Form recipe at `https://ui.shadcn.com/docs/forms/react-hook-form` (last verified 2026-05-19). Type signatures shown are simplified for clarity ; the full generics are in `react-hook-form`'s `dist/types`.

## useForm

```ts
function useForm<TFieldValues = FieldValues, TContext = any, TTransformed = TFieldValues>(
  props?: UseFormProps<TFieldValues, TContext, TTransformed>
): UseFormReturn<TFieldValues, TContext, TTransformed>

interface UseFormProps<TFieldValues> {
  // Validation strategy. Default "onSubmit". "onChange" or "onTouched" enables live
  // isValid updates ; cheaper modes do not update isValid until after first submit.
  mode?: "onSubmit" | "onChange" | "onBlur" | "onTouched" | "all"

  // Re-validation strategy after first submission. Default "onChange".
  reValidateMode?: "onSubmit" | "onChange" | "onBlur"

  // Initial values. SET ONCE on mount. Subsequent reference changes are ignored.
  // MUST include every field that will become a controlled input ; partial
  // defaultValues cause uncontrolled-to-controlled flip warnings.
  defaultValues?: DefaultValues<TFieldValues> | (() => Promise<DefaultValues<TFieldValues>>)

  // Controlled initial values. Reactive : every reference change resets the form.
  // Use when initial data arrives async and may change (edit-record flow).
  values?: TFieldValues

  // Fine-grained control over the reactive `values` reset behaviour.
  resetOptions?: KeepStateOptions

  // Async validator. zodResolver(schema) returns a Resolver<TFieldValues>.
  resolver?: Resolver<TFieldValues, TContext, TTransformed>

  // Extra context passed to the resolver (rarely needed with zod).
  context?: TContext

  // Skip mount-time validation. Default false.
  shouldFocusError?: boolean

  // Strip unregistered fields from final values. Default false.
  shouldUnregister?: boolean

  // Use native HTML validation alongside JS validators. Default false.
  shouldUseNativeValidation?: boolean

  // Disable the entire form (bulk-disable). React 18+ only.
  disabled?: boolean

  // Validation delay in milliseconds. Useful for "onChange" mode with async refine.
  delayError?: number

  // Allow non-string error messages (e.g. JSX). Default false.
  criteriaMode?: "firstError" | "all"
}
```

### UseFormReturn

```ts
interface UseFormReturn<TFieldValues> {
  control: Control<TFieldValues>                     // pass to FormField / Controller
  register: UseFormRegister<TFieldValues>            // native-input shortcut
  handleSubmit: UseFormHandleSubmit<TFieldValues>    // wraps onSubmit + validation
  watch: UseFormWatch<TFieldValues>                  // SUBSCRIPTION ; re-renders host
  getValues: UseFormGetValues<TFieldValues>          // snapshot, no subscription
  getFieldState: UseFormGetFieldState<TFieldValues>  // per-field formState slice
  setError: UseFormSetError<TFieldValues>            // imperative error injection
  clearErrors: UseFormClearErrors<TFieldValues>
  setValue: UseFormSetValue<TFieldValues>            // imperative value update
  setFocus: UseFormSetFocus<TFieldValues>
  trigger: UseFormTrigger<TFieldValues>              // run validation imperatively
  reset: UseFormReset<TFieldValues>                  // bulk reset + KeepStateOptions
  resetField: UseFormResetField<TFieldValues>        // reset single field
  unregister: UseFormUnregister<TFieldValues>
  formState: FormState<TFieldValues>                 // Proxy ; subscribe by reading
  subscribe: UseFormSubscribe<TFieldValues>          // event-based subscription
}
```

## Controller

```ts
function Controller<TFieldValues, TName>(
  props: ControllerProps<TFieldValues, TName>
): React.ReactElement

interface ControllerProps<TFieldValues, TName> {
  name: TName                                        // path string into TFieldValues
  control: Control<TFieldValues>                     // from useForm().control
  defaultValue?: FieldPathValue<TFieldValues, TName> // optional ; prefer defaultValues on useForm
  rules?: RegisterOptions                            // per-field rules (less relevant with resolver)
  shouldUnregister?: boolean
  disabled?: boolean
  render: ({
    field: {
      name: TName
      value: FieldPathValue<TFieldValues, TName>
      onChange: (value: any) => void                 // CALL THIS to update form state
      onBlur: () => void                             // CALL THIS to mark touched
      ref: RefCallBack                               // forward to focusable element
      disabled?: boolean
    },
    fieldState: {
      invalid: boolean
      isTouched: boolean
      isDirty: boolean
      isValidating: boolean
      error?: FieldError
    },
    formState: FormState<TFieldValues>               // full form state ; subscribed
  }) => React.ReactElement
}
```

The shadcn `FormField` is a thin wrapper around `Controller` that ALSO publishes a context for `FormItem`, `FormLabel`, `FormControl`, `FormDescription`, and `FormMessage` to read. Using `Controller` directly works but loses the aria wiring those primitives provide.

## formState (Proxy subscription)

```ts
interface FormState<TFieldValues> {
  isDirty: boolean                          // any value changed from defaultValues
  dirtyFields: Partial<{ [K in keyof TFieldValues]: boolean }>
  touchedFields: Partial<{ [K in keyof TFieldValues]: boolean }>
  defaultValues: Readonly<DeepPartial<TFieldValues>>
  isSubmitted: boolean                      // handleSubmit was called at least once
  isSubmitSuccessful: boolean               // last submit completed without throwing
  isSubmitting: boolean                     // submit handler is in-flight (async)
  isLoading: boolean                        // async defaultValues are loading
  submitCount: number
  isValid: boolean                          // updates only in onChange/onTouched/onBlur/all modes
  isValidating: boolean                     // async resolver is running
  errors: FieldErrors<TFieldValues>
  disabled: boolean
}
```

CRITICAL : `formState` is a Proxy. Reading a key creates a subscription. Reading no keys means no re-renders. Always destructure :

```ts
// Correct subscription pattern.
const { formState: { isSubmitting, isValid, errors } } = useForm<FormData>()
```

For component-isolated subscription (no host re-render), use `useFormState({ control })` in a child.

## handleSubmit signature

```ts
type UseFormHandleSubmit<TFieldValues, TTransformedValues = TFieldValues> = (
  onValid: SubmitHandler<TTransformedValues>,
  onInvalid?: SubmitErrorHandler<TFieldValues>
) => (e?: React.BaseSyntheticEvent) => Promise<void>

type SubmitHandler<T> = (data: T, event?: React.BaseSyntheticEvent) => unknown | Promise<unknown>
type SubmitErrorHandler<T> = (errors: FieldErrors<T>, event?: React.BaseSyntheticEvent) => unknown | Promise<unknown>
```

The returned function is what you pass to `<form onSubmit={...}>`. It calls `preventDefault`, runs validation, then either invokes `onValid(data, event)` or `onInvalid(errors, event)`. If `onInvalid` is omitted, invalid submissions silently do nothing : errors land in `formState.errors` and FormMessage renders them, but no callback fires.

## setError (server validation handoff)

```ts
type UseFormSetError<TFieldValues> = (
  name: FieldPath<TFieldValues> | `root.${string}` | "root",
  error: { type: string; message?: string },
  options?: { shouldFocus?: boolean }
) => void
```

Use the literal `"root"` (or `` `root.${string}` ``) to attach a form-wide error that is NOT bound to any single field. Read it via `formState.errors.root` or `formState.errors.root.<name>`.

```ts
async function onSubmit(data: FormData) {
  const res = await api.submit(data)
  if (!res.ok) {
    // Map server field errors to react-hook-form.
    for (const [field, message] of Object.entries(res.fieldErrors)) {
      form.setError(field as keyof FormData, { type: "server", message }, { shouldFocus: true })
    }
    // Form-wide error.
    if (res.formError) {
      form.setError("root.serverError", { type: "server", message: res.formError })
    }
  }
}
```

## reset and resetField

```ts
type UseFormReset<TFieldValues> = (
  values?: DefaultValues<TFieldValues> | TFieldValues,
  keepStateOptions?: KeepStateOptions
) => void

interface KeepStateOptions {
  keepErrors?: boolean
  keepDirty?: boolean
  keepDirtyValues?: boolean
  keepValues?: boolean
  keepDefaultValues?: boolean
  keepIsSubmitted?: boolean
  keepIsSubmitSuccessful?: boolean
  keepTouched?: boolean
  keepIsValid?: boolean
  keepSubmitCount?: boolean
}
```

`form.reset()` with no arguments resets to `defaultValues` and clears all error/dirty/touched state. Pass `keepStateOptions` to retain specific slices.

## watch vs useWatch (the re-render distinction)

`form.watch()` has three call signatures :

```ts
form.watch()                                          // ALL values, subscribes host
form.watch("fieldName")                               // ONE value, subscribes host
form.watch(["a", "b"])                                // MULTIPLE, subscribes host
form.watch((values, { name, type }) => { ... })       // CALLBACK form ; side effect only, no host re-render
```

The first three subscribe the CALLING component to re-render. Inline at the form root, this re-renders every field on every keystroke.

`useWatch({ control, name })` is the hook form. It subscribes the HOOK SCOPE only :

```ts
function PreviewField({ control }: { control: Control<FormData> }) {
  const value = useWatch({ control, name: "email" })  // only THIS component re-renders
  return <p>Preview : {value}</p>
}
```

Use `useWatch` when you need a value for rendering. Use `form.watch(callback)` when you need a side effect (logging, autosave). Avoid `form.watch()` at the form-root component body unless you want the entire form to re-render on every keystroke.

## Async zod refine

```ts
import * as z from "zod"

const schema = z.object({
  username: z.string().min(3).refine(
    async (value) => {
      const res = await fetch(`/api/check-username?u=${encodeURIComponent(value)}`)
      const { available } = await res.json()
      return available
    },
    { message: "Username already taken" }
  ),
})
```

While the async refine runs, `formState.isValidating` is `true`. To avoid hammering the server on every keystroke, debounce or gate by `mode: "onBlur"` :

```ts
const form = useForm({
  resolver: zodResolver(schema),
  mode: "onBlur",                                     // validate on blur, not on change
  defaultValues: { username: "" },
})
```

For more advanced control (abort on next keystroke, debounce internally), use `z.superRefine` with an explicit `AbortController` :

```ts
const schema = z.object({
  username: z.string().min(3),
}).superRefine(async (data, ctx) => {
  const controller = new AbortController()
  try {
    const res = await fetch(`/api/check?u=${data.username}`, { signal: controller.signal })
    const { available } = await res.json()
    if (!available) {
      ctx.addIssue({ code: z.ZodIssueCode.custom, path: ["username"], message: "Already taken" })
    }
  } catch (e) {
    if ((e as Error).name !== "AbortError") throw e
  }
})
```

In practice, prefer `mode: "onBlur"` + `delayError` over manual AbortController plumbing : it produces the same UX with less code.

## zodResolver

```ts
import { zodResolver } from "@hookform/resolvers/zod"

const resolver = zodResolver(schema, /* schemaOptions */ undefined, /* factoryOptions */ {
  mode: "async",                                      // default "async" ; "sync" disables async refines
  raw: false,                                         // false = transformed output ; true = raw input
})
```

Schema path identifiers MUST match `FormField name` strings exactly. `z.object({ email: ... })` errors land in `errors.email`. `<FormField name="email" />` reads `errors.email`. A one-character drift (`name="Email"` or schema key `e_mail`) means the field is never validated.
