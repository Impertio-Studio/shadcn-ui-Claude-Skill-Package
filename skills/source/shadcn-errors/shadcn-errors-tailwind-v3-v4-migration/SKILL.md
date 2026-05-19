---
name: shadcn-errors-tailwind-v3-v4-migration
description: >
  Use when a shadcn ui project must move from Tailwind CSS v3 to v4, when colors
  break or look washed out after a Tailwind upgrade, when the dark mode toggle
  stops applying after switching the stylesheet, when `bg-background` renders
  transparent instead of the theme color, when `tailwindcss-animate` is missing
  in `package.json` but components still try to animate, when the Tailwind build
  emits "unknown at-rule @theme" or "unknown at-rule @custom-variant", when
  `tailwind.config.js` is still present in a project that imports the v4
  stylesheet, when ring outlines on Button or Input changed thickness after the
  upgrade, when PostCSS fails with "tailwindcss is not a PostCSS plugin", or when
  AI-generated code mixes v3 syntax (`@tailwind base`, `hsl(var(--background))`,
  `darkMode: ['class']`) with v4 syntax (`@import "tailwindcss"`, `oklch()`,
  `@custom-variant dark`) in the same file. Prevents the canonical migration
  bugs: keeping `tailwind.config.js` alongside `@theme inline` so two config
  sources fight, leaving HSL space-separated tokens after switching to v4 so
  colors silently fail to parse, keeping the deprecated `tailwindcss-animate`
  plugin after v4 removes it, leaving `ring-2` in code that was upgraded so
  default ring width changed, leaving `darkMode: ['class']` in the JS config
  while also adding `@custom-variant dark` in CSS so dark mode runs twice or not
  at all, and using `bg-[hsl(var(--background))]` arbitrary classes in v4 where
  `bg-background` resolves directly. Covers the detection rules (which files
  signal v3 vs v4), the exact migration runbook (codemod, dependencies, CSS
  rewrite, plugin swap, verification), the per-file diffs for `globals.css`,
  `tailwind.config.js`, `postcss.config.js`, `vite.config.ts`, and
  `package.json`, the v3 vs v4 syntax reference per concept (theme, colors,
  dark mode, animate plugin, ring utility, PostCSS, Vite), and six common
  anti-patterns that surface as silent visual regressions.
  Keywords: tailwind v3 to v4, tailwind v4 migration shadcn, oklch migration,
  @theme inline, tw-animate-css, tailwindcss-animate removed, ring utility v4,
  ring-3 vs ring-2, dark mode v4, @custom-variant dark, darkMode class v3,
  tailwind upgrade shadcn, tailwind config removed, @tailwindcss/postcss,
  @tailwindcss/vite, why my colors broke after tailwind upgrade, bg-background
  transparent, hsl var background, oklch var background, @import tailwindcss,
  @tailwind base removed, npx @tailwindcss/upgrade, shadcn evergreen-2026
  tailwind, postcss tailwindcss is not a PostCSS plugin, unknown at-rule @theme,
  unknown at-rule @custom-variant, animations stopped working tailwind v4,
  accordion not animating shadcn, sheet not sliding shadcn, dialog not fading
  shadcn, ring focus too thin, ring focus too thick, my dark toggle stopped,
  tailwind config conflict, v3 v4 mixed syntax, tailwind upgrade codemod.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui Errors: Tailwind v3 to v4 Migration

Tailwind CSS v4 (Jan 2025) rewrote the configuration model, the color format, the animation plugin, the dark mode trigger, the ring utility default, and the PostCSS integration. shadcn ui projects bootstrapped before v4 cannot be upgraded by `npm install` alone: the CSS file, the config file, the PostCSS file, and the `package.json` plugin list must all change in lockstep, or the project enters a half-migrated state where some classes resolve and others fail silently. This is the single largest hallucination risk in AI-generated shadcn code, because training data spans both generations and the v3 patterns look syntactically valid when pasted into a v4 project.

This skill gives a deterministic migration runbook and the detection rules to identify which generation a project is on before changing any code.

## Quick Reference

### Major shifts table (v3 -> v4)

| Concept | v3 (pre-2025) | v4 (evergreen-2026) |
|---|---|---|
| Config location | `tailwind.config.js` (JS) | `@theme inline` block in CSS |
| CSS import | `@tailwind base; @tailwind components; @tailwind utilities;` | `@import "tailwindcss";` |
| Color format | `--background: 0 0% 100%` (HSL space-separated) | `--background: oklch(1 0 0)` |
| Color reference | `bg-[hsl(var(--background))]` or `tailwind.config.js` color extension | `bg-background` (direct, via `@theme inline`) |
| Animation plugin | `tailwindcss-animate` (npm package, in `plugins`) | `tw-animate-css` (CSS `@import`, no plugin entry) |
| Dark mode trigger | `darkMode: ['class']` in `tailwind.config.js` | `@custom-variant dark (&:is(.dark *));` in CSS |
| Ring default width | `ring` = 3px, `ring-2` = 2px | `ring` = 1px, `ring-3` = 3px |
| PostCSS plugin | `tailwindcss: {}` | `'@tailwindcss/postcss': {}` |
| Vite plugin | none required (via PostCSS) | `@tailwindcss/vite` preferred |
| `@tailwind` directives | required | REMOVED, do not use |
| `forwardRef` in components | required pattern | REMOVED, plain function declarations |

### Detection rules (read these BEFORE touching code)

| Signal | Conclusion |
|---|---|
| `tailwind.config.js` or `tailwind.config.ts` exists at project root | Project was bootstrapped as v3 |
| `globals.css` contains `@tailwind base;` | Project is v3 |
| `globals.css` contains `@import "tailwindcss";` | Project is v4 |
| `globals.css` contains `@theme inline` | Project is v4 |
| `globals.css` contains `--background: 0 0% 100%` (no `oklch`, no `hsl()`) | Token block is v3 |
| `globals.css` contains `--background: oklch(...)` | Token block is v4 |
| `package.json` lists `tailwindcss-animate` | Plugin is v3-era |
| `package.json` lists `tw-animate-css` | Plugin is v4-era |
| `postcss.config.js` lists `tailwindcss` plugin | PostCSS is v3-style |
| `postcss.config.js` lists `'@tailwindcss/postcss'` plugin | PostCSS is v4-style |
| `components/ui/*.tsx` uses `React.forwardRef` | Components are v3-era |
| `components/ui/*.tsx` uses plain `function Component({...})` with `data-slot` | Components are v4-era |

ALWAYS run the detection scan first. NEVER edit one signal in isolation: a project where `globals.css` is v4 but `tailwind.config.js` still exists is a half-migrated project and produces unpredictable builds.

### Migration runbook (deterministic order)

ALWAYS execute these steps in this exact order. Each step depends on the previous.

1. **Branch and snapshot** : `git checkout -b tailwind-v4-migration`. NEVER migrate on `main`.
2. **Verify Node version** : `node --version` must be `>= 20`. The v4 codemod requires Node 20+.
3. **Run the official codemod** : `npx @tailwindcss/upgrade`. This rewrites dependencies, the config file, and template files automatically.
4. **Update dependencies to latest shadcn-compatible versions** : `pnpm up "@radix-ui/*" lucide-react recharts tailwind-merge clsx --latest`.
5. **Replace the animate plugin** : remove `tailwindcss-animate` from `package.json` dependencies. Add `tw-animate-css`. Replace `@plugin 'tailwindcss-animate';` (if present) with `@import "tw-animate-css";` at the top of `globals.css`.
6. **Verify token format** : every CSS variable in `:root` and `.dark` must use `oklch(...)`, `hsl(...)`, or a named color, NEVER raw space-separated HSL components.
7. **Verify @theme inline block** : `globals.css` must contain `@theme inline { --color-background: var(--background); ... }` so semantic class names like `bg-background` resolve.
8. **Verify dark mode wiring** : `globals.css` must contain `@custom-variant dark (&:is(.dark *));`. The `darkMode` key in `tailwind.config.js` is no longer read. If `tailwind.config.js` still exists, delete it OR keep it ONLY if the project intentionally retains v3-compat mode (rare).
9. **Verify ring utility usage** : grep for `ring-2` in the codebase. v3 components used `ring-2` for the focus ring. In v4 the visual equivalent is `ring-3` (because `ring` itself dropped from 3px to 1px). Audit shadcn components that were copied pre-v4 (Button, Input, Select, Tabs, Switch).
10. **Re-run shadcn-add for any pre-v4 components** : `pnpm dlx shadcn@latest add button input select --overwrite` regenerates the components in v4 form (with `data-slot`, without `forwardRef`, with v4-correct ring width). NEVER hand-edit the upgrade for a component when re-running the CLI will do it deterministically.
11. **Update PostCSS** : `postcss.config.js` must list `'@tailwindcss/postcss': {}` instead of `tailwindcss: {}`. Remove `postcss-import` and `autoprefixer` entries (v4 handles imports and prefixing internally).
12. **Update Vite (if applicable)** : install `@tailwindcss/vite` and add `tailwindcss()` to the `vite.config.ts` `plugins` array. This is now the preferred integration for Vite projects.
13. **Build and verify** : run `pnpm build`. The build must complete with zero "unknown at-rule" warnings. Open the app, toggle dark mode, focus a Button, and confirm the ring appears at the expected thickness.
14. **Visual regression sweep** : open every page. v3 -> v4 frequently regresses cursor on Button (v4 ships `cursor: default` on `<button>`; shadcn does NOT add `cursor-pointer`), ring thickness on focused inputs, accordion expand/collapse animation timing, and chart fill colors (when `chartConfig` still wraps `hsl(var(--chart-1))` instead of `var(--chart-1)`).

NEVER stop at step 3. The codemod handles most of the rewrite but does not always migrate `tailwindcss-animate` to `tw-animate-css`, does not always rewrite `ring-2` usages, and does not re-run the shadcn CLI to refresh component files. Steps 5 through 13 are manual.

### Per-file changes

#### `globals.css`

The file flips from `@tailwind` directives + `@layer base` token block to `@import "tailwindcss"` + raw token block + `@theme inline` mapping + `@custom-variant dark` + `@import "tw-animate-css"`. See `examples.md` for the full before/after.

#### `tailwind.config.js`

In v4 the JS config is OPTIONAL and not auto-detected. The preferred path for shadcn evergreen-2026 is: DELETE `tailwind.config.js` and move all theming into `globals.css`. If the project has app-specific extensions (custom fonts, custom keyframes that are not animation-related) the v4 escape hatch is `@config "../../tailwind.config.js";` at the top of `globals.css`, but for stock shadcn projects this is not needed.

#### `postcss.config.js`

Three changes: rename the plugin key from `tailwindcss` to `'@tailwindcss/postcss'`, remove `postcss-import` (v4 handles it), and remove `autoprefixer` (v4 handles vendor prefixing). The whole file becomes a single-plugin export. See `examples.md`.

#### `package.json`

Remove `tailwindcss` v3 entry. Add `tailwindcss` v4 + `@tailwindcss/postcss` (or `@tailwindcss/vite` for Vite projects). Remove `tailwindcss-animate`. Add `tw-animate-css`. Remove `autoprefixer` and `postcss-import` if they were standalone. The codemod from step 3 typically handles dependency bumps, but the animate-plugin swap is manual.

### Verification checklist

After completing the runbook, ALL of these must be true:

- [ ] `node --version` reports >= 20
- [ ] `tailwind.config.js` is either deleted OR referenced explicitly via `@config` in CSS
- [ ] `globals.css` starts with `@import "tailwindcss";`
- [ ] `globals.css` contains exactly one `@theme inline` block
- [ ] `globals.css` contains `@custom-variant dark (&:is(.dark *));`
- [ ] `globals.css` imports `tw-animate-css` (NOT references `tailwindcss-animate`)
- [ ] All CSS variables use `oklch(...)`, `hsl(...)`, or named colors; NO raw `0 0% 100%`
- [ ] `package.json` has NO `tailwindcss-animate` entry
- [ ] `package.json` has `tw-animate-css` AND `tailwindcss` v4
- [ ] `postcss.config.js` lists `'@tailwindcss/postcss'`, not `tailwindcss`
- [ ] Vite projects use `@tailwindcss/vite` plugin
- [ ] No file mixes `bg-[hsl(var(--background))]` with `bg-background` (the former is v3 syntax)
- [ ] No file mixes `ring-2` with components that were re-pulled in v4 form
- [ ] Dark mode toggle adds `class="dark"` on `<html>` AND the page restyles
- [ ] Build emits zero "unknown at-rule" warnings

If ANY checkbox fails, the project is half-migrated. Re-enter the runbook at the appropriate step.

## Decision Trees

### "My colors broke after upgrade"

```
Did colors break?
├── Yes : open globals.css
│   ├── Tokens use `0 0% 100%` (no oklch, no hsl wrapper) AND file imports tailwindcss v4
│   │   └── This is the canonical v3-token-in-v4-file bug
│   │       Fix : wrap each token with `oklch(...)` or `hsl(...)`, e.g. `--background: oklch(1 0 0);`
│   ├── Tokens use `oklch(...)` AND tailwind.config.js still extends colors with `hsl(var(--background))`
│   │   └── Two color sources fighting; the JS config still expects HSL
│   │       Fix : delete tailwind.config.js color extension, add `@theme inline { --color-background: var(--background); }` in CSS
│   └── Tokens look correct, but className uses `bg-[hsl(var(--background))]`
│       └── This is v3 arbitrary-class syntax
│           Fix : replace with `bg-background`
└── No : skip
```

### "My animations stopped working"

```
Is package.json still listing tailwindcss-animate?
├── Yes : v4 removed support for this plugin
│   Fix :
│   1. `pnpm remove tailwindcss-animate`
│   2. `pnpm add tw-animate-css`
│   3. In globals.css, add `@import "tw-animate-css";` near the top
│   4. Remove any `@plugin 'tailwindcss-animate';` line from CSS
└── No : check the build log for "unknown at-rule" warnings
    └── @plugin syntax may still reference the old plugin name
```

### "My dark toggle stopped working"

```
Open globals.css
├── No `@custom-variant dark` line found
│   └── v3 relied on `darkMode: ['class']` in JS config; v4 needs CSS variant
│       Fix : add `@custom-variant dark (&:is(.dark *));` after the `@import "tailwindcss";` line
├── `@custom-variant dark` present AND tailwind.config.js still has `darkMode: ['class']`
│   └── Both wired; usually harmless but indicates incomplete migration
│       Fix : delete tailwind.config.js OR remove the `darkMode` key
└── `@custom-variant dark` present AND .dark token block missing in globals.css
    └── Variant exists but has nothing to switch to
        Fix : add a `.dark { --background: oklch(...); ... }` block defining every token
```

## Companion Skills

- **shadcn-core-theming** (B2) : Full theming reference covering tokens, semantic color naming, sidebar tokens, chart tokens. Use AFTER migration to understand the v4 token shape.
- **shadcn-errors-styling-conflicts** (B11) : When `className` overrides do not visually win, especially during a half-migrated v3/v4 project where class semantics shift mid-file.
- **shadcn-impl-init** : Bootstrapping a fresh project goes through `shadcn init` which generates v4 by default in evergreen-2026; use this skill if migrating an existing project rather than starting fresh.

## Reference Links

- shadcn ui Tailwind v4 guide : https://ui.shadcn.com/docs/tailwind-v4
- Tailwind CSS v3 to v4 upgrade guide : https://tailwindcss.com/docs/upgrade-guide
- Tailwind v4 codemod : https://www.npmjs.com/package/@tailwindcss/upgrade
- tw-animate-css : https://www.npmjs.com/package/tw-animate-css
- shadcn ui theming : https://ui.shadcn.com/docs/theming
- @tailwindcss/postcss : https://www.npmjs.com/package/@tailwindcss/postcss
- @tailwindcss/vite : https://www.npmjs.com/package/@tailwindcss/vite

See `references/methods.md` for the v3-vs-v4 syntax reference table per concept.
See `references/examples.md` for full before/after files (globals.css, tailwind.config.js, postcss.config.js, package.json).
See `references/anti-patterns.md` for the six most common half-migration bugs and their root causes.
