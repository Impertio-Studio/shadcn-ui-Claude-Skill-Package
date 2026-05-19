# Anti-Patterns : RSC Boundary Validator

Six anti-patterns the validator catches, plus one bandaid trap. Each
entry lists the symptom, the diagnosis (which rule the validator
fires), the fix, and the verification step that confirms the fix.

Verified against the shadcn ui evergreen-2026 catalogue,
https://react.dev/reference/rsc/use-client, and
https://nextjs.org/docs/app/building-your-application/rendering/composition-patterns.
Last verified : 2026-05-19.

---

## AP-1 : `"use client"` on `app/layout.tsx` (defeats RSC)

### Symptom

```tsx
// app/layout.tsx
"use client"                                  // <-- WRONG
import "./globals.css"
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return <html><body>{children}</body></html>
}
```

The app still builds and renders. Bundle size balloons : every page in
the app is now a client tree. RSC streaming, server-only data fetches
without an extra round-trip, and the JavaScript bundle savings of App
Router all evaporate.

### Why it fails

`"use client"` marks the BOUNDARY, not just the current file. Every
import below a client file is treated as client too. When the boundary
is at the root layout, EVERY page becomes a client page. The whole
point of App Router (`ship less JS, render data on the server`) is
silently lost.

This is the most common AI-generated defect because it makes the build
green and the dev pages work. The cost is invisible until the
production bundle analyser shows 800 kB of JavaScript on a static
landing page.

### Validator diagnosis

Rule 3 fires : `app/layout.tsx` / `app/page.tsx` / `app/template.tsx`
MUST NOT start with the directive. Verdict : **FAIL**.

### Fix

Move the directive to the smallest leaf that actually needs it. If the
layout uses `next-themes`, extract the Provider into a one-file client
wrapper :

```tsx
// app/layout.tsx (server, fixed)
import { ThemeProvider } from "@/components/theme-provider"
import "./globals.css"

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

### Verification

```bash
head -1 app/layout.tsx                # -> import ...   NOT "use client"
rg -n '^"use client"' app/layout.tsx app/page.tsx app/template.tsx   # -> empty
```

Bundle-analyser comparison : initial JS for the landing page drops
from ~800 kB to ~30 kB after the fix on a representative shadcn app.

---

## AP-2 : Missing `"use client"` on a Dialog / Sheet / Drawer File

### Symptom

```tsx
// components/ui/dialog.tsx (DEFECT : no directive on line 1)
import * as React from "react"
import * as DialogPrimitive from "@radix-ui/react-dialog"
// ... rest of the wrapper
```

Server-render crash :

```
Error: useState is not a function
    at DialogPrimitive.Root (.../@radix-ui/react-dialog/...)
    at renderToString (.../react-dom/server.edge.js:...)
```

Or, on the App Router error overlay :
`Cannot read properties of null (reading 'useContext')`.

### Why it fails

Radix Dialog calls `React.useState` during its component initialisation.
React hooks have no server implementation in the React Server
Components runtime ; they resolve to `null` (or undefined) on the
server. The Radix Dialog explodes the moment it tries to call the
hook. Same root cause for Sheet (extends Dialog), Drawer (vaul +
useState), Popover, Tooltip, HoverCard, Form (react-hook-form),
Select, Combobox (Popover + Command), Command (cmdk), Sonner Toaster,
Tabs, Accordion, Carousel (embla), Sidebar (useSidebar hook),
Calendar (react-day-picker), DropdownMenu / ContextMenu / Menubar /
NavigationMenu, Switch, Slider, Checkbox, RadioGroup, Collapsible,
ScrollArea, InputOTP, Toggle, ToggleGroup, Resizable, Chart (Recharts).

The full client-required list lives in `methods.md` §4.

### Validator diagnosis

Rule 2 fires : the file is in the CLIENT-REQUIRED list and lacks
the directive. Verdict : **FAIL**.

### Fix

Regenerate the file via the CLI rather than hand-editing the
directive ; the CLI also restores any other fields that may have
drifted (cva variants, import aliases) :

```bash
shadcn add dialog --overwrite       # restores "use client" + the canonical body
```

Hand-edit fallback :

```tsx
// components/ui/dialog.tsx (fixed)
"use client"
import * as React from "react"
import * as DialogPrimitive from "@radix-ui/react-dialog"
// ...
```

### Verification

```bash
head -1 components/ui/dialog.tsx       # -> "use client"
npm run build                           # -> succeeds, no useState error
```

---

## AP-3 : Conditional / Template-Literal Directive (Silently Ignored)

### Symptom

```tsx
// components/feature.tsx
const isClient = typeof window !== "undefined"
if (isClient) "use client"               // <-- WRONG, ignored

import { Dialog } from "@radix-ui/react-dialog"
// ...
```

Or :

```tsx
`use client`                              // <-- WRONG, template literal

import { Dialog } from "@radix-ui/react-dialog"
```

Or :

```tsx
// some comment
;("use client")                           // <-- WRONG, expression
```

The build does NOT warn. The file ships as a server module. The Radix
import then throws `useState is not a function` at render.

### Why it fails

The bundler (Turbopack, webpack via `next/dist`, Vite's React plugin)
recognises EXACTLY ONE shape : a top-of-file string literal with the
text `"use client"` or `'use client'`, on its own statement, after at
most blank lines or block comments. Anything else (template literal,
expression statement, conditional, parenthesised expression, line
comment ABOVE the directive followed by code, indented directive
inside a function) is treated as a normal JavaScript string and
discarded.

This is silent because the bundler does not lint user code for
misplaced directives ; it only LOOKS for the correct shape.

### Validator diagnosis

Rule 4 fires : the directive is not well-formed. The validator's
ripgrep patterns detect backticks, conditionals, and indented
directives. Verdict : **FAIL**.

### Fix

```tsx
"use client"                              // line 1, double-quoted, statement

import { Dialog } from "@/components/ui/dialog"
// ...
```

Place the directive as the FIRST non-blank, non-comment line.
Double-quote it. Make it a standalone statement (no semicolon prefix,
no parentheses, no conditional).

### Verification

```bash
rg -n '`use client`'                     # -> empty
rg -n 'if\s*\([^)]+\)\s*"use client"'   # -> empty
head -1 <suspect-file>                   # -> "use client"
```

---

## AP-4 : Server Component Passes `onClick` to a Client Child

### Symptom

```tsx
// app/products/page.tsx (server)
import { ProductActions } from "@/components/product-actions"   // "use client"

export default async function ProductsPage() {
  const products = await db.product.findMany()
  return (
    <ul>
      {products.map((p) => (
        <ProductActions
          key={p.id}
          productId={p.id}
          onDelete={() => deleteProduct(p.id)}                  // <-- WRONG
        />
      ))}
    </ul>
  )
}
```

Build error :

```
Error: Functions cannot be passed directly to Client Components
unless you explicitly expose it by marking it with "use server".
```

### Why it fails

The server-to-client boundary serialises every prop. Functions are not
serialisable. The only function shape allowed across the boundary is a
Server Action (`"use server"`) imported from a server module ; the
runtime then sends a stable ID to the client and the React runtime
invokes the server function over the network when the client calls it.

An inline arrow function has no stable identity, no network mapping,
and the runtime correctly refuses it.

### Validator diagnosis

Rule 6 fires : a server file passes an inline arrow / function literal
to a client child. Verdict : **FAIL (candidate, confirm with
`tsc --noEmit` output)**.

### Fix

Move the handler into a Server Action and pass the action :

```ts
// app/products/actions.ts
"use server"
import { db } from "@/lib/db"

export async function deleteProductAction(productId: string) {
  await db.product.delete({ where: { id: productId } })
}
```

```tsx
// app/products/page.tsx (server, fixed)
import { ProductActions } from "@/components/product-actions"
import { deleteProductAction } from "./actions"

export default async function ProductsPage() {
  const products = await db.product.findMany()
  return (
    <ul>
      {products.map((p) => (
        <ProductActions
          key={p.id}
          productId={p.id}
          deleteAction={deleteProductAction}                    // <-- OK
        />
      ))}
    </ul>
  )
}
```

Alternative pattern : move the handler INTO the client child and let
it manage state locally ; the parent server file only passes data.

### Verification

```bash
npm run build                # -> succeeds
rg -n '<[A-Z][A-Za-z]+\s+[^>]*on[A-Z][A-Za-z]+=\{[^}]*=>' app/products
# -> empty inside server files
```

---

## AP-5 : `ThemeProvider` Imported Directly from `next-themes` in `app/layout.tsx`

### Symptom

```tsx
// app/layout.tsx (server)
import { ThemeProvider } from "next-themes"                      // <-- WRONG

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html><body>
      <ThemeProvider attribute="class" defaultTheme="system">{children}</ThemeProvider>
    </body></html>
  )
}
```

Hydration crash :
`Error: useTheme is not a function` or
`Cannot read properties of null (reading 'useContext')`.

The `next-themes` package internally calls `useContext` / `useState`
inside its Provider component. Imported into a server file, the
Provider tries to invoke hooks that have no server runtime.

The same trap applies to `@tanstack/react-query` `QueryClientProvider`,
`jotai` `Provider`, `zustand` `Provider`, `sonner` `Toaster`, `nuqs`
`NuqsAdapter`, `react-aria` `OverlayProvider`, and any other library
whose Provider relies on `useContext` / `useState`.

### Why it fails

A server component MAY render a client component (the parent stays
server, the child carries the directive). The inverse is forbidden : a
server file cannot CONTAIN a hook-using component. Importing the
Provider directly puts the hook-using component INSIDE the server
file, which then tries to call hooks at render time on the server.

### Validator diagnosis

Rule 5 fires : a known Provider library is imported directly into a
file that does not start with `"use client"`. Verdict : **FAIL**.

### Fix

Wrap the Provider in a thin one-line client wrapper, then import the
wrapper :

```tsx
// components/theme-provider.tsx
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
// app/layout.tsx (server, fixed)
import { ThemeProvider } from "@/components/theme-provider"      // wrapper, not the lib

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

ALWAYS keep the wrapper a one-line passthrough. NEVER mix unrelated
client logic into a Provider wrapper ; it becomes a magnet for hooks
that don't belong and breaks the file's single responsibility.

### Verification

```bash
rg -n 'from "next-themes"' app                # -> empty
rg -n 'from "next-themes"' components         # -> components/theme-provider.tsx
head -1 components/theme-provider.tsx          # -> "use client"
```

---

## AP-6 : Non-Serialisable Prop Across the Boundary (Date / Decimal / Class Instance / Map)

### Symptom

```tsx
// app/orders/page.tsx (server)
import { OrderRow } from "@/components/order-row"               // "use client"
import { db } from "@/lib/db"

export default async function OrdersPage() {
  const orders = await db.order.findMany()                       // each row has a Decimal `total`
  return (
    <ul>
      {orders.map((o) => (
        <OrderRow key={o.id} order={o} />                        // <-- WRONG : Decimal prop
      ))}
    </ul>
  )
}
```

No build error. At runtime the client component sees `order.total` as
either a string (the ORM coerced it) or `[object Object]` (the
serialiser stringified it). Hydration mismatch warning in DevTools.
Subsequent comparisons in the client component
(`order.total.gt(100)`) throw `total.gt is not a function`.

Same trap for `Date` with methods that depend on prototype (custom
`Day.js` instances, `Temporal` objects), `Map<string, classInstance>`,
`Set<classInstance>`, and class instances generally.

### Why it fails

The server-to-client serialiser converts every prop with the React
flight serialiser. Plain values (string, number, boolean, plain
object, plain array, `Date`, `Map`, `Set`, `null`, `undefined`) are
round-tripped correctly. Class instances are NOT : the serialiser
strips the prototype, sometimes preserves the field bag, sometimes
calls `toJSON()` if present, and the client receives a plain object
shape that no longer has the original methods.

`Decimal` from `decimal.js` or `prisma`'s `Prisma.Decimal` is a class
instance. Same for `Day.js` and `moment` objects. Same for any
ORM-returned model that has methods.

### Validator diagnosis

Rule 6 fires (heuristic) : the validator surfaces `<ClientComp
order={...}>` when the value comes from a known instance-returning
source (`db.*.findMany`, `Prisma.Decimal`, `new <Class>(...)`).
Verdict : **FAIL (candidate, confirm with the actual prop type)**.

### Fix

Serialise the data in the server file BEFORE passing it across the
boundary :

```tsx
// app/orders/page.tsx (server, fixed)
import { OrderRow } from "@/components/order-row"
import { db } from "@/lib/db"

export default async function OrdersPage() {
  const orders = await db.order.findMany()
  const serialised = orders.map((o) => ({
    id: o.id,
    total: o.total.toString(),         // Decimal -> string
    createdAt: o.createdAt.toISOString(),
    customerId: o.customerId,
  }))
  return (
    <ul>
      {serialised.map((o) => (<OrderRow key={o.id} order={o} />))}
    </ul>
  )
}
```

ALWAYS shape an explicit DTO at the server-to-client boundary. NEVER
pass an ORM row directly ; the prototype lift is invisible until
production.

### Verification

```bash
# Manual : the client component's prop type is a plain object, not the ORM type.
# Compile error (TypeScript) at the call-site after switching the prop type :
tsc --noEmit
```

---

## AP-7 (Bandaid Trap) : `suppressHydrationWarning` as a Workaround for Missing `"use client"`

### Symptom

```tsx
// app/layout.tsx (server, MISSING ThemeProvider wrapper but trying to silence the warning)
import { ThemeProvider } from "next-themes"

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body suppressHydrationWarning>                            // <-- DOES NOT FIX THE BUG
        <ThemeProvider attribute="class">{children}</ThemeProvider>
      </body>
    </html>
  )
}
```

The hydration WARNING about class-mismatch on `<html>` is silenced.
The actual crash (`useTheme is not a function`) is not. The developer
sees the warning go away in dev mode, ships, and the production build
breaks differently.

### Why it fails

`suppressHydrationWarning` is a LEGITIMATE fix for ONE problem : the
first-paint class-mismatch on `<html>` when `next-themes` reads the
user's saved theme from `localStorage` and toggles the class AFTER
React's initial hydration. It tells React "I know the server HTML
class will differ from the client class for this single element ;
don't warn".

It is NOT a fix for missing-directive errors, server-passing-function
errors, or any other RSC boundary violation. Treating it as a
silver-bullet hides bugs.

### Validator diagnosis

The validator does NOT fire on `suppressHydrationWarning` itself. It
fires on the underlying defect (rule 2, rule 5, rule 6). The bandaid
is documented here because the symptom (the warning goes away) makes
developers stop investigating. Verdict : the validator's FAIL on the
underlying rule is the source of truth.

### Fix

Keep `suppressHydrationWarning` on `<html>` for `next-themes` (per the
shadcn-impl-rsc-vs-client-boundaries §"Provider Wrapper" guidance) AND
fix the actual rule violation : extract the Provider into a client
wrapper.

```tsx
// app/layout.tsx (server, fixed)
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

### Verification

```bash
# The original validator failures (rules 2 / 5) must clear.
bash scripts/validate-rsc.sh .                # -> Verdict : PASS
# suppressHydrationWarning on <html> is kept for the legitimate class-mismatch.
rg -n 'suppressHydrationWarning' app/layout.tsx   # -> on the <html> element only
```

---

## Summary Lookup Table

| AP   | Rule | One-line cause                                            | One-line fix                                  |
|------|------|-----------------------------------------------------------|-----------------------------------------------|
| AP-1 | 3    | `app/layout.tsx` starts with `"use client"`.              | Extract the Provider into a client wrapper.   |
| AP-2 | 2    | Dialog / Sheet / Drawer / ... file missing the directive. | `shadcn add <name> --overwrite`.              |
| AP-3 | 4    | Conditional / backticked / indented directive.            | Make it the first line, double-quoted.        |
| AP-4 | 6    | Server file passes inline `onClick` to a client child.    | Use a `"use server"` Server Action.           |
| AP-5 | 5    | Provider imported directly in `app/layout.tsx`.           | Wrap in a one-file client wrapper.            |
| AP-6 | 6    | Class instance / Decimal / Date with methods crosses.     | Serialise to plain DTO before the boundary.   |
| AP-7 | (n/a)| `suppressHydrationWarning` used to silence real bugs.     | Fix the real bug ; keep the attr for class-mismatch only. |

ALWAYS treat the validator verdict as the source of truth. NEVER paper
over a FAIL with a hydration-warning-suppress, a `@ts-ignore`, or an
ESLint disable comment ; the underlying defect ships to production.
