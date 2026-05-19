# Layout Primitives : Working Examples

All examples are version-explicit : shadcn ui evergreen-2026, Tailwind v4 (oklch theme tokens, `bg-background` class), React 18+ / 19, react-resizable-panels v4, unified `radix-ui` package. Copy-paste-runnable.

---

## Example 1 : Horizontal Resizable split pane (editor + preview)

A two-pane layout : code editor on the left (50%), preview on the right (50%). User drags the handle to redistribute width. Min width per pane : 25%.

```tsx
"use client"

import {
  ResizablePanelGroup,
  ResizablePanel,
  ResizableHandle,
} from "@/components/ui/resizable"

export function EditorPreview() {
  return (
    <ResizablePanelGroup
      orientation="horizontal"
      autoSaveId="editor-preview"
      className="min-h-[480px] rounded-lg border"
    >
      <ResizablePanel defaultSize={50} minSize={25}>
        <div className="flex h-full items-center justify-center p-6">
          <span className="font-mono text-sm">// editor</span>
        </div>
      </ResizablePanel>
      <ResizableHandle withHandle />
      <ResizablePanel defaultSize={50} minSize={25}>
        <div className="flex h-full items-center justify-center p-6">
          <span className="text-sm">preview</span>
        </div>
      </ResizablePanel>
    </ResizablePanelGroup>
  )
}
```

Notes :
- The outer Group has `min-h-[480px]` ; without a bounded height the group is 0 tall and the handle is unreachable.
- `autoSaveId="editor-preview"` persists the layout to localStorage. Reload restores the user's last drag position.
- `withHandle` on the handle renders the visible grip icon ; without it the handle is an invisible 1-pixel line.

---

## Example 2 : Vertical Resizable with a nested horizontal group (IDE layout)

A top region (file tree + main pane) and a bottom region (terminal). The top region itself splits horizontally. Two distinct `autoSaveId` values because the groups are independent.

```tsx
"use client"

import {
  ResizablePanelGroup,
  ResizablePanel,
  ResizableHandle,
} from "@/components/ui/resizable"

export function IdeLayout() {
  return (
    <ResizablePanelGroup
      orientation="vertical"
      autoSaveId="ide-outer"
      className="h-[600px] rounded-lg border"
    >
      <ResizablePanel defaultSize={70} minSize={40}>
        <ResizablePanelGroup
          orientation="horizontal"
          autoSaveId="ide-top"
          className="h-full"
        >
          <ResizablePanel defaultSize={25} minSize={15} maxSize={40}>
            <div className="p-4 text-sm">file tree</div>
          </ResizablePanel>
          <ResizableHandle />
          <ResizablePanel defaultSize={75}>
            <div className="p-4 text-sm">main editor</div>
          </ResizablePanel>
        </ResizablePanelGroup>
      </ResizablePanel>
      <ResizableHandle withHandle />
      <ResizablePanel
        defaultSize={30}
        minSize={10}
        collapsible
        collapsedSize={0}
      >
        <div className="p-4 font-mono text-sm">$ terminal</div>
      </ResizablePanel>
    </ResizablePanelGroup>
  )
}
```

Notes :
- Outer group is `orientation="vertical"`. Inner group inside the top panel is `orientation="horizontal"`. They are separate Groups, so drag events do not leak.
- Distinct `autoSaveId` values : `"ide-outer"` and `"ide-top"`. NEVER share an `autoSaveId` across groups ; localStorage state collides.
- The terminal panel is `collapsible` with `collapsedSize={0}` ; a user can drag it shut and the `onCollapse` hook can update toolbar state.
- File tree panel is bounded by `minSize={15}` + `maxSize={40}` so the user cannot drag it to extremes.

---

## Example 3 : ScrollArea with a horizontal ScrollBar (horizontally-scrolling thumbnail row)

A row of items wider than the viewport. The vertical bar is suppressed naturally because the content fits vertically ; the horizontal bar is composed manually because the shadcn `ScrollArea` only renders a vertical bar internally.

```tsx
import { ScrollArea, ScrollBar } from "@/components/ui/scroll-area"

const works = [
  { id: 1, title: "Photo 1" },
  { id: 2, title: "Photo 2" },
  { id: 3, title: "Photo 3" },
  { id: 4, title: "Photo 4" },
  { id: 5, title: "Photo 5" },
]

export function ThumbnailRow() {
  return (
    <ScrollArea className="w-96 whitespace-nowrap rounded-md border">
      <div className="flex w-max gap-4 p-4">
        {works.map((w) => (
          <figure key={w.id} className="shrink-0">
            <div className="size-40 rounded-md bg-muted" />
            <figcaption className="pt-2 text-xs text-muted-foreground">
              {w.title}
            </figcaption>
          </figure>
        ))}
      </div>
      <ScrollBar orientation="horizontal" />
    </ScrollArea>
  )
}
```

Notes :
- `w-96` on `ScrollArea` constrains the viewport.
- The inner row uses `w-max` so its intrinsic width grows past 96 units ; the overflow is what triggers the scrollbar.
- `<ScrollBar orientation="horizontal" />` is REQUIRED for horizontal scroll. The internal vertical bar still renders but stays invisible because the content does not overflow vertically.
- `whitespace-nowrap` on ScrollArea is a defensive style ; the `w-max` flex row already prevents wrapping.

---

## Example 4 : Separator orientation contrast (vertical and horizontal in one block)

The same data shown two ways : horizontal Separator dividing stack blocks, vertical Separator dividing inline labels.

```tsx
import { Separator } from "@/components/ui/separator"

export function SeparatorContrast() {
  return (
    <div className="w-80">
      <div>
        <h4 className="text-sm font-medium">Radix Primitives</h4>
        <p className="text-sm text-muted-foreground">
          An open-source UI component library.
        </p>
      </div>
      <Separator className="my-4" />
      <div className="flex h-5 items-center gap-4 text-sm">
        <span>Blog</span>
        <Separator orientation="vertical" />
        <span>Docs</span>
        <Separator orientation="vertical" />
        <span>Source</span>
      </div>
    </div>
  )
}
```

Notes :
- `<Separator className="my-4" />` uses the default `orientation="horizontal"` ; renders as `h-px w-full` and lives in normal flow with vertical margin.
- `<Separator orientation="vertical" />` inside a `flex` row renders as `h-full w-px`. The row MUST have a bounded height ; `h-5` on the flex container gives the vertical line a height to stretch to.
- Both separators are `decorative={true}` by default (visual only ; not announced by screen readers). For semantic structural separators set `decorative={false}`.

---

## Example 5 : AspectRatio 16/9 video container

A YouTube-style embed that stays 16/9 regardless of container width.

```tsx
import { AspectRatio } from "@/components/ui/aspect-ratio"

export function VideoEmbed() {
  return (
    <div className="w-full max-w-2xl">
      <AspectRatio ratio={16 / 9} className="bg-muted rounded-md overflow-hidden">
        <iframe
          src="https://www.youtube-nocookie.com/embed/dQw4w9WgXcQ"
          title="embedded video"
          allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
          allowFullScreen
          className="size-full"
        />
      </AspectRatio>
    </div>
  )
}
```

Notes :
- Outer wrapper has `max-w-2xl` ; AspectRatio derives the height from `width * (9 / 16)`.
- `ratio={16 / 9}` is the JS division expression (1.777...), not the string `"16:9"`.
- The `<iframe>` uses `className="size-full"` to fill the absolutely-positioned slot Radix creates. Radix already positions children as `absolute; inset: 0` ; you only need to make them stretch.
- `overflow-hidden` on AspectRatio clips iframe corners so they follow the `rounded-md` radius.

---

## Example 6 : AspectRatio 1/1 avatar grid

A grid of square avatars built from arbitrary images of mixed intrinsic dimensions. Each cell uses AspectRatio + `object-cover`.

```tsx
import { AspectRatio } from "@/components/ui/aspect-ratio"

const team = [
  { name: "Ada", src: "/team/ada.jpg" },
  { name: "Linus", src: "/team/linus.jpg" },
  { name: "Grace", src: "/team/grace.jpg" },
  { name: "Donald", src: "/team/donald.jpg" },
]

export function AvatarGrid() {
  return (
    <ul className="grid grid-cols-4 gap-4">
      {team.map((p) => (
        <li key={p.name}>
          <AspectRatio ratio={1 / 1} className="overflow-hidden rounded-full bg-muted">
            <img src={p.src} alt={p.name} className="size-full object-cover" />
          </AspectRatio>
          <p className="mt-2 text-center text-sm">{p.name}</p>
        </li>
      ))}
    </ul>
  )
}
```

Notes :
- `ratio={1 / 1}` produces a square ; the grid track width drives the avatar size.
- `<img className="size-full object-cover">` crops the image to fill the square slot. Without `object-cover` the image distorts.
- `rounded-full` on AspectRatio + `overflow-hidden` clips the image to a circle.
- The grid layout (`grid-cols-4`) varies the cell width responsively ; AspectRatio keeps each cell square at every width.
