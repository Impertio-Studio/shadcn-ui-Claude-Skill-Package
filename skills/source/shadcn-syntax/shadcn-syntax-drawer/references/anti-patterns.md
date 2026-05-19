# Drawer : Anti-Patterns

Five concrete failures that block real shadcn ui Drawer usage. Each entry follows : WRONG code, WHY it fails, FIX.

Sources : `apps/v4/registry/new-york-v4/ui/drawer.tsx`, https://ui.shadcn.com/docs/components/radix/drawer, https://github.com/emilkowalski/vaul (Vaul `src/index.tsx`, `src/use-snap-points.ts`), https://www.radix-ui.com/primitives/docs/components/dialog (a11y contract Vaul mirrors). Verified 2026-05-19.

## 1. Missing DrawerTitle

### WRONG

```tsx
<Drawer>
  <DrawerTrigger asChild><Button>Open</Button></DrawerTrigger>
  <DrawerContent>
    <DrawerDescription>Pick a destination folder.</DrawerDescription>
    <FolderPicker />
    <DrawerFooter>
      <Button>Move</Button>
    </DrawerFooter>
  </DrawerContent>
</Drawer>
```

### WHY it fails

Vaul implements the WAI-ARIA Dialog pattern and wires `aria-labelledby` from the `DrawerPrimitive.Title` element. Without a `<DrawerTitle>`, the drawer has NO accessible name. Three concrete consequences :

1. Vaul (forwarding Radix Dialog) logs a development warning to the console : `DrawerContent requires a DrawerTitle for the component to be accessible for screen reader users.`
2. axe-core, axe DevTools, and Lighthouse all flag the drawer with `aria-dialog-name` (severity critical).
3. Screen-reader users hear "dialog" with no further context and cannot tell what the drawer is for.

This is the same failure mode as missing DialogTitle (`shadcn-syntax-dialog` anti-pattern 2). Vaul is NOT exempt because it mirrors the contract verbatim.

### FIX

Always include a `<DrawerTitle>`. If you do not want it visible, use `sr-only` :

```tsx
<Drawer>
  <DrawerTrigger asChild><Button>Open</Button></DrawerTrigger>
  <DrawerContent>
    <DrawerHeader>
      <DrawerTitle>Move file</DrawerTitle>
      <DrawerDescription>Pick a destination folder.</DrawerDescription>
    </DrawerHeader>
    <FolderPicker />
    <DrawerFooter>
      <Button>Move</Button>
    </DrawerFooter>
  </DrawerContent>
</Drawer>
```

To hide visually :

```tsx
<DrawerTitle className="sr-only">Move file</DrawerTitle>
```

NEVER drop DrawerTitle entirely. The cost of `sr-only` is zero ; the cost of missing it is a critical a11y violation.

## 2. Drawer used for a desktop side panel (should be Sheet)

### WRONG

```tsx
// Desktop SaaS dashboard, right-side filter panel.
// Audience is mouse / pointer users on >=md viewports.

<Drawer direction="right">
  <DrawerTrigger asChild>
    <Button>Filters</Button>
  </DrawerTrigger>
  <DrawerContent>
    <DrawerHeader>
      <DrawerTitle>Filters</DrawerTitle>
    </DrawerHeader>
    <FilterForm />
  </DrawerContent>
</Drawer>
```

### WHY it fails

Drawer is Vaul-backed and mobile-first. It optimises for pointer-drag, snap points, and the iOS bottom-sheet aesthetic. Using it for a desktop right-side filter panel produces three concrete problems :

1. Drag-to-dismiss is enabled by default (`dismissible: true`). Mouse users do not expect a side panel to follow their cursor when they grab anywhere on the content surface.
2. The drag-handle div renders only for `direction="bottom"` (per the `group-data-[vaul-drawer-direction=bottom]/drawer-content:block` selector). On `direction="right"` there is no visual affordance for the drag, yet the gesture is still wired.
3. Focus management and animation are tuned for the bottom-sheet pattern. Sheet (Radix Dialog under the hood) is the documented desktop primitive and is what the shadcn examples ship.

Per `vooronderzoek-shadcn.md` §2 entry 23 and §11 : Drawer is mobile-first ; Sheet is the desktop side-panel. Mixing them produces inconsistent UX and ships gesture surface area you do not want.

### FIX

Use Sheet for the desktop side-panel :

```tsx
import {
  Sheet,
  SheetTrigger,
  SheetContent,
  SheetHeader,
  SheetTitle,
} from "@/components/ui/sheet"

<Sheet>
  <SheetTrigger asChild>
    <Button>Filters</Button>
  </SheetTrigger>
  <SheetContent side="right">
    <SheetHeader>
      <SheetTitle>Filters</SheetTitle>
    </SheetHeader>
    <FilterForm />
  </SheetContent>
</Sheet>
```

If the same UI must adapt across viewports (Sheet on desktop, Drawer on mobile), use the `shadcn-impl-responsive-dialog-drawer` recipe (Batch B10) which swaps the surface via `useMediaQuery`.

## 3. snapPoints conflicting with content height

### WRONG

```tsx
<Drawer snapPoints={[0.1, 0.5, 1]}>
  <DrawerContent>
    <DrawerHeader>
      <DrawerTitle>Now playing</DrawerTitle>
      <DrawerDescription>Drag to expand the queue.</DrawerDescription>
    </DrawerHeader>
    <PlayerControls />
    <TrackList />
  </DrawerContent>
</Drawer>
```

### WHY it fails

The first snap point `0.1` (10 % of viewport height) is shorter than the DrawerHeader alone : on a 800 px viewport that is 80 px, while a `p-4` header with title + description renders at roughly 110-120 px. Vaul snaps to the requested fraction regardless of content height, so :

1. The header is cropped or pushed above the visible drawer area.
2. The drag-handle div sits ON or ABOVE the title, breaking the affordance.
3. Users see a blank slice of the drawer and cannot tell what to drag.

This is the most common reported snap-point issue (verified against Vaul's `src/use-snap-points.ts` height math : snap fractions are absolute, not content-aware).

### FIX

ALWAYS make the first snap point at least as tall as the DrawerHeader + any always-visible content. Measure in DevTools, then use pixel strings so the value is independent of viewport height :

```tsx
<Drawer snapPoints={["180px", 0.5, 1]}>
  <DrawerContent>
    <DrawerHeader>
      <DrawerTitle>Now playing</DrawerTitle>
      <DrawerDescription>Drag to expand the queue.</DrawerDescription>
    </DrawerHeader>
    <PlayerControls />
    <TrackList />
  </DrawerContent>
</Drawer>
```

Rules :
- ALWAYS sort snapPoints ASCENDING. Vaul snaps to the nearest value by drag-offset ; an unordered array produces non-monotonic transitions.
- ALWAYS make the first value tall enough for the always-visible peek content (header + any controls that must always show).
- NEVER use 3+ snap points unless the UX rationale is documented ; users get confused past two-stop sheets.

## 4. Drawer + Tooltip / Popover z-index collision

### WRONG

```tsx
<Drawer>
  <DrawerContent>
    <DrawerHeader>
      <DrawerTitle>Settings</DrawerTitle>
    </DrawerHeader>
    <div className="p-4">
      <TooltipProvider>
        <Tooltip>
          <TooltipTrigger asChild>
            <Button variant="ghost">?</Button>
          </TooltipTrigger>
          <TooltipContent>Help text</TooltipContent>
        </Tooltip>
      </TooltipProvider>
    </div>
  </DrawerContent>
</Drawer>
```

### WHY it fails

DrawerOverlay sits at `z-50`. DrawerContent also sits at `z-50` (with `fixed` positioning later in DOM order so it paints above the overlay). The default shadcn Tooltip popper also renders at `z-50` and ALSO portals to `document.body`. Result :

1. Both the drawer surface and the tooltip popper compete at the same z-index.
2. DOM order determines paint order. Tooltip portals AFTER the drawer portal, so it usually wins, but the stacking is fragile : any future change to portal mount order or to either component's z-index turns the tooltip into a flicker behind the drawer.
3. Worse : if a Popover with its own overlay is inside the drawer, that overlay can blanket the drawer's interactive surface, breaking touch + click.

This was flagged in `vooronderzoek-shadcn.md` §9 entry 18 (Z-index / Portal stacking surprises ; required to be flagged for Drawer + Dialog inside transformed containers and for nested popper layers).

### FIX

Bump the z-index of popper-layer primitives INSIDE the drawer so they paint above the drawer content surface :

```tsx
<TooltipContent className="z-[60]">Help text</TooltipContent>
```

For Popover / DropdownMenu / HoverCard inside a drawer, do the same on their `Content` :

```tsx
<PopoverContent className="z-[60]">…</PopoverContent>
<DropdownMenuContent className="z-[60]">…</DropdownMenuContent>
```

Alternative : retarget the Drawer's portal to a child of the drawer wrapper using `<DrawerPortal container={…}>` so the drawer stays scoped to the wrapper's stacking context. This is rarer and only worth it if you control the whole layout.

NEVER rely on default `z-50` stacking when two portals overlap. Always promote the inner layer to `z-[60]` (or higher) explicitly.

## 5. Half-controlled `open` (without `onOpenChange`)

### WRONG

```tsx
const [open, setOpen] = useState(true)

<Drawer open={open}>
  <DrawerContent>
    <DrawerHeader>
      <DrawerTitle>Welcome</DrawerTitle>
    </DrawerHeader>
    <DrawerClose>Got it</DrawerClose>
  </DrawerContent>
</Drawer>
```

### WHY it fails

Vaul mirrors the Radix controlled-state contract verbatim. When `open` is passed without `onOpenChange` :

1. Vaul treats `open` as authoritative.
2. Internal change-events from DrawerTrigger, DrawerClose, Esc, drag-dismiss, and pointer-down-outside all fire ; Vaul forwards them to `onOpenChange` ; the callback is missing ; the events are silently DROPPED.
3. The drawer renders open and cannot be closed.

The reverse failure mode is identical : passing `onOpenChange` without `open` causes the callback to fire while Vaul still uses its internal uncontrolled state, so the parent sees a closed-then-open lifecycle that never reflects the displayed state.

Same failure mode also applies to `activeSnapPoint` without `setActiveSnapPoint` : the snap state freezes at the initial value.

Verified in `vooronderzoek-shadcn.md` §9 entry 3 (Dialog version) and confirmed for Vaul via `src/index.tsx` which uses the same `use-controllable-state.ts` hook as Radix Dialog.

### FIX

Pair them, every time :

```tsx
const [open, setOpen] = useState(true)

<Drawer open={open} onOpenChange={setOpen}>
  <DrawerContent>
    <DrawerHeader>
      <DrawerTitle>Welcome</DrawerTitle>
    </DrawerHeader>
    <DrawerClose asChild>
      <Button>Got it</Button>
    </DrawerClose>
  </DrawerContent>
</Drawer>
```

If you do not need controlled state, pass NEITHER prop and let Vaul track visibility internally. For a one-shot initial-open with no ongoing control, use `<Drawer defaultOpen>`.

Same rule for snap points :

```tsx
const [snap, setSnap] = useState<number | string | null>("148px")
<Drawer
  snapPoints={["148px", 0.5, 1]}
  activeSnapPoint={snap}          // CORRECT
  setActiveSnapPoint={setSnap}    // PAIRED
>…</Drawer>
```

NEVER pass `activeSnapPoint` alone, NEVER pass `open` alone. Half-control freezes the corresponding state.
