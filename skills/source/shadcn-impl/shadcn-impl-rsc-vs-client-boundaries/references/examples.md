# Examples : RSC vs Client Boundaries

Five worked examples covering the most common composition patterns in
a Next.js App Router project that uses shadcn primitives.

---

## Example 1 : Server Page Rendering Button + Card (RSC-safe)

A pure list page. No interactivity. The whole page stays a server
component ; zero JavaScript ships for these primitives.

```tsx
// app/products/page.tsx                                 (SERVER)
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card"
import { Button } from "@/components/ui/button"
import { Badge } from "@/components/ui/badge"
import Link from "next/link"
import { db } from "@/lib/db"

export default async function ProductsPage() {
  const products = await db.product.findMany({ orderBy: { createdAt: "desc" } })

  return (
    <main className="container mx-auto p-6">
      <h1 className="text-2xl font-semibold mb-6">Products</h1>
      <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
        {products.map((p) => (
          <Card key={p.id}>
            <CardHeader>
              <CardTitle>{p.name}</CardTitle>
              <Badge variant={p.inStock ? "default" : "secondary"}>
                {p.inStock ? "In stock" : "Sold out"}
              </Badge>
            </CardHeader>
            <CardContent>
              <p className="text-sm text-muted-foreground">{p.summary}</p>
              <Button asChild className="mt-4">
                <Link href={`/products/${p.id}`}>View</Link>
              </Button>
            </CardContent>
          </Card>
        ))}
      </div>
    </main>
  )
}
```

Notes :
- No `"use client"` anywhere. The page is a server component (async + fetch).
- `Button` is RSC-safe ; the navigation comes from `<Link>`, not an
  `onClick` handler.
- `Card`, `CardHeader`, `CardTitle`, `CardContent`, `Badge` are all
  RSC-safe per the matrix.

---

## Example 2 : Server-Shell + Client-Island (ProductActions)

The page stays server. Interactive controls live in a child file
that carries the directive.

```tsx
// app/products/[id]/page.tsx                            (SERVER)
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card"
import { ProductActions } from "./product-actions"
import { db } from "@/lib/db"

export default async function ProductPage({ params }: { params: { id: string } }) {
  const product = await db.product.findUnique({ where: { id: params.id } })
  if (!product) return <div>Not found</div>

  return (
    <Card>
      <CardHeader>
        <CardTitle>{product.name}</CardTitle>
      </CardHeader>
      <CardContent className="space-y-4">
        <p>{product.description}</p>
        <ProductActions
          productId={product.id}
          initialQuantity={product.stock}
        />
      </CardContent>
    </Card>
  )
}
```

```tsx
// app/products/[id]/product-actions.tsx                 ("use client")
"use client"

import * as React from "react"
import { Button } from "@/components/ui/button"
import {
  Dialog, DialogContent, DialogDescription, DialogFooter,
  DialogHeader, DialogTitle, DialogTrigger,
} from "@/components/ui/dialog"
import { toast } from "sonner"

interface ProductActionsProps {
  productId: string
  initialQuantity: number
}

export function ProductActions({ productId, initialQuantity }: ProductActionsProps) {
  const [quantity, setQuantity] = React.useState(initialQuantity)
  const [open, setOpen] = React.useState(false)

  async function handleDelete() {
    await fetch(`/api/products/${productId}`, { method: "DELETE" })
    toast.success("Deleted")
    setOpen(false)
  }

  return (
    <div className="flex gap-2">
      <Button onClick={() => setQuantity((q) => q + 1)}>Add one</Button>
      <Dialog open={open} onOpenChange={setOpen}>
        <DialogTrigger asChild>
          <Button variant="destructive">Delete</Button>
        </DialogTrigger>
        <DialogContent>
          <DialogHeader>
            <DialogTitle>Delete product ?</DialogTitle>
            <DialogDescription>This cannot be undone.</DialogDescription>
          </DialogHeader>
          <DialogFooter>
            <Button variant="outline" onClick={() => setOpen(false)}>Cancel</Button>
            <Button variant="destructive" onClick={handleDelete}>Delete</Button>
          </DialogFooter>
        </DialogContent>
      </Dialog>
    </div>
  )
}
```

Notes :
- `page.tsx` reads from the database in async land and renders a Card
  with no interactivity.
- `product-actions.tsx` is the client island : Dialog (Radix), Sonner
  toast (client mount), `useState` (hooks). One file carries the
  directive ; the page does not.
- The page passes only serialisable props (`productId: string`,
  `initialQuantity: number`).

---

## Example 3 : ThemeProvider Wrapper (`"use client"`)

next-themes ships its own Provider, but the Provider uses `useTheme()`
context internally. It must live in a `"use client"` file.

```tsx
// components/theme-provider.tsx                         ("use client")
"use client"

import * as React from "react"
import { ThemeProvider as NextThemesProvider } from "next-themes"

export function ThemeProvider({
  children,
  ...props
}: React.ComponentProps<typeof NextThemesProvider>) {
  return <NextThemesProvider {...props}>{children}</NextThemesProvider>
}
```

```tsx
// app/layout.tsx                                         (SERVER)
import type { Metadata } from "next"
import { Inter } from "next/font/google"
import { ThemeProvider } from "@/components/theme-provider"
import "./globals.css"

const inter = Inter({ subsets: ["latin"] })

export const metadata: Metadata = {
  title: "App",
  description: "shadcn ui evergreen-2026",
}

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body className={inter.className}>
        <ThemeProvider
          attribute="class"
          defaultTheme="system"
          enableSystem
          disableTransitionOnChange
        >
          {children}
        </ThemeProvider>
      </body>
    </html>
  )
}
```

Notes :
- `layout.tsx` stays a server component. It RENDERS `ThemeProvider`
  but the `"use client"` directive lives inside `theme-provider.tsx`.
- `suppressHydrationWarning` on `<html>` is REQUIRED because
  next-themes mutates the `class` attribute on `<html>` on first paint,
  which produces a legitimate (and ignored) hydration mismatch.

---

## Example 4 : TanStack Query Provider Wrapper

`QueryClientProvider` provides React context backed by a
`QueryClient` instance. The instance must be created on the client
(per-request fresh instances) and the Provider needs the directive.

```tsx
// components/providers/query-provider.tsx               ("use client")
"use client"

import * as React from "react"
import { QueryClient, QueryClientProvider } from "@tanstack/react-query"
import { ReactQueryDevtools } from "@tanstack/react-query-devtools"

export function QueryProvider({ children }: { children: React.ReactNode }) {
  const [client] = React.useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            staleTime: 60 * 1000,
            refetchOnWindowFocus: false,
          },
        },
      })
  )

  return (
    <QueryClientProvider client={client}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  )
}
```

```tsx
// app/layout.tsx                                         (SERVER)
import { ThemeProvider } from "@/components/theme-provider"
import { QueryProvider } from "@/components/providers/query-provider"

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body>
        <QueryProvider>
          <ThemeProvider attribute="class" defaultTheme="system" enableSystem>
            {children}
          </ThemeProvider>
        </QueryProvider>
      </body>
    </html>
  )
}
```

Notes :
- The `QueryClient` is created inside a `useState` initialiser so each
  request gets its own instance (avoids cross-request data leaks).
- The layout composes two client Providers ; the layout itself stays
  a server component.

---

## Example 5 : Form Inside a Server Page

`Form` is always a client component (react-hook-form `useForm`). The
hosting page may still be a server component ; only the Form file
carries the directive.

```tsx
// app/contact/page.tsx                                   (SERVER)
import { Card, CardHeader, CardTitle, CardContent } from "@/components/ui/card"
import { ContactForm } from "./contact-form"

export default function ContactPage() {
  return (
    <main className="container mx-auto p-6 max-w-md">
      <Card>
        <CardHeader>
          <CardTitle>Contact us</CardTitle>
        </CardHeader>
        <CardContent>
          <ContactForm />
        </CardContent>
      </Card>
    </main>
  )
}
```

```tsx
// app/contact/contact-form.tsx                           ("use client")
"use client"

import * as React from "react"
import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import * as z from "zod"
import { Button } from "@/components/ui/button"
import {
  Form, FormControl, FormDescription, FormField,
  FormItem, FormLabel, FormMessage,
} from "@/components/ui/form"
import { Input } from "@/components/ui/input"
import { Textarea } from "@/components/ui/textarea"
import { toast } from "sonner"
import { submitContact } from "./actions"        // server action ("use server")

const schema = z.object({
  name: z.string().min(1, "Name is required"),
  email: z.string().email("Invalid email"),
  message: z.string().min(10, "At least 10 characters"),
})

export function ContactForm() {
  const form = useForm<z.infer<typeof schema>>({
    resolver: zodResolver(schema),
    defaultValues: { name: "", email: "", message: "" },
  })

  async function onSubmit(values: z.infer<typeof schema>) {
    const result = await submitContact(values)     // <- crosses to server
    if (result.ok) {
      toast.success("Sent")
      form.reset()
    } else {
      toast.error(result.error)
    }
  }

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
        <FormField
          control={form.control}
          name="name"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Name</FormLabel>
              <FormControl><Input {...field} /></FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        <FormField
          control={form.control}
          name="email"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Email</FormLabel>
              <FormControl><Input type="email" {...field} /></FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        <FormField
          control={form.control}
          name="message"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Message</FormLabel>
              <FormControl><Textarea rows={5} {...field} /></FormControl>
              <FormDescription>Tell us what you need.</FormDescription>
              <FormMessage />
            </FormItem>
          )}
        />
        <Button type="submit" disabled={form.formState.isSubmitting}>
          {form.formState.isSubmitting ? "Sending ..." : "Send"}
        </Button>
      </form>
    </Form>
  )
}
```

```tsx
// app/contact/actions.ts                                 ("use server")
"use server"

import { z } from "zod"

const schema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
  message: z.string().min(10),
})

export async function submitContact(values: z.infer<typeof schema>) {
  const parsed = schema.safeParse(values)
  if (!parsed.success) return { ok: false as const, error: "Invalid input" }

  // ... send email / write to db ...
  return { ok: true as const }
}
```

Notes :
- `page.tsx` is a server component. It hosts the form-card layout.
- `contact-form.tsx` is a client component. It hosts react-hook-form
  state and event handlers.
- `actions.ts` is a server module. It is imported by the client form
  but the function executes on the server ; the framework wires up the
  RPC.
- The boundary in this example is bidirectional : client calls server,
  server returns a serialisable result, no class instances or
  functions cross the wire.
