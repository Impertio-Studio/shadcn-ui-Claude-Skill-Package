# Menu Primitives : Anti-patterns

Seven canonical menu failures. Each entry : the broken pattern, WHY it breaks, the FIX.

## 1. Missing `"use client"` on the menu file (hydration error)

### Broken

```tsx
// components/ui/dropdown-menu.tsx
// no "use client" directive
import * as React from "react"
import { DropdownMenu as DropdownMenuPrimitive } from "radix-ui"
// ...
```

### Why it breaks

The menu primitives (DropdownMenu, ContextMenu, Menubar) all use Radix React Context, the `radix-ui` package's client-only hooks, and a Portal that runs on `document.body`. None of that works in a React Server Component.

When imported into a Server Component without `"use client"`, Next.js logs `Error: useContext is not a function` at build time, or a runtime `Hydration failed because the initial UI does not match what was rendered on the server`.

The shadcn v4 registry source for `dropdown-menu.tsx`, `context-menu.tsx`, and `menubar.tsx` all ship with `"use client"` as the first line. Stripping it (during a manual refactor, a codemod, or a "clean up" pass) breaks the menu.

### Fix

ALWAYS keep `"use client"` as the first non-comment line of `components/ui/dropdown-menu.tsx`, `context-menu.tsx`, and `menubar.tsx`. If you ran a codemod that stripped it, restore it from the shadcn registry source.

(NavigationMenu is a special case : the v4 source does NOT have `"use client"` at the top, because the file's `cva()` factory is module-level. Next.js infers the client boundary from the `radix-ui` import. If you migrate to a framework without auto-inference, add `"use client"` defensively.)

## 2. Wrong primitive : DropdownMenu where ContextMenu fits

### Broken

```tsx
function PhotoRow({ photo }) {
  const [open, setOpen] = useState(false)

  return (
    <div
      onContextMenu={(e) => {
        e.preventDefault()
        setOpen(true)
      }}
    >
      <DropdownMenu open={open} onOpenChange={setOpen}>
        <DropdownMenuTrigger className="sr-only">
          Open menu
        </DropdownMenuTrigger>
        <DropdownMenuContent>
          <DropdownMenuItem>Rename</DropdownMenuItem>
          <DropdownMenuItem>Delete</DropdownMenuItem>
        </DropdownMenuContent>
      </DropdownMenu>
    </div>
  )
}
```

### Why it breaks

The author wanted a right-click menu on a row, so they wired `onContextMenu` to manually open a DropdownMenu. This produces three failures :

1. The menu opens at the DropdownMenuTrigger's location (off-screen because `sr-only`), not at the cursor where the user right-clicked.
2. The trigger is hidden, so keyboard users cannot tab to it and open the menu without a mouse.
3. The a11y tree shows a `button` with `aria-haspopup="menu"`, not a `contextmenu`-opened menu. Screen-reader users hear nothing on right-click.

ContextMenu exists exactly for this use case and handles cursor positioning, keyboard activation (Shift+F10), and a11y wiring automatically.

### Fix

```tsx
function PhotoRow({ photo }) {
  return (
    <ContextMenu>
      <ContextMenuTrigger asChild>
        <div>{/* row content */}</div>
      </ContextMenuTrigger>
      <ContextMenuContent>
        <ContextMenuItem>Rename</ContextMenuItem>
        <ContextMenuItem>Delete</ContextMenuItem>
      </ContextMenuContent>
    </ContextMenu>
  )
}
```

Decision rule : button-click → DropdownMenu. Right-click on an area → ContextMenu. Never simulate one with the other.

## 3. MenubarSeparator nested incorrectly

### Broken

```tsx
<Menubar>
  <MenubarMenu>
    <MenubarTrigger>File</MenubarTrigger>
    <MenubarSeparator />          {/* WRONG : separator at MenubarMenu level */}
    <MenubarContent>
      <MenubarItem>New</MenubarItem>
      <MenubarItem>
        Save
        <MenubarSeparator />        {/* WRONG : separator inside an Item */}
      </MenubarItem>
    </MenubarContent>
  </MenubarMenu>
</Menubar>
```

### Why it breaks

`MenubarSeparator` wraps `MenubarPrimitive.Separator`, which Radix expects as a *sibling* of `MenubarItem` / `MenubarLabel` / `MenubarGroup` inside `MenubarContent` (or inside `MenubarSubContent`). Putting it elsewhere causes :

- Visual breakage : the separator inherits styles that assume an item-row sibling context.
- A11y noise : Radix may emit a development warning about menu structure violations.
- Inside an Item, the separator renders as a child of a `menuitem`, which is invalid per the ARIA Menu pattern.

The same rule applies to DropdownMenuSeparator and ContextMenuSeparator : direct children of `*MenuContent` or `*MenuSubContent` only.

### Fix

```tsx
<Menubar>
  <MenubarMenu>
    <MenubarTrigger>File</MenubarTrigger>
    <MenubarContent>
      <MenubarItem>New</MenubarItem>
      <MenubarSeparator />            {/* correct : sibling of Items */}
      <MenubarItem>Save</MenubarItem>
    </MenubarContent>
  </MenubarMenu>
</Menubar>
```

## 4. NavigationMenuLink with Next.js `<Link legacyBehavior>`

### Broken

```tsx
<NavigationMenuLink asChild>
  <Link href="/pricing" legacyBehavior passHref>
    <a>Pricing</a>
  </Link>
</NavigationMenuLink>
```

### Why it breaks

Next.js 13+ app router does not need `legacyBehavior` / `passHref` ; modern `<Link>` already renders an `<a>` itself. Combining `<NavigationMenuLink asChild>` with `<Link legacyBehavior>` produces :

- Two nested `<a>` tags in the DOM : the one from `legacyBehavior`'s inner `<a>` plus the one from `NavigationMenuLink`'s default render. Invalid HTML.
- A hydration mismatch warning : the server renders one structure, the client renders another after Radix Slot's prop merge.
- React DevTools shows the `<a>` being mounted twice with conflicting click handlers ; the second handler (often Radix's `onSelect`) is dropped.

### Fix

```tsx
<NavigationMenuLink asChild>
  <Link href="/pricing">Pricing</Link>
</NavigationMenuLink>
```

The modern Next.js Link renders its own `<a>`. `<NavigationMenuLink asChild>` merges its props (including `data-active` and event handlers) onto that single `<a>` via Radix Slot. One anchor element, correct hydration.

Same rule for `<react-router-dom>`'s `<Link>` and `@tanstack/router`'s `<Link>` : pass `asChild` and let Slot do the prop merge ; do not stack legacy compat layers.

## 5. ContextMenuTrigger without `asChild` swallows right-click

### Broken

```tsx
<ContextMenu>
  <ContextMenuTrigger>
    <Card onContextMenu={(e) => e.preventDefault()}>
      {/* ... */}
    </Card>
  </ContextMenuTrigger>
  <ContextMenuContent>
    <ContextMenuItem>Delete</ContextMenuItem>
  </ContextMenuContent>
</ContextMenu>
```

### Why it breaks

Two failures stack here :

1. Without `asChild`, `<ContextMenuTrigger>` renders its own `<span>` wrapping the `<Card>`. The right-click event fires on the Card first, where `e.preventDefault()` stops it bubbling to the wrapping `<span>` that Radix actually listens on.
2. Even without the inner `preventDefault`, the wrapping `<span>` is `display: inline` by default and breaks any flex / grid sizing of the Card.

The result : right-click does nothing, or it opens the browser's native context menu instead of the custom one.

### Fix

```tsx
<ContextMenu>
  <ContextMenuTrigger asChild>
    <Card>
      {/* ... */}
    </Card>
  </ContextMenuTrigger>
  <ContextMenuContent>
    <ContextMenuItem>Delete</ContextMenuItem>
  </ContextMenuContent>
</ContextMenu>
```

With `asChild`, Radix Slot merges the `onContextMenu` listener directly onto the `<Card>`. Do not attach your own `onContextMenu` to the same element ; if you must, forward via :

```tsx
<Card
  onContextMenu={(e) => {
    e.preventDefault() // optional, Radix already does this
    // ... your logic
  }}
>
```

NEVER call `e.preventDefault()` *before* delegating to Radix unless you intend to disable the context menu entirely.

## 6. SubContent rendered without its Sub Root

### Broken

```tsx
<DropdownMenuContent>
  <DropdownMenuItem>Profile</DropdownMenuItem>

  <DropdownMenuSubTrigger>More</DropdownMenuSubTrigger>
  <DropdownMenuSubContent>          {/* WRONG : no <Sub> wrapper */}
    <DropdownMenuItem>Settings</DropdownMenuItem>
  </DropdownMenuSubContent>
</DropdownMenuContent>
```

### Why it breaks

`DropdownMenuSubTrigger` and `DropdownMenuSubContent` are the children of `DropdownMenuSub` (which wraps `DropdownMenuPrimitive.Sub`, the nested-menu Root). Without the Sub wrapper :

- `DropdownMenuSubTrigger` has no Sub context to register against ; Radix throws `useDropdownMenuContext` errors, or silently degrades to nothing happening on hover / arrow-right.
- `DropdownMenuSubContent` is not portaled into the parent menu's stacking scope ; even if it renders, it can be clipped by `overflow:hidden` ancestors.

### Fix

```tsx
<DropdownMenuContent>
  <DropdownMenuItem>Profile</DropdownMenuItem>
  <DropdownMenuSub>
    <DropdownMenuSubTrigger>More</DropdownMenuSubTrigger>
    <DropdownMenuSubContent>
      <DropdownMenuItem>Settings</DropdownMenuItem>
    </DropdownMenuSubContent>
  </DropdownMenuSub>
</DropdownMenuContent>
```

ALWAYS wrap `SubTrigger` + `SubContent` in `Sub`. The same rule applies to ContextMenuSub and MenubarSub.

If the SubContent gets clipped by an ancestor with `overflow: hidden`, the SubContent is *already* portaled by the v4 shadcn registry source. The cause is usually a forgotten `<Sub>` wrapper, not a portal issue.

## 7. Stateful CheckboxItem without `onCheckedChange`

### Broken

```tsx
const [showBookmarks] = useState(true)

<DropdownMenuCheckboxItem checked={showBookmarks}>
  Show bookmarks
</DropdownMenuCheckboxItem>
```

### Why it breaks

The author passed `checked` (controlled prop) but forgot `onCheckedChange` (the setter). Radix sees the controlled prop and treats it as authoritative ; the row is locked to its initial value and clicks do nothing.

Same failure pattern as Dialog's `open` without `onOpenChange`, Select's `value` without `onValueChange`, Tabs' `value` without `onValueChange`. The Radix controlled-state contract is uniform.

### Fix

```tsx
const [showBookmarks, setShowBookmarks] = useState(true)

<DropdownMenuCheckboxItem
  checked={showBookmarks}
  onCheckedChange={setShowBookmarks}
>
  Show bookmarks
</DropdownMenuCheckboxItem>
```

ALWAYS pair `checked` with `onCheckedChange`. NEVER pass `checked` alone (the row becomes a frozen prop).

The same applies to `MenubarCheckboxItem`, `ContextMenuCheckboxItem`, `DropdownMenuRadioGroup`'s `value` / `onValueChange`, and Menubar's `value` / `onValueChange` on the Root.

For uncontrolled CheckboxItem, omit both `checked` and `onCheckedChange` and let Radix track internal state ; read it via `defaultChecked` if you need an initial value.
