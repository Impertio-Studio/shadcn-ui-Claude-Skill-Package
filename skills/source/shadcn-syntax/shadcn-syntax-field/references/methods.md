# shadcn ui : Field primitives API reference

All signatures verified against `apps/v4/registry/new-york-v4/ui/field.tsx` in `shadcn-ui/ui` (2026-05-19). The Field family is exported from `@/components/ui/field`.

## Import surface

```ts
export {
  Field,
  FieldLabel,
  FieldDescription,
  FieldError,
  FieldGroup,
  FieldLegend,
  FieldSeparator,
  FieldSet,
  FieldContent,
  FieldTitle,
}
```

Ten exported primitives. No `FieldRoot`, no `FieldProvider`, no `useField` hook. The primitive layer is composition-only.

## Field

```ts
type FieldProps = React.ComponentProps<"div"> & {
  orientation?: "vertical" | "horizontal" | "responsive"
}

function Field(props: FieldProps): JSX.Element
```

Renders :

```html
<div
  role="group"
  data-slot="field"
  data-orientation="vertical|horizontal|responsive"
  class="group/field flex w-full gap-3 data-[invalid=true]:text-destructive [orientation classes...]"
>
  {children}
</div>
```

Props :
- `orientation` : controls layout via `cva`. `"vertical"` (default) stacks. `"horizontal"` aligns label and control on one row. `"responsive"` is `flex-col` until the parent `FieldGroup` reaches `@md`, then switches to `flex-row`.
- `data-invalid` : pass `data-invalid={true|false}` directly as a JSX attribute. Drives `text-destructive` on the whole field via the `data-[invalid=true]:text-destructive` class.
- All other native `<div>` attributes (`className`, `id`, `aria-*`, `data-*`, `onClick`, ...).

ALWAYS one `Field` per logical input. NEVER nest a `Field` inside another `Field` (Fields stack as siblings inside a `FieldGroup` or `FieldSet`).

## FieldLabel

```ts
type FieldLabelProps = React.ComponentProps<typeof Label>
// where Label = shadcn @/components/ui/label (Radix Label)

function FieldLabel(props: FieldLabelProps): JSX.Element
```

Renders a shadcn `<Label>` with `data-slot="field-label"`. Inherits the Radix Label props :
- `htmlFor` : id of the associated control. REQUIRED for accessibility.
- `asChild` (boolean, default `false`) : forwards label semantics to a child element when needed.
- `className`, all other Radix Label props.

Special class behaviour : when a `FieldLabel` has a `<Field>` as a direct descendant (the selectable-card pattern), the label expands to a bordered card and highlights when the nested input is `data-state=checked`. This is the `has-[>[data-slot=field]]:` selector in the class string.

ALWAYS set `htmlFor` matching the control's `id`. NEVER rely on click-through via `<FieldLabel><Input/></FieldLabel>` wrapping : the inner input loses Field positioning.

## FieldDescription

```ts
type FieldDescriptionProps = React.ComponentProps<"p">

function FieldDescription(props: FieldDescriptionProps): JSX.Element
```

Renders :

```html
<p data-slot="field-description" class="text-sm leading-normal font-normal text-muted-foreground ...">
  {children}
</p>
```

Props :
- All native `<p>` attributes.
- ALWAYS provide an `id` and link it with `aria-describedby` on the input when the description is content the user needs (e.g. password rules). The primitive does NOT auto-generate ids.

Multiple `FieldDescription` siblings inside one `Field` are supported : the `nth-last-2:-mt-1` class adjusts spacing for the second-from-last child. The `[[data-variant=legend]+&]:-mt-1.5` selector tightens spacing immediately after a `<FieldLegend>`.

## FieldError

```ts
type FieldErrorProps = React.ComponentProps<"div"> & {
  errors?: Array<{ message?: string } | undefined>
}

function FieldError(props: FieldErrorProps): JSX.Element | null
```

Renders :

```html
<div role="alert" data-slot="field-error" class="text-sm font-normal text-destructive">
  {content}
</div>
```

`content` resolution (in order) :
1. If `children` is provided, use it verbatim.
2. Else if `errors` is empty or all entries are nullish, return `null` (no DOM at all).
3. Else build a unique-by-`message` set via `new Map(errors.map(e => [e?.message, e])).values()`.
4. If exactly one unique error remains, render its `message` as a string.
5. If more than one, render a `<ul class="ml-4 flex list-disc flex-col gap-1">` of `<li>{error.message}</li>` items.

Props :
- `errors` : Standard-Schema-compatible issue array. Accepts the shape emitted by Zod (`fieldState.error`), Valibot, ArkType, TanStack Form's `field.state.meta.errors`.
- `children` : escape hatch for custom error markup. Overrides `errors`.
- All native `<div>` attributes.

ALWAYS pass `errors` as an array even when there is a single error : `errors={[fieldState.error]}`. NEVER pass `errors={fieldState.error}` (object, not array) : the map iteration fails.

## FieldGroup

```ts
type FieldGroupProps = React.ComponentProps<"div">

function FieldGroup(props: FieldGroupProps): JSX.Element
```

Renders :

```html
<div data-slot="field-group" class="group/field-group @container/field-group flex w-full flex-col gap-7 ...">
  {children}
</div>
```

The two critical classes :
- `group/field-group` : establishes the named group for class scoping.
- `@container/field-group` : establishes a Tailwind v4 container query named `field-group`. The `Field orientation="responsive"` rule (`@md/field-group:flex-row`) keys to this name.

Props : all native `<div>` attributes.

ALWAYS wrap responsive fields in a `FieldGroup`. NEVER use `FieldGroup` for non-form layout : the `gap-7` is form-tuned and conflicts with general layouts.

## FieldSet

```ts
type FieldSetProps = React.ComponentProps<"fieldset">

function FieldSet(props: FieldSetProps): JSX.Element
```

Renders :

```html
<fieldset data-slot="field-set" class="flex flex-col gap-6 has-[>[data-slot=checkbox-group]]:gap-3 has-[>[data-slot=radio-group]]:gap-3">
  {children}
</fieldset>
```

Semantic HTML `<fieldset>`. The `has-[>[data-slot=checkbox-group]]:gap-3` selectors tighten spacing when the fieldset wraps a `CheckboxGroup` or `RadioGroup`.

ALWAYS pair with a `<FieldLegend>` as the first child. NEVER use `FieldSet` for non-grouped fields : a one-field `FieldSet` adds semantic noise.

## FieldLegend

```ts
type FieldLegendProps = React.ComponentProps<"legend"> & {
  variant?: "legend" | "label"
}

function FieldLegend(props: FieldLegendProps): JSX.Element
```

Renders :

```html
<legend data-slot="field-legend" data-variant="legend|label" class="mb-3 font-medium ...">
  {children}
</legend>
```

Variants :
- `"legend"` (default) : `text-base` ; used for top-level `FieldSet`.
- `"label"` : `text-sm` ; used for nested `FieldSet` inside another `FieldSet`.

Props : all native `<legend>` attributes.

ALWAYS use `variant="label"` for inner fieldsets to avoid the typography hierarchy collapsing. NEVER use a `<FieldLegend>` outside a `<FieldSet>` : browsers ignore `<legend>` outside `<fieldset>`.

## FieldContent

```ts
type FieldContentProps = React.ComponentProps<"div">

function FieldContent(props: FieldContentProps): JSX.Element
```

Renders :

```html
<div data-slot="field-content" class="group/field-content flex flex-1 flex-col gap-1.5 leading-snug">
  {children}
</div>
```

Use in the selectable-card pattern : a `FieldLabel` wraps a `<Field>` (gaining the card border), and `FieldContent` arranges a `FieldTitle` + `FieldDescription` next to a `Checkbox` / `Switch` / `RadioGroupItem`.

ALWAYS use inside a `FieldLabel` that wraps a card-style `Field`. NEVER use as a generic flex container.

## FieldSeparator

```ts
type FieldSeparatorProps = React.ComponentProps<"div"> & {
  children?: React.ReactNode
}

function FieldSeparator(props: FieldSeparatorProps): JSX.Element
```

Renders a horizontal divider via shadcn `<Separator>`, optionally with inline text (e.g. "or") centered on top of the line :

```html
<div data-slot="field-separator" data-content={!!children} class="relative -my-2 h-5 text-sm">
  <Separator class="absolute inset-0 top-1/2" />
  {children && <span class="relative mx-auto block w-fit bg-background px-2 text-muted-foreground">{children}</span>}
</div>
```

Props :
- `children` : optional inline content (commonly the string `"or"` between social and credentials sign-in).
- All native `<div>` attributes.

ALWAYS place between sibling `<Field>` elements inside a `<FieldGroup>`. NEVER use as a generic page divider : it is form-tuned (negative margin, height fixed at `h-5`).

## FieldTitle

```ts
type FieldTitleProps = React.ComponentProps<"div">

function FieldTitle(props: FieldTitleProps): JSX.Element
```

Renders :

```html
<div data-slot="field-label" class="flex w-fit items-center gap-2 text-sm leading-snug font-medium ...">
  {children}
</div>
```

Note : `data-slot="field-label"` matches `FieldLabel` but the element is a `<div>` (not a `<label>`), so it has NO `htmlFor`. Use when the title is decorative (inside a `FieldContent` next to an icon) and the actual focusable label is provided by the inner control's own labelling.

ALWAYS use inside a `FieldContent` in the selectable-card pattern. NEVER use as a replacement for `FieldLabel` : without `htmlFor` the label does not focus the control.

## cva variants (Field internals)

```ts
const fieldVariants = cva(
  "group/field flex w-full gap-3 data-[invalid=true]:text-destructive",
  {
    variants: {
      orientation: {
        vertical: ["flex-col [&>*]:w-full [&>.sr-only]:w-auto"],
        horizontal: [
          "flex-row items-center",
          "[&>[data-slot=field-label]]:flex-auto",
          "has-[>[data-slot=field-content]]:items-start has-[>[data-slot=field-content]]:[&>[role=checkbox],[role=radio]]:mt-px",
        ],
        responsive: [
          "flex-col @md/field-group:flex-row @md/field-group:items-center [&>*]:w-full @md/field-group:[&>*]:w-auto [&>.sr-only]:w-auto",
          "@md/field-group:[&>[data-slot=field-label]]:flex-auto",
          "@md/field-group:has-[>[data-slot=field-content]]:items-start @md/field-group:has-[>[data-slot=field-content]]:[&>[role=checkbox],[role=radio]]:mt-px",
        ],
      },
    },
    defaultVariants: {
      orientation: "vertical",
    },
  }
)
```

The container-query classes (`@md/field-group:...`) REQUIRE Tailwind v4 with the container plugin enabled and a `FieldGroup` ancestor providing `@container/field-group`. Without either, `orientation="responsive"` silently stays vertical.

## Hook : none

The Field family does NOT export a `useField` hook. State binding is the caller's responsibility (use `Controller` from `react-hook-form`, `form.Field` from `@tanstack/react-form`, or plain React `useState`).
