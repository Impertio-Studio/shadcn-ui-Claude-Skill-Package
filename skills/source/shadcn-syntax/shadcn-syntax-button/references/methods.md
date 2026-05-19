# Button : Methods Reference

Verbatim API surface for the shadcn ui Button component. Sources : `apps/v4/registry/new-york-v4/ui/button.tsx` (Tailwind v4, current canonical) and `apps/v4/public/r/styles/default/button.json` (Tailwind v3 legacy default style). Both fetched from `https://github.com/shadcn-ui/ui` on 2026-05-19.

## 1. Button component signature (v4, current canonical)

```tsx
import * as React from "react"
import { cva, type VariantProps } from "class-variance-authority"
import { Slot } from "radix-ui"

import { cn } from "@/lib/utils"

const buttonVariants = cva(/* ... see Section 2 ... */)

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

Key shape facts :

- Plain function component (no `React.forwardRef`). Refs forward implicitly under React 19.
- Props type is `React.ComponentProps<"button">` intersected with `VariantProps<typeof buttonVariants>` and `{ asChild?: boolean }`.
- `defaultVariants` from cva are doubled with explicit JS defaults (`variant = "default"`, `size = "default"`) so both the className composition and the data-attribute output reflect those defaults even when the caller passes the props as `undefined`.
- Emits three stable selectors for downstream styling and testing : `data-slot="button"`, `data-variant=<variant>`, `data-size=<size>`.
- `Slot` is imported from the unified `radix-ui` package (Feb 2026 consolidation). The single Slot export is the `Slot.Root` namespace member.

## 2. buttonVariants cva instance (v4, verbatim)

```tsx
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
    defaultVariants: {
      variant: "default",
      size: "default",
    },
  }
)
```

### Base layer responsibilities (the first cva argument, line-by-line)

| Utility | Job |
|---------|-----|
| `inline-flex shrink-0 items-center justify-center gap-2` | Horizontal layout : icon + text gap of 8px ; shrinks-to-content but does not collapse |
| `rounded-md` | 6px corner radius |
| `text-sm font-medium whitespace-nowrap` | Typography : 14px font, 500 weight, no wrap |
| `transition-all` | All animatable properties transition (background, border, shadow) |
| `outline-none focus-visible:border-ring focus-visible:ring-[3px] focus-visible:ring-ring/50` | v4 focus ring : 3px ring at 50% ring-token opacity, plus a border-colour shift |
| `disabled:pointer-events-none disabled:opacity-50` | Disabled state : no hover, 50% opacity |
| `aria-invalid:border-destructive aria-invalid:ring-destructive/20 dark:aria-invalid:ring-destructive/40` | Invalid state shifts border and ring colour |
| `[&_svg]:pointer-events-none [&_svg]:shrink-0` | Child SVGs never block pointer events ; never shrink |
| `[&_svg:not([class*='size-'])]:size-4` | Default icon size 16px (size-4) UNLESS the icon child already declares its own `size-*` class |

ALWAYS keep this base layer intact when extending buttonVariants. NEVER remove `[&_svg:not([class*='size-'])]:size-4` ; the entire icon-button + leading-icon pattern depends on it.

## 3. Legacy v3 Button signature (registry "default" style)

```tsx
import * as React from "react"
import { Slot } from "@radix-ui/react-slot"
import { cva, type VariantProps } from "class-variance-authority"

import { cn } from "@/lib/utils"

const buttonVariants = cva(
  "inline-flex items-center justify-center gap-2 whitespace-nowrap rounded-md text-sm font-medium ring-offset-background transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:pointer-events-none disabled:opacity-50 [&_svg]:pointer-events-none [&_svg]:size-4 [&_svg]:shrink-0",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground hover:bg-primary/90",
        destructive:
          "bg-destructive text-destructive-foreground hover:bg-destructive/90",
        outline:
          "border border-input bg-background hover:bg-accent hover:text-accent-foreground",
        secondary:
          "bg-secondary text-secondary-foreground hover:bg-secondary/80",
        ghost: "hover:bg-accent hover:text-accent-foreground",
        link: "text-primary underline-offset-4 hover:underline",
      },
      size: {
        default: "h-10 px-4 py-2",
        sm: "h-9 rounded-md px-3",
        lg: "h-11 rounded-md px-8",
        icon: "h-10 w-10",
      },
    },
    defaultVariants: {
      variant: "default",
      size: "default",
    },
  }
)

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  asChild?: boolean
}

const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, asChild = false, ...props }, ref) => {
    const Comp = asChild ? Slot : "button"
    return (
      <Comp
        className={cn(buttonVariants({ variant, size, className }))}
        ref={ref}
        {...props}
      />
    )
  }
)
Button.displayName = "Button"

export { Button, buttonVariants }
```

Differences from v4 :

| Concern | v3 | v4 |
|---------|----|-----|
| Slot import | `import { Slot } from "@radix-ui/react-slot"` | `import { Slot } from "radix-ui"` |
| Component shape | `React.forwardRef<HTMLButtonElement, ButtonProps>` + `displayName` | Plain function ; relies on React 19 implicit ref forward |
| Props type | exported `interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement>` | inline `React.ComponentProps<"button">` intersection |
| Focus ring | `focus-visible:ring-2 ring-ring ring-offset-2` + named `ring-offset-background` | `focus-visible:ring-[3px] ring-ring/50` + `focus-visible:border-ring` |
| Default size | `h-10 px-4 py-2` (40px) | `h-9 px-4 py-2 has-[>svg]:px-3` (36px) |
| Size keys | `default / sm / lg / icon` | `default / xs / sm / lg / icon / icon-xs / icon-sm / icon-lg` |
| `data-*` attributes | absent | `data-slot`, `data-variant`, `data-size` emitted |
| Icon size selector | `[&_svg]:size-4` (always) | `[&_svg:not([class*='size-'])]:size-4` (override-aware) |

## 4. VariantProps<typeof buttonVariants>

cva exports `VariantProps` as a TypeScript helper. When applied to a cva instance, it extracts the union types of each variant key :

```ts
import { type VariantProps } from "class-variance-authority"

type ButtonVariantProps = VariantProps<typeof buttonVariants>
// => {
//   variant?: "default" | "destructive" | "outline" | "secondary" | "ghost" | "link"
//   size?: "default" | "xs" | "sm" | "lg" | "icon" | "icon-xs" | "icon-sm" | "icon-lg"
// }
```

Each variant key is OPTIONAL on the inferred type because the cva instance has `defaultVariants`. Without `defaultVariants`, the keys would be required.

ALWAYS prefer `VariantProps<typeof buttonVariants>` over hand-written union literals. If the cva definition is extended with a new variant, the type updates automatically.

## 5. Full prop enumeration

| Prop | Type | Default | Source |
|------|------|---------|--------|
| `variant` | `"default" \| "destructive" \| "outline" \| "secondary" \| "ghost" \| "link"` | `"default"` | cva variants |
| `size` (v4) | `"default" \| "xs" \| "sm" \| "lg" \| "icon" \| "icon-xs" \| "icon-sm" \| "icon-lg"` | `"default"` | cva variants |
| `size` (v3) | `"default" \| "sm" \| "lg" \| "icon"` | `"default"` | cva variants |
| `asChild` | `boolean` | `false` | Button component |
| `className` | `string` (passed through `cn()` + `twMerge`) | `undefined` | React.ComponentProps<"button"> |
| `disabled` | `boolean` | `false` | React.ComponentProps<"button"> |
| `type` | `"button" \| "submit" \| "reset"` | `"submit"` inside `<form>`, `"button"` otherwise | React.ComponentProps<"button"> |
| `onClick`, `onFocus`, ... | standard handlers | `undefined` | React.ComponentProps<"button"> |
| `aria-label`, `aria-pressed`, ... | standard ARIA | `undefined` | React.ComponentProps<"button"> |

ALWAYS set explicit `type="button"` when the button is INSIDE a `<form>` but is NOT the submit action. Browsers default `<button>` inside a form to `type="submit"`.

## 6. Slot.Root resolution

`asChild` toggles between two rendered components :

```tsx
const Comp = asChild ? Slot.Root : "button"
```

When `asChild` is true :

1. Slot.Root receives the merged props : `className`, `data-*`, event handlers, ref, ARIA props.
2. Slot.Root expects EXACTLY ONE child React element. It calls `React.Children.only(children)` internally ; multiple children throw.
3. Slot.Root clones that single child and merges the merged props onto it. Event handlers are concatenated (slot handler runs BEFORE child handler unless `event.defaultPrevented`).
4. The child element MUST accept ref-forwarding for the ref to attach. Native intrinsics (`a`, `button`, `div`) accept refs natively. Custom components must use `React.forwardRef` (v3-era components) or be a function component under React 19 (refs forward implicitly).

For the Slottable multi-child case, see `https://www.radix-ui.com/primitives/docs/utilities/slot` and the Slottable pattern in `shadcn-errors-radix-controlled`.

## 7. Class composition order (twMerge resolution)

The composition is :

```ts
className={cn(buttonVariants({ variant, size, className }))}
```

cva concatenates the strings in this exact order :
1. The base layer (first cva argument)
2. The variant value
3. The size value
4. Compound variants (none in the default Button)
5. The caller-supplied `className` (passed INTO the cva call so it lands LAST)

Then `cn()` (= `twMerge(clsx(...))`) :
- `clsx` flattens conditionals and arrays into a single string.
- `twMerge` deduplicates conflicting Tailwind utilities, keeping the LAST occurrence.

This means a caller-supplied `className="bg-red-500"` WINS over the variant's `bg-primary`. ALWAYS rely on this. NEVER work around it by re-importing or string-concatenating.

## 8. Exports

```ts
export { Button, buttonVariants }
```

`buttonVariants` is exported deliberately so that consumers can compose the styles onto non-Button elements without using `asChild`. Example :

```tsx
import { buttonVariants } from "@/components/ui/button"
import Link from "next/link"

<Link href="/login" className={buttonVariants({ variant: "outline" })}>
  Login
</Link>
```

This is an alternative to `<Button asChild><Link/></Button>` when ref-forwarding is unnecessary and the caller wants explicit class control. ALWAYS prefer `asChild` for the common case ; reach for direct `buttonVariants()` only when you need the classes without the data-attributes or the ref-merge.

## Sources verified 2026-05-19

- `https://github.com/shadcn-ui/ui` (apps/v4/registry/new-york-v4/ui/button.tsx)
- `https://github.com/shadcn-ui/ui` (apps/v4/public/r/styles/default/button.json)
- `https://ui.shadcn.com/docs/components/radix/button`
- `https://cva.style/docs/api-reference`
- `https://www.radix-ui.com/primitives/docs/utilities/slot`
