# shadcn ui : Field primitive anti-patterns

Six canonical anti-patterns observed when wiring the Field family. Each entry lists the failing code, WHY it fails, and the fix.

---

## 1. Missing `htmlFor` on `FieldLabel`

### Failing code

```tsx
<Controller
  name="title"
  control={form.control}
  render={({ field, fieldState }) => (
    <Field data-invalid={fieldState.invalid}>
      <FieldLabel>Title</FieldLabel>            {/* no htmlFor */}
      <Input
        {...field}
        id="title"
        aria-invalid={fieldState.invalid}
      />
      {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
    </Field>
  )}
/>
```

### Why this fails

`FieldLabel` wraps a Radix `<Label>` which renders a native `<label>` element. Without `htmlFor`, the browser does NOT associate the label with the input :
1. Clicking the label does NOT focus the input.
2. Screen readers do NOT announce the label when the input receives focus.
3. Voice-control software cannot reach the input via "Click Title".

The `Field` primitive does NOT auto-generate ids (unlike the older `FormItem` which calls `useId()` internally). The pairing is the caller's responsibility.

### Fix

ALWAYS set `htmlFor={field.name}` AND `id={field.name}` :

```tsx
<FieldLabel htmlFor={field.name}>Title</FieldLabel>
<Input {...field} id={field.name} aria-invalid={fieldState.invalid} />
```

Using `field.name` for both keeps them in lockstep ; `name` is unique per schema key and Controller already provides it.

---

## 2. `FieldError` rendered outside its parent `Field`

### Failing code

```tsx
<Controller
  name="title"
  control={form.control}
  render={({ field, fieldState }) => (
    <>
      <Field data-invalid={fieldState.invalid}>
        <FieldLabel htmlFor={field.name}>Title</FieldLabel>
        <Input {...field} id={field.name} aria-invalid={fieldState.invalid} />
      </Field>
      {fieldState.invalid && <FieldError errors={[fieldState.error]} />}  {/* outside Field */}
    </>
  )}
/>
```

### Why this fails

The Field styling system uses Tailwind's named-group selectors. `Field` declares `group/field` on its root. `FieldError` lives inside the same `group/field` scope and inherits destructive colour styling and spacing rules. When `FieldError` is moved OUTSIDE the `Field`:
1. The `group/field` selector does not match the error : it sits in the sibling cascade.
2. The negative-margin coordination between `Field`, `FieldDescription`, and `FieldError` breaks (the `nth-last-2:-mt-1` selector inside `FieldDescription` references its position inside the `Field`).
3. Multiple Fields rendered as siblings produce ambiguous error placement : which Field does the orphan error describe?

### Fix

ALWAYS render `FieldError` as a direct child of the `Field` it describes :

```tsx
<Field data-invalid={fieldState.invalid}>
  <FieldLabel htmlFor={field.name}>Title</FieldLabel>
  <Input {...field} id={field.name} aria-invalid={fieldState.invalid} />
  {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
</Field>
```

---

## 3. Mixing `Field` with `FormField` in the same tree

### Failing code

```tsx
<Form {...form}>
  <form onSubmit={form.handleSubmit(onSubmit)}>
    <FormField
      control={form.control}
      name="email"
      render={({ field }) => (
        <Field data-invalid={false}>                {/* Field instead of FormItem */}
          <FieldLabel>Email</FieldLabel>
          <FormControl>
            <Input {...field} />
          </FormControl>
          <FieldError />                            {/* expects errors prop ; gets none */}
        </Field>
      )}
    />
  </form>
</Form>
```

### Why this fails

`FormField` publishes a `FormFieldContext` and a `FormItemContext`. The shadcn `FormControl`, `FormLabel`, `FormDescription`, and `FormMessage` all call `useFormField()` which reads BOTH contexts to wire `id`, `aria-describedby`, `aria-invalid`, and the error message.

`Field` does NOT participate in this context system. Mixing the two produces :
1. `FormControl` wraps `<Input>` in a Radix `<Slot>` and forwards generated `id` and `aria-describedby` IDs that point to non-existent `FormItem`-generated ids (because the parent is `Field`, not `FormItem`).
2. `FieldError` rendered with no `errors` prop and no `children` returns `null` : the user sees no error.
3. The `data-invalid` on `Field` is hardcoded (`false` in the failing code) : the destructive colour never fires.

### Fix

Pick ONE path per form and stick to it. Either :

```tsx
// Path A : Field + Controller (recommended for new code)
<Controller
  name="email"
  control={form.control}
  render={({ field, fieldState }) => (
    <Field data-invalid={fieldState.invalid}>
      <FieldLabel htmlFor={field.name}>Email</FieldLabel>
      <Input {...field} id={field.name} aria-invalid={fieldState.invalid} />
      {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
    </Field>
  )}
/>
```

OR :

```tsx
// Path B : Form composition (legacy)
<FormField
  control={form.control}
  name="email"
  render={({ field }) => (
    <FormItem>
      <FormLabel>Email</FormLabel>
      <FormControl>
        <Input {...field} />
      </FormControl>
      <FormMessage />
    </FormItem>
  )}
/>
```

NEVER mix imports from `@/components/ui/field` and `@/components/ui/form` inside the same `<form>` tree.

---

## 4. Missing `aria-invalid` on the inner input

### Failing code

```tsx
<Controller
  name="title"
  control={form.control}
  render={({ field, fieldState }) => (
    <Field data-invalid={fieldState.invalid}>          {/* visual cue only */}
      <FieldLabel htmlFor={field.name}>Title</FieldLabel>
      <Input {...field} id={field.name} />              {/* no aria-invalid */}
      {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
    </Field>
  )}
/>
```

### Why this fails

`data-invalid` on the `Field` wrapper is PRESENTATIONAL. It drives the destructive colour token via the `data-[invalid=true]:text-destructive` class but emits NO accessibility signal :
1. Screen readers do NOT announce the input as invalid.
2. Voice-control software has no way to detect the error state programmatically.
3. WCAG 2.1 SC 1.3.1 (Info and Relationships) and SC 4.1.2 (Name, Role, Value) fail : the relationship between the error and the control is lost.

`Field` does NOT propagate `aria-invalid` down to its children (unlike the older `FormControl` which wrapped the input in a `Slot.Root` that forwarded attributes). The input must carry `aria-invalid` itself.

### Fix

ALWAYS set BOTH attributes :

```tsx
<Field data-invalid={fieldState.invalid}>
  <FieldLabel htmlFor={field.name}>Title</FieldLabel>
  <Input
    {...field}
    id={field.name}
    aria-invalid={fieldState.invalid}
  />
  {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
</Field>
```

For non-input focusable controls (Radix Select, Checkbox, Switch), put `aria-invalid` on the focusable element :
- `Select` -> `SelectTrigger aria-invalid={...}`
- `Checkbox` -> `Checkbox aria-invalid={...}`
- `RadioGroup` -> `RadioGroup aria-invalid={...}` (the root receives focus management).

---

## 5. Using `Field` as a generic layout wrapper

### Failing code

```tsx
<Field orientation="horizontal">           {/* no form control inside */}
  <Avatar src={user.avatar} />
  <div>
    <p>{user.name}</p>
    <p>{user.email}</p>
  </div>
  <Button>Edit</Button>
</Field>
```

### Why this fails

`Field` is `<div role="group">` with form-specific behaviour baked in :
1. `role="group"` advertises a form-control group to assistive tech ; a profile card with no controls confuses screen readers.
2. The `data-[invalid=true]:text-destructive` class on the root means a stray `data-invalid` attribute anywhere on the page can light up the card in red.
3. The `gap-3`, `group/field`, and `[&>*]:w-full` rules are tuned for label-plus-control geometry, not arbitrary content.
4. The `orientation` cva variant assumes a `FieldGroup` ancestor for the responsive case ; placing `Field` outside a form may not provide one.

### Fix

For non-form layouts, use a plain `<div>` with Tailwind utilities, or build an app-specific composition component. Reserve `Field` for one logical form control plus its label, description, and error.

```tsx
// Profile card : use a plain div, not Field
<div className="flex items-center gap-3" role="group" aria-labelledby="profile-heading">
  <Avatar src={user.avatar} />
  <div className="flex flex-col">
    <p id="profile-heading">{user.name}</p>
    <p className="text-sm text-muted-foreground">{user.email}</p>
  </div>
  <Button>Edit</Button>
</div>
```

---

## 6. Passing a single error object to `FieldError errors` instead of an array

### Failing code

```tsx
<FieldError errors={fieldState.error} />          {/* object, not array */}
```

### Why this fails

`FieldError` declares `errors?: Array<{ message?: string } | undefined>`. Internally it calls :

```ts
const uniqueErrors = [
  ...new Map(errors.map((error) => [error?.message, error])).values(),
]
```

When `errors` is a plain object (not an array) :
1. `errors.map` throws `TypeError: errors.map is not a function`.
2. The component crashes with a React error boundary trigger.
3. The whole form view replaces with the error fallback.

This is the most common copy-paste mistake : the older `FormMessage` did not need an explicit error prop (it read from context), and developers carry that habit over.

### Fix

ALWAYS wrap in an array literal :

```tsx
{/* single error from react-hook-form Controller */}
<FieldError errors={[fieldState.error]} />

{/* multiple errors from TanStack Form */}
<FieldError errors={field.state.meta.errors} />     {/* already an array */}

{/* custom error message */}
<FieldError>Custom error text</FieldError>          {/* children path, no errors prop */}
```

NEVER pass `fieldState.error.message` (string) inside the array : the de-dup map keys on `.message` and bare strings produce undefined keys. Pass the full issue object.
