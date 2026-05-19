---
name: shadcn-impl-rsc-vs-client-boundaries
description: >
  Use when wiring shadcn components into a Next.js App Router project, when
  deciding whether a given shadcn primitive needs a `"use client"` directive,
  when seeing the "Error: useState is not a function" or "Cannot read
  properties of null (reading useContext)" hydration errors from a Radix
  primitive in a server component, when designing the
  server-shell-plus-client-island layout for a page, when wrapping a
  context-providing library (next-themes, TanStack Query, jotai, zustand)
  for the App Router, or when the `rsc` field in `components.json` is set
  but the generated component file is still missing the directive.
  Prevents the over-clientification anti-pattern of marking the whole
  layout `"use client"` (defeats the streaming + RSC bundle savings of
  App Router), the missing-directive crash where a Radix-state component
  (Dialog, Select, Sidebar) renders on the server and throws
  `useState is not a function`, the forgotten Provider wrapper where
  next-themes ThemeProvider is imported directly into a server
  layout and crashes on hydration, the boundary-violation pattern where
  a server component tries to pass `onClick` or a class instance into
  a client child, and the unstable-directive anti-pattern of trying to
  apply `"use client"` conditionally or inline.
  Covers the per-primitive RSC compatibility matrix for the full
  evergreen-2026 component set (Card, Button, Badge, Alert, Skeleton,
  Separator, Label, Avatar, AspectRatio, Table, Typography are
  RSC-safe ; Dialog, Sheet, Drawer, Popover, Tooltip, HoverCard, Select,
  Combobox, Command, Form, Sonner, Tabs, Accordion, Carousel, Sidebar,
  Calendar, Toggle, Switch, Slider, Checkbox, Radio Group, Collapsible,
  ContextMenu, DropdownMenu, Menubar, NavigationMenu, ScrollArea require
  `"use client"`), the meaning and exact behaviour of `components.json`
  `rsc: true`, the file-level placement rule for the `"use client"`
  directive, the hybrid server-shell-plus-client-island composition
  pattern, the Provider-wrapper pattern (one-line `"use client"`
  wrapper around next-themes, TanStack Query, jotai), the rule that
  server components may pass a client component as `children` but not
  as an event-handler prop, and the serialisation requirement for any
  data crossing the boundary.
  Keywords: use client, "use client" directive, react server components,
  rsc, app router, Next.js server component, Next.js App Router,
  rsc:true components.json, where to put use client, server component
  cannot use hooks, server component hydration error, hydration mismatch,
  client boundary, RSC boundary, RSC compatibility, server component vs
  client component, server-component-cannot-pass-onclick, can I use
  Dialog in a server component, can I use shadcn Card in a server
  component, is Button server safe, ThemeProvider use client, next-themes
  use client, TanStack Query Provider use client, Sidebar server side,
  Form server side, Sonner Toaster server side, shadcn rsc flag, shadcn
  next.js, blank screen, app crashes on hydration, hydration error
  shadcn, useState is not a function, why is my dialog not opening on
  the server.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026, Next.js App Router (14+ / 15.x)."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui : RSC vs Client Boundaries

This skill is the rulebook for placing `"use client"` directives in a
Next.js App Router project that consumes shadcn primitives. It covers
which primitives are RSC-safe, which require the client boundary, where
the directive belongs in the file tree, how the `components.json` `rsc`
flag affects what the CLI emits, and how to build a page as a server
shell that hosts small client islands.

Companion skills :

- `shadcn-core-cli` : `components.json` schema including the `rsc` flag.
- `shadcn-impl-component-install` : the install workflow that emits files
  with or without the directive.
- `shadcn-syntax-form` : the Form primitive (always client) and how it
  composes inside a server-rendered page.
- `shadcn-syntax-dialog` : the Dialog primitive (always client) and the
  `Trigger`+`Content` boundary patterns.
- `shadcn-agents-rsc-boundary-validator` (B14) : the automated validator
  that lints a project's `"use client"` placement against this matrix.

## Quick Reference : The Boundary Rule

```
The "use client" directive marks the BOUNDARY, not the dependency.
Everything imported below a "use client" file is bundled for the client.
Everything above is server-only.
```

ALWAYS draw the boundary as DEEP in the tree as possible. NEVER mark a
`layout.tsx` or `page.tsx` `"use client"` if a smaller leaf can carry
the directive instead. The whole point of App Router is to ship less
JavaScript ; a top-level `"use client"` defeats that.

```
ALWAYS : page.tsx (server) -> ClientIsland.tsx ("use client") -> Dialog
NEVER  : page.tsx ("use client") -> Dialog
```

## Quick Reference : Per-Primitive Compatibility Matrix

The full 60-row matrix lives in `references/methods.md`. Top-line summary :

```
RSC-SAFE (no "use client" required, pure presentation) :
  Alert, AlertDialog content body, AspectRatio, Avatar, Badge, Breadcrumb,
  Button, Card, Empty, Field, Input, InputGroup, Item, Kbd, Label, Pagination,
  Separator, Skeleton, Spinner, Table primitives (Table/TableRow/...),
  Textarea, Typography elements.

CLIENT-REQUIRED ("use client" mandatory, uses Radix state / hooks / browser API) :
  Accordion, AlertDialog (root + trigger), Calendar, Carousel, Checkbox,
  Collapsible, Combobox, Command, ContextMenu, Dialog, Drawer,
  DropdownMenu, Form (react-hook-form), HoverCard, Input OTP, Menubar,
  Native Select (interaction states), NavigationMenu, Popover, Progress
  (animated variant), RadioGroup, Resizable, ScrollArea, Select, Sheet,
  Sidebar (uses useSidebar), Slider, Sonner (Toaster mount), Switch, Tabs,
  Toggle, ToggleGroup, Tooltip.
```

Rule of thumb : if the primitive wraps a Radix package, it carries
`"use client"`. Static-HTML primitives (Card, Badge, Table, ...) do not.

## Quick Reference : components.json `rsc` Flag

```jsonc
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "rsc": true,
  "tsx": true,
  "style": "new-york",
  "tailwind": { ... },
  "aliases": { ... }
}
```

```
rsc: true   -> CLI prepends "use client" to generated files that need it.
                Generated Dialog/Sheet/Drawer/... start with "use client".
                Generated Card/Badge/Button/... do NOT (they are RSC-safe).
rsc: false  -> CLI prepends "use client" to EVERY generated component.
                Use this when the project is not App Router (Pages Router,
                Vite + React, Astro client islands).
```

ALWAYS set `rsc: true` for App Router projects. NEVER hand-edit the
directive after `add` ; re-run `shadcn add <name>` if the directive
appears wrong.

## Quick Reference : Where the Directive Goes

```
"use client"

import * as React from "react"
import { Dialog, DialogTrigger, DialogContent } from "@/components/ui/dialog"
// ... rest of the file
```

ALWAYS place `"use client"` at the very TOP of the file, BEFORE any
import. The directive is FILE-LEVEL ; it applies to every export in the
file. NEVER try to mark a single function or component within a file as
client-only ; the directive does not support that and the build will
silently treat the whole module as a client module.

NEVER quote the directive with backticks. ALWAYS use double quotes or
single quotes : `"use client"` or `'use client'`. A template literal or
a conditional is silently ignored by the bundler.

## Decision Tree : Does This File Need `"use client"` ?

```
Does the file ...

1. Import a hook (useState, useEffect, useReducer, useRef, useContext,
   useMemo, useCallback, useTransition, useDeferredValue, useId,
   useSyncExternalStore, custom hooks) ?
                                       YES -> "use client"

2. Import a shadcn primitive from the CLIENT-REQUIRED list above ?
                                       YES -> "use client"

3. Define an event handler (onClick, onChange, onSubmit, onKeyDown, ...) ?
                                       YES -> "use client"

4. Access browser globals (window, document, localStorage,
   navigator, IntersectionObserver, ResizeObserver) ?
                                       YES -> "use client"

5. Use a Context Provider (ThemeProvider, QueryClientProvider, ...) ?
                                       YES -> "use client" (the WRAPPER, not the consumer page)

6. Render a server-async component (async function default export with
   a server-side fetch) ?
                                       YES -> the page stays SERVER ;
                                              extract any interactive
                                              children into a "use client"
                                              child file.

7. None of the above ?                 NO  -> leave it RSC-safe.
```

## Pattern : Server Shell + Client Island

The canonical App Router page : data on the server, interaction on the
client, no `"use client"` on the page itself.

```tsx
// app/products/page.tsx                                 (SERVER)
import { Card, CardHeader, CardTitle, CardContent } from "@/components/ui/card"
import { ProductActions } from "./product-actions"      // client island
import { db } from "@/lib/db"

export default async function ProductsPage() {
  const products = await db.product.findMany()
  return (
    <div className="grid gap-4">
      {products.map((p) => (
        <Card key={p.id}>
          <CardHeader><CardTitle>{p.name}</CardTitle></CardHeader>
          <CardContent>
            <p>{p.description}</p>
            <ProductActions productId={p.id} />        {/* "use client" boundary lives in this file */}
          </CardContent>
        </Card>
      ))}
    </div>
  )
}
```

The `Card` family stays in the server module because it is RSC-safe.
The interactive bits (delete buttons, popover menus, ...) live in
`product-actions.tsx` which carries the directive.

See `references/examples.md` for the full file body of
`product-actions.tsx`, the ThemeProvider wrapper, a TanStack Query
provider wrapper, and a Form-in-server-page recipe.

## Pattern : Provider Wrapper (next-themes, TanStack Query, jotai, ...)

Every React context Provider lives in a thin one-purpose client file :

```tsx
// components/theme-provider.tsx                        ("use client")
"use client"
import { ThemeProvider as NextThemesProvider } from "next-themes"
import type * as React from "react"

export function ThemeProvider({
  children,
  ...props
}: React.ComponentProps<typeof NextThemesProvider>) {
  return <NextThemesProvider {...props}>{children}</NextThemesProvider>
}
```

```tsx
// app/layout.tsx                                        (SERVER)
import { ThemeProvider } from "@/components/theme-provider"

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body>
        <ThemeProvider attribute="class" defaultTheme="system" enableSystem>
          {children}
        </ThemeProvider>
      </body>
    </html>
  )
}
```

ALWAYS put `"use client"` on the WRAPPER, NEVER on `layout.tsx`. The
layout stays a server component and STILL renders the client provider
because server components may RENDER client components as descendants ;
the inverse (client renders server) requires the children-prop pattern.

ALWAYS add `suppressHydrationWarning` on `<html>` when using `next-themes`
to avoid React 18+ FOUC warnings (verified, recurring as issue #5552 on
the shadcn-ui repo with 99 reactions). The directive is NOT a workaround
for missing `"use client"` ; it is a separate fix for the legitimate
class-mismatch on first paint.

## Pattern : Passing Data Across the Boundary

```
SERVER passes to CLIENT  : only serialisable data
                           (string, number, boolean, plain object, plain array,
                            Date, Map, Set since React 19, null, undefined).
SERVER passes to CLIENT  : NEVER class instances, functions, symbols,
                           Promises, JSX with onClick props.
CLIENT renders SERVER    : ONLY via the `children` slot. Pass server JSX
                           as a children prop from a server parent.
```

```tsx
// server page
<ClientIsland data={serialisedProduct} />              {/* OK */}
<ClientIsland onSelect={() => save()} />               {/* COMPILE ERROR : Functions cannot be passed to Client Components */}
<ClientIsland>{<ServerChild />}</ClientIsland>         {/* OK : children slot */}
```

Pass DATA in, host SERVER JSX via `children`, return interaction
HANDLERS as Server Actions (`"use server"`) inside form `action` props or
returned from server modules.

## Pattern : Forms in a Server Page

The `Form` primitive (react-hook-form) is always a client component.
Drop it inside the server page :

```tsx
// app/contact/page.tsx                                  (SERVER)
import { ContactForm } from "./contact-form"
export default function ContactPage() {
  return (
    <main>
      <h1>Contact</h1>
      <ContactForm />                                    {/* "use client" inside */}
    </main>
  )
}
```

`ContactForm` carries the directive ; the surrounding `<h1>` and the
page wrapper stay server-rendered, so only the form bundle ships.

For full code see `references/examples.md` §5.

## Anti-Patterns

See `references/anti-patterns.md` for the full catalogue. Top 6 :

1. `"use client"` on the entire `app/layout.tsx` "to be safe" : every
   page becomes a client page ; you have just deleted RSC from the
   project. Move the directive to the smallest leaf.
2. Importing Dialog into a server component without a client wrapper.
   Build fails with `useState is not a function` because Radix Dialog
   calls `React.useState` during render.
3. Forgetting `"use client"` on the next-themes ThemeProvider wrapper.
   Hydration error : `useTheme is not a function` because the hook
   resolves to its server stub.
4. Server component passing an `onClick` handler to a client child.
   `Error: Functions cannot be passed directly to Client Components
   unless you explicitly expose it by marking it with "use server".`
5. Client component receiving a non-serialisable prop (class instance,
   Map of class instances, Date with extra prototype methods). Silent
   mis-render after hydration.
6. Conditional or template-literal directive (`` `use client` ``, or
   `if (...) "use client"` somewhere mid-file). The bundler treats this
   as a normal string and the file ships as a server module ; the
   Radix import then throws at render.

## When to Hand Off to Other Skills

- Designing a new primitive ?                 -> `shadcn-syntax-{primitive}`.
- Choosing where to drop a Form ?             -> `shadcn-syntax-form`.
- Composing Dialog + Drawer responsively ?    -> `shadcn-impl-responsive-dialog-drawer`.
- Re-running `shadcn add` after toggling the
  `rsc` flag ?                                -> `shadcn-impl-component-install`.
- Auditing an existing project's directive
  placement ?                                 -> `shadcn-agents-rsc-boundary-validator` (B14).

## Reference Files

- `references/methods.md` : full 60-row per-primitive RSC matrix +
  `components.json` `rsc` field reference.
- `references/examples.md` : five worked examples : server-page +
  Card render, ProductActions client island, ThemeProvider wrapper,
  TanStack Query provider wrapper, Form-in-server-page.
- `references/anti-patterns.md` : six anti-patterns with diagnosis,
  fix, and verification step.
