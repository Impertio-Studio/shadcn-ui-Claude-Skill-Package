# Methods Reference : Radix Controlled-State Signatures

All signatures verified against the Radix UI documentation
(`radix-ui.com/primitives/docs/components/*`) and the shadcn ui
source (`apps/v4/registry/new-york-v4/ui/*.tsx`), 2026-05-19.
The unified `radix-ui` package (Feb 2026) re-exports the same
prop shapes that the per-primitive `@radix-ui/react-*` packages
expose.

## Root-Level State Props

### Dialog.Root, AlertDialog.Root, Sheet (Dialog variant)

```ts
type DialogRootProps = {
  open?: boolean
  defaultOpen?: boolean
  onOpenChange?: (open: boolean) => void
  modal?: boolean              // default: true
  children?: React.ReactNode
}
```

- `open` : controlled visibility. When set, `defaultOpen` is ignored.
- `defaultOpen` : initial state in uncontrolled mode. Defaults to `false`.
- `onOpenChange` : called with the next open state when the user or
  Radix internals request a state change (Escape, overlay click,
  Close button, `DialogClose` element).
- `modal` : when `true`, content is treated as inert (focus trap +
  scroll lock + `aria-hidden` on siblings). When `false`, the user
  can interact with outside elements while the dialog is open.
  AlertDialog hardcodes `modal=true` ; it cannot be turned off.

### Popover.Root, HoverCard.Root

```ts
type PopoverRootProps = {
  open?: boolean
  defaultOpen?: boolean
  onOpenChange?: (open: boolean) => void
  modal?: boolean              // Popover default: false ; HoverCard: not applicable
}
```

`HoverCard.Root` does NOT accept `modal` ; hover cards are
always non-modal by definition.

### DropdownMenu.Root, ContextMenu.Root, Menubar.Root (per menu)

```ts
type DropdownMenuRootProps = {
  open?: boolean
  defaultOpen?: boolean
  onOpenChange?: (open: boolean) => void
  modal?: boolean              // default: true (modal scroll-lock)
  dir?: "ltr" | "rtl"
}
```

For `ContextMenu`, the user opens via right-click ; `open` controls
the same state.

### Collapsible.Root

```ts
type CollapsibleRootProps = {
  open?: boolean
  defaultOpen?: boolean
  onOpenChange?: (open: boolean) => void
  disabled?: boolean
}
```

No portal, no focus trap, no `modal`. Simplest of the open-family
primitives.

### Tooltip.Root, Tooltip.Provider

```ts
type TooltipRootProps = {
  open?: boolean
  defaultOpen?: boolean
  onOpenChange?: (open: boolean) => void
  delayDuration?: number       // ms, overrides Provider
  disableHoverableContent?: boolean
}

type TooltipProviderProps = {
  delayDuration?: number       // default: 700
  skipDelayDuration?: number   // default: 300
  disableHoverableContent?: boolean
  children: React.ReactNode
}
```

Tooltip is rarely controlled. The PROVIDER governs cross-tooltip
timing (show next tooltip immediately after the first if it appears
within `skipDelayDuration`). For programmatic show-on-error
patterns, control `open` per Tooltip.Root.

### Select.Root

```ts
type SelectRootProps = {
  value?: string
  defaultValue?: string
  onValueChange?: (value: string) => void
  open?: boolean
  defaultOpen?: boolean
  onOpenChange?: (open: boolean) => void
  dir?: "ltr" | "rtl"
  name?: string
  disabled?: boolean
  required?: boolean
}
```

Two independent state pairs. The chosen value lives in
`value`+`onValueChange`. The popper visibility lives in
`open`+`onOpenChange`. Half-binding any pair causes the same
read-only symptom on that axis.

### Accordion.Root

```ts
// type="single"
type AccordionSingleProps = {
  type: "single"
  value?: string
  defaultValue?: string
  onValueChange?: (value: string) => void
  collapsible?: boolean
  dir?: "ltr" | "rtl"
  orientation?: "horizontal" | "vertical"
}

// type="multiple"
type AccordionMultipleProps = {
  type: "multiple"
  value?: string[]
  defaultValue?: string[]
  onValueChange?: (value: string[]) => void
}
```

`onValueChange` receives a `string` for single mode and a
`string[]` for multiple. Mixing them silently breaks because
TypeScript can discriminate but runtime cannot recover from a
wrong shape.

### Tabs.Root

```ts
type TabsRootProps = {
  value?: string
  defaultValue?: string
  onValueChange?: (value: string) => void
  orientation?: "horizontal" | "vertical"
  dir?: "ltr" | "rtl"
  activationMode?: "automatic" | "manual"
}
```

`activationMode="manual"` decouples focus from activation : Tab to
focus, Space/Enter to activate. Useful for tab panels that load
expensive content.

## Content-Level Focus & Dismissal Props

These props live on the `Content` of every overlay primitive
(`Dialog.Content`, `Popover.Content`, `DropdownMenu.Content`,
`AlertDialog.Content`, `Sheet.Content` (shadcn wraps `Dialog.Content`),
`HoverCard.Content`, `ContextMenu.Content`, `Select.Content`).

```ts
type OverlayContentProps = {
  onOpenAutoFocus?: (event: Event) => void
  onCloseAutoFocus?: (event: Event) => void
  onEscapeKeyDown?: (event: KeyboardEvent) => void
  onPointerDownOutside?: (event: PointerDownOutsideEvent) => void
  onInteractOutside?: (event: PointerDownOutsideEvent | FocusOutsideEvent) => void
  forceMount?: true            // pair with Presence for animation control
}
```

Call `event.preventDefault()` inside ANY of these handlers to
cancel the default behavior :

- `onOpenAutoFocus.preventDefault()` : skip Radix's auto-focus on
  the first focusable element. You usually pair this with a manual
  `myRef.current?.focus()` call.
- `onCloseAutoFocus.preventDefault()` : skip return-focus to the
  trigger. Useful when the trigger element is being unmounted by
  the close action.
- `onEscapeKeyDown.preventDefault()` : Escape key fires but does
  NOT close the overlay.
- `onPointerDownOutside.preventDefault()` : clicking outside fires
  but does NOT close.
- `onInteractOutside.preventDefault()` : the union event (pointer
  OR focus outside) fires but does NOT close.

`onInteractOutside` is a superset of `onPointerDownOutside`. Prefer
`onInteractOutside` for "do not close while a child portal (toast,
combobox) takes focus" patterns.

## Portal Props

```ts
type PortalProps = {
  container?: HTMLElement | null
  forceMount?: true
  children: React.ReactNode
}
```

- `container` : the DOM node to portal into. Defaults to
  `document.body`. Set this to a div you control to scope the portal
  inside a transformed parent or a custom z-index context.
- `forceMount` : always render the portal subtree, even when
  closed. Required when you wrap content in CSS-transition-based
  unmount-deferred logic (rarely needed in shadcn since the default
  animations are mount/unmount-driven by Radix Presence).

shadcn's `dialog.tsx` auto-wraps `DialogContent` in `DialogPortal`
+ `DialogOverlay`. To override the portal target, render
`DialogPortal` explicitly :

```tsx
<DialogPortal container={portalHostRef.current}>
  <DialogOverlay />
  <DialogContent>...</DialogContent>
</DialogPortal>
```

## Slot Contract (asChild)

```ts
import { Slot } from "@radix-ui/react-slot"
// or post-Feb-2026 :
import { Slot } from "radix-ui"

// Slot accepts EXACTLY ONE React child.
// React.Children.only(children) throws otherwise.
// The child MUST forward refs (be a native element OR use React.forwardRef).
```

`asChild={true}` on a trigger replaces the default `<button>` with
the single child while merging the trigger's props onto it. The
contract surface is :

1. `React.Children.only(children)` : throws "React.Children.only
   expected to receive a single React element child" for 0, 2, or
   more children.
2. `React.cloneElement(child, mergedProps, child.props.children)` :
   the child must accept arbitrary props (`onClick`, `aria-*`,
   `data-state`, `ref`).
3. Refs are merged via `composeRefs(forwardedRef, child.ref)`. A
   child that does NOT forward refs loses focus management.

Common asChild-compatible targets :

- `<button>` (native, accepts ref)
- `<a>` with `href` (native, accepts ref)
- Next.js `<Link>` (uses `React.forwardRef` since v13)
- React Router `<Link>` (uses `React.forwardRef`)
- shadcn `<Button>` (uses `React.forwardRef` and supports its own
  `asChild` for double-Slot composition)

Incompatible targets :

- A second sibling. The Slot crash is immediate.
- A `<div>` : accepts `onClick` and `ref` but ignores
  `aria-expanded`, `aria-haspopup`, `aria-controls`,
  `data-state`. The trigger has no accessible semantics.
- An `<input>` : focus is fine but `aria-haspopup` is meaningless
  on inputs ; semantics are wrong.
- A custom component without `React.forwardRef` : ref is dropped,
  focus return-on-close points to nothing.

## modal Prop Semantics Summary

| Primitive | modal default | modal=true effects | modal=false effects |
|-----------|---------------|---------------------|----------------------|
| Dialog, Sheet | `true` | focus trap, scroll lock on body, sibling `aria-hidden`, overlay blocks pointer events to outside | NO scroll lock, NO sibling `aria-hidden`, outside clicks still close via `onPointerDownOutside` |
| AlertDialog | `true` (locked) | same as Dialog modal=true, plus NO outside-click dismissal (only Action/Cancel) | n/a (cannot disable) |
| Popover | `false` | focus trap inside content, scroll lock, sibling `aria-hidden` | default behavior : focus does not trap, click-outside closes via pointer event |
| DropdownMenu, ContextMenu, Menubar | `true` | scroll lock + outside `aria-hidden`, item-select calls `onOpenChange(false)` | reduced inertness, advanced use only |
| HoverCard | n/a | n/a | always non-modal ; opens on hover/focus, closes on hover-leave |
| Tooltip | n/a | n/a | always non-modal ; never traps focus |

Scroll lock is implemented by adding `data-scroll-locked="1"` and
`overflow: hidden` to `<body>`. If a parent uses a transform-based
fixed layout, the scroll lock can still let the body scroll
visually : this is a known interaction with iOS Safari and
fixed-position headers.

## Drawer (Vaul) Controlled Contract

Drawer is NOT a Radix primitive. It uses the `vaul` library which
mirrors the controlled-state shape :

```ts
type DrawerRootProps = {
  open?: boolean
  defaultOpen?: boolean
  onOpenChange?: (open: boolean) => void
  shouldScaleBackground?: boolean   // shadcn default: true
  dismissible?: boolean             // default: true
  snapPoints?: (string | number)[]
  activeSnapPoint?: string | number | null
  setActiveSnapPoint?: (snap: string | number | null) => void
  modal?: boolean                   // default: true
  direction?: "top" | "bottom" | "left" | "right"  // default: "bottom"
  handleOnly?: boolean
  closeThreshold?: number           // 0..1, fraction of drawer height
}
```

Same `open` + `onOpenChange` rule applies. Additionally :

- `dismissible={false}` blocks drag-to-close even when state is
  controlled correctly. Use `open={false}` from the parent to
  programmatically close.
- `modal={true}` adds the same body-scroll-lock + outside-pointer
  blocking as Radix Dialog.

## Per-Component Imports

After the Feb 2026 unified package, prefer :

```ts
import {
  Dialog,
  DialogContent,
  DialogTrigger,
  DialogPortal,
  DialogOverlay,
} from "radix-ui"            // unified package, tree-shakeable
```

over the per-primitive form (still available, deprecated for new
code) :

```ts
import * as Dialog from "@radix-ui/react-dialog"   // legacy
```

shadcn's `dialog.tsx` import line was switched to `radix-ui` in the
`migrate radix` flow.
