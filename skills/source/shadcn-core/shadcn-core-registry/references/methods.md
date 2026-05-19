# Reference : components.json + Registry Schemas

Exhaustive field-by-field schema reference for `components.json`, the
`registries` sub-object, and the server-side `registry.json` plus
registry-item JSON. All entries verified 2026-05-19 against :

- https://ui.shadcn.com/docs/components-json
- https://ui.shadcn.com/docs/registry
- https://ui.shadcn.com/docs/registry/namespace
- https://ui.shadcn.com/docs/registry/getting-started
- https://ui.shadcn.com/docs/registry/examples

## Section 1 : components.json Top-Level Fields

### `$schema`

- **Type** : string (URL)
- **Required** : No
- **Default** : none ; the canonical value is `https://ui.shadcn.com/schema.json`
- **Allowed values** : any URL ; only the canonical value gives IDE
  validation
- **Immutable after init** : No
- **Purpose** : Enables JSON-schema validation and autocomplete in
  schema-aware editors (VS Code, JetBrains).

### `style`

- **Type** : string (enum)
- **Required** : Yes (set by `init`)
- **Default** : `"new-york"` (since 4.x)
- **Allowed values** : `"new-york"` (recommended), `"sera"` (added 4.3.0),
  `"luma"` (added 4.1.2), `"default"` (DEPRECATED)
- **Immutable after init** : **YES**
- **Purpose** : Selects the style preset used for generated components.
  The CLI fetches per-style file content from the registry ; the
  `{style}` URL placeholder resolves to this value.

### `rsc`

- **Type** : boolean
- **Required** : No
- **Default** : `false`
- **Immutable after init** : No
- **Purpose** : When `true`, the CLI emits `"use client"` directives on
  components that need them and omits the directive from pure-presentation
  components (Card, Badge, Alert, Skeleton, Separator). When `false`,
  every generated client component carries `"use client"` unconditionally.

### `tsx`

- **Type** : boolean
- **Required** : No
- **Default** : `true`
- **Immutable after init** : No
- **Purpose** : Controls output file extension. `true` writes `.tsx`,
  `false` writes `.jsx`. The component source is otherwise identical.

## Section 2 : tailwind Sub-Object

### `tailwind.config`

- **Type** : string (path) OR omitted
- **Required** : Conditional ; required for Tailwind v3, OMIT for v4
- **Default** : `"tailwind.config.js"` for v3 ; absent for v4
- **Allowed values** : `"tailwind.config.js"`, `"tailwind.config.ts"`, or
  any project-local path
- **Immutable after init** : No (but switching Tailwind generations is
  a separate migration)
- **Purpose** : Points the CLI at the Tailwind config file for v3
  projects. v4 is CSS-first and has no JS/TS config.

### `tailwind.css`

- **Type** : string (path)
- **Required** : Yes
- **Default** : none
- **Allowed values** : any project-local path, typically
  `"app/globals.css"` (Next.js), `"src/index.css"` (Vite),
  `"styles/global.css"` (Astro)
- **Immutable after init** : No
- **Purpose** : Points the CLI at the CSS file that imports Tailwind. The
  CLI writes generated theme tokens into this file.

### `tailwind.baseColor`

- **Type** : string (enum)
- **Required** : Yes
- **Default** : none ; the `init` flow asks
- **Allowed values** : `"neutral"`, `"stone"`, `"zinc"`, `"mauve"`,
  `"olive"`, `"mist"`, `"taupe"`
- **Immutable after init** : **YES**
- **Purpose** : Generates the default theme-token palette
  (`--background`, `--foreground`, `--primary`, etc.) from one of seven
  curated neutrals. ALL installed components reference the palette,
  which is why this is immutable post-install without a full sweep.

### `tailwind.cssVariables`

- **Type** : boolean
- **Required** : Yes
- **Default** : `true`
- **Allowed values** : `true`, `false`
- **Immutable after init** : **YES**
- **Purpose** : `true` generates semantic-token usage
  (`<div className="bg-background">`). `false` generates inline-utility
  usage (`<div className="bg-zinc-950">`). Theme switching only works
  with `true`. Mixing files generated under different settings breaks
  visually.

### `tailwind.prefix`

- **Type** : string
- **Required** : No
- **Default** : `""` (empty)
- **Allowed values** : any prefix matching the Tailwind `prefix` config
  (`"tw-"`, `"shadcn-"`, etc.)
- **Immutable after init** : No (but altering it requires updating the
  Tailwind config in lock-step)
- **Purpose** : Prefixes every generated Tailwind class. Useful when
  embedding shadcn into a host page that already has a competing
  utility-class system.

## Section 3 : aliases Sub-Object

All alias fields are strings holding an import path. Both classic
tsconfig path aliases (`@/lib/utils`) and Node.js `package.json#imports`
subpaths (`#lib/utils`, new in shadcn@4.7.0) are accepted. Use one
form consistently per project.

| Field | Required | Default | Purpose |
|-------|----------|---------|---------|
| `aliases.components` | Yes | `@/components` | Root for non-UI components and block scaffolds |
| `aliases.utils` | Yes | `@/lib/utils` | The `cn` helper module path ; imported by every generated component |
| `aliases.ui` | Yes | `@/components/ui` | Destination for `registry:ui` items (Button, Card, Dialog, etc.) |
| `aliases.lib` | Yes | `@/lib` | Destination for `registry:lib` items |
| `aliases.hooks` | Yes | `@/hooks` | Destination for `registry:hook` items (e.g., `use-media-query`) |

Immutability note : changing aliases AFTER components are installed
does NOT rewrite existing files. The CLI only honours aliases on NEW
writes. Renaming a path means manually editing every existing import.

## Section 4 : iconLibrary

- **Type** : string
- **Required** : No
- **Default** : `"lucide"`
- **Allowed values** : `"lucide"`, `"radix"`, `"tabler"`, and any
  additional libraries the CLI registers
- **Immutable after init** : No ; the supported way to change it is
  `shadcn migrate icons` (see [shadcn-core-cli](../../shadcn-core-cli/SKILL.md))
- **Purpose** : Identifies the icon library the CLI uses for components
  that ship icons (e.g., Dialog close, Select chevron). `migrate icons`
  rewrites every icon import across `aliases.ui` when this field changes.

## Section 5 : registries

- **Type** : object (keyed by namespace string)
- **Required** : No
- **Default** : `{}` (implicit `@shadcn` only)
- **Allowed values** : namespace keys MUST start with `@`. Values are
  either a URL string (short form) or an object (advanced form).
- **Immutable after init** : No

### Short form

Value is a URL template :

```json
{ "registries": { "@acme": "https://registry.acme.com/{name}.json" } }
```

Constraints :

- The template MUST include `{name}`.
- The template MAY include `{style}`.
- No headers, no params ; the request is anonymous.

### Advanced form

Value is an object :

| Field | Type | Required | Purpose |
|-------|------|----------|---------|
| `url` | string (template) | Yes | Same template rules as short form |
| `headers` | object (string -> string) | No | Sent on every fetch ; values support `${VAR_NAME}` |
| `params` | object (string -> string) | No | Appended as query string ; values support `${VAR_NAME}` |

### URL placeholders

| Placeholder | Source | Required |
|-------------|--------|----------|
| `{name}` | The item portion after the namespace separator (`@acme/button` -> `button`) | Yes |
| `{style}` | The top-level `style` field of `components.json` | No |

### Env-var expansion (`${VAR_NAME}`)

- Applies inside : `url`, `headers` values, `params` values.
- Resolved from `process.env` at CLI invocation time.
- Unset variables expand to the empty string. No error is raised.

### Resolution algorithm

```
input: shadcn add @ns/item
  ├─ parse namespace ("@ns") and item name ("item")
  ├─ lookup registries["@ns"]
  │   ├─ if missing → error
  │   └─ if short form → use as `url`, empty headers/params
  │   └─ if object form → read url/headers/params
  ├─ substitute placeholders ({name}=item, {style}=components.json#style)
  ├─ interpolate ${VAR_NAME} against process.env
  ├─ HTTP fetch with headers + params and:
  │   ├─ User-Agent: shadcn
  │   └─ Accept: application/vnd.shadcn.v1+json
  └─ validate JSON response against registry-item schema
```

Bare names (no `@` prefix) always resolve against `@shadcn`. The default
`@shadcn` registry has implicit URL `https://ui.shadcn.com/r/{name}.json`
and does not need to be declared.

## Section 6 : registry.json (Server-Side Index)

Schema URL : `https://ui.shadcn.com/schema/registry.json`.

The `registry.json` file is the INDEX of items a publisher serves. The
shadcn CLI does NOT consume `registry.json` directly ; the CLI fetches
per-item JSON. `registry.json` is the publisher-side source from which
`shadcn build` generates per-item JSON files.

| Field | Type | Required | Purpose |
|-------|------|----------|---------|
| `$schema` | string (URL) | No | Validation pointer |
| `name` | string | Yes | Registry display name (`"acme"`) |
| `homepage` | string (URL) | No | Public homepage link |
| `items` | array of registry-item objects | Yes | All published items |

## Section 7 : Registry Item Schema

Schema URL : `https://ui.shadcn.com/schema/registry-item.json`.

Per-item shape (verified from /docs/registry/getting-started and
/docs/registry/examples) :

| Field | Type | Required | Purpose |
|-------|------|----------|---------|
| `$schema` | string (URL) | No | Validation pointer |
| `name` | string | Yes | Unique identifier within the registry |
| `type` | string (enum) | Yes | Item category (see Section 8) |
| `title` | string | Yes | Human-readable display name |
| `description` | string | Yes | One-line purpose explanation |
| `dependencies` | array of strings | No | npm packages required (e.g., `"zod@^3.20.0"`) |
| `registryDependencies` | array of strings | No | Other registry items required (resolved recursively) |
| `files` | array of file-object | Yes | Source files to write |
| `cssVars` | object | No | Theme tokens to merge into project CSS (see Section 9) |
| `tailwind` | object | No | Tailwind config overrides |
| `docs` | string (URL) | No | Documentation link surfaced by `shadcn docs` |

### File-object shape

| Field | Type | Required | Purpose |
|-------|------|----------|---------|
| `path` | string | Yes | Source-relative path used during `shadcn build` |
| `type` | string (enum) | Yes | Item-type for THIS file ; controls destination via alias mapping |
| `target` | string | No | Override destination ; supports `@components/`, `@ui/`, `@lib/`, `@hooks/` placeholders that resolve via project aliases |
| `content` | string | Yes (in built output) | The full file source (multi-line ; JSON-escaped). Authored as actual files in the publisher project ; `shadcn build` inlines them. |

## Section 8 : Registry Item Types

Nine item types are documented (verified) :

| Type | Default destination | Use for |
|------|---------------------|---------|
| `registry:ui` | `aliases.ui` | Primitive UI components (Button, Card, Dialog) |
| `registry:component` | `aliases.components` | Composite components (DataTable, ResponsiveDialog) |
| `registry:block` | `aliases.components` | Multi-file scaffolds (login pages, dashboards) |
| `registry:hook` | `aliases.hooks` | React hooks (`use-media-query`) |
| `registry:lib` | `aliases.lib` | Utility code (`format-date`, `cn`) |
| `registry:page` | project-relative | Full page or route files |
| `registry:file` | none (explicit `target` REQUIRED) | Arbitrary file |
| `registry:style` | `tailwind.css` (merged) | Style/theme bundles |
| `registry:theme` | `tailwind.css` (merged) | Pure theme definitions |

The `type` field on the ITEM controls the default destination ; the
`type` field on each FILES entry may differ when one item bundles
multiple kinds (e.g., a block that includes a hook).

## Section 9 : cssVars Sub-Object

Items MAY ship theme tokens that get merged into the consumer's
`tailwind.css` (when `cssVariables: true`). Shape :

```json
{
  "cssVars": {
    "light": {
      "card-bg": "oklch(0.985 0 0)",
      "card-border": "oklch(0.92 0.004 286.32)"
    },
    "dark": {
      "card-bg": "oklch(0.141 0.005 285.823)",
      "card-border": "oklch(0.274 0.006 286.033)"
    }
  }
}
```

Keys at the second level are CSS-custom-property names (without the
leading `--`). The values follow whatever color format the project
uses (oklch for Tailwind v4 ; HSL space-separated for v3). The CLI
merges these into the corresponding `:root` and `.dark` blocks.

## Section 10 : Content Negotiation Headers

The CLI sends two distinguishing request headers on EVERY fetch :

| Header | Value |
|--------|-------|
| `User-Agent` | `shadcn` |
| `Accept` | `application/vnd.shadcn.v1+json` |

Servers MAY route the same URL to JSON for the CLI and HTML for
browsers based on these headers (verified at
/docs/registry/getting-started). This enables "root hosting" where the
registry sits at the domain root (e.g., `https://acme.com/button` returns
HTML in a browser and JSON to the CLI).
