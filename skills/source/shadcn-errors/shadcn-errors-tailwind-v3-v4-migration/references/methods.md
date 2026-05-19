# Tailwind v3 vs v4 Syntax Reference

This document lists, per concept, the exact syntax differences between Tailwind CSS v3 (pre-2025) and Tailwind CSS v4 (Jan 2025 onward, the default in shadcn evergreen-2026). Every row is verified against the official upgrade guides at https://tailwindcss.com/docs/upgrade-guide and https://ui.shadcn.com/docs/tailwind-v4.

## 1. CSS entry directives

| Concern | v3 | v4 |
|---|---|---|
| Import Tailwind | `@tailwind base;`<br>`@tailwind components;`<br>`@tailwind utilities;` | `@import "tailwindcss";` |
| Plugin registration | `plugins: [require("tailwindcss-animate")]` in JS config | `@import "tw-animate-css";` or `@plugin "..."` in CSS |
| Theme config | `theme.extend` in `tailwind.config.js` | `@theme { --color-...: ...; }` in CSS |
| Inline theme mapping | not applicable | `@theme inline { --color-background: var(--background); }` |
| Optional JS config | required | optional, via `@config "../tailwind.config.js";` |

The v4 file usually starts:

```css
@import "tailwindcss";
@import "tw-animate-css";

@custom-variant dark (&:is(.dark *));
```

## 2. Color tokens

| Concern | v3 | v4 |
|---|---|---|
| Token format | `--background: 0 0% 100%;` (space-separated HSL) | `--background: oklch(1 0 0);` |
| Wrapping needed | yes : `bg-[hsl(var(--background))]` | no : `bg-background` |
| Where mapping lives | `theme.extend.colors.background: "hsl(var(--background))"` in JS config | `@theme inline { --color-background: var(--background); }` in CSS |
| Chart color | `color: "hsl(var(--chart-1))"` | `color: "var(--chart-1)"` |
| Token wrapping in CSS file | raw component values | `oklch(...)` or `hsl(...)` always wrapped |

NEVER mix forms. A v4 file that contains a raw `0 0% 100%` token will fail color resolution silently because v4 parses each token as a CSS color value, not as raw HSL components.

## 3. Dark mode

| Concern | v3 | v4 |
|---|---|---|
| Where defined | `darkMode: ['class']` in `tailwind.config.js` | `@custom-variant dark (&:is(.dark *));` in CSS |
| Token override | `.dark { --background: ... }` in CSS | `.dark { --background: ... }` in CSS (unchanged) |
| Toggle wiring | next-themes ThemeProvider with `attribute="class"` | next-themes ThemeProvider with `attribute="class"` (unchanged) |
| Variant operator | none, attribute-driven | `&:is(.dark *)` ancestor-class selector |

The user-facing toggle code (next-themes, the mode-toggle dropdown, the ThemeProvider wrapper) is UNCHANGED between v3 and v4. Only the variant-registration moved from JS to CSS.

## 4. Animation plugin

| Concern | v3 | v4 |
|---|---|---|
| Package name | `tailwindcss-animate` (npm) | `tw-animate-css` (npm) |
| How loaded | `plugins: [require("tailwindcss-animate")]` in JS config | `@import "tw-animate-css";` in CSS |
| Keyframes available | `accordion-down`, `accordion-up`, slide/fade utilities for Dialog/Sheet | same set of keyframes, exposed via CSS-only import |
| package.json entry | dependencies : `"tailwindcss-animate": "^1.0.7"` | dependencies : `"tw-animate-css": "^1.x"` |
| Removal during migration | uninstall AND remove from CSS or config | install AND add `@import` at top of globals.css |

NEVER leave `tailwindcss-animate` in `package.json` after migrating CSS to v4. The plugin code is dead in v4 and produces no keyframes; components that depend on `accordion-down` (Accordion), `slide-in-from-right` (Sheet, Drawer), `fade-in-0` (Dialog), or `zoom-in-95` (Popover, Tooltip) will silently render without animation.

## 5. Ring utility

| Concern | v3 | v4 |
|---|---|---|
| Default `ring` width | 3px | 1px |
| Equivalent of v3 `ring` | `ring` | `ring-3` |
| `ring-2` semantic | 2px | 2px (unchanged number, but visually thinner because the default fell) |
| Default `ring` color | `blue-500` | `currentColor` |
| Compat mode | not applicable | `@theme { --default-ring-width: 3px; --default-ring-color: var(--color-blue-500); }` |

shadcn ui components copied PRE-v4 typically have `ring-2 ring-ring ring-offset-2` for focus styles. The number is still legal in v4 but the visual appearance was tuned against the v3 default. Re-running `pnpm dlx shadcn@latest add button input select --overwrite` after migration regenerates the components with v4-correct ring values.

## 6. PostCSS

| Concern | v3 | v4 |
|---|---|---|
| Plugin name | `tailwindcss` | `'@tailwindcss/postcss'` |
| Auto-imports | requires `postcss-import` plugin | built in, NO `postcss-import` needed |
| Vendor prefixes | requires `autoprefixer` | built in, NO `autoprefixer` needed |
| Typical config size | three plugins | one plugin |

v4 `postcss.config.js` becomes a single-plugin file. Keeping `autoprefixer` or `postcss-import` is harmless on a fresh v4 project but indicates incomplete cleanup.

## 7. Vite plugin

| Concern | v3 | v4 |
|---|---|---|
| Approach | use PostCSS pipeline (no Vite plugin needed) | use `@tailwindcss/vite` (preferred) OR keep PostCSS pipeline |
| Install | not applicable | `pnpm add -D @tailwindcss/vite` |
| Wiring | not applicable | `plugins: [tailwindcss()]` in `vite.config.ts` |
| Performance | PostCSS pass on every file | dedicated Vite plugin, faster HMR |

For Vite projects in evergreen-2026, ALWAYS use `@tailwindcss/vite` over the PostCSS plugin. For Next.js, Astro, Remix, and React Router projects, ALWAYS use `@tailwindcss/postcss`.

## 8. Removed utilities (v3 names that no longer exist)

| v3 utility | v4 replacement |
|---|---|
| `bg-opacity-50` | `bg-black/50` (opacity-modifier syntax) |
| `text-opacity-50` | `text-black/50` |
| `border-opacity-50` | `border-black/50` |
| `flex-shrink-0` | `shrink-0` |
| `flex-grow` | `grow` |
| `overflow-ellipsis` | `text-ellipsis` |
| `outline-none` | `outline-hidden` |

shadcn-generated components rarely use these, but project-level CSS often does. Grep before committing.

## 9. Renamed utilities (silent breaking change)

| v3 | v4 |
|---|---|
| `shadow-sm` | `shadow-xs` |
| `shadow` | `shadow-sm` |
| `blur-sm` | `blur-xs` |
| `blur` | `blur-sm` |
| `rounded-sm` | `rounded-xs` |
| `rounded` | `rounded-sm` |

These renames are silent: `shadow-sm` still parses in v4 but produces a DIFFERENT visual result than in v3 because the scale shifted.

## 10. `!important` position

| v3 | v4 |
|---|---|
| `!bg-red-500` (prefix) | `bg-red-500!` (suffix) |

`tailwind-merge` understands BOTH forms but the v3 prefix syntax is deprecated in v4 output.

## 11. Arbitrary value syntax for CSS variables

| v3 | v4 |
|---|---|
| `bg-[--brand-color]` | `bg-(--brand-color)` |

v4 uses parentheses to distinguish CSS-variable references from arbitrary literal values.

## 12. Component code shape (shadcn-specific)

| Concern | v3-era shadcn component | v4-era shadcn component |
|---|---|---|
| Function form | `React.forwardRef<...>((...), ref) => (<Primitive ref={ref} .../>))` | `function Name({...}: React.ComponentProps<...>)` |
| displayName | `Component.displayName = "Name"` | not needed |
| Styling hook | className only | `data-slot="name"` attribute |
| Radix import | per-primitive `@radix-ui/react-X` | unified `radix-ui` (post-Feb-2026) |

These are shadcn refactors that landed alongside the Tailwind v4 migration. Components copied via `shadcn add` after the v4 cutover have all four properties; pre-v4 components have none.

## 13. Migration commands (deterministic order)

```bash
# Step 1 : codemod
npx @tailwindcss/upgrade

# Step 2 : refresh shadcn-managed dependencies
pnpm up "@radix-ui/*" lucide-react recharts tailwind-merge clsx --latest

# Step 3 : swap animate plugin
pnpm remove tailwindcss-animate
pnpm add tw-animate-css

# Step 4 : re-pull shadcn components in v4 form
pnpm dlx shadcn@latest add button input select tabs switch dialog sheet popover --overwrite

# Step 5 : verify
pnpm build
```

NEVER skip step 4. The codemod does not refresh component source files; only the shadcn CLI does.

## 14. `components.json` after migration

| Field | v3 value | v4 value |
|---|---|---|
| `tailwind.config` | path to `tailwind.config.js` | empty string or omitted |
| `tailwind.css` | path to `globals.css` | path to `globals.css` (unchanged) |
| `tailwind.baseColor` | one of `slate`/`gray`/... | one of `neutral`/`stone`/`zinc`/`mauve`/`olive`/`mist`/`taupe` |
| `tailwind.cssVariables` | usually `true` | usually `true` (default) |

The `components.json` schema is immutable for `style`, `baseColor`, and `cssVariables` after `init`. Migrating Tailwind generations does NOT require rewriting these fields.
