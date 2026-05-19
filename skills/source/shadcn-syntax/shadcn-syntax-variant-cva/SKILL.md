---
name: shadcn-syntax-variant-cva
description: >
  Use when defining or modifying variants on a shadcn ui component, writing a
  new component that needs `variant` / `size` props, extending the Button with
  a custom variant, debugging why a `className` override is not winning, or
  inferring TypeScript props from a `cva()` call.
  Prevents the common mistakes : concatenating Tailwind class strings with
  `+ " "` instead of `cn()`, forgetting `tailwind-merge` so duplicate
  utilities leak through, mis-ordering `compoundVariants` so the override
  loses to the default, omitting `VariantProps` so consumers cannot extend
  the component type, and re-implementing variant logic with hand-written
  `if`-branches that destroy type inference.
  Covers the full `cva(base, config)` signature (base as string or string
  array, `variants`, `compoundVariants`, `defaultVariants`), the
  `VariantProps<typeof X>` inference pattern, the canonical
  `cn(...inputs)` helper composition of `clsx` plus `twMerge`, the
  evaluation order (default then variant then compound), and the Radix
  `Slot` (`asChild`) polymorphic-rendering pattern. Code examples are
  version-explicit for Tailwind v3 versus Tailwind v4.
  Keywords: cva, class-variance-authority, VariantProps, cn helper,
  twMerge, tailwind-merge, clsx, compoundVariants, defaultVariants,
  variants, Slot, asChild, polymorphic component, buttonVariants,
  variant not applying, className override not working, duplicate
  tailwind classes, classes not merging, how do I add a variant,
  what is cva, how does VariantProps work, why is className ignored,
  why are my Tailwind classes duplicated, type-safe variants, Radix Slot.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui : Variant API with cva

shadcn components express their visual variants through `class-variance-authority` (cva). The single most common source of styling bugs in a shadcn project is misuse of cva or its companion helper `cn()`. ALWAYS internalise this skill before adding, editing, or reviewing a `variants` object.

## Quick Reference

### cva in three lines

```ts
import { cva, type VariantProps } from "class-variance-authority"
import { cn } from "@/lib/utils"

const buttonVariants = cva("inline-flex items-center", {
  variants: { variant: { default: "bg-primary", outline: "border" } },
  defaultVariants: { variant: "default" },
})
```

Three rules apply to every cva call :

1. ALWAYS pass the result of `buttonVariants({ ... })` through `cn(...)` together with any caller `className`. NEVER return the raw `cva()` output to the DOM.
2. ALWAYS export `VariantProps<typeof buttonVariants>` (directly or via a `Props` type) so consumers can extend the component. NEVER hand-roll the variant union by typing strings.
3. ALWAYS list variant values as quoted string keys (`outline: "..."`), never raw booleans or numbers as keys ; cva treats keys as strings under the hood and TypeScript inference relies on it.

### Canonical Button pattern (verbatim from shadcn registry, Tailwind v4)

```tsx
import * as React from "react"
import { cva, type VariantProps } from "class-variance-authority"
import { Slot } from "radix-ui"
import { cn } from "@/lib/utils"

const buttonVariants = cva(
  "inline-flex shrink-0 items-center justify-center gap-2 rounded-md text-sm font-medium",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground hover:bg-primary/90",
        outline: "border bg-background hover:bg-accent",
        ghost: "hover:bg-accent hover:text-accent-foreground",
        link: "text-primary underline-offset-4 hover:underline",
      },
      size: {
        default: "h-9 px-4 py-2",
        sm: "h-8 rounded-md px-3",
        lg: "h-10 rounded-md px-6",
        icon: "size-9",
      },
    },
    defaultVariants: { variant: "default", size: "default" },
  }
)

function Button({
  className,
  variant = "default",
  size = "default",
  asChild = false,
  ...props
}: React.ComponentProps<"button"> &
  VariantProps<typeof buttonVariants> & { asChild?: boolean }) {
  const Comp = asChild ? Slot.Root : "button"
  return (
    <Comp
      data-slot="button"
      className={cn(buttonVariants({ variant, size, className }))}
      {...props}
    />
  )
}

export { Button, buttonVariants }
```

The full `references/examples.md` carries the unabbreviated registry source plus extension recipes.

## cva signature breakdown

The function lives in the `class-variance-authority` package (verified at https://cva.style/docs/api-reference, 2026-05-19) :

```ts
function cva(base: ClassValue | ClassValue[], config?: Config): VariantFn
```

| Position | Name | Type | Role |
|----------|------|------|------|
| 1 | `base` | `string` OR `string[]` OR any `clsx` value | Classes that ALWAYS apply, regardless of variant |
| 2 | `config.variants` | object : variant-key to (value-key to class string) | The variant axis enumeration |
| 2 | `config.compoundVariants` | array of `{ ...match-keys, class }` | Extra classes applied when multiple variants match simultaneously |
| 2 | `config.defaultVariants` | object : variant-key to default value-key | Fallback when caller omits the prop |

### base : string or string array

ALWAYS prefer a string array when the base list is long enough that a single line becomes unreadable :

```ts
const card = cva([
  "rounded-lg border bg-card text-card-foreground shadow-sm",
  "transition-colors",
])
```

Both forms produce identical output. NEVER concatenate with `+ " "` ; pass the array verbatim.

### variants : the variant axis enumeration

Each top-level key is one variant axis (`variant`, `size`, `tone`, ...). Each value is a map from value-name to a class string (or array of strings for clsx-style composition) :

```ts
variants: {
  tone: {
    info: "bg-blue-50 text-blue-900",
    success: ["bg-green-50", "text-green-900"],
    danger: "bg-red-50 text-red-900",
  },
}
```

ALWAYS use quoted string keys. NEVER use raw identifiers like `true:` or `1:` without quotes ; cva will accept them, but TypeScript inference via `VariantProps` then surfaces awkward union members. For boolean-shaped variants, use `"true"` and `"false"` as string keys (see `references/anti-patterns.md` AP-3).

### compoundVariants : combination-only overrides

Each entry is an object that lists one OR MORE variant-keys to match, plus a `class` string (or array) to apply when ALL listed keys match the caller's invocation :

```ts
compoundVariants: [
  { variant: "outline", size: "sm", class: "ring-1 ring-ring/20" },
  { variant: "outline", size: "lg", class: "ring-2 ring-ring/30" },
]
```

ALWAYS understand the evaluation order : `defaultVariants` resolve first (filling in missing props), THEN `variants` classes apply, THEN `compoundVariants` classes apply LAST. The last-applied class wins via `tailwind-merge` only because of position in the string ; if a compound entry needs to override a variant entry, ALWAYS rely on `cn()` (and therefore `twMerge`) to resolve the conflict. NEVER assume the compound class wins solely because it is declared in `compoundVariants` ; without `twMerge`, conflicting utilities BOTH appear in the DOM and CSS specificity decides arbitrarily.

### defaultVariants : the no-prop fallback

```ts
defaultVariants: { variant: "default", size: "default" }
```

ALWAYS provide a `defaultVariants` entry for EVERY variant axis. Consumers may omit any prop ; missing defaults produce `undefined` lookups that yield no classes for that axis. Setting a variant to `null` in `defaultVariants` explicitly opts out of any default for that axis (verified at https://cva.style/docs/api-reference, 2026-05-19).

## VariantProps inference pattern

`VariantProps` is a TypeScript helper exported by `class-variance-authority` (verified at https://cva.style/docs/getting-started/typescript). Use it to derive the variant prop union from the cva return value :

```ts
import { cva, type VariantProps } from "class-variance-authority"

const buttonVariants = cva(/* ... */)

type ButtonVariantProps = VariantProps<typeof buttonVariants>
//   ^ { variant?: "default" | "outline" | "ghost" | "link" | null | undefined,
//       size?: "default" | "sm" | "lg" | "icon" | null | undefined }
```

The canonical shadcn pattern composes `VariantProps` with native DOM props via TypeScript intersection :

```ts
type ButtonProps =
  React.ComponentProps<"button"> &
  VariantProps<typeof buttonVariants> &
  { asChild?: boolean }
```

ALWAYS export `buttonVariants` (the cva return value) AND the component. Without the variants export consumers cannot reuse the variant classes for sibling components (link styled as a button, Next.js `<Link>` carrying button visuals, etc.). NEVER duplicate the variant-class strings into a second `cva()` call ; reuse the existing one.

### Making a variant required

cva makes every variant optional by default. To force a variant to be required, use TypeScript utility types (verified pattern at https://cva.style/docs/getting-started/typescript) :

```ts
type ButtonProps =
  Omit<VariantProps<typeof buttonVariants>, "variant"> &
  Required<Pick<VariantProps<typeof buttonVariants>, "variant">>
```

## cn() helper usage rules

The `cn()` helper composes `clsx` and `tailwind-merge`. The shadcn CLI ships it at `@/lib/utils` (verified, present in every project initialised with `pnpm dlx shadcn@latest init`) :

```ts
// lib/utils.ts
import { clsx, type ClassValue } from "clsx"
import { twMerge } from "tailwind-merge"

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

Two responsibilities are stacked :

| Layer | Library | Responsibility |
|-------|---------|----------------|
| Inner | `clsx` | CONDITIONAL composition : flatten arrays, filter falsy, expand `{ "foo": cond }` objects |
| Outer | `tailwind-merge` | CONFLICT resolution : `twMerge("px-2 p-3")` returns `"p-3"`, dropping the override-loser |

### The cn() ALWAYS-rule

ALWAYS pass any composed Tailwind class string through `cn()` before assigning to `className`. This applies to :

1. The output of `buttonVariants({ variant, size, className })` (callers can pass `className` to override).
2. Any inline conditional class composition (`cn("base", active && "bg-primary")`).
3. Any prop-pass-through where the parent forwards `className` to a child.

NEVER write either of these :

```tsx
// WRONG : template-string concat. Conflicting utilities both ship to DOM, twMerge bypassed.
<button className={`bg-primary ${className}`}>

// WRONG : plus-string concat. Same problem ; also brittle on undefined.
<button className={"bg-primary " + (active ? "ring-2" : "")}>
```

ALWAYS write :

```tsx
<button className={cn("bg-primary", active && "ring-2", className)}>
```

The conflict-resolution role of `twMerge` is what makes caller `className` "win" over base classes. Skip it and your component silently fails to honour parent overrides. See `references/anti-patterns.md` AP-1 for the full damage report.

## Variant evaluation order

When `buttonVariants({ variant: "outline", size: "sm", className: "bg-red-500" })` is called, classes accumulate in this order :

1. **Base** : the first cva argument (e.g. `"inline-flex shrink-0 ..."`).
2. **defaultVariants** : applied for any axis the caller did not specify. (Here the caller specified both `variant` and `size`, so defaults are skipped.)
3. **variants[axis][value]** : for each axis the caller specified : `variant.outline` then `size.sm`.
4. **compoundVariants** : every entry whose match-keys ALL equal the caller's props.
5. **className** (when included in the call object) : the caller's literal override string.

The accumulator is then passed through `clsx` (to flatten) and `twMerge` (to resolve conflicts). The LAST conflicting utility wins ; the order of accumulation determines that last-position.

### Why ordering matters in practice

Suppose `variant.outline` sets `border-input` and a compound entry `{ variant: "outline", size: "lg" }` sets `border-primary`. Because the compound runs AFTER the variant, `twMerge` correctly drops `border-input`. If you flip the conceptual model and treat compound as "an extra hint" rather than "the final word", you will write compound classes that lose to variants and wonder why nothing changes. ALWAYS treat compound entries as final-position overrides ; NEVER expect them to be merged via specificity.

## asChild Slot pattern

Shadcn components accept an `asChild?: boolean` prop. When `true`, the component renders its child as the actual DOM element while still applying all of its own classes and props. The mechanism is Radix's `Slot.Root` (since the Feb 2026 unified `radix-ui` package ; pre-unification it was `@radix-ui/react-slot`'s `Slot`) :

```tsx
import { Slot } from "radix-ui"

function Button({ asChild = false, className, ...props }) {
  const Comp = asChild ? Slot.Root : "button"
  return <Comp className={cn(buttonVariants({ ...props, className }))} {...props} />
}
```

Usage : render a button-styled Next.js `<Link>` or a react-router `NavLink` without rendering an actual `<button>` :

```tsx
<Button asChild variant="outline">
  <Link href="/about">About</Link>
</Button>
```

### asChild rules

1. ALWAYS pass exactly ONE child to a component with `asChild`. Radix `Slot` throws on zero or multiple children.
2. ALWAYS ensure the child accepts the props the parent forwards (`className`, `onClick`, ref). A plain `<div>` accepts most ; a custom component must spread props.
3. NEVER nest `asChild` more than one level without inspecting forwarded refs ; ref-forwarding through multiple Slots requires `Slot.Root` + `Slot.Slottable` (see Radix docs).
4. NEVER use `asChild` to inject a SECOND interactive element. The combined element must still be a single accessible widget (button OR link, not both).

See `references/anti-patterns.md` AP-5 for the multi-child failure mode.

## Companion Skills

- [shadcn-core-stack](../../shadcn-core/shadcn-core-stack/SKILL.md) : where cva and tailwind-merge sit in the dependency layer cake
- [shadcn-syntax-button](../shadcn-syntax-button/SKILL.md) : button variants and sizes in full, plus `asChild` worked examples
- [shadcn-syntax-form](../shadcn-syntax-form/SKILL.md) : Form components compose `cn()` for FormControl wiring
- [shadcn-errors-styling-conflicts](../../shadcn-errors/shadcn-errors-styling-conflicts/SKILL.md) : debugging duplicate utilities, fast-refresh lint warnings on cva exports
- [shadcn-errors-radix-controlled](../../shadcn-errors/shadcn-errors-radix-controlled/SKILL.md) : `asChild` misuse and Slot single-child requirement

## Reference Links

- [references/methods.md](references/methods.md) : full `cva()` TypeScript signature, `VariantProps` definition, `cn()` source, `Slot` API surface
- [references/examples.md](references/examples.md) : minimal Badge, verbatim Button source, compound-variant example, variant extension, WRONG-vs-RIGHT className override
- [references/anti-patterns.md](references/anti-patterns.md) : six numbered anti-patterns with WHY and FIX

## Sources

All claims in this skill trace to URLs in `SOURCES.md` :

- https://cva.style/docs (cva introduction)
- https://cva.style/docs/api-reference (cva signature)
- https://cva.style/docs/getting-started/typescript (VariantProps and required-variant patterns)
- https://github.com/dcastil/tailwind-merge (twMerge semantics)
- https://ui.shadcn.com/docs/components/radix/button (canonical Button source)
- https://www.radix-ui.com/primitives (Slot primitive)

Verified 2026-05-19.
