# Sidebar Methods Reference

Full TypeScript signatures, sourced from
`apps/v4/registry/new-york-v4/ui/sidebar.tsx` (verified 2026-05-19).

## Constants (internal, exposed via CSS vars on the Provider wrapper)

```ts
const SIDEBAR_COOKIE_NAME = "sidebar_state"
const SIDEBAR_COOKIE_MAX_AGE = 60 * 60 * 24 * 7   // 7 days
const SIDEBAR_WIDTH = "16rem"                      // desktop expanded
const SIDEBAR_WIDTH_MOBILE = "18rem"               // mobile Sheet
const SIDEBAR_WIDTH_ICON = "3rem"                  // desktop collapsed (icon variant)
const SIDEBAR_KEYBOARD_SHORTCUT = "b"              // Cmd/Ctrl + B
```

## Context shape

```ts
type SidebarContextProps = {
  state: "expanded" | "collapsed"
  open: boolean
  setOpen: (open: boolean) => void
  openMobile: boolean
  setOpenMobile: (open: boolean) => void
  isMobile: boolean
  toggleSidebar: () => void
}
```

## useSidebar hook

```ts
function useSidebar(): SidebarContextProps
```

Throws `Error("useSidebar must be used within a SidebarProvider.")` when
invoked outside a `<SidebarProvider>` subtree.

## SidebarProvider

```ts
function SidebarProvider(
  props: React.ComponentProps<"div"> & {
    defaultOpen?: boolean             // default: true
    open?: boolean                    // controlled
    onOpenChange?: (open: boolean) => void
  }
): JSX.Element
```

Side effects:

- Writes `document.cookie = "sidebar_state=<value>; path=/; max-age=604800"` on
  every desktop open change.
- Registers a `window` `keydown` listener for `(metaKey || ctrlKey) + "b"`
  that calls `toggleSidebar()`.
- Mounts a `TooltipProvider` with `delayDuration={0}` around its children, so
  `SidebarMenuButton tooltip` renders without delay when collapsed.

## Sidebar

```ts
function Sidebar(
  props: React.ComponentProps<"div"> & {
    side?: "left" | "right"                             // default: "left"
    variant?: "sidebar" | "floating" | "inset"          // default: "sidebar"
    collapsible?: "offcanvas" | "icon" | "none"         // default: "offcanvas"
  }
): JSX.Element
```

Render branches:

1. `collapsible === "none"` -> static `<div>` of width `--sidebar-width`.
2. `isMobile === true` -> `<Sheet open={openMobile} onOpenChange={setOpenMobile}>`
   containing an `sr-only` `<SheetTitle>Sidebar</SheetTitle>` and
   `<SheetDescription>Displays the mobile sidebar.</SheetDescription>` for a11y.
3. Otherwise -> desktop `<div>` with `data-state`, `data-collapsible`,
   `data-variant`, `data-side` attributes for CSS targeting.

## SidebarTrigger

```ts
function SidebarTrigger(
  props: React.ComponentProps<typeof Button>
): JSX.Element
```

Renders `<Button variant="ghost" size="icon">` containing a `PanelLeftIcon`
and an `sr-only` "Toggle Sidebar" label. Calls user `onClick` first, then
`toggleSidebar()`.

## SidebarRail

```ts
function SidebarRail(
  props: React.ComponentProps<"button">
): JSX.Element
```

Click target: thin vertical strip on the sidebar's edge that calls
`toggleSidebar()`. Has `tabIndex={-1}`, `aria-label="Toggle Sidebar"`, and
`title="Toggle Sidebar"`. Hidden on the mobile path.

## SidebarInset

```ts
function SidebarInset(
  props: React.ComponentProps<"main">
): JSX.Element
```

Renders a `<main>` with `data-slot="sidebar-inset"`. When the sibling
`Sidebar` has `variant="inset"`, the inset becomes a rounded card offset
2rem from the sidebar.

## SidebarInput

```ts
function SidebarInput(
  props: React.ComponentProps<typeof Input>
): JSX.Element
```

## SidebarHeader / SidebarFooter / SidebarContent / SidebarGroup / SidebarGroupContent

```ts
function SidebarHeader(props: React.ComponentProps<"div">): JSX.Element
function SidebarFooter(props: React.ComponentProps<"div">): JSX.Element
function SidebarContent(props: React.ComponentProps<"div">): JSX.Element
function SidebarGroup(props: React.ComponentProps<"div">): JSX.Element
function SidebarGroupContent(props: React.ComponentProps<"div">): JSX.Element
```

## SidebarSeparator

```ts
function SidebarSeparator(
  props: React.ComponentProps<typeof Separator>
): JSX.Element
```

## SidebarGroupLabel

```ts
function SidebarGroupLabel(
  props: React.ComponentProps<"div"> & { asChild?: boolean }
): JSX.Element
```

Hidden via `group-data-[collapsible=icon]:opacity-0` when the sidebar is in
icon-collapsed state.

## SidebarGroupAction

```ts
function SidebarGroupAction(
  props: React.ComponentProps<"button"> & { asChild?: boolean }
): JSX.Element
```

Top-right action button on a group. Hidden when collapsible mode is icon.

## SidebarMenu / SidebarMenuItem

```ts
function SidebarMenu(props: React.ComponentProps<"ul">): JSX.Element
function SidebarMenuItem(props: React.ComponentProps<"li">): JSX.Element
```

## sidebarMenuButtonVariants (cva)

```ts
const sidebarMenuButtonVariants = cva(
  /* base */,
  {
    variants: {
      variant: {
        default: "hover:bg-sidebar-accent hover:text-sidebar-accent-foreground",
        outline: "bg-background shadow-[0_0_0_1px_var(--sidebar-border)] ...",
      },
      size: {
        default: "h-8 text-sm",
        sm:      "h-7 text-xs",
        lg:      "h-12 text-sm group-data-[collapsible=icon]:p-0!",
      },
    },
    defaultVariants: { variant: "default", size: "default" },
  }
)
```

## SidebarMenuButton

```ts
function SidebarMenuButton(
  props: React.ComponentProps<"button"> & {
    asChild?: boolean
    isActive?: boolean
    tooltip?: string | React.ComponentProps<typeof TooltipContent>
  } & VariantProps<typeof sidebarMenuButtonVariants>
): JSX.Element
```

Behavior:

- If `tooltip` is provided, the button is wrapped in a `<Tooltip>` whose
  `<TooltipContent>` has `hidden={state !== "collapsed" || isMobile}` so the
  tooltip only shows when the desktop sidebar is collapsed in icon mode.
- If `tooltip` is a string, it becomes `<TooltipContent>{tooltip}</TooltipContent>`.
- If `tooltip` is an object, it is spread into `<TooltipContent {...tooltip} />`.
- Sets `data-active={isActive}` and `data-size={size}` for CSS targeting.
- `asChild` swaps the wrapper for `Slot.Root` (e.g., to render a Next.js
  `<Link>` or react-router `<NavLink>` while keeping all classes).

## SidebarMenuAction

```ts
function SidebarMenuAction(
  props: React.ComponentProps<"button"> & {
    asChild?: boolean
    showOnHover?: boolean
  }
): JSX.Element
```

When `showOnHover` is true, the action is `opacity-0` on desktop and reveals
on hover, focus-within, or `data-state=open` of an adjacent menu.

## SidebarMenuBadge

```ts
function SidebarMenuBadge(props: React.ComponentProps<"div">): JSX.Element
```

Positioned absolute-right. Hidden in `collapsible=icon` collapsed state.

## SidebarMenuSkeleton

```ts
function SidebarMenuSkeleton(
  props: React.ComponentProps<"div"> & { showIcon?: boolean }
): JSX.Element
```

Generates a row with a randomized 50-90% width skeleton; optional 16x16
icon skeleton on the left.

## SidebarMenuSub / SidebarMenuSubItem

```ts
function SidebarMenuSub(props: React.ComponentProps<"ul">): JSX.Element
function SidebarMenuSubItem(props: React.ComponentProps<"li">): JSX.Element
```

The submenu list. Hidden via `group-data-[collapsible=icon]:hidden`.

## SidebarMenuSubButton

```ts
function SidebarMenuSubButton(
  props: React.ComponentProps<"a"> & {
    asChild?: boolean
    size?: "sm" | "md"        // default: "md"
    isActive?: boolean
  }
): JSX.Element
```

Renders an `<a>` by default (or any Slot child via `asChild`). Sets
`data-active={isActive}` and `data-size={size}`.

## Data attributes for CSS targeting

The Provider wrapper carries `data-slot="sidebar-wrapper"`. The desktop
Sidebar root carries:

- `data-state="expanded" | "collapsed"`
- `data-collapsible="offcanvas" | "icon" | ""` (empty when expanded)
- `data-variant="sidebar" | "floating" | "inset"`
- `data-side="left" | "right"`

Use these to target the descendant tree, e.g.:

```css
[data-state="collapsed"][data-collapsible="icon"] .my-thing { ... }
```

Most styling needs are already covered by the built-in classes; reach for
data-attribute selectors only for project-specific overrides.

## Exports (verbatim)

```ts
export {
  Sidebar,
  SidebarContent,
  SidebarFooter,
  SidebarGroup,
  SidebarGroupAction,
  SidebarGroupContent,
  SidebarGroupLabel,
  SidebarHeader,
  SidebarInput,
  SidebarInset,
  SidebarMenu,
  SidebarMenuAction,
  SidebarMenuBadge,
  SidebarMenuButton,
  SidebarMenuItem,
  SidebarMenuSkeleton,
  SidebarMenuSub,
  SidebarMenuSubButton,
  SidebarMenuSubItem,
  SidebarProvider,
  SidebarRail,
  SidebarSeparator,
  SidebarTrigger,
  useSidebar,
}
```
