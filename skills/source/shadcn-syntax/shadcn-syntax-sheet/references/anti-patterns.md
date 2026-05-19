# shadcn-syntax-sheet : Anti-Patterns

Each entry shows the broken pattern, the failure symptom, and the corrected
version. Symptoms are reproduced against shadcn ui evergreen-2026 (v4 registry,
new-york style) on Radix UI Dialog primitive.

## 1. Missing SheetTitle

`SheetContent` mounts without an accessible name.

### Symptom
Radix prints a console warning in dev:

> `Warning: DialogContent` requires a `DialogTitle` for the component to be accessible for screen reader users.

Screen readers announce the dialog without a name. Lighthouse a11y audit
flags the page.

### Broken
```tsx
<Sheet>
  <SheetTrigger>Open</SheetTrigger>
  <SheetContent>
    <p>Some content with no title.</p>
  </SheetContent>
</Sheet>
```

### Fixed
ALWAYS render `SheetTitle`. If the title would be visually redundant, hide it
with `sr-only` so it stays accessible:

```tsx
<Sheet>
  <SheetTrigger>Open</SheetTrigger>
  <SheetContent>
    <SheetHeader>
      <SheetTitle className="sr-only">Filters</SheetTitle>
      <SheetDescription className="sr-only">
        Filter panel for the result list.
      </SheetDescription>
    </SheetHeader>
    <p>Some content.</p>
  </SheetContent>
</Sheet>
```

NOTE: this is the same root cause as shadcn-ui/ui issue #5746 (Sidebar on
mobile : 49 reactions). The internal mobile Sidebar uses a Sheet and requires
a `SheetTitle` for the same reason.

## 2. Using Sheet on mobile where Drawer is the documented pattern

A bottom-anchored Sheet on a touch device feels wrong because Sheet has no
drag-to-dismiss and no spring physics. Users expect a bottom sheet to be
draggable.

### Symptom
Mobile users tap-and-hold the top of the sheet expecting it to follow the
finger and dismiss on flick. Nothing happens. Reviews call the UI
"not native".

### Broken
```tsx
// On a mobile-first surface, this gives a clipped, non-draggable panel.
<Sheet>
  <SheetTrigger>Open</SheetTrigger>
  <SheetContent side="bottom" className="h-1/2">
    {/* mobile content */}
  </SheetContent>
</Sheet>
```

### Fixed
Use Drawer (Vaul) on mobile. Compose the responsive pattern with
`useMediaQuery` to pick Dialog on desktop and Drawer on mobile:

```tsx
"use client"
import { useMediaQuery } from "@/hooks/use-media-query"
import { Drawer, DrawerContent, DrawerTrigger } from "@/components/ui/drawer"
import { Dialog, DialogContent, DialogTrigger } from "@/components/ui/dialog"

export function ResponsivePanel({ children }: { children: React.ReactNode }) {
  const isDesktop = useMediaQuery("(min-width: 768px)")
  if (isDesktop) {
    return (
      <Dialog>
        <DialogTrigger>Open</DialogTrigger>
        <DialogContent>{children}</DialogContent>
      </Dialog>
    )
  }
  return (
    <Drawer>
      <DrawerTrigger>Open</DrawerTrigger>
      <DrawerContent>{children}</DrawerContent>
    </Drawer>
  )
}
```

See `shadcn-syntax-drawer` and `shadcn-impl-responsive-dialog-drawer` for the
full recipe.

## 3. Tooltip inside a bottom-anchored Sheet

Radix Portals render at `document.body`. Tooltip and Sheet each create their
own Portal at a default `z-50`. When the Tooltip opens inside a `side="bottom"`
Sheet, the Tooltip can render behind the Sheet overlay or get clipped by the
Sheet's `inset-x-0 bottom-0` positioning.

### Symptom
Hovering a trigger inside the Sheet shows a Tooltip that flashes briefly and
then disappears under the Sheet, or appears clipped at the bottom edge of the
viewport. Sometimes the Tooltip is rendered but positioned off-screen.

### Broken
```tsx
<Sheet>
  <SheetContent side="bottom">
    <Tooltip>
      <TooltipTrigger>Hover me</TooltipTrigger>
      <TooltipContent>Helpful text</TooltipContent>
    </Tooltip>
  </SheetContent>
</Sheet>
```

### Fixed
Pin the Tooltip's portal container to the Sheet content itself so the Tooltip
inherits the Sheet's stacking context, OR raise the Tooltip's z-index above
the Sheet's `z-50`:

```tsx
<Sheet>
  <SheetContent side="bottom">
    <Tooltip>
      <TooltipTrigger>Hover me</TooltipTrigger>
      {/* z-[60] keeps the Tooltip above the Sheet overlay (z-50) and Sheet
          content (z-50). */}
      <TooltipContent className="z-[60]">Helpful text</TooltipContent>
    </Tooltip>
  </SheetContent>
</Sheet>
```

Verified against vooronderzoek §9 anti-pattern #18 (Z-index / Portal stacking
surprises) : Radix Portals at `document.body` collide when multiple primitives
share `z-50`.

## 4. Nesting a Sheet inside a CSS-transformed parent

If any ancestor of `SheetContent` uses `transform`, `filter`, or `will-change`,
the Sheet's portal escapes the transform context but the trigger's anchor
calculations can break. This is the same class of bug as Dialog and Drawer
inside transformed containers.

### Symptom
The Sheet opens at a wrong viewport position, or the overlay covers only part
of the screen. Sometimes the Sheet does not animate.

### Broken
```tsx
<div className="transform scale-100">
  <Sheet>
    <SheetTrigger>Open</SheetTrigger>
    <SheetContent>{/* ... */}</SheetContent>
  </Sheet>
</div>
```

### Fixed
Lift the Sheet out of the transformed container. The trigger can stay inside,
but the `Sheet` root must be rendered above any `transform` ancestor. Or
forward a `container` prop to `SheetContent` pointing at a known top-level DOM
node:

```tsx
// Option A : lift the Sheet out.
<>
  <div className="transform scale-100">
    <Button onClick={() => setOpen(true)}>Open</Button>
  </div>
  <Sheet open={open} onOpenChange={setOpen}>
    <SheetContent>{/* ... */}</SheetContent>
  </Sheet>
</>

// Option B : explicit portal container at the document root.
<Sheet>
  <SheetTrigger>Open</SheetTrigger>
  <SheetContent container={typeof document !== "undefined" ? document.body : null}>
    {/* ... */}
  </SheetContent>
</Sheet>
```

Verified against vooronderzoek §9 anti-pattern #18.

## 5. Overriding SheetContent className by replacement

Replacing the entire `className` string strips the slide-in keyframes, the
positioning utilities, and `flex flex-col`. The Sheet appears without
animation, sometimes fully covers the viewport, or stacks its header/body
incorrectly.

### Symptom
The Sheet "pops" in without sliding. Sticky `SheetFooter` no longer sticks.
The custom-width override looks correct but the visual transition is missing.

### Broken
```tsx
// Fork the className and you lose every default.
<SheetContent
  className="fixed inset-y-0 right-0 w-[420px] bg-background p-6"
>
  {/* No slide-in animation. No flex column. No border. */}
</SheetContent>
```

### Fixed
Pass ONLY the overrides as `className`. The component internally merges via
`cn()` so your classes win where they conflict and the rest is preserved:

```tsx
<SheetContent className="sm:max-w-md">
  {/* Base classes preserved : fixed, z-50, flex flex-col, slide-in keyframes,
      side="right" positioning. */}
</SheetContent>
```

When you need to override a specific Tailwind utility that conflicts with a
default, use a more specific variant (e.g., `sm:max-w-2xl` to override
`sm:max-w-sm`).

## 6. Half-controlled Sheet : passing `open` without `onOpenChange`

The Sheet renders, but user-initiated close actions (Escape, outside click,
the X close button, `SheetClose`) cannot update the parent state. The Sheet
becomes uncloseable from the user's perspective.

### Symptom
The Sheet opens. Clicking the X button does nothing. Pressing Escape does
nothing. The page is effectively trapped.

### Broken
```tsx
const [open, setOpen] = React.useState(false)

// open is controlled but onOpenChange is missing -> Radix freezes the state.
<Sheet open={open}>
  <SheetContent>{/* ... */}</SheetContent>
</Sheet>
```

### Fixed
ALWAYS pair `open` with `onOpenChange` when going controlled. If you want an
uncontrolled Sheet that defaults open, use `defaultOpen` instead.

```tsx
const [open, setOpen] = React.useState(false)

<Sheet open={open} onOpenChange={setOpen}>
  <SheetContent>{/* ... */}</SheetContent>
</Sheet>

// OR uncontrolled with initial-open:
<Sheet defaultOpen>
  <SheetContent>{/* ... */}</SheetContent>
</Sheet>
```

Same root cause as vooronderzoek §9 anti-pattern #3 : Radix controlled-state
half-open. Applies identically to Dialog, Sheet, Drawer, Popover, Tooltip.
