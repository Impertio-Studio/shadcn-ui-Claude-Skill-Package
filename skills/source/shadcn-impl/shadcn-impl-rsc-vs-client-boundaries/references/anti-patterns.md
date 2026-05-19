# Anti-Patterns : RSC vs Client Boundaries

Six high-frequency anti-patterns observed across the shadcn-ui issue
tracker, the Next.js App Router migration discussions, and the
shadcn-ui Discord. Each entry follows the same shape : symptom,
diagnosis, fix, verification.

---

## AP-1 : `"use client"` at the Top of `app/layout.tsx`

### Symptom

```tsx
// app/layout.tsx                                         ("use client")  <- WRONG
"use client"

import { ThemeProvider } from "next-themes"

export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>
        <ThemeProvider attribute="class" defaultTheme="system">
          {children}
        </ThemeProvider>
      </body>
    </html>
  )
}
```

The whole app boots as a client tree. Bundle size balloons. Async
server components inside the tree silently lose their server-side
fetches. The Next.js build emits a warning about losing static
optimisation.

### Diagnosis

The author wanted to use a Provider that needs the directive, so they
applied the directive to `layout.tsx` itself. This propagates to every
descendant module : every `page.tsx`, every layout, every component
becomes a client module.

### Fix

Move the directive to a wrapper file. The layout stays a server
component and only the Provider is client.

```tsx
// components/theme-provider.tsx                         ("use client")
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
// app/layout.tsx                                         (SERVER)
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
head -1 app/layout.tsx                       # -> import ... (NOT "use client")
head -1 components/theme-provider.tsx        # -> "use client"
```

---

## AP-2 : Missing `"use client"` on a File Using Dialog

### Symptom

Build succeeds. At runtime the page crashes with :

```
Error : useState is not a function
   at Dialog (.../dialog.tsx:42)
```

Or in the browser console :

```
Uncaught TypeError : Cannot read properties of null (reading 'useContext')
```

### Diagnosis

The file imports `Dialog` (which wraps Radix Dialog), but the file
itself lacks `"use client"`. React renders the Dialog on the server,
the Radix code calls `React.useState` (which is a no-op on the server
during the RSC phase), and the call throws.

This often happens when a developer copies a snippet from the shadcn
docs into a page-component-then-extract-later workflow but forgets the
extract step.

### Fix

Add `"use client"` to the top of the file containing the Dialog, OR
extract the Dialog into its own client file and import it from the
server page.

```tsx
// app/some-page/page.tsx                                 (SERVER)
import { DeleteButton } from "./delete-button"
export default function Page() {
  return <DeleteButton id="42" />
}
```

```tsx
// app/some-page/delete-button.tsx                        ("use client")
"use client"
import { Dialog, DialogTrigger, DialogContent } from "@/components/ui/dialog"
import { Button } from "@/components/ui/button"

export function DeleteButton({ id }: { id: string }) {
  return (
    <Dialog>
      <DialogTrigger asChild><Button>Delete</Button></DialogTrigger>
      <DialogContent>...</DialogContent>
    </Dialog>
  )
}
```

### Verification

```bash
grep -L '^"use client"' components/ui/dialog.tsx       # -> empty (directive present)
# In any consumer file that imports Dialog :
head -1 path/to/consumer.tsx                            # -> "use client"
```

If `components/ui/dialog.tsx` is missing the directive, regenerate :

```bash
pnpm dlx shadcn@latest add dialog --overwrite
```

---

## AP-3 : ThemeProvider Imported Directly Into a Server Layout

### Symptom

```tsx
// app/layout.tsx                                         (SERVER)
import { ThemeProvider } from "next-themes"             // <- WRONG

export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>
        <ThemeProvider attribute="class">{children}</ThemeProvider>
      </body>
    </html>
  )
}
```

Server build fails or warns :

```
Error : Functions cannot be passed directly to Client Components unless you explicitly expose it by marking it with "use server".
  ... at ThemeProvider
```

Or the theme silently does not apply : `useTheme()` returns
`undefined` everywhere, the class on `<html>` never changes, and
toggling the mode does nothing.

### Diagnosis

`next-themes`' `ThemeProvider` uses React context. Server components
cannot host context Providers. The Provider must live in a file that
carries `"use client"`.

### Fix

Wrap `next-themes`' `ThemeProvider` in a one-line client component (see
AP-1 fix). NEVER import a Provider from a non-shadcn-shipped library
directly into a server component.

### Verification

```bash
# layout.tsx imports the WRAPPER, not the library
grep -n 'from "next-themes"' app/layout.tsx              # -> empty
grep -n 'from "@/components/theme-provider"' app/layout.tsx
```

---

## AP-4 : Server Component Passing `onClick` to a Client Child

### Symptom

```tsx
// app/page.tsx                                           (SERVER)
import { ClientButton } from "./client-button"

export default function Page() {
  function handleClick() { console.log("clicked") }
  return <ClientButton onClick={handleClick} />          // <- WRONG
}
```

Compile error :

```
Error : Functions cannot be passed directly to Client Components
unless you explicitly expose it by marking it with "use server".
```

### Diagnosis

A function is not a serialisable value. The RSC payload between server
and client is plain JSON ; functions cannot survive the wire. The only
exceptions are Server Actions (functions marked `"use server"`).

### Fix : option A : keep the handler client-side

```tsx
// app/page.tsx                                           (SERVER)
import { ClientButton } from "./client-button"
export default function Page() {
  return <ClientButton message="clicked" />              // serialisable prop
}
```

```tsx
// app/client-button.tsx                                  ("use client")
"use client"
export function ClientButton({ message }: { message: string }) {
  return <button onClick={() => console.log(message)}>Click</button>
}
```

### Fix : option B : expose the function as a Server Action

```tsx
// app/actions.ts                                         ("use server")
"use server"
export async function logClick(message: string) {
  console.log("server-side :", message)
}
```

```tsx
// app/page.tsx                                           (SERVER)
import { logClick } from "./actions"
import { ClientButton } from "./client-button"
export default function Page() {
  return <ClientButton action={logClick} />              // server action survives the wire
}
```

```tsx
// app/client-button.tsx                                  ("use client")
"use client"
export function ClientButton({ action }: { action: (m: string) => void }) {
  return <button onClick={() => action("clicked")}>Click</button>
}
```

### Verification

The compile error disappears. The Next.js build emits a `[server]` log
when the action runs (option B) or a browser-side console log
(option A).

---

## AP-5 : Client Component Receives a Non-Serialisable Prop

### Symptom

```tsx
// app/page.tsx                                           (SERVER)
import { db } from "@/lib/db"
import { ProductCard } from "./product-card"

export default async function Page() {
  const products = await db.product.findMany()           // each `product` is a Prisma model instance
  return products.map((p) => <ProductCard product={p} />)
}
```

The cards render correctly on first paint, but client-side state on
the card (a `useState` that does `setState(p)`) starts producing
React warnings :

```
Warning : Cannot update an immutable record.
Warning : Each child in a list should have a unique "key" prop. (Sometimes triggered by prototype loss on the Prisma model.)
```

Or, after `JSON.stringify(product)` in a debug log, fields silently
disappear (Decimal, BigInt, Date in some setups).

### Diagnosis

Prisma model instances are not plain objects. Decimal and BigInt are
not serialisable. Date IS serialisable in App Router but only as an
ISO string after the wire ; methods like `.toISOString()` survive but
class instance identity does not.

### Fix

Serialise EXPLICITLY before passing across the boundary.

```tsx
// app/page.tsx                                           (SERVER)
import { db } from "@/lib/db"
import { ProductCard } from "./product-card"

export default async function Page() {
  const products = await db.product.findMany()
  const serialised = products.map((p) => ({
    id: p.id,
    name: p.name,
    price: p.price.toString(),                           // Decimal -> string
    createdAt: p.createdAt.toISOString(),                // Date    -> ISO string
  }))
  return serialised.map((p) => <ProductCard product={p} />)
}
```

```tsx
// app/product-card.tsx                                   ("use client")
"use client"
interface SerialisedProduct {
  id: string; name: string; price: string; createdAt: string
}
export function ProductCard({ product }: { product: SerialisedProduct }) {
  // ...
}
```

### Verification

```bash
# Warning disappears. The client component now has a typed,
# serialisable surface area :
grep -n "SerialisedProduct" app/product-card.tsx        # -> the interface lives here
```

---

## AP-6 : Conditional / Template-Literal Directive

### Symptom

```tsx
// some-component.tsx
const directive = "use client"                           // <- WRONG : assigned to a variable
// or
`use client`                                             // <- WRONG : template literal
// or
if (process.env.NODE_ENV !== "production") "use client"  // <- WRONG : statement, not a directive
```

The file imports a Radix-wrapped primitive. At runtime the same
hydration error from AP-2 appears (`useState is not a function`).

### Diagnosis

The `"use client"` directive is a SYNTACTIC marker, not a runtime
expression. The bundler scans for the literal first-non-comment
expression statement `"use client"` (or `'use client'`). Any of these
forms are ignored :

- Backtick template literal : `` `use client` ``
- String assigned to a variable : `const x = "use client"`
- Conditional : `if (...) "use client"`
- Inside a function body : `function foo() { "use client" }`
- After any import statement : `import x from "..."` then `"use client"` is too late.

### Fix

Use the exact form, at the exact top of the file, with no
non-comment content above it :

```tsx
"use client"
import * as React from "react"
// ...
```

Comments above the directive are allowed :

```tsx
// SPDX-License-Identifier : MIT
"use client"
import * as React from "react"
```

### Verification

```bash
# The directive is the first non-comment line, double-quoted, no semicolon needed.
head -3 path/to/file.tsx
# Acceptable :
#   "use client"
#   import * as React from "react"
#   ...

# Or with leading comment :
#   // ...
#   "use client"
#   import * as React from "react"

# Sanity-check the bundler treated it as a client module :
grep -n "^\"use client\"$" path/to/file.tsx              # -> 1 (or first non-comment line)
```

---

## Summary Table

| AP   | Symptom                                                     | Root cause                                                          | One-line fix                                          |
|------|-------------------------------------------------------------|----------------------------------------------------------------------|--------------------------------------------------------|
| AP-1 | Whole app becomes a client tree, large bundle, lost RSC     | `"use client"` placed on `app/layout.tsx`                            | Move directive into a Provider wrapper file.            |
| AP-2 | `useState is not a function` at render                      | File using Dialog (or any Radix wrap) lacks the directive            | Add `"use client"` to the file OR extract Dialog into a client island. |
| AP-3 | Theme toggle does nothing, hydration error in console       | next-themes `ThemeProvider` imported into a server file              | Wrap `ThemeProvider` in a `components/theme-provider.tsx` (`"use client"`). |
| AP-4 | `Functions cannot be passed directly to Client Components`  | Server passing `onClick` into a client child                          | Pass serialisable data and host the handler client-side, OR mark the handler `"use server"`. |
| AP-5 | Silent prop loss (Decimal, BigInt, class instances)         | Class instances passed across the RSC wire                           | Serialise explicitly to plain objects before passing.   |
| AP-6 | Hydration error despite a `"use client"` line in the file   | Directive is a template literal, conditional, or below an import     | Use the literal `"use client"` as the first non-comment line, double-quoted. |
