# Examples : cva, VariantProps, cn, Slot

Every example below is annotated with the Tailwind generation it targets. Pre-v4 examples use `hsl(var(--token))` ; v4 examples use semantic utilities like `bg-primary` directly.

## Example 1 : Minimal Badge with three variants (Tailwind v4)

```tsx
// components/ui/badge.tsx
import { cva, type VariantProps } from "class-variance-authority"
import { cn } from "@/lib/utils"

const badgeVariants = cva(
  "inline-flex items-center rounded-md border px-2.5 py-0.5 text-xs font-semibold",
  {
    variants: {
      variant: {
        default: "border-transparent bg-primary text-primary-foreground",
        secondary: "border-transparent bg-secondary text-secondary-foreground",
        outline: "text-foreground",
      },
    },
    defaultVariants: { variant: "default" },
  }
)

type BadgeProps =
  React.HTMLAttributes<HTMLSpanElement> &
  VariantProps<typeof badgeVariants>

function Badge({ className, variant, ...props }: BadgeProps) {
  return (
    <span className={cn(badgeVariants({ variant, className }))} {...props} />
  )
}

export { Badge, badgeVariants }
```

## Example 2 : Full Button with size + variant + asChild (verbatim from shadcn registry, Tailwind v4)

```tsx
// components/ui/button.tsx
import * as React from "react"
import { cva, type VariantProps } from "class-variance-authority"
import { Slot } from "radix-ui"
import { cn } from "@/lib/utils"

const buttonVariants = cva(
  "inline-flex shrink-0 items-center justify-center gap-2 rounded-md text-sm font-medium whitespace-nowrap transition-all outline-none focus-visible:border-ring focus-visible:ring-[3px] focus-visible:ring-ring/50 disabled:pointer-events-none disabled:opacity-50 aria-invalid:border-destructive aria-invalid:ring-destructive/20 dark:aria-invalid:ring-destructive/40 [&_svg]:pointer-events-none [&_svg]:shrink-0 [&_svg:not([class*='size-'])]:size-4",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground hover:bg-primary/90",
        destructive:
          "bg-destructive text-white hover:bg-destructive/90 focus-visible:ring-destructive/20 dark:bg-destructive/60 dark:focus-visible:ring-destructive/40",
        outline:
          "border bg-background shadow-xs hover:bg-accent hover:text-accent-foreground dark:border-input dark:bg-input/30 dark:hover:bg-input/50",
        secondary:
          "bg-secondary text-secondary-foreground hover:bg-secondary/80",
        ghost:
          "hover:bg-accent hover:text-accent-foreground dark:hover:bg-accent/50",
        link: "text-primary underline-offset-4 hover:underline",
      },
      size: {
        default: "h-9 px-4 py-2 has-[>svg]:px-3",
        xs: "h-6 gap-1 rounded-md px-2 text-xs has-[>svg]:px-1.5 [&_svg:not([class*='size-'])]:size-3",
        sm: "h-8 gap-1.5 rounded-md px-3 has-[>svg]:px-2.5",
        lg: "h-10 rounded-md px-6 has-[>svg]:px-4",
        icon: "size-9",
        "icon-xs": "size-6 rounded-md [&_svg:not([class*='size-'])]:size-3",
        "icon-sm": "size-8",
        "icon-lg": "size-10",
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
  VariantProps<typeof buttonVariants> & {
    asChild?: boolean
  }) {
  const Comp = asChild ? Slot.Root : "button"
  return (
    <Comp
      data-slot="button"
      data-variant={variant}
      data-size={size}
      className={cn(buttonVariants({ variant, size, className }))}
      {...props}
    />
  )
}

export { Button, buttonVariants }
```

Source : https://github.com/shadcn-ui/ui/blob/main/apps/v4/registry/new-york-v4/ui/button.tsx (verified 2026-05-19).

### Same Button under Tailwind v3 (pre Feb-2025)

Pre-v4 projects do NOT use `data-slot`, the `size-*` utility is unavailable, and color tokens are referenced through `hsl(var(...))` in `tailwind.config.js`. The cva structure is identical ; only the class strings differ :

```tsx
// Tailwind v3 variant snippet (illustrative, replaces the v4 classes above)
variants: {
  variant: {
    default: "bg-primary text-primary-foreground hover:bg-primary/90",
    outline: "border border-input bg-background hover:bg-accent",
  },
  size: {
    default: "h-10 px-4 py-2",
    sm: "h-9 rounded-md px-3",
    lg: "h-11 rounded-md px-8",
    icon: "h-10 w-10",
  },
},
```

In v3 the helper utilities `bg-primary` and `bg-background` work because `tailwind.config.js` extends `theme.colors` with `primary: "hsl(var(--primary) / <alpha-value>)"`. Without that extension the same classes fail silently. ALWAYS verify the Tailwind generation before copying snippets.

## Example 3 : compoundVariants (Tailwind v4)

A compound rule adds an extra ring when the outline variant meets the small size :

```tsx
const buttonVariants = cva(
  "inline-flex items-center justify-center rounded-md text-sm font-medium",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground",
        outline: "border bg-background",
      },
      size: {
        default: "h-9 px-4",
        sm: "h-8 px-3",
        lg: "h-10 px-6",
      },
    },
    compoundVariants: [
      { variant: "outline", size: "sm", class: "ring-1 ring-ring/20" },
      { variant: "outline", size: "lg", class: "ring-2 ring-ring/30" },
    ],
    defaultVariants: { variant: "default", size: "default" },
  }
)
```

Invocations :

```ts
buttonVariants({ variant: "outline", size: "sm" })
// base + "border bg-background" + "h-8 px-3" + "ring-1 ring-ring/20"

buttonVariants({ variant: "outline", size: "default" })
// base + "border bg-background" + "h-9 px-4"   (no compound matches)

buttonVariants({ variant: "default", size: "sm" })
// base + "bg-primary text-primary-foreground" + "h-8 px-3"   (no compound matches)
```

Compound entries can list MORE than two variant axes ; all listed axes must match for the entry to apply.

## Example 4 : Extending an existing component's variants (add `warning` to Button)

Because `buttonVariants` lives in the consumer project (Open Code pillar), extending it is a direct edit. Tailwind v4 :

```tsx
// components/ui/button.tsx
const buttonVariants = cva(
  "inline-flex items-center justify-center rounded-md text-sm font-medium",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground hover:bg-primary/90",
        destructive: "bg-destructive text-white hover:bg-destructive/90",
        outline: "border bg-background hover:bg-accent",
        // NEW :
        warning:
          "bg-yellow-500 text-yellow-50 hover:bg-yellow-500/90 focus-visible:ring-yellow-500/30",
      },
      size: { default: "h-9 px-4", sm: "h-8 px-3", lg: "h-10 px-6" },
    },
    defaultVariants: { variant: "default", size: "default" },
  }
)
```

No additional plumbing is required. `VariantProps<typeof buttonVariants>` automatically inferrers `variant?: "default" | "destructive" | "outline" | "warning"`. Consumers can now write `<Button variant="warning">Save</Button>`.

For project-level design tokens, prefer adding a `--warning` CSS variable in `globals.css` and referencing `bg-warning text-warning-foreground` (v4 `@theme inline` semantics). This keeps dark-mode wiring consistent.

## Example 5 : VariantProps inference + extends button DOM props

```tsx
import * as React from "react"
import { cva, type VariantProps } from "class-variance-authority"
import { cn } from "@/lib/utils"

const buttonVariants = cva("inline-flex items-center", {
  variants: {
    variant: { default: "bg-primary", outline: "border" },
    size: { default: "h-9", sm: "h-8" },
  },
  defaultVariants: { variant: "default", size: "default" },
})

// Compose with native DOM props :
type ButtonProps =
  React.ComponentProps<"button"> &
  VariantProps<typeof buttonVariants> &
  { asChild?: boolean }

// Consumer usage gets full IntelliSense :
function Demo() {
  return (
    <>
      <Button variant="outline" size="sm" onClick={() => {}}>Click</Button>
      {/* @ts-expect-error : "danger" is not in the variant union */}
      <Button variant="danger">No</Button>
    </>
  )
}

function Button(props: ButtonProps) { /* ... */ return null }
```

The `@ts-expect-error` comment proves the variant union is type-checked. Without `VariantProps` the consumer would have to remember the strings by hand and TypeScript could not catch typos.

## Example 6 : WRONG vs RIGHT className override

A common consumer pattern : a parent wants to push a different background colour onto a Button.

### WRONG : template-string concat

```tsx
// components/some-page.tsx
<Button className={`bg-red-500 ${someCondition ? "ring-2" : ""}`}>
  Delete
</Button>
```

Why this fails (Tailwind v4 baseline) :
- `buttonVariants` already sets `bg-primary` in the base + variant. The button DOM ends up with BOTH `bg-primary` and `bg-red-500`. CSS specificity decides which wins, based on source order in the generated stylesheet ; this is brittle.
- If `someCondition` is `false`, the string ends with a trailing space, which is harmless but indicates the pattern is unaware of clsx-style conditionals.
- A future reorder of utility classes in the base will silently break the override.

### RIGHT : let cn() + twMerge resolve the conflict

```tsx
import { cn } from "@/lib/utils"

<Button className={cn("bg-red-500", someCondition && "ring-2")}>
  Delete
</Button>
```

What happens inside the Button :

```tsx
function Button({ className, ...props }) {
  return <button className={cn(buttonVariants({ ...props, className }))} />
}
```

`buttonVariants({ ..., className: "bg-red-500 ring-2" })` appends the caller string to the end of the variant output. The whole accumulated string is then passed through `cn()` (and therefore `twMerge`), which detects the `bg-primary` vs `bg-red-500` conflict and drops `bg-primary`. Result : the DOM gets exactly one background utility, the caller's. The component owner did not have to special-case anything.

This pattern is what makes shadcn components composable. Skip `cn()` and the entire override chain collapses.

## Example 7 : asChild Slot with Next.js Link

```tsx
import Link from "next/link"
import { Button } from "@/components/ui/button"

export function Header() {
  return (
    <Button asChild variant="outline" size="sm">
      <Link href="/about">About</Link>
    </Button>
  )
}
```

What the DOM renders : a single `<a href="/about">` element that carries every Button class (`inline-flex ...`, `border bg-background hover:bg-accent`, `h-8 ...`). No `<button>` is rendered. `next/link` receives the merged `className` and forwards it to the underlying anchor.

Common pitfalls in `references/anti-patterns.md` AP-5.
