# Examples : Popover, Tooltip, HoverCard recipes

Working code, copy-paste safe. Every snippet assumes shadcn ui new-york-v4 with components installed via `pnpm dlx shadcn@latest add {popover|tooltip|hover-card}`. All snippets target Tailwind v4.

## 1. Popover : form inside (filter popover)

A classic table-column filter popover. Inputs ARE interactive, hence Popover, not Tooltip / HoverCard.

```tsx
"use client"

import * as React from "react"
import {
  Popover,
  PopoverTrigger,
  PopoverContent,
} from "@/components/ui/popover"
import { Button } from "@/components/ui/button"
import { Input } from "@/components/ui/input"
import { Label } from "@/components/ui/label"

export function ColumnFilterPopover() {
  const [open, setOpen] = React.useState(false)
  const [min, setMin] = React.useState("")
  const [max, setMax] = React.useState("")

  function onApply() {
    // ...apply filter to table state
    setOpen(false)
  }

  return (
    <Popover open={open} onOpenChange={setOpen}>
      <PopoverTrigger asChild>
        <Button variant="outline" size="sm">
          Filter price
        </Button>
      </PopoverTrigger>
      <PopoverContent className="w-72" align="start">
        <div className="grid gap-3">
          <div className="grid gap-2">
            <Label htmlFor="min">Min</Label>
            <Input
              id="min"
              type="number"
              value={min}
              onChange={(e) => setMin(e.target.value)}
            />
          </div>
          <div className="grid gap-2">
            <Label htmlFor="max">Max</Label>
            <Input
              id="max"
              type="number"
              value={max}
              onChange={(e) => setMax(e.target.value)}
            />
          </div>
          <Button onClick={onApply}>Apply</Button>
        </div>
      </PopoverContent>
    </Popover>
  )
}
```

Notes :
- Controlled (`open` + `onOpenChange`) because the Apply button must programmatically close the popover.
- `align="start"` left-aligns the popover with the trigger (typical for column filters).
- Focus moves into the first Input on open (Popover default focus behaviour).
- `modal` left as default `false`; users can click out / press Escape to close.

## 2. Tooltip : icon button hint

The canonical Tooltip use case : an icon-only button needs a label for sighted hover users AND screen readers.

```tsx
// app/layout.tsx (Next.js App Router)
import { TooltipProvider } from "@/components/ui/tooltip"

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <TooltipProvider>{children}</TooltipProvider>
      </body>
    </html>
  )
}
```

```tsx
// app/components/save-button.tsx
"use client"

import { Save } from "lucide-react"
import {
  Tooltip,
  TooltipTrigger,
  TooltipContent,
} from "@/components/ui/tooltip"
import { Button } from "@/components/ui/button"

export function SaveButton({ onSave }: { onSave: () => void }) {
  return (
    <Tooltip>
      <TooltipTrigger asChild>
        <Button
          size="icon"
          variant="ghost"
          aria-label="Save document"
          onClick={onSave}
        >
          <Save />
        </Button>
      </TooltipTrigger>
      <TooltipContent side="bottom">
        Save document (Ctrl+S)
      </TooltipContent>
    </Tooltip>
  )
}
```

Notes :
- `aria-label="Save document"` on the Button is the accessible name; the Tooltip is a redundant visual hint.
- `<TooltipTrigger asChild>` merges trigger props onto the Button (NEVER wrap a Button inside an extra `<button>` Trigger; that double-button breaks accessibility).
- The Tooltip is hidden on touch devices; that is acceptable here because the icon + `aria-label` already communicates the action to mobile users.

## 3. HoverCard : user profile preview on @mention

Rich preview content with a follow button : interactive enough to need HoverCard (not Tooltip), but trigger-on-hover (not click), and trigger IS keyboard-reachable (anchor with href).

```tsx
"use client"

import {
  HoverCard,
  HoverCardTrigger,
  HoverCardContent,
} from "@/components/ui/hover-card"
import { Avatar, AvatarImage, AvatarFallback } from "@/components/ui/avatar"
import { Button } from "@/components/ui/button"
import { CalendarDays } from "lucide-react"

export function MentionPreview({ handle }: { handle: string }) {
  return (
    <HoverCard openDelay={400} closeDelay={200}>
      <HoverCardTrigger asChild>
        <a
          href={`/users/${handle}`}
          className="font-medium text-primary underline-offset-2 hover:underline"
        >
          @{handle}
        </a>
      </HoverCardTrigger>
      <HoverCardContent className="w-80">
        <div className="flex justify-between gap-4">
          <Avatar>
            <AvatarImage src={`/avatars/${handle}.png`} />
            <AvatarFallback>
              {handle.slice(0, 2).toUpperCase()}
            </AvatarFallback>
          </Avatar>
          <div className="space-y-1">
            <h4 className="text-sm font-semibold">@{handle}</h4>
            <p className="text-sm">
              Building deterministic Claude skills for the open AEC stack.
            </p>
            <div className="flex items-center pt-2">
              <CalendarDays className="mr-2 h-4 w-4 opacity-70" />
              <span className="text-xs text-muted-foreground">
                Joined December 2024
              </span>
            </div>
          </div>
        </div>
      </HoverCardContent>
    </HoverCard>
  )
}
```

Notes :
- `openDelay={400}` faster than Radix default `700` to feel responsive in dense mention lists.
- `closeDelay={200}` shorter than Radix default `300` to avoid lingering panels.
- Trigger is an `<a href>` : keyboard users can Tab to it; focus opens the card.
- Tap behaviour : on mobile the user taps the link, the link navigates (HoverCard opens momentarily). Acceptable because the preview is supplementary.

## 4. Controlled Popover : closing on save (cross-state with parent)

```tsx
"use client"

import * as React from "react"
import {
  Popover,
  PopoverTrigger,
  PopoverContent,
} from "@/components/ui/popover"
import { Button } from "@/components/ui/button"
import { Input } from "@/components/ui/input"

export function RenamePopover({
  initialName,
  onSave,
}: {
  initialName: string
  onSave: (next: string) => Promise<void>
}) {
  const [open, setOpen] = React.useState(false)
  const [name, setName] = React.useState(initialName)
  const [saving, setSaving] = React.useState(false)

  async function commit() {
    setSaving(true)
    try {
      await onSave(name)
      setOpen(false)
    } finally {
      setSaving(false)
    }
  }

  return (
    <Popover open={open} onOpenChange={setOpen}>
      <PopoverTrigger asChild>
        <Button variant="outline">Rename</Button>
      </PopoverTrigger>
      <PopoverContent className="w-64">
        <div className="grid gap-2">
          <Input
            value={name}
            onChange={(e) => setName(e.target.value)}
            disabled={saving}
          />
          <Button onClick={commit} disabled={saving}>
            {saving ? "Saving..." : "Save"}
          </Button>
        </div>
      </PopoverContent>
    </Popover>
  )
}
```

Notes :
- Controlled (`open` + `onOpenChange`) so `commit()` can close after the async save resolves.
- ALWAYS keep `onOpenChange={setOpen}` even when programmatically closing. Without it, Escape and outside-click stop closing the popover.

## 5. Tooltip with per-instance delayDuration override

The shadcn `TooltipProvider` defaults to `delayDuration={0}` (instant). Override per-instance for a "help text" tooltip where instant feels jumpy.

```tsx
"use client"

import { HelpCircle } from "lucide-react"
import {
  Tooltip,
  TooltipTrigger,
  TooltipContent,
} from "@/components/ui/tooltip"

export function FieldHelpHint({ text }: { text: string }) {
  return (
    <Tooltip delayDuration={500}>
      <TooltipTrigger asChild>
        <button
          type="button"
          aria-label="Field help"
          className="inline-flex items-center text-muted-foreground hover:text-foreground"
        >
          <HelpCircle className="h-4 w-4" />
        </button>
      </TooltipTrigger>
      <TooltipContent side="right" className="max-w-xs">
        {text}
      </TooltipContent>
    </Tooltip>
  )
}
```

Notes :
- `delayDuration={500}` shadows the Provider's `0` for this one instance.
- `side="right"` keeps the hint clear of the input below.
- `max-w-xs` enables wrapping for longer hints (the shadcn default uses `text-balance`).

## 6. Nested Popovers (two layers)

Legitimate use case : a Popover containing a multi-select that itself opens a sub-Popover for "advanced filters". Both Popovers must portal independently and z-index must stack.

```tsx
"use client"

import * as React from "react"
import {
  Popover,
  PopoverTrigger,
  PopoverContent,
} from "@/components/ui/popover"
import { Button } from "@/components/ui/button"

export function NestedFiltersPopover() {
  const [outer, setOuter] = React.useState(false)
  const [inner, setInner] = React.useState(false)

  return (
    <Popover open={outer} onOpenChange={setOuter}>
      <PopoverTrigger asChild>
        <Button variant="outline">Filters</Button>
      </PopoverTrigger>
      <PopoverContent className="w-80">
        <div className="grid gap-3">
          <p className="text-sm font-medium">Quick filters</p>
          {/* ...quick filter UI */}
          <Popover open={inner} onOpenChange={setInner}>
            <PopoverTrigger asChild>
              <Button variant="ghost" size="sm">
                Advanced...
              </Button>
            </PopoverTrigger>
            <PopoverContent className="w-72 z-[60]" side="right" align="start">
              <p className="text-sm">Advanced filter content here.</p>
            </PopoverContent>
          </Popover>
        </div>
      </PopoverContent>
    </Popover>
  )
}
```

Notes :
- Both Popovers are controlled to allow programmatic close after Apply.
- The inner `PopoverContent` overrides `z-[60]` to render ABOVE the outer (`z-50` default).
- `side="right" align="start"` flips the inner panel to the side, avoiding overlap with the outer.
- ALWAYS keep `modal={false}` (the default) on the outer popover; with `modal={true}`, the focus trap prevents the inner Popover's Trigger from receiving focus.

## 7. Anti-recipe converted to right-recipe : Tooltip with link converted to HoverCard

Wrong (Tooltip with interactive content) :

```tsx
// WRONG : the link is unreachable on keyboard and hidden on touch.
<Tooltip>
  <TooltipTrigger asChild>
    <span>@freek</span>
  </TooltipTrigger>
  <TooltipContent>
    See <a href="/users/freek">profile</a>
  </TooltipContent>
</Tooltip>
```

Right (HoverCard with focusable trigger) :

```tsx
<HoverCard>
  <HoverCardTrigger asChild>
    <a href="/users/freek">@freek</a>
  </HoverCardTrigger>
  <HoverCardContent>
    Profile preview content...
  </HoverCardContent>
</HoverCard>
```

The HoverCard trigger MUST be focusable (anchor satisfies this); the content is allowed to host interactive elements; the panel is reachable by keyboard users via Tab + Enter on the trigger.

## 8. Disabled-button Tooltip workaround

Disabled buttons swallow pointer events; the Tooltip trigger never receives `pointerEnter`. Wrap a focusable `<span>` (or a wrapping div with `tabIndex={0}`) to make the trigger receive the hover.

```tsx
"use client"

import {
  Tooltip,
  TooltipTrigger,
  TooltipContent,
} from "@/components/ui/tooltip"
import { Button } from "@/components/ui/button"

export function DisabledPublishButton() {
  return (
    <Tooltip>
      <TooltipTrigger asChild>
        <span tabIndex={0}>
          <Button disabled>Publish</Button>
        </span>
      </TooltipTrigger>
      <TooltipContent>
        Add a title before publishing.
      </TooltipContent>
    </Tooltip>
  )
}
```

Notes :
- `tabIndex={0}` makes the wrapper focusable for keyboard users.
- ALWAYS prefer enabling the button + showing a Sonner toast on click for accessibility-critical disabled states; the wrapping-span trick is a last-resort visual hint.

## 9. PopoverAnchor : panel anchored to row, trigger inside row

```tsx
"use client"

import { MoreHorizontal } from "lucide-react"
import {
  Popover,
  PopoverAnchor,
  PopoverTrigger,
  PopoverContent,
} from "@/components/ui/popover"
import { Button } from "@/components/ui/button"

export function RowMenu({ row }: { row: { id: string; name: string } }) {
  return (
    <Popover>
      <PopoverAnchor asChild>
        <tr>
          <td>{row.name}</td>
          <td className="text-right">
            <PopoverTrigger asChild>
              <Button size="icon" variant="ghost" aria-label="Row actions">
                <MoreHorizontal />
              </Button>
            </PopoverTrigger>
          </td>
        </tr>
      </PopoverAnchor>
      <PopoverContent align="end" className="w-48">
        Row-level actions...
      </PopoverContent>
    </Popover>
  )
}
```

Notes :
- The `PopoverContent` anchors to the entire `<tr>` (the Anchor), not the small icon button.
- Useful when the visual relationship "this panel belongs to this row" is clearer than "this panel belongs to this small icon".
