# Layout Primitives : Methods + Prop Signatures

Verbatim from `apps/v4/registry/new-york-v4/ui/{resizable,scroll-area,separator,aspect-ratio}.tsx` (shadcn-ui repo, evergreen-2026). Each section lists the imports, the function signatures from the source, the complete prop surface from the underlying library, and any v3 -> v4 renames.

---

## 1. Resizable

### Source-file imports (v4 registry)

```tsx
"use client"

import { GripVerticalIcon } from "lucide-react"
import * as ResizablePrimitive from "react-resizable-panels"

import { cn } from "@/lib/utils"
```

### Exports

```tsx
export { ResizableHandle, ResizablePanel, ResizablePanelGroup }
```

### Function signatures (verbatim)

```tsx
function ResizablePanelGroup(
  { className, ...props }: ResizablePrimitive.GroupProps
)

function ResizablePanel(
  { ...props }: ResizablePrimitive.PanelProps
)

function ResizableHandle(
  { withHandle, className, ...props }:
    ResizablePrimitive.SeparatorProps & { withHandle?: boolean }
)
```

`ResizablePanelGroup` adds `data-slot="resizable-panel-group"` and a base className `flex h-full w-full aria-[orientation=vertical]:flex-col`.

`ResizablePanel` adds only `data-slot="resizable-panel"`. All resize behavior is delegated to `react-resizable-panels`.

`ResizableHandle` adds `data-slot="resizable-handle"`, a styled `bg-border` divider, focus-ring styling, and conditionally renders `<GripVerticalIcon className="size-2.5" />` inside a `rounded-xs border` chip when `withHandle` is true.

### `ResizablePrimitive.GroupProps` (react-resizable-panels v4)

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `orientation` | `"horizontal" \| "vertical"` | `"horizontal"` | **v4 rename** of v3's `direction`. |
| `onLayoutChange` | `(sizes: number[]) => void` | `undefined` | **v4 rename** of v3's `onLayout`. |
| `autoSaveId` | `string` | `undefined` | localStorage key for layout persistence. |
| `id` | `string` | auto | Stable id (required for SSR-aware autoSave restoration). |
| `keyboardResizeBy` | `number \| null` | `10` | Px-step for arrow-key resize. `null` disables keyboard resize. |
| `tagName` | `string` | `"div"` | Element to render as. |
| `style` | `React.CSSProperties` | `undefined` | Inline style. |
| `className` | `string` | `undefined` | Forwarded ; merged with shadcn defaults via `cn()`. |

### `ResizablePrimitive.PanelProps`

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `defaultSize` | `number` | auto-distributed | Initial percent (0-100). |
| `minSize` | `number` | `0` | Lower bound percent (drag clamps to this). |
| `maxSize` | `number` | `100` | Upper bound percent (drag clamps to this). |
| `collapsible` | `boolean` | `false` | Enables collapse-below-minSize behavior. |
| `collapsedSize` | `number` | `0` | Percent the panel collapses to (must be < `minSize`). |
| `defaultCollapsed` | `boolean` | `false` | Initial collapsed state. |
| `onCollapse` | `() => void` | `undefined` | Fires when panel collapses. |
| `onExpand` | `() => void` | `undefined` | Fires when panel expands from collapsed. |
| `onResize` | `(size: number, prevSize: number \| undefined) => void` | `undefined` | Fires on every resize event for this panel. |
| `order` | `number` | auto | Stable order for conditional-render scenarios. |
| `id` | `string` | auto | Stable id for SSR / conditional render. |
| `tagName` | `string` | `"div"` | Element to render as. |

### `ResizablePrimitive.SeparatorProps` (handle)

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `withHandle` | `boolean` | `false` | shadcn-only addition : render the visible grip icon. |
| `disabled` | `boolean` | `false` | Disable drag interaction. |
| `hitAreaMargins` | `{ coarse?: number; fine?: number }` | `{ coarse: 15, fine: 5 }` | Px-margin around the handle for easier hit-testing. |
| `onDragging` | `(isDragging: boolean) => void` | `undefined` | Fires on drag start / stop. |
| `tagName` | `string` | `"div"` | Element to render as. |

### v3 -> v4 rename summary (Resizable)

| v3 | v4 |
|----|----|
| `direction` (on PanelGroup) | `orientation` |
| `onLayout` (on PanelGroup) | `onLayoutChange` |

All other v3 props carried over without rename. The shadcn `Resizable*` exports themselves did not rename ; only the underlying library renamed.

---

## 2. ScrollArea

### Source-file imports (v4 registry)

```tsx
"use client"

import * as React from "react"
import { ScrollArea as ScrollAreaPrimitive } from "radix-ui"

import { cn } from "@/lib/utils"
```

Note : v4 imports from the unified `radix-ui` package (Feb 2026 consolidation), not the per-primitive `@radix-ui/react-scroll-area`.

### Exports

```tsx
export { ScrollArea, ScrollBar }
```

### Function signatures (verbatim)

```tsx
function ScrollArea(
  { className, children, ...props }:
    React.ComponentProps<typeof ScrollAreaPrimitive.Root>
)

function ScrollBar(
  { className, orientation = "vertical", ...props }:
    React.ComponentProps<typeof ScrollAreaPrimitive.ScrollAreaScrollbar>
)
```

The `ScrollArea` function source composes internally :

```tsx
<ScrollAreaPrimitive.Root data-slot="scroll-area" className={cn("relative", className)} {...props}>
  <ScrollAreaPrimitive.Viewport
    data-slot="scroll-area-viewport"
    className="size-full rounded-[inherit] transition-[color,box-shadow] outline-none focus-visible:ring-[3px] focus-visible:ring-ring/50 focus-visible:outline-1"
  >
    {children}
  </ScrollAreaPrimitive.Viewport>
  <ScrollBar />
  <ScrollAreaPrimitive.Corner />
</ScrollAreaPrimitive.Root>
```

The internal `<ScrollBar />` is rendered with the default `orientation="vertical"`. This is why ScrollArea "just works" for vertical scroll : the bar is already composed.

### `ScrollArea` (Root) props

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `type` | `"auto" \| "always" \| "scroll" \| "hover"` | `"hover"` | When the scrollbar appears. `auto` mimics native overflow. |
| `scrollHideDelay` | `number` | `600` | ms before hiding scrollbar after interaction (for `scroll` / `hover`). |
| `dir` | `"ltr" \| "rtl"` | inherited | Reading direction (affects horizontal bar orientation). |
| `asChild` | `boolean` | `false` | Radix Slot composition. |
| `className` | `string` | `undefined` | Merged via `cn()` with `"relative"`. |

### `ScrollBar` props

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `orientation` | `"vertical" \| "horizontal"` | `"vertical"` | Required for horizontal axis. |
| `forceMount` | `boolean` | `false` | Force render even when `type` would hide it (useful for animations). |
| `asChild` | `boolean` | `false` | Radix Slot composition. |

The shadcn `ScrollBar` adds a styled `<ScrollAreaPrimitive.ScrollAreaThumb />` child internally. You never compose Thumb manually.

---

## 3. Separator

### Source-file imports (v4 registry)

```tsx
"use client"

import * as React from "react"
import { Separator as SeparatorPrimitive } from "radix-ui"

import { cn } from "@/lib/utils"
```

### Exports

```tsx
export { Separator }
```

### Function signature (verbatim)

```tsx
function Separator(
  { className, orientation = "horizontal", decorative = true, ...props }:
    React.ComponentProps<typeof SeparatorPrimitive.Root>
)
```

shadcn defaults : `orientation="horizontal"`, `decorative={true}`. The styled element is :

```tsx
<SeparatorPrimitive.Root
  data-slot="separator"
  decorative={decorative}
  orientation={orientation}
  className={cn(
    "shrink-0 bg-border data-[orientation=horizontal]:h-px data-[orientation=horizontal]:w-full data-[orientation=vertical]:h-full data-[orientation=vertical]:w-px",
    className
  )}
  {...props}
/>
```

### Props

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `orientation` | `"horizontal" \| "vertical"` | `"horizontal"` | Axis of the line. |
| `decorative` | `boolean` | `true` | `true` -> `role="none"` (omitted from a11y tree). `false` -> `role="separator"` with `aria-orientation`. |
| `asChild` | `boolean` | `false` | Radix Slot composition. |
| `className` | `string` | `undefined` | Merged via `cn()` ; `data-orientation` drives the size utilities. |

### `data-orientation` styling hook

The shadcn className uses Tailwind data-attribute selectors :

- `data-[orientation=horizontal]:h-px data-[orientation=horizontal]:w-full` -> 1-pixel-tall, full-width line.
- `data-[orientation=vertical]:h-full data-[orientation=vertical]:w-px` -> full-height, 1-pixel-wide line.

For a vertical separator the parent MUST have a bounded height (a flex row with intrinsic height, or an explicit `h-*` class).

---

## 4. AspectRatio

### Source-file imports (v4 registry)

```tsx
"use client"

import { AspectRatio as AspectRatioPrimitive } from "radix-ui"
```

### Exports

```tsx
export { AspectRatio }
```

### Function signature (verbatim)

```tsx
function AspectRatio({ ...props }: React.ComponentProps<typeof AspectRatioPrimitive.Root>)
```

Renders :

```tsx
<AspectRatioPrimitive.Root data-slot="aspect-ratio" {...props} />
```

### Props

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `ratio` | `number` | `1` | Width / height. Pass as expression : `16 / 9`, `4 / 3`, `1 / 1`, `9 / 16`. |
| `asChild` | `boolean` | `false` | Radix Slot composition. |
| `className` | `string` | `undefined` | Applied to the box. |

### How the ratio is enforced

Radix `AspectRatio` uses a `padding-bottom: calc(100% / ratio)` trick internally. The component renders :

- An outer wrapper with `position: relative`.
- An inner spacer with `padding-bottom`.
- An absolutely-positioned slot for children.

Children render in an `absolute; inset: 0` slot. You don't add `inset-0` yourself unless you wrap children in your own positioned div. Direct media elements (`<img>`, `<video>`, Next.js `<Image fill>`) typically use `className="size-full object-cover"` to fill the slot.

---

## v3 -> v4 props summary (all four primitives)

| Primitive | v3 prop | v4 prop |
|-----------|---------|---------|
| Resizable PanelGroup | `direction` | `orientation` |
| Resizable PanelGroup | `onLayout` | `onLayoutChange` |
| ScrollArea / Separator / AspectRatio | (no renames) | (no renames) |

The Tailwind v4 / `radix-ui` consolidation does not introduce new prop renames for these primitives beyond the Resizable two listed above.
