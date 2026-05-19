# Examples : WRONG, RIGHT, and edge cases

This file gives one fully WRONG cva component (with multiple violations and the verdict block per failure), one fully RIGHT cva component verified against all 7 checks, and three edge cases that the validator MUST recognise as PASS even though they look unusual at first glance.

## WRONG : Button with four violations

The file below is a real-shape `components/ui/button.tsx` an AI assistant produced when asked to "make a Button with size and variant". It compiles and runs, but the validator catches four distinct violations.

```tsx
// components/ui/button.tsx
import * as React from "react"
import { cva } from "class-variance-authority"      // (Violation A : VariantProps not imported)
import { cn } from "@/lib/utils"

const buttonVariants = cva(
  cn("inline-flex", "items-center"),                 // (Violation B : cn nested inside cva base)
  {
    compoundVariants: [                              // (Violation C : compoundVariants declared before variants)
      { variant: "outline", size: "sm", class: "ring-1 ring-ring/20" },
    ],
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground",
        outline: "border bg-background",
      },
      size: {
        default: "h-9 px-4 py-2",
        sm: "h-8 px-3",
      },
    },
    defaultVariants: { varient: "default", size: "default" },  // (Violation D : axis-name typo "varient")
  }
)

type ButtonProps = React.ComponentProps<"button"> & {
  variant?: "default" | "outline"      // (related to Violation A : hand-rolled union, VariantProps not used)
  size?: "default" | "sm"
}

function Button({ className, variant, size, ...props }: ButtonProps) {
  return (
    <button
      className={`${buttonVariants({ variant, size })} ${className}`}   // (Violation E : template-string concat)
      {...props}
    />
  )
}

export { Button }                       // (Violation F : buttonVariants not exported)
```

### Verdict block (one per violation, ordered by check number)

```
VERDICT: FAIL
COMPONENT: components/ui/button.tsx:5
FAILED CHECK: Check 1 : Base classes shape
REASON: Rule R1.2 forbids nesting cn() inside the cva base argument. cva already runs clsx internally.
CANONICAL FIX: references/examples.md § RIGHT-Button base block
```

```
VERDICT: FAIL
COMPONENT: components/ui/button.tsx:7
FAILED CHECK: Check 3 : compoundVariants position
REASON: Rule R3.1 requires compoundVariants to appear AFTER variants. Here compoundVariants is declared first.
CANONICAL FIX: references/examples.md § RIGHT-Button config block
```

```
VERDICT: FAIL
COMPONENT: components/ui/button.tsx:21
FAILED CHECK: Check 4 : defaultVariants completeness
REASON: Rule R4.3 forbids axis-name typos in defaultVariants. "varient" is not a declared axis ; cva silently no-ops the default.
CANONICAL FIX: references/examples.md § RIGHT-Button defaultVariants
```

```
VERDICT: FAIL
COMPONENT: components/ui/button.tsx:24
FAILED CHECK: Check 5 : VariantProps export
REASON: Rule R5.3 forbids hand-rolling the variant union. Use VariantProps<typeof buttonVariants> instead.
CANONICAL FIX: references/examples.md § RIGHT-Button type block
```

```
VERDICT: FAIL
COMPONENT: components/ui/button.tsx:31
FAILED CHECK: Check 6 : cn() merge
REASON: Rule R6.3 forbids template-string concat. twMerge is bypassed and duplicate utilities leak to the DOM.
CANONICAL FIX: references/examples.md § RIGHT-Button render block
```

```
VERDICT: FAIL
COMPONENT: components/ui/button.tsx:36
FAILED CHECK: Check 5 : VariantProps export
REASON: Rule R5.1 requires the cva return value to appear in an export. Only Button is exported here.
CANONICAL FIX: references/examples.md § RIGHT-Button export block
```

The WRONG file has SIX failures across FIVE distinct checks (Check 5 fires twice : once for the hand-rolled union, once for the missing export). The validator emits one block per failure ordered by check number.

## RIGHT : Button verified against all 7 checks

This is the canonical evergreen-2026 shape. Every check passes.

```tsx
// components/ui/button.tsx
import * as React from "react"
import { cva, type VariantProps } from "class-variance-authority"
import { Slot } from "radix-ui"
import { cn } from "@/lib/utils"

const buttonVariants = cva(
  // RIGHT-Button base block : string literal (or array), no nested helpers
  "inline-flex shrink-0 items-center justify-center gap-2 rounded-md text-sm font-medium",
  // RIGHT-Button config block : variants, then compoundVariants, then defaultVariants
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground hover:bg-primary/90",
        destructive: "bg-destructive text-destructive-foreground hover:bg-destructive/90",
        outline: "border bg-background hover:bg-accent hover:text-accent-foreground",
        secondary: "bg-secondary text-secondary-foreground hover:bg-secondary/80",
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
    compoundVariants: [
      { variant: "outline", size: "sm", class: "ring-1 ring-ring/20" },
      { variant: "outline", size: "lg", class: "ring-2 ring-ring/30" },
    ],
    // RIGHT-Button defaultVariants : every axis covered, names spelled exactly as in variants
    defaultVariants: { variant: "default", size: "default" },
  }
)

// RIGHT-Button type block : VariantProps inference, no hand-rolled union
type ButtonProps =
  React.ComponentProps<"button"> &
  VariantProps<typeof buttonVariants> &
  { asChild?: boolean }

// RIGHT-Button render block : Slot branch + cn(variants({ ..., className }))
function Button({
  className,
  variant,
  size,
  asChild = false,
  ...props
}: ButtonProps) {
  const Comp = asChild ? Slot.Root : "button"
  return (
    <Comp
      data-slot="button"
      className={cn(buttonVariants({ variant, size, className }))}
      {...props}
    />
  )
}

// RIGHT-Button export block : Button AND buttonVariants
export { Button, buttonVariants }
```

### Verdict block

```
VERDICT: PASS
COMPONENT: components/ui/button.tsx:7
SUMMARY: All 7 checks passed.
```

## Edge case 1 : variant value as a class-array

Cva accepts a string array as a variant value (it threads through clsx). The validator MUST recognise this shape as PASS for Rule R2.2 :

```ts
const cardVariants = cva("rounded-lg border bg-card text-card-foreground", {
  variants: {
    elevation: {
      none: "shadow-none",
      sm: ["shadow-sm", "ring-1", "ring-border/40"],     // ← array, PASS
      md: ["shadow-md", "ring-1", "ring-border/40"],
      lg: ["shadow-lg", "ring-2", "ring-border/60"],
    },
  },
  defaultVariants: { elevation: "none" },
})
```

The validator MUST NOT treat array-valued variants as suspicious. The array form is identical in output to the space-joined string form ; it is encouraged when the list is long enough that a single-line string becomes unreadable.

## Edge case 2 : compoundVariants with multiple match-keys and an array class

Compound entries may carry two or more match-keys and an array `class` value :

```ts
compoundVariants: [
  {
    variant: "outline",
    size: "lg",
    tone: "danger",
    class: ["border-destructive", "text-destructive", "hover:bg-destructive/10"],
  },
]
```

The validator MUST cross-check that `variant.outline`, `size.lg`, and `tone.danger` all exist in `variants`. The array `class` is valid per R3.2.

The validator MUST NOT count multi-key entries as more or less suspicious than single-key entries ; multi-key is the documented way to express AND semantics. Splitting one multi-key entry into multiple single-key entries would silently change semantics from AND to OR ; the validator MUST NOT propose that refactor as a fix.

## Edge case 3 : extending an existing component via variant spread

A consumer extending the shadcn Button with a brand-specific variant uses the spread pattern on the variants object :

```ts
// app/lib/brand-button.ts
import { cva, type VariantProps } from "class-variance-authority"
import { buttonVariants as baseButtonVariants } from "@/components/ui/button"

// Reuse the base by re-cva-ing with extended variants
const brandButtonVariants = cva(
  "inline-flex shrink-0 items-center justify-center gap-2 rounded-md text-sm font-medium",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground hover:bg-primary/90",
        outline: "border bg-background hover:bg-accent",
        brand: "bg-brand text-brand-foreground hover:bg-brand/90",       // new variant
        brandOutline: "border-brand text-brand hover:bg-brand/10",       // new variant
      },
      size: {
        default: "h-9 px-4 py-2",
        sm: "h-8 px-3",
        lg: "h-10 px-6",
      },
    },
    defaultVariants: { variant: "default", size: "default" },
  }
)

type BrandButtonProps =
  React.ComponentProps<"button"> &
  VariantProps<typeof brandButtonVariants>

export { brandButtonVariants }
export type { BrandButtonProps }
```

The validator MUST treat this as PASS even though it re-declares the base classes and the standard variant axes. Re-cva-ing is the documented extension pattern (cva does not currently expose a `extend()` API). The alternative (importing the original `cva()` output and calling it as a function for the new variant set) does NOT exist : the cva return value is a class-string-producing function, not a config object.

The validator MAY flag a SOFT warning when base classes are duplicated verbatim between the original and the extended variants (drift risk), but the verdict MUST remain PASS on the 7-point checklist. The soft warning is informational, not a failure.

## What the validator does NOT cover

The validator catches structural issues in the cva source. It does NOT diagnose :

- Visual override failures where all 7 checks pass but the consumer reports the override does not win. Read `shadcn-errors-styling-conflicts` (B11) for the tailwind-merge debug.
- Tailwind v3-vs-v4 class-rename surprises. Read `shadcn-impl-tailwind-v3-v4-migration` for the migration table.
- Fast-Refresh ESLint warnings on files that export both a component and a cva object. Read `shadcn-errors-fast-refresh-lint` for the workaround.
- Wrong component selection (Dialog vs Drawer vs Sheet). Read `shadcn-agents-component-selector`.

The validator stays narrowly scoped to cva structural correctness. Mixing diagnostics across skills is anti-pattern.
