---
name: shadcn-syntax-sheet
description: >
  Use when building a side panel, slide-in drawer, off-canvas navigation, filter
  panel, settings tray, or any container that enters from the top, right, bottom,
  or left edge of the viewport in shadcn ui. Prevents the common mistakes of
  omitting SheetTitle (Radix a11y violation), choosing Sheet on mobile where
  Drawer would feel native, using side="bottom" Sheet where Drawer is the
  documented mobile-first pattern, and overriding SheetContent styles in a way
  that breaks the slide-in animation. Covers the side prop (top, right, bottom,
  left), the eight exported primitives (Sheet, SheetTrigger, SheetClose,
  SheetContent, SheetHeader, SheetFooter, SheetTitle, SheetDescription),
  controlled vs uncontrolled state (open + onOpenChange forwarded to the Radix
  Dialog primitive), showCloseButton prop on SheetContent, Sheet-vs-Dialog
  decision, and Sheet-vs-Drawer decision.
  Keywords: shadcn sheet, side panel, drawer-style dialog, SheetContent,
  SheetTitle, SheetTrigger, SheetClose, SheetDescription, navigation menu sheet,
  settings sheet, filter panel sheet, side="left" sheet, side="right" sheet,
  side="bottom" sheet, side="top" sheet, off-canvas panel, slide-in panel,
  Sheet vs Dialog, Sheet vs Drawer, how do I make a side panel, how do I make
  a slide-in drawer, sheet not showing, sheet missing title warning, sheet
  animation not working, showCloseButton sheet, sheet a11y warning, sheet
  describedby, mobile navigation drawer, responsive side panel.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn-syntax-sheet

Sheet is a Dialog-derivative styled to slide in from one of the four viewport
edges. It wraps the same Radix Dialog primitive as `shadcn-syntax-dialog`, so
the state model, a11y rules, and controlled-vs-uncontrolled semantics are
identical. The only meaningful additions on top of Dialog are: the `side` prop
on SheetContent (which swaps the slide-in animation and the fixed-position
classes) and the renamed exports.

## Quick Reference

Minimal Sheet (defaults to `side="right"`):

```tsx
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

export function FiltersSheet() {
  return (
    <Sheet>
      <SheetTrigger asChild>
        <Button variant="outline">Open filters</Button>
      </SheetTrigger>
      <SheetContent>
        <SheetHeader>
          <SheetTitle>Filters</SheetTitle>
          <SheetDescription>
            Narrow the result list. Changes apply immediately.
          </SheetDescription>
        </SheetHeader>
        {/* body content */}
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

When to reach for Sheet vs Dialog vs Drawer:

| Scenario | Pick | Why |
|----------|------|-----|
| Centered confirmation, short form, alert | Dialog | Centered modal is the conventional shape for confirmation. |
| Side navigation, filter panel, settings tray, long form | Sheet | Slide-in from edge keeps the underlying context visible. |
| Mobile-first bottom sheet, touch-drag affordance | Drawer (Vaul) | Drawer has native drag-to-dismiss + spring physics. |
| Same UI on desktop AND mobile, want native feel each | Responsive Dialog + Drawer | Use `shadcn-impl-responsive-dialog-drawer` recipe. |

## side prop

`SheetContent` accepts a `side` prop with four values. Default is `"right"`.

| Value | Slide direction | Fixed position classes | Typical use |
|-------|-----------------|------------------------|-------------|
| `right` (default) | Slides in from right edge | `inset-y-0 right-0 h-full w-3/4 sm:max-w-sm` | Filters, settings, contextual side panel |
| `left` | Slides in from left edge | `inset-y-0 left-0 h-full w-3/4 sm:max-w-sm` | Primary navigation, app menu |
| `top` | Slides down from top edge | `inset-x-0 top-0 h-auto` | Header-attached search, banner-style panel |
| `bottom` | Slides up from bottom edge | `inset-x-0 bottom-0 h-auto` | Desktop-only bottom panel. On mobile, prefer Drawer. |

ALWAYS pass `side` as a literal string on `SheetContent`, NEVER on the root
`<Sheet>`. The root forwards directly to `Dialog.Root` and ignores `side`.

```tsx
// CORRECT
<Sheet>
  <SheetTrigger>Open</SheetTrigger>
  <SheetContent side="left">{/* ... */}</SheetContent>
</Sheet>

// WRONG: side is silently ignored on root
<Sheet side="left">
  <SheetContent>{/* ... */}</SheetContent>
</Sheet>
```

## Decision tree: Sheet vs Dialog vs Drawer

```
Is the content a short confirmation, short form, or alert?
  YES -> Use Dialog (see shadcn-syntax-dialog).
  NO  -> continue.

Is the experience primarily mobile (touch-first, expect drag-to-dismiss)?
  YES -> Use Drawer (see shadcn-syntax-drawer).
  NO  -> continue.

Does the same UI need to feel native on BOTH desktop and mobile?
  YES -> Use Dialog (desktop) + Drawer (mobile) via useMediaQuery.
         See shadcn-impl-responsive-dialog-drawer.
  NO  -> continue.

Is it a side-anchored panel (nav, filters, settings, long form)?
  YES -> Use Sheet. Choose side: left for primary nav, right for
         contextual / filters / settings, top/bottom for header- or
         footer-attached panels.
```

## Composition primitives

shadcn ui exports eight Sheet primitives in v4 (mirrors Dialog naming):

1. `Sheet` : root. Forwards to `Dialog.Root` (Radix). Accepts `open`,
   `onOpenChange`, `defaultOpen`, `modal`.
2. `SheetTrigger` : button that opens the sheet. Forwards to `Dialog.Trigger`.
   Supports `asChild` for slotting your own button.
3. `SheetClose` : button that closes the sheet. Forwards to `Dialog.Close`.
   Supports `asChild`.
4. `SheetContent` : the slide-in surface. Forwards to `Dialog.Content`. Adds
   the `side` and `showCloseButton` props. Internally renders Radix
   `Dialog.Portal` and `Dialog.Overlay` so you do NOT compose them yourself.
5. `SheetHeader` : `<div>` wrapper, `flex flex-col gap-1.5 p-4`.
6. `SheetFooter` : `<div>` wrapper, `mt-auto flex flex-col gap-2 p-4`.
7. `SheetTitle` : forwards to `Dialog.Title`. REQUIRED for a11y.
8. `SheetDescription` : forwards to `Dialog.Description`. Strongly recommended
   for a11y. If you intentionally omit it, pass `aria-describedby={undefined}`
   to `SheetContent` to silence the Radix dev warning.

NOTE: v4 does NOT export `SheetPortal` or `SheetOverlay`. They exist as
internal helpers inside the shadcn `sheet.tsx` file but are not part of the
public API. If you need to override the overlay or render into a custom
container, drop down to the Radix Dialog primitive directly (or copy
`sheet.tsx` and extend the internal helpers locally).

The eight exports are imported from `@/components/ui/sheet` after running
`pnpm dlx shadcn@latest add sheet`.

## Accessibility

Sheet inherits Radix Dialog a11y semantics in full.

- `SheetTitle` is MANDATORY. Radix logs a console warning in dev when
  `Dialog.Content` mounts without an accessible name. Screen readers will
  announce the sheet without a title.
- `SheetDescription` is strongly recommended. If you do NOT want a visible
  description, either:
  - Render `SheetDescription` inside a visually-hidden wrapper, OR
  - Pass `aria-describedby={undefined}` to `SheetContent` to opt out of the
    Radix warning.
- Focus is automatically trapped inside `SheetContent` while open and returned
  to the trigger on close. Do NOT add your own focus-management code that
  fights Radix.
- `Escape` closes by default; outside click closes by default. Override via
  `onEscapeKeyDown` / `onPointerDownOutside` on `SheetContent` (forwarded to
  the Radix primitive).
- Body scroll is locked while the sheet is open and `modal` is `true` (the
  default). Setting `modal={false}` on `Sheet` disables focus trap and scroll
  lock; only do this for non-blocking patterns.
- When the trigger is a non-button element, wrap with `asChild` so the
  composed child is the accessible button:
  ```tsx
  <SheetTrigger asChild>
    <Button variant="ghost">Open</Button>
  </SheetTrigger>
  ```

## Common UX patterns

### Filters panel (right side, default)
The right side is the conventional location for contextual side panels because
it does not collide with the primary left-edge navigation in most layouts. Use
`side="right"` (or omit, since it is the default).

### Settings tray (right side, wider)
Override the width with a Tailwind utility, but preserve the slide-animation
classes. Pass an extra `className` to `SheetContent` and let `cn()` merge:
```tsx
<SheetContent className="sm:max-w-lg">{/* ... */}</SheetContent>
```
NEVER replace the full className string. Doing so strips the
`data-[state=open]:slide-in-from-right` keyframe and the sheet appears without
animation.

### Mobile navigation menu (left side)
Use `side="left"` for the primary app navigation drawer on small viewports.
This is the same pattern the shadcn Sidebar uses internally on mobile (see
LESSONS L-#5746: the internal mobile Sidebar Sheet REQUIRES `SheetTitle` for
a11y).

### Bottom panel on desktop
`side="bottom"` is valid for desktop bottom panels (e.g., docked output
console, secondary toolbar). On mobile, prefer Drawer for the same anchor
because Drawer provides drag-to-dismiss and spring physics that Sheet does
not.

### Form inside a Sheet
Long forms are a primary reason to pick Sheet over Dialog: the vertical
real estate of a side panel comfortably hosts many fields. Wrap the form body
in a scroll container so only the body scrolls, keeping `SheetHeader` and
`SheetFooter` pinned:
```tsx
<SheetContent className="flex flex-col">
  <SheetHeader>{/* title + description */}</SheetHeader>
  <div className="flex-1 overflow-y-auto px-4">{/* form fields */}</div>
  <SheetFooter>{/* submit + cancel */}</SheetFooter>
</SheetContent>
```

### Controlled state
Use the same controlled-state pattern as Dialog:
```tsx
const [open, setOpen] = React.useState(false)
<Sheet open={open} onOpenChange={setOpen}>{/* ... */}</Sheet>
```
NEVER pass `open` without `onOpenChange`. A half-controlled sheet becomes
read-only and the user cannot close it (anti-pattern, see Radix #half-open).

## Companion Skills

- `shadcn-syntax-dialog` (sibling, B4) : the parent primitive. Sheet inherits
  all Dialog semantics (state, a11y, controlled mode, focus trap).
- `shadcn-syntax-drawer` (B5) : mobile-first slide-in (Vaul). Use Drawer when
  you want native touch drag-to-dismiss, especially on mobile bottom sheets.
- `shadcn-impl-responsive-dialog-drawer` (B10) : end-to-end recipe for
  Dialog-on-desktop / Drawer-on-mobile with `useMediaQuery` and a shared
  content component. Compose with Sheet when you specifically want an
  edge-anchored desktop variant.
- `shadcn-core-cn` : the `cn()` helper used to merge `className` onto
  `SheetContent` without clobbering the slide animation classes.

## Reference files

- `references/methods.md` : full signature of every exported Sheet primitive,
  every forwarded Radix Dialog prop, and the typed `side` and
  `showCloseButton` props.
- `references/examples.md` : six full working examples (minimal right-side,
  left-side navigation, bottom on mobile fallback, sheet with form, sheet with
  controlled state, sheet with custom width).
- `references/anti-patterns.md` : six common anti-patterns with the actual
  failure mode and the corrected version side-by-side.
