# shadcn-core-blocks : Methods Reference

Complete API signatures and identifiers relevant to the block distribution surface. Verified against `SOURCES.md` primary URLs on 2026-05-19.

## CLI : block-add invocation

The CLI uses the SAME `add` verb for blocks and components. The block-id (`dashboard-01`, `sidebar-07`, `login-03`, etc.) is the only signal of multi-file scaffold output.

### Canonical signature

```
pnpm dlx shadcn@latest add <name> [...names] [flags]
npx   shadcn@latest add <name> [...names] [flags]
bunx  shadcn@latest add <name> [...names] [flags]
yarn  dlx shadcn@latest add <name> [...names] [flags]
```

`<name>` accepts :
- A component identifier : `button`, `card`, `sidebar`.
- A block identifier : `dashboard-01`, `sidebar-07`, `login-03`.
- A custom-registry reference : `@acme/datepicker`.
- A direct registry URL : `https://registry.example.com/login.json`.
- A preset code.

### Flags relevant to block install

Verified verbatim at https://ui.shadcn.com/docs/cli :

| Flag | Short | Purpose |
|------|-------|---------|
| `--yes` | `-y` | skip confirmation prompt (default false) |
| `--overwrite` | `-o` | overwrite existing files (default false) |
| `--cwd <cwd>` | `-c` | working directory (defaults to current) |
| `--all` | `-a` | add all available components (default false) |
| `--path <path>` | `-p` | path to add the component to |
| `--silent` | `-s` | mute output (default false) |
| `--dry-run` | | preview changes without writing files (default false) |
| `--diff [path]` | | show diff for a file |
| `--view [path]` | | show file contents |

The `--diff` and `--view` flags are the safe-preview surface ALWAYS used before re-running `add` on customised blocks.

### Multi-file output

Blocks produce multi-file scaffolds. The CLI documentation does not enumerate the file output per block ; the canonical inventory is the gallery preview at https://ui.shadcn.com/blocks (per-block detail view shows the file tree).

Expected file categories for a typical dashboard block :

| Category | Example path | Notes |
|----------|--------------|-------|
| Page entry | `app/<route>/page.tsx` (Next.js App Router) ; `src/pages/<route>.tsx` (Pages Router) ; `app/routes/<route>.tsx` (React Router v7) | The framework-specific path is resolved by the CLI from `components.json` aliases. |
| Sub-component folder | `components/<block-name>/` | One `.tsx` per sub-component. |
| Data file | `app/<route>/data.json` or `data.ts` | Only when the block demos data-driven UI. |
| Primitive dependencies | `components/ui/sidebar.tsx`, `components/ui/data-table.tsx`, etc. | Auto-installed if missing. |
| `lib/utils.ts` | `lib/utils.ts` | Auto-installed if missing (contains the `cn` helper). |

ALWAYS use `--dry-run` to enumerate the exact file list for a specific block before installing.

## components.json : style enum

Verified at https://ui.shadcn.com/docs/components-json and https://ui.shadcn.com/docs/changelog (2026-05-19) :

```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "new-york",
  "tailwind": {
    "baseColor": "neutral",
    "cssVariables": true
  },
  "rsc": true,
  "tsx": true,
  "aliases": {
    "components": "@/components",
    "utils": "@/lib/utils",
    "ui": "@/components/ui",
    "lib": "@/lib",
    "hooks": "@/hooks"
  }
}
```

### Style enum values (verified)

| Value | Status | Introduced | CLI minimum | Visual character |
|-------|--------|-----------|-------------|------------------|
| `"default"` | Legacy. Retained for backward compatibility. Deprecated for new projects. | Original (pre-2024) | any | The original shadcn look ; softer than `new-york`. |
| `"new-york"` | Current. Recommended for most new projects. | 2024 | any | Geometric, tight radii, design-system-grade. |
| `"luma"` | Current. | March 2026 | `shadcn@4.1.2` | Soft, ambient, gradient-friendly. SaaS-modern. |
| `"sera"` | Current. | April 2026 | `shadcn@4.3.0` | Typography-first. Print-design principles. Underline controls, uppercase headings. |

### Immutability of `style`

The `style` field is set once at `init` and is operationally **immutable** :

- The CLI provides NO documented `--style-switch` or migration command.
- Manually editing `components.json#style` does NOT retroactively re-style already-installed components (they were copied with the OLD style's classes / variants baked in).
- The only correct "style switch" is : full re-init + re-add of every component and block + manual reconciliation of any local customisations.

ALWAYS treat `style` as a one-time architectural decision. NEVER attempt to mix styles within a project.

## registry.json : block discriminator

Verified at https://ui.shadcn.com/docs/registry. The `registry.json` schema (used by anyone publishing a custom registry) accepts a `type` discriminator on each registry item :

```json
{
  "name": "my-dashboard",
  "type": "registry:block",
  "registryDependencies": ["sidebar", "data-table", "card"],
  "files": [
    { "path": "app/dashboard/page.tsx", "type": "registry:page", "target": "app/dashboard/page.tsx" },
    { "path": "components/dashboard/nav.tsx", "type": "registry:component", "target": "components/dashboard/nav.tsx" }
  ]
}
```

### `type` values observed in registry items

| Value | Use |
|-------|-----|
| `registry:ui` | A primitive component going into `components/ui/`. |
| `registry:component` | A composed component (not a primitive). |
| `registry:block` | A multi-file scaffold (page + sub-components). |
| `registry:page` | An individual page file inside a block. |
| `registry:lib` | A library helper (e.g. `lib/utils.ts`). |
| `registry:hook` | A custom hook. |
| `registry:theme` | A theme override file. |
| `registry:style` | A style preset. |

Consumers do NOT supply these values ; the registry publisher does. The CLI reads them to decide where to copy files. ALWAYS see [shadcn-core-registry](../../shadcn-core-registry/SKILL.md) for the full schema.

## CLI version checks

Verify the installed CLI version before installing a block that depends on a recent style :

```bash
pnpm dlx shadcn@latest --version
# expected : 4.7.0 or newer as of 2026-05-19
```

| Feature | Minimum CLI |
|---------|-------------|
| Block install (`add <block-id>`) | `shadcn@3.x` and later |
| `sera` style | `shadcn@4.3.0` |
| `luma` style | `shadcn@4.1.2` |
| `--dry-run` flag | `shadcn@4.0.0` |
| `--diff` flag | `shadcn@4.0.0` |
| `--view` flag | `shadcn@4.0.0` |
| Package imports / `files.target` aliases | `shadcn@4.7.0` |

ALWAYS bump the CLI before installing a block from a recent style. NEVER assume an older CLI handles the modern style enum values.

## Sources

- https://ui.shadcn.com/docs/cli (verbatim flag list)
- https://ui.shadcn.com/docs/components-json (style enum and schema)
- https://ui.shadcn.com/docs/registry (registry item type discriminator)
- https://ui.shadcn.com/docs/changelog (style introduction dates and CLI version gates)
- https://ui.shadcn.com/blocks (gallery and per-block file inventory)

Verified 2026-05-19.
