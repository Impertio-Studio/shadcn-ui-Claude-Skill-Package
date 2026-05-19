# References : Examples (theming-custom)

Working code examples for each workflow in the SKILL.md. Every snippet is version-explicit (v3 or v4, Next.js or Vite). Verified against https://ui.shadcn.com/docs/dark-mode/next, /vite, and /theming on 2026-05-19.

## Example 1 : Full Tailwind v4 `globals.css` with brand-primary override

Drop-in `app/globals.css` for a Next.js v4 project with a custom purple brand primary. Replaces the default shadcn `--primary` only ; every other token stays at the shadcn default.

```css
@import "tailwindcss";
@import "tw-animate-css";

@custom-variant dark (&:is(.dark *));

:root {
  --background: oklch(1 0 0);
  --foreground: oklch(0.145 0 0);
  --card: oklch(1 0 0);
  --card-foreground: oklch(0.145 0 0);
  --popover: oklch(1 0 0);
  --popover-foreground: oklch(0.145 0 0);

  /* Brand primary : purple, contrast-tuned with near-white foreground */
  --primary: oklch(0.488 0.243 264.376);
  --primary-foreground: oklch(0.985 0 0);

  --secondary: oklch(0.97 0 0);
  --secondary-foreground: oklch(0.205 0 0);
  --muted: oklch(0.97 0 0);
  --muted-foreground: oklch(0.556 0 0);
  --accent: oklch(0.97 0 0);
  --accent-foreground: oklch(0.205 0 0);
  --destructive: oklch(0.577 0.245 27.325);
  --border: oklch(0.922 0 0);
  --input: oklch(0.922 0 0);
  --ring: oklch(0.488 0.243 264.376);  /* matches primary */
  --radius: 0.625rem;

  /* Sidebar tokens : also tuned to brand-purple for consistency */
  --sidebar: oklch(0.985 0 0);
  --sidebar-foreground: oklch(0.145 0 0);
  --sidebar-primary: oklch(0.488 0.243 264.376);
  --sidebar-primary-foreground: oklch(0.985 0 0);
  --sidebar-accent: oklch(0.97 0 0);
  --sidebar-accent-foreground: oklch(0.205 0 0);
  --sidebar-border: oklch(0.922 0 0);
  --sidebar-ring: oklch(0.488 0.243 264.376);

  --chart-1: oklch(0.488 0.243 264.376);
  --chart-2: oklch(0.696 0.170 162.480);
  --chart-3: oklch(0.769 0.188 70.080);
  --chart-4: oklch(0.627 0.265 303.900);
  --chart-5: oklch(0.645 0.246 16.439);
}

.dark {
  --background: oklch(0.145 0 0);
  --foreground: oklch(0.985 0 0);
  --card: oklch(0.205 0 0);
  --card-foreground: oklch(0.985 0 0);
  --popover: oklch(0.205 0 0);
  --popover-foreground: oklch(0.985 0 0);

  /* Brand primary in dark mode : slightly lighter purple, contrast-tuned with near-black foreground */
  --primary: oklch(0.692 0.180 270.000);
  --primary-foreground: oklch(0.205 0 0);

  --secondary: oklch(0.269 0 0);
  --secondary-foreground: oklch(0.985 0 0);
  --muted: oklch(0.269 0 0);
  --muted-foreground: oklch(0.708 0 0);
  --accent: oklch(0.269 0 0);
  --accent-foreground: oklch(0.985 0 0);
  --destructive: oklch(0.704 0.191 22.216);
  --border: oklch(0.269 0 0);
  --input: oklch(0.269 0 0);
  --ring: oklch(0.692 0.180 270.000);

  --sidebar: oklch(0.205 0 0);
  --sidebar-foreground: oklch(0.985 0 0);
  --sidebar-primary: oklch(0.692 0.180 270.000);
  --sidebar-primary-foreground: oklch(0.205 0 0);
  --sidebar-accent: oklch(0.269 0 0);
  --sidebar-accent-foreground: oklch(0.985 0 0);
  --sidebar-border: oklch(0.269 0 0);
  --sidebar-ring: oklch(0.692 0.180 270.000);

  --chart-1: oklch(0.488 0.243 264.376);
  --chart-2: oklch(0.696 0.170 162.480);
  --chart-3: oklch(0.769 0.188 70.080);
  --chart-4: oklch(0.627 0.265 303.900);
  --chart-5: oklch(0.645 0.246 16.439);
}

@theme inline {
  --color-background: var(--background);
  --color-foreground: var(--foreground);
  --color-card: var(--card);
  --color-card-foreground: var(--card-foreground);
  --color-popover: var(--popover);
  --color-popover-foreground: var(--popover-foreground);
  --color-primary: var(--primary);
  --color-primary-foreground: var(--primary-foreground);
  --color-secondary: var(--secondary);
  --color-secondary-foreground: var(--secondary-foreground);
  --color-muted: var(--muted);
  --color-muted-foreground: var(--muted-foreground);
  --color-accent: var(--accent);
  --color-accent-foreground: var(--accent-foreground);
  --color-destructive: var(--destructive);
  --color-border: var(--border);
  --color-input: var(--input);
  --color-ring: var(--ring);

  --color-sidebar: var(--sidebar);
  --color-sidebar-foreground: var(--sidebar-foreground);
  --color-sidebar-primary: var(--sidebar-primary);
  --color-sidebar-primary-foreground: var(--sidebar-primary-foreground);
  --color-sidebar-accent: var(--sidebar-accent);
  --color-sidebar-accent-foreground: var(--sidebar-accent-foreground);
  --color-sidebar-border: var(--sidebar-border);
  --color-sidebar-ring: var(--sidebar-ring);

  --color-chart-1: var(--chart-1);
  --color-chart-2: var(--chart-2);
  --color-chart-3: var(--chart-3);
  --color-chart-4: var(--chart-4);
  --color-chart-5: var(--chart-5);

  --radius-sm: calc(var(--radius) * 0.6);
  --radius-md: calc(var(--radius) * 0.8);
  --radius-lg: var(--radius);
  --radius-xl: calc(var(--radius) * 1.4);
  --radius-2xl: calc(var(--radius) * 1.8);
  --radius-3xl: calc(var(--radius) * 2.2);
  --radius-4xl: calc(var(--radius) * 2.6);
}

@layer base {
  * {
    @apply border-border outline-ring/50;
  }
  body {
    @apply bg-background text-foreground;
  }
}
```

Total token count : 19 core + 8 sidebar + 5 chart + 1 radius = 33 variables in `:root`, 33 again in `.dark`. The full reference for token defaults from the shadcn-distributed `default` style is in `shadcn-core-theming/references/methods.md`.

## Example 2 : Next.js `app/layout.tsx` with ThemeProvider

```tsx
import type { Metadata } from "next"
import { Inter } from "next/font/google"
import { ThemeProvider } from "@/components/theme-provider"
import "./globals.css"

const inter = Inter({ subsets: ["latin"] })

export const metadata: Metadata = {
  title: "Brand App",
  description: "Brand description",
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

Companion `components/theme-provider.tsx` :

```tsx
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

ALWAYS mount EXACTLY ONE `ThemeProvider`. Never nest a custom context provider on top of next-themes (or vice versa) ; the toggle becomes unpredictable.

## Example 3 : Vite `App.tsx` with custom ThemeContext

`src/components/theme-provider.tsx` (verbatim from https://ui.shadcn.com/docs/dark-mode/vite, verified 2026-05-19) :

```tsx
import { createContext, useContext, useEffect, useState } from "react"

type Theme = "dark" | "light" | "system"

type ThemeProviderProps = {
  children: React.ReactNode
  defaultTheme?: Theme
  storageKey?: string
}

type ThemeProviderState = {
  theme: Theme
  setTheme: (theme: Theme) => void
}

const initialState: ThemeProviderState = {
  theme: "system",
  setTheme: () => null,
}

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
      const systemTheme = window.matchMedia("(prefers-color-scheme: dark)")
        .matches
        ? "dark"
        : "light"
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

  return (
    <ThemeProviderContext.Provider {...props} value={value}>
      {children}
    </ThemeProviderContext.Provider>
  )
}

export const useTheme = () => {
  const context = useContext(ThemeProviderContext)

  if (context === undefined)
    throw new Error("useTheme must be used within a ThemeProvider")

  return context
}
```

`src/main.tsx` :

```tsx
import React from "react"
import ReactDOM from "react-dom/client"
import App from "./App"
import { ThemeProvider } from "@/components/theme-provider"
import "./index.css"

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode>
    <ThemeProvider defaultTheme="system" storageKey="brand-ui-theme">
      <App />
    </ThemeProvider>
  </React.StrictMode>
)
```

ALWAYS pick a stable `storageKey` per app (`"brand-ui-theme"` here). NEVER share storage keys across apps on the same domain.

## Example 4 : ModeToggle component (light / dark / system)

Works in BOTH Next.js and Vite, with one import-line difference for `useTheme` :

```tsx
"use client"  // only needed in Next.js ; remove in Vite

import { Moon, Sun } from "lucide-react"
import { useTheme } from "next-themes"
// In Vite : import { useTheme } from "@/components/theme-provider"

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
        <DropdownMenuItem onClick={() => setTheme("light")}>
          Light
        </DropdownMenuItem>
        <DropdownMenuItem onClick={() => setTheme("dark")}>
          Dark
        </DropdownMenuItem>
        <DropdownMenuItem onClick={() => setTheme("system")}>
          System
        </DropdownMenuItem>
      </DropdownMenuContent>
    </DropdownMenu>
  )
}
```

Drop into any layout :

```tsx
<header className="flex justify-between items-center p-4">
  <h1>Brand App</h1>
  <ModeToggle />
</header>
```

ALWAYS include all three items (Light / Dark / System). The System item respects OS preference and is the documented shadcn behavior.

ALWAYS drive the icon cross-fade via `dark:` Tailwind variants, NEVER via `theme === "dark" ? <Moon/> : <Sun/>`. The conditional approach hydrates with the server-rendered choice then flips, producing a one-frame flicker.

## Example 5 : Per-component override via data-slot

Three patterns for overriding ONE specific component instance without touching the global tokens.

### Pattern A : Tailwind arbitrary-variant on a wrapper

```tsx
import { Card, CardHeader, CardTitle, CardDescription, CardContent } from "@/components/ui/card"

export function PricingSection() {
  return (
    <section className="[&_[data-slot=card]]:border-2 [&_[data-slot=card]]:border-primary [&_[data-slot=card-header]]:pb-2">
      <Card>
        <CardHeader>
          <CardTitle>Pro Plan</CardTitle>
          <CardDescription>For growing teams</CardDescription>
        </CardHeader>
        <CardContent>$29 / month</CardContent>
      </Card>
    </section>
  )
}
```

Every Card inside the `<section>` gets a 2px primary-colored border ; every CardHeader gets reduced padding-bottom. Cards OUTSIDE the section are unaffected.

### Pattern B : Plain CSS selector scoped under a class

In `globals.css` (anywhere after the `@layer base` block) :

```css
.pricing-section [data-slot=card] {
  border-width: 2px;
  border-color: var(--primary);
}

.pricing-section [data-slot=card-header] {
  padding-bottom: 0.5rem;
}
```

Apply via `<section className="pricing-section">`. The CSS rules are global but only fire under the wrapper.

### Pattern C : Per-instance className using cn()

```tsx
import { cn } from "@/lib/utils"

export function HighlightCard({ className, ...props }) {
  return (
    <Card className={cn("border-2 border-primary", className)} {...props} />
  )
}
```

This pattern is fine for ONE special variant ; for a project-wide tweak to every Card under a section, Pattern A or B is cleaner and avoids per-instance work.

ALWAYS prefer `data-slot` selectors over component-internal className overrides when the goal is a stable, version-resilient project-wide tweak.

## Example 6 : Adding a brand-warning token (full path)

The four-step path to introduce a brand-new `--warning` token. v4 path :

```css
/* globals.css : 1. Declare in :root and .dark */
:root {
  /* ... existing tokens ... */
  --warning: oklch(0.795 0.184 86.047);          /* amber-ish */
  --warning-foreground: oklch(0.205 0 0);
}
.dark {
  /* ... existing tokens ... */
  --warning: oklch(0.795 0.184 86.047);
  --warning-foreground: oklch(0.205 0 0);
}

/* 2. Map in @theme inline */
@theme inline {
  /* ... existing mappings ... */
  --color-warning: var(--warning);
  --color-warning-foreground: var(--warning-foreground);
}
```

```tsx
// 3. Use the new utility
<div className="bg-warning text-warning-foreground p-4 rounded-md">
  Heads up : your trial expires soon.
</div>
```

```tsx
// 4. (optional) Create a reusable component
export function WarningBanner({ children }: { children: React.ReactNode }) {
  return (
    <div className="bg-warning text-warning-foreground p-4 rounded-md">
      {children}
    </div>
  )
}
```

ALWAYS keep the variable declaration AND the `@theme inline` mapping in lockstep. A variable without a mapping produces no Tailwind utility ; a mapping without a variable produces `var(--undefined)` which resolves to nothing.
