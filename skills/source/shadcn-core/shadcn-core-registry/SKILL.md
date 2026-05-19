---
name: shadcn-core-registry
description: >
  Use when authoring or consuming a shadcn registry, when configuring the
  `registries` field in components.json, when wiring a private or paid
  component library behind a namespace prefix like `@myorg/datepicker`,
  when securing a custom registry with bearer tokens or API keys, when
  hosting a `registry.json` plus per-item JSON files, or when explaining
  why the `style`, `baseColor`, or `cssVariables` fields cannot be changed
  after `init`. Prevents the missing-`{name}`-placeholder trap that breaks
  every namespace install, the committed-secret leak from inlining an API
  key instead of using `${ENV_VAR}` expansion, the alias-mismatch silent
  failure from setting `aliases.components` to one path while imports
  reference another, the immutable-field surprise from editing
  `style`/`baseColor`/`cssVariables` post-init, and the Tailwind v3-vs-v4
  config-shape confusion. Covers every components.json field in depth
  (purpose, constraints, defaults, immutability), the URL-template syntax
  with `{name}` and `{style}` placeholders, env-var expansion in headers
  and params, the advanced registry config object, the server-side
  `registry.json` schema (items[] with name/type/title/description/files/
  dependencies/registryDependencies/cssVars/tailwind), the nine
  registry:* item types, and content-negotiation headers used by the
  CLI. The CLI command surface itself lives in shadcn-core-cli.
  Keywords: shadcn registry, custom registry, private registry, components.json,
  registries field, namespace, namespaces, registry namespace, @namespace,
  @myorg, @v0, @shadcn, URL template, {name} placeholder, {style}
  placeholder, env var expansion, REGISTRY_TOKEN, bearer token, API key,
  registry headers, registry params, advanced registry config, registry.json,
  registry item schema, registry:ui, registry:component, registry:block,
  registry:hook, registry:lib, registry:page, registry:file, registry:style,
  registry:theme, cssVars, registryDependencies, target placeholder, shadcn
  build, public/r, host my own registry, internal design system,
  paid component library, immutable fields, baseColor cannot change,
  cssVariables immutable, style immutable, aliases mismatch, monorepo
  registry, content negotiation, Accept header, vnd.shadcn.v1+json,
  Authorization Bearer, X-API-Key, how do I publish components, how
  do I sell components, fallback to default registry, registry
  resolution order, what is components.json registries.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui : Registry + components.json (Deep)

The shadcn registry is a code-distribution protocol, not a package index.
A registry serves JSON descriptions of items (components, blocks, hooks,
themes) ; the CLI fetches those descriptions and writes the embedded files
into the consumer project. Every project carries a `components.json` that
describes WHERE to write those files and WHICH registries to read from.

This skill covers the SCHEMA and the HOSTING model. The CLI command
surface (`init`, `add`, `view`, `search`, `build`, `migrate`) lives in
[shadcn-core-cli](../shadcn-core-cli/SKILL.md). Do not duplicate command
syntax here ; link out.

## Quick Reference : Which Registry For Which Need

```
Q1. Are you installing PUBLIC shadcn components only?
    yes  → default `@shadcn` registry, no `registries` field needed
    no   → Q2

Q2. Do you need PRIVATE components (internal design system)?
    yes  → custom registry with `${ENV_VAR}`-expanded auth headers (Pattern 3)
    no   → Q3

Q3. Are you BUYING third-party components (paid registry)?
    yes  → vendor-supplied namespace + token, env-var-expanded (Pattern 3)
    no   → Q4

Q4. Are you publishing a registry for OTHER teams to consume?
    yes  → author `registry.json` + run `shadcn build`, serve `./public/r/*`
           (Pattern 5). The CONSUMER then adds your URL via `registries`.
    no   → you do NOT need to touch `registries` ; the default `@shadcn` works
```

ALWAYS pick the smallest configuration that meets the need. NEVER declare
the `@shadcn` namespace in `registries` unless you are intentionally
overriding the default URL or attaching headers to it.

## Quick Reference : The Four Immutable Decisions

Three fields in `components.json` are IMMUTABLE after `init` (verified at
https://ui.shadcn.com/docs/components-json) :

| Field | Why immutable | What happens if you change it |
|-------|---------------|-------------------------------|
| `style` | The CLI fetches per-style JSON from the registry (`{style}` placeholder is a real construct). Already-installed files were templated against the original style. | Future `add` calls fetch new-style files ; existing files are unchanged. Visual drift across components. |
| `tailwind.baseColor` | Generates the entire CSS-vars palette at install time. Already-installed files reference the original palette names. | Future `add` calls produce mismatched palettes. Old components stay on old palette. |
| `tailwind.cssVariables` | Switches the generated source between `bg-background` (true) and `bg-zinc-950`-style inline utilities (false). | Mixing produces components that read different layers. Theme switching breaks. |

A fourth quasi-immutable concern : the alias triplet (`aliases.components`,
`aliases.ui`, `aliases.utils`). Changing them after install does NOT
rewrite existing imports. The CLI only uses aliases for NEW writes.

ALWAYS commit to all four decisions BEFORE running `init`. NEVER edit
these fields post-init without committing to a full `add --overwrite --all`
sweep AND a manual reconciliation pass.

## The components.json Schema (Overview)

Full per-field reference with types, defaults, and immutability is in
`references/methods.md`. High-level field map :

| Section | Fields | Purpose |
|---------|--------|---------|
| Top-level | `$schema`, `style`, `rsc`, `tsx` | Identity + output format |
| `tailwind` | `config`, `css`, `baseColor`, `cssVariables`, `prefix` | Styling integration |
| `aliases` | `components`, `utils`, `ui`, `lib`, `hooks` | Import + file destinations |
| Optional | `iconLibrary`, `registries` | Icon library + custom registries |

The schema URL is `https://ui.shadcn.com/schema.json`. Pointing
`$schema` at it gives IDE validation and autocomplete.

### Tailwind v3 vs v4 in components.json

| Generation | `tailwind.config` field | Token format |
|------------|-------------------------|--------------|
| Tailwind v3.x | REQUIRED, points to `tailwind.config.js` or `.ts` | HSL space-separated, consumed via `hsl(var(--background))` |
| Tailwind v4.x | OMIT the field entirely (CSS-first config) | oklch, exposed via `@theme inline` so `bg-background` resolves directly |

NEVER mix : with v4 the `config` field MUST be blank ; with v3 omitting it
breaks the CLI's ability to resolve theme tokens. See
[shadcn-errors-tailwind-v3-v4-migration](../../shadcn-errors/shadcn-errors-tailwind-v3-v4-migration/SKILL.md).

### Aliases : The Two Forms

The aliases section accepts BOTH classic tsconfig path aliases (`@/lib/utils`)
and Node.js `package.json#imports` subpath aliases (`#lib/utils`, new in
shadcn@4.7.0). Pick ONE form per project. The CLI uses these aliases for :

1. WHERE to write new component files (`aliases.ui` and `aliases.components`).
2. HOW to rewrite imports inside those files (`aliases.utils` resolves
   `cn` from `@/lib/utils` or `#lib/utils`).

NEVER define the same logical alias under BOTH tsconfig paths AND
`package.json#imports`. Resolution order is implementation-defined.

Command-level alias usage is in [shadcn-core-cli](../shadcn-core-cli/SKILL.md).

## Default Registry vs Custom Registries

The default registry is `@shadcn` (https://ui.shadcn.com/registry,
verified at https://ui.shadcn.com/docs/registry). It is implicit : a
fresh `components.json` does NOT need a `registries` field for `shadcn
add button` to work. The CLI resolves bare names against `@shadcn`.

A CUSTOM registry is any other namespace declared under the `registries`
field. Two shapes are accepted :

```json
{
  "registries": {
    "@v0": "https://v0.dev/chat/b/{name}",
    "@acme": "https://registry.acme.com/{name}.json"
  }
}
```

Short form : namespace key maps directly to a URL TEMPLATE.

```json
{
  "registries": {
    "@private": {
      "url": "https://api.company.com/registry/{name}.json",
      "headers": {
        "Authorization": "Bearer ${REGISTRY_TOKEN}"
      },
      "params": {
        "version": "latest"
      }
    }
  }
}
```

Object form : adds `headers` and `params`. Use when the registry requires
authentication or query parameters.

### Namespace Resolution Order (verified)

1. `shadcn add @myorg/datepicker` parses the namespace `@myorg`.
2. The CLI looks up `@myorg` in `registries`.
3. The URL template is materialised : `{name}` becomes `datepicker`,
   `{style}` becomes the value of `components.json#style`.
4. Headers and params are interpolated against `process.env`.
5. The HTTP request is made with `User-Agent: shadcn` and
   `Accept: application/vnd.shadcn.v1+json`.
6. The returned JSON is validated against the registry-item schema.
7. Files are written, dependencies installed, `registryDependencies`
   recursively resolved.

If the namespace is NOT found in `registries`, the CLI errors. There is
NO silent fallback to `@shadcn` for arbitrary namespaces. The fallback
only applies to BARE names (no `@`-prefix), which always resolve against
`@shadcn`.

## URL Template Syntax

Two placeholders, verified at https://ui.shadcn.com/docs/registry/namespace :

| Placeholder | Source | Example |
|-------------|--------|---------|
| `{name}` | The item portion after the namespace | `@acme/button` -> `{name}` = `button` |
| `{style}` | The `style` field of components.json | `style: "new-york"` -> `{style}` = `new-york` |

The `{name}` placeholder is REQUIRED in every URL template. The `{style}`
placeholder is OPTIONAL ; include it only when the registry serves
per-style variants.

Example with both placeholders :

```json
{
  "registries": {
    "@themes": "https://registry.example.com/{style}/{name}.json"
  }
}
```

Installing `@themes/card` with `style: "new-york"` resolves to
`https://registry.example.com/new-york/card.json`.

NEVER omit `{name}` ; the CLI does NOT append the item name implicitly.
A URL like `"@acme": "https://registry.acme.com"` is broken : it points
at the same file regardless of the requested item.

## Environment Variable Expansion

Variables in `${VAR_NAME}` format expand from `process.env` at the time
the CLI runs. They are valid inside :

- `headers` values
- `params` values
- The `url` string itself

```json
{
  "registries": {
    "@private": {
      "url": "https://${REGISTRY_HOST}/registry/{name}.json",
      "headers": {
        "Authorization": "Bearer ${REGISTRY_TOKEN}",
        "X-API-Key": "${API_KEY}"
      },
      "params": {
        "key": "${API_KEY}"
      }
    }
  }
}
```

Resolution rule : if a referenced variable is unset, the substitution
yields an empty string. The CLI does NOT error on missing variables.

ALWAYS keep secrets in `.env.local` (gitignored) or a CI secret store.
NEVER commit literal tokens in `components.json` ; the file is in source
control.

## Hosting Your Own Registry

A registry is a static set of JSON files conforming to two schemas :

| Schema | URL | Role |
|--------|-----|------|
| Registry root | `https://ui.shadcn.com/schema/registry.json` | Describes the index file (`registry.json`) listing all items |
| Registry item | `https://ui.shadcn.com/schema/registry-item.json` | Describes one item (`button.json`, `card.json`, etc.) |

The publisher workflow :

1. Author `registry.json` describing every item the registry exposes.
2. Author per-item content (in source files within the publisher project).
3. Run `shadcn build` (see [shadcn-core-cli](../shadcn-core-cli/SKILL.md))
   to generate one JSON per item under `./public/r/` (default output).
4. Deploy `public/r/*.json` (or any other path) to a static host.
5. Consumers point the `registries` field at the served URL with `{name}`.

The `registry.json` shape (verified at
https://ui.shadcn.com/docs/registry/getting-started) :

```json
{
  "$schema": "https://ui.shadcn.com/schema/registry.json",
  "name": "acme",
  "homepage": "https://acme.com",
  "items": [
    {
      "name": "button",
      "type": "registry:ui",
      "title": "Button",
      "description": "A reusable button component",
      "dependencies": ["radix-ui"],
      "files": [
        { "path": "ui/button.tsx", "type": "registry:ui" }
      ]
    }
  ]
}
```

Each item declaration is shallow ; the full item content (including the
embedded file source) lives in the generated per-item JSON (`./public/r/button.json`).

### Registry Item Types

Nine item types are documented (verified at
https://ui.shadcn.com/docs/registry/getting-started) :

| Type | Used for |
|------|----------|
| `registry:ui` | Primitive UI components ; written under `aliases.ui` |
| `registry:component` | Composite components ; written under `aliases.components` |
| `registry:block` | Multi-file scaffolds (full pages, dashboards) |
| `registry:hook` | React hooks ; written under `aliases.hooks` |
| `registry:lib` | Utility code ; written under `aliases.lib` |
| `registry:page` | Full page or route file |
| `registry:file` | Arbitrary file ; explicit `target` required |
| `registry:style` | Style/theme bundle (tokens, CSS) |
| `registry:theme` | Pure theme definition (palette + tokens) |

The `type` field controls where the file is written by default. The
`target` field on individual `files[]` entries overrides destination
on a per-file basis.

### Content Negotiation

The CLI sends two distinguishing headers on every fetch :

- `User-Agent: shadcn`
- `Accept: application/vnd.shadcn.v1+json`

Servers MAY route the same URL to JSON for the CLI and HTML for browsers
based on these headers (verified at /docs/registry/getting-started). This
enables "root hosting" where the registry sits at the domain root rather
than under `/r/`.

## Cross-References

- [shadcn-core-architecture](../shadcn-core-architecture/SKILL.md) :
  copy-not-install paradigm, ownership doctrine
- [shadcn-core-cli](../shadcn-core-cli/SKILL.md) : `init`, `add`, `view`,
  `search`, `build`, `migrate` commands, flag surface
- [shadcn-core-theming](../shadcn-core-theming/SKILL.md) : CSS-var palette
  generated from `tailwind.baseColor`
- [shadcn-core-blocks](../shadcn-core-blocks/SKILL.md) : `registry:block`
  item type, multi-file scaffolds
- [shadcn-errors-tailwind-v3-v4-migration](../../shadcn-errors/shadcn-errors-tailwind-v3-v4-migration/SKILL.md) :
  `tailwind.config` field shape per generation

## Reference Files

- `references/methods.md` : every `components.json` field with type,
  default, allowed values, immutability flag, and the registries-object
  sub-schema and registry-item schema
- `references/examples.md` : minimal Vite-v3 config, full Next.js-v4
  config with multiple registries, custom registry with bearer auth,
  env-var-expanded URL, private monorepo pattern, minimal
  `registry.json`, single-item per-item JSON
- `references/anti-patterns.md` : six recurring registry/config mistakes
  with WHY each fails and the verified fix

## Verified Sources

- https://ui.shadcn.com/docs/components-json (schema)
- https://ui.shadcn.com/docs/registry (registry model)
- https://ui.shadcn.com/docs/registry/namespace (URL templates, env vars,
  advanced config object)
- https://ui.shadcn.com/docs/registry/getting-started (`registry.json`
  shape, registry-item schema, item types, content negotiation)
- https://ui.shadcn.com/docs/registry/examples (per-item JSON examples,
  cssVars, target placeholders)
- https://ui.shadcn.com/docs/changelog (4.7.0 `package.json#imports`,
  sera + luma styles, `--pointer` flag)

All claims verified 2026-05-19.
