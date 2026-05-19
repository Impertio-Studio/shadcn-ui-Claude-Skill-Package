# shadcn-syntax-sheet : Method Reference

Verified against `apps/v4/registry/new-york-v4/ui/sheet.tsx` in `shadcn-ui/ui`
and the Radix UI Dialog primitive documentation. Sheet wraps Radix Dialog under
the alias `import { Dialog as SheetPrimitive } from "radix-ui"`, so every prop
listed here forwards directly to the Radix Dialog primitive unless explicitly
marked as shadcn-added.

## Public exports

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
```

NOTE: `SheetPortal` and `SheetOverlay` are defined inside `sheet.tsx` but are
NOT in the public export list. `SheetContent` composes them internally.

## Type aliases

```tsx
type SheetSide = "top" | "right" | "bottom" | "left"
```

## 1. Sheet

Root container. Forwards to `Dialog.Root`.

```tsx
function Sheet(
  props: React.ComponentProps<typeof SheetPrimitive.Root>
): JSX.Element
```

Forwarded Radix `Dialog.Root` props:

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `open` | `boolean` | `undefined` | Controlled open state. MUST be paired with `onOpenChange`. |
| `defaultOpen` | `boolean` | `false` | Uncontrolled initial open state. |
| `onOpenChange` | `(open: boolean) => void` | `undefined` | Called when open state changes. |
| `modal` | `boolean` | `true` | When `false`, disables focus trap and pointer-event blocking. |
| `children` | `ReactNode` | required | Must contain a `SheetTrigger` and a `SheetContent`. |

Renders with `data-slot="sheet"` for downstream styling hooks.

## 2. SheetTrigger

Button that opens the sheet. Forwards to `Dialog.Trigger`.

```tsx
function SheetTrigger(
  props: React.ComponentProps<typeof SheetPrimitive.Trigger>
): JSX.Element
```

| Prop | Type | Description |
|------|------|-------------|
| `asChild` | `boolean` | If `true`, merges props into the single child via Radix Slot. |
| `...buttonProps` | native `<button>` props | All standard button attributes pass through. |

Renders with `data-slot="sheet-trigger"` and exposes `data-state="open" | "closed"`.

## 3. SheetClose

Button that closes the sheet. Forwards to `Dialog.Close`.

```tsx
function SheetClose(
  props: React.ComponentProps<typeof SheetPrimitive.Close>
): JSX.Element
```

| Prop | Type | Description |
|------|------|-------------|
| `asChild` | `boolean` | Slot the close behavior onto a custom child. |
| `...buttonProps` | native `<button>` props | Standard button attributes. |

Renders with `data-slot="sheet-close"`.

## 4. SheetContent

The slide-in surface. Forwards to `Dialog.Content` and internally renders the
Portal and Overlay.

```tsx
function SheetContent(
  props: React.ComponentProps<typeof SheetPrimitive.Content> & {
    side?: "top" | "right" | "bottom" | "left"
    showCloseButton?: boolean
  }
): JSX.Element
```

shadcn-added props:

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `side` | `"top" \| "right" \| "bottom" \| "left"` | `"right"` | Edge of the viewport to slide in from. |
| `showCloseButton` | `boolean` | `true` | Render the built-in X close button (positioned `top-4 right-4`). |
| `className` | `string` | `undefined` | Merged via `cn()` with the side-specific defaults. ALWAYS append, NEVER replace. |

Forwarded Radix `Dialog.Content` props (selection):

| Prop | Type | Description |
|------|------|-------------|
| `onEscapeKeyDown` | `(e: KeyboardEvent) => void` | Call `e.preventDefault()` to keep open on Escape. |
| `onPointerDownOutside` | `(e: PointerDownOutsideEvent) => void` | Call `e.preventDefault()` to keep open on outside click. |
| `onInteractOutside` | `(e: PointerDownOutsideEvent \| FocusOutsideEvent) => void` | Combined outside interaction handler. |
| `onOpenAutoFocus` | `(e: Event) => void` | Customize initial focus target. |
| `onCloseAutoFocus` | `(e: Event) => void` | Customize focus restoration target. |
| `forceMount` | `true \| undefined` | Force the content to mount (useful for AnimatePresence-style libraries). |
| `aria-describedby` | `string \| undefined` | Pass `undefined` to opt out of the Radix description-warning when no `SheetDescription` is used. |
| `container` | `HTMLElement \| null` | Custom portal container (forwarded to the internal Portal). |

Renders with `data-slot="sheet-content"` and exposes `data-state="open" | "closed"`.

## 5. SheetHeader

Plain `<div>` wrapper for the title block.

```tsx
function SheetHeader(props: React.ComponentProps<"div">): JSX.Element
```

Default className: `"flex flex-col gap-1.5 p-4"`.

Renders with `data-slot="sheet-header"`.

## 6. SheetFooter

Plain `<div>` wrapper for the action block. Uses `mt-auto` to stick to the
bottom when `SheetContent` is `flex flex-col`.

```tsx
function SheetFooter(props: React.ComponentProps<"div">): JSX.Element
```

Default className: `"mt-auto flex flex-col gap-2 p-4"`.

Renders with `data-slot="sheet-footer"`.

## 7. SheetTitle

Accessible title. Forwards to `Dialog.Title`. REQUIRED for screen-reader
compliance.

```tsx
function SheetTitle(
  props: React.ComponentProps<typeof SheetPrimitive.Title>
): JSX.Element
```

Default className: `"font-semibold text-foreground"`.

Renders with `data-slot="sheet-title"`. To visually hide while keeping
accessible, wrap with `sr-only`:

```tsx
<SheetTitle className="sr-only">Navigation</SheetTitle>
```

## 8. SheetDescription

Accessible description. Forwards to `Dialog.Description`. Strongly recommended.

```tsx
function SheetDescription(
  props: React.ComponentProps<typeof SheetPrimitive.Description>
): JSX.Element
```

Default className: `"text-sm text-muted-foreground"`.

Renders with `data-slot="sheet-description"`. If omitted, pass
`aria-describedby={undefined}` to `SheetContent` to silence Radix.

## Internal helpers (NOT exported)

These exist inside `sheet.tsx` but are not in the public surface. Do not import
them. If overrides are needed, drop down to Radix Dialog directly.

```tsx
function SheetPortal(
  props: React.ComponentProps<typeof SheetPrimitive.Portal>
): JSX.Element

function SheetOverlay(
  props: React.ComponentProps<typeof SheetPrimitive.Overlay>
): JSX.Element
```

Default overlay className:
`"fixed inset-0 z-50 bg-black/50 data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:animate-in data-[state=open]:fade-in-0"`.

## Side-specific className matrix (applied inside SheetContent)

```
right  : inset-y-0 right-0 h-full w-3/4 border-l
         data-[state=closed]:slide-out-to-right
         data-[state=open]:slide-in-from-right sm:max-w-sm
left   : inset-y-0 left-0  h-full w-3/4 border-r
         data-[state=closed]:slide-out-to-left
         data-[state=open]:slide-in-from-left  sm:max-w-sm
top    : inset-x-0 top-0   h-auto border-b
         data-[state=closed]:slide-out-to-top
         data-[state=open]:slide-in-from-top
bottom : inset-x-0 bottom-0 h-auto border-t
         data-[state=closed]:slide-out-to-bottom
         data-[state=open]:slide-in-from-bottom
```

Shared base className (all sides):
`"fixed z-50 flex flex-col gap-4 bg-background shadow-lg transition ease-in-out data-[state=closed]:animate-out data-[state=closed]:duration-300 data-[state=open]:animate-in data-[state=open]:duration-500"`.
