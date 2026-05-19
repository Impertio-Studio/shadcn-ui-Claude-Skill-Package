---
name: shadcn-syntax-drawer
description: >
  Use when building a mobile-first bottom-sheet, swipe-to-dismiss panel,
  multi-stop drawer (snap points), or any sliding surface that enters from
  the bottom (default), top, left, or right edge of the viewport in shadcn
  ui. The Drawer is Vaul-backed (emilkowalski/vaul) and is a separate
  primitive family from Dialog and Sheet : it owns its own
  `Drawer / DrawerTrigger / DrawerPortal / DrawerOverlay / DrawerContent /
  DrawerHeader / DrawerFooter / DrawerTitle / DrawerDescription /
  DrawerClose` composition, exposes Vaul-only props (`snapPoints`,
  `shouldScaleBackground`, `setBackgroundColorOnScale`, `direction`,
  `dismissible`, `nested`, `handleOnly`, `closeThreshold`,
  `activeSnapPoint`, `onDrag`, `onRelease`), and is the documented
  half of the responsive Dialog-on-desktop / Drawer-on-mobile recipe.
  Prevents the canonical Drawer failures : omitting DrawerTitle (Radix
  Dialog a11y rule applies via Vaul ; axe-core flags the dialog as
  missing accessible name), reaching for Drawer on a desktop side-panel
  (Sheet is the correct primitive ; Drawer is mobile-first bottom-sheet),
  declaring `snapPoints` whose first fraction is smaller than the
  content's natural height (the drawer pops up over its own header and
  swallows the handle), nesting a Tooltip / DropdownMenu / Popover inside
  a Drawer without bumping z-index (Drawer overlay sits at z-50 and
  competes with the popper layer), and half-controlling state by passing
  `open` without `onOpenChange` (identical Radix controlled-state contract
  to Dialog : the drawer becomes read-only).
  Covers the ten Drawer primitives verbatim from `apps/v4/registry/
  new-york-v4/ui/drawer.tsx`, the Vaul Root prop surface (snapPoints,
  shouldScaleBackground, setBackgroundColorOnScale, direction default
  bottom, dismissible default true, modal default true, nested,
  handleOnly, closeThreshold default 0.25, activeSnapPoint controlled
  pair with setActiveSnapPoint, onDrag / onRelease / onAnimationEnd
  callbacks, repositionInputs, disablePreventScroll, container,
  autoFocus, defaultOpen, open + onOpenChange), the
  `data-vaul-drawer-direction` attribute that drives the
  per-direction Tailwind variants in DrawerContent, the mobile-first
  semantics (drag-handle only rendered when `direction="bottom"`), the
  Drawer-vs-Sheet-vs-Dialog decision tree, the mandatory
  `"use client"` directive, and the cross-link to the responsive
  Dialog-on-desktop / Drawer-on-mobile recipe.
  Keywords: shadcn drawer, vaul, bottom sheet, mobile drawer,
  snapPoints, snap points drawer, multi-stop drawer, shouldScaleBackground,
  setBackgroundColorOnScale, direction prop drawer, direction bottom,
  direction top, direction left, direction right, swipe to close,
  swipe to dismiss, drag handle drawer, DrawerTitle, DrawerTrigger,
  DrawerContent, DrawerPortal, DrawerOverlay, DrawerClose, DrawerHeader,
  DrawerFooter, DrawerDescription, dismissible drawer, modal drawer,
  nested drawer, handleOnly, closeThreshold, activeSnapPoint,
  onDrag drawer, onRelease drawer, repositionInputs, container drawer,
  Drawer vs Sheet, Drawer vs Dialog, when to use Drawer, mobile dialog,
  responsive dialog drawer, useMediaQuery drawer, drawer not closing,
  drawer stuck open, drawer missing title warning, drawer accessibility
  warning, drawer snap points not working, drawer pops over header,
  drawer z-index, drawer tooltip behind, drawer popover behind,
  drawer scaleBackground not working, drawer drag handle missing,
  how do I make a bottom sheet, how do I add a drag to dismiss panel,
  drawer blank screen, drawer read-only, drawer half controlled.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui : Drawer Syntax

Drawer is shadcn's mobile-first sliding surface. Unlike Dialog and Sheet (both Radix Dialog under the hood), Drawer wraps the **Vaul** library by Emil Kowalski (`emilkowalski/vaul`). Vaul re-implements the WAI-ARIA Dialog pattern with native-feeling drag gestures, snap points, and background scaling, then exposes a Radix-Dialog-compatible composition surface. That means the controlled-state contract and a11y rules from `shadcn-syntax-dialog` carry over verbatim, while the gesture / snap-point / scale-background features are Vaul-specific.

## Quick Reference

### The 10 primitives

| Primitive | Purpose |
|-----------|---------|
| `Drawer` | Root state container ; owns `open` / `onOpenChange` / `defaultOpen` / `modal` / `direction` / `snapPoints` / `shouldScaleBackground` / `dismissible` / `nested` |
| `DrawerTrigger` | Element that opens the drawer ; `asChild`-aware |
| `DrawerPortal` | Renders Overlay + Content inside `document.body` (or `container`) |
| `DrawerOverlay` | Backdrop ; styled by shadcn with `bg-black/50` + fade |
| `DrawerContent` | Main surface ; AUTO-WRAPS in `DrawerPortal` + `DrawerOverlay` ; renders a drag-handle only when `direction="bottom"` ; per-direction Tailwind variants driven by `data-vaul-drawer-direction` |
| `DrawerHeader` | Layout div ; auto-centers text when direction is `top` or `bottom` |
| `DrawerFooter` | Layout div with `mt-auto` so it pins to the bottom of the surface |
| `DrawerTitle` | Accessible name (REQUIRED, same as DialogTitle) |
| `DrawerDescription` | Accessible description (optional but recommended) |
| `DrawerClose` | Closes the drawer ; `asChild`-aware |

### Minimal Drawer (uncontrolled, default direction bottom)

```tsx
"use client"

import {
  Drawer, DrawerTrigger, DrawerContent,
  DrawerHeader, DrawerFooter,
  DrawerTitle, DrawerDescription, DrawerClose,
} from "@/components/ui/drawer"
import { Button } from "@/components/ui/button"

<Drawer>
  <DrawerTrigger asChild>
    <Button>Open</Button>
  </DrawerTrigger>
  <DrawerContent>
    <DrawerHeader>
      <DrawerTitle>Move file</DrawerTitle>
      <DrawerDescription>Pick a destination folder.</DrawerDescription>
    </DrawerHeader>
    <DrawerFooter>
      <Button>Move</Button>
      <DrawerClose asChild>
        <Button variant="outline">Cancel</Button>
      </DrawerClose>
    </DrawerFooter>
  </DrawerContent>
</Drawer>
```

Notice : no `direction` prop. Vaul defaults to `"bottom"`. The drag-handle (a small `bg-muted` pill) renders automatically because direction is bottom.

### Five invariants

1. ALWAYS provide a `DrawerTitle`. NEVER omit it ; Vaul forwards the Radix Dialog a11y contract, so missing title triggers the same dev-mode warning and axe-core critical flag as Dialog.
2. ALWAYS either control BOTH `open` and `onOpenChange`, or control NEITHER. NEVER pass `open` alone (the drawer becomes read-only ; identical failure mode to Dialog because Vaul mirrors the contract).
3. ALWAYS keep the `"use client"` directive at the top of `components/ui/drawer.tsx`. NEVER strip it ; Vaul uses pointer-events, refs, and Portal mounting.
4. ALWAYS render `<DrawerContent>` directly (it wraps itself in `DrawerPortal` + `DrawerOverlay`). NEVER hand-wrap unless you are deliberately retargeting `container` ; in that case render the explicit `<DrawerPortal container={el}>` form.
5. ALWAYS use Drawer for mobile-first bottom-sheets. NEVER reach for Drawer to build a desktop side panel ; that is `shadcn-syntax-sheet` territory.

## Decision Tree 1 : Drawer or Sheet or Dialog?

```
Q1. Is the surface a CENTERED modal (e.g., confirm dialog, profile edit
    modal) on every viewport?
    yes -> Dialog. Stop. Read shadcn-syntax-dialog.
    no  -> Q2

Q2. Is the surface a DESKTOP side panel that slides in from an edge
    (left nav, right filters, top notification tray) and the target
    audience is mostly mouse / pointer users ?
    yes -> Sheet. Stop. Read shadcn-syntax-sheet.
    no  -> Q3

Q3. Is the surface MOBILE-FIRST, draggable, possibly with snap points,
    and should the page behind scale and round on open (iOS sheet
    aesthetic) ?
    yes -> Drawer. Continue here.
    no  -> Q4

Q4. Does the UI need to ADAPT across viewports (centered dialog on
    desktop, bottom-sheet on mobile) ?
    yes -> Read shadcn-impl-responsive-dialog-drawer (B10) for the
           useMediaQuery + shared content recipe.
    no  -> Re-read Q1 ; the surface probably is a Dialog or Sheet.
```

NEVER use Drawer with `direction="left"` or `direction="right"` to replace Sheet on desktop. Both libraries can render left/right surfaces, but Sheet is wired to Radix Dialog's focus model and animations and is the documented desktop primitive. Drawer's left/right modes exist mainly for the responsive-mobile-pattern.

## Decision Tree 2 : Snap points : how many and what values?

```
Q1. Does the drawer have a SINGLE resting state (fully open, swipe to
    close) ?
    yes -> Omit `snapPoints` entirely. Vaul uses content height.
    no  -> Q2

Q2. Does the drawer have 2-3 stops (e.g., peek -> half -> full) ?
    yes -> Pass `snapPoints={[0.4, 0.75, 1]}` (fractions of viewport)
           or `snapPoints={["148px", "355px", 1]}` (mixed px + fraction).
           The LAST value is the fully-open state.
    no  -> Q3

Q3. Does the drawer have more than 3 stops?
    yes -> Reconsider UX. More than 3 snap-stops confuses users.
           Vaul supports any count, but UX cost grows non-linearly.
    no  -> Drop snapPoints.
```

ALWAYS sort snapPoints ASCENDING. NEVER pass `[0.75, 0.4, 1]` ; Vaul snaps to whichever value is nearest by drag offset and an unordered array produces unpredictable jumps.

ALWAYS make the FIRST snap point at least as tall as your DrawerHeader. NEVER pass `snapPoints={[0.1, 1]}` when the header is `p-4` + title + description : the drawer pops over its own header and the drag-handle disappears.

For controlled snap-point selection, pair `activeSnapPoint` with `setActiveSnapPoint`. Half-controlling here freezes the snap state the same way half-controlled `open` freezes visibility.

## Decision Tree 3 : direction = top / bottom / left / right?

```
Q1. Mobile-first bottom-sheet (iOS / Android style)?
    yes -> direction omitted (default "bottom"). Drag-handle renders.
    no  -> Q2

Q2. Notification tray or filter chips entering from the top edge?
    yes -> direction="top". DrawerHeader auto-centers.
    no  -> Q3

Q3. Off-canvas menu sliding from left/right edge AND the UI is
    mobile-first AND the responsive-dialog-drawer recipe does not apply?
    yes -> direction="left" or "right". Width is 3/4 viewport by
           default ; max-w-sm at sm: breakpoint.
    no  -> Reconsider Sheet.
```

The shadcn DrawerContent uses `data-[vaul-drawer-direction=<dir>]:*` Tailwind variants to swap fixed-position + border-radius classes per direction. This is verbatim from `apps/v4/registry/new-york-v4/ui/drawer.tsx` ; you can extend it the same way for custom directions.

## The 10 primitives in detail

### Drawer (Root)

Owns the state. Vaul Root accepts EVERY prop below ; the shadcn wrapper forwards them all via `...props`.

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `open` | `boolean` | - | Controlled visibility. PAIR WITH `onOpenChange`. |
| `defaultOpen` | `boolean` | `false` | Uncontrolled initial state. EXCLUDES `open`. |
| `onOpenChange` | `(open: boolean) => void` | - | Fires on Trigger, Close, Esc, drag-dismiss. |
| `modal` | `boolean` | `true` | When false, page stays interactive. |
| `direction` | `"top" \| "bottom" \| "left" \| "right"` | `"bottom"` | Edge the drawer enters from. |
| `snapPoints` | `(number \| string)[]` | - | Multi-stop drawer fractions / px values. |
| `activeSnapPoint` | `number \| string \| null` | - | Controlled current snap. Pair with `setActiveSnapPoint`. |
| `setActiveSnapPoint` | `(snap) => void` | - | Setter for activeSnapPoint. |
| `fadeFromIndex` | `number` | last | Snap index at which overlay starts fading in. |
| `shouldScaleBackground` | `boolean` | `false` | Scale + round the `[data-vaul-drawer-wrapper]` element on open (iOS aesthetic). REQUIRES a wrapper. |
| `setBackgroundColorOnScale` | `boolean` | `true` | Tints body background black when scaling. |
| `dismissible` | `boolean` | `true` | When false, drawer ignores drag-dismiss + Esc. |
| `nested` | `boolean` | `false` | Set on inner Drawer when nesting drawers (avoids double-scale). |
| `handleOnly` | `boolean` | `false` | Drag only fires from the handle, not the whole content. |
| `closeThreshold` | `number` | `0.25` | Fraction of height past which release closes the drawer. |
| `scrollLockTimeout` | `number` | `100` | ms before re-enabling body scroll on close. |
| `onDrag` | `(event, percentage) => void` | - | Fires on every pointer-move during drag. |
| `onRelease` | `(event, willOpen) => void` | - | Fires on pointer-up. |
| `onAnimationEnd` | `(open) => void` | - | Fires when open/close animation completes. |
| `repositionInputs` | `boolean` | `true` | Scroll focused input into view on mobile keyboard. |
| `disablePreventScroll` | `boolean` | `false` | Lets the page behind scroll while open. |
| `preventScrollRestoration` | `boolean` | `true` | Don't restore body scroll position on close. |
| `noBodyStyles` | `boolean` | `false` | Skip body-style manipulation entirely. |
| `autoFocus` | `boolean` | `false` | Auto-focus first focusable on open (matches Radix). |
| `container` | `HTMLElement \| null` | `document.body` | Portal target. |

ALWAYS choose either `defaultOpen` or `open`+`onOpenChange`. NEVER both. Vaul prefers the controlled prop and ignores the default, identical to Radix.

### DrawerTrigger

Element that toggles `open` to true. Default render is a `<button type="button">`. `asChild` to wrap any focusable element. Receives `data-state="open|closed"` for styling.

```tsx
<DrawerTrigger asChild>
  <Button variant="outline">Pick destination</Button>
</DrawerTrigger>
```

NEVER wrap two siblings in `<DrawerTrigger asChild>` ; Slot rejects multi-child.

### DrawerPortal

Renders Overlay + Content inside `document.body` by default. Props :

- `container?: HTMLElement` : alternate render target.
- `forceMount?: boolean` : keep mounted when closed (manual exit animation).

DrawerContent calls DrawerPortal internally with no props. To customise, use the explicit form.

### DrawerOverlay

Backdrop. Styled by shadcn with `fixed inset-0 z-50 bg-black/50` + fade animation via `data-[state=open]:animate-in` / `data-[state=closed]:animate-out`. Same z-index as DialogOverlay and SheetOverlay.

NEVER set `pointer-events-none` on the overlay : Vaul needs pointer events to fire drag-dismiss.

### DrawerContent

The main surface. Auto-portals (wraps in `DrawerPortal` + `DrawerOverlay`). Key behaviours :

- Renders a `<div className="mx-auto mt-4 hidden h-2 w-[100px] ... bg-muted ..." />` drag-handle that is hidden by default but unveiled by the `group-data-[vaul-drawer-direction=bottom]/drawer-content:block` selector. So the handle only appears when `direction="bottom"`.
- Per-direction Tailwind variants live in the className via `data-[vaul-drawer-direction=top|bottom|left|right]:*`. Top/bottom span full width with `max-h-[80vh]`. Left/right span 3/4 viewport width (`w-3/4`) and cap at `max-w-sm` from the `sm:` breakpoint up.
- Same `onOpenAutoFocus` / `onCloseAutoFocus` / `onEscapeKeyDown` / `onPointerDownOutside` / `onInteractOutside` callbacks as Radix Dialog.

ALWAYS render exactly ONE `<DrawerContent>` per `<Drawer>`. NEVER conditionally render two and toggle which is shown.

### DrawerHeader

Layout div with `flex flex-col gap-0.5 p-4`. Includes `group-data-[vaul-drawer-direction=bottom]/drawer-content:text-center` and the top-direction equivalent, so the header auto-centers on top/bottom drawers. On `md:` the layout switches to `text-left` regardless of direction.

### DrawerFooter

Layout div with `mt-auto flex flex-col gap-2 p-4`. The `mt-auto` is what pins the footer to the bottom of the drawer surface. There is NO `showCloseButton` prop here (unlike DialogFooter in v4) ; if you need an explicit close button, render `<DrawerClose asChild>` yourself.

### DrawerTitle

REQUIRED. Renders the accessible name. Styled with `font-semibold text-foreground`. Vaul forwards Radix Dialog's a11y wiring : without DrawerTitle, the same console warning and axe-core critical flag fires as for Dialog.

To hide visually : `<DrawerTitle className="sr-only">…</DrawerTitle>` or wrap with `<VisuallyHidden asChild>`.

### DrawerDescription

Optional supplemental text styled with `text-sm text-muted-foreground`. If you intentionally omit it, ALSO pass `aria-describedby={undefined}` to `<DrawerContent>` to suppress the Radix-style warning forwarded by Vaul.

### DrawerClose

Closes the drawer. Default render is `<button type="button">`. `asChild` to wrap any focusable element. Useful inside DrawerFooter for an explicit Cancel button.

```tsx
<DrawerClose asChild>
  <Button variant="outline">Cancel</Button>
</DrawerClose>
```

## a11y requirement : DrawerTitle is non-optional

Vaul forwards the WAI-ARIA Dialog pattern. Same rule, same failure surface as `shadcn-syntax-dialog` :

1. Missing DrawerTitle logs a development warning to the console.
2. axe-core / axe DevTools / Lighthouse a11y audit flag the drawer as critical (no accessible name).
3. Screen-reader users hear "dialog" with no further context.

If your drawer has no visible heading (e.g., a media-controls drawer), ALWAYS add `<DrawerTitle className="sr-only">…</DrawerTitle>`. NEVER drop the title primitive.

## Vaul-specific behaviours

### shouldScaleBackground

When true, Vaul scales and rounds the element with `data-vaul-drawer-wrapper` while the drawer is open, producing the iOS-style stacked sheet look. The wrapper element MUST exist :

```tsx
<div data-vaul-drawer-wrapper className="bg-background min-h-screen">
  {children /* the whole app */}
</div>
```

Without the wrapper, `shouldScaleBackground` is a no-op. Pair with `setBackgroundColorOnScale={false}` if your app already has a dark background so Vaul does not over-tint.

### Snap points

`snapPoints={[0.4, 0.75, 1]}` produces three resting positions at 40 %, 75 %, and 100 % of viewport height (for `direction="bottom"` and `"top"`) or width (for `"left"` and `"right"`). Mixed units are allowed : `snapPoints={["148px", "355px", 1]}`. Vaul interpolates the overlay fade from the index at `fadeFromIndex` (defaults to the last index).

For controlled snap selection :

```tsx
const [snap, setSnap] = useState<number | string | null>("148px")
<Drawer
  snapPoints={["148px", "355px", 1]}
  activeSnapPoint={snap}
  setActiveSnapPoint={setSnap}
>…</Drawer>
```

### dismissible / handleOnly / closeThreshold

- `dismissible={false}` disables drag-to-close AND Esc-to-close. Use for mandatory flows (paywall, terms-acceptance).
- `handleOnly={true}` confines drag-initiation to the drag-handle. Useful when the content has scrollable areas you do NOT want the user to accidentally drag.
- `closeThreshold={0.5}` raises the drag-distance required to close. Default `0.25`.

### Nested drawers

Set `nested` on the INNER drawer. Vaul then avoids re-scaling the wrapper and instead stacks the inner drawer over the outer with separate overlays.

```tsx
<Drawer>
  <DrawerContent>
    …
    <Drawer nested>
      <DrawerContent>…</DrawerContent>
    </Drawer>
  </DrawerContent>
</Drawer>
```

## Portal rendering and z-index

`<DrawerPortal>` mounts inside `document.body` by default. Same stacking behaviour as `<DialogPortal>` :

- Drawer escapes any `overflow: hidden`, `transform`, `filter`, or `will-change` ancestor.
- DrawerOverlay sits at `z-50`. Any Popover / DropdownMenu / Tooltip / HoverCard mounted INSIDE the drawer will need a higher z-index (default Radix popper is `z-50` too) or it stacks under the drawer's content surface unpredictably. The shadcn evergreen-2026 Tooltip defaults to `z-50` and SHOULD work, but custom popovers may need a `z-[60]` override.
- Nested drawers each use their own portal scope. Set `nested` on the inner Drawer.

To retarget (rare) :

```tsx
<DrawerPortal container={hostRef.current}>
  <DrawerOverlay />
  <DrawerContent>…</DrawerContent>
</DrawerPortal>
```

## RSC and the `"use client"` directive

The shadcn `drawer.tsx` ships with `"use client"` at the top. Same rationale as Dialog :

- Vaul uses pointer events, refs, and `useEffect` for body-style mutations.
- Snap-point math runs in `useState` + `useLayoutEffect`.
- The Portal mounts client-side.

ALWAYS keep `"use client"` in `components/ui/drawer.tsx`. NEVER strip it.

Components that consume `<Drawer>` may live in Server Components, but the drawer file itself draws the client boundary. See `shadcn-impl-rsc-vs-client-boundaries` for the broader RSC contract.

## Companion Skills

- [shadcn-syntax-dialog](../shadcn-syntax-dialog/SKILL.md) : the centered modal pattern ; shares the controlled-state contract and a11y rules.
- [shadcn-syntax-sheet](../shadcn-syntax-sheet/SKILL.md) : Dialog-derivative for desktop side panels ; read when the surface is NOT mobile-first.
- [shadcn-impl-responsive-dialog-drawer](../../shadcn-impl/shadcn-impl-responsive-dialog-drawer/SKILL.md) (Batch B10) : Dialog-on-desktop / Drawer-on-mobile recipe using `useMediaQuery` ; the canonical responsive pattern that combines this skill with `shadcn-syntax-dialog`.
- [shadcn-errors-radix-controlled](../../shadcn-errors/shadcn-errors-radix-controlled/SKILL.md) : the controlled-state failure catalogue, applies to Drawer through Vaul's mirrored contract.
- [shadcn-syntax-button](../shadcn-syntax-button/SKILL.md) : the `asChild` pattern used by DrawerTrigger and DrawerClose.

## Reference Links

- [references/methods.md](references/methods.md) : verbatim prop signatures for all ten primitives and the full Vaul Root prop surface.
- [references/examples.md](references/examples.md) : six canonical recipes (minimal bottom drawer, multi-snapPoint drawer, direction left drawer, controlled drawer + form, shouldScaleBackground, mobile-only via useMediaQuery).
- [references/anti-patterns.md](references/anti-patterns.md) : five anti-patterns with WHY and FIX.

## Sources

All claims trace to URLs in `SOURCES.md` :

- https://ui.shadcn.com/docs/components/radix/drawer (canonical Drawer docs, recipe with snap points)
- https://github.com/emilkowalski/vaul (Vaul library : props, snap-point semantics, shouldScaleBackground behaviour)
- https://github.com/shadcn-ui/ui (apps/v4/registry/new-york-v4/ui/drawer.tsx, verbatim v4 source ; per-direction variants ; drag-handle rendering)
- https://www.radix-ui.com/primitives/docs/components/dialog (a11y contract that Vaul mirrors)
- https://www.w3.org/WAI/ARIA/apg/patterns/dialog-modal/ (WAI-ARIA Dialog pattern)

Verified 2026-05-19.
