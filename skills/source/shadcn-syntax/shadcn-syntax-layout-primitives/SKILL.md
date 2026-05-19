---
name: shadcn-syntax-layout-primitives
description: >
  Use when adding, debugging, or refactoring any of the four shadcn ui
  layout-helper primitives : Resizable (drag-to-resize split panes),
  ScrollArea (custom-styled scrollable region with a Radix scrollbar),
  Separator (horizontal or vertical divider), or AspectRatio (lock a
  container to a width:height ratio like 16/9, 1/1, 9/16). Composes
  the right subcomponents per primitive (ResizablePanelGroup /
  ResizablePanel / ResizableHandle ; ScrollArea / ScrollBar ;
  Separator with orientation + decorative ; AspectRatio with ratio),
  wires their prop surfaces against the underlying libraries
  (react-resizable-panels v4 for Resizable, Radix UI for ScrollArea /
  Separator / AspectRatio), and explains the `"use client"` story
  per primitive (all four ship with `"use client"` in the v4 registry
  source, contrary to the "pure-styling = server-safe" rumor).
  Prevents the canonical layout-helper failures : rendering a
  ResizablePanel outside a ResizablePanelGroup so react-resizable-
  panels crashes ("Panel components must be rendered within a
  PanelGroup container"), giving a ScrollArea no explicit height so
  the viewport never overflows and the custom scrollbar never
  appears, omitting `orientation="vertical"` on a Separator inside a
  row layout so the default horizontal `h-px` div collapses to zero,
  putting an absolutely-positioned child inside an AspectRatio
  without `inset-0` so the ratio container is empty, and nesting
  multiple ResizablePanelGroups in the same handle-id namespace so
  drag state leaks across groups.
  Covers the four shadcn primitives verbatim from
  `apps/v4/registry/new-york-v4/ui/{resizable,scroll-area,separator,
  aspect-ratio}.tsx`, the v4 `orientation` rename on PanelGroup (no
  longer `direction`), the v4 `onLayoutChange` rename (no longer
  `onLayout`), Panel-level `collapsible` + `collapsedSize` +
  `onCollapse` + `onExpand`, the `autoSaveId` localStorage
  persistence pattern, `ResizableHandle withHandle` for the visible
  grip, the ScrollArea auto-vertical-scrollbar (no need to compose
  ScrollBar for vertical scroll, only for horizontal or when both
  axes are needed), the Radix Separator `decorative` accessibility
  flag (true = `role="none"`, false = `role="separator"`), and the
  AspectRatio inset-0 child pattern.
  Keywords: shadcn layout primitives, shadcn resizable, resizable
  panels, ResizablePanelGroup, ResizablePanel, ResizableHandle,
  split pane, splitter, drag to resize, react-resizable-panels,
  PanelGroup orientation, defaultSize, minSize, maxSize,
  collapsible panel, collapsedSize, autoSaveId, withHandle,
  shadcn scroll area, ScrollArea, ScrollBar, custom scrollbar,
  horizontal scrollbar, scroll area horizontal, Radix ScrollArea,
  scrollbar orientation, ScrollArea no scroll, ScrollArea fixed
  height, shadcn separator, Separator orientation horizontal,
  Separator orientation vertical, vertical divider, horizontal
  divider, decorative separator, role separator, shadcn aspect
  ratio, AspectRatio, 16/9 container, 1/1 square, 9/16 portrait,
  video container, image container, iframe container, responsive
  media, how do I make resizable panels, how do I add a vertical
  divider, how do I lock a video to 16:9, why is my scroll area
  not scrolling, why is my separator invisible, Panel must be
  rendered within a PanelGroup, ScrollArea no overflow, layout
  helpers, divider, split pane, splitter.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui : Layout Primitives

Four shadcn primitives ship as pure layout helpers. They hold no business state and expose no independent decision logic. They differ only in *which layout problem they solve* : drag-to-resize panes, custom-styled scroll, content divider, fixed width:height ratio. Pick by the layout problem and compose. This skill covers all four because none of them is large enough to warrant a dedicated skill, and the composition rules are isolated per primitive.

## Quick Reference

### The four primitives

| Primitive | Solves | Library | Subcomponents | `"use client"` in v4 source |
|-----------|--------|---------|---------------|------------------------------|
| `Resizable` | Drag-to-resize split panes | `react-resizable-panels` v4 | `ResizablePanelGroup`, `ResizablePanel`, `ResizableHandle` | YES |
| `ScrollArea` | Custom-styled scrollable region | Radix `ScrollArea` | `ScrollArea`, `ScrollBar` | YES |
| `Separator` | Horizontal or vertical divider | Radix `Separator` | `Separator` | YES |
| `AspectRatio` | Lock container to width:height ratio | Radix `AspectRatio` | `AspectRatio` | YES |

### Decision tree : Which primitive?

```
Q. What layout problem do I have?
   Two or more regions, user drags a handle to resize them   -> Resizable
   A scroll region that needs styled scrollbars (or a thin,
   always-visible scrollbar that matches the theme)          -> ScrollArea
   Visual line dividing two content blocks (in a list, in
   a stack, between toolbar groups)                          -> Separator
   Element whose visible height must be a fixed fraction of
   its width (video, image, iframe, map embed, canvas)       -> AspectRatio
```

ALWAYS pick the primitive from the *layout problem*. NEVER pick AspectRatio for a generic flex / grid layout problem ; it freezes the ratio and absolutely-positions the child, which fights flex / grid layouts.

### `"use client"` invariant

The v4 registry source for ALL FOUR primitives starts with `"use client"`. This is true even for `Separator` and `AspectRatio`, which appear stateless. The directive is in the source because the file uses Radix primitives whose internals call React client hooks. NEVER strip `"use client"` from `components/ui/{resizable,scroll-area,separator,aspect-ratio}.tsx` even if your linter suggests it is unused. Stripping it makes the file a Server Component, and Radix throws a "client hooks in server component" error at hydration.

### Five invariants

1. ALWAYS wrap every `ResizablePanel` inside a `ResizablePanelGroup`. NEVER render `ResizablePanel` at the root of a layout ; react-resizable-panels throws "Panel components must be rendered within a PanelGroup container".
2. ALWAYS give `ScrollArea` an explicit height (or `max-height`) via a Tailwind class such as `h-[200px]`, `h-72`, or `max-h-screen`. NEVER expect ScrollArea to scroll without a constrained height ; it just expands to its content.
3. ALWAYS pass `orientation="vertical"` on Separator inside a horizontal flex / row. NEVER rely on the default `orientation="horizontal"` for a vertical divider ; the default renders an `h-px w-full` div which collapses to zero inside a row.
4. ALWAYS give an AspectRatio child `className="absolute inset-0 ..."` if the child is positioned absolutely (or use `object-cover` / `object-contain` on an `<img>` / `<video>` to fill). NEVER leave a non-filling child inside AspectRatio ; the ratio container will appear empty.
5. ALWAYS pass distinct `autoSaveId` values to nested `ResizablePanelGroup` instances when both groups should persist layout. NEVER share an `autoSaveId` across two groups ; localStorage state collides and one group resets the other on mount.

## Resizable : drag-to-resize split panes

Wraps `react-resizable-panels` v4 (peer dependency installed automatically by `shadcn add resizable`). The library handles all drag math, keyboard resize, and ARIA wiring. shadcn adds Tailwind styling and a `withHandle` grip prop on the handle.

### Subcomponents

`ResizablePanelGroup`, `ResizablePanel`, `ResizableHandle`. Imports :

```tsx
import {
  ResizablePanelGroup,
  ResizablePanel,
  ResizableHandle,
} from "@/components/ui/resizable"
```

### PanelGroup props (v4, post-rename)

| Prop | Type | Notes |
|------|------|-------|
| `orientation` | `"horizontal" \| "vertical"` | Layout axis. **v4 rename** : was `direction` in v3. ALWAYS use `orientation` in v4. |
| `onLayoutChange` | `(sizes: number[]) => void` | Fires after layout changes. **v4 rename** : was `onLayout` in v3. |
| `autoSaveId` | `string` | localStorage key for layout persistence. Distinct per group. |
| `id` | `string` | Stable id for SSR layout-restoration when `autoSaveId` is set. |
| `keyboardResizeBy` | `number \| null` | Keyboard arrow resize step in pixels (default 10, null disables). |

### Panel props

| Prop | Type | Notes |
|------|------|-------|
| `defaultSize` | `number` | Initial percent (0-100). ALL panels' `defaultSize` should sum to ~100. |
| `minSize` | `number` | Minimum percent (default 0). Drag clamps to this floor. |
| `maxSize` | `number` | Maximum percent (default 100). Drag clamps to this ceiling. |
| `collapsible` | `boolean` | When true, the panel can collapse below `minSize` to `collapsedSize`. |
| `collapsedSize` | `number` | Percent the panel collapses to (default 0). |
| `onCollapse` | `() => void` | Fires when panel collapses. |
| `onExpand` | `() => void` | Fires when panel expands from collapsed. |
| `order` | `number` | Stable order for SSR / conditional rendering. |
| `id` | `string` | Stable id for SSR / conditional rendering. |

### Handle props

| Prop | Type | Notes |
|------|------|-------|
| `withHandle` | `boolean` | Render the visible grip icon (`GripVerticalIcon`). Default false (invisible drag line). |
| `disabled` | `boolean` | Disable drag interaction. |

### Minimal pattern : horizontal split

```tsx
"use client"

import {
  ResizablePanelGroup,
  ResizablePanel,
  ResizableHandle,
} from "@/components/ui/resizable"

<ResizablePanelGroup
  orientation="horizontal"
  className="min-h-[400px] max-w-md rounded-lg border"
>
  <ResizablePanel defaultSize={50} minSize={20}>
    <div className="flex h-full items-center justify-center p-6">Left</div>
  </ResizablePanel>
  <ResizableHandle withHandle />
  <ResizablePanel defaultSize={50} minSize={20}>
    <div className="flex h-full items-center justify-center p-6">Right</div>
  </ResizablePanel>
</ResizablePanelGroup>
```

The Group itself MUST have a bounded height (the example uses `min-h-[400px]`). Without a height bound, the group is 0 tall and the handle is unreachable.

## ScrollArea : custom-styled scrollable region

Wraps Radix `ScrollArea`. The shadcn `ScrollArea` composes `ScrollAreaPrimitive.Root` + `Viewport` + a default vertical `ScrollBar` + `Corner` in its source. You only need to add a `ScrollBar` element manually for horizontal scroll or to override the vertical default.

### Subcomponents

`ScrollArea`, `ScrollBar`. Imports :

```tsx
import { ScrollArea, ScrollBar } from "@/components/ui/scroll-area"
```

### ScrollArea props

Forwarded directly to `ScrollAreaPrimitive.Root` :

| Prop | Type | Notes |
|------|------|-------|
| `type` | `"auto" \| "always" \| "scroll" \| "hover"` | When scrollbars appear (default `"hover"`). |
| `scrollHideDelay` | `number` | ms to keep scrollbar visible after scroll stops (default 600). Only when `type` is `"scroll"` or `"hover"`. |
| `dir` | `"ltr" \| "rtl"` | Reading direction. Inherited from `<DirectionProvider>` by default. |

### ScrollBar props

| Prop | Type | Notes |
|------|------|-------|
| `orientation` | `"vertical" \| "horizontal"` | Default `"vertical"`. The shadcn `ScrollArea` already renders a vertical one ; add a horizontal `ScrollBar` as a child for horizontal scroll. |

### Minimal pattern : vertical scroll (default)

```tsx
import { ScrollArea } from "@/components/ui/scroll-area"

<ScrollArea className="h-72 w-48 rounded-md border">
  <div className="p-4">
    {tags.map((tag) => <div key={tag} className="text-sm">{tag}</div>)}
  </div>
</ScrollArea>
```

A vertical scrollbar appears automatically. NEVER add `<ScrollBar />` manually for the vertical case ; the shadcn `ScrollArea` already renders one in its source.

### Adding a horizontal ScrollBar

```tsx
<ScrollArea className="w-96 whitespace-nowrap rounded-md border">
  <div className="flex w-max gap-4 p-4">{children}</div>
  <ScrollBar orientation="horizontal" />
</ScrollArea>
```

The `ScrollBar orientation="horizontal"` is required for horizontal axis. The shadcn source only renders the vertical ScrollBar internally.

## Separator : horizontal or vertical divider

Wraps Radix `Separator`. Renders a styled div (`bg-border`) sized to `h-px w-full` (horizontal) or `h-full w-px` (vertical) via `data-orientation`. Defaults to `decorative={true}` (which renders as `role="none"`, omitted from the accessibility tree).

### Subcomponent

`Separator`. Import :

```tsx
import { Separator } from "@/components/ui/separator"
```

### Props

| Prop | Type | Notes |
|------|------|-------|
| `orientation` | `"horizontal" \| "vertical"` | Default `"horizontal"`. Choose by the *axis of the line*, not the parent layout. |
| `decorative` | `boolean` | Default `true`. When true, omitted from a11y tree (`role="none"`). When false, exposed as `role="separator"` and announced. |

### Minimal pattern : horizontal (in a stack)

```tsx
import { Separator } from "@/components/ui/separator"

<div>
  <h4>Radix Primitives</h4>
  <p className="text-sm text-muted-foreground">An open-source UI library.</p>
  <Separator className="my-4" />
  <div className="flex gap-4">
    <span>Blog</span>
    <Separator orientation="vertical" />
    <span>Docs</span>
    <Separator orientation="vertical" />
    <span>Source</span>
  </div>
</div>
```

The horizontal `<Separator className="my-4" />` divides two stack blocks. The vertical `<Separator orientation="vertical" />` between inline labels MUST be inside a parent with a bounded height (typically `flex` items which take the row's height automatically).

### `decorative` rule

- ALWAYS leave `decorative={true}` (the default) when the visual line is purely cosmetic (between two visually-related blocks, between toolbar groups).
- ALWAYS pass `decorative={false}` when the separator marks a *semantic* break (e.g., between distinct list groups in a screen-reader navigation). Then `role="separator"` is announced.

## AspectRatio : lock container to a width:height ratio

Wraps Radix `AspectRatio`. Computes a `padding-bottom` hack so the box height equals `(1 / ratio) * width`. Children are positioned absolutely inside the box. Use for video, image, iframe, map embed, canvas.

### Subcomponent

`AspectRatio`. Import :

```tsx
import { AspectRatio } from "@/components/ui/aspect-ratio"
```

### Props

| Prop | Type | Notes |
|------|------|-------|
| `ratio` | `number` | Width / height. Common : `16 / 9` (video), `1 / 1` (square avatar), `9 / 16` (portrait), `4 / 3` (legacy media), `21 / 9` (cinema). |

### Minimal pattern : 16/9 image

```tsx
import Image from "next/image"
import { AspectRatio } from "@/components/ui/aspect-ratio"

<div className="w-[450px]">
  <AspectRatio ratio={16 / 9} className="bg-muted rounded-md">
    <Image src="/photo.jpg" alt="" fill className="rounded-md object-cover" />
  </AspectRatio>
</div>
```

The outer wrapper (`w-[450px]`) sets the width ; AspectRatio derives the height. The Next.js `<Image>` with `fill` fills the absolutely-positioned slot. For a plain `<img>` or `<video>` use `className="size-full object-cover"` on the child.

### Children pattern

AspectRatio's internal box uses `position: relative` + a padding-bottom spacer. Children are stacked in the same coordinate space. ALWAYS make a positioned child fill the box, either with :

- `fill` (Next.js `<Image>`), or
- `className="size-full object-cover"` (plain `<img>`, `<video>`), or
- `className="absolute inset-0"` (for a generic positioned wrapper).

Without one of these, the child sits at its natural size in the top-left and the ratio container appears empty.

## When to use which

| Want | Primitive |
|------|-----------|
| Two-panel editor / preview / sidebar that the user can drag-resize | `Resizable` |
| Long list constrained to a fixed height with a styled scrollbar | `ScrollArea` |
| Theme-aware divider between blocks in a stack | `Separator` (horizontal) |
| Theme-aware divider between inline items in a row | `Separator` (vertical) |
| Video / iframe / image that stays 16:9 (or any ratio) regardless of width | `AspectRatio` |
| Square thumbnail / avatar with consistent height in a grid | `AspectRatio` (1/1) |

NEVER reach for `Resizable` when CSS `resize: both` on a textarea is enough ; Resizable is for multi-panel application chrome, not for single-element resize affordances.

NEVER reach for `AspectRatio` for an `<img>` that already has intrinsic dimensions and is rendered with `width` / `height` HTML attributes ; the browser holds the ratio for you. AspectRatio is for cases where the *container* needs the ratio (positioned children, background images, iframes, videos with unknown intrinsic size).

## Companion Skills

- `shadcn-impl-rsc-vs-client-boundaries` for the full `"use client"` decision rules across the catalog.
- `shadcn-syntax-sidebar` for application chrome that pairs with Resizable for sidebar+content split layouts.
- `shadcn-syntax-table` for the inner content of a horizontally-scrolled ScrollArea (responsive wide-table pattern).
- `shadcn-errors-styling-conflicts` for `cn()` and Tailwind merge rules that apply to every primitive's `className` prop.

## References

- `references/methods.md` : per-primitive composition + full prop signatures verbatim from the v4 registry source.
- `references/examples.md` : six working examples (horizontal Resizable, vertical Resizable + nested groups, ScrollArea with horizontal ScrollBar, Separator orientation contrast, AspectRatio 16/9 video, AspectRatio 1/1 avatar grid).
- `references/anti-patterns.md` : five canonical failures (Panel without PanelGroup, ScrollArea without height, Separator without orientation, AspectRatio child without inset-0, nested Resizable groups sharing autoSaveId).
