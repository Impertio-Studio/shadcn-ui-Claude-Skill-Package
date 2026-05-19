# Sidebar Anti-Patterns

Six common mistakes when building with Sidebar, with the canonical fix.

## 1. Missing SidebarProvider (context error)

**Symptom:**

```
Error: useSidebar must be used within a SidebarProvider.
```

Or: the sidebar renders but the trigger does not toggle, no cookie is
written, and Cmd/Ctrl+B does nothing.

**Cause:**

`Sidebar`, `SidebarTrigger`, `SidebarMenuButton`, and every other primitive
reads the `SidebarContext` via `useSidebar()`. If no `<SidebarProvider>`
ancestor exists, the hook throws.

**Wrong:**

```tsx
// app/page.tsx
export default function Page() {
  return (
    <>
      <Sidebar>{/* throws inside */}</Sidebar>
      <main>...</main>
    </>
  )
}
```

**Right:**

ALWAYS wrap the application shell in `<SidebarProvider>` once, at the
highest layout level that contains both the `Sidebar` and its content
sibling.

```tsx
<SidebarProvider>
  <Sidebar>...</Sidebar>
  <SidebarInset>{children}</SidebarInset>
</SidebarProvider>
```

NEVER mount more than one Provider in the same subtree; the inner one wins
and the outer state desyncs.

## 2. Sidebar Outside the Provider Tree (sibling, not descendant)

**Symptom:**

`useSidebar must be used within a SidebarProvider.` even though a Provider
exists elsewhere in the tree.

**Cause:**

React context only flows DOWN the tree. A Provider in `<header>` does not
reach a Sidebar in `<aside>` if they are siblings.

**Wrong:**

```tsx
<>
  <SidebarProvider>
    <SidebarTrigger />
  </SidebarProvider>
  <Sidebar>...</Sidebar>   {/* sibling, not descendant */}
</>
```

**Right:**

ALWAYS make the Sidebar a descendant of the Provider:

```tsx
<SidebarProvider>
  <Sidebar>...</Sidebar>
  <main>
    <SidebarTrigger />
    {children}
  </main>
</SidebarProvider>
```

## 3. Wrong Collapsible Variant for the Surface

**Symptom:**

- Desktop: sidebar slides fully off-screen and the user has no edge to
  trigger from. (`collapsible="offcanvas"` was chosen for a dashboard.)
- Mobile: sidebar refuses to open as a drawer. (`collapsible="none"` was
  chosen.)
- Collapsed labels are invisible with no tooltip. (`collapsible="icon"` was
  chosen but no `tooltip` prop was set on `SidebarMenuButton`.)

**Cause:**

The three variants are not interchangeable. `collapsible="none"` opts out of
the responsive Sheet mount; `offcanvas` hides the rail entirely on collapse;
`icon` requires labels to be addressable as tooltips.

**Wrong:**

```tsx
<Sidebar collapsible="none">  {/* no mobile drawer ever */}
  <SidebarMenuButton>Home</SidebarMenuButton>
</Sidebar>
```

```tsx
<Sidebar collapsible="icon">
  <SidebarMenuButton>Home</SidebarMenuButton>  {/* no tooltip when collapsed */}
</Sidebar>
```

**Right:**

| Goal | Variant | Extra |
|---|---|---|
| Dashboard with mini-icon rail | `collapsible="icon"` | `tooltip` on every `SidebarMenuButton` |
| Mobile-first off-canvas drawer | `collapsible="offcanvas"` (default) | place a `SidebarTrigger` in the top bar |
| Always-visible static rail | `collapsible="none"` | accept that there is no responsive mobile sheet |

ALWAYS set `tooltip="..."` on each `SidebarMenuButton` when using
`collapsible="icon"`.

## 4. Missing 'use client' on the Provider Host

**Symptom (Next.js app router):**

```
Error: Could not find SidebarContext.
```

Or a hydration mismatch at the `<SidebarProvider>` wrapper, or the build
fails with `useState only works in Client Components`.

**Cause:**

`sidebar.tsx` starts with `"use client"`. Importing `SidebarProvider` into a
server component crosses the server/client boundary at the import edge, but
hooks (`useSidebar`) inside descendant components still need a client
ancestor that hosts the context. If you read `cookies()` server-side AND
host the Provider in the same file, that file MUST be a client component,
which breaks `cookies()` access.

**Wrong:**

```tsx
// app/layout.tsx (no "use client" - server file)
import { cookies } from "next/headers"
import { SidebarProvider } from "@/components/ui/sidebar"

export default async function Layout({ children }: { children: React.ReactNode }) {
  const cookieStore = await cookies()
  // ...
  return (
    <SidebarProvider defaultOpen={...}>
      <SomeClientThing />
      {children}                          {/* OK */}
    </SidebarProvider>
  )
}
```

The above ACTUALLY works in current Next.js because `SidebarProvider` is a
client component that boundary-crosses correctly. But if you wrap it in a
local component:

```tsx
// app/layout.tsx (server)
function ShellWrapper({ children }: { children: React.ReactNode }) {
  return <SidebarProvider>{children}</SidebarProvider>
  //     ^ this server-side wrapper cannot pass non-serializable defaults later
}
```

**Right:**

Either:

(a) Read the cookie in a server layout and pass `defaultOpen` directly to
`<SidebarProvider>` as a serializable boolean prop:

```tsx
// app/layout.tsx (server)
const defaultOpen = (await cookies()).get("sidebar_state")?.value !== "false"
return <SidebarProvider defaultOpen={defaultOpen}>{children}</SidebarProvider>
```

(b) Or place the entire shell in a `"use client"` component:

```tsx
// components/shell.tsx
"use client"
export function Shell({ children, defaultOpen }: {...}) {
  return <SidebarProvider defaultOpen={defaultOpen}>...</SidebarProvider>
}
```

ALWAYS mark every file that imports `useSidebar` (or any Sidebar primitive
hooked into context) with `"use client"`.

## 5. SidebarTrigger Outside the Sidebar Tree but Inside Provider (works) versus outside Provider entirely (broken)

**Symptom:**

Trigger renders, click does nothing, no error in console. Or:
`useSidebar must be used within a SidebarProvider.`

**Cause:**

`SidebarTrigger` is intentionally placeable ANYWHERE inside the Provider,
including a topbar that is a sibling of the Sidebar. But if it is mounted
OUTSIDE the Provider, the `useSidebar()` call in `SidebarTrigger` throws.

**Wrong:**

```tsx
<>
  <header>
    <SidebarTrigger />  {/* outside Provider - throws */}
  </header>
  <SidebarProvider>
    <Sidebar>...</Sidebar>
  </SidebarProvider>
</>
```

**Right:**

```tsx
<SidebarProvider>
  <Sidebar>...</Sidebar>
  <SidebarInset>
    <header>
      <SidebarTrigger />  {/* descendant of Provider - works */}
    </header>
    {children}
  </SidebarInset>
</SidebarProvider>
```

The trigger does NOT need to be a descendant of `<Sidebar>`. It only needs
to be a descendant of `<SidebarProvider>`.

## 6. SidebarInset Without a Sibling Sidebar (layout collapse)

**Symptom:**

The main content area lays out at full viewport width, ignoring the inset
variant styling. Or: a visible gap appears where the sidebar should be but
nothing renders.

**Cause:**

`SidebarInset` styles itself via `peer-data-[variant=inset]` selectors that
target a sibling element with `data-variant="inset"`. If no `<Sidebar>` is
mounted as a sibling, the peer selectors never match, the inset styling
collapses, and the layout looks broken.

**Wrong:**

```tsx
<SidebarProvider>
  <SidebarInset>
    {/* No <Sidebar> sibling, inset variant never applies */}
    {children}
  </SidebarInset>
</SidebarProvider>
```

```tsx
<SidebarProvider>
  <SidebarInset>
    <Sidebar variant="inset">...</Sidebar>  {/* Sidebar nested INSIDE inset, peer fails */}
    {children}
  </SidebarInset>
</SidebarProvider>
```

**Right:**

ALWAYS render `<Sidebar variant="inset">` as a direct sibling of
`<SidebarInset>` inside the same Provider, in that order:

```tsx
<SidebarProvider>
  <Sidebar variant="inset" collapsible="icon">...</Sidebar>
  <SidebarInset>{children}</SidebarInset>
</SidebarProvider>
```

The CSS peer selector requires sibling adjacency; nesting breaks the
relationship and the inset variant silently degrades.
