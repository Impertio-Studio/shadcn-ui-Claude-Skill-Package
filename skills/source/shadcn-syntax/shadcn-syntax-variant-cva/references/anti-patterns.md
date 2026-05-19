# Anti-patterns : cva, VariantProps, cn, Slot

Six numbered anti-patterns. Each entry follows the structure : symptom, WRONG code, WHY it fails, FIX.

---

## AP-1 : Concatenating className with template-string instead of cn()

**Symptom** : duplicate Tailwind utilities ship to the DOM (both `bg-primary` and `bg-red-500`). Caller `className` sometimes wins and sometimes loses depending on the order utilities appear in the generated CSS. Style "randomly" changes when unrelated utilities are added or removed elsewhere.

### WRONG

```tsx
function Button({ className, ...props }) {
  return (
    <button
      className={"inline-flex bg-primary text-primary-foreground " + className}
      {...props}
    />
  )
}
```

Or, equivalently :

```tsx
<button className={`inline-flex bg-primary ${className ?? ""}`} {...props} />
```

### WHY it fails

The concatenation produces a string with BOTH the component's `bg-primary` AND the caller's potential `bg-red-500`. The browser receives both utilities. Final visual result is decided by CSS source order (essentially random for the consumer). `tailwind-merge` was never invoked, so there is no conflict resolution. This breaks the entire shadcn override model.

Additional damage : `className` may be `undefined`, in which case the literal string `"undefined"` ends up as a class name (without the `?? ""` guard). Even with the guard, every consumer must remember to add it.

### FIX

ALWAYS route through `cn()` AND pass the caller `className` through `buttonVariants({ ..., className })` so it lands at the END of the accumulator :

```tsx
import { cn } from "@/lib/utils"

function Button({ className, variant, size, ...props }) {
  return (
    <button
      className={cn(buttonVariants({ variant, size, className }))}
      {...props}
    />
  )
}
```

`cn()` wraps `clsx` + `twMerge`. `clsx` flattens undefined values silently. `twMerge` resolves the `bg-primary` vs `bg-red-500` conflict in favour of the caller (last position wins). The DOM receives exactly one background utility.

---

## AP-2 : Wrong order assumption about compoundVariants

**Symptom** : a compound entry like `{ variant: "outline", size: "sm", class: "border-red-500" }` does not override the `border-input` declared inside `variants.variant.outline`. The DOM shows `border-input` instead, or shows both classes with CSS specificity arbitrating.

### WRONG

Believing that compound classes carry higher specificity than variant classes :

```tsx
const buttonVariants = cva("inline-flex", {
  variants: {
    variant: { outline: "border border-input bg-background" },
    size: { sm: "h-8 px-3" },
  },
  compoundVariants: [
    // ASSUMPTION : this overrides "border-input" automatically.
    { variant: "outline", size: "sm", class: "border-red-500" },
  ],
  defaultVariants: { variant: "outline", size: "sm" },
})

// Then the COMPONENT BYPASSES cn() :
function Button({ variant, size }) {
  return <button className={buttonVariants({ variant, size })} />
}
```

### WHY it fails

cva concatenates classes in this order : base, defaults-fill, variants, compound, then caller `className` (if any). The compound IS later in the string than the variant, so `tailwind-merge` correctly resolves the conflict IF `cn()` is invoked. But CSS specificity alone does NOT pick a winner ; both `border-input` and `border-red-500` map to `border-color` and have equal specificity. Without `twMerge` the visual result depends on which class appears later in the generated stylesheet, which is implementation-defined.

### FIX

ALWAYS wrap the cva return in `cn()`. The compound's later position guarantees `twMerge` drops the variant's `border-input` :

```tsx
import { cn } from "@/lib/utils"

function Button({ variant, size, className }) {
  return (
    <button
      className={cn(buttonVariants({ variant, size, className }))}
    />
  )
}
```

When designing the compound, mentally reach for the most specific override (e.g. `border-2 border-red-500` rather than just `border-red-500`) only when the conflict is across DIFFERENT properties. For same-property conflicts, `twMerge` does the work for you ; you do not need extra utility classes.

---

## AP-3 : Using boolean variant value without quoting

**Symptom** : `VariantProps<typeof X>` produces a variant union that includes `"true" | "false"` (strings), forcing consumers to write `<Component loading="true">` instead of `<Component loading>`. Or worse : TypeScript silently accepts an arbitrary string.

### WRONG

```ts
const spinnerVariants = cva("inline-block", {
  variants: {
    // Misconception : keys can be JS booleans.
    loading: {
      true: "animate-spin",   // valid JS, but inferred as the string "true"
      false: "",
    },
  },
  defaultVariants: { loading: false },
})

type Props = VariantProps<typeof spinnerVariants>
//   ^ { loading?: "true" | "false" | null | undefined }
```

Consumers must now type :

```tsx
<Spinner loading="true" />     // works, ugly
<Spinner loading={true} />     // TYPE ERROR : `true` is not "true"
```

### WHY it fails

JavaScript object keys are always strings (or symbols). `{ true: "..." }` is shorthand for `{ "true": "..." }`. cva reads the variant axis as a record of string keys, so `VariantProps` reports `"true"` (the string) and not `true` (the boolean). The component API now looks broken to anyone expecting a React-style boolean prop.

### FIX

Two clean options :

**Option A** : keep the variant as a string with `"on"`/`"off"` keys (or `"yes"`/`"no"`, etc.) :

```ts
const spinnerVariants = cva("inline-block", {
  variants: { state: { on: "animate-spin", off: "" } },
  defaultVariants: { state: "off" },
})
```

**Option B** : drop the variant from cva and handle the boolean OUTSIDE the cva call :

```ts
const spinnerBase = cva("inline-block")

function Spinner({ loading = false, className }: { loading?: boolean; className?: string }) {
  return (
    <div
      className={cn(spinnerBase(), loading && "animate-spin", className)}
    />
  )
}
```

Option B preserves the React-idiomatic `<Spinner loading />` API while still routing through `cn()`. Use Option A only when the axis genuinely has more than two states or you want a single source of truth for the class strings.

---

## AP-4 : Forgetting to export VariantProps

**Symptom** : consumers cannot extend the component's prop type, must manually retype the variant union, and their typings drift when the component adds or removes variants.

### WRONG

```tsx
// components/ui/button.tsx
const buttonVariants = cva(/* ... */)

function Button(/* ... */) { /* ... */ }

export { Button }   // Only the component is exported.
```

A consumer wants to build a `LinkButton` that styles a Next.js Link with the same variants :

```tsx
// app/components/link-button.tsx
import { Button } from "@/components/ui/button"

// CANNOT import buttonVariants. CANNOT import VariantProps<typeof buttonVariants>.
// Has to retype by hand :
type LinkButtonProps = {
  variant?: "default" | "destructive" | "outline" | "secondary" | "ghost" | "link"
  size?: "default" | "sm" | "lg" | "icon"
}
```

### WHY it fails

The retyped union duplicates knowledge that already exists in `buttonVariants`. When a maintainer adds `variant: { warning: "..." }` to the cva config, the consumer's type silently misses it ; passing `variant="warning"` from a parent surfaces no error AT the component boundary, even though IntelliSense should have shown the new option. Refactoring becomes a manual chase across the codebase.

### FIX

ALWAYS export the cva return value alongside the component :

```tsx
// components/ui/button.tsx
const buttonVariants = cva(/* ... */)

function Button(/* ... */) { /* ... */ }

export { Button, buttonVariants }
```

Now consumers can derive types and reuse classes :

```tsx
// app/components/link-button.tsx
import Link, { type LinkProps } from "next/link"
import { type VariantProps } from "class-variance-authority"
import { buttonVariants } from "@/components/ui/button"
import { cn } from "@/lib/utils"

type LinkButtonProps =
  LinkProps &
  VariantProps<typeof buttonVariants> &
  { className?: string }

export function LinkButton({ variant, size, className, ...props }: LinkButtonProps) {
  return (
    <Link
      className={cn(buttonVariants({ variant, size, className }))}
      {...props}
    />
  )
}
```

The union stays in sync automatically. Adding a new variant to `buttonVariants` immediately enriches `LinkButtonProps` without further edits.

Note : exporting both a component AND a cva object from one file triggers the `react-refresh/only-export-components` ESLint rule in some projects (issue #7736). See `shadcn-errors-fast-refresh-lint` for the workaround (per-file disable or split variants into `button.variants.ts`).

---

## AP-5 : asChild with multiple children (or with zero)

**Symptom** : runtime error `React.Children.only expected to receive a single React element child`, OR `Slot` silently renders nothing, OR icons inside the button disappear when `asChild` is added.

### WRONG : multiple children

```tsx
<Button asChild>
  <Link href="/about">About</Link>
  <ChevronRight />
</Button>
```

### WRONG : zero children

```tsx
<Button asChild />            // No child to slot into
```

### WRONG : raw text and an element side-by-side

```tsx
<Button asChild>
  Click :
  <Link href="/about">About</Link>
</Button>
```

### WHY it fails

Radix `Slot.Root` (and the legacy `@radix-ui/react-slot` Slot) calls `React.Children.only(children)` under the hood. That function throws synchronously when the children count is anything other than exactly ONE React element. Strings or fragments containing multiple nodes count as multiple children.

When `asChild` is `true`, the Slot becomes the rendered output. Without `asChild`, the same Button would happily render multiple children (the literal `<button>` element accepts any children). The contract changes the moment `asChild` is set.

### FIX

When `asChild` is needed, place ONE root element inside ; put all sub-content inside that root :

```tsx
<Button asChild variant="outline">
  <Link href="/about">
    About <ChevronRight />
  </Link>
</Button>
```

Or omit `asChild` and let Button render a real `<button>` :

```tsx
<Button variant="outline" onClick={() => navigate("/about")}>
  About <ChevronRight />
</Button>
```

For dynamic single-child situations (conditionally wrap with Link), use the standard ternary :

```tsx
{href ? (
  <Button asChild>
    <Link href={href}>{label}</Link>
  </Button>
) : (
  <Button onClick={onClick}>{label}</Button>
)}
```

Additional rule : NEVER use `asChild` to wrap a second interactive element (button inside a link, link inside a button). The combined output must be a single accessible widget.

---

## AP-6 : Re-implementing cva manually with if/else chains

**Symptom** : code accumulates a series of `if (variant === "outline") classes += "border ..."` lines, type inference is lost, the component drifts from the shadcn convention, ESLint may flag the variable-mutation pattern, and the codebase loses the discoverability that `VariantProps<typeof X>` provides.

### WRONG

```tsx
function Button({
  variant = "default",
  size = "default",
  className,
  asChild = false,
  ...props
}: {
  variant?: "default" | "outline"
  size?: "default" | "sm"
  className?: string
  asChild?: boolean
}) {
  let classes = "inline-flex items-center justify-center rounded-md"

  if (variant === "default") classes += " bg-primary text-primary-foreground"
  if (variant === "outline") classes += " border bg-background"

  if (size === "default") classes += " h-9 px-4"
  if (size === "sm") classes += " h-8 px-3"

  // compoundVariant equivalent :
  if (variant === "outline" && size === "sm") classes += " ring-1 ring-ring/20"

  if (className) classes += " " + className

  const Comp = asChild ? Slot.Root : "button"
  return <Comp className={classes} {...props} />
}
```

### WHY it fails

1. **No conflict resolution** : the final `classes` string is fed directly to `className` ; if the caller passes `bg-red-500`, the DOM gets BOTH `bg-primary` and `bg-red-500`. `twMerge` was never invoked.
2. **No type inference** : the prop type was hand-written. Adding a new variant requires editing TWO places (the runtime `if` branch AND the type union). Consumers cannot derive types via `VariantProps`.
3. **Mutability code smell** : the `let classes` pattern with successive `+=` violates immutability conventions and triggers some ESLint configs.
4. **Loses the convention** : every shadcn component in the project follows the cva pattern. Hand-rolling one component breaks the discoverability of `buttonVariants` for sibling components (`LinkButton`, badge-styled-as-button variants).
5. **Compound logic is unreadable** : with three or more variant axes, the `if` chains explode combinatorially. cva's `compoundVariants` array is a declarative source of truth.

### FIX

ALWAYS use cva for the variant logic, ALWAYS export the variants object, ALWAYS pass the result through `cn()` :

```tsx
import { cva, type VariantProps } from "class-variance-authority"
import { Slot } from "radix-ui"
import { cn } from "@/lib/utils"

const buttonVariants = cva(
  "inline-flex items-center justify-center rounded-md",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground",
        outline: "border bg-background",
      },
      size: { default: "h-9 px-4", sm: "h-8 px-3" },
    },
    compoundVariants: [
      { variant: "outline", size: "sm", class: "ring-1 ring-ring/20" },
    ],
    defaultVariants: { variant: "default", size: "default" },
  }
)

function Button({
  className,
  variant,
  size,
  asChild = false,
  ...props
}: React.ComponentProps<"button"> &
  VariantProps<typeof buttonVariants> & { asChild?: boolean }) {
  const Comp = asChild ? Slot.Root : "button"
  return (
    <Comp
      className={cn(buttonVariants({ variant, size, className }))}
      {...props}
    />
  )
}

export { Button, buttonVariants }
```

The new variant union is automatic. Compound rules are declarative. Conflict resolution is centralised. The component is consistent with every other shadcn component in the project.
