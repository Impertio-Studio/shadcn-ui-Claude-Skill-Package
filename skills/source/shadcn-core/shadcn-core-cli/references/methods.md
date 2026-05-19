# shadcn CLI : Complete Method Reference

Source : https://ui.shadcn.com/docs/cli (verified 2026-05-19).
Companion source : https://ui.shadcn.com/docs/components-json.
CLI version : shadcn@4.7.0 (2026-05-05).

## Invocation

All commands run via npx, pnpm dlx, or any package-manager runner :

```bash
npx shadcn@latest <cmd>
pnpm dlx shadcn@latest <cmd>
bunx shadcn@latest <cmd>
yarn dlx shadcn@latest <cmd>
```

The `@latest` tag is REQUIRED to avoid stale cached binaries.

## Shared Flags (every command)

| Flag | Type | Default | Purpose |
|------|------|---------|---------|
| `-c, --cwd <path>` | string | `process.cwd()` | Working directory |
| `-s, --silent` | bool | false | Suppress non-error output |

Not every command honours `--silent` in interactive prompts ; use `-y` to
suppress prompts.

## init

Aliases : `create`.

Arguments : `[components...]` : optional names, URLs, or local paths to add
during init.

| Flag | Type | Default | Purpose |
|------|------|---------|---------|
| `-t, --template <template>` | enum | none | `next \| vite \| start \| react-router \| laravel \| astro` |
| `-b, --base <base>` | enum | none | `radix \| base` (component primitives library) |
| `-p, --preset [name]` | string | none | Apply a preset during init |
| `-y, --yes` | bool | true | Skip prompts |
| `-d, --defaults` | bool | false | Use next template + nova preset, no prompts |
| `-f, --force` | bool | false | Overwrite existing components.json |
| `-n, --name <name>` | string | dir name | New project name (when scaffolding) |
| `--css-variables` | bool | true | Use CSS variables for theming |
| `--no-css-variables` | bool | false | Inline color utilities instead |
| `--monorepo` | bool | false | Scaffold a monorepo |
| `--no-monorepo` | bool | false | Skip monorepo prompt |
| `--rtl` | bool | false | Enable RTL support |
| `--no-rtl` | bool | false | Disable RTL |
| `--pointer` | bool | false | Add `cursor-pointer` to buttons (April 2026) |
| `--no-pointer` | bool | false | Skip pointer styling |
| `--reinstall` | bool | false | Re-install UI components from scratch |
| `--no-reinstall` | bool | false | Skip reinstall prompt |

## add

Arguments : `[components...]` : component names, registry URLs, namespaced
items (`@myorg/name`), or local JSON paths.

| Flag | Type | Default | Purpose |
|------|------|---------|---------|
| `-y, --yes` | bool | false | Skip confirmation |
| `-o, --overwrite` | bool | false | Overwrite existing files (NO undo) |
| `-a, --all` | bool | false | Add every component in registry |
| `-p, --path <path>` | string | `aliases.components` | Custom destination path |
| `--dry-run` | bool | false | Preview, write nothing |
| `--diff [path]` | string OR bool | false | Show file differences |
| `--view [path]` | string OR bool | false | Print contents that would be written |

## view

Arguments : `[items...]` : item names or registry URLs.

Side-effects : none, prints to stdout only.

## search

Aliases : `list`.

Arguments : `<registries...>` : one or more registry namespaces (prefix `@`).

| Flag | Type | Default | Purpose |
|------|------|---------|---------|
| `-q, --query <query>` | string | none | Search query |
| `-l, --limit <number>` | int | 100 | Max results per registry |
| `-o, --offset <number>` | int | 0 | Items to skip (pagination) |

## apply

Arguments : `[preset]` : preset code.

| Flag | Type | Default | Purpose |
|------|------|---------|---------|
| `--preset <preset>` | string | argument | Preset code |
| `--only [parts]` | enum | none | `theme` OR `font` (apply only one part) |
| `-y, --yes` | bool | false | Skip confirmation |

## preset

Subcommands : `decode`, `resolve` (alias `info`), `url`, `open`.

### preset decode `<code>`

| Flag | Type | Purpose |
|------|------|---------|
| `--json` | bool | Emit JSON instead of pretty print |

### preset resolve

| Flag | Type | Purpose |
|------|------|---------|
| `--json` | bool | JSON output |

### preset url `<code>`

Prints shareable URL ; no other flags.

### preset open `<code>`

Opens preset in default browser ; no other flags.

## build

Arguments : `[registry]` : path to `registry.json`. Default `./registry.json`.

| Flag | Type | Default | Purpose |
|------|------|---------|---------|
| `-o, --output <path>` | string | `./public/r` | Output directory |

## docs

Arguments : `[component]` : component name.

| Flag | Type | Default | Purpose |
|------|------|---------|---------|
| `-b, --base <base>` | enum | from components.json | `radix \| base` |
| `--json` | bool | false | JSON output for tooling |

## info

No arguments.

| Flag | Type | Purpose |
|------|------|---------|
| `--json` | bool | JSON output |

## migrate

Arguments : `[migration] [path]` : migration name (`icons`, `radix`, `rtl`)
and optional file or glob.

| Flag | Type | Default | Purpose |
|------|------|---------|---------|
| `-l, --list` | bool | false | List available migrations |
| `-y, --yes` | bool | false | Skip confirmation |

### Available migrations

| Name | Effect |
|------|--------|
| `icons` | Swap icon library (lucide / radix / tabler / heroicons / ...) and rewrite imports |
| `radix` | Rewrite scattered `@radix-ui/react-<X>` imports to the unified `radix-ui` package (Feb 2026) |
| `rtl` | Add right-to-left CSS variants ; updates `tailwind.css` and rewrites components |

## components.json : Complete Field Schema

Verified at https://ui.shadcn.com/docs/components-json.

### Top-level fields

| Field | Type | Default | Required | Immutable after init |
|-------|------|---------|----------|----------------------|
| `$schema` | string URL | `https://ui.shadcn.com/schema.json` | No | No |
| `style` | enum | `new-york` | Yes | YES |
| `rsc` | bool | `false` | No | No |
| `tsx` | bool | `true` | No | No |
| `tailwind` | object | none | Yes | partial |
| `aliases` | object | none | Yes | No |
| `iconLibrary` | string | `lucide` | No (managed by `migrate icons`) | No |
| `registries` | object | none | No | No |

### `style` enum values

| Value | Notes |
|-------|-------|
| `new-york` | Current recommended ; clean modern aesthetic |
| `sera` | Typography-first, serif headings (April 2026, 4.3.0+) |
| `luma` | Soft palette variant (4.1.2+) |
| `default` | DEPRECATED ; no new components shipped against it |

### `tailwind` object

| Field | Type | Default | Required | Immutable | Notes |
|-------|------|---------|----------|-----------|-------|
| `config` | string (path) | `tailwind.config.js` (v3) | Conditional | No | OMIT entirely for v4 (CSS-first) |
| `css` | string (path) | none | Yes | No | Points to file with `@import "tailwindcss"` or `@tailwind` directives |
| `baseColor` | enum | none | Yes | YES | `neutral \| stone \| zinc \| mauve \| olive \| mist \| taupe` |
| `cssVariables` | bool | none | Yes | YES | `true` for semantic tokens, `false` for inline colour utilities |
| `prefix` | string | none | No | No | Tailwind class prefix (v3 `tw-`, v4 `tw:`) |

### `aliases` object

| Field | Type | Default | Required |
|-------|------|---------|----------|
| `utils` | string (alias) | `@/lib/utils` | Yes |
| `components` | string (alias) | `@/components` | Yes |
| `ui` | string (alias) | `@/components/ui` | Yes |
| `lib` | string (alias) | `@/lib` | Yes |
| `hooks` | string (alias) | `@/hooks` | Yes |

Alias values accept either tsconfig-style `@/...` paths OR
package.json#imports-style `#...` paths (since shadcn@4.7.0). Mix-and-match
is NOT supported within the same project.

### `registries` object

Two forms accepted per namespace :

```jsonc
{
  "registries": {
    // Form A : bare URL string with {name} placeholder
    "@shadcn": "https://ui.shadcn.com/r/{name}.json",

    // Form B : object with url, headers, params
    "@private": {
      "url": "https://registry.example.com/{name}.json",
      "headers": { "Authorization": "Bearer ${TOKEN}" },
      "params": { "version": "stable" }
    }
  }
}
```

| Sub-field | Type | Purpose |
|-----------|------|---------|
| `url` | string with `{name}` placeholder | Resolved at fetch time |
| `headers` | object<string,string> | Sent with HTTP request, supports `${ENV_VAR}` |
| `params` | object<string,string> | Query string params, supports `${ENV_VAR}` |

Resolution order :

1. Bare name (`button`) -> default `@shadcn` registry
2. Namespace (`@myorg/datepicker`) -> namespace lookup in `registries`
3. Absolute URL -> direct fetch
4. Local path (starts with `./` or `/`) -> read from disk

### Reading components.json from JS

```ts
import { readFileSync } from 'node:fs'
const config = JSON.parse(readFileSync('components.json', 'utf-8'))
```

The CLI itself validates the schema ; consumer scripts that touch
components.json should validate against `https://ui.shadcn.com/schema.json`.

## package.json#imports : alias alternative (4.7.0+)

Source : https://ui.shadcn.com/docs/changelog (May 2026 entry).

```jsonc
{
  "imports": {
    "#components/*": "./src/components/*",
    "#lib/*": "./src/lib/*",
    "#hooks/*": "./src/hooks/*"
  }
}
```

```jsonc
{
  "aliases": {
    "utils": "#lib/utils",
    "components": "#components",
    "ui": "#components/ui",
    "lib": "#lib",
    "hooks": "#hooks"
  }
}
```

The CLI rewrites generated `import { cn } from "@/lib/utils"` to
`import { cn } from "#lib/utils"` automatically when aliases start with `#`.

Node.js ESM resolves `#`-prefixed specifiers natively via package.json
imports : no bundler-side path-mapping required. The trade-off : `#` aliases
are visible ONLY within the package that defines them (true privacy).
tsconfig `@/*` aliases are visible to TypeScript at compile time but
require build-tool support to resolve at runtime.

## CLI Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | Generic error (registry fetch failed, schema invalid, file IO error) |
| 2 | Argument parse error |

Non-zero exit + stderr message ; the CLI does NOT have stable structured
error codes ; consumers parse exit code only.
