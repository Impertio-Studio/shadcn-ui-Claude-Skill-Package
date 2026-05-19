# Sidebar Examples

Seven end-to-end compositions. Each example is copy-paste-ready and uses
only the canonical shadcn imports. All examples assume `shadcn add sidebar`
has been run.

## 1. Minimal Sidebar Layout

The smallest viable Sidebar. No groups, no submenus, no persistence. Useful
for prototypes.

```tsx
// app/layout.tsx (Next.js app router, marked client at the top)
"use client"
import { SidebarProvider, SidebarTrigger } from "@/components/ui/sidebar"
import { AppSidebar } from "@/components/app-sidebar"

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <SidebarProvider>
          <AppSidebar />
          <main className="flex-1">
            <SidebarTrigger />
            {children}
          </main>
        </SidebarProvider>
      </body>
    </html>
  )
}
```

```tsx
// components/app-sidebar.tsx
"use client"
import {
  Sidebar, SidebarContent, SidebarMenu, SidebarMenuButton, SidebarMenuItem,
} from "@/components/ui/sidebar"

export function AppSidebar() {
  return (
    <Sidebar>
      <SidebarContent>
        <SidebarMenu>
          <SidebarMenuItem>
            <SidebarMenuButton asChild>
              <a href="/">Home</a>
            </SidebarMenuButton>
          </SidebarMenuItem>
          <SidebarMenuItem>
            <SidebarMenuButton asChild>
              <a href="/settings">Settings</a>
            </SidebarMenuButton>
          </SidebarMenuItem>
        </SidebarMenu>
      </SidebarContent>
    </Sidebar>
  )
}
```

## 2. Icon-Collapsed Sidebar with Tooltips

Desktop collapses to a 3rem icon rail. ALWAYS pair `collapsible="icon"`
with the `tooltip` prop on every `SidebarMenuButton`.

```tsx
"use client"
import { Home, Settings, Users } from "lucide-react"
import {
  Sidebar, SidebarContent, SidebarMenu, SidebarMenuButton, SidebarMenuItem,
  SidebarRail,
} from "@/components/ui/sidebar"

export function AppSidebar() {
  return (
    <Sidebar collapsible="icon">
      <SidebarContent>
        <SidebarMenu>
          <SidebarMenuItem>
            <SidebarMenuButton tooltip="Home">
              <Home />
              <span>Home</span>
            </SidebarMenuButton>
          </SidebarMenuItem>
          <SidebarMenuItem>
            <SidebarMenuButton tooltip="Users">
              <Users />
              <span>Users</span>
            </SidebarMenuButton>
          </SidebarMenuItem>
          <SidebarMenuItem>
            <SidebarMenuButton tooltip="Settings">
              <Settings />
              <span>Settings</span>
            </SidebarMenuButton>
          </SidebarMenuItem>
        </SidebarMenu>
      </SidebarContent>
      <SidebarRail />
    </Sidebar>
  )
}
```

The `<span>` is hidden in icon mode via the built-in
`group-data-[collapsible=icon]` styling; the icon child stays visible.

## 3. Mobile Offcanvas (Default Behavior)

`collapsible="offcanvas"` is the default. On screens below the mobile
breakpoint, `Sidebar` mounts as a `Sheet`. Trigger toggles open the mobile
sheet automatically.

```tsx
"use client"
import { Menu } from "lucide-react"
import {
  Sidebar, SidebarContent, SidebarProvider, SidebarTrigger,
  SidebarMenu, SidebarMenuButton, SidebarMenuItem,
} from "@/components/ui/sidebar"

export default function Page() {
  return (
    <SidebarProvider>
      <Sidebar collapsible="offcanvas">
        <SidebarContent>
          <SidebarMenu>
            <SidebarMenuItem>
              <SidebarMenuButton>Home</SidebarMenuButton>
            </SidebarMenuItem>
            <SidebarMenuItem>
              <SidebarMenuButton>Search</SidebarMenuButton>
            </SidebarMenuItem>
          </SidebarMenu>
        </SidebarContent>
      </Sidebar>
      <main className="flex-1">
        <header className="flex h-14 items-center gap-2 border-b px-4">
          <SidebarTrigger>
            <Menu />
          </SidebarTrigger>
          <h1 className="font-semibold">My App</h1>
        </header>
      </main>
    </SidebarProvider>
  )
}
```

`SidebarTrigger` automatically calls the correct toggle (mobile or desktop)
based on `isMobile`. No `useMediaQuery` plumbing required in user code.

## 4. Persisted Sidebar State (Next.js App Router)

Read the `sidebar_state` cookie on the server and pass `defaultOpen` to the
Provider to avoid the open / collapsed flicker on first paint.

```tsx
// app/layout.tsx (server component)
import { cookies } from "next/headers"
import { SidebarProvider, SidebarInset } from "@/components/ui/sidebar"
import { AppSidebar } from "@/components/app-sidebar"

export default async function Layout({ children }: { children: React.ReactNode }) {
  const cookieStore = await cookies()
  const defaultOpen = cookieStore.get("sidebar_state")?.value !== "false"

  return (
    <html lang="en">
      <body>
        <SidebarProvider defaultOpen={defaultOpen}>
          <AppSidebar />
          <SidebarInset>{children}</SidebarInset>
        </SidebarProvider>
      </body>
    </html>
  )
}
```

The cookie is written by the Provider client-side on every desktop open
change with `max-age=604800` (7 days). The default-true semantics match the
upstream behavior (sidebar starts expanded unless the cookie says otherwise).

## 5. Dashboard Layout with SidebarInset

`variant="inset"` produces the modern "card-in-a-card" dashboard look. The
`SidebarInset` element MUST be a sibling of `Sidebar` inside the Provider.

```tsx
"use client"
import {
  Sidebar, SidebarContent, SidebarInset, SidebarMenu, SidebarMenuButton,
  SidebarMenuItem, SidebarProvider, SidebarTrigger,
} from "@/components/ui/sidebar"

export default function DashboardLayout({ children }: { children: React.ReactNode }) {
  return (
    <SidebarProvider>
      <Sidebar variant="inset" collapsible="icon">
        <SidebarContent>
          <SidebarMenu>
            <SidebarMenuItem>
              <SidebarMenuButton tooltip="Overview" isActive>
                Overview
              </SidebarMenuButton>
            </SidebarMenuItem>
            <SidebarMenuItem>
              <SidebarMenuButton tooltip="Reports">Reports</SidebarMenuButton>
            </SidebarMenuItem>
          </SidebarMenu>
        </SidebarContent>
      </Sidebar>
      <SidebarInset>
        <header className="flex h-12 items-center gap-2 border-b px-4">
          <SidebarTrigger />
          <h1 className="font-semibold">Dashboard</h1>
        </header>
        <div className="p-4">{children}</div>
      </SidebarInset>
    </SidebarProvider>
  )
}
```

## 6. Groups with a Submenu

Two groups, the second of which has an item with a submenu. Submenus
auto-hide in `collapsible="icon"` collapsed state.

```tsx
"use client"
import { ChevronRight, Folder, Inbox, Star } from "lucide-react"
import {
  Sidebar, SidebarContent, SidebarGroup, SidebarGroupContent,
  SidebarGroupLabel, SidebarMenu, SidebarMenuButton, SidebarMenuItem,
  SidebarMenuSub, SidebarMenuSubButton, SidebarMenuSubItem,
} from "@/components/ui/sidebar"

export function AppSidebar() {
  return (
    <Sidebar>
      <SidebarContent>
        <SidebarGroup>
          <SidebarGroupLabel>Inbox</SidebarGroupLabel>
          <SidebarGroupContent>
            <SidebarMenu>
              <SidebarMenuItem>
                <SidebarMenuButton>
                  <Inbox />
                  <span>All mail</span>
                </SidebarMenuButton>
              </SidebarMenuItem>
              <SidebarMenuItem>
                <SidebarMenuButton>
                  <Star />
                  <span>Starred</span>
                </SidebarMenuButton>
              </SidebarMenuItem>
            </SidebarMenu>
          </SidebarGroupContent>
        </SidebarGroup>

        <SidebarGroup>
          <SidebarGroupLabel>Folders</SidebarGroupLabel>
          <SidebarGroupContent>
            <SidebarMenu>
              <SidebarMenuItem>
                <SidebarMenuButton>
                  <Folder />
                  <span>Projects</span>
                  <ChevronRight className="ml-auto" />
                </SidebarMenuButton>
                <SidebarMenuSub>
                  <SidebarMenuSubItem>
                    <SidebarMenuSubButton href="/projects/alpha">
                      Alpha
                    </SidebarMenuSubButton>
                  </SidebarMenuSubItem>
                  <SidebarMenuSubItem>
                    <SidebarMenuSubButton href="/projects/beta" isActive>
                      Beta
                    </SidebarMenuSubButton>
                  </SidebarMenuSubItem>
                </SidebarMenuSub>
              </SidebarMenuItem>
            </SidebarMenu>
          </SidebarGroupContent>
        </SidebarGroup>
      </SidebarContent>
    </Sidebar>
  )
}
```

For collapsible submenu animation, wrap `<SidebarMenuItem>` with a
`<Collapsible>` from `@/components/ui/collapsible` and put the button in
`<CollapsibleTrigger asChild>`. See the `sidebar-07` block for the canonical
pattern.

## 7. Header Logo + Footer User Menu

The full "shell" composition: branding header, scrollable content, footer
user menu, edge rail for quick toggle.

```tsx
"use client"
import { ChevronUp, GalleryVerticalEnd, User2 } from "lucide-react"
import {
  Sidebar, SidebarContent, SidebarFooter, SidebarHeader, SidebarMenu,
  SidebarMenuButton, SidebarMenuItem, SidebarRail, SidebarSeparator,
} from "@/components/ui/sidebar"
import {
  DropdownMenu, DropdownMenuContent, DropdownMenuItem, DropdownMenuTrigger,
} from "@/components/ui/dropdown-menu"

export function AppSidebar() {
  return (
    <Sidebar collapsible="icon">
      <SidebarHeader>
        <SidebarMenu>
          <SidebarMenuItem>
            <SidebarMenuButton size="lg">
              <div className="flex aspect-square size-8 items-center justify-center rounded-lg bg-sidebar-primary text-sidebar-primary-foreground">
                <GalleryVerticalEnd className="size-4" />
              </div>
              <div className="flex flex-col gap-0.5 leading-none">
                <span className="font-semibold">Acme Inc</span>
                <span className="text-xs">Enterprise</span>
              </div>
            </SidebarMenuButton>
          </SidebarMenuItem>
        </SidebarMenu>
      </SidebarHeader>

      <SidebarSeparator />

      <SidebarContent>{/* groups go here */}</SidebarContent>

      <SidebarFooter>
        <SidebarMenu>
          <SidebarMenuItem>
            <DropdownMenu>
              <DropdownMenuTrigger asChild>
                <SidebarMenuButton>
                  <User2 />
                  <span>username</span>
                  <ChevronUp className="ml-auto" />
                </SidebarMenuButton>
              </DropdownMenuTrigger>
              <DropdownMenuContent side="top" className="w-(--radix-popper-anchor-width)">
                <DropdownMenuItem>Account</DropdownMenuItem>
                <DropdownMenuItem>Billing</DropdownMenuItem>
                <DropdownMenuItem>Sign out</DropdownMenuItem>
              </DropdownMenuContent>
            </DropdownMenu>
          </SidebarMenuItem>
        </SidebarMenu>
      </SidebarFooter>

      <SidebarRail />
    </Sidebar>
  )
}
```

The DropdownMenu inside `SidebarMenuButton` is the canonical user-menu
pattern; the `w-(--radix-popper-anchor-width)` arbitrary Tailwind value
makes the dropdown match the trigger's width.

For a complete shell with team-switcher, search, nav-main, nav-projects, and
nav-user (all wired up), install the `sidebar-07` block:

```bash
npx shadcn@latest add sidebar-07
```
