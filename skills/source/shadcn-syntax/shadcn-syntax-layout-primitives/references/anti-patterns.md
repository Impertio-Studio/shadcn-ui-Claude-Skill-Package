# Layout Primitives : Anti-Patterns

Six canonical failures observed in real shadcn / react-resizable-panels / Radix usage, plus three subtle pitfalls. Each entry : what NOT to write, what to write instead, and WHY the wrong version fails.

---

## 1. `ResizablePanel` rendered outside a `ResizablePanelGroup`

### Wrong

```tsx
"use client"

import { ResizablePanel, ResizableHandle } from "@/components/ui/resizable"

export function Broken() {
  return (
    <div className="flex h-96">
      <ResizablePanel defaultSize={50}>Left</ResizablePanel>
      <ResizableHandle />
      <ResizablePanel defaultSize={50}>Right</ResizablePanel>
    </div>
  )
}
```

### Right

```tsx
"use client"

import {
  ResizablePanelGroup,
  ResizablePanel,
  ResizableHandle,
} from "@/components/ui/resizable"

export function Fixed() {
  return (
    <ResizablePanelGroup orientation="horizontal" className="h-96">
      <ResizablePanel defaultSize={50}>Left</ResizablePanel>
      <ResizableHandle />
      <ResizablePanel defaultSize={50}>Right</ResizablePanel>
    </ResizablePanelGroup>
  )
}
```

### Why

react-resizable-panels stores resize state, drag handlers, and ARIA wiring on a Group context that PanelGroup mounts. A bare `<div className="flex">` creates no such context. `react-resizable-panels` throws : *"Panel components must be rendered within a PanelGroup container"*. The error is loud at runtime but easy to miss in dev because the message is in the console, not the rendered tree. ALWAYS pair `ResizablePanel` with a `ResizablePanelGroup` parent.

---

## 2. `ScrollArea` without an explicit height

### Wrong

```tsx
import { ScrollArea } from "@/components/ui/scroll-area"

export function NoScroll() {
  return (
    <ScrollArea className="w-72 rounded-md border">
      {longList.map((item) => (
        <div key={item.id} className="p-2">{item.label}</div>
      ))}
    </ScrollArea>
  )
}
```

### Right

```tsx
import { ScrollArea } from "@/components/ui/scroll-area"

export function Scroll() {
  return (
    <ScrollArea className="h-72 w-72 rounded-md border">
      {longList.map((item) => (
        <div key={item.id} className="p-2">{item.label}</div>
      ))}
    </ScrollArea>
  )
}
```

### Why

ScrollArea wraps Radix `ScrollAreaPrimitive.Root` + `Viewport`. The Viewport is `size-full` (i.e. `100% / 100%` of its parent). Without a height constraint on the Root, the Root expands to fit content, the Viewport matches that height, there is no overflow, and the scrollbar never renders. ALWAYS give ScrollArea an explicit `h-*` or `max-h-*` class. Use `max-h-screen`, `max-h-[80vh]`, or a fixed `h-72` depending on the layout.

---

## 3. `Separator` without `orientation="vertical"` inside a row

### Wrong

```tsx
import { Separator } from "@/components/ui/separator"

export function InvisibleDivider() {
  return (
    <div className="flex h-5 items-center gap-4 text-sm">
      <span>Blog</span>
      <Separator />
      <span>Docs</span>
    </div>
  )
}
```

### Right

```tsx
import { Separator } from "@/components/ui/separator"

export function VerticalDivider() {
  return (
    <div className="flex h-5 items-center gap-4 text-sm">
      <span>Blog</span>
      <Separator orientation="vertical" />
      <span>Docs</span>
    </div>
  )
}
```

### Why

The default `orientation="horizontal"` triggers the className `data-[orientation=horizontal]:h-px data-[orientation=horizontal]:w-full`. Inside a flex row this renders a 1-pixel-tall, full-width line that fills the entire row's width and pushes "Docs" to the next line (or, with `flex-shrink`, collapses to zero width). The user sees no divider. ALWAYS pass `orientation="vertical"` when the separator is between inline items. Verify the parent has bounded height (`h-5`, `h-full` inside an outer container, etc.) so `data-[orientation=vertical]:h-full` has a height to stretch to.

---

## 4. `AspectRatio` with a child that does not fill the slot

### Wrong

```tsx
import { AspectRatio } from "@/components/ui/aspect-ratio"

export function EmptyRatio() {
  return (
    <AspectRatio ratio={16 / 9} className="bg-muted">
      <img src="/hero.jpg" alt="" className="rounded-md" />
    </AspectRatio>
  )
}
```

### Right

```tsx
import { AspectRatio } from "@/components/ui/aspect-ratio"

export function FilledRatio() {
  return (
    <AspectRatio ratio={16 / 9} className="bg-muted">
      <img
        src="/hero.jpg"
        alt=""
        className="size-full rounded-md object-cover"
      />
    </AspectRatio>
  )
}
```

If a wrapper div is needed instead of a media element :

```tsx
<AspectRatio ratio={16 / 9} className="bg-muted">
  <div className="absolute inset-0 grid place-items-center">centered text</div>
</AspectRatio>
```

### Why

Radix `AspectRatio` renders an absolutely-positioned slot at `inset: 0` for its children. Children are stacked in that absolute coordinate space, NOT in normal flow. An `<img>` with intrinsic dimensions but no `size-full` (or `width="100%" height="100%"`) sits at its natural size in the top-left of the slot. The ratio box itself is correctly sized, but only the small image is visible, leaving a "broken" appearance. ALWAYS make the child fill : `size-full object-cover` (or `object-contain`) on media, `absolute inset-0` on a generic wrapper, `fill` on Next.js `<Image>`.

---

## 5. Multiple nested Resizable groups sharing one `autoSaveId`

### Wrong

```tsx
"use client"

import {
  ResizablePanelGroup,
  ResizablePanel,
  ResizableHandle,
} from "@/components/ui/resizable"

export function Collision() {
  return (
    <ResizablePanelGroup orientation="vertical" autoSaveId="layout">
      <ResizablePanel defaultSize={70}>
        <ResizablePanelGroup orientation="horizontal" autoSaveId="layout">
          <ResizablePanel defaultSize={25}>tree</ResizablePanel>
          <ResizableHandle />
          <ResizablePanel defaultSize={75}>main</ResizablePanel>
        </ResizablePanelGroup>
      </ResizablePanel>
      <ResizableHandle />
      <ResizablePanel defaultSize={30}>terminal</ResizablePanel>
    </ResizablePanelGroup>
  )
}
```

### Right

```tsx
<ResizablePanelGroup orientation="vertical" autoSaveId="ide-outer">
  <ResizablePanel defaultSize={70}>
    <ResizablePanelGroup orientation="horizontal" autoSaveId="ide-top">
      ...
    </ResizablePanelGroup>
  </ResizablePanel>
  <ResizableHandle />
  <ResizablePanel defaultSize={30}>terminal</ResizablePanel>
</ResizablePanelGroup>
```

### Why

`autoSaveId` is the localStorage key under which `react-resizable-panels` persists each Group's panel sizes. Two Groups sharing the same key both write to the same slot on every drag. On reload, both Groups try to restore from the same payload, but each Group has a *different* number of panels and panel orientation, so one Group's restoration overwrites the other's, producing unstable sizes that jump on mount. ALWAYS give each Group a unique `autoSaveId` (`"ide-outer"`, `"ide-top"`, `"settings-pane"` etc.). If a Group is purely ephemeral and should not persist, OMIT `autoSaveId` entirely.

---

## 6. Stripping `"use client"` from a layout-primitive file

### Wrong

```tsx
// components/ui/separator.tsx

import * as React from "react"
import { Separator as SeparatorPrimitive } from "radix-ui"

import { cn } from "@/lib/utils"

function Separator({ /* ... */ }) { /* ... */ }

export { Separator }
```

### Right

```tsx
// components/ui/separator.tsx

"use client"

import * as React from "react"
import { Separator as SeparatorPrimitive } from "radix-ui"

import { cn } from "@/lib/utils"

function Separator({ /* ... */ }) { /* ... */ }

export { Separator }
```

### Why

It looks like Separator and AspectRatio are stateless and should be Server-Component-safe. They are not. The Radix `Separator` and `AspectRatio` primitives import React client hooks (for ref forwarding, context, and SSR-safe id generation in Radix v2+). Without `"use client"`, Next.js / React Server Components treat the file as a Server Component, and hydration fails with *"useRef / useId / useContext is not allowed in a Server Component"*. The shadcn v4 registry ships ALL FOUR layout primitives with `"use client"` at the top of the file precisely for this reason. NEVER strip it on the assumption that "pure styling = server-safe".

---

## Subtle pitfall A : ResizablePanelGroup without a height

```tsx
<ResizablePanelGroup orientation="horizontal" className="rounded-lg border">
  <ResizablePanel>Left</ResizablePanel>
  <ResizableHandle />
  <ResizablePanel>Right</ResizablePanel>
</ResizablePanelGroup>
```

The default className `flex h-full w-full` resolves to `h-full` of nothing. The group is 0px tall. The handle is technically there but has no clickable area. ALWAYS bound the height : `className="min-h-[400px]"`, `className="h-screen"`, or place the group inside a parent that has a bounded height.

---

## Subtle pitfall B : v3 `direction` prop on PanelGroup

```tsx
// v3 (DEPRECATED in v4)
<ResizablePanelGroup direction="horizontal">...</ResizablePanelGroup>
```

Use `orientation` in v4. `direction` is silently ignored in v4 (the panel group defaults to horizontal). Layouts that worked in v3 with `direction="vertical"` now render horizontal until the prop is renamed. ALWAYS use `orientation` post-v4. Same story for `onLayout` -> `onLayoutChange`.

---

## Subtle pitfall C : composing a manual vertical ScrollBar on shadcn ScrollArea

```tsx
<ScrollArea className="h-72 w-48">
  <ScrollBar orientation="vertical" />
  ...
</ScrollArea>
```

The shadcn `ScrollArea` source already renders `<ScrollBar />` (default `orientation="vertical"`) internally. Adding a second vertical ScrollBar produces two stacked thumbs that fight each other on drag. ONLY add a manual `<ScrollBar />` for horizontal scroll (`<ScrollBar orientation="horizontal" />`). For vertical scroll, the internal bar is sufficient.

---

## Subtle pitfall D : `decorative={false}` on a purely visual line

```tsx
<Separator decorative={false} className="my-4" />
```

Setting `decorative={false}` exposes `role="separator"` to screen readers. The separator is announced (e.g. "separator") between content blocks. For a purely visual line this is noise pollution. ALWAYS leave `decorative={true}` (the default) unless the divider marks a structural / semantic break that screen-reader users need to perceive (e.g. between distinct list groups in a navigation menu).
