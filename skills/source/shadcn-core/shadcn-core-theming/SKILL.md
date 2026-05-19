---
name: shadcn-core-theming
description: >
  Use when wiring shadcn ui colors, dark mode, the design-token palette,
  or migrating a project between Tailwind v3 and Tailwind v4, or when
  pasting theme output from https://ui.shadcn.com/themes into a project.
  Prevents the single largest source of AI-generated style bugs in shadcn
  projects : the HSL-space-separated (Tailwind v3) versus oklch (Tailwind v4)
  format split, the hsl(var(--background)) wrapper convention versus the
  bare var(--background) convention, the bg-background utility semantics
  difference, and dark-mode flash-of-unstyled-content from a missing
  suppressHydrationWarning on the html element.
  Covers the exhaustive CSS-custom-property token catalog (core, sidebar,
  chart, radius scale), the @theme inline directive (v4) versus the
  tailwind.config.js theme.extend.colors block (v3), the next-themes
  ThemeProvider wiring on Next.js, the custom localStorage ThemeProvider
  on Vite, the useTheme hook, the mode-toggle component, and the
  copy-from-themes-page workflow.
  Keywords: shadcn theming, design tokens, CSS custom properties,
  CSS variables, --background, --foreground, --primary, --secondary,
  --muted, --accent, --destructive, --border, --input, --ring, --radius,
  HSL space separated, oklch, Tailwind v3, Tailwind v4, @theme inline,
  tailwind.config.js colors, hsl var background, bg-background, dark mode,
  next-themes, ThemeProvider, useTheme, suppressHydrationWarning,
  flash of unstyled content, FOUC, theme builder, ui.shadcn.com themes,
  copy theme CSS, light dark system, default new-york sera luma,
  why are my colors wrong, why is my theme not applying,
  bg-background not working, hsl wrapper, oklch wrapper, my colors look gray,
  how do I add dark mode, how do I change the primary color,
  custom palette, theme override, tw-animate-css, @custom-variant dark,
  semantic colors, color tokens, design system colors.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui : Theming

Deterministic mental model for shadcn theming. The HSL versus oklch split, the wrapper conventions, and the dark-mode wiring are the single largest source of AI-generated style bugs in shadcn projects. ALWAYS identify the Tailwind generation FIRST, then pick the matching wiring.

## Quick Reference

### One question, one answer

ALWAYS ask first : "Is the project Tailwind v3.4 or Tailwind v4?" Every theming decision branches from that answer. NEVER mix v3 and v4 conventions in the same `globals.css`.

### v3 vs v4 at a glance

| Concern | Tailwind v3.4 | Tailwind v4 |
|---------|---------------|-------------|
| Color format in CSS vars | HSL space-separated : `0 0% 100%` | oklch : `oklch(1 0 0)` |
| Wiring location | `tailwind.config.js` extends `colors` | `globals.css` `@theme inline` block |
| Variable usage in utilities | `bg-[hsl(var(--background))]` (in arbitrary) or `bg-background` (after wiring) | `bg-background` (after `@theme inline` mapping) |
| Tailwind base import | `@tailwind base; @tailwind components; @tailwind utilities;` | `@import "tailwindcss";` |
| Dark-mode declaration | `darkMode: ["class"]` in JS config | `@custom-variant dark (&:is(.dark *));` in CSS |
| Animations plugin | `tailwindcss-animate` (npm plugin) | `tw-animate-css` (CSS import) |
| Components.json setting | `"cssVariables": true` | `"cssVariables": true` |
| Where `:root` and `.dark` live | inside `@layer base` | top level, outside any `@layer` |

ALWAYS verify the project's Tailwind generation by checking `package.json` for `"tailwindcss": "^4..."` versus `"tailwindcss": "^3..."`. NEVER infer from filenames.

### The bg-background semantic trap

ALWAYS understand : `bg-background` is NOT a built-in Tailwind utility. It exists ONLY when one of the following is true :

- (v3) `tailwind.config.js` has `theme.extend.colors.background = "hsl(var(--background))"`
- (v4) `globals.css` has `@theme inline { --color-background: var(--background); }`

NEVER write `bg-background` in a project that has neither wiring : the class silently produces no style and AI assistants will repeatedly suggest it. Diagnose missing utility by inspecting the generated CSS, NEVER by guessing.

### Five invariants

1. ALWAYS write CSS variable values in the format dictated by the project's Tailwind generation. HSL space-separated for v3 (no commas, no `hsl()` wrapper inside the variable). oklch for v4.
2. ALWAYS define `.dark` as a class selector overriding the SAME variables. NEVER use `@media (prefers-color-scheme: dark)` directly in shadcn projects; the toggle uses a class.
3. ALWAYS map raw variables into Tailwind's color namespace : via `tailwind.config.js` in v3, via `@theme inline` in v4. The variables alone do nothing for Tailwind utilities until they are mapped.
4. ALWAYS put `suppressHydrationWarning` on the `<html>` element when using next-themes in Next.js, and add the `attribute="class"` prop on the `ThemeProvider`. Omitting either produces flash-of-unstyled-content and hydration warnings.
5. ALWAYS use `cn()` (from `@/lib/utils`) for any conditional class composition that touches theme classes. NEVER concatenate strings with `+` or template literals.

## Token Catalog

Exhaustive list of CSS custom properties shadcn ships in the default style, verified at https://ui.shadcn.com/docs/theming on 2026-05-19. Full type, purpose, and per-mode values in `references/methods.md`.

### Core tokens (19 variables)

`--background`, `--foreground`, `--card`, `--card-foreground`, `--popover`, `--popover-foreground`, `--primary`, `--primary-foreground`, `--secondary`, `--secondary-foreground`, `--muted`, `--muted-foreground`, `--accent`, `--accent-foreground`, `--destructive`, `--border`, `--input`, `--ring`, `--radius`.

### Sidebar tokens (8 variables)

`--sidebar`, `--sidebar-foreground`, `--sidebar-primary`, `--sidebar-primary-foreground`, `--sidebar-accent`, `--sidebar-accent-foreground`, `--sidebar-border`, `--sidebar-ring`.

### Chart tokens (5 variables)

`--chart-1`, `--chart-2`, `--chart-3`, `--chart-4`, `--chart-5`.

### Radius scale (7 derived values, v4 only)

`--radius` is the base (default `0.625rem`). Tailwind v4 derives :

- `--radius-sm: calc(var(--radius) * 0.6)`
- `--radius-md: calc(var(--radius) * 0.8)`
- `--radius-lg: var(--radius)`
- `--radius-xl: calc(var(--radius) * 1.4)`
- `--radius-2xl: calc(var(--radius) * 1.8)`
- `--radius-3xl: calc(var(--radius) * 2.2)`
- `--radius-4xl: calc(var(--radius) * 2.6)`

ALWAYS expose these via `@theme inline { --radius-sm: ...; --radius-md: ...; }` so utilities like `rounded-md` resolve. NEVER hardcode `rounded-[6px]`; respect the token.

### `--destructive-foreground` is NOT in the default catalog

ALWAYS pair color-foreground with its base (`--card` + `--card-foreground`, `--primary` + `--primary-foreground`). The default style ships `--destructive` WITHOUT a paired `--destructive-foreground`. Components that need destructive text use `--foreground` or a hardcoded white. Custom themes MAY add `--destructive-foreground` but it is not a default token. Verified at https://ui.shadcn.com/docs/theming 2026-05-19.

## Decision Tree 1 : Setting up theming from scratch

```
Q1. Is the project Tailwind v4?
    yes → Q2 (v4 path)
    no  → Q3 (v3 path)

Q2 (v4). globals.css MUST contain in this order :
    1. @import "tailwindcss";
    2. @import "tw-animate-css";
    3. @custom-variant dark (&:is(.dark *));
    4. :root { --background: oklch(...); ... } (oklch values, no @layer wrapper)
    5. .dark { --background: oklch(...); ... } (oklch values, no @layer wrapper)
    6. @theme inline { --color-background: var(--background); ... --radius-sm: calc(...); ... }
    7. @layer base { * { @apply border-border outline-ring/50; } body { @apply bg-background text-foreground; } }
    NO tailwind.config.js needed for colors. END.

Q3 (v3). Two files MUST be wired :
    file 1 : globals.css
      @tailwind base; @tailwind components; @tailwind utilities;
      @layer base {
        :root { --background: 0 0% 100%; ... } (HSL space-separated, NO hsl() wrapper, NO commas)
        .dark { --background: 0 0% 3.9%; ... }
      }
    file 2 : tailwind.config.js
      module.exports = {
        darkMode: ["class"],
        theme: { extend: { colors: { background: "hsl(var(--background))", ... }, borderRadius: { lg: "var(--radius)", ... } } },
        plugins: [require("tailwindcss-animate")],
      }
    END.
```

## Decision Tree 2 : Adding dark mode toggle

```
Q1. Is the project Next.js (App Router or Pages Router)?
    yes → ALWAYS use next-themes :
        1. npm i next-themes
        2. components/theme-provider.tsx :
             "use client"; export const ThemeProvider = (props) =>
               <NextThemesProvider {...props}>{props.children}</NextThemesProvider>
        3. app/layout.tsx :
             <html lang="en" suppressHydrationWarning>
               <body>
                 <ThemeProvider attribute="class" defaultTheme="system" enableSystem disableTransitionOnChange>
                   {children}
                 </ThemeProvider>
               </body>
             </html>
        4. components/mode-toggle.tsx : useTheme() + DropdownMenu with light/dark/system
        END.
    no  → Q2

Q2. Is the project Vite + React?
    yes → ALWAYS use the official shadcn Vite ThemeProvider (custom Context, localStorage,
          NOT next-themes). Storage key default "vite-ui-theme". Identical mode-toggle.
          END.
    no  → Astro / Remix / TanStack Start each have a dedicated page :
          https://ui.shadcn.com/docs/dark-mode/{astro|remix|tanstack-start}
          ALWAYS follow the framework-specific snippet ; the wrappers differ.
```

## Decision Tree 3 : Pasting a theme from https://ui.shadcn.com/themes

```
Q1. The theme builder outputs CSS for the project's Tailwind generation. Project on v4?
    yes → copy the oklch block from the theme builder, paste into globals.css
          replacing the existing :root and .dark blocks. KEEP the existing
          @theme inline mapping and @custom-variant dark line; the builder
          outputs only the variable values.
    no  → switch the theme builder's "format" toggle to HSL, then paste
          the HSL block into globals.css inside @layer base.
          KEEP the existing tailwind.config.js color mappings.
```

## Patterns

### Pattern 1 : The v3 wiring (HSL space-separated)

The historical (Tailwind v3.4) wiring uses HSL components stored as space-separated triples in CSS variables, then wrapped by `hsl()` at the consumption site (`tailwind.config.js`). The wrapper convention is the source of >40% of AI-generated theming bugs (verified anti-pattern, see `references/anti-patterns.md`).

ALWAYS in v3 :
- Store : `--background: 0 0% 100%;` (three components, two spaces, no commas, no `hsl()`)
- Map  : `background: "hsl(var(--background))"` inside `tailwind.config.js`
- Use  : `bg-background` (now resolves)

NEVER in v3 :
- `--background: hsl(0, 0%, 100%);` (commas + wrapper inside the variable)
- `--background: 0, 0%, 100%;` (commas)
- `background: "var(--background)"` (missing `hsl()` wrapper at the mapping site)

Full v3 `globals.css` and `tailwind.config.js` in `references/examples.md`.

### Pattern 2 : The v4 wiring (oklch + @theme inline)

Tailwind v4 abandons `tailwind.config.js` for color theming and consumes variables directly via the `@theme inline` directive. The variables hold raw `oklch()` strings, the directive maps them into Tailwind's color namespace, and utilities like `bg-background` resolve automatically.

ALWAYS in v4 :
- Store : `--background: oklch(1 0 0);` (full `oklch()` function call)
- Map  : `@theme inline { --color-background: var(--background); }`
- Use  : `bg-background` (resolves via the `@theme inline` mapping)

NEVER in v4 :
- `--background: 0 0% 100%;` (HSL legacy format; the inline mapping expects a color value)
- `bg-[hsl(var(--background))]` (the wrapper convention from v3; works but defeats the point)
- A `tailwind.config.js` `colors` block (v4 ignores it for the new theming model)

Full v4 `globals.css` in `references/examples.md`.

### Pattern 3 : The `.dark` class strategy

shadcn ships a class-based dark mode by default. The toggle (next-themes on Next.js, custom Context on Vite) adds or removes the `.dark` class on `<html>`. Both `:root` and `.dark` declare the SAME variable names with different values; cascade picks the closer match.

ALWAYS in v4 declare the variant : `@custom-variant dark (&:is(.dark *));` (this is the v4 replacement for `darkMode: ["class"]`).

NEVER use `@media (prefers-color-scheme: dark)` for the variable overrides : it breaks the user-explicit toggle and produces inconsistency between system preference and explicit choice. next-themes already handles system mode by translating it into the class.

### Pattern 4 : The next-themes ThemeProvider (Next.js)

The shadcn-distributed `components/theme-provider.tsx` is a four-line `"use client"` wrapper around `next-themes`'s `ThemeProvider`. Verified verbatim at https://ui.shadcn.com/docs/dark-mode/next on 2026-05-19 :

```tsx
"use client"
import * as React from "react"
import { ThemeProvider as NextThemesProvider } from "next-themes"

export function ThemeProvider({
  children,
  ...props
}: React.ComponentProps<typeof NextThemesProvider>) {
  return <NextThemesProvider {...props}>{children}</NextThemesProvider>
}
```

Applied in `app/layout.tsx` :

```tsx
<html lang="en" suppressHydrationWarning>
  <body>
    <ThemeProvider attribute="class" defaultTheme="system" enableSystem disableTransitionOnChange>
      {children}
    </ThemeProvider>
  </body>
</html>
```

ALWAYS pass all four props : `attribute="class"`, `defaultTheme="system"`, `enableSystem`, `disableTransitionOnChange`. ALWAYS keep `suppressHydrationWarning` on `<html>`. Removing any of these reintroduces hydration mismatch warnings or flash-of-unstyled-content. See `references/anti-patterns.md` for failure modes.

### Pattern 5 : The Vite ThemeProvider (custom Context, NOT next-themes)

The Vite distribution uses a custom Context-based provider (no `next-themes` dependency). The storage key default is `"vite-ui-theme"`. Functionally identical : light / dark / system, `useTheme()` hook, identical mode-toggle component. Full source in `references/examples.md`.

ALWAYS confirm the framework before installing : a Vite project that adds `next-themes` will work but adds a runtime dependency the shadcn docs do not prescribe. NEVER mix the two providers in the same project.

### Pattern 6 : The mode-toggle component

The shadcn-distributed mode-toggle is a `DropdownMenu` with three items (Light / Dark / System) reading `useTheme()` and calling `setTheme(...)`. Sun and Moon icons from `lucide-react` cross-fade with conditional Tailwind classes. ALWAYS compose its className with `cn()` ; NEVER concatenate.

Full mode-toggle component in `references/examples.md`.

### Pattern 7 : Custom palette overrides

The recommended flow for a custom palette is :

1. Open https://ui.shadcn.com/themes
2. Pick a base color (Zinc / Slate / Stone / Gray / Neutral / Red / Rose / Orange / Green / Blue / Yellow / Violet)
3. Tune the radius (`0.0rem` / `0.25rem` / `0.5rem` / `0.75rem` / `1.0rem`)
4. Copy the generated CSS block
5. Paste into the project's `globals.css`, replacing `:root` and `.dark` blocks ONLY

ALWAYS keep wiring intact (the `@theme inline` block in v4, the `tailwind.config.js` color block in v3). The theme builder outputs only variable values, NEVER the mapping layer. NEVER hand-edit individual oklch values without re-running the contrast check at the theme builder ; the foreground / background pairs are tuned for WCAG-AA.

## Common Pitfalls (full details in references/anti-patterns.md)

NEVER do these. Each is verified from a published anti-pattern in the field :

1. NEVER use commas inside HSL variable values (`0, 0%, 100%`). The wrapper `hsl(var(--background))` breaks silently; the output utility renders the literal string.
2. NEVER omit the `hsl()` wrapper in the v3 `tailwind.config.js` mapping. `background: "var(--background)"` produces an invalid color and Tailwind silently drops the utility.
3. NEVER mix v3 and v4 wiring : either `tailwind.config.js` colors OR `@theme inline`, never both. v4 ignores the JS config for theming; v3 ignores the directive.
4. NEVER store oklch values in a v3 project's variables; the `hsl(var(--background))` wrapper at the mapping site will produce `hsl(oklch(1 0 0))`, which is invalid.
5. NEVER omit `suppressHydrationWarning` from `<html>` when using next-themes. React 18+ logs hydration mismatch warnings; first paint shows the wrong theme.
6. NEVER apply `@media (prefers-color-scheme: dark)` to override variables in a class-based shadcn setup. The class toggle and the media query conflict; the result depends on cascade order and confuses every reader.

## Reference Links

- [references/methods.md](references/methods.md) : exhaustive CSS-variable catalog with type, purpose, light and dark defaults
- [references/examples.md](references/examples.md) : full `globals.css` for v3 and v4, both `ThemeProvider` files, the `mode-toggle` component, and a custom-palette override example
- [references/anti-patterns.md](references/anti-patterns.md) : six canonical anti-patterns with WHY each fails and the fix

## Companion Skills

- [shadcn-core-architecture](../shadcn-core-architecture/SKILL.md) : copy-not-install paradigm and the ownership doctrine
- [shadcn-core-stack](../shadcn-core-stack/SKILL.md) : Radix + cva + tailwind-merge composition (which utilities the tokens drive)
- [shadcn-core-cli](../shadcn-core-cli/SKILL.md) : `init` flags (`--css-variables` is theming-relevant), `apply --only theme`
- [shadcn-core-blocks](../shadcn-core-blocks/SKILL.md) : style enum (`default` / `new-york` / `sera` / `luma`) impact on default tokens
- [shadcn-impl-theming-custom](../../shadcn-impl/shadcn-impl-theming-custom/SKILL.md) : end-to-end custom-theme workflow
- [shadcn-errors-tailwind-v3-v4-migration](../../shadcn-errors/shadcn-errors-tailwind-v3-v4-migration/SKILL.md) : full migration path including the codemod

## Sources

All claims in this skill trace to URLs in `SOURCES.md` :

- https://ui.shadcn.com/docs/theming (full token catalog, oklch defaults, sidebar + chart tokens, radius scale)
- https://ui.shadcn.com/docs/dark-mode (framework matrix)
- https://ui.shadcn.com/docs/dark-mode/next (next-themes wrapper code, layout integration)
- https://ui.shadcn.com/docs/dark-mode/vite (custom Context wrapper, storage key)
- https://ui.shadcn.com/docs/tailwind-v4 (v3 vs v4 wiring differences, `@theme inline`, `tw-animate-css`)
- https://ui.shadcn.com/themes (theme builder, style enum)

Verified 2026-05-19.
