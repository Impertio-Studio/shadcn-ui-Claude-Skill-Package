---
name: shadcn-syntax-menu-primitives
description: >
  Use when adding, debugging, or refactoring any shadcn ui menu surface :
  DropdownMenu (button-triggered popup), ContextMenu (right-click on an
  area), Menubar (horizontal File / Edit / View desktop-app bar), or
  NavigationMenu (site-wide nav with viewport + mega-menu content).
  Picks the correct primitive family from the interaction model rather
  than from the visual shape, composes the shared Radix Menu surface
  (Root / Trigger / Content / Item / Group / Label / Separator /
  CheckboxItem / RadioGroup / RadioItem / Sub / SubTrigger / SubContent /
  Portal / Shortcut), and wires CheckboxItem / RadioGroup state, sub-
  menus, item shortcuts, and the special NavigationMenu surface
  (List / Item / Trigger / Content / Link / Indicator / Viewport).
  Prevents the canonical menu failures : stripping `"use client"` from
  the menu file so DropdownMenu / ContextMenu / Menubar break with a
  hydration error, reaching for DropdownMenu when the UX is a right-
  click context menu (or vice versa), nesting MenubarSeparator outside
  MenubarContent so Radix throws, composing NavigationMenuLink around
  a Next.js `<Link legacyBehavior>` so two anchors render, wrapping
  ContextMenuTrigger around a child without `asChild` so the child's
  own onContextMenu swallows the right-click, and rendering a
  SubContent without DropdownMenuPortal so it gets clipped by an
  overflow ancestor.
  Covers the four shadcn menu primitives verbatim from
  `apps/v4/registry/new-york-v4/ui/{dropdown-menu,context-menu,menubar,
  navigation-menu}.tsx`, the shared Radix Menu API (CheckboxItem +
  RadioGroup + RadioItem semantics, Sub / SubTrigger / SubContent
  composition, asChild on Trigger, controlled `open` / `onOpenChange`
  on Root and Sub roots, `data-state="open|closed"` styling hook),
  the per-primitive differences (ContextMenuTrigger as area wrapper
  with no explicit button, MenubarMenu wrapping each top-level menu,
  NavigationMenu's separate List / Viewport / Indicator surface and
  the `viewport={false}` opt-out), the v4 DropdownMenuContent auto-
  Portal, the Menubar default `align="start"` / `alignOffset=-4` /
  `sideOffset=8`, the NavigationMenuLink asChild + Next.js Link
  pattern (no `legacyBehavior` in app router), and the "icon + label +
  shortcut" composition convention for menu items.
  Keywords: shadcn dropdown menu, dropdown menu, DropdownMenuItem,
  DropdownMenuTrigger, DropdownMenuContent, DropdownMenuCheckboxItem,
  DropdownMenuRadioGroup, DropdownMenuRadioItem, DropdownMenuShortcut,
  DropdownMenuSeparator, DropdownMenuSub, DropdownMenuSubTrigger,
  DropdownMenuSubContent, DropdownMenuPortal, context menu, right
  click menu, ContextMenuTrigger, ContextMenuContent, ContextMenuItem,
  right click on row, right click on card, menubar, app menu, File
  Edit View, MenubarMenu, MenubarTrigger, MenubarContent, MenubarSub,
  MenubarShortcut, navigation menu, mega menu, site nav, header nav,
  NavigationMenuList, NavigationMenuItem, NavigationMenuTrigger,
  NavigationMenuContent, NavigationMenuLink, NavigationMenuIndicator,
  NavigationMenuViewport, when which menu, dropdown vs context menu,
  menubar vs navigation menu, which menu primitive, three dots menu,
  kebab menu, overflow menu, action menu, options menu, submenu,
  nested menu, checkbox in menu, radio in menu, keyboard shortcut in
  menu, menu hydration error, use client menu, right click swallowed,
  menu clipped by overflow.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui : Menu Primitives

Four shadcn primitives share the Radix Menu surface : DropdownMenu, ContextMenu, Menubar, NavigationMenu. They look similar (a list of items in a popover-ish surface), but each maps to a different *interaction model*. Picking the right one is a one-question decision : how does the user open the menu? Composition of the body is then nearly identical (Item / CheckboxItem / RadioGroup / Sub / Separator / Shortcut), with NavigationMenu being the one exception that swaps the body for a list-based site-nav surface with a shared Viewport.

## Quick Reference

### Decision Tree : Which menu primitive?

```
Q1. How does the user OPEN the menu?
    click a button (icon, "...", "Options", avatar)   -> DropdownMenu
    right-click (or long-press) on an area / row / canvas -> ContextMenu
    click a label in a desktop-app top bar (File, Edit, View, Help, with
    arrow-key navigation between top-level labels)    -> Menubar
    hover or click an item in a site / page header
    nav, often opening a mega-menu of links           -> NavigationMenu
```

ALWAYS pick from the interaction model first. NEVER pick by visual shape ; DropdownMenu and ContextMenu render almost identically but expose different a11y semantics (`role="menu"` opened from a button vs from `contextmenu` event).

### Shared body surface (Dropdown / Context / Menubar)

| Slot | Purpose |
|------|---------|
| `*Menu` (Root) | State container ; `open` / `onOpenChange` / `defaultOpen` / `modal` |
| `*MenuTrigger` | Element that opens the menu ; `asChild`-aware |
| `*MenuPortal` | Renders Content inside `document.body` (or `container`) |
| `*MenuContent` | The popover surface ; `align` / `side` / `sideOffset` / `alignOffset` |
| `*MenuGroup` | Logical group ; pair with `*MenuLabel` for a header |
| `*MenuLabel` | Non-interactive heading inside a group |
| `*MenuItem` | Clickable row ; `onSelect` ; `disabled` ; `data-inset` (Dropdown / Context) |
| `*MenuCheckboxItem` | Toggleable row ; `checked` / `onCheckedChange` |
| `*MenuRadioGroup` | Wraps RadioItems ; `value` / `onValueChange` |
| `*MenuRadioItem` | Mutually-exclusive choice inside a RadioGroup ; `value` |
| `*MenuSeparator` | Horizontal divider (a `<hr>`-shaped div) |
| `*MenuShortcut` | Right-aligned `kbd`-styled text (Cmd+K, etc.) |
| `*MenuSub` | Nested-submenu state container |
| `*MenuSubTrigger` | Opens the submenu ; auto-prepends a chevron-right |
| `*MenuSubContent` | The nested popover surface |

The `*` prefix is one of `DropdownMenu` / `ContextMenu` / `Menubar`. The slot names, props, and composition rules are identical across all three.

### NavigationMenu surface (different)

| Slot | Purpose |
|------|---------|
| `NavigationMenu` | Root ; `viewport?: boolean` (default `true`) toggles the shared Viewport |
| `NavigationMenuList` | The `<ul>` of top-level items |
| `NavigationMenuItem` | One `<li>` ; container for Trigger + Content (or a bare Link) |
| `NavigationMenuTrigger` | The hover/click target ; auto-appends a chevron-down |
| `NavigationMenuContent` | The mega-menu surface (rendered into Viewport) |
| `NavigationMenuLink` | A styled link ; `asChild` to wrap a Next.js `<Link>` |
| `NavigationMenuIndicator` | The optional pointing-triangle that tracks the active Trigger |
| `NavigationMenuViewport` | The shared sliding container (auto-rendered when `viewport=true`) |
| `navigationMenuTriggerStyle` | Exported `cva` helper for styling non-Trigger links to match |

NavigationMenu does NOT use `Portal` / `Group` / `Item` / `CheckboxItem` / `RadioGroup` / `Sub`. It is a navigation primitive, not a command-menu primitive.

### Five invariants

1. ALWAYS keep `"use client"` at the top of `components/ui/{dropdown-menu,context-menu,menubar}.tsx`. NEVER strip it ; the menu primitives use Radix client-only context. (NavigationMenu currently ships WITHOUT `"use client"` in the v4 registry source because the `cva` factory is module-level, but the file becomes a client boundary the moment it is imported into a Server Component. Let Next.js infer.)
2. ALWAYS pick the primitive from the interaction model (button-click / right-click / app-bar / site-nav). NEVER use DropdownMenu where the UX is right-click ; screen-reader users hear "menu" with no context of what was right-clicked.
3. ALWAYS render exactly ONE `*MenuContent` per `*Menu` Root. ALWAYS render exactly ONE `MenubarContent` per `MenubarMenu`. ALWAYS render exactly ONE `NavigationMenuContent` per `NavigationMenuItem`. NEVER conditionally render two and toggle which is shown.
4. ALWAYS place `*MenuSeparator` as a *direct* child of `*MenuContent` (or `*MenuSubContent`). NEVER nest it inside a `*MenuItem` or `MenubarMenu` ; Radix expects it at the same level as items.
5. ALWAYS wrap a non-default trigger via `<*MenuTrigger asChild>`. ALWAYS wrap a non-default link via `<NavigationMenuLink asChild>`. NEVER place two children inside `asChild` ; Radix Slot rejects multi-child.

## DropdownMenu : button-triggered popup

The default action-menu primitive. Open via click on a button (icon-only `<Button variant="ghost" size="icon">`, a three-dot kebab, an avatar, a "Filter" pill). Used for table row actions, app header user menu, settings toggles, anywhere a button reveals a list.

### Subcomponents

`DropdownMenu`, `DropdownMenuTrigger`, `DropdownMenuPortal`, `DropdownMenuContent`, `DropdownMenuGroup`, `DropdownMenuLabel`, `DropdownMenuItem`, `DropdownMenuCheckboxItem`, `DropdownMenuRadioGroup`, `DropdownMenuRadioItem`, `DropdownMenuSeparator`, `DropdownMenuShortcut`, `DropdownMenuSub`, `DropdownMenuSubTrigger`, `DropdownMenuSubContent`.

`DropdownMenuItem` exposes a shadcn-specific `inset?: boolean` (left-pads to align with CheckboxItem / RadioItem rows) and `variant?: "default" | "destructive"` (`data-variant` drives the destructive red colouring).

### Minimal pattern

```tsx
"use client"

import {
  DropdownMenu, DropdownMenuTrigger, DropdownMenuContent,
  DropdownMenuItem, DropdownMenuSeparator, DropdownMenuShortcut,
} from "@/components/ui/dropdown-menu"
import { Button } from "@/components/ui/button"

<DropdownMenu>
  <DropdownMenuTrigger asChild>
    <Button variant="outline">Open</Button>
  </DropdownMenuTrigger>
  <DropdownMenuContent align="end">
    <DropdownMenuItem>
      Profile <DropdownMenuShortcut>Shift+P</DropdownMenuShortcut>
    </DropdownMenuItem>
    <DropdownMenuItem>Settings</DropdownMenuItem>
    <DropdownMenuSeparator />
    <DropdownMenuItem variant="destructive">Sign out</DropdownMenuItem>
  </DropdownMenuContent>
</DropdownMenu>
```

`DropdownMenuContent` in the v4 source auto-wraps itself in `DropdownMenuPortal`. Use `<DropdownMenuPortal>` explicitly only when you need to override the `container` (shadow DOM, iframe, deliberately-constrained scope).

NEVER omit `asChild` on the Trigger when wrapping a `<Button>`. Without `asChild`, the Trigger renders its own default `<button>` and the `<Button>` becomes a nested button : invalid HTML and broken focus management.

## ContextMenu : right-click area

For right-click (or long-press on touch) on a target *area*. Used on table rows, file thumbnails, canvas selections, anywhere "right-click to show options" is the UX pattern.

### Subcomponents

`ContextMenu`, `ContextMenuTrigger`, `ContextMenuPortal`, `ContextMenuContent`, `ContextMenuGroup`, `ContextMenuLabel`, `ContextMenuItem`, `ContextMenuCheckboxItem`, `ContextMenuRadioGroup`, `ContextMenuRadioItem`, `ContextMenuSeparator`, `ContextMenuShortcut`, `ContextMenuSub`, `ContextMenuSubTrigger`, `ContextMenuSubContent`.

API surface is identical to DropdownMenu *except*:

- `ContextMenuTrigger` is an *area wrapper*, not a button. It wraps the element that should respond to right-click. Pass `asChild` if the child element is already a styled component.
- There is no "click to open" affordance ; the user must know to right-click. Pair with a visible hint (a "..." icon, a tooltip) when discoverability matters.

### Minimal pattern

```tsx
"use client"

import {
  ContextMenu, ContextMenuTrigger, ContextMenuContent, ContextMenuItem,
} from "@/components/ui/context-menu"

<ContextMenu>
  <ContextMenuTrigger asChild>
    <div className="rounded border p-4">Right-click on this card</div>
  </ContextMenuTrigger>
  <ContextMenuContent>
    <ContextMenuItem>Rename</ContextMenuItem>
    <ContextMenuItem>Duplicate</ContextMenuItem>
    <ContextMenuItem variant="destructive">Delete</ContextMenuItem>
  </ContextMenuContent>
</ContextMenu>
```

ALWAYS use `asChild` when the trigger child already renders a focusable / styled element (a Card, a Button, a row `<tr>`). Without `asChild`, ContextMenuTrigger renders an extra `<span>` wrapper that can break grid layout, table semantics, or flex sizing.

NEVER attach your own `onContextMenu` to the trigger's child without forwarding the event ; the child handler can `preventDefault()` and swallow the right-click before Radix sees it.

## Menubar : desktop-app top bar

For desktop-style application menus : `File`, `Edit`, `View`, `Help`. One horizontal `Menubar` contains multiple `MenubarMenu` siblings, each owning its own Trigger + Content. Arrow keys move between top-level menus ; opening one menu and then hovering another auto-switches.

### Subcomponents

`Menubar`, `MenubarMenu`, `MenubarPortal`, `MenubarTrigger`, `MenubarContent`, `MenubarGroup`, `MenubarLabel`, `MenubarItem`, `MenubarCheckboxItem`, `MenubarRadioGroup`, `MenubarRadioItem`, `MenubarSeparator`, `MenubarShortcut`, `MenubarSub`, `MenubarSubTrigger`, `MenubarSubContent`.

`MenubarContent` defaults : `align="start"`, `alignOffset={-4}`, `sideOffset={8}` (verified from `apps/v4/registry/new-york-v4/ui/menubar.tsx`). The negative alignOffset aligns content with the Trigger's left edge.

### Minimal pattern

```tsx
"use client"

import {
  Menubar, MenubarMenu, MenubarTrigger, MenubarContent, MenubarItem,
  MenubarSeparator, MenubarShortcut,
} from "@/components/ui/menubar"

<Menubar>
  <MenubarMenu>
    <MenubarTrigger>File</MenubarTrigger>
    <MenubarContent>
      <MenubarItem>New tab <MenubarShortcut>Cmd+T</MenubarShortcut></MenubarItem>
      <MenubarItem>New window <MenubarShortcut>Cmd+N</MenubarShortcut></MenubarItem>
      <MenubarSeparator />
      <MenubarItem>Print <MenubarShortcut>Cmd+P</MenubarShortcut></MenubarItem>
    </MenubarContent>
  </MenubarMenu>
  <MenubarMenu>
    <MenubarTrigger>Edit</MenubarTrigger>
    <MenubarContent>
      <MenubarItem>Undo <MenubarShortcut>Cmd+Z</MenubarShortcut></MenubarItem>
      <MenubarItem>Redo <MenubarShortcut>Cmd+Shift+Z</MenubarShortcut></MenubarItem>
    </MenubarContent>
  </MenubarMenu>
</Menubar>
```

ALWAYS wrap each top-level menu in its own `<MenubarMenu>`. NEVER place multiple `MenubarTrigger` / `MenubarContent` pairs as direct children of `<Menubar>` ; Radix uses the `MenubarMenu` wrapper to scope state per top-level entry.

Menubar is the right call when the app feels like a desktop application (an editor, a DAW, a CAD tool, a spreadsheet). For a website's main navigation, use NavigationMenu instead.

## NavigationMenu : site-nav with viewport

For site / app header navigation : a horizontal list of top-level entries where each can open a panel of links (often a mega-menu with sections and descriptions). Different surface from the other three primitives.

### Subcomponents

`NavigationMenu`, `NavigationMenuList`, `NavigationMenuItem`, `NavigationMenuTrigger`, `NavigationMenuContent`, `NavigationMenuLink`, `NavigationMenuIndicator`, `NavigationMenuViewport`, `navigationMenuTriggerStyle` (cva helper).

`NavigationMenu` accepts a shadcn-specific `viewport?: boolean` prop (default `true`). When `true`, the Root auto-renders a `<NavigationMenuViewport />` at the bottom and every `NavigationMenuContent` slides through it. When `false`, each `NavigationMenuContent` renders inline below its own Trigger (a per-item dropdown, like DropdownMenu but link-only).

### Minimal pattern (with viewport)

```tsx
import {
  NavigationMenu, NavigationMenuList, NavigationMenuItem,
  NavigationMenuTrigger, NavigationMenuContent, NavigationMenuLink,
} from "@/components/ui/navigation-menu"
import Link from "next/link"

<NavigationMenu>
  <NavigationMenuList>
    <NavigationMenuItem>
      <NavigationMenuTrigger>Products</NavigationMenuTrigger>
      <NavigationMenuContent>
        <ul className="grid gap-3 p-4 md:w-[400px]">
          <li>
            <NavigationMenuLink asChild>
              <Link href="/products/cad">CAD</Link>
            </NavigationMenuLink>
          </li>
          <li>
            <NavigationMenuLink asChild>
              <Link href="/products/bim">BIM</Link>
            </NavigationMenuLink>
          </li>
        </ul>
      </NavigationMenuContent>
    </NavigationMenuItem>
    <NavigationMenuItem>
      <NavigationMenuLink asChild>
        <Link href="/pricing">Pricing</Link>
      </NavigationMenuLink>
    </NavigationMenuItem>
  </NavigationMenuList>
</NavigationMenu>
```

ALWAYS use `<NavigationMenuLink asChild>` when wrapping a Next.js `<Link>`. NEVER use `<Link legacyBehavior>` together with `<NavigationMenuLink asChild>` ; modern Next.js Link no longer needs `legacyBehavior` / `passHref` (Next.js 13+ app router), and combining the two produces two nested anchor tags and a hydration mismatch.

For a per-item inline dropdown instead of the shared viewport, pass `viewport={false}` on `<NavigationMenu>`. Each `<NavigationMenuContent>` then renders directly below its Trigger via `group-data-[viewport=false]` Tailwind utilities (see `apps/v4/registry/new-york-v4/ui/navigation-menu.tsx`).

## Shared pattern : sub-menus

Identical across Dropdown / Context / Menubar. NavigationMenu does NOT support sub-menus (use a nested `NavigationMenuContent` panel instead).

```tsx
<DropdownMenuSub>
  <DropdownMenuSubTrigger>More</DropdownMenuSubTrigger>
  <DropdownMenuSubContent>
    <DropdownMenuItem>Move to ...</DropdownMenuItem>
    <DropdownMenuItem>Copy to ...</DropdownMenuItem>
  </DropdownMenuSubContent>
</DropdownMenuSub>
```

`*MenuSubTrigger` auto-appends a `<ChevronRightIcon className="ml-auto" />` in the shadcn v4 source. Do not add your own chevron.

ALWAYS render `*MenuSubContent` inside a `*MenuSub` ; the SubContent is automatically portaled (verified from `apps/v4/registry/new-york-v4/ui/dropdown-menu.tsx`).

NEVER nest a `*MenuSub` inside a `NavigationMenuContent` ; the primitive families are not interchangeable.

## Shared pattern : CheckboxItem and RadioGroup

```tsx
const [showBookmarks, setShowBookmarks] = useState(true)
const [layout, setLayout] = useState("compact")

<DropdownMenuCheckboxItem
  checked={showBookmarks}
  onCheckedChange={setShowBookmarks}
>
  Bookmarks bar
</DropdownMenuCheckboxItem>

<DropdownMenuRadioGroup value={layout} onValueChange={setLayout}>
  <DropdownMenuRadioItem value="compact">Compact</DropdownMenuRadioItem>
  <DropdownMenuRadioItem value="comfortable">Comfortable</DropdownMenuRadioItem>
</DropdownMenuRadioGroup>
```

Identical names exist on ContextMenu (`ContextMenuCheckboxItem`, `ContextMenuRadioGroup`, `ContextMenuRadioItem`) and Menubar (`MenubarCheckboxItem`, `MenubarRadioGroup`, `MenubarRadioItem`).

ALWAYS pair CheckboxItem's `checked` with `onCheckedChange`, identical to the controlled-prop contract everywhere else in Radix. NEVER pass `checked` alone (the row becomes unresponsive).

## Shared pattern : MenuShortcut

The shortcut hint at the right edge of a menu item. Pure layout, not interactive ; the actual keyboard shortcut must be implemented separately (a global key listener, a `useHotkeys` hook, a CommandDialog).

```tsx
<DropdownMenuItem>
  Save <DropdownMenuShortcut>Cmd+S</DropdownMenuShortcut>
</DropdownMenuItem>
```

NEVER expect `<DropdownMenuShortcut>` to *bind* the keystroke. It only renders the styled hint. See `shadcn-syntax-command` for the CommandDialog pattern that pairs visual shortcuts with real key handling.

## a11y : roles per primitive

Each primitive maps to a different WAI-ARIA pattern. Radix wires the roles ; do not override.

| Primitive | Trigger role | Content role | Item role |
|-----------|--------------|--------------|-----------|
| DropdownMenu | `button` with `aria-haspopup="menu"` | `menu` | `menuitem` / `menuitemcheckbox` / `menuitemradio` |
| ContextMenu | (none ; opens on `contextmenu` event on the trigger area) | `menu` | identical |
| Menubar | `menuitem` (inside the bar) | `menu` | identical |
| NavigationMenu | `button` with `aria-haspopup="menu"` (Trigger) | `menu` | `link` for NavigationMenuLink |

Keyboard navigation is handled by Radix : arrow keys cycle items, Enter / Space selects, Esc closes, typing characters does type-ahead search. Do not re-implement these.

ALWAYS provide visible text content inside each `*MenuItem`. If the item is icon-only, wrap the icon's visual meaning in an `<span className="sr-only">` so the role + name pair is valid. NEVER ship a menu item with no accessible name.

## RSC and the `"use client"` directive

`dropdown-menu.tsx`, `context-menu.tsx`, and `menubar.tsx` all ship with `"use client"` at the top of the shadcn v4 registry source. This is non-negotiable :

- Each uses Radix context internally.
- State (`open`, `data-state`), focus trap, and Portal mounting all run client-side.

`navigation-menu.tsx` in the v4 registry source currently does NOT have `"use client"` as the first line, because the file's top-level `cva()` factory is module-level. Next.js treats the file as a client component when it is imported into a Server Component, via the radix-ui import boundary. If you migrate to a different React framework (Remix, vanilla Vite), add `"use client"` defensively.

ALWAYS keep `"use client"` in the Dropdown / Context / Menubar files. NEVER strip it ; the file imports `radix-ui` which requires a client boundary.

See `shadcn-impl-rsc-vs-client-boundaries` for the broader RSC contract.

## Companion Skills

- [shadcn-syntax-button](../shadcn-syntax-button/SKILL.md) : the `asChild` pattern used by every `*MenuTrigger` and by `NavigationMenuLink`. Read first if `asChild` is unfamiliar.
- [shadcn-syntax-popover-tooltip-hovercard](../shadcn-syntax-popover-tooltip-hovercard/SKILL.md) : the other Radix `*Content` family ; same `align` / `side` / `sideOffset` / `Portal` contract but no menu-item composition.
- [shadcn-syntax-command](../shadcn-syntax-command/SKILL.md) : the `cmdk` command palette ; pairs MenuShortcut hints with actual key-binding logic via CommandDialog.
- [shadcn-errors-radix-controlled](../../shadcn-errors/shadcn-errors-radix-controlled/SKILL.md) : the controlled-state failure catalogue across Radix primitives, including the Slot multi-child crash that bites `*MenuTrigger asChild` users.
- [shadcn-agents-component-selector](../../shadcn-agents/shadcn-agents-component-selector/SKILL.md) : the validator that catches "DropdownMenu where ContextMenu fits" and other wrong-primitive choices.

## Reference Links

- [references/methods.md](references/methods.md) : per-primitive primitive list (table form), prop signatures, defaults from the v4 registry source.
- [references/examples.md](references/examples.md) : six canonical recipes (DropdownMenu table-row actions, ContextMenu on a card, Menubar File/Edit/View, NavigationMenu mega-menu, CheckboxItem + RadioGroup, MenuShortcut composition).
- [references/anti-patterns.md](references/anti-patterns.md) : six anti-patterns with WHY and FIX.

## Sources

All claims in this skill trace to URLs in `SOURCES.md` :

- https://ui.shadcn.com/docs/components/radix/dropdown-menu (canonical DropdownMenu docs, composition tree)
- https://ui.shadcn.com/docs/components/radix/context-menu (ContextMenu docs, ContextMenuTrigger area-wrapper pattern)
- https://ui.shadcn.com/docs/components/radix/menubar (Menubar docs, MenubarMenu wrapper pattern)
- https://ui.shadcn.com/docs/components/radix/navigation-menu (NavigationMenu docs, viewport prop, asChild + Next.js Link)
- https://github.com/shadcn-ui/ui (apps/v4/registry/new-york-v4/ui/{dropdown-menu,context-menu,menubar,navigation-menu}.tsx, verbatim v4 source ; auto-Portal wrapping ; MenubarContent defaults ; `data-variant="destructive"` ; `navigationMenuTriggerStyle` cva)
- https://www.radix-ui.com/primitives/docs/components/dropdown-menu (Radix DropdownMenu primitive API)
- https://www.radix-ui.com/primitives/docs/components/context-menu (Radix ContextMenu primitive API)
- https://www.radix-ui.com/primitives/docs/components/menubar (Radix Menubar primitive API)
- https://www.radix-ui.com/primitives/docs/components/navigation-menu (Radix NavigationMenu primitive API)
- https://www.w3.org/WAI/ARIA/apg/patterns/menubar/ (WAI-ARIA Menubar pattern, keyboard navigation contract)

Verified 2026-05-19.
