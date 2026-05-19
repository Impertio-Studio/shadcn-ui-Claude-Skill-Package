# shadcn-syntax-sheet : Working Examples

All examples assume:
- shadcn ui evergreen-2026 (v4 registry, `new-york` style).
- `sheet` component installed via `pnpm dlx shadcn@latest add sheet`.
- Tailwind v4 with `tw-animate-css` (the `tailwindcss-animate` replacement)
  imported in `globals.css`; the slide keyframes depend on it.
- `Button`, `Input`, `Label`, `Form*` primitives installed where used.

## 1. Minimal Sheet : right side (default)

The simplest valid Sheet. Uses uncontrolled state, default `side="right"`, the
built-in close button, and renders `SheetTitle` + `SheetDescription` to satisfy
Radix a11y.

```tsx
"use client"

import {
  Sheet,
  SheetClose,
  SheetContent,
  SheetDescription,
  SheetFooter,
  SheetHeader,
  SheetTitle,
  SheetTrigger,
} from "@/components/ui/sheet"
import { Button } from "@/components/ui/button"

export function MinimalSheet() {
  return (
    <Sheet>
      <SheetTrigger asChild>
        <Button variant="outline">Open panel</Button>
      </SheetTrigger>
      <SheetContent>
        <SheetHeader>
          <SheetTitle>Panel</SheetTitle>
          <SheetDescription>
            This is a minimal sheet. It slides in from the right.
          </SheetDescription>
        </SheetHeader>
        <SheetFooter>
          <SheetClose asChild>
            <Button>Done</Button>
          </SheetClose>
        </SheetFooter>
      </SheetContent>
    </Sheet>
  )
}
```

## 2. Left-side navigation sheet

Mobile-style primary navigation that slides in from the left. The `SheetTitle`
is visually hidden with `sr-only` because the menu items themselves are the
visual heading.

```tsx
"use client"

import Link from "next/link"
import { MenuIcon } from "lucide-react"

import {
  Sheet,
  SheetContent,
  SheetDescription,
  SheetHeader,
  SheetTitle,
  SheetTrigger,
} from "@/components/ui/sheet"
import { Button } from "@/components/ui/button"

const NAV = [
  { href: "/", label: "Home" },
  { href: "/projects", label: "Projects" },
  { href: "/billing", label: "Billing" },
  { href: "/settings", label: "Settings" },
]

export function MobileNavSheet() {
  return (
    <Sheet>
      <SheetTrigger asChild>
        <Button variant="ghost" size="icon" aria-label="Open navigation">
          <MenuIcon className="size-5" />
        </Button>
      </SheetTrigger>
      <SheetContent side="left" className="w-72">
        <SheetHeader>
          <SheetTitle className="sr-only">Navigation</SheetTitle>
          <SheetDescription className="sr-only">
            Primary site navigation.
          </SheetDescription>
        </SheetHeader>
        <nav className="flex flex-col gap-1 px-2">
          {NAV.map((item) => (
            <Link
              key={item.href}
              href={item.href}
              className="rounded-md px-3 py-2 text-sm hover:bg-accent hover:text-accent-foreground"
            >
              {item.label}
            </Link>
          ))}
        </nav>
      </SheetContent>
    </Sheet>
  )
}
```

## 3. Bottom-side sheet (desktop-only fallback)

Use `side="bottom"` only on desktop. For touch-first mobile bottom sheets,
switch to Drawer (Vaul) via `shadcn-impl-responsive-dialog-drawer`. This
example renders a docked output panel on a desktop dashboard.

```tsx
"use client"

import {
  Sheet,
  SheetContent,
  SheetDescription,
  SheetHeader,
  SheetTitle,
  SheetTrigger,
} from "@/components/ui/sheet"
import { Button } from "@/components/ui/button"

export function DesktopOutputSheet() {
  return (
    <Sheet>
      <SheetTrigger asChild>
        <Button variant="outline">Show output</Button>
      </SheetTrigger>
      <SheetContent side="bottom" className="h-64">
        <SheetHeader>
          <SheetTitle>Build output</SheetTitle>
          <SheetDescription>
            Live log from the current build job.
          </SheetDescription>
        </SheetHeader>
        <pre className="mx-4 mb-4 flex-1 overflow-auto rounded-md bg-muted p-3 text-xs">
          {`> compiling...
> 12 modules transformed
> done in 1.4s`}
        </pre>
      </SheetContent>
    </Sheet>
  )
}
```

## 4. Sheet with a form inside

Long form in a side panel. The header and footer are pinned via `flex flex-col`
on `SheetContent` and `mt-auto` (built into `SheetFooter`); the form body
scrolls independently.

```tsx
"use client"

import * as React from "react"

import {
  Sheet,
  SheetClose,
  SheetContent,
  SheetDescription,
  SheetFooter,
  SheetHeader,
  SheetTitle,
  SheetTrigger,
} from "@/components/ui/sheet"
import { Button } from "@/components/ui/button"
import { Input } from "@/components/ui/input"
import { Label } from "@/components/ui/label"

export function EditProfileSheet() {
  return (
    <Sheet>
      <SheetTrigger asChild>
        <Button variant="outline">Edit profile</Button>
      </SheetTrigger>
      <SheetContent className="flex flex-col sm:max-w-md">
        <SheetHeader>
          <SheetTitle>Edit profile</SheetTitle>
          <SheetDescription>
            Update your details. Changes save when you click Save.
          </SheetDescription>
        </SheetHeader>
        <form
          id="edit-profile"
          className="flex-1 space-y-4 overflow-y-auto px-4"
          onSubmit={(e) => e.preventDefault()}
        >
          <div className="space-y-2">
            <Label htmlFor="name">Name</Label>
            <Input id="name" defaultValue="Ada Lovelace" />
          </div>
          <div className="space-y-2">
            <Label htmlFor="email">Email</Label>
            <Input id="email" type="email" defaultValue="ada@example.com" />
          </div>
          {/* additional fields */}
        </form>
        <SheetFooter>
          <SheetClose asChild>
            <Button variant="outline">Cancel</Button>
          </SheetClose>
          <Button type="submit" form="edit-profile">Save</Button>
        </SheetFooter>
      </SheetContent>
    </Sheet>
  )
}
```

## 5. Controlled Sheet

When you need to open the Sheet programmatically (after a successful API call,
in response to a keyboard shortcut, from a parent component), use the
controlled-state pattern. ALWAYS pair `open` with `onOpenChange`.

```tsx
"use client"

import * as React from "react"

import {
  Sheet,
  SheetContent,
  SheetDescription,
  SheetHeader,
  SheetTitle,
} from "@/components/ui/sheet"
import { Button } from "@/components/ui/button"

export function ControlledSheet() {
  const [open, setOpen] = React.useState(false)

  // Open programmatically on Ctrl/Cmd+K.
  React.useEffect(() => {
    const onKey = (e: KeyboardEvent) => {
      if ((e.metaKey || e.ctrlKey) && e.key === "k") {
        e.preventDefault()
        setOpen(true)
      }
    }
    window.addEventListener("keydown", onKey)
    return () => window.removeEventListener("keydown", onKey)
  }, [])

  return (
    <>
      <Button onClick={() => setOpen(true)}>Open via state</Button>
      <Sheet open={open} onOpenChange={setOpen}>
        <SheetContent>
          <SheetHeader>
            <SheetTitle>Command palette</SheetTitle>
            <SheetDescription>
              Opens with Ctrl+K or Cmd+K.
            </SheetDescription>
          </SheetHeader>
        </SheetContent>
      </Sheet>
    </>
  )
}
```

## 6. Custom width while preserving slide animation

Override the width with an extra `className` on `SheetContent`. ALWAYS pass via
`className` so `cn()` merges with the base + side-specific classes; NEVER
construct a new className string from scratch.

```tsx
"use client"

import {
  Sheet,
  SheetContent,
  SheetDescription,
  SheetHeader,
  SheetTitle,
  SheetTrigger,
} from "@/components/ui/sheet"
import { Button } from "@/components/ui/button"

export function WideSheet() {
  return (
    <Sheet>
      <SheetTrigger asChild>
        <Button variant="outline">Open wide sheet</Button>
      </SheetTrigger>
      {/* Override sm:max-w-sm with a wider breakpoint utility. The base
          'fixed', the slide-in keyframes, and 'flex flex-col' are preserved
          because we are appending, not replacing. */}
      <SheetContent className="sm:max-w-2xl">
        <SheetHeader>
          <SheetTitle>Wide sheet</SheetTitle>
          <SheetDescription>
            Up to 42rem at the sm breakpoint, full 3/4 width below.
          </SheetDescription>
        </SheetHeader>
      </SheetContent>
    </Sheet>
  )
}
```
