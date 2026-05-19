# shadcn ui Stack: Working Examples

Each example is verified against the listed primary source URL (2026-05-19).

## Example 1: The `cn()` helper (lib/utils.ts)

```ts
// lib/utils.ts
import { clsx, type ClassValue } from "clsx"
import { twMerge } from "tailwind-merge"

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

This file is created by `shadcn init` and is the single merge primitive for the entire project. NEVER hand-roll a replacement.

### `cn()` resolution showcase

```ts
import { cn } from "@/lib/utils"

cn("p-2", "p-4")
// => "p-4"  (twMerge drops earlier conflicting padding)

cn("bg-red-500", isHover && "bg-blue-500")
// when isHover === true => "bg-blue-500"
// when isHover === false => "bg-red-500"

cn({ "text-red-500": isError, "text-green-500": !isError })
// => "text-red-500" or "text-green-500" depending on isError

cn("base", undefined, null, false, 0, "")
// => "base"  (clsx drops falsy values cleanly)

cn("rounded-md px-4 py-2", className)
// caller's className wins on conflicting utilities
```

## Example 2: Minimal Button with cva (the canonical shadcn pattern)

```tsx
// components/ui/button.tsx
import * as React from "react"
import { Slot } from "@radix-ui/react-slot"
// or: import { Slot } from "radix-ui"
import { cva, type VariantProps } from "class-variance-authority"
import { cn } from "@/lib/utils"

const buttonVariants = cva(
  "inline-flex items-center justify-center gap-2 whitespace-nowrap rounded-md text-sm font-medium transition-colors focus-visible:outline-hidden focus-visible:ring-2 focus-visible:ring-ring disabled:pointer-events-none disabled:opacity-50",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground hover:bg-primary/90",
        destructive: "bg-destructive text-destructive-foreground hover:bg-destructive/90",
        outline: "border border-input bg-background hover:bg-accent hover:text-accent-foreground",
        secondary: "bg-secondary text-secondary-foreground hover:bg-secondary/80",
        ghost: "hover:bg-accent hover:text-accent-foreground",
        link: "text-primary underline-offset-4 hover:underline",
      },
      size: {
        default: "h-9 px-4 py-2",
        sm: "h-8 rounded-md px-3 text-xs",
        lg: "h-10 rounded-md px-8",
        icon: "size-9",
      },
    },
    defaultVariants: { variant: "default", size: "default" },
  },
)

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  asChild?: boolean
}

export const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, asChild = false, ...props }, ref) => {
    const Comp = asChild ? Slot : "button"
    return (
      <Comp
        ref={ref}
        className={cn(buttonVariants({ variant, size }), className)}
        {...props}
      />
    )
  },
)
Button.displayName = "Button"

export { buttonVariants }
```

### Usage

```tsx
<Button>Default primary</Button>
<Button variant="destructive">Delete</Button>
<Button variant="outline" size="sm">Small outline</Button>
<Button variant="ghost" size="icon"><Check /></Button>

// asChild composes onto another element (link, NextLink, etc.)
<Button asChild>
  <a href="/sign-in">Sign in</a>
</Button>

// caller className wins on conflicts
<Button variant="default" className="rounded-full px-12">Custom rounded</Button>
```

## Example 3: Radix Dialog wrapped by shadcn

```tsx
// components/ui/dialog.tsx
"use client"

import * as React from "react"
import * as DialogPrimitive from "@radix-ui/react-dialog"
// or: import { Dialog as DialogPrimitive } from "radix-ui"
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
    className={cn(
      "fixed inset-0 z-50 bg-black/80 data-[state=open]:animate-in data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0",
      className,
    )}
    {...props}
  />
))
DialogOverlay.displayName = DialogPrimitive.Overlay.displayName

const DialogContent = React.forwardRef<
  React.ElementRef<typeof DialogPrimitive.Content>,
  React.ComponentPropsWithoutRef<typeof DialogPrimitive.Content>
>(({ className, children, ...props }, ref) => (
  <DialogPortal>
    <DialogOverlay />
    <DialogPrimitive.Content
      ref={ref}
      className={cn(
        "fixed left-1/2 top-1/2 z-50 grid w-full max-w-lg -translate-x-1/2 -translate-y-1/2 gap-4 border bg-background p-6 shadow-lg duration-200 sm:rounded-lg",
        className,
      )}
      {...props}
    >
      {children}
      <DialogPrimitive.Close className="absolute right-4 top-4 rounded-sm opacity-70 ring-offset-background transition-opacity hover:opacity-100 focus:outline-hidden focus:ring-2 focus:ring-ring disabled:pointer-events-none">
        <X className="size-4" />
        <span className="sr-only">Close</span>
      </DialogPrimitive.Close>
    </DialogPrimitive.Content>
  </DialogPortal>
))
DialogContent.displayName = DialogPrimitive.Content.displayName

const DialogTitle = React.forwardRef<
  React.ElementRef<typeof DialogPrimitive.Title>,
  React.ComponentPropsWithoutRef<typeof DialogPrimitive.Title>
>(({ className, ...props }, ref) => (
  <DialogPrimitive.Title
    ref={ref}
    className={cn("text-lg font-semibold leading-none tracking-tight", className)}
    {...props}
  />
))
DialogTitle.displayName = DialogPrimitive.Title.displayName

const DialogDescription = React.forwardRef<
  React.ElementRef<typeof DialogPrimitive.Description>,
  React.ComponentPropsWithoutRef<typeof DialogPrimitive.Description>
>(({ className, ...props }, ref) => (
  <DialogPrimitive.Description
    ref={ref}
    className={cn("text-sm text-muted-foreground", className)}
    {...props}
  />
))
DialogDescription.displayName = DialogPrimitive.Description.displayName

export { Dialog, DialogTrigger, DialogContent, DialogTitle, DialogDescription, DialogClose }
```

### Usage

```tsx
import {
  Dialog, DialogTrigger, DialogContent, DialogTitle, DialogDescription,
} from "@/components/ui/dialog"

<Dialog>
  <DialogTrigger asChild>
    <Button>Open</Button>
  </DialogTrigger>
  <DialogContent>
    <DialogTitle>Are you sure?</DialogTitle>
    <DialogDescription>
      This action cannot be undone.
    </DialogDescription>
  </DialogContent>
</Dialog>
```

`DialogTitle` and `DialogDescription` are REQUIRED for screen-reader compliance per Radix docs.

## Example 4: Lucide icon imports

```tsx
import {
  Check, ChevronRight, ChevronLeft, ChevronDown, ChevronUp,
  X, Plus, Minus, Search, Settings, User, Loader2,
} from "lucide-react"

// Default shadcn-convention size
<Check className="size-4" />

// Animated spinner
<Loader2 className="size-4 animate-spin" />

// Inside a button (`gap-2` between text and icon, set on Button base)
<Button>
  <Plus />
  New item
</Button>

// As-child Slot composition (icon-only button)
<Button variant="ghost" size="icon" aria-label="Settings">
  <Settings />
</Button>
```

## Example 5: cn() override semantics

```tsx
// Component defines defaults
function Card({ className, ...props }: React.HTMLAttributes<HTMLDivElement>) {
  return (
    <div
      className={cn(
        "rounded-lg border bg-card p-6 shadow-sm",
        className,
      )}
      {...props}
    />
  )
}

// Caller can override conflicting utilities
<Card className="rounded-none border-0 p-2">…</Card>
// Resulting class string : "bg-card shadow-sm rounded-none border-0 p-2"
// (rounded-lg, border, p-6 are dropped by twMerge ; bg-card and shadow-sm stay)
```

## Example 6: cva compoundVariants

```ts
const alertVariants = cva(
  "relative w-full rounded-lg border px-4 py-3 text-sm",
  {
    variants: {
      variant: {
        default: "bg-background text-foreground",
        destructive: "border-destructive/50 text-destructive",
      },
      size: {
        default: "",
        compact: "px-2 py-1 text-xs",
      },
    },
    compoundVariants: [
      // destructive + compact gets extra emphasis
      { variant: "destructive", size: "compact", class: "font-semibold" },
    ],
    defaultVariants: { variant: "default", size: "default" },
  },
)
```

`compoundVariants` fires when ALL listed variants match. The `class` field appends to whatever the matched single variants produce.

## Example 7: Mixing radix-ui unified with the rest of the stack

```tsx
// Modern project (post Feb 2026) : import from unified package
import { Popover, DropdownMenu, Dialog } from "radix-ui"
import { cva, type VariantProps } from "class-variance-authority"
import { ChevronDown } from "lucide-react"
import { cn } from "@/lib/utils"

const dropdownItem = cva(
  "flex items-center gap-2 px-2 py-1.5 text-sm rounded-sm cursor-default",
  {
    variants: {
      variant: {
        default: "focus:bg-accent focus:text-accent-foreground",
        destructive: "text-destructive focus:bg-destructive focus:text-destructive-foreground",
      },
    },
    defaultVariants: { variant: "default" },
  },
)

export function MenuItem({ className, variant, ...props }: React.ComponentPropsWithoutRef<typeof DropdownMenu.Item> & VariantProps<typeof dropdownItem>) {
  return (
    <DropdownMenu.Item
      className={cn(dropdownItem({ variant }), className)}
      {...props}
    />
  )
}
```
