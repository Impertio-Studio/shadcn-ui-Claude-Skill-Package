---
name: shadcn-impl-responsive-dialog-drawer
description: >
  Use when building a modal surface that must look correct on both desktop and
  mobile in shadcn ui, when the design calls for a centered Dialog on large
  screens but a bottom-sheet Drawer on phones, when the user reports that the
  current Dialog feels cramped on a phone or that the on-screen keyboard
  covers the form fields, when wiring a delete-confirmation, edit-profile,
  filter-panel, share-sheet, or settings modal that must respect platform
  conventions (native swipe-down dismiss on iOS / Android), when building a
  responsive Combobox where the dropdown must become a full-height Drawer on
  mobile, when porting an existing Dialog to also work well on touch devices,
  or when the user asks "how do I make a responsive modal".
  Prevents the recurring failure modes of an ad-hoc responsive modal: rendering
  BOTH the Dialog and the Drawer simultaneously so two open surfaces appear at
  once and the ARIA tree has duplicate role=dialog nodes, calling
  useMediaQuery without an SSR guard so Next.js logs a hydration mismatch on
  every first paint, keeping two separate useState pairs for the Dialog open
  state and the Drawer open state so closing one does not close the other when
  the viewport resizes mid-session, hard-coding a 768px or 640px check inline
  in every component instead of extracting one named useMediaQuery hook,
  rendering different field sets or different content per breakpoint so the
  user loses input when they rotate the device, omitting DialogTitle and
  DialogDescription (or DrawerTitle and DrawerDescription) so the screen
  reader announces an unlabelled dialog, and forgetting that DrawerClose
  must wrap an asChild Button to actually close the Drawer.
  Covers the canonical Dialog-desktop / Drawer-mobile pattern from the
  official shadcn ui responsive example, the useMediaQuery hook with
  useSyncExternalStore (SSR-safe), the shared content component contract
  (same component for both surfaces, optional className for layout
  adjustments), the single-source open state pattern (one useState shared by
  Dialog and Drawer), the breakpoint choice rationale (md: 768px is the
  shadcn default), the ResponsiveModal orchestrator pattern that hides the
  branch from feature code, and the responsive Combobox variant
  (Popover-on-desktop / Drawer-on-mobile).
  This skill is the IMPLEMENTATION recipe that orchestrates the primitives
  taught in shadcn-syntax-dialog (B4) and shadcn-syntax-drawer (B5). Read
  those first if the Dialog or Drawer subcomponent trees are unclear.
  Keywords: responsive dialog, mobile drawer, desktop dialog, useMediaQuery,
  responsive modal, Dialog mobile, Drawer mobile, how do I make a responsive
  modal, breakpoint switch, vaul mobile, shadcn responsive modal,
  Dialog Drawer pattern, conditional render Dialog Drawer, matchMedia react,
  useSyncExternalStore media query, SSR safe useMediaQuery,
  hydration mismatch useMediaQuery, shared form Dialog Drawer,
  ResponsiveModal component, responsive Combobox Popover Drawer,
  Dialog cramped on phone, Drawer on desktop looks wrong,
  bottom sheet desktop, swipe down dismiss mobile,
  why is my modal too small on mobile, why does my modal show twice,
  768px breakpoint shadcn, md breakpoint Tailwind, vaul on desktop,
  Drawer instead of Dialog on mobile, delete confirmation responsive,
  edit profile responsive modal, filter panel mobile drawer.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui: responsive Dialog (desktop) + Drawer (mobile)

This skill is the IMPLEMENTATION recipe for the most-requested compositional pattern in shadcn ui: a single modal surface that renders as a centered `Dialog` on desktop and as a bottom-sheet `Drawer` on mobile. The pattern comes directly from the official `drawer-dialog` example in the shadcn ui repository.

ALWAYS read `shadcn-syntax-dialog` first if the Dialog subcomponent tree (`DialogTrigger`, `DialogContent`, `DialogHeader`, `DialogTitle`, `DialogDescription`, `DialogFooter`, `DialogClose`) is unclear.

ALWAYS read `shadcn-syntax-drawer` first if the Drawer subcomponent tree, the `direction` prop, or the Vaul-specific semantics (snap points, dismissible, repositionInputs) are unclear.

## Quick Reference

### The pattern in five lines

```tsx
"use client"
const [open, setOpen] = React.useState(false)
const isDesktop = useMediaQuery("(min-width: 768px)")
if (isDesktop) return <Dialog open={open} onOpenChange={setOpen}>...<Content />...</Dialog>
return <Drawer open={open} onOpenChange={setOpen}>...<Content />...</Drawer>
```

Five non-negotiable invariants:
1. ONE `useState` pair for the open flag. Dialog and Drawer share it.
2. ONE `useMediaQuery` hook to pick the surface. NEVER hard-code the breakpoint inline.
3. ONE shared content component (`<ProfileForm />`, `<DeleteForm />`, etc). Same props on both branches.
4. NEVER render both Dialog and Drawer at the same time. Use `if (isDesktop) return ...` (early return), NOT both branches mounted.
5. ALWAYS include `DialogTitle` + `DialogDescription` on the desktop branch and `DrawerTitle` + `DrawerDescription` on the mobile branch. Both Radix Dialog and Vaul require a labelled title for screen readers.

### Files you generate

```
src/
├── hooks/
│   └── use-media-query.ts          # the SSR-safe useMediaQuery hook
└── components/
    ├── ui/
    │   ├── dialog.tsx              # shadcn add dialog
    │   └── drawer.tsx              # shadcn add drawer
    └── responsive-modal.tsx        # the orchestrator (optional but recommended)
```

## Decision Tree: Dialog only, Drawer only, or responsive?

```
Is the modal ALWAYS modal (blocking, must-acknowledge)?
├─ YES, AND target is form-factor-agnostic (edit profile, delete confirm, filters)
│   └─ USE responsive Dialog + Drawer (this skill)
├─ YES, AND target is desktop-only admin UI
│   └─ USE Dialog only (see shadcn-syntax-dialog)
└─ NO, it is a mobile-first action sheet (share, sort, add-to-cart)
    └─ USE Drawer only on all viewports (see shadcn-syntax-drawer)
```

When in doubt: responsive. The official example uses it for an "Edit Profile" form, which is the canonical case.

## Step 1: Install the useMediaQuery hook

shadcn ui itself does NOT ship a `useMediaQuery` hook in `components/ui/`. You write it once into `src/hooks/use-media-query.ts`. The official shadcn ui example imports from `@/hooks/use-media-query`, which means the project owner authored it.

ALWAYS use the `useSyncExternalStore` variant (React 18+). NEVER use a `useState` + `useEffect` variant in a Next.js App Router project, because the initial render runs server-side, where `window` does not exist, and a `useEffect` variant produces a hydration mismatch on the very first paint.

Full implementation in `references/examples.md`. Signature:

```ts
function useMediaQuery(query: string): boolean
```

Returns `true` when the query matches, `false` otherwise. On the server, `useSyncExternalStore` falls back to the `getServerSnapshot` slot, which in this hook throws. shadcn ui's official example calls the hook from inside a `"use client"` component, so the server never executes it.

Breakpoint choice: `(min-width: 768px)` matches Tailwind's `md:` breakpoint. This is what the official shadcn `drawer-dialog` example uses and what this skill standardizes on. If a project uses a different design system breakpoint, pass it as an argument to a `ResponsiveModal` prop. NEVER copy the literal `768px` across many components.

## Step 2: Write the shared content component

The single most important design rule of the pattern: ONE content component renders inside BOTH surfaces. NEVER duplicate the form, body, or action layout per branch. NEVER show fewer fields on mobile and more on desktop, because a viewport rotation mid-edit then silently drops the hidden state.

The shared component MUST accept `className` (because Drawer often needs `px-4` padding that Dialog already supplies via `DialogContent`'s default padding) and MAY accept callbacks like `onSubmit` that close the modal.

```tsx
function ProfileForm({ className, onDone }: {
  className?: string
  onDone?: () => void
}) {
  return (
    <form className={cn("grid items-start gap-6", className)} onSubmit={...}>
      ...inputs...
      <Button type="submit">Save changes</Button>
    </form>
  )
}
```

Full example with all imports in `references/examples.md`.

## Step 3: Wire the conditional render

The canonical shape from the official shadcn ui example, with the responsive branching kept inside ONE component:

```tsx
"use client"

import * as React from "react"
import { useMediaQuery } from "@/hooks/use-media-query"
import { Button } from "@/components/ui/button"
import {
  Dialog, DialogContent, DialogDescription, DialogHeader,
  DialogTitle, DialogTrigger,
} from "@/components/ui/dialog"
import {
  Drawer, DrawerClose, DrawerContent, DrawerDescription,
  DrawerFooter, DrawerHeader, DrawerTitle, DrawerTrigger,
} from "@/components/ui/drawer"

export function EditProfileModal() {
  const [open, setOpen] = React.useState(false)
  const isDesktop = useMediaQuery("(min-width: 768px)")

  if (isDesktop) {
    return (
      <Dialog open={open} onOpenChange={setOpen}>
        <DialogTrigger asChild>
          <Button variant="outline">Edit Profile</Button>
        </DialogTrigger>
        <DialogContent className="sm:max-w-[425px]">
          <DialogHeader>
            <DialogTitle>Edit profile</DialogTitle>
            <DialogDescription>
              Make changes to your profile here. Click save when done.
            </DialogDescription>
          </DialogHeader>
          <ProfileForm onDone={() => setOpen(false)} />
        </DialogContent>
      </Dialog>
    )
  }

  return (
    <Drawer open={open} onOpenChange={setOpen}>
      <DrawerTrigger asChild>
        <Button variant="outline">Edit Profile</Button>
      </DrawerTrigger>
      <DrawerContent>
        <DrawerHeader className="text-left">
          <DrawerTitle>Edit profile</DrawerTitle>
          <DrawerDescription>
            Make changes to your profile here. Click save when done.
          </DrawerDescription>
        </DrawerHeader>
        <ProfileForm className="px-4" onDone={() => setOpen(false)} />
        <DrawerFooter className="pt-2">
          <DrawerClose asChild>
            <Button variant="outline">Cancel</Button>
          </DrawerClose>
        </DrawerFooter>
      </DrawerContent>
    </Drawer>
  )
}
```

Note four details from the official example:
- `DialogContent` carries `className="sm:max-w-[425px]"` so it does not stretch to the full Dialog max-width.
- `DrawerHeader` carries `className="text-left"` because the default `DrawerHeader` is centered (mobile bottom sheet convention) but a form header reads better left-aligned.
- `ProfileForm` carries `className="px-4"` only inside the Drawer, because `DialogContent` already pads its content while `DrawerContent` does not pad horizontally.
- `DrawerFooter` contains a `DrawerClose asChild` Cancel button. The Dialog branch does not include a Cancel button because Dialog ships a built-in close affordance (top-right X) via `DialogContent`'s `showCloseButton` default. The Drawer does not.

## Step 4: Extract the orchestrator (`ResponsiveModal`)

After two or three usages of the pattern, the if-else branching duplicates. Extract a `ResponsiveModal` orchestrator that hides the branch from feature code. ALWAYS forward `open`, `onOpenChange`, `title`, `description`, and `children` so the orchestrator stays content-agnostic.

```tsx
// components/responsive-modal.tsx
"use client"

import * as React from "react"
import { useMediaQuery } from "@/hooks/use-media-query"
import { Dialog, DialogContent, DialogDescription, DialogHeader, DialogTitle, DialogTrigger } from "@/components/ui/dialog"
import { Drawer, DrawerContent, DrawerDescription, DrawerHeader, DrawerTitle, DrawerTrigger } from "@/components/ui/drawer"

type ResponsiveModalProps = {
  open: boolean
  onOpenChange: (open: boolean) => void
  title: string
  description?: string
  trigger?: React.ReactNode
  children: React.ReactNode
  breakpoint?: string
}

export function ResponsiveModal({
  open, onOpenChange, title, description, trigger, children,
  breakpoint = "(min-width: 768px)",
}: ResponsiveModalProps) {
  const isDesktop = useMediaQuery(breakpoint)
  if (isDesktop) {
    return (
      <Dialog open={open} onOpenChange={onOpenChange}>
        {trigger ? <DialogTrigger asChild>{trigger}</DialogTrigger> : null}
        <DialogContent className="sm:max-w-[425px]">
          <DialogHeader>
            <DialogTitle>{title}</DialogTitle>
            {description ? <DialogDescription>{description}</DialogDescription> : null}
          </DialogHeader>
          {children}
        </DialogContent>
      </Dialog>
    )
  }
  return (
    <Drawer open={open} onOpenChange={onOpenChange}>
      {trigger ? <DrawerTrigger asChild>{trigger}</DrawerTrigger> : null}
      <DrawerContent>
        <DrawerHeader className="text-left">
          <DrawerTitle>{title}</DrawerTitle>
          {description ? <DrawerDescription>{description}</DrawerDescription> : null}
        </DrawerHeader>
        {children}
      </DrawerContent>
    </Drawer>
  )
}
```

Feature code then reads cleanly:

```tsx
<ResponsiveModal
  open={open}
  onOpenChange={setOpen}
  title="Edit profile"
  description="Make changes to your profile here."
  trigger={<Button variant="outline">Edit Profile</Button>}
>
  <ProfileForm className="md:px-0 px-4" onDone={() => setOpen(false)} />
</ResponsiveModal>
```

The orchestrator deliberately does NOT include a `DrawerFooter` + `DrawerClose` block, because the Drawer footer is content-specific (Cancel vs. Confirm vs. Apply). Place footer buttons inside `children`. See `references/examples.md` for a `DeleteConfirmation` variant that adds an explicit destructive-action footer.

## Step 5: Breakpoint choice rationale

`(min-width: 768px)` maps to Tailwind's `md:` and matches the official shadcn `drawer-dialog` example. The rationale:
- iPhone 14 Pro Max landscape is 932px wide so it lands on the Dialog branch in landscape; the Drawer is reserved for portrait phones, which is where the bottom-sheet ergonomics matter (thumb reach).
- iPad Mini portrait is 768px so it lands on the boundary; the example uses `min-width: 768px`, which includes 768px exactly, so iPad Mini portrait gets the Dialog.
- Smaller tablets (768px landscape but narrower in portrait) follow the same logic.

If the design system uses a different breakpoint (some teams prefer 640px to match Tailwind `sm:`, others 1024px to match `lg:` because they consider iPad-sized devices "mobile"), pass it via the `ResponsiveModal` `breakpoint` prop. NEVER fork the hook per call site.

## Anti-patterns

NEVER render both `<Dialog>` and `<Drawer>` and toggle them via CSS (`hidden md:block` + `block md:hidden`). Both Radix Dialog and Vaul mount portal nodes with `role="dialog"` and `aria-modal="true"`; rendering both creates a duplicated ARIA tree, focus-trap collisions, and `Title must be inside Dialog.Root` warnings from Radix when Vaul portals into a Dialog-managed focus zone. Use the early-return pattern instead.

Full list of seven failure modes with concrete examples in `references/anti-patterns.md`.

## Reference Files

- `references/methods.md`: `useMediaQuery` signature, `ResponsiveModal` props contract, shared-content-component contract.
- `references/examples.md`: SSR-safe `useMediaQuery` source, shared `FormContent`, `ResponsiveModal` orchestrator, full delete-confirmation flow, responsive Combobox (Popover-on-desktop / Drawer-on-mobile).
- `references/anti-patterns.md`: seven recurring failure modes with the wrong-then-right code for each.

## Companion Skills

- `shadcn-syntax-dialog` (Batch B4): Dialog subcomponent tree, `showCloseButton`, `DialogTitle` / `DialogDescription` a11y requirements, controlled `open` / `onOpenChange`.
- `shadcn-syntax-drawer` (Batch B5): Drawer subcomponent tree, Vaul-specific props (`direction`, `snapPoints`, `fadeFromIndex`, `dismissible`, `modal`, `repositionInputs`), bottom-sheet conventions.
- `shadcn-impl-form-validation`: when the responsive modal hosts a form (90% of cases), the form-state, submit, and `setError` patterns live here.
- `shadcn-impl-rsc-vs-client-boundaries`: the responsive modal MUST live in a `"use client"` boundary because `useMediaQuery` reads `window.matchMedia`. The orchestrator file itself carries `"use client"`; pages importing it stay server components.

## Sources

- Official responsive Dialog/Drawer example: `apps/v4/registry/new-york-v4/examples/drawer-dialog.tsx` in `shadcn-ui/ui` (verified 2026-05-19).
- shadcn ui Drawer docs: https://ui.shadcn.com/docs/components/radix/drawer (verified 2026-05-19).
- shadcn ui Dialog docs: https://ui.shadcn.com/docs/components/radix/dialog (verified 2026-05-19).
- `useMediaQuery` with `useSyncExternalStore`: https://github.com/uidotdev/usehooks (verified 2026-05-19).
- Vaul Drawer primitive: https://github.com/emilkowalski/vaul.
