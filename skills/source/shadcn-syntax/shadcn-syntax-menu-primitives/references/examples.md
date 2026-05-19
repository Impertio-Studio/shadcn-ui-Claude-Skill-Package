# Menu Primitives : Canonical Examples

Six recipes covering the most common menu patterns across shadcn ui. All examples assume the file containing the JSX has `"use client"` at the top (or is imported into a client boundary). The component files themselves (`components/ui/{dropdown-menu,context-menu,menubar,navigation-menu}.tsx`) already declare `"use client"` per the v4 registry.

## 1. DropdownMenu : table-row action menu

The canonical use : a `...` icon in the last column of a table row, opening a list of row actions. Three actions plus a destructive Delete, separated.

```tsx
"use client"

import {
  DropdownMenu, DropdownMenuTrigger, DropdownMenuContent,
  DropdownMenuItem, DropdownMenuSeparator, DropdownMenuShortcut,
} from "@/components/ui/dropdown-menu"
import { Button } from "@/components/ui/button"
import { MoreHorizontalIcon } from "lucide-react"

function RowActions({ rowId }: { rowId: string }) {
  return (
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <Button variant="ghost" size="icon" aria-label="Row actions">
          <MoreHorizontalIcon />
        </Button>
      </DropdownMenuTrigger>
      <DropdownMenuContent align="end">
        <DropdownMenuItem onSelect={() => openEditDrawer(rowId)}>
          Edit <DropdownMenuShortcut>E</DropdownMenuShortcut>
        </DropdownMenuItem>
        <DropdownMenuItem onSelect={() => duplicateRow(rowId)}>
          Duplicate <DropdownMenuShortcut>D</DropdownMenuShortcut>
        </DropdownMenuItem>
        <DropdownMenuItem onSelect={() => copyId(rowId)}>
          Copy ID
        </DropdownMenuItem>
        <DropdownMenuSeparator />
        <DropdownMenuItem variant="destructive" onSelect={() => deleteRow(rowId)}>
          Delete <DropdownMenuShortcut>Del</DropdownMenuShortcut>
        </DropdownMenuItem>
      </DropdownMenuContent>
    </DropdownMenu>
  )
}
```

Key points :

- `align="end"` keeps the menu right-aligned with the trigger (the row's right edge).
- `aria-label="Row actions"` on the icon-only `<Button>` gives screen-reader users an accessible name.
- `variant="destructive"` on the Delete item drives the red colour via `data-variant`.
- `<DropdownMenuShortcut>` is visual-only ; bind `E`, `D`, `Delete` separately with a `useHotkeys` hook or a key listener.

## 2. ContextMenu : right-click on a card

A right-click context menu attached to a content card. `asChild` is critical so the Card's existing layout / focus / styles are preserved.

```tsx
"use client"

import {
  ContextMenu, ContextMenuTrigger, ContextMenuContent, ContextMenuItem,
  ContextMenuSeparator, ContextMenuShortcut,
} from "@/components/ui/context-menu"
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card"

function PhotoCard({ photo }: { photo: Photo }) {
  return (
    <ContextMenu>
      <ContextMenuTrigger asChild>
        <Card>
          <CardHeader>
            <CardTitle>{photo.title}</CardTitle>
          </CardHeader>
          <CardContent>
            <img src={photo.url} alt={photo.title} />
          </CardContent>
        </Card>
      </ContextMenuTrigger>
      <ContextMenuContent>
        <ContextMenuItem onSelect={() => openInNewTab(photo.url)}>
          Open in new tab
        </ContextMenuItem>
        <ContextMenuItem onSelect={() => copyImageUrl(photo.url)}>
          Copy image address <ContextMenuShortcut>Cmd+C</ContextMenuShortcut>
        </ContextMenuItem>
        <ContextMenuItem onSelect={() => downloadPhoto(photo)}>
          Save image as
        </ContextMenuItem>
        <ContextMenuSeparator />
        <ContextMenuItem variant="destructive" onSelect={() => deletePhoto(photo.id)}>
          Delete
        </ContextMenuItem>
      </ContextMenuContent>
    </ContextMenu>
  )
}
```

Key points :

- `<ContextMenuTrigger asChild>` makes the `<Card>` itself the right-click target, not a wrapping `<span>`.
- ContextMenu has no explicit "Trigger button" ; users must know to right-click. Consider pairing with a tooltip or a kebab-DropdownMenu for discoverability.
- The menu opens at the cursor position, not anchored to the card edge.

## 3. Menubar : File / Edit / View with sub-menus

A classic desktop-app top bar. Each top-level entry is one `<MenubarMenu>` ; a sub-menu inside Edit shows the nested pattern.

```tsx
"use client"

import {
  Menubar, MenubarMenu, MenubarTrigger, MenubarContent,
  MenubarItem, MenubarSeparator, MenubarShortcut,
  MenubarSub, MenubarSubTrigger, MenubarSubContent,
  MenubarCheckboxItem,
} from "@/components/ui/menubar"
import { useState } from "react"

function AppMenubar() {
  const [showStatusBar, setShowStatusBar] = useState(true)

  return (
    <Menubar>
      <MenubarMenu>
        <MenubarTrigger>File</MenubarTrigger>
        <MenubarContent>
          <MenubarItem>New tab <MenubarShortcut>Cmd+T</MenubarShortcut></MenubarItem>
          <MenubarItem>New window <MenubarShortcut>Cmd+N</MenubarShortcut></MenubarItem>
          <MenubarSeparator />
          <MenubarItem>Open <MenubarShortcut>Cmd+O</MenubarShortcut></MenubarItem>
          <MenubarItem>Print <MenubarShortcut>Cmd+P</MenubarShortcut></MenubarItem>
        </MenubarContent>
      </MenubarMenu>

      <MenubarMenu>
        <MenubarTrigger>Edit</MenubarTrigger>
        <MenubarContent>
          <MenubarItem>Undo <MenubarShortcut>Cmd+Z</MenubarShortcut></MenubarItem>
          <MenubarItem>Redo <MenubarShortcut>Cmd+Shift+Z</MenubarShortcut></MenubarItem>
          <MenubarSeparator />
          <MenubarSub>
            <MenubarSubTrigger>Find</MenubarSubTrigger>
            <MenubarSubContent>
              <MenubarItem>Search the web</MenubarItem>
              <MenubarSeparator />
              <MenubarItem>Find ... <MenubarShortcut>Cmd+F</MenubarShortcut></MenubarItem>
              <MenubarItem>Find next <MenubarShortcut>Cmd+G</MenubarShortcut></MenubarItem>
            </MenubarSubContent>
          </MenubarSub>
        </MenubarContent>
      </MenubarMenu>

      <MenubarMenu>
        <MenubarTrigger>View</MenubarTrigger>
        <MenubarContent>
          <MenubarCheckboxItem
            checked={showStatusBar}
            onCheckedChange={setShowStatusBar}
          >
            Status bar
          </MenubarCheckboxItem>
          <MenubarSeparator />
          <MenubarItem>Reload <MenubarShortcut>Cmd+R</MenubarShortcut></MenubarItem>
        </MenubarContent>
      </MenubarMenu>
    </Menubar>
  )
}
```

Key points :

- Each top-level menu is wrapped in `<MenubarMenu>`. Arrow keys navigate between top-level menus once one is open.
- `<MenubarSub>` is a self-contained nested-menu Root. The auto-chevron-right comes from `MenubarSubTrigger`.
- `<MenubarCheckboxItem>` uses the controlled `checked` / `onCheckedChange` pair, identical to RadioGroup.

## 4. NavigationMenu : mega-menu site header

The complex "Products" panel pattern from shadcn's own docs : a multi-column grid of links inside a NavigationMenuContent. Uses Next.js `<Link>` via `asChild`.

```tsx
import * as React from "react"
import Link from "next/link"
import {
  NavigationMenu, NavigationMenuList, NavigationMenuItem,
  NavigationMenuTrigger, NavigationMenuContent, NavigationMenuLink,
  navigationMenuTriggerStyle,
} from "@/components/ui/navigation-menu"
import { cn } from "@/lib/utils"

function MarketingNav() {
  return (
    <NavigationMenu>
      <NavigationMenuList>
        <NavigationMenuItem>
          <NavigationMenuTrigger>Products</NavigationMenuTrigger>
          <NavigationMenuContent>
            <ul className="grid gap-3 p-4 md:w-[500px] md:grid-cols-2">
              <ListItem href="/products/cad" title="CAD">
                Drafting and modelling tools for engineers.
              </ListItem>
              <ListItem href="/products/bim" title="BIM">
                Building Information Modelling for AEC teams.
              </ListItem>
              <ListItem href="/products/api" title="API">
                Programmatic access for automation pipelines.
              </ListItem>
              <ListItem href="/products/cloud" title="Cloud">
                Hosted execution and storage.
              </ListItem>
            </ul>
          </NavigationMenuContent>
        </NavigationMenuItem>

        <NavigationMenuItem>
          <NavigationMenuLink asChild className={navigationMenuTriggerStyle()}>
            <Link href="/pricing">Pricing</Link>
          </NavigationMenuLink>
        </NavigationMenuItem>

        <NavigationMenuItem>
          <NavigationMenuLink asChild className={navigationMenuTriggerStyle()}>
            <Link href="/docs">Docs</Link>
          </NavigationMenuLink>
        </NavigationMenuItem>
      </NavigationMenuList>
    </NavigationMenu>
  )
}

function ListItem({
  href, title, children,
}: {
  href: string
  title: string
  children: React.ReactNode
}) {
  return (
    <li>
      <NavigationMenuLink asChild>
        <Link href={href}>
          <div className="text-sm font-medium leading-none">{title}</div>
          <p className="line-clamp-2 text-sm leading-snug text-muted-foreground">
            {children}
          </p>
        </Link>
      </NavigationMenuLink>
    </li>
  )
}
```

Key points :

- `<NavigationMenuLink asChild>` wraps the Next.js `<Link>`. No `legacyBehavior`, no `passHref` ; the app router does not need them.
- Top-level entries without a content panel (Pricing, Docs) use `<NavigationMenuLink>` directly inside `<NavigationMenuItem>`, styled via the exported `navigationMenuTriggerStyle()` cva helper to match the visual height of NavigationMenuTrigger.
- The mega-menu content is just custom JSX inside `<NavigationMenuContent>`. Radix portals it into the shared Viewport (the slid panel below the bar).

## 5. CheckboxItem and RadioGroup state

The two stateful-row patterns. Both use the controlled-pair contract identical to every Radix primitive.

```tsx
"use client"

import { useState } from "react"
import {
  DropdownMenu, DropdownMenuTrigger, DropdownMenuContent,
  DropdownMenuLabel, DropdownMenuSeparator,
  DropdownMenuCheckboxItem,
  DropdownMenuRadioGroup, DropdownMenuRadioItem,
} from "@/components/ui/dropdown-menu"
import { Button } from "@/components/ui/button"

function ViewMenu() {
  const [showBookmarks, setShowBookmarks] = useState(true)
  const [showFullUrls, setShowFullUrls] = useState(false)
  const [layout, setLayout] = useState("compact")

  return (
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <Button variant="outline">View</Button>
      </DropdownMenuTrigger>
      <DropdownMenuContent className="w-56">
        <DropdownMenuLabel>Appearance</DropdownMenuLabel>
        <DropdownMenuCheckboxItem
          checked={showBookmarks}
          onCheckedChange={setShowBookmarks}
        >
          Show bookmarks bar
        </DropdownMenuCheckboxItem>
        <DropdownMenuCheckboxItem
          checked={showFullUrls}
          onCheckedChange={setShowFullUrls}
        >
          Show full URLs
        </DropdownMenuCheckboxItem>
        <DropdownMenuSeparator />
        <DropdownMenuLabel>Layout</DropdownMenuLabel>
        <DropdownMenuRadioGroup value={layout} onValueChange={setLayout}>
          <DropdownMenuRadioItem value="compact">Compact</DropdownMenuRadioItem>
          <DropdownMenuRadioItem value="comfortable">Comfortable</DropdownMenuRadioItem>
          <DropdownMenuRadioItem value="spacious">Spacious</DropdownMenuRadioItem>
        </DropdownMenuRadioGroup>
      </DropdownMenuContent>
    </DropdownMenu>
  )
}
```

Key points :

- Selecting any row CLOSES the menu by default. To keep the menu open while a CheckboxItem is toggled (so the user can flip several), pass `onSelect={(e) => e.preventDefault()}` on each row.
- The same RadioGroup / CheckboxItem pattern works identically inside ContextMenu and Menubar ; only the prefix changes.

## 6. Icon + label + shortcut composition

The standard menu-item layout : leading icon, text label, trailing keyboard hint.

```tsx
import {
  DropdownMenuItem, DropdownMenuShortcut,
} from "@/components/ui/dropdown-menu"
import { SettingsIcon, UserIcon, LogOutIcon } from "lucide-react"

<DropdownMenuItem>
  <UserIcon />
  Profile
  <DropdownMenuShortcut>Shift+P</DropdownMenuShortcut>
</DropdownMenuItem>
<DropdownMenuItem>
  <SettingsIcon />
  Settings
  <DropdownMenuShortcut>Shift+S</DropdownMenuShortcut>
</DropdownMenuItem>
<DropdownMenuItem variant="destructive">
  <LogOutIcon />
  Sign out
  <DropdownMenuShortcut>Shift+Q</DropdownMenuShortcut>
</DropdownMenuItem>
```

Key points :

- shadcn's v4 source applies `[&_svg]:size-4 [&_svg]:text-muted-foreground` rules to `DropdownMenuItem` (see `apps/v4/registry/new-york-v4/ui/dropdown-menu.tsx`). The icon auto-sizes to 16px and dims to the muted colour. Override with `[&_svg]:text-foreground` if you want a brighter icon.
- `<DropdownMenuShortcut>` is rendered with `ml-auto` so it always sticks to the right edge regardless of label length.
- The same composition works inside `ContextMenuItem` and `MenubarItem`.
