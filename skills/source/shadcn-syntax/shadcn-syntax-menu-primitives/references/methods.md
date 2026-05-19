# Menu Primitives : Methods & Prop Signatures

All four shadcn menu primitives wrap Radix Menu (or NavigationMenu) and surface a near-identical API surface for Dropdown / Context / Menubar, with NavigationMenu using a different surface for site-nav.

Source : `apps/v4/registry/new-york-v4/ui/{dropdown-menu,context-menu,menubar,navigation-menu}.tsx`. Last verified 2026-05-19.

## DropdownMenu : full primitive list

| Primitive | Underlying Radix part | shadcn additions |
|-----------|----------------------|-------------------|
| `DropdownMenu` | `DropdownMenuPrimitive.Root` | `data-slot="dropdown-menu"` |
| `DropdownMenuPortal` | `DropdownMenuPrimitive.Portal` | `data-slot="dropdown-menu-portal"` |
| `DropdownMenuTrigger` | `DropdownMenuPrimitive.Trigger` | `data-slot="dropdown-menu-trigger"` ; `asChild`-aware |
| `DropdownMenuContent` | `DropdownMenuPrimitive.Content` (auto-wrapped in `Portal`) | default `sideOffset={4}` ; popover-themed Tailwind |
| `DropdownMenuGroup` | `DropdownMenuPrimitive.Group` | `data-slot="dropdown-menu-group"` |
| `DropdownMenuLabel` | `DropdownMenuPrimitive.Label` | `inset?: boolean` |
| `DropdownMenuItem` | `DropdownMenuPrimitive.Item` | `inset?: boolean` ; `variant?: "default" \| "destructive"` (sets `data-variant`) |
| `DropdownMenuCheckboxItem` | `DropdownMenuPrimitive.CheckboxItem` | auto-renders `CheckIcon` inside `ItemIndicator` |
| `DropdownMenuRadioGroup` | `DropdownMenuPrimitive.RadioGroup` | `data-slot="dropdown-menu-radio-group"` |
| `DropdownMenuRadioItem` | `DropdownMenuPrimitive.RadioItem` | auto-renders `CircleIcon` inside `ItemIndicator` |
| `DropdownMenuSeparator` | `DropdownMenuPrimitive.Separator` | `bg-border` divider |
| `DropdownMenuShortcut` | plain `<span>` | right-aligned `text-xs tracking-widest text-muted-foreground` |
| `DropdownMenuSub` | `DropdownMenuPrimitive.Sub` | nested-submenu Root |
| `DropdownMenuSubTrigger` | `DropdownMenuPrimitive.SubTrigger` | `inset?: boolean` ; auto-prepends `ChevronRightIcon` |
| `DropdownMenuSubContent` | `DropdownMenuPrimitive.SubContent` | popover-themed Tailwind |

## DropdownMenu : Root prop signature

```ts
type DropdownMenuRootProps = {
  defaultOpen?: boolean
  open?: boolean
  onOpenChange?: (open: boolean) => void
  modal?: boolean      // default true ; when false, outside interaction does not close
  dir?: "ltr" | "rtl"
  children: React.ReactNode
}
```

ALWAYS pass `open` + `onOpenChange` together when controlling. ALWAYS use `defaultOpen` alone when initially opening uncontrolled.

## DropdownMenu : Content prop signature

```ts
type DropdownMenuContentProps = {
  loop?: boolean              // arrow-key wrap-around
  onCloseAutoFocus?: (event: Event) => void
  onEscapeKeyDown?: (event: KeyboardEvent) => void
  onPointerDownOutside?: (event: PointerDownOutsideEvent) => void
  onFocusOutside?: (event: FocusOutsideEvent) => void
  onInteractOutside?: (event: PointerDownOutsideEvent | FocusOutsideEvent) => void
  forceMount?: boolean
  side?: "top" | "right" | "bottom" | "left"   // default "bottom"
  sideOffset?: number         // shadcn default 4
  align?: "start" | "center" | "end"           // default "center"
  alignOffset?: number        // default 0
  avoidCollisions?: boolean   // default true
  collisionBoundary?: Element | null | Array<Element | null>
  collisionPadding?: number | Partial<Record<Side, number>>
  arrowPadding?: number
  sticky?: "partial" | "always"
  hideWhenDetached?: boolean
  className?: string
  children: React.ReactNode
}
```

## DropdownMenu : Item prop signature

```ts
type DropdownMenuItemProps = {
  disabled?: boolean
  onSelect?: (event: Event) => void   // call event.preventDefault() to keep menu open
  textValue?: string                  // override for type-ahead search
  inset?: boolean                     // shadcn-only ; left-pads with pl-8
  variant?: "default" | "destructive" // shadcn-only ; drives data-variant red theme
  className?: string
  children: React.ReactNode
}
```

ALWAYS call `event.preventDefault()` inside `onSelect` when the action does NOT navigate away ; otherwise the menu closes immediately and the user loses context.

## DropdownMenu : CheckboxItem prop signature

```ts
type DropdownMenuCheckboxItemProps = {
  checked?: boolean | "indeterminate"
  onCheckedChange?: (checked: boolean) => void
  disabled?: boolean
  onSelect?: (event: Event) => void
  textValue?: string
  className?: string
  children: React.ReactNode
}
```

## DropdownMenu : RadioGroup + RadioItem

```ts
type DropdownMenuRadioGroupProps = {
  value?: string
  onValueChange?: (value: string) => void
  children: React.ReactNode
}

type DropdownMenuRadioItemProps = {
  value: string                       // REQUIRED ; the value committed to the group
  disabled?: boolean
  onSelect?: (event: Event) => void
  textValue?: string
  className?: string
  children: React.ReactNode
}
```

## ContextMenu : full primitive list

| Primitive | Underlying Radix part | Notes |
|-----------|----------------------|-------|
| `ContextMenu` | `ContextMenuPrimitive.Root` | identical Root |
| `ContextMenuTrigger` | `ContextMenuPrimitive.Trigger` | AREA wrapper, not a button |
| `ContextMenuPortal` | `ContextMenuPrimitive.Portal` | identical |
| `ContextMenuContent` | `ContextMenuPrimitive.Content` (wrapped in `Portal`) | renders at cursor on `contextmenu` |
| `ContextMenuGroup` | `ContextMenuPrimitive.Group` | |
| `ContextMenuLabel` | `ContextMenuPrimitive.Label` | `inset?: boolean` |
| `ContextMenuItem` | `ContextMenuPrimitive.Item` | `inset?: boolean` ; `variant?: "default" \| "destructive"` |
| `ContextMenuCheckboxItem` | `ContextMenuPrimitive.CheckboxItem` | |
| `ContextMenuRadioGroup` | `ContextMenuPrimitive.RadioGroup` | |
| `ContextMenuRadioItem` | `ContextMenuPrimitive.RadioItem` | |
| `ContextMenuSeparator` | `ContextMenuPrimitive.Separator` | |
| `ContextMenuShortcut` | plain `<span>` | identical styling to DropdownMenuShortcut |
| `ContextMenuSub` | `ContextMenuPrimitive.Sub` | |
| `ContextMenuSubTrigger` | `ContextMenuPrimitive.SubTrigger` | `inset?: boolean` ; auto-prepends `ChevronRightIcon` |
| `ContextMenuSubContent` | `ContextMenuPrimitive.SubContent` | |

## ContextMenu : Root prop signature

```ts
type ContextMenuRootProps = {
  onOpenChange?: (open: boolean) => void   // no `open` / `defaultOpen` ; position is cursor-driven
  modal?: boolean                          // default true
  dir?: "ltr" | "rtl"
  children: React.ReactNode
}
```

ContextMenu has NO `open` or `defaultOpen`. Opening is triggered exclusively by the `contextmenu` event on its Trigger area. To force-close, call `onOpenChange(false)` via the callback if you stored it.

## ContextMenu : Trigger prop signature

```ts
type ContextMenuTriggerProps = {
  disabled?: boolean   // disables right-click handling
  asChild?: boolean
  className?: string
  children: React.ReactNode
}
```

ALWAYS use `asChild` if the trigger child is already styled (Card, table row, image). Without `asChild`, ContextMenuTrigger renders a `<span>` wrapper.

## ContextMenu : Content prop signature

Same as DropdownMenuContent (`loop` / `onEscapeKeyDown` / `onPointerDownOutside` / `forceMount` / `side` / `sideOffset` / `align` / `alignOffset` / `avoidCollisions` / `collisionBoundary` / `collisionPadding` / `sticky` / `hideWhenDetached`).

## Menubar : full primitive list

| Primitive | Underlying Radix part | Notes |
|-----------|----------------------|-------|
| `Menubar` | `MenubarPrimitive.Root` | horizontal `flex h-9 items-center gap-1 rounded-md border` |
| `MenubarMenu` | `MenubarPrimitive.Menu` | wraps EACH top-level menu (one MenubarMenu per File / Edit / View) |
| `MenubarTrigger` | `MenubarPrimitive.Trigger` | the label visible in the bar |
| `MenubarPortal` | `MenubarPrimitive.Portal` | |
| `MenubarContent` | `MenubarPrimitive.Content` (wrapped in `MenubarPortal`) | defaults `align="start"` ; `alignOffset={-4}` ; `sideOffset={8}` |
| `MenubarGroup` | `MenubarPrimitive.Group` | |
| `MenubarLabel` | `MenubarPrimitive.Label` | `inset?: boolean` |
| `MenubarItem` | `MenubarPrimitive.Item` | `inset?: boolean` ; `variant?: "default" \| "destructive"` |
| `MenubarCheckboxItem` | `MenubarPrimitive.CheckboxItem` | |
| `MenubarRadioGroup` | `MenubarPrimitive.RadioGroup` | |
| `MenubarRadioItem` | `MenubarPrimitive.RadioItem` | |
| `MenubarSeparator` | `MenubarPrimitive.Separator` | |
| `MenubarShortcut` | plain `<span>` | identical styling |
| `MenubarSub` | `MenubarPrimitive.Sub` | |
| `MenubarSubTrigger` | `MenubarPrimitive.SubTrigger` | `inset?: boolean` ; auto-prepends `ChevronRightIcon` |
| `MenubarSubContent` | `MenubarPrimitive.SubContent` | |

## Menubar : Root prop signature

```ts
type MenubarRootProps = {
  value?: string                       // controlled : which MenubarMenu (by `value`) is open
  defaultValue?: string                // uncontrolled
  onValueChange?: (value: string) => void
  dir?: "ltr" | "rtl"
  loop?: boolean                        // arrow-key wrap between top-level menus
  className?: string
  children: React.ReactNode
}
```

Each `<MenubarMenu>` can take an optional `value` prop to identify itself for controlled `Menubar.value`. If you control which top-level menu is open, ALWAYS pair `value` + `onValueChange` ; never one alone.

## NavigationMenu : full primitive list

| Primitive | Underlying Radix part | Notes |
|-----------|----------------------|-------|
| `NavigationMenu` | `NavigationMenuPrimitive.Root` | shadcn extra : `viewport?: boolean` (default `true`) ; auto-renders `<NavigationMenuViewport />` |
| `NavigationMenuList` | `NavigationMenuPrimitive.List` | the `<ul>` ; flex with `gap-1` |
| `NavigationMenuItem` | `NavigationMenuPrimitive.Item` | one `<li>` ; relative positioned |
| `NavigationMenuTrigger` | `NavigationMenuPrimitive.Trigger` | auto-appends `ChevronDownIcon` ; uses `navigationMenuTriggerStyle()` |
| `NavigationMenuContent` | `NavigationMenuPrimitive.Content` | renders into shared Viewport (or inline when `viewport={false}`) |
| `NavigationMenuLink` | `NavigationMenuPrimitive.Link` | `asChild`-aware ; `data-active` style hook |
| `NavigationMenuIndicator` | `NavigationMenuPrimitive.Indicator` | optional pointing-triangle that tracks the active Trigger |
| `NavigationMenuViewport` | `NavigationMenuPrimitive.Viewport` | shared sliding container ; auto-rendered by Root when `viewport=true` |
| `navigationMenuTriggerStyle` | `cva` factory | exported helper to style non-Trigger links to match Trigger appearance |

## NavigationMenu : Root prop signature

```ts
type NavigationMenuRootProps = {
  value?: string                       // controlled : which NavigationMenuItem (by `value`) is open
  defaultValue?: string                // uncontrolled
  onValueChange?: (value: string) => void
  delayDuration?: number               // ms before hover opens content (default 200)
  skipDelayDuration?: number           // ms grace between two hover targets (default 300)
  dir?: "ltr" | "rtl"
  orientation?: "horizontal" | "vertical"   // default "horizontal"
  viewport?: boolean                    // shadcn extra ; default true
  className?: string
  children: React.ReactNode
}
```

## NavigationMenu : Link prop signature

```ts
type NavigationMenuLinkProps = {
  asChild?: boolean
  active?: boolean                      // sets data-active=true for highlight style
  onSelect?: (event: Event) => void
  className?: string
  children: React.ReactNode
}
```

ALWAYS use `<NavigationMenuLink asChild>` when wrapping `next/link`, `react-router-dom`'s `<Link>`, or `@tanstack/router`'s `<Link>`. NEVER pass `legacyBehavior` / `passHref` on the Next.js Link inside `asChild` ; the app router does not need them and combining causes a double anchor.

## Controlled-state contract (all four)

Identical to every other Radix primitive : pair the controlling prop with its setter callback.

| Primitive | Controlled prop | Setter callback | Default-only prop |
|-----------|-----------------|------------------|-------------------|
| DropdownMenu | `open?: boolean` | `onOpenChange?: (open: boolean) => void` | `defaultOpen?: boolean` |
| ContextMenu | (no `open` ; cursor-driven) | `onOpenChange?: (open: boolean) => void` | (n/a) |
| Menubar | `value?: string` | `onValueChange?: (value: string) => void` | `defaultValue?: string` |
| NavigationMenu | `value?: string` | `onValueChange?: (value: string) => void` | `defaultValue?: string` |

ALWAYS pair controlled prop + setter. NEVER pass one alone.

## Data attributes used for styling

| Attribute | Where | Values |
|-----------|-------|--------|
| `data-state` | every Trigger / Content / Item-with-state | `"open" \| "closed"` ; `"checked" \| "unchecked" \| "indeterminate"` (CheckboxItem) |
| `data-disabled` | Item, CheckboxItem, RadioItem, SubTrigger | present when disabled |
| `data-highlighted` | Item, CheckboxItem, RadioItem | present when keyboard / pointer highlight |
| `data-inset` | DropdownMenuItem / ContextMenuItem / MenubarItem (when `inset` prop true) | applies `pl-8` |
| `data-variant` | Same Items | `"default" \| "destructive"` |
| `data-active` | NavigationMenuLink | present when `active={true}` |
| `data-motion` | NavigationMenuContent | `"from-start" \| "from-end" \| "to-start" \| "to-end"` for slide direction |
| `data-viewport` | NavigationMenu Root | `"true" \| "false"` ; drives the `group-data-[viewport=false]` Tailwind variants |

Use these for state-aware Tailwind variants : `data-[state=open]:bg-accent`, `data-[disabled]:opacity-50`, `data-[variant=destructive]:text-destructive`, `group-data-[viewport=false]/navigation-menu:top-full`.
