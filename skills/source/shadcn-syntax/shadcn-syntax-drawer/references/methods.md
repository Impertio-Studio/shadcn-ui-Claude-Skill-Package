# Drawer : Full API Reference

Verbatim signatures sourced from `apps/v4/registry/new-york-v4/ui/drawer.tsx` (shadcn-ui/ui, evergreen-2026 registry) and the Vaul source at `emilkowalski/vaul` (`src/index.tsx`, `src/types.ts`). Verified 2026-05-19.

## File header

```tsx
"use client"

import * as React from "react"
import { Drawer as DrawerPrimitive } from "vaul"

import { cn } from "@/lib/utils"
```

`"use client"` is REQUIRED. The `vaul` package re-exports its primitives under a single `Drawer.*` namespace, mirroring Radix Dialog's shape. shadcn re-wraps each primitive to inject `data-slot` attributes and the per-direction Tailwind variants.

## 1. Drawer (Root)

```tsx
function Drawer({
  ...props
}: React.ComponentProps<typeof DrawerPrimitive.Root>) {
  return <DrawerPrimitive.Root data-slot="drawer" {...props} />
}
```

### Props (verbatim from vaul `src/index.tsx`)

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `open` | `boolean` | - | Controlled visibility. MUST pair with `onOpenChange`. |
| `defaultOpen` | `boolean` | `false` | Uncontrolled initial state. EXCLUDES `open`. |
| `onOpenChange` | `(open: boolean) => void` | - | Fires on Trigger, Close, Esc, drag-dismiss, programmatic. |
| `modal` | `boolean` | `true` | When false, focus trap disabled and page stays interactive. |
| `direction` | `"top" \| "bottom" \| "left" \| "right"` | `"bottom"` | Edge the drawer enters from. Drives `data-vaul-drawer-direction`. |
| `snapPoints` | `(number \| string)[]` | - | Multi-stop fractions (0..1) or px strings ("148px"). |
| `activeSnapPoint` | `number \| string \| null` | - | Controlled active snap. Pair with `setActiveSnapPoint`. |
| `setActiveSnapPoint` | `(snap) => void` | - | Setter for `activeSnapPoint`. |
| `fadeFromIndex` | `number` | last snap index | Snap index at which overlay starts fading in. |
| `shouldScaleBackground` | `boolean` | `false` | Scales `[data-vaul-drawer-wrapper]` on open (iOS aesthetic). |
| `setBackgroundColorOnScale` | `boolean` | `true` | Tints body background black during scale. |
| `dismissible` | `boolean` | `true` | When false, drag-to-close AND Esc-to-close are disabled. |
| `nested` | `boolean` | `false` | Set on inner Drawer when nesting drawers. |
| `handleOnly` | `boolean` | `false` | Drag fires only from the handle, not the content surface. |
| `closeThreshold` | `number` | `0.25` | Drag fraction past which release closes the drawer. |
| `scrollLockTimeout` | `number` | `100` | ms before re-enabling body scroll on close. |
| `onDrag` | `(event, percentageDragged) => void` | - | Fires on every pointer-move during drag. |
| `onRelease` | `(event, open) => void` | - | Fires on pointer-up ; second arg is whether the drawer will end open. |
| `onAnimationEnd` | `(open) => void` | - | Fires when open/close animation completes. |
| `repositionInputs` | `boolean` | `true` | Scroll focused input into view on mobile keyboard. |
| `disablePreventScroll` | `boolean` | `false` | Lets the page behind scroll while drawer is open. |
| `preventScrollRestoration` | `boolean` | `true` | Don't restore body scroll on close. |
| `noBodyStyles` | `boolean` | `false` | Skip body-style manipulation entirely. |
| `autoFocus` | `boolean` | `false` | Auto-focus first focusable on open. |
| `container` | `HTMLElement \| null` | `document.body` | Portal target. |
| `children` | `ReactNode` | - | Trigger + Content tree. |

### Controlled-state contract

Pass BOTH `open` and `onOpenChange`, or NEITHER. Half-control freezes the drawer.

```tsx
// CORRECT
const [open, setOpen] = useState(false)
<Drawer open={open} onOpenChange={setOpen}>…</Drawer>

// CORRECT
<Drawer>…</Drawer>

// CORRECT
<Drawer defaultOpen>…</Drawer>

// WRONG (read-only, uncloseable)
<Drawer open={true}>…</Drawer>
```

### Controlled snap-point contract

Pair `activeSnapPoint` with `setActiveSnapPoint`. Same half-control failure mode.

```tsx
const [snap, setSnap] = useState<number | string | null>("148px")
<Drawer
  snapPoints={["148px", "355px", 1]}
  activeSnapPoint={snap}
  setActiveSnapPoint={setSnap}
>…</Drawer>
```

## 2. DrawerTrigger

```tsx
function DrawerTrigger({
  ...props
}: React.ComponentProps<typeof DrawerPrimitive.Trigger>) {
  return <DrawerPrimitive.Trigger data-slot="drawer-trigger" {...props} />
}
```

### Props

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `asChild` | `boolean` | `false` | Slot pattern : merges props into the child element. |
| `disabled` | `boolean` | `false` | Disabled triggers will not open the drawer. |
| Standard `<button>` props | - | - | When `asChild` is false, renders `<button type="button">`. |

### Data attributes

- `data-state="open" | "closed"`
- `data-slot="drawer-trigger"`

## 3. DrawerPortal

```tsx
function DrawerPortal({
  ...props
}: React.ComponentProps<typeof DrawerPrimitive.Portal>) {
  return <DrawerPrimitive.Portal data-slot="drawer-portal" {...props} />
}
```

### Props

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `container` | `HTMLElement` | `document.body` | Alternate render target. |
| `forceMount` | `boolean` | - | Keep mounted when closed (manual exit animation). |

## 4. DrawerClose

```tsx
function DrawerClose({
  ...props
}: React.ComponentProps<typeof DrawerPrimitive.Close>) {
  return <DrawerPrimitive.Close data-slot="drawer-close" {...props} />
}
```

### Props

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `asChild` | `boolean` | `false` | Slot pattern : wraps child as the close affordance. |
| Standard `<button>` props | - | - | When `asChild` is false, renders `<button type="button">`. |

## 5. DrawerOverlay

```tsx
function DrawerOverlay({
  className,
  ...props
}: React.ComponentProps<typeof DrawerPrimitive.Overlay>) {
  return (
    <DrawerPrimitive.Overlay
      data-slot="drawer-overlay"
      className={cn(
        "fixed inset-0 z-50 bg-black/50 data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:animate-in data-[state=open]:fade-in-0",
        className
      )}
      {...props}
    />
  )
}
```

### Default classes

- `fixed inset-0 z-50` : full viewport coverage at z-50.
- `bg-black/50` : 50 % black backdrop.
- `data-[state=open]:animate-in` + `data-[state=open]:fade-in-0` : fade-in on open.
- `data-[state=closed]:animate-out` + `data-[state=closed]:fade-out-0` : fade-out on close.

NEVER override `pointer-events-none` : Vaul needs pointer events for drag.

## 6. DrawerContent

```tsx
function DrawerContent({
  className,
  children,
  ...props
}: React.ComponentProps<typeof DrawerPrimitive.Content>) {
  return (
    <DrawerPortal data-slot="drawer-portal">
      <DrawerOverlay />
      <DrawerPrimitive.Content
        data-slot="drawer-content"
        className={cn(
          "group/drawer-content fixed z-50 flex h-auto flex-col bg-background",
          "data-[vaul-drawer-direction=top]:inset-x-0 data-[vaul-drawer-direction=top]:top-0 data-[vaul-drawer-direction=top]:mb-24 data-[vaul-drawer-direction=top]:max-h-[80vh] data-[vaul-drawer-direction=top]:rounded-b-lg data-[vaul-drawer-direction=top]:border-b",
          "data-[vaul-drawer-direction=bottom]:inset-x-0 data-[vaul-drawer-direction=bottom]:bottom-0 data-[vaul-drawer-direction=bottom]:mt-24 data-[vaul-drawer-direction=bottom]:max-h-[80vh] data-[vaul-drawer-direction=bottom]:rounded-t-lg data-[vaul-drawer-direction=bottom]:border-t",
          "data-[vaul-drawer-direction=right]:inset-y-0 data-[vaul-drawer-direction=right]:right-0 data-[vaul-drawer-direction=right]:w-3/4 data-[vaul-drawer-direction=right]:border-l data-[vaul-drawer-direction=right]:sm:max-w-sm",
          "data-[vaul-drawer-direction=left]:inset-y-0 data-[vaul-drawer-direction=left]:left-0 data-[vaul-drawer-direction=left]:w-3/4 data-[vaul-drawer-direction=left]:border-r data-[vaul-drawer-direction=left]:sm:max-w-sm",
          className
        )}
        {...props}
      >
        <div className="mx-auto mt-4 hidden h-2 w-[100px] shrink-0 rounded-full bg-muted group-data-[vaul-drawer-direction=bottom]/drawer-content:block" />
        {children}
      </DrawerPrimitive.Content>
    </DrawerPortal>
  )
}
```

### Notable behaviours

- Auto-wraps itself in `DrawerPortal` + `DrawerOverlay`. NEVER hand-wrap.
- Drag-handle div (`bg-muted` pill) is hidden by default, unveiled by `group-data-[vaul-drawer-direction=bottom]/drawer-content:block`. So the handle only appears for `direction="bottom"`.
- Per-direction Tailwind variants : top/bottom span full width (`inset-x-0`) with `max-h-[80vh]` and rounded inner edge ; left/right span full height (`inset-y-0`) with `w-3/4` and `sm:max-w-sm`.

### Props (forwarded to Vaul Content)

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `onOpenAutoFocus` | `(event) => void` | - | Called when focus moves into the content on open. |
| `onCloseAutoFocus` | `(event) => void` | - | Called when focus moves back on close. |
| `onEscapeKeyDown` | `(event) => void` | - | Called on Esc. `preventDefault()` keeps open. |
| `onPointerDownOutside` | `(event) => void` | - | Called when user clicks outside. |
| `onInteractOutside` | `(event) => void` | - | Union of pointer-down-outside + focus-outside. |
| `forceMount` | `boolean` | - | Keep mounted for custom exit animation. |
| `className` | `string` | - | Merged with the default direction-aware classes via `cn`. |

## 7. DrawerHeader

```tsx
function DrawerHeader({ className, ...props }: React.ComponentProps<"div">) {
  return (
    <div
      data-slot="drawer-header"
      className={cn(
        "flex flex-col gap-0.5 p-4 group-data-[vaul-drawer-direction=bottom]/drawer-content:text-center group-data-[vaul-drawer-direction=top]/drawer-content:text-center md:gap-1.5 md:text-left",
        className
      )}
      {...props}
    />
  )
}
```

Pure layout `<div>` with auto-centering on top/bottom drawers. Not a Vaul part ; safe to omit if your layout differs.

## 8. DrawerFooter

```tsx
function DrawerFooter({ className, ...props }: React.ComponentProps<"div">) {
  return (
    <div
      data-slot="drawer-footer"
      className={cn("mt-auto flex flex-col gap-2 p-4", className)}
      {...props}
    />
  )
}
```

Layout div pinned to the bottom of the drawer surface via `mt-auto`. No `showCloseButton` prop (unlike DialogFooter in v4).

## 9. DrawerTitle

```tsx
function DrawerTitle({
  className,
  ...props
}: React.ComponentProps<typeof DrawerPrimitive.Title>) {
  return (
    <DrawerPrimitive.Title
      data-slot="drawer-title"
      className={cn("font-semibold text-foreground", className)}
      {...props}
    />
  )
}
```

REQUIRED. Renders the accessible name (Vaul wires it via `aria-labelledby` on the content). Hide visually via `className="sr-only"` or `<VisuallyHidden asChild>`.

## 10. DrawerDescription

```tsx
function DrawerDescription({
  className,
  ...props
}: React.ComponentProps<typeof DrawerPrimitive.Description>) {
  return (
    <DrawerPrimitive.Description
      data-slot="drawer-description"
      className={cn("text-sm text-muted-foreground", className)}
      {...props}
    />
  )
}
```

Optional. If omitted, pass `aria-describedby={undefined}` to `DrawerContent` to suppress the Vaul-forwarded Radix warning.

## Export surface

```tsx
export {
  Drawer,
  DrawerPortal,
  DrawerOverlay,
  DrawerTrigger,
  DrawerClose,
  DrawerContent,
  DrawerHeader,
  DrawerFooter,
  DrawerTitle,
  DrawerDescription,
}
```

Ten primitives, all named exports.

## Data attributes summary

| Attribute | Where | Values |
|-----------|-------|--------|
| `data-state` | Trigger, Content, Overlay | `"open" \| "closed"` |
| `data-vaul-drawer-direction` | Content | `"top" \| "bottom" \| "left" \| "right"` |
| `data-vaul-drawer-wrapper` | App wrapper (manual) | presence-only ; required for `shouldScaleBackground` |
| `data-slot` | All shadcn-wrapped primitives | `"drawer"`, `"drawer-trigger"`, `"drawer-portal"`, `"drawer-overlay"`, `"drawer-content"`, `"drawer-header"`, `"drawer-footer"`, `"drawer-title"`, `"drawer-description"`, `"drawer-close"` |

Sourced verbatim from `apps/v4/registry/new-york-v4/ui/drawer.tsx`. Verified 2026-05-19.
