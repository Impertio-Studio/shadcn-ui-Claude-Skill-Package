# Methods : per-framework init commands, components.json samples, alias matrix

Authoritative reference. ALWAYS verify against
https://ui.shadcn.com/docs/installation before deviating.

## Init Command Matrix

| Framework | Create command | shadcn init command |
|---|---|---|
| Next.js | `pnpm create next-app@latest my-app` | `pnpm dlx shadcn@latest init -t next` |
| Vite | `pnpm create vite@latest my-app --template react-ts` | `pnpm dlx shadcn@latest init -t vite` |
| React Router v7 | `pnpm create react-router@latest my-app` | `pnpm dlx shadcn@latest init -t react-router` |
| Astro | `pnpm create astro@latest my-app -- --template with-tailwindcss --install --add react --git` | `pnpm dlx shadcn@latest init -t astro` |
| TanStack Start | (use shadcn preset) | `pnpm dlx shadcn@latest init -t start` |
| Laravel + Inertia | `laravel new my-app` | `npx shadcn@latest init` (no `-t`) |
| Monorepo (any) | varies | append `--monorepo` to init |

Component add (all frameworks):

```bash
pnpm dlx shadcn@latest add <component-name>
# monorepo:
pnpm dlx shadcn@latest add <component-name> -c apps/web
```

## components.json Per Framework

### Next.js App Router

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

### Next.js Pages Router

Identical to App Router EXCEPT `rsc: false`. Pages Router has no
React Server Components; setting `rsc:true` produces components missing
`"use client"` that crash at runtime.

### Vite + React

```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "new-york",
  "rsc": false,
  "tsx": true,
  "tailwind": {
    "config": "",
    "css": "src/index.css",
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

### React Router v7

```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "new-york",
  "rsc": false,
  "tsx": true,
  "tailwind": {
    "config": "",
    "css": "app/app.css",
    "baseColor": "neutral",
    "cssVariables": true
  },
  "aliases": {
    "components": "~/components",
    "ui": "~/components/ui",
    "utils": "~/lib/utils",
    "lib": "~/lib",
    "hooks": "~/hooks"
  },
  "iconLibrary": "lucide"
}
```

NOTE the `~` prefix, not `@`. React Router's convention.

### Astro + React

```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "new-york",
  "rsc": false,
  "tsx": true,
  "tailwind": {
    "config": "",
    "css": "src/styles/globals.css",
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

### TanStack Start

```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "new-york",
  "rsc": true,
  "tsx": true,
  "tailwind": {
    "config": "",
    "css": "src/styles/app.css",
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

TanStack Start supports RSC via its new SSR pipeline (2026). The
`-t start` template ships `rsc:true`. If the project disables RSC,
manually flip to `false`.

## tsconfig / Alias Matrix

| Framework | tsconfig file(s) | Alias prefix | Bundler alias file | Bundler key |
|---|---|---|---|---|
| Next.js | `tsconfig.json` | `@/*` -> `./*` or `./src/*` | none (Next resolves) | n/a |
| Vite | `tsconfig.json` + `tsconfig.app.json` | `@/*` -> `./src/*` | `vite.config.ts` | `resolve.alias["@"]` |
| React Router | `tsconfig.json` | `~/*` -> `./app/*` | `vite.config.ts` (auto) | auto by create-react-router |
| Astro | `tsconfig.json` | `@/*` -> `./src/*` | none (Astro Vite auto) | n/a |
| TanStack Start | `tsconfig.json` (or `package.json#imports`) | `@/*` -> `./src/*` | Vite auto | n/a |

### Vite : both tsconfig files MUST agree

```jsonc
// tsconfig.json
{
  "files": [],
  "references": [{ "path": "./tsconfig.app.json" }, { "path": "./tsconfig.node.json" }],
  "compilerOptions": { "baseUrl": ".", "paths": { "@/*": ["./src/*"] } }
}
```

```jsonc
// tsconfig.app.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }
  }
}
```

```ts
// vite.config.ts
import path from "path"
import tailwindcss from "@tailwindcss/vite"
import react from "@vitejs/plugin-react"
import { defineConfig } from "vite"
export default defineConfig({
  plugins: [react(), tailwindcss()],
  resolve: { alias: { "@": path.resolve(__dirname, "./src") } },
})
```

### TanStack Start : alternative via package.json imports

```jsonc
// package.json
{
  "imports": {
    "#components/*": "./src/components/*",
    "#lib/*": "./src/lib/*"
  }
}
```

If using `package.json#imports`, the `components.json` aliases must use
the `#` prefix. The default `-t start` setup uses tsconfig paths with
`@` and is the recommended path.

## Tailwind v4 vs v3 Per Framework

| Framework | Tailwind v4 plugin | v3 fallback |
|---|---|---|
| Vite | `@tailwindcss/vite` plugin in `vite.config.ts` | `tailwindcss` + `autoprefixer` via PostCSS |
| Next.js | `@tailwindcss/postcss` in `postcss.config.mjs` | `tailwindcss` + `autoprefixer` in PostCSS |
| React Router | `@tailwindcss/vite` (auto-configured by create-react-router) | `tailwindcss` PostCSS plugin |
| Astro | `@tailwindcss/vite` in `astro.config.mjs#vite.plugins` | `@astrojs/tailwind` integration (deprecated path) |
| TanStack Start | `@tailwindcss/vite` | `tailwindcss` + PostCSS |

Tailwind v4 single-file CSS:

```css
@import "tailwindcss";
@layer base { :root { --background: oklch(1 0 0); /* ... */ } }
```

Tailwind v3 setup needs a `tailwind.config.{js,ts}` with `content` glob.
Mixing v3 config and v4 import in the same project breaks token
resolution. See `shadcn-errors-tailwind-v3-v4-migration`.

## init Flags Quick Reference

| Flag | Effect |
|---|---|
| `-t, --template <name>` | Template: `next`, `vite`, `react-router`, `astro`, `start` |
| `--preset <code>` | Apply a shadcn-hosted preset (paired with `--template`) |
| `--monorepo` | Scaffold for pnpm workspaces with `apps/` + `packages/` |
| `--src-dir` | Only Next.js: use `src/app` instead of `app` |
| `-c, --cwd <path>` | Run in a specific workspace directory |
| `--force` | Overwrite existing files without prompt |
| `--yes` | Accept all prompts (non-interactive) |

ALWAYS run `init --help` against the installed CLI version to confirm
flag availability; the CLI evolves frequently.
