# Dialog : Full API Reference

Verbatim signatures sourced from `apps/v4/registry/new-york-v4/ui/dialog.tsx` (shadcn-ui/ui, evergreen-2026 registry) and https://www.radix-ui.com/primitives/docs/components/dialog. Verified 2026-05-19.

## File header

```tsx
"use client"

import * as React from "react"
import { XIcon } from "lucide-react"
import { Dialog as DialogPrimitive } from "radix-ui"

import { cn } from "@/lib/utils"
import { Button } from "@/registry/new-york-v4/ui/button"
```

`"use client"` is REQUIRED. The `radix-ui` import is the post-Feb-2026 unified package ; pre-Feb-2026 projects use `@radix-ui/react-dialog` and the same surface.

## 1. Dialog (Root)

```tsx
function Dialog({
  ...props
}: React.ComponentProps<typeof DialogPrimitive.Root>) {
  return <DialogPrimitive.Root data-slot="dialog" {...props} />
}
```

### Props

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `open` | `boolean` | - | Controlled visibility. MUST pair with `onOpenChange`. |
| `defaultOpen` | `boolean` | `false` | Uncontrolled initial state. EXCLUDES `open`. |
| `onOpenChange` | `(open: boolean) => void` | - | Fires on Trigger, Close, Esc, pointer-down-outside, programmatic. |
| `modal` | `boolean` | `true` | When false, page stays interactive (focus trap disabled). |
| `children` | `ReactNode` | - | Trigger + Content tree. |

### Controlled-state contract

Pass BOTH `open` and `onOpenChange`, or NEITHER. Half-control freezes the dialog.

```tsx
// CORRECT
const [open, setOpen] = useState(false)
<Dialog open={open} onOpenChange={setOpen}>…</Dialog>

// CORRECT
<Dialog>…</Dialog>

// CORRECT
<Dialog defaultOpen>…</Dialog>

// WRONG (read-only, uncloseable)
<Dialog open={true}>…</Dialog>
```

## 2. DialogTrigger

```tsx
function DialogTrigger({
  ...props
}: React.ComponentProps<typeof DialogPrimitive.Trigger>) {
  return <DialogPrimitive.Trigger data-slot="dialog-trigger" {...props} />
}
```

### Props

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `asChild` | `boolean` | `false` | Slot pattern : wraps the child as the trigger. |
| `disabled` | `boolean` | `false` | Disabled triggers will not open the dialog. |
| Standard `<button>` props | - | - | When `asChild` is false, renders `<button type="button">`. |

### Data attributes

- `data-state="open" | "closed"`
- `data-slot="dialog-trigger"` (shadcn addition)

## 3. DialogPortal

```tsx
function DialogPortal({
  ...props
}: React.ComponentProps<typeof DialogPrimitive.Portal>) {
  return <DialogPrimitive.Portal data-slot="dialog-portal" {...props} />
}
```

### Props

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `container` | `HTMLElement` | `document.body` | Alternate render target. |
| `forceMount` | `boolean` | `undefined` | Keep mounted when closed (for custom exit anim). |
| `children` | `ReactNode` | - | Overlay + Content. |

DialogContent already calls DialogPortal internally. Use the explicit form only when overriding `container` or `forceMount`.

## 4. DialogOverlay

```tsx
function DialogOverlay({
  className,
  ...props
}: React.ComponentProps<typeof DialogPrimitive.Overlay>) {
  return (
    <DialogPrimitive.Overlay
      data-slot="dialog-overlay"
      className={cn(
        "fixed inset-0 z-50 bg-black/50 data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:animate-in data-[state=open]:fade-in-0",
        className
      )}
      {...props}
    />
  )
}
```

### Props

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `asChild` | `boolean` | `false` | Slot pattern. |
| `forceMount` | `boolean` | `undefined` | Keep mounted for exit anim. |
| `className` | `string` | - | Merged via `cn()` with the shadcn default. |

### Data attributes

- `data-state="open" | "closed"` (drives the fade-in / fade-out animation).

## 5. DialogContent

```tsx
function DialogContent({
  className,
  children,
  showCloseButton = true,
  ...props
}: React.ComponentProps<typeof DialogPrimitive.Content> & {
  showCloseButton?: boolean
}) {
  return (
    <DialogPortal data-slot="dialog-portal">
      <DialogOverlay />
      <DialogPrimitive.Content
        data-slot="dialog-content"
        className={cn(
          "fixed top-[50%] left-[50%] z-50 grid w-full max-w-[calc(100%-2rem)] translate-x-[-50%] translate-y-[-50%] gap-4 rounded-lg border bg-background p-6 shadow-lg duration-200 outline-none data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=closed]:zoom-out-95 data-[state=open]:animate-in data-[state=open]:fade-in-0 data-[state=open]:zoom-in-95 sm:max-w-lg",
          className
        )}
        {...props}
      >
        {children}
        {showCloseButton && (
          <DialogPrimitive.Close /* default XIcon close button */ />
        )}
      </DialogPrimitive.Content>
    </DialogPortal>
  )
}
```

### Props

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `showCloseButton` | `boolean` | `true` | shadcn-only addition. Toggles the built-in top-right X. |
| `onOpenAutoFocus` | `(event: Event) => void` | - | Fires when content receives focus on open. `preventDefault()` to skip. |
| `onCloseAutoFocus` | `(event: Event) => void` | - | Fires when focus returns on close. |
| `onEscapeKeyDown` | `(event: KeyboardEvent) => void` | - | `preventDefault()` to keep open. |
| `onPointerDownOutside` | `(event: PointerDownOutsideEvent) => void` | - | `preventDefault()` to keep open. |
| `onInteractOutside` | `(event: InteractOutsideEvent) => void` | - | Union of pointer + focus outside. |
| `forceMount` | `boolean` | `undefined` | Keep mounted for exit anim. |
| `asChild` | `boolean` | `false` | Slot pattern (rarely used here). |
| `className` | `string` | - | Merged with shadcn default. |

### Data attributes

- `data-state="open" | "closed"` (drives the zoom + fade animation).

### Auto-portal wrap

DialogContent renders this tree internally :

```
DialogPortal
  DialogOverlay
  DialogPrimitive.Content
    children
    [optional XIcon close button]
```

NEVER hand-wrap DialogContent in `<DialogPortal>` unless you are overriding `container`.

## 6. DialogHeader

```tsx
function DialogHeader({ className, ...props }: React.ComponentProps<"div">) {
  return (
    <div
      data-slot="dialog-header"
      className={cn("flex flex-col gap-2 text-center sm:text-left", className)}
      {...props}
    />
  )
}
```

Pure `<div>`. Not a Radix part. Layout-only.

## 7. DialogFooter

```tsx
function DialogFooter({
  className,
  showCloseButton = false,
  children,
  ...props
}: React.ComponentProps<"div"> & {
  showCloseButton?: boolean
}) {
  return (
    <div
      data-slot="dialog-footer"
      className={cn(
        "flex flex-col-reverse gap-2 sm:flex-row sm:justify-end",
        className
      )}
      {...props}
    >
      {children}
      {showCloseButton && (
        <DialogPrimitive.Close asChild>
          <Button variant="outline">Close</Button>
        </DialogPrimitive.Close>
      )}
    </div>
  )
}
```

### Props

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `showCloseButton` | `boolean` | `false` | Auto-renders a Close button at the end of children. shadcn evergreen-2026 addition. |
| `children` | `ReactNode` | - | Action buttons. |
| `className` | `string` | - | Merged via `cn()`. |

## 8. DialogTitle

```tsx
function DialogTitle({
  className,
  ...props
}: React.ComponentProps<typeof DialogPrimitive.Title>) {
  return (
    <DialogPrimitive.Title
      data-slot="dialog-title"
      className={cn("text-lg leading-none font-semibold", className)}
      {...props}
    />
  )
}
```

### Props

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `asChild` | `boolean` | `false` | Slot pattern. |
| `className` | `string` | - | Merged with shadcn default. |
| `children` | `ReactNode` | - | The accessible name. |

REQUIRED. Renders `<h2>` by default. To hide visually : `className="sr-only"` or wrap in `<VisuallyHidden asChild>` from `radix-ui`.

## 9. DialogDescription

```tsx
function DialogDescription({
  className,
  ...props
}: React.ComponentProps<typeof DialogPrimitive.Description>) {
  return (
    <DialogPrimitive.Description
      data-slot="dialog-description"
      className={cn("text-sm text-muted-foreground", className)}
      {...props}
    />
  )
}
```

Optional. Renders `<p>` by default. If intentionally omitted, pass `aria-describedby={undefined}` to `<DialogContent>` to suppress the Radix warning.

## 10. DialogClose

```tsx
function DialogClose({
  ...props
}: React.ComponentProps<typeof DialogPrimitive.Close>) {
  return <DialogPrimitive.Close data-slot="dialog-close" {...props} />
}
```

### Props

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `asChild` | `boolean` | `false` | Slot pattern : wrap a Button or any focusable child. |
| `disabled` | `boolean` | `false` | Disabled won't close on click. |
| Standard `<button>` props | - | - | Default render is `<button type="button">`. |

## Keyboard interactions (verbatim from Radix docs)

| Key | Behaviour |
|-----|-----------|
| `Space` / `Enter` | When focus is on DialogTrigger, opens. When focus is on DialogClose, closes. |
| `Tab` | Moves focus to next focusable inside Content. Focus is trapped : after the last focusable, returns to the first. |
| `Shift + Tab` | Moves focus to previous focusable inside Content. |
| `Esc` | Closes the dialog. Returns focus to the trigger element. Suppress via `onEscapeKeyDown` + `event.preventDefault()`. |

## Imports

All ten primitives are exported from `@/components/ui/dialog` :

```tsx
import {
  Dialog,
  DialogClose,
  DialogContent,
  DialogDescription,
  DialogFooter,
  DialogHeader,
  DialogOverlay,
  DialogPortal,
  DialogTitle,
  DialogTrigger,
} from "@/components/ui/dialog"
```

ALWAYS import from the local alias, NEVER from `radix-ui` directly ; the shadcn copies add the `data-slot` attributes and the default styling that the rest of your design system depends on.
