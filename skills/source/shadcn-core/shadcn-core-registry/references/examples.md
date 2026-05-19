# Reference : components.json + Registry Examples

Real, copy-pasteable configurations covering every common shape. All
examples verified against the official docs (2026-05-19) and exercised
against `shadcn@4.7.0`.

## Example 1 : Minimal components.json (Vite + Tailwind v3)

For a fresh Vite + React + Tailwind v3.4 project. The `tailwind.config`
field is REQUIRED on v3.

```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "new-york",
  "rsc": false,
  "tsx": true,
  "tailwind": {
    "config": "tailwind.config.ts",
    "css": "src/index.css",
    "baseColor": "zinc",
    "cssVariables": true
  },
  "aliases": {
    "components": "@/components",
    "utils": "@/lib/utils",
    "ui": "@/components/ui",
    "lib": "@/lib",
    "hooks": "@/hooks"
  }
}
```

Notes :

- `rsc: false` because Vite has no React Server Components.
- `tailwind.config` POINTS at the v3 config file ; omit this field on v4.
- `style: "new-york"` is the recommended default. `"default"` is
  deprecated.
- Aliases assume the project uses `@/*` -> `src/*` in `tsconfig.json`.

## Example 2 : Full components.json (Next.js + Tailwind v4 + multiple registries)

For an App-Router Next.js 15 project with Tailwind v4 (CSS-first config),
RSC enabled, a custom prefix, and three registries declared :

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
    "cssVariables": true,
    "prefix": ""
  },
  "aliases": {
    "components": "@/components",
    "utils": "@/lib/utils",
    "ui": "@/components/ui",
    "lib": "@/lib",
    "hooks": "@/hooks"
  },
  "iconLibrary": "lucide",
  "registries": {
    "@v0": "https://v0.dev/chat/b/{name}",
    "@acme": "https://registry.acme.com/{name}.json",
    "@private": {
      "url": "https://api.company.com/registry/{name}.json",
      "headers": {
        "Authorization": "Bearer ${REGISTRY_TOKEN}"
      },
      "params": {
        "version": "stable"
      }
    }
  }
}
```

Notes :

- `tailwind.config` is `""` (empty string is also accepted) ; CSS-first
  config means there is no JS/TS config to point at.
- `rsc: true` enables auto-placement of `"use client"` directives.
- The `@private` registry uses the advanced object form with an env-var
  bearer token. The literal value `${REGISTRY_TOKEN}` is what gets
  committed ; the actual secret lives in `.env.local` or CI secrets.

## Example 3 : package.json#imports Aliases (NEW in shadcn@4.7.0)

The same `components.json` but using Node.js `#`-prefixed subpath
imports instead of TypeScript path aliases :

```json
// package.json (excerpt)
{
  "imports": {
    "#components/*": "./src/components/*",
    "#lib/*": "./src/lib/*",
    "#hooks/*": "./src/hooks/*"
  }
}
```

```json
// components.json (excerpt)
{
  "aliases": {
    "components": "#components",
    "utils": "#lib/utils",
    "ui": "#components/ui",
    "lib": "#lib",
    "hooks": "#hooks"
  }
}
```

Notes :

- `#`-prefixed imports are an ESM-native feature ; no build-tool
  transform is required.
- All generated component files will import `cn` from `#lib/utils`.
- NEVER define the same alias both in `tsconfig.json#compilerOptions.paths`
  AND `package.json#imports`. Pick one.

## Example 4 : Custom Registry With Bearer Auth + Query Params

A paid component vendor publishes at `registry.vendor.io`. Their
documentation specifies a bearer token and a `tier` query parameter.

```json
{
  "registries": {
    "@vendor": {
      "url": "https://registry.vendor.io/{style}/{name}.json",
      "headers": {
        "Authorization": "Bearer ${VENDOR_TOKEN}"
      },
      "params": {
        "tier": "pro"
      }
    }
  }
}
```

Install :

```bash
pnpm dlx shadcn@latest add @vendor/data-grid
```

The resolved request becomes :

```
GET https://registry.vendor.io/new-york/data-grid.json?tier=pro
Authorization: Bearer <value of VENDOR_TOKEN>
User-Agent: shadcn
Accept: application/vnd.shadcn.v1+json
```

The `{style}` placeholder resolved to `new-york` because that is the
top-level `style` value.

## Example 5 : Env-Var Expansion Inside the URL Itself

For a registry whose host varies per environment :

```json
{
  "registries": {
    "@internal": {
      "url": "https://${REGISTRY_HOST}/r/{name}.json",
      "headers": {
        "X-API-Key": "${INTERNAL_API_KEY}"
      }
    }
  }
}
```

`.env.local` (gitignored) :

```
REGISTRY_HOST=registry.dev.internal.example.com
INTERNAL_API_KEY=dev-key-xxxxx
```

Install :

```bash
pnpm dlx shadcn@latest add @internal/audit-log-table
```

Resolved request :

```
GET https://registry.dev.internal.example.com/r/audit-log-table.json
X-API-Key: dev-key-xxxxx
```

Switching to staging is a single `.env.local` change. The
`components.json` is the same across environments.

## Example 6 : Private Monorepo Registry Pattern

A pnpm-workspace monorepo with a private design-system package at
`packages/design-system` that hosts a registry for OTHER apps in the same
repo at `apps/web` and `apps/admin`.

The design-system package serves its built registry via a local
file-system path (no HTTP) :

```json
// apps/web/components.json
{
  "aliases": {
    "components": "@/components",
    "utils": "@/lib/utils",
    "ui": "@/components/ui",
    "lib": "@/lib",
    "hooks": "@/hooks"
  },
  "registries": {
    "@ds": "../../packages/design-system/public/r/{name}.json"
  }
}
```

Install an internal component :

```bash
pnpm --filter web dlx shadcn@latest add @ds/feature-flag-toggle
```

The CLI fetches from the local path, so no HTTP server is needed.
Publishers re-run `shadcn build` in the design-system package whenever
they update items.

## Example 7 : Minimal Server-Side registry.json

The publisher project's source-of-truth index. `shadcn build` consumes
this and emits per-item JSON files into `./public/r/`.

```json
{
  "$schema": "https://ui.shadcn.com/schema/registry.json",
  "name": "acme",
  "homepage": "https://acme.com",
  "items": [
    {
      "name": "feature-flag-toggle",
      "type": "registry:component",
      "title": "Feature Flag Toggle",
      "description": "A switch bound to the internal feature-flag store.",
      "dependencies": ["zustand@^4.0.0"],
      "registryDependencies": ["switch", "label"],
      "files": [
        {
          "path": "components/feature-flag-toggle.tsx",
          "type": "registry:component",
          "target": "@components/feature-flag-toggle.tsx"
        },
        {
          "path": "hooks/use-feature-flag.ts",
          "type": "registry:hook",
          "target": "@hooks/use-feature-flag.ts"
        }
      ]
    }
  ]
}
```

Notes :

- `registryDependencies: ["switch", "label"]` says : install the default
  `@shadcn/switch` and `@shadcn/label` first.
- `dependencies: ["zustand@^4.0.0"]` is an npm install (the publisher's
  required peer dep).
- The two `files[]` entries are written to different aliases :
  `@components` and `@hooks` resolve from the CONSUMER's
  `components.json`, not the publisher's.

## Example 8 : Minimal Per-Item JSON (post-`build` output)

After `shadcn build`, `./public/r/feature-flag-toggle.json` contains
the same item entry with each file's `content` inlined :

```json
{
  "$schema": "https://ui.shadcn.com/schema/registry-item.json",
  "name": "feature-flag-toggle",
  "type": "registry:component",
  "title": "Feature Flag Toggle",
  "description": "A switch bound to the internal feature-flag store.",
  "dependencies": ["zustand@^4.0.0"],
  "registryDependencies": ["switch", "label"],
  "files": [
    {
      "path": "components/feature-flag-toggle.tsx",
      "type": "registry:component",
      "target": "@components/feature-flag-toggle.tsx",
      "content": "\"use client\"\nimport { Switch } from \"@/components/ui/switch\"\nimport { Label } from \"@/components/ui/label\"\nimport { useFeatureFlag } from \"@/hooks/use-feature-flag\"\n\nexport function FeatureFlagToggle({ flag }: { flag: string }) {\n  const { value, set } = useFeatureFlag(flag)\n  return (\n    <Label className=\"flex items-center gap-2\">\n      <Switch checked={value} onCheckedChange={set} />\n      {flag}\n    </Label>\n  )\n}\n"
    }
  ],
  "cssVars": {
    "light": { "feature-flag-accent": "oklch(0.92 0.06 200)" },
    "dark":  { "feature-flag-accent": "oklch(0.55 0.10 200)" }
  }
}
```

Notes :

- The `content` field is the literal file source, JSON-escaped.
- `cssVars` will be merged into the consumer's `tailwind.css` because
  `tailwind.cssVariables: true` is the typical setting.

## Example 9 : Item With `target` Placeholder Overrides

```json
{
  "name": "auth-redirect-middleware",
  "type": "registry:component",
  "title": "Auth Redirect Middleware",
  "description": "Server middleware that redirects unauthenticated users.",
  "files": [
    {
      "path": "middleware/auth-redirect.ts",
      "type": "registry:file",
      "target": "middleware.ts"
    }
  ]
}
```

The `target: "middleware.ts"` is a project-root-relative path (no `@`
placeholder). `registry:file` items REQUIRE an explicit `target`.

## Example 10 : Multi-Style Registry URL

A registry that serves separate file content for `new-york` and `sera` :

```json
{
  "registries": {
    "@themes": "https://themes.example.com/{style}/{name}.json"
  }
}
```

`components.json#style = "new-york"` and `shadcn add @themes/dashboard-card`
resolves to `https://themes.example.com/new-york/dashboard-card.json`.

If the publisher only serves the `new-york` variant, attempting `shadcn add
@themes/dashboard-card` on a `style: "sera"` project produces a 404. The
publisher is responsible for serving every style declared on its site.
