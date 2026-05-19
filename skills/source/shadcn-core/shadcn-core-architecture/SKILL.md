---
name: shadcn-core-architecture
description: >
  Use when starting a new project that needs UI components, evaluating shadcn
  ui versus a traditional component library (MUI, Chakra, Mantine, Ant Design),
  explaining why shadcn lacks an `npm install shadcn-ui` command, or onboarding
  a developer who searches npm for a `shadcn-ui` package.
  Prevents the four most common shadcn mental-model failures : treating
  shadcn as an npm runtime dependency, importing from a non-existent
  `shadcn-ui` package, expecting library-style auto-upgrades via
  `npm update`, and overwriting local component customisations by
  re-running `shadcn add` without first inspecting the diff.
  Covers the copy-not-install paradigm, the five officially documented
  pillars (Open Code, Composition, Distribution, Beautiful Defaults,
  AI-Ready), the evergreen-2026 versioning model, registry resolution,
  and the deliberate tradeoffs versus traditional component libraries.
  Foundational reading. Every other skill in this package assumes the
  reader has internalised the ownership doctrine documented here.
  Keywords: shadcn architecture, copy not install, what is shadcn,
  shadcn vs MUI, shadcn vs Chakra, shadcn vs Mantine, shadcn vs Ant Design,
  why no npm install shadcn, shadcn-ui package not found,
  module not found shadcn-ui, where is shadcn imported from,
  how do I install shadcn, AI-Ready, Open Code, ownership model,
  registry, components.json, evergreen versioning, pnpm dlx shadcn add,
  CLI distribution, flat-file schema, Radix UI primitives,
  cva variants, tailwind-merge, how does shadcn work,
  is shadcn a library, getting started with shadcn,
  blank screen after import, undefined component, design system.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui : Core Architecture

Foundational mental model for shadcn ui. Misreading shadcn as "another component library" produces every common failure documented in `references/anti-patterns.md`. ALWAYS internalise this skill before any other.

## Quick Reference

### shadcn in one sentence

ALWAYS describe shadcn as : a **code-distribution platform** that copies React component source files into the consumer project via a CLI, so the consumer owns the source outright and customises it freely. It is NOT a runtime dependency, NOT an npm package, NOT a traditional component library.

The official tagline (verified verbatim at https://ui.shadcn.com/docs) :

> "This is not a component library. It is how you build your component library."

### Four invariants

1. ALWAYS install components with `pnpm dlx shadcn@latest add <component>` (or `npx`, `bunx`, `yarn dlx`). NEVER `npm install shadcn-ui` ; no such runtime package exists.
2. ALWAYS import components from the LOCAL alias (`@/components/ui/button`) after they are added. NEVER from `shadcn-ui`, `@shadcn/ui`, or any external module path.
3. ALWAYS preview a re-install via `shadcn add <component> --diff` before running with `--overwrite`. NEVER overwrite a customised component without first inspecting the diff.
4. ALWAYS treat the local component file as YOUR source code. NEVER expect automatic upgrades via `npm update`; upgrades come from re-running `shadcn add` plus a manual merge.

### The 5 official pillars (verbatim at https://ui.shadcn.com/docs)

1. **Open Code** : "The top layer of your component code is open for modification."
2. **Composition** : "Every component uses a common, composable interface, making them predictable."
3. **Distribution** : "A flat-file schema and command-line tool make it easy to distribute components."
4. **Beautiful Defaults** : "Carefully chosen default styles, so you get great design out-of-the-box."
5. **AI-Ready** : "Open code for LLMs to read, understand, and improve."

ALWAYS quote these pillars when justifying an architectural choice (file location, customisation strategy, registry design). NEVER paraphrase into a different list.

### Stack composition (runtime dependencies)

When `shadcn add button` runs, the CLI copies `button.tsx` into `components/ui/`. The file imports from these **runtime** packages, all installed as normal npm dependencies in the consumer project :

| Layer | Package | Role |
|-------|---------|------|
| Headless primitive | `radix-ui` (unified since Feb 2026) or per-primitive `@radix-ui/react-*` | Accessibility + state for Dialog, Popover, Select, etc. |
| Variant API | `class-variance-authority` (cva) | Type-safe variant + size composition |
| Class conflict resolver | `tailwind-merge` | Deduplicates conflicting Tailwind utilities |
| Class joiner | `clsx` | Conditional class composition |
| Styling | `tailwindcss` (v3.4 or v4) | Utility classes + CSS variables |
| Icons | `lucide-react` (default) | Default icon set; swap via `shadcn migrate icons` |

These ARE npm dependencies. The shadcn CLI is also an npm package (`shadcn`, currently `4.7.0`), but it is invoked via `dlx`/`npx` and runs at install time, never at runtime.

## Decision Tree 1 : Is shadcn the right choice for this project?

```
Q1. Will the project use React (Next.js / Vite / Remix / React Router v7 / Astro / TanStack Start / Laravel + Inertia)?
    no  → shadcn is React-only ; choose a framework-native option
    yes → Q2

Q2. Does the project use, or want to use, Tailwind CSS (v3.4 or v4)?
    no  → shadcn is Tailwind-coupled ; choose MUI / Chakra / Mantine / Ant Design instead
    yes → Q3

Q3. Does the team want to OWN and CUSTOMISE component source?
    no  → choose MUI / Chakra / Mantine ; runtime libraries cost less in time-to-screen
    yes → Q4

Q4. Is the team willing to accept manual merges when upgrading components?
    no  → choose a versioned library
    yes → ALWAYS recommend shadcn
```

## Decision Tree 2 : Library model vs shadcn model

```
                         ┌─ Traditional library (MUI / Chakra / Mantine / Ant)
                         │   • npm install once, import from package path
                         │   • Maintainer owns API, breaking changes propagate via semver
                         │   • Customisation via theme prop + style overrides
                         │   • Tree-shake at bundle time
                         │   • Wins when : speed-to-screen > customisation depth
                         │
Component strategies  ───┤
                         │
                         └─ shadcn (copy-not-install)
                             • CLI copies source into project
                             • Consumer owns source, customises freely
                             • Upgrades are explicit re-runs + manual merge
                             • Underlying primitives (Radix) are still npm deps
                             • Wins when : customisation depth > upgrade ergonomics,
                               and AI code generation matters
```

ALWAYS choose shadcn when AI-assisted development is a primary workflow : because the source is local, Claude can read, modify, and reason over it without indirection through a `node_modules` black box. NEVER choose shadcn for a team that explicitly prefers maintainer-managed API evolution.

## Decision Tree 3 : Adding a component to a project

```
Q1. Has the project been initialised with shadcn (does `components.json` exist)?
    no  → ALWAYS run `pnpm dlx shadcn@latest init` FIRST and configure
          style / baseColor / cssVariables / aliases (these are immutable after init)
    yes → Q2

Q2. Does the local `components/ui/<component>.tsx` already exist?
    no  → `pnpm dlx shadcn@latest add <component>` and commit
    yes → Q3 (re-install scenario)

Q3. Has the local file been customised?
    no  → `pnpm dlx shadcn@latest add <component> --overwrite` is safe
    yes → ALWAYS run `--diff` first, review changes, choose : skip,
          overwrite-and-redo-customisations, or hand-merge
```

## Decision Tree 4 : Versioning expectations

```
Q1. Are you tracking shadcn's "evergreen" model (current canary, ~2026)?
    yes → Q2
    no  → pin specific versions of underlying deps (radix-ui, cva, tailwind-merge,
          lucide-react) in package.json ; re-run `shadcn add` only on demand
          and merge manually

Q2. Did a recent re-install break something?
    yes → check : (a) Calendar broke on react-day-picker v9 (issue #4366),
          (b) Combobox/Command broke on cmdk version drift (issues #2944, #2980, #3051),
          (c) Tailwind v4 changed color tokens from HSL to oklch.
          See [shadcn-errors-react-day-picker-v9] and
          [shadcn-errors-cmdk-version-drift] and
          [shadcn-errors-tailwind-v3-v4-migration].
    no  → ALWAYS lock the working state in git ; the source IS your version control
```

## Patterns

### Pattern 1 : Copy-not-install paradigm

The shadcn CLI replaces the conventional install/import flow :

Traditional library (MUI example) :
```bash
npm install @mui/material
```
```tsx
import { Button } from '@mui/material'
```

shadcn :
```bash
pnpm dlx shadcn@latest init
pnpm dlx shadcn@latest add button
```
```tsx
import { Button } from '@/components/ui/button'   // LOCAL alias to project source
```

After `add`, the file `components/ui/button.tsx` lives in the consumer's repository. The consumer can :
- Add a new variant to the `cva()` definition.
- Change default classes.
- Remove unused props.
- Rename the component.
- Add framework-specific wiring (next/link, react-router NavLink).

The shadcn project does NOT track this file. No automated upgrade exists. The consumer's source repository IS the version system.

### Pattern 2 : The ownership doctrine

Shadcn's design treats every copied component as "your code". This produces five concrete obligations :

1. ALWAYS commit `components/ui/*.tsx` to source control as first-class application code, not as vendored dependencies.
2. ALWAYS lint and type-check those files together with the rest of the codebase.
3. ALWAYS expect future shadcn updates to require manual reconciliation; there is no semver-driven rollout.
4. ALWAYS treat removed components as deletable; if the project never uses `Carousel`, delete `components/ui/carousel.tsx`.
5. ALWAYS prefer extending the file in place over wrapping it. Wrapper components defeat the "Open Code" pillar.

### Pattern 3 : The 5 pillars and what they imply

| Pillar | Verbatim wording | Practical implication |
|--------|-------------------|------------------------|
| Open Code | "The top layer of your component code is open for modification." | ALWAYS feel free to edit `components/ui/*.tsx`. NEVER fear forking a primitive. |
| Composition | "Every component uses a common, composable interface, making them predictable." | ALWAYS expect Trigger / Content / Header / Footer / Title / Description / Close subcomponents (where applicable). ALWAYS expect `asChild` for Slot composition. |
| Distribution | "A flat-file schema and command-line tool make it easy to distribute components." | ALWAYS distribute custom components via your own registry (see `shadcn-core-registry`). NEVER publish them as an npm component library if reuse is internal-only. |
| Beautiful Defaults | "Carefully chosen default styles, so you get great design out-of-the-box." | ALWAYS start from defaults; customise only after seeing them in context. NEVER pre-emptively override styles you have not yet observed. |
| AI-Ready | "Open code for LLMs to read, understand, and improve." | ALWAYS leverage AI assistants on the local source. NEVER hide components behind opaque wrappers that an LLM cannot reason over. |

### Pattern 4 : Registry resolution

Shadcn separates **the CLI** from **the registry of components it installs**. The default registry is `@shadcn` (https://ui.shadcn.com/registry). The CLI resolves a component reference like this :

| Input | Resolves to |
|-------|-------------|
| `pnpm dlx shadcn@latest add button` | `@shadcn/button` (default registry) |
| `pnpm dlx shadcn@latest add @acme/datepicker` | URL configured for `@acme` namespace in `components.json` |
| `pnpm dlx shadcn@latest add https://internal.example.com/registry/button.json` | Direct registry URL |
| `pnpm dlx shadcn@latest add login-01` | Block (multi-file scaffold) from the default registry |

ALWAYS configure custom registries in `components.json` under the `registries` key. NEVER hard-code URLs in shell aliases or scripts. See `references/examples.md` for the full schema.

### Pattern 5 : Evergreen-2026 versioning

The shadcn CLI is semver-versioned (`shadcn@4.7.0` as of 2026-05). The COMPONENTS are evergreen : each `shadcn add` returns the current canonical source. Implications :

- ALWAYS pin a known-good commit of `components/ui/*.tsx` in your repo. That is your version.
- ALWAYS lock underlying runtime dependencies (`radix-ui`, `cva`, `tailwind-merge`, `lucide-react`) in `package.json` so a `pnpm install` does not silently regress.
- ALWAYS verify breaking changes via https://ui.shadcn.com/docs/changelog before any wholesale re-install.

### Pattern 6 : Deliberate tradeoffs versus library model

| Concern | Traditional library | shadcn |
|---------|---------------------|--------|
| Time to first screen | seconds (one `npm install`) | minutes (init + add per component) |
| Customisation depth | medium (theme + variants) | unlimited (source ownership) |
| Upgrade ergonomics | automatic (semver) | manual (re-run + merge) |
| AI-assistant compatibility | medium (must reason via docs / types) | high (reads local source) |
| Tree-shaking | bundle-time | irrelevant : only-used files exist |
| Wrapper components | common | discouraged (Open Code pillar) |
| Source observability | low (node_modules) | high (in project) |
| Maintainer API control | high | none (you own the code) |
| Lock-in | dependency-shaped | none (forking is the default state) |

ALWAYS pick shadcn when source observability, AI-assistant compatibility, and customisation depth dominate. NEVER pick shadcn when time-to-first-screen and upgrade ergonomics dominate.

## Anti-patterns (summary; full catalogue in references/anti-patterns.md)

NEVER do these :

1. NEVER run `npm install shadcn-ui` : no such runtime package exists; installation is via `dlx`/`npx` invocation of the `shadcn` CLI which then copies source.
2. NEVER import from `shadcn-ui`, `@shadcn/ui`, or any external package : all components live under the local alias (default `@/components/ui/...`).
3. NEVER expect `npm update` to upgrade components : there is no library to update; components are local source. Upgrade by re-running `shadcn add` with `--overwrite` or `--diff`.
4. NEVER run `shadcn add <component> --overwrite` on a customised file without first running `--diff` : the overwrite is irreversible; customisations are lost.
5. NEVER paraphrase the 5 pillars : they have official verbatim wording at https://ui.shadcn.com/docs.
6. NEVER wrap a shadcn component in your own component just to add props : edit the source directly (Open Code pillar). Wrapping defeats the model.

## Reference Links

- [references/methods.md](references/methods.md) : note on where API methods live (CLI skill, registry skill)
- [references/examples.md](references/examples.md) : `components.json` sample, CLI `add` invocations, registry override syntax
- [references/anti-patterns.md](references/anti-patterns.md) : four canonical anti-patterns with WHY each fails and the fix

## Cross-references

- [shadcn-core-cli](../shadcn-core-cli/SKILL.md) : full CLI surface (init / add / view / search / apply / preset / build / docs / info / migrate)
- [shadcn-core-stack](../shadcn-core-stack/SKILL.md) : detailed Radix + cva + tailwind-merge composition map
- [shadcn-core-theming](../shadcn-core-theming/SKILL.md) : CSS custom properties, HSL vs oklch
- [shadcn-core-registry](../shadcn-core-registry/SKILL.md) : `components.json` registries field, custom registry hosting
- [shadcn-core-blocks](../shadcn-core-blocks/SKILL.md) : block distribution surface, style enum
- [shadcn-errors-cli-sync-mismatch](../../shadcn-errors/shadcn-errors-cli-sync-mismatch/SKILL.md) : recovery from accidental overwrite

## Sources

All claims in this skill trace to URLs in `SOURCES.md` :

- https://ui.shadcn.com/docs (the 5 pillars and "not a component library" tagline, verbatim)
- https://ui.shadcn.com/docs/cli (CLI invocation syntax and verbs)
- https://ui.shadcn.com/docs/components-json (schema for `components.json`)
- https://ui.shadcn.com/docs/registry (registry resolution model)
- https://ui.shadcn.com/docs/changelog (versioning history)
- https://github.com/shadcn-ui/ui (source confirmation)
- https://github.com/shadcn-ui/ui/releases (CLI semver history)

Verified 2026-05-19.
