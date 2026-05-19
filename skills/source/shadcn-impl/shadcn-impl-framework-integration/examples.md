# Examples : end-to-end framework setups

Five complete recipes. Each goes from `create` to a working
`<Button>` plus dark mode toggle.

## Example 1 : Next.js App Router (init + Button + dark mode)

```bash
pnpm create next-app@latest acme --typescript --tailwind --app
cd acme
pnpm dlx shadcn@latest init -t next
pnpm dlx shadcn@latest add button dropdown-menu
pnpm add next-themes
```

`components/theme-provider.tsx` :

```tsx
"use client"
import * as React from "react"
import { ThemeProvider as NextThemesProvider } from "next-themes"

export function ThemeProvider(props: React.ComponentProps<typeof NextThemesProvider>) {
  return <NextThemesProvider {...props} />
}
```

`components/mode-toggle.tsx` :

```tsx
"use client"
import * as React from "react"
import { Moon, Sun } from "lucide-react"
import { useTheme } from "next-themes"
import { Button } from "@/components/ui/button"
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuTrigger,
} from "@/components/ui/dropdown-menu"

export function ModeToggle() {
  const { setTheme } = useTheme()
  return (
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <Button variant="outline" size="icon">
          <Sun className="h-[1.2rem] w-[1.2rem] scale-100 rotate-0 transition-all dark:scale-0 dark:-rotate-90" />
          <Moon className="absolute h-[1.2rem] w-[1.2rem] scale-0 rotate-90 transition-all dark:scale-100 dark:rotate-0" />
          <span className="sr-only">Toggle theme</span>
        </Button>
      </DropdownMenuTrigger>
      <DropdownMenuContent align="end">
        <DropdownMenuItem onClick={() => setTheme("light")}>Light</DropdownMenuItem>
        <DropdownMenuItem onClick={() => setTheme("dark")}>Dark</DropdownMenuItem>
        <DropdownMenuItem onClick={() => setTheme("system")}>System</DropdownMenuItem>
      </DropdownMenuContent>
    </DropdownMenu>
  )
}
```

`app/layout.tsx` :

```tsx
import "./globals.css"
import { ThemeProvider } from "@/components/theme-provider"

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body>
        <ThemeProvider attribute="class" defaultTheme="system" enableSystem disableTransitionOnChange>
          {children}
        </ThemeProvider>
      </body>
    </html>
  )
}
```

`app/page.tsx` :

```tsx
import { Button } from "@/components/ui/button"
import { ModeToggle } from "@/components/mode-toggle"
export default function Home() {
  return (
    <main className="flex min-h-screen items-center justify-center gap-4">
      <Button>Click me</Button>
      <ModeToggle />
    </main>
  )
}
```

`suppressHydrationWarning` on `<html>` is REQUIRED. Without it,
next-themes mutates the class on the client before React hydration
and React logs a mismatch.

## Example 2 : Vite + React (init + Button + custom Context theme)

```bash
pnpm create vite@latest acme --template react-ts
cd acme
pnpm install
pnpm add tailwindcss @tailwindcss/vite
pnpm add -D @types/node
pnpm dlx shadcn@latest init -t vite
pnpm dlx shadcn@latest add button dropdown-menu
```

`vite.config.ts` (the CLI writes this; verify shape) :

```ts
import path from "path"
import tailwindcss from "@tailwindcss/vite"
import react from "@vitejs/plugin-react"
import { defineConfig } from "vite"

export default defineConfig({
  plugins: [react(), tailwindcss()],
  resolve: { alias: { "@": path.resolve(__dirname, "./src") } },
})
```

`src/index.css` :

```css
@import "tailwindcss";
```

`src/components/theme-provider.tsx` :

```tsx
import { createContext, useContext, useEffect, useState } from "react"
type Theme = "dark" | "light" | "system"
type ThemeProviderProps = { children: React.ReactNode; defaultTheme?: Theme; storageKey?: string }
type ThemeProviderState = { theme: Theme; setTheme: (theme: Theme) => void }

const initialState: ThemeProviderState = { theme: "system", setTheme: () => null }
const ThemeProviderContext = createContext<ThemeProviderState>(initialState)

export function ThemeProvider({
  children,
  defaultTheme = "system",
  storageKey = "vite-ui-theme",
  ...props
}: ThemeProviderProps) {
  const [theme, setTheme] = useState<Theme>(
    () => (localStorage.getItem(storageKey) as Theme) || defaultTheme
  )
  useEffect(() => {
    const root = window.document.documentElement
    root.classList.remove("light", "dark")
    if (theme === "system") {
      const systemTheme = window.matchMedia("(prefers-color-scheme: dark)").matches ? "dark" : "light"
      root.classList.add(systemTheme)
      return
    }
    root.classList.add(theme)
  }, [theme])

  const value = {
    theme,
    setTheme: (theme: Theme) => {
      localStorage.setItem(storageKey, theme)
      setTheme(theme)
    },
  }
  return <ThemeProviderContext.Provider {...props} value={value}>{children}</ThemeProviderContext.Provider>
}

export const useTheme = () => {
  const ctx = useContext(ThemeProviderContext)
  if (ctx === undefined) throw new Error("useTheme must be used within a ThemeProvider")
  return ctx
}
```

`src/App.tsx` :

```tsx
import { Button } from "@/components/ui/button"
import { ThemeProvider } from "@/components/theme-provider"

export default function App() {
  return (
    <ThemeProvider defaultTheme="system" storageKey="vite-ui-theme">
      <main className="flex min-h-screen items-center justify-center">
        <Button>Click me</Button>
      </main>
    </ThemeProvider>
  )
}
```

NEVER install `next-themes` in a Vite project. It depends on Next.js
runtime APIs that are absent and will throw at build time.

## Example 3 : Astro + React islands (init + Button + dark mode inline script)

```bash
pnpm create astro@latest acme -- --template with-tailwindcss --install --add react --git
cd acme
pnpm dlx shadcn@latest init -t astro
pnpm dlx shadcn@latest add button dropdown-menu
```

`astro.config.mjs` :

```js
import { defineConfig } from "astro/config"
import react from "@astrojs/react"
import tailwindcss from "@tailwindcss/vite"

export default defineConfig({
  integrations: [react()],
  vite: { plugins: [tailwindcss()] },
})
```

`src/layouts/Layout.astro` :

```astro
---
import "@/styles/globals.css"
---
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Acme</title>
    <script is:inline>
      const getThemePreference = () => {
        if (typeof localStorage !== "undefined" && localStorage.getItem("theme")) {
          return localStorage.getItem("theme")
        }
        return window.matchMedia("(prefers-color-scheme: dark)").matches ? "dark" : "light"
      }
      const isDark = getThemePreference() === "dark"
      document.documentElement.classList[isDark ? "add" : "remove"]("dark")
      if (typeof localStorage !== "undefined") {
        const observer = new MutationObserver(() => {
          const isDark = document.documentElement.classList.contains("dark")
          localStorage.setItem("theme", isDark ? "dark" : "light")
        })
        observer.observe(document.documentElement, { attributes: true, attributeFilter: ["class"] })
      }
    </script>
  </head>
  <body><slot /></body>
</html>
```

`src/components/ModeToggle.tsx` :

```tsx
import * as React from "react"
import { Moon, Sun } from "lucide-react"
import { Button } from "@/components/ui/button"

export default function ModeToggle() {
  const [theme, setTheme] = React.useState<"light" | "dark">("light")
  React.useEffect(() => {
    const isDark = document.documentElement.classList.contains("dark")
    setTheme(isDark ? "dark" : "light")
  }, [])
  React.useEffect(() => {
    const isDark = theme === "dark"
    document.documentElement.classList[isDark ? "add" : "remove"]("dark")
  }, [theme])
  return (
    <Button variant="outline" size="icon" onClick={() => setTheme(theme === "light" ? "dark" : "light")}>
      <Sun className="h-[1.2rem] w-[1.2rem] scale-100 rotate-0 transition-all dark:scale-0 dark:-rotate-90" />
      <Moon className="absolute h-[1.2rem] w-[1.2rem] scale-0 rotate-90 transition-all dark:scale-100 dark:rotate-0" />
      <span className="sr-only">Toggle theme</span>
    </Button>
  )
}
```

`src/pages/index.astro` :

```astro
---
import Layout from "@/layouts/Layout.astro"
import { Button } from "@/components/ui/button"
import ModeToggle from "@/components/ModeToggle"
---
<Layout>
  <main class="flex min-h-screen items-center justify-center gap-4">
    <Button>Static button (no hydration)</Button>
    <ModeToggle client:load />
  </main>
</Layout>
```

The `<Button>` here is rendered at build time as static HTML, since
nothing interactive uses it. The `<ModeToggle client:load />` hydrates
on page load. Use `client:idle` if hydration can wait, `client:visible`
for components below the fold.

## Example 4 : React Router v7 loader + shadcn Form

```bash
pnpm create react-router@latest acme
cd acme
pnpm dlx shadcn@latest init -t react-router
pnpm dlx shadcn@latest add button form input label
```

`app/routes/contact.tsx` :

```tsx
import type { Route } from "./+types/contact"
import { Form, useActionData } from "react-router"
import { Button } from "~/components/ui/button"
import { Input } from "~/components/ui/input"
import { Label } from "~/components/ui/label"

export async function loader({ params }: Route.LoaderArgs) {
  return { defaultEmail: "" }
}

export async function action({ request }: Route.ActionArgs) {
  const formData = await request.formData()
  const email = formData.get("email")
  if (!email || typeof email !== "string") {
    return { error: "Email required" }
  }
  // process server-side
  return { success: true, email }
}

export default function Contact({ loaderData }: Route.ComponentProps) {
  const actionData = useActionData<typeof action>()
  return (
    <Form method="post" className="space-y-4 p-8">
      <div>
        <Label htmlFor="email">Email</Label>
        <Input id="email" name="email" type="email" defaultValue={loaderData.defaultEmail} />
      </div>
      {actionData?.error && <p className="text-destructive">{actionData.error}</p>}
      {actionData?.success && <p className="text-green-600">Saved {actionData.email}</p>}
      <Button type="submit">Submit</Button>
    </Form>
  )
}
```

NOTE the `~/` import prefix and the use of React Router's native
`<Form>` from `react-router`, NOT shadcn's `<Form>` from
`~/components/ui/form`. shadcn `<Form>` is a react-hook-form wrapper;
use it when you need client-side validation. Combine the two:
react-router `<Form>` for the submit boundary, shadcn `<Form>` inside
for field-level UX. See `shadcn-impl-form-validation` for the
combined pattern.

## Example 5 : TanStack Start SSR + client hydration

```bash
pnpm dlx shadcn@latest init -t start
cd my-start-app  # or wherever the preset places it
pnpm dlx shadcn@latest add card button
```

`src/routes/__root.tsx` :

```tsx
import { Outlet, createRootRoute } from "@tanstack/react-router"
import { ThemeProvider } from "@/components/theme-provider"
import "@/styles/app.css"

export const Route = createRootRoute({
  component: () => (
    <ThemeProvider defaultTheme="system" storageKey="start-ui-theme">
      <Outlet />
    </ThemeProvider>
  ),
})
```

`src/routes/index.tsx` :

```tsx
import { createFileRoute } from "@tanstack/react-router"
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card"
import { Button } from "@/components/ui/button"

export const Route = createFileRoute("/")({ component: Home })

function Home() {
  return (
    <main className="flex min-h-screen items-center justify-center">
      <Card>
        <CardHeader><CardTitle>Hello TanStack Start</CardTitle></CardHeader>
        <CardContent><Button>Action</Button></CardContent>
      </Card>
    </main>
  )
}
```

TanStack Start renders on the server first, then hydrates on the client.
`<Card>` and `<Button>` work in both. If `components.json.rsc` is `true`,
the CLI omits `"use client"` from pure-presentation components and
includes it on interactive ones. Use the same custom ThemeProvider as
Vite; `next-themes` is not compatible.
