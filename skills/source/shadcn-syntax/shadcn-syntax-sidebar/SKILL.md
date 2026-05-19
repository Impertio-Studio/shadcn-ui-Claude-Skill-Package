---
name: shadcn-syntax-sidebar
description: >
  Use when building an application shell, dashboard navigation, admin sidebar,
  collapsible side navigation, or any persistent vertical navigation rail in
  shadcn ui. Prevents the common mistakes of rendering Sidebar outside a
  SidebarProvider (throws "useSidebar must be used within a SidebarProvider"),
  omitting the 'use client' directive on the layout that hosts the Provider,
  choosing the wrong collapsible variant for desktop versus mobile, placing
  SidebarInset without a sibling Sidebar (layout collapse), and skipping the
  cookie-based persistence handshake in server-rendered frameworks. Covers all
  23 exported primitives (SidebarProvider, Sidebar, SidebarTrigger, SidebarRail,
  SidebarInset, SidebarInput, SidebarHeader, SidebarFooter, SidebarSeparator,
  SidebarContent, SidebarGroup, SidebarGroupLabel, SidebarGroupAction,
  SidebarGroupContent, SidebarMenu, SidebarMenuItem, SidebarMenuButton,
  SidebarMenuAction, SidebarMenuBadge, SidebarMenuSkeleton, SidebarMenuSub,
  SidebarMenuSubItem, SidebarMenuSubButton) plus the useSidebar hook, the three
  collapsible variants (offcanvas, icon, none), the three layout variants
  (sidebar, floating, inset), left and right sides, mobile sheet a11y handling,
  cookie persistence under the name sidebar_state, the Cmd or Ctrl plus B
  keyboard shortcut, and the block system (sidebar-01 through sidebar-16) as a
  ready-made copy source.
  Keywords: shadcn sidebar, SidebarProvider, SidebarTrigger, SidebarMenu,
  SidebarMenuButton, SidebarInset, SidebarRail, useSidebar, collapsible
  sidebar, sidebar offcanvas, sidebar icon, icon sidebar, sidebar mobile,
  dashboard sidebar, app shell, application shell sidebar, admin sidebar,
  sidebar persistence, sidebar cookie, sidebar_state cookie, sidebar
  keyboard shortcut, cmd b sidebar, ctrl b sidebar, sidebar-01 block,
  sidebar-02 block, sidebar block, sidebar variant inset, sidebar variant
  floating, sidebar side right, how do I make a sidebar, how do I add a
  collapsible sidebar, useSidebar must be used within a SidebarProvider,
  sidebar context error, sidebar not collapsing, sidebar not showing on
  mobile, sidebar layout collapse, persistent sidebar state.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn-syntax-sidebar

Sidebar is the largest single component in the shadcn catalog: 23 exported
primitives, one context hook, three collapsible behaviors, three layout
variants, two sides, separate desktop and mobile rendering paths, and built-in
cookie-based state persistence. It is intended as the foundation for the
"application shell" pattern (sidebar + topbar + main content area) and is the
shared substrate for every block named `sidebar-01` through `sidebar-16` in the
shadcn block registry.

The component lives in a single client file (`components/ui/sidebar.tsx`) and
internally composes Sheet (mobile drawer), Tooltip (icon-collapsed labels),
Input, Skeleton, and Button. Because the file begins with `"use client"` and
exports a React context, ALWAYS treat every consumer of Sidebar primitives as
a client subtree.

## Quick Reference

Install:

```bash
pnpm dlx shadcn@latest add sidebar
```

This also installs sheet, tooltip, input, skeleton, button, separator, and the
`use-mobile` hook as transitive dependencies.

Minimal application shell (Next.js app router layout, server component file
that imports a client provider wrapper):

```tsx
// app/layout.tsx (server component)
import { cookies } from "next/headers"
import { SidebarProvider, SidebarInset } from "@/components/ui/sidebar"
import { AppSidebar } from "@/components/app-sidebar"

export default async function Layout({ children }: { children: React.ReactNode }) {
  const cookieStore = await cookies()
  const defaultOpen = cookieStore.get("sidebar_state")?.value === "true"

  return (
    <SidebarProvider defaultOpen={defaultOpen}>
      <AppSidebar />
      <SidebarInset>{children}</SidebarInset>
    </SidebarProvider>
  )
}
```

```tsx
// components/app-sidebar.tsx
"use client"
import {
  Sidebar, SidebarContent, SidebarFooter, SidebarGroup, SidebarGroupContent,
  SidebarGroupLabel, SidebarHeader, SidebarMenu, SidebarMenuButton,
  SidebarMenuItem, SidebarRail,
} from "@/components/ui/sidebar"

export function AppSidebar() {
  return (
    <Sidebar collapsible="icon">
      <SidebarHeader>{/* logo / workspace switcher */}</SidebarHeader>
      <SidebarContent>
        <SidebarGroup>
          <SidebarGroupLabel>Application</SidebarGroupLabel>
          <SidebarGroupContent>
            <SidebarMenu>
              <SidebarMenuItem>
                <SidebarMenuButton tooltip="Home">Home</SidebarMenuButton>
              </SidebarMenuItem>
            </SidebarMenu>
          </SidebarGroupContent>
        </SidebarGroup>
      </SidebarContent>
      <SidebarFooter>{/* user menu */}</SidebarFooter>
      <SidebarRail />
    </Sidebar>
  )
}
```

## Primitive Map (all 23)

| Primitive | Renders | Required parent | Purpose |
|---|---|---|---|
| `SidebarProvider` | `<div>` wrapper + Context | Root of subtree | Owns open / openMobile / state / isMobile, writes cookie, binds Cmd/Ctrl+B |
| `Sidebar` | `<div>` (desktop) or `Sheet` (mobile) | `SidebarProvider` | The actual panel; carries side, variant, collapsible |
| `SidebarTrigger` | `<Button>` (ghost, size icon) | `SidebarProvider` | Toggles open state; place anywhere in the tree |
| `SidebarRail` | `<button>` (hidden hit-strip) | `Sidebar` | Click-to-toggle vertical strip on the sidebar edge |
| `SidebarInset` | `<main>` | `SidebarProvider` (sibling to `Sidebar`) | Main content region; reserves space, applies inset variant styling |
| `SidebarInput` | `<Input>` | `Sidebar` (typically header) | Search input pre-sized for sidebar |
| `SidebarHeader` | `<div>` | `Sidebar` | Sticky top region |
| `SidebarFooter` | `<div>` | `Sidebar` | Sticky bottom region |
| `SidebarSeparator` | `<Separator>` | `Sidebar` | Pre-styled separator using sidebar tokens |
| `SidebarContent` | `<div>` (scrollable) | `Sidebar` | Scrollable middle region between header and footer |
| `SidebarGroup` | `<div>` | `SidebarContent` | Logical grouping of menus |
| `SidebarGroupLabel` | `<div>` (or Slot if asChild) | `SidebarGroup` | Section label; hides on `collapsible=icon` |
| `SidebarGroupAction` | `<button>` (or Slot) | `SidebarGroup` | Top-right action button on a group |
| `SidebarGroupContent` | `<div>` | `SidebarGroup` | Wraps the actual menu inside the group |
| `SidebarMenu` | `<ul>` | `SidebarGroupContent` | The menu list |
| `SidebarMenuItem` | `<li>` | `SidebarMenu` | One row in the menu |
| `SidebarMenuButton` | `<button>` (or Slot) | `SidebarMenuItem` | The clickable row; supports tooltip when collapsed |
| `SidebarMenuAction` | `<button>` (or Slot) | `SidebarMenuItem` | Right-aligned action on a row |
| `SidebarMenuBadge` | `<div>` | `SidebarMenuItem` | Right-aligned numeric / status badge |
| `SidebarMenuSkeleton` | `<div>` with `<Skeleton>` | `SidebarMenuItem` | Loading placeholder row |
| `SidebarMenuSub` | `<ul>` (nested, hidden in icon mode) | `SidebarMenuItem` | Submenu container |
| `SidebarMenuSubItem` | `<li>` | `SidebarMenuSub` | Submenu row |
| `SidebarMenuSubButton` | `<a>` (or Slot) | `SidebarMenuSubItem` | Submenu link |

## SidebarProvider Props

ALWAYS wrap the application shell once at the top of the client subtree.

| Prop | Type | Default | Effect |
|---|---|---|---|
| `defaultOpen` | `boolean` | `true` | Uncontrolled initial open state |
| `open` | `boolean` | undefined | Controlled open state (pair with `onOpenChange`) |
| `onOpenChange` | `(open: boolean) => void` | undefined | Required if `open` is set |
| `className` | `string` | | Forwarded to the root `<div data-slot="sidebar-wrapper">` |
| `style` | `React.CSSProperties` | | Merged with `--sidebar-width` and `--sidebar-width-icon` |

The provider exposes the following CSS custom properties on its wrapper:

- `--sidebar-width` (default `16rem`)
- `--sidebar-width-icon` (default `3rem`)
- `--sidebar-width-mobile` (default `18rem`, applied only on the mobile Sheet)

Override widths via the `style` prop on `SidebarProvider`.

## Sidebar Props

| Prop | Type | Default | Effect |
|---|---|---|---|
| `side` | `"left" \| "right"` | `"left"` | Which edge to anchor to |
| `variant` | `"sidebar" \| "floating" \| "inset"` | `"sidebar"` | Visual treatment (flush, floating card, inset card paired with `SidebarInset`) |
| `collapsible` | `"offcanvas" \| "icon" \| "none"` | `"offcanvas"` | Desktop collapse behavior |
| `className` | `string` | | Forwarded to the inner `data-slot="sidebar-container"` |

## Collapsible Variants

| Value | Desktop behavior | Mobile behavior |
|---|---|---|
| `offcanvas` | Slides fully off-screen when collapsed (width 0) | Always renders as Sheet drawer |
| `icon` | Collapses to `--sidebar-width-icon` (3rem), shows icons only; `SidebarMenuButton tooltip="..."` becomes visible | Always renders as Sheet drawer |
| `none` | Non-collapsible, renders as a static `<div>` of width `--sidebar-width` | Same as desktop (no Sheet, no responsive switch) |

ALWAYS pair `collapsible="icon"` with the `tooltip` prop on every
`SidebarMenuButton` so collapsed labels remain accessible.

NEVER pick `collapsible="none"` when you also need a mobile drawer; that
variant intentionally opts out of the responsive Sheet path.

## Mobile vs Desktop

Internally, `Sidebar` reads `isMobile` from `useIsMobile()`:

- Below the mobile breakpoint, `Sidebar` mounts a `Sheet` (slide-in dialog).
  The mobile open state is owned by `openMobile` / `setOpenMobile`, NOT by
  `open` / `setOpen`.
- At and above the breakpoint, `Sidebar` renders an inline `<div>` driven by
  `open` / `setOpen`.

The mobile Sheet receives an `sr-only` `SheetTitle` ("Sidebar") and
`SheetDescription` to satisfy Radix Dialog a11y requirements. Consumers do
NOT add their own SheetTitle.

`SidebarTrigger` always calls `toggleSidebar()` which internally switches
between `setOpenMobile` and `setOpen` based on `isMobile`. One trigger covers
both surfaces.

## Cookie Persistence

`SidebarProvider` writes `document.cookie = "sidebar_state=<true|false>;
path=/; max-age=604800"` on every desktop open change (max-age 7 days).

For server-rendered frameworks (Next.js app router, Remix, TanStack Start),
read this cookie in the layout and pass the parsed boolean as `defaultOpen` to
prevent flicker between server and client render. See the Quick Reference
example for the canonical Next.js layout.

The cookie name is **`sidebar_state`** (underscore, not colon). Earlier
community examples used `sidebar:state`; the canonical shadcn source uses an
underscore.

## Keyboard Shortcut

The Provider binds a global `keydown` listener for `Cmd+B` (Mac) /
`Ctrl+B` (Windows / Linux). Pressing the shortcut calls `toggleSidebar()`.

If your application already binds Cmd/Ctrl+B (rare), call `event.stopPropagation()`
in your handler before the Provider's listener fires, or wrap the Provider's
subtree in a container that calls `event.preventDefault()` at capture phase.

## 'use client' Requirements

`components/ui/sidebar.tsx` starts with `"use client"`. EVERY file that
imports any Sidebar primitive (including `SidebarInset`) IS a client module,
because they all read or depend on the React context.

In Next.js app router:

- The layout file that wraps `<SidebarProvider>` MUST be a client component,
  OR it must import a client wrapper component that hosts the Provider. The
  canonical pattern (shown in Quick Reference) keeps the layout server-side
  and delegates the Provider to a client child component, so `cookies()` can
  still be read on the server.
- Custom sidebar component files (`app-sidebar.tsx`, `nav-main.tsx`, etc.)
  MUST declare `"use client"` at the top.

## useSidebar Hook

```ts
const {
  state,         // "expanded" | "collapsed"
  open,          // boolean (desktop)
  setOpen,       // (open: boolean | (prev: boolean) => boolean) => void
  openMobile,    // boolean (mobile Sheet)
  setOpenMobile, // (open: boolean) => void
  isMobile,      // boolean
  toggleSidebar, // () => void (switches based on isMobile)
} = useSidebar()
```

Throws `Error("useSidebar must be used within a SidebarProvider.")` when
called outside the Provider tree.

## Layout Variants

- `variant="sidebar"` (default): flush against the viewport edge, full-height
  bar, optional 1px border with neighboring content.
- `variant="floating"`: rendered as a floating rounded card with shadow,
  inset 2rem from the edge.
- `variant="inset"`: paired with `SidebarInset`. The Sidebar sits flush; the
  `<SidebarInset>` content area is a rounded card with its own shadow,
  offset 2rem from the sidebar. ALWAYS pair `variant="inset"` with a
  sibling `<SidebarInset>` or the layout collapses.

## Block System

The shadcn block library publishes a complete family of sidebar templates,
each a copy-paste-ready compositional example:

- `sidebar-01` through `sidebar-16` (see
  https://ui.shadcn.com/blocks/sidebar)

Each block ships as a multi-file composition (typically `app-sidebar.tsx`,
`nav-main.tsx`, `nav-projects.tsx`, `nav-user.tsx`, `team-switcher.tsx`,
`layout.tsx`). Install with `npx shadcn@latest add sidebar-07` (or any block
name). For real apps, ALWAYS start from a block rather than composing all 23
primitives from scratch.

## Companion Skills

- `shadcn-core-blocks` : block registry mechanics, copy semantics, style enum
- `shadcn-syntax-menu-primitives` : DropdownMenu inside SidebarMenuButton for
  user menus and workspace switchers
- `shadcn-impl-component-install` : `components.json`, aliases, the `add`
  command, and post-install fixups
- `shadcn-syntax-sheet` : how the mobile path renders (Sidebar reuses Sheet
  internally)

## Reference Files

- `references/methods.md` : full TypeScript signatures of every primitive,
  the cva schema for `SidebarMenuButton`, and the `useSidebar` hook
- `references/examples.md` : seven end-to-end compositions covering minimal,
  icon-collapsed, mobile offcanvas, persisted state, inset dashboard,
  groups with submenus, and header / footer composition
- `references/anti-patterns.md` : six common mistakes (missing Provider,
  Provider mounted on a server component, wrong collapsible variant,
  SidebarInset without sibling Sidebar, trigger outside the Provider tree,
  hand-rolled mobile rendering)

## Sources

- https://ui.shadcn.com/docs/components/radix/sidebar (canonical docs)
- https://github.com/shadcn-ui/ui/blob/main/apps/v4/registry/new-york-v4/ui/sidebar.tsx
  (source of truth, verified 2026-05-19)
- https://ui.shadcn.com/blocks/sidebar (block registry)
- Issue #5746 (49 reactions) : mobile DialogTitle a11y requirement (now
  handled internally by shadcn since the sr-only SheetTitle was added)
