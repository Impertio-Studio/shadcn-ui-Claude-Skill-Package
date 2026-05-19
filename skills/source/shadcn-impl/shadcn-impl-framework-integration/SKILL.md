---
name: shadcn-impl-framework-integration
description: >
  Use when initializing shadcn ui inside a Vite, Next.js (App or Pages Router),
  React Router v7 (former Remix), Astro, or TanStack Start project, or when
  diagnosing per-framework init issues like wrong rsc flag, broken `@/*` alias
  resolution, or missing ThemeProvider wiring. Prevents the recurring failure
  mode where the CLI is run with the wrong `-t` template and produces a
  `components.json` whose `rsc` value mismatches the framework's RSC capability,
  silently dropping `"use client"` directives in Vite or duplicating them in
  Next.js. Covers per-framework init commands, the four alias-resolution
  systems (tsconfig paths for Next/Vite/React-Router, package.json#imports
  for TanStack Start, astro.config + tsconfig for Astro), components.json
  defaults per framework, Tailwind v3 vs v4 wiring per framework, and the
  three distinct ThemeProvider patterns (next-themes for Next.js, custom
  React Context for Vite/React-Router/TanStack Start, inline script for
  Astro).
  Keywords: shadcn vite, shadcn nextjs, shadcn next.js app router,
  shadcn next.js pages router, shadcn remix, shadcn react router v7,
  shadcn astro, shadcn tanstack start, framework integration, alias setup vite,
  components.json next.js, components.json vite, rsc true, rsc false,
  init template flag, -t next, -t vite, -t astro, -t start, -t react-router,
  ThemeProvider next-themes, ThemeProvider vite, dark mode astro inline script,
  suppressHydrationWarning, app router vs pages router, init for next.js,
  how do I set up shadcn in vite, my @ alias does not resolve, use client
  missing, hydration mismatch dark mode, tailwind v4 @tailwindcss/vite
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn-impl-framework-integration

Per-framework setup matrix for the shadcn ui CLI. Five supported templates
(`next`, `vite`, `react-router`, `astro`, `start`) differ in: init flag,
`components.json.rsc` default, alias-resolution system, Tailwind plugin,
and dark-mode strategy. Each axis is non-substitutable. Pick by framework
first, then apply the exact recipe.

## Quick Reference Decision Tree

```
Q1. Which framework is the host project?
    Next.js (App or Pages Router)   --> Section: Next.js
    Vite + React                    --> Section: Vite
    React Router v7 (former Remix)  --> Section: React Router
    Astro + React integration       --> Section: Astro
    TanStack Start                  --> Section: TanStack Start
    Laravel + Inertia               --> Run `laravel new` first, then
                                        `npx shadcn@latest init` (no `-t`)
    other (CRA, Parcel, Rspack)     --> NOT supported by CLI; use Vite recipe
                                        and adapt alias config manually
```

```
Q2. Do I run rsc:true or rsc:false?
    framework has React Server Components (Next.js App Router, TanStack Start
        with RSC enabled) --> rsc:true
    framework is client-only (Vite, React Router v7 by default, Astro,
        Next.js Pages Router) --> rsc:false
    rule: NEVER set rsc:true unless the bundler actually compiles
          "use client" boundaries. Wrong value silently breaks SSR.
```

```
Q3. Which Tailwind version?
    Tailwind v4 (default for Vite, Next.js, Astro, TanStack Start in 2026)
        --> single CSS file with `@import "tailwindcss";`, NO tailwind.config.js,
            plugin: @tailwindcss/vite (Vite/TanStack Start/Astro) OR
            @tailwindcss/postcss (Next.js).
    Tailwind v3 (legacy projects only)
        --> tailwind.config.{js,ts} with `content` array, HSL space-separated
            CSS variables. See shadcn-errors-tailwind-v3-v4-migration.
```

## Per-Framework Setup Matrix

| Framework | Init flag | `rsc` | `tsx` | Alias system | Tailwind plugin | Dark-mode strategy |
|---|---|---|---|---|---|---|
| Next.js (App Router) | `-t next` | `true` | `true` | `tsconfig.json#paths` (`@/*`) | `@tailwindcss/postcss` | `next-themes` ThemeProvider in `app/layout.tsx` |
| Next.js (Pages Router) | `-t next` | `false` | `true` | `tsconfig.json#paths` (`@/*`) | `@tailwindcss/postcss` | `next-themes` ThemeProvider in `_app.tsx` |
| Vite + React | `-t vite` | `false` | `true` | `tsconfig.json` + `tsconfig.app.json` + `vite.config.ts#resolve.alias` | `@tailwindcss/vite` | Custom React Context ThemeProvider |
| React Router v7 | `-t react-router` | `false` | `true` | tsconfig paths (`~/*`) | `@tailwindcss/vite` | Custom Context in `app/root.tsx` |
| Astro + React | `-t astro` | `false` | `true` | `tsconfig.json#paths` (`@/*`) | `@tailwindcss/vite` | Inline `<script is:inline>` in Layout |
| TanStack Start | `-t start` | `true` | `true` | tsconfig paths OR `package.json#imports` | `@tailwindcss/vite` | Custom Context in `__root.tsx` |

ALWAYS use the matching `-t` flag. NEVER hand-edit `components.json.rsc`
to `true` in a Vite project. NEVER omit the alias entry from
`vite.config.ts` (TypeScript resolves but Vite's bundler will not).

## Section: Next.js

ALWAYS pass `-t next`. App Router is the documented default; Pages Router
still works but loses the auto-injected `"use client"` benefit.

```bash
# new project
pnpm create next-app@latest my-app
cd my-app
pnpm dlx shadcn@latest init -t next

# add components
pnpm dlx shadcn@latest add button
```

`components.json` for Next.js App Router:

```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "new-york",
  "rsc": true,
  "tsx": true,
  "tailwind": {
    "config": "",
    "css": "app/globals.css",
    "baseColor": "neutral",
    "cssVariables": true
  },
  "aliases": {
    "components": "@/components",
    "ui": "@/components/ui",
    "utils": "@/lib/utils",
    "lib": "@/lib",
    "hooks": "@/hooks"
  },
  "iconLibrary": "lucide"
}
```

`tsconfig.json` without `--src-dir`:

```json
{ "compilerOptions": { "paths": { "@/*": ["./*"] } } }
```

`tsconfig.json` with `--src-dir`:

```json
{ "compilerOptions": { "paths": { "@/*": ["./src/*"] } } }
```

ThemeProvider in `app/layout.tsx`:

```tsx
// components/theme-provider.tsx
"use client"
import * as React from "react"
import { ThemeProvider as NextThemesProvider } from "next-themes"
export function ThemeProvider(props: React.ComponentProps<typeof NextThemesProvider>) {
  return <NextThemesProvider {...props} />
}
```

```tsx
// app/layout.tsx
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

ALWAYS set `suppressHydrationWarning` on `<html>`. Without it, `next-themes`
will log a hydration mismatch on every page load because the server renders
without a theme class and the client adds one before React hydrates.

For Pages Router, mount the same `ThemeProvider` in `pages/_app.tsx` and
set `components.json.rsc` to `false` (Pages Router has no RSC).

## Section: Vite

```bash
pnpm create vite@latest my-app --template react-ts
cd my-app
pnpm add tailwindcss @tailwindcss/vite
pnpm add -D @types/node
pnpm dlx shadcn@latest init -t vite
pnpm dlx shadcn@latest add button
```

`vite.config.ts` MUST register `@tailwindcss/vite` AND the `@` alias:

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

Vite's TypeScript template ships TWO tsconfig files. Both MUST declare the
alias or the editor and the bundler will disagree:

```jsonc
// tsconfig.json
{ "compilerOptions": { "baseUrl": ".", "paths": { "@/*": ["./src/*"] } } }
```

```jsonc
// tsconfig.app.json
{ "compilerOptions": { "baseUrl": ".", "paths": { "@/*": ["./src/*"] } } }
```

`src/index.css` (Tailwind v4 single-line import):

```css
@import "tailwindcss";
```

`components.json` for Vite: `"rsc": false`. ALWAYS. Vite has no concept of
server components and the CLI will skip the `"use client"` directive if
`rsc` is `true`, producing components that fail at runtime when they touch
browser-only APIs.

Vite has NO `next-themes` equivalent. Use the custom React Context
ThemeProvider in `examples.md` and mount it at the root of `App.tsx`.

## Section: React Router v7 (former Remix)

```bash
pnpm create react-router@latest my-app
cd my-app
pnpm dlx shadcn@latest init -t react-router
pnpm dlx shadcn@latest add button
```

`create-react-router` pre-configures Tailwind and the `~/*` alias. The
shadcn CLI writes `components.json` with `rsc:false` and aliases pointing
at `~/components/ui`, `~/lib/utils` etc. Imports use `~`, NOT `@`:

```tsx
import { Button } from "~/components/ui/button"
```

Loaders and actions are React Router primitives, not shadcn primitives.
shadcn `<Form>` works with `useFetcher` or `<Form method="post">` from
react-router. See `examples.md` for the loader-driven pattern.

Dark mode: same custom React Context as Vite, mounted in `app/root.tsx`.
The `<Meta />`, `<Links />`, and `<Scripts />` from react-router stay
intact; wrap `<Outlet />`.

## Section: Astro

Astro is islands-first. Static HTML is shipped by default; React
components hydrate only when given a `client:*` directive.

```bash
pnpm create astro@latest astro-app -- --template with-tailwindcss --install --add react --git
cd astro-app
pnpm dlx shadcn@latest init -t astro
pnpm dlx shadcn@latest add button
```

`astro.config.mjs`:

```js
import { defineConfig } from "astro/config"
import react from "@astrojs/react"
import tailwindcss from "@tailwindcss/vite"
export default defineConfig({
  integrations: [react()],
  vite: { plugins: [tailwindcss()] },
})
```

`tsconfig.json`:

```json
{ "compilerOptions": { "baseUrl": ".", "paths": { "@/*": ["./src/*"] } } }
```

`components.json` has `rsc:false`. Astro components (`.astro` files) are
server-only by definition; shadcn components are React and need a hydration
directive when interactive:

```astro
---
import { Button } from "@/components/ui/button"
import ModeToggle from "@/components/mode-toggle"
---
<!-- presentational, no JS needed -->
<Button>Static</Button>

<!-- interactive, MUST hydrate -->
<ModeToggle client:load />
```

`client:load` hydrates immediately. `client:idle` waits for `requestIdleCallback`.
`client:visible` waits for intersection. `client:only="react"` skips SSR.
ALWAYS pick the latest valid directive; never ship interactivity without one.

Dark mode in Astro: inline `<script is:inline>` in the Layout (runs before
hydration to prevent flash). See `examples.md`.

## Section: TanStack Start

```bash
pnpm dlx shadcn@latest init --template start
# or for an existing project:
pnpm dlx shadcn@latest init -t start
pnpm dlx shadcn@latest add card
```

TanStack Start supports RSC (newer feature). `components.json` ships with
`rsc:true` when generated by the `-t start` template. The bundler is Vite
under the hood; alias resolution can use either tsconfig paths or
`package.json#imports`. Default is tsconfig paths with `@/*`.

Components import inside route files:

```tsx
import { createFileRoute } from "@tanstack/react-router"
import { Card, CardHeader, CardTitle } from "@/components/ui/card"

export const Route = createFileRoute("/")({ component: App })
function App() { return <Card><CardHeader><CardTitle>Hi</CardTitle></CardHeader></Card> }
```

Dark mode: TanStack Start does NOT ship `next-themes`. Use the same custom
Context pattern as Vite, mounted in `app/routes/__root.tsx` around `<Outlet />`.

## Alias Setup Per Framework

ALWAYS keep tsconfig paths and the bundler alias in lockstep. A mismatch
produces "module not found" at runtime even when the editor resolves it.

| Framework | Editor (tsconfig) | Bundler | Astro/runtime |
|---|---|---|---|
| Next.js | `paths`: `"@/*": ["./*"]` or `["./src/*"]` | webpack/turbopack auto | n/a |
| Vite | `paths` in `tsconfig.json` AND `tsconfig.app.json` | `vite.config.ts#resolve.alias` | n/a |
| React Router | `paths`: `"~/*": ["./app/*"]` (auto-generated) | Vite under the hood | n/a |
| Astro | `paths`: `"@/*": ["./src/*"]` | Vite under the hood | `astro.config.mjs` integrations only |
| TanStack Start | `paths`: `"@/*": ["./src/*"]` OR `package.json#imports` | Vite | n/a |

If `components.json.aliases.ui = "@/components/ui"` then tsconfig MUST
resolve `@/components/ui/button` to `./src/components/ui/button.tsx`
(or `./components/ui/button.tsx` without `--src-dir`). The bundler MUST
agree. NEVER let them drift.

## ThemeProvider Per Framework

Three distinct strategies. Mixing them is a recipe for hydration warnings.

1. **Next.js** : `next-themes` with `<ThemeProvider attribute="class">` and
   `suppressHydrationWarning` on `<html>`. App Router places it in
   `app/layout.tsx`; Pages Router in `pages/_app.tsx`. Pages Router users
   also need a `<Script>` in `_document.tsx` to set the class before
   hydration, OR accept a flash.
2. **Vite, React Router, TanStack Start** : custom `ThemeProvider` using
   React Context + localStorage + `matchMedia`. No external library. Full
   code in `examples.md`.
3. **Astro** : inline `<script is:inline>` in the layout, runs before any
   React hydration, sets `dark` class on `documentElement`. The React
   `ModeToggle` island uses `client:load` and a `useEffect` that re-syncs
   the class. This is the only pattern that survives Astro's SSR-first
   model without a flash.

ALWAYS pick the strategy that matches the framework template flag. NEVER
install `next-themes` in a Vite or Astro project; it depends on Next.js
runtime APIs that are absent.

## Companion Skills

- `shadcn-core-cli` : full `init`, `add`, `diff`, `build` flag reference
- `shadcn-impl-component-install` : per-component add workflow, dry-run,
  overwrite policy, monorepo `-c` flag
- `shadcn-impl-rsc-vs-client-boundaries` : when each shadcn component
  needs `"use client"`, validator agent, RSC-safe component list
- `shadcn-impl-theming-custom` : token customization, oklch vs HSL,
  `@theme inline`, per-framework token wiring
- `shadcn-errors-tailwind-v3-v4-migration` : migrating from
  `tailwind.config.js` + HSL to `@import "tailwindcss"` + oklch
- `shadcn-errors-cli-sync-mismatch` : recovering from `init` overwriting
  customized `components.json` or `globals.css`

## Reference Links

- https://ui.shadcn.com/docs/installation
- https://ui.shadcn.com/docs/installation/next
- https://ui.shadcn.com/docs/installation/vite
- https://ui.shadcn.com/docs/installation/react-router
- https://ui.shadcn.com/docs/installation/astro
- https://ui.shadcn.com/docs/installation/tanstack
- https://ui.shadcn.com/docs/components-json
- https://ui.shadcn.com/docs/dark-mode/next
- https://ui.shadcn.com/docs/dark-mode/vite
- https://ui.shadcn.com/docs/dark-mode/astro

See `methods.md` for the init-command + tsconfig matrix per framework.
See `examples.md` for end-to-end setups per framework.
See `anti-patterns.md` for the five recurring init failures.
