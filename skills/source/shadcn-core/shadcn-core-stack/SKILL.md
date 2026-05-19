---
name: shadcn-core-stack
description: >
  Use when reading or writing any shadcn ui component, especially when figuring out
  which library owns which concern (a11y, variants, class composition, icons), when a
  prop or class string is not behaving as expected, when picking between raw Radix and
  the shadcn-wrapped component, when adding tailwind-merge or cva to an existing project,
  when an LLM is asked to "rebuild a shadcn-style component" and needs to know the
  exact runtime layers, and when migrating from Tailwind v3 to v4 inside a shadcn project.
  Prevents the common mistakes of (a) concatenating className strings without `cn()`,
  causing duplicate utility classes to leak through, (b) bypassing the shadcn-copied
  component to import Radix primitives directly, losing the project's styling, (c)
  defining `cva` variants in the wrong order so that `compoundVariants` never trigger,
  (d) installing wrong package versions (radix-ui unified vs @radix-ui/react-* legacy,
  cva 1.0 beta vs 0.7 stable, tailwind-merge v2 for Tailwind v3 vs v3 for Tailwind v4),
  and (e) treating shadcn ui as a runtime dependency when it is a code-generation CLI.
  Covers the complete runtime stack composition (Radix UI for headless primitives,
  class-variance-authority for type-safe variants, tailwind-merge for class conflict
  resolution, clsx for conditional composition, Tailwind CSS v3.4 or v4 for styling,
  lucide-react for icons), the canonical `cn()` helper, the cva variant signature with
  TypeScript inference via `VariantProps`, what each layer owns and where it lives in
  the consumer project, the ownership-model implications (open code, no runtime
  shadcn import), and the per-component Radix-primitive mapping for the 39 Radix-backed
  components in the catalog.
  Keywords: shadcn stack, cva, class-variance-authority, tailwind-merge, twMerge, clsx,
  cn helper, lib/utils, Radix UI primitives, radix-ui unified package, lucide-react,
  class composition, why className not applied, duplicate classes Tailwind, variant
  ordering, compoundVariants not firing, shadcn under the hood, runtime dependencies,
  ownership model, open code, how does shadcn work, what is cva, how do I merge tailwind
  classes, shadcn vs radix, why does my override not work, p-3 px-2 conflict.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui Core: Stack Composition

shadcn ui is **not a runtime component library** : it is a CLI that copies TypeScript source files into your project. Once `shadcn add button` finishes, your project owns `components/ui/button.tsx` outright, and the only things that remain at runtime are the underlying primitives (Radix, cva, tailwind-merge, clsx, lucide-react, Tailwind). Understanding this stack is the foundation for every other shadcn skill.

## Quick Reference

### Stack layers (every shadcn project has these)

| Layer : owns : package : runtime cost |
|--|
| Headless behaviour + a11y : focus-trap, ARIA, keyboard, RTL, controlled state : `radix-ui` (unified, Feb 2026) or `@radix-ui/react-*` (legacy per-component) : small (component-scoped, tree-shaken) |
| Variant API : type-safe variant + size composition : `class-variance-authority` (cva) : tiny (~1 KB gzipped) |
| Class conflict resolution : `twMerge('px-2 p-3')` returns `p-3`, dropping `px-2` : `tailwind-merge` v3.x (Tailwind v4) or v2.x (Tailwind v3) : ~6 KB gzipped, cached per call site |
| Conditional class composition : `clsx('a', cond && 'b', { c: enabled })` : `clsx` : ~250 bytes |
| Utility CSS : every visible style : `tailwindcss` v3.4 or v4.0+ : build-time |
| Icons : default icon set : `lucide-react` : tree-shaken per icon |

### Top-3 most-used patterns

```tsx
// 1. The cn() helper : single merge primitive used by every shadcn component.
import { clsx, type ClassValue } from "clsx"
import { twMerge } from "tailwind-merge"
export function cn(...inputs: ClassValue[]) { return twMerge(clsx(inputs)) }

// 2. A cva-driven Button (the canonical shadcn pattern).
import { cva, type VariantProps } from "class-variance-authority"
import { cn } from "@/lib/utils"

const buttonVariants = cva(
  "inline-flex items-center justify-center rounded-md text-sm font-medium",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground hover:bg-primary/90",
        destructive: "bg-destructive text-destructive-foreground hover:bg-destructive/90",
        outline: "border border-input bg-background hover:bg-accent hover:text-accent-foreground",
      },
      size: {
        default: "h-9 px-4 py-2",
        sm: "h-8 rounded-md px-3 text-xs",
        lg: "h-10 rounded-md px-8",
        icon: "size-9",
      },
    },
    defaultVariants: { variant: "default", size: "default" },
  }
)

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> { asChild?: boolean }

export function Button({ className, variant, size, ...props }: ButtonProps) {
  return <button className={cn(buttonVariants({ variant, size }), className)} {...props} />
}

// 3. A Radix-wrapped Dialog : shadcn-copied component owns the styling,
//    Radix owns the behaviour.
import * as DialogPrimitive from "@radix-ui/react-dialog" // or: from "radix-ui"
import { cn } from "@/lib/utils"

export const Dialog = DialogPrimitive.Root
export const DialogTrigger = DialogPrimitive.Trigger

export const DialogContent = React.forwardRef<
  React.ElementRef<typeof DialogPrimitive.Content>,
  React.ComponentPropsWithoutRef<typeof DialogPrimitive.Content>
>(({ className, ...props }, ref) => (
  <DialogPrimitive.Portal>
    <DialogPrimitive.Overlay className="fixed inset-0 z-50 bg-black/80" />
    <DialogPrimitive.Content
      ref={ref}
      className={cn("fixed left-1/2 top-1/2 -translate-x-1/2 -translate-y-1/2 ...", className)}
      {...props}
    />
  </DialogPrimitive.Portal>
))
```

## Decision Trees

### Picking which layer solves your problem

```
need to ...
├── make a component a11y, focus-trapped, keyboard-navigable, ARIA-compliant?
│   └── ALWAYS use the Radix primitive (via shadcn-wrapped component)
├── add a typed variant prop (`variant="destructive"`, `size="lg"`)?
│   └── ALWAYS use cva + VariantProps ; NEVER inline if/switch on className
├── compose className conditionally (`cn('base', isOpen && 'rotate-180')`)?
│   └── ALWAYS use cn() ; clsx handles the conditional, twMerge resolves conflicts
├── allow caller to override styles (`<Button className="rounded-none" />`)?
│   └── ALWAYS pass caller's className LAST to cn() so twMerge keeps it
├── add an icon?
│   └── ALWAYS use lucide-react (`import { Check } from "lucide-react"`) unless migrated
├── style something visible?
│   └── ALWAYS Tailwind utility classes ; NEVER inline `style={...}` for design tokens
└── render dynamic content with no Radix equivalent (Alert, Card, Badge, Breadcrumb)?
    └── pure cva + Tailwind (no Radix), still composed via cn()
```

### Picking between `radix-ui` unified vs `@radix-ui/react-*` legacy

```
new project after Feb 2026?
├── ALWAYS install the unified package : `npm install radix-ui`
└── Import per-primitive : `import { Dialog as DialogPrimitive } from "radix-ui"`

existing project on @radix-ui/react-*?
├── keep `@radix-ui/react-dialog`, `@radix-ui/react-dropdown-menu`, etc.
└── Imports unchanged : `import * as DialogPrimitive from "@radix-ui/react-dialog"`

mixed install (some unified, some per-component)?
└── NEVER mix in the same project ; pick one and migrate fully
```

### Picking tailwind-merge version

```
project Tailwind version?
├── Tailwind v3.4 (or older) : `tailwind-merge@^2.6.0`
├── Tailwind v4.0 through v4.3 : `tailwind-merge@^3.0.0`
└── mismatch produces silent merge failures on new v4 utilities (e.g. `shadow-xs`)
```

### Picking cva 0.7 stable vs 1.0 beta

```
production app, want stability?
├── `class-variance-authority@^0.7.1` (stable)

want latest variant features (slot-typed compoundVariants, recipe inheritance)?
├── `class-variance-authority@^1.0.0-beta` (beta as of May 2026)
└── beta API is mostly source-compatible ; review migration notes in cva changelog
```

## Patterns

### Pattern 1: The `cn()` helper

Every shadcn project ships `lib/utils.ts` with this exact function. NEVER replace it with raw string concatenation.

```ts
// lib/utils.ts
import { clsx, type ClassValue } from "clsx"
import { twMerge } from "tailwind-merge"

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

What each call does :

```ts
// clsx handles conditionals + falsy values + objects
clsx("base", isOpen && "rotate-180", { active: enabled })
// => "base rotate-180 active"

// twMerge handles Tailwind utility conflicts
twMerge("px-2 py-1 bg-red-500", "p-4 bg-blue-500")
// => "p-4 bg-blue-500"

// cn combines both
cn("px-2 py-1 bg-red-500", isPrimary && "p-4 bg-blue-500", className)
// when isPrimary === true and caller passes className="rounded-full" :
// => "p-4 bg-blue-500 rounded-full"
```

### Pattern 2: cva variant signature with TypeScript inference

```ts
import { cva, type VariantProps } from "class-variance-authority"

const buttonVariants = cva(
  // base classes : ALWAYS apply
  "inline-flex items-center justify-center rounded-md text-sm font-medium",
  {
    // variants : one prop per variant family
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground hover:bg-primary/90",
        destructive: "bg-destructive text-destructive-foreground hover:bg-destructive/90",
        outline: "border border-input bg-background",
        ghost: "hover:bg-accent hover:text-accent-foreground",
      },
      size: {
        default: "h-9 px-4 py-2",
        sm: "h-8 px-3 text-xs",
        lg: "h-10 px-8",
        icon: "size-9",
      },
    },
    // compoundVariants : combinations that need extra classes
    compoundVariants: [
      { variant: "outline", size: "sm", class: "border-2" },
    ],
    // defaultVariants : applied when prop is `undefined` (NOT empty string)
    defaultVariants: { variant: "default", size: "default" },
  }
)

// VariantProps extracts the literal-union prop types from cva()
type ButtonVariantProps = VariantProps<typeof buttonVariants>
// => { variant?: "default" | "destructive" | "outline" | "ghost" | null,
//      size?: "default" | "sm" | "lg" | "icon" | null }

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    ButtonVariantProps { asChild?: boolean }
```

ALWAYS call `cva` once at module scope (not inside the component), so the function and types are constant.

### Pattern 3: Caller override semantics

```tsx
// Component definition : caller's className passed LAST to cn()
export function Button({ className, variant, size, ...props }: ButtonProps) {
  return (
    <button
      className={cn(buttonVariants({ variant, size }), className)}
      {...props}
    />
  )
}

// Usage : caller wins on conflicting utilities
<Button variant="default" className="rounded-none px-12" />
// => "inline-flex items-center justify-center text-sm font-medium
//     bg-primary text-primary-foreground hover:bg-primary/90 h-9 py-2
//     rounded-none px-12"
// (rounded-md and px-4 from variants are dropped by twMerge)
```

If `className` came FIRST inside `cn()`, the variant classes would override the caller : exactly the opposite of the intended ergonomics.

### Pattern 4: Radix primitive wrapping (shadcn convention)

```tsx
// components/ui/dialog.tsx (shadcn-copied)
"use client"

import * as React from "react"
import * as DialogPrimitive from "@radix-ui/react-dialog" // legacy
// or: import { Dialog as DialogPrimitive } from "radix-ui" // unified
import { X } from "lucide-react"
import { cn } from "@/lib/utils"

const Dialog = DialogPrimitive.Root
const DialogTrigger = DialogPrimitive.Trigger
const DialogPortal = DialogPrimitive.Portal
const DialogClose = DialogPrimitive.Close

const DialogOverlay = React.forwardRef<
  React.ElementRef<typeof DialogPrimitive.Overlay>,
  React.ComponentPropsWithoutRef<typeof DialogPrimitive.Overlay>
>(({ className, ...props }, ref) => (
  <DialogPrimitive.Overlay
    ref={ref}
    className={cn("fixed inset-0 z-50 bg-black/80", className)}
    {...props}
  />
))
DialogOverlay.displayName = DialogPrimitive.Overlay.displayName

export { Dialog, DialogTrigger, DialogPortal, DialogClose, DialogOverlay /* etc */ }
```

ALWAYS import the **shadcn-wrapped** component from `@/components/ui/dialog`, NOT the Radix primitive directly. Importing Radix bypasses all of the project's styling.

### Pattern 5: Lucide icon usage inside shadcn components

```tsx
import { Check, ChevronRight, Loader2 } from "lucide-react"
import { cn } from "@/lib/utils"

export function CheckIcon({ className }: { className?: string }) {
  return <Check className={cn("size-4", className)} />
}
```

Tree-shaken : only the icons you actually import end up in the bundle. The default size used by shadcn components is `size-4` (16px). Animated states use the spinning `Loader2` icon with `animate-spin`.

## Reference Links

- `references/methods.md` : complete API signatures (cva, VariantProps, twMerge, clsx, cn, Radix primitive list with import paths).
- `references/examples.md` : working examples (Button with cva, Dialog wrapping Radix, cn merge cases, lucide icon imports).
- `references/anti-patterns.md` : common mistakes (raw className concatenation, missing twMerge, bypassing shadcn-wrapped components, wrong cva variant ordering, mixing radix-ui unified with @radix-ui/react-*).

## Verified Sources

- https://ui.shadcn.com/docs (open-code + composition + ownership model)
- https://www.radix-ui.com/primitives (Radix primitives, unified `radix-ui` package since Feb 2026)
- https://www.radix-ui.com/primitives/docs/overview/getting-started (`npm install radix-ui`, `import { Popover } from "radix-ui"`)
- https://cva.style/docs (cva signature, current 1.0 beta)
- https://github.com/dcastil/tailwind-merge (twMerge API, v3.x for Tailwind v4, v2.x for Tailwind v3)
- https://github.com/shadcn-ui/ui (shadcn source code for canonical patterns)

Last verified : 2026-05-19 (shadcn evergreen-2026 / canary).
