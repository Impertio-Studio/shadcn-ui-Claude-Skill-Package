# Drawer : Canonical Examples

Six working recipes. Every snippet is verified against `apps/v4/registry/new-york-v4/ui/drawer.tsx`, https://ui.shadcn.com/docs/components/radix/drawer, and the Vaul source at `emilkowalski/vaul`. Verified 2026-05-19.

All examples assume :
- shadcn ui evergreen-2026 (registry style `new-york-v4`)
- Tailwind v4 (no `tailwind.config.js` ; tokens via `@theme inline`)
- React 19 (no explicit `forwardRef` needed)
- The drawer file `components/ui/drawer.tsx` has `"use client"` at the top.

## Example 1 : Minimal bottom drawer (uncontrolled)

The smallest correct drawer. Default direction `"bottom"`. Drag-handle renders automatically. Vaul manages state.

```tsx
"use client"

import {
  Drawer,
  DrawerTrigger,
  DrawerContent,
  DrawerHeader,
  DrawerFooter,
  DrawerTitle,
  DrawerDescription,
  DrawerClose,
} from "@/components/ui/drawer"
import { Button } from "@/components/ui/button"

export function MoveFileDrawer() {
  return (
    <Drawer>
      <DrawerTrigger asChild>
        <Button variant="outline">Move file</Button>
      </DrawerTrigger>
      <DrawerContent>
        <DrawerHeader>
          <DrawerTitle>Move file</DrawerTitle>
          <DrawerDescription>Pick a destination folder.</DrawerDescription>
        </DrawerHeader>
        <div className="px-4 pb-2">
          {/* destination picker UI */}
        </div>
        <DrawerFooter>
          <Button>Move</Button>
          <DrawerClose asChild>
            <Button variant="outline">Cancel</Button>
          </DrawerClose>
        </DrawerFooter>
      </DrawerContent>
    </Drawer>
  )
}
```

Notes :
- No `useState`, no `open`, no `onOpenChange`. Vaul tracks visibility internally.
- DrawerTrigger wraps a Button via `asChild`. The Button receives merged props (data-state, click handler).
- DrawerClose wraps the "Cancel" Button so clicking it closes without a manual handler.
- The "Move" Button is plain : the parent handles the move logic and may flip to controlled state if it needs to close after a network call (see Example 4).

## Example 2 : Multi-snapPoint drawer (peek / half / full)

A drawer with three resting positions. Useful for "drag up to expand" UX (maps app, music player, comment thread).

```tsx
"use client"

import { useState } from "react"
import {
  Drawer,
  DrawerTrigger,
  DrawerContent,
  DrawerHeader,
  DrawerTitle,
  DrawerDescription,
} from "@/components/ui/drawer"
import { Button } from "@/components/ui/button"

export function PlaylistDrawer() {
  const [snap, setSnap] = useState<number | string | null>("148px")

  return (
    <Drawer
      snapPoints={["148px", "355px", 1]}
      activeSnapPoint={snap}
      setActiveSnapPoint={setSnap}
    >
      <DrawerTrigger asChild>
        <Button>Open playlist</Button>
      </DrawerTrigger>
      <DrawerContent>
        <DrawerHeader>
          <DrawerTitle>Now playing</DrawerTitle>
          <DrawerDescription>
            Drag the handle to peek, expand, or fullscreen.
          </DrawerDescription>
        </DrawerHeader>
        <div className="px-4 pb-8">
          {/* track list */}
        </div>
      </DrawerContent>
    </Drawer>
  )
}
```

Notes :
- `snapPoints={["148px", "355px", 1]}` mixes px strings (fixed peek height) with fraction `1` (fully open).
- ALWAYS sort ascending. NEVER pass `["355px", "148px", 1]` ; snap-distance math becomes ambiguous.
- The FIRST value (`"148px"`) MUST be tall enough for the DrawerHeader (title + description + `p-4`) ; otherwise the drawer pops over its own handle and header (see anti-patterns).
- `activeSnapPoint` + `setActiveSnapPoint` are paired (mirrors `open` + `onOpenChange`). Half-controlling freezes the snap state.

## Example 3 : Direction left drawer (off-canvas menu)

A drawer entering from the left edge. Useful for mobile off-canvas nav when the responsive-dialog-drawer recipe does NOT apply (i.e., the surface is mobile-only).

```tsx
"use client"

import {
  Drawer,
  DrawerTrigger,
  DrawerContent,
  DrawerHeader,
  DrawerTitle,
  DrawerDescription,
} from "@/components/ui/drawer"
import { Button } from "@/components/ui/button"
import { MenuIcon } from "lucide-react"

export function MobileNavDrawer() {
  return (
    <Drawer direction="left">
      <DrawerTrigger asChild>
        <Button variant="ghost" size="icon" aria-label="Open menu">
          <MenuIcon />
        </Button>
      </DrawerTrigger>
      <DrawerContent>
        <DrawerHeader>
          <DrawerTitle>Menu</DrawerTitle>
          <DrawerDescription>Navigate the app.</DrawerDescription>
        </DrawerHeader>
        <nav className="flex flex-col gap-1 p-4">
          <a href="/dashboard">Dashboard</a>
          <a href="/projects">Projects</a>
          <a href="/settings">Settings</a>
        </nav>
      </DrawerContent>
    </Drawer>
  )
}
```

Notes :
- `direction="left"` swaps the per-direction Tailwind variants : drawer becomes `inset-y-0 left-0 w-3/4 border-r sm:max-w-sm`.
- The drag-handle div is HIDDEN because the `group-data-[vaul-drawer-direction=bottom]/drawer-content:block` selector only unveils it for bottom.
- DrawerHeader auto-centers ONLY for top/bottom ; for left/right it stays `text-left` on `md:`.
- DrawerDescription is required-recommended ; if you genuinely have no description text, ALSO pass `aria-describedby={undefined}` on `<DrawerContent>` to silence the Radix-style warning.

## Example 4 : Controlled drawer + form (close on submit)

Form inside a drawer, closed after the submit succeeds. The shadcn docs' canonical recipe for mobile forms.

```tsx
"use client"

import { useState } from "react"
import {
  Drawer,
  DrawerTrigger,
  DrawerContent,
  DrawerHeader,
  DrawerFooter,
  DrawerTitle,
  DrawerDescription,
  DrawerClose,
} from "@/components/ui/drawer"
import { Button } from "@/components/ui/button"
import { Input } from "@/components/ui/input"
import { Label } from "@/components/ui/label"

export function EditProfileDrawer() {
  const [open, setOpen] = useState(false)

  async function onSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault()
    const formData = new FormData(e.currentTarget)
    await fetch("/api/profile", { method: "POST", body: formData })
    setOpen(false) // ALWAYS synchronously after the await
  }

  return (
    <Drawer open={open} onOpenChange={setOpen}>
      <DrawerTrigger asChild>
        <Button>Edit profile</Button>
      </DrawerTrigger>
      <DrawerContent>
        <form onSubmit={onSubmit}>
          <DrawerHeader>
            <DrawerTitle>Edit profile</DrawerTitle>
            <DrawerDescription>
              Update your name and email. Changes are saved on submit.
            </DrawerDescription>
          </DrawerHeader>
          <div className="space-y-3 px-4 pb-2">
            <div className="space-y-1">
              <Label htmlFor="name">Name</Label>
              <Input id="name" name="name" required />
            </div>
            <div className="space-y-1">
              <Label htmlFor="email">Email</Label>
              <Input id="email" name="email" type="email" required />
            </div>
          </div>
          <DrawerFooter>
            <Button type="submit">Save</Button>
            <DrawerClose asChild>
              <Button type="button" variant="outline">Cancel</Button>
            </DrawerClose>
          </DrawerFooter>
        </form>
      </DrawerContent>
    </Drawer>
  )
}
```

Notes :
- Both `open` and `onOpenChange` are passed. PAIRED ALWAYS.
- `setOpen(false)` runs SYNCHRONOUSLY after the `await`. NEVER rely on a `<DrawerClose>` wrapping the submit button : if validation fails, DrawerClose closes the drawer before the user sees the error.
- Cancel button uses `type="button"` so it does NOT trigger form submit.

## Example 5 : shouldScaleBackground (iOS-style stacked sheet)

Vaul scales the `[data-vaul-drawer-wrapper]` element on open, producing the iOS Files / Maps aesthetic.

```tsx
// app/layout.tsx (Next.js App Router)
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <div data-vaul-drawer-wrapper className="min-h-screen bg-background">
          {children}
        </div>
      </body>
    </html>
  )
}
```

```tsx
// components/help-drawer.tsx
"use client"

import {
  Drawer,
  DrawerTrigger,
  DrawerContent,
  DrawerHeader,
  DrawerTitle,
  DrawerDescription,
} from "@/components/ui/drawer"
import { Button } from "@/components/ui/button"

export function HelpDrawer() {
  return (
    <Drawer shouldScaleBackground>
      <DrawerTrigger asChild>
        <Button variant="outline">Help</Button>
      </DrawerTrigger>
      <DrawerContent>
        <DrawerHeader>
          <DrawerTitle>Help and feedback</DrawerTitle>
          <DrawerDescription>
            Find articles, contact support, or share feedback.
          </DrawerDescription>
        </DrawerHeader>
        <div className="px-4 pb-6">
          {/* help content */}
        </div>
      </DrawerContent>
    </Drawer>
  )
}
```

Notes :
- The wrapper div with `data-vaul-drawer-wrapper` is REQUIRED. Without it, `shouldScaleBackground` is a no-op (Vaul has nothing to scale).
- Pair with `setBackgroundColorOnScale={false}` if the app already uses a dark theme so Vaul does not over-tint the body.
- Only meaningful for `direction="bottom"` ; the iOS aesthetic is the bottom-sheet stack.

## Example 6 : Mobile-only drawer via useMediaQuery (forward to B10)

Quick conditional rendering : show the drawer ONLY on mobile, fall back to a non-drawer affordance on desktop. For the full responsive Dialog-on-desktop / Drawer-on-mobile pattern (shared content, single trigger) see `shadcn-impl-responsive-dialog-drawer` (Batch B10).

```tsx
"use client"

import { useEffect, useState } from "react"
import {
  Drawer,
  DrawerTrigger,
  DrawerContent,
  DrawerHeader,
  DrawerTitle,
  DrawerDescription,
} from "@/components/ui/drawer"
import { Button } from "@/components/ui/button"

function useMediaQuery(query: string) {
  const [match, setMatch] = useState(false)
  useEffect(() => {
    const m = window.matchMedia(query)
    setMatch(m.matches)
    const onChange = (e: MediaQueryListEvent) => setMatch(e.matches)
    m.addEventListener("change", onChange)
    return () => m.removeEventListener("change", onChange)
  }, [query])
  return match
}

export function ShareAction() {
  const isMobile = useMediaQuery("(max-width: 768px)")

  if (!isMobile) {
    return <Button variant="outline">Share</Button>
  }

  return (
    <Drawer>
      <DrawerTrigger asChild>
        <Button variant="outline">Share</Button>
      </DrawerTrigger>
      <DrawerContent>
        <DrawerHeader>
          <DrawerTitle>Share</DrawerTitle>
          <DrawerDescription>Pick a destination.</DrawerDescription>
        </DrawerHeader>
        <div className="grid grid-cols-3 gap-3 p-4">
          {/* share targets */}
        </div>
      </DrawerContent>
    </Drawer>
  )
}
```

Notes :
- `useMediaQuery` runs in `useEffect`, so initial render returns `false`. That is fine for client components ; if SSR matters, gate the entire return on a `mounted` state.
- This is the SIMPLE conditional pattern. The richer pattern (single trigger, swapped surface, shared content component) lives in `shadcn-impl-responsive-dialog-drawer`.
- ALWAYS keep DrawerTitle present even when the surface is conditional. The a11y rule applies whenever Drawer mounts.
