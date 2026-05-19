# Methods : shadcn Core Architecture

This skill is **conceptual**, not API-oriented. It establishes the mental model for the shadcn ecosystem. Concrete methods, commands, and APIs live in dedicated skills.

## Where to find concrete methods

| Topic | Skill |
|-------|-------|
| CLI commands (`init`, `add`, `view`, `search`, `apply`, `preset`, `build`, `docs`, `info`, `migrate`), flag matrix | [shadcn-core-cli](../../shadcn-core-cli/SKILL.md) |
| `components.json` schema (style / rsc / tsx / tailwind / aliases / registries / iconLibrary) | [shadcn-core-registry](../../shadcn-core-registry/SKILL.md) |
| Stack composition map (Radix + cva + tailwind-merge + clsx + lucide-react), `cn()` helper, per-layer responsibilities | [shadcn-core-stack](../../shadcn-core-stack/SKILL.md) |
| CSS-variable tokens (`--background`, `--foreground`, `--primary`, etc.), HSL space-separated (pre-v4) vs oklch (v4), `next-themes` dark-mode wiring | [shadcn-core-theming](../../shadcn-core-theming/SKILL.md) |
| Registry resolution algorithm, custom registries with namespaces, URL templates, env-var expansion, headers, query params | [shadcn-core-registry](../../shadcn-core-registry/SKILL.md) |
| Blocks distribution surface, style enum (`new-york`, `sera`, `luma`), block-add workflow | [shadcn-core-blocks](../../shadcn-core-blocks/SKILL.md) |
| Variant API : `cva(base, { variants, compoundVariants, defaultVariants })`, `VariantProps<typeof X>` | [shadcn-syntax-variant-cva](../../../shadcn-syntax/shadcn-syntax-variant-cva/SKILL.md) |

## Conceptual primitives this skill defines

This skill defines no callable methods. It defines vocabulary :

| Term | Definition |
|------|------------|
| **Copy-not-install** | The CLI copies component source files into the consumer project at install time; nothing is pulled at runtime. |
| **Ownership doctrine** | Once a file is copied, the consumer owns it, may edit it freely, and is responsible for merging future updates. |
| **Evergreen** | Components have no per-component version. Each `shadcn add` returns the current canonical source. Versioning lives in the consumer's git history. |
| **Registry** | A flat-file schema (JSON) addressable by URL or namespace, listing components and their dependencies. |
| **Pillar** | One of the five officially documented design principles. Five exists, all officially-worded; this skill quotes them verbatim. |
| **CLI** | The `shadcn` npm package, invoked via `dlx`/`npx`. Semver-versioned (currently `4.7.0`). |
| **Registry namespace** | A prefix like `@shadcn` or `@acme` that resolves to a registry URL via `components.json`. |

## Reading order for related skills

ALWAYS read in this order when onboarding to shadcn :

1. `shadcn-core-architecture` (this skill) : mental model.
2. `shadcn-core-cli` : how to operate the CLI.
3. `shadcn-core-stack` : what each underlying primitive does.
4. `shadcn-core-theming` : how design tokens are wired.
5. `shadcn-core-registry` : how `components.json` resolves component references.
6. `shadcn-core-blocks` : how block-level distribution differs from component-level.

After core, proceed to category-specific skills (`shadcn-syntax-*`, `shadcn-impl-*`, `shadcn-errors-*`, `shadcn-agents-*`).

## Sources

All conceptual definitions above are anchored in :

- https://ui.shadcn.com/docs (pillars and tagline)
- https://ui.shadcn.com/docs/cli (CLI verbs)
- https://ui.shadcn.com/docs/components-json (registry + aliases schema)
- https://ui.shadcn.com/docs/registry (registry resolution)
- https://github.com/shadcn-ui/ui (source code confirms behaviour)

Verified 2026-05-19.
