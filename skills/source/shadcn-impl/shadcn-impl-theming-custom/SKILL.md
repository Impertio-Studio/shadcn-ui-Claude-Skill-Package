---
name: shadcn-impl-theming-custom
description: >
  Use when building or customizing a theme in a shadcn ui project,
  wiring the dark mode toggle, replacing the default primary color
  with a brand color, copying the output of the ui.shadcn.com themes
  builder into globals.css, or overriding the styling of one specific
  component instance without touching the global tokens. This is the
  end-to-end workflow recipe : the token catalog, oklch versus HSL
  format split, and v3 versus v4 wiring rules live in shadcn-core-theming.
  Prevents the flash-of-unstyled-content on first paint that comes from
  a missing suppressHydrationWarning, the silent "bg-primary did not
  change" symptom that comes from editing tokens without re-running
  the dev server, the two-ThemeProvider double-mount that breaks the
  toggle, the next-themes attribute mismatch that disables Tailwind's
  dark variants, and the brand-color contrast trap that ships an
  inaccessible palette.
  Covers the three-step theme-builder workflow (pick base style and
  radius, pick primary, copy CSS), the Tailwind v3 globals.css plus
  tailwind.config.js paste-target, the Tailwind v4 single-file
  globals.css paste-target with the @theme inline mapping, the
  Next.js ThemeProvider mount on app/layout.tsx with the four required
  props, the Vite custom Context ThemeProvider mount on main.tsx, the
  ModeToggle component with light/dark/system items, the brand-color
  override pattern keeping the foreground pair in sync, the custom
  palette build path with WCAG-AA verification, and the per-component
  override via the data-slot attribute that ships on every v4
  primitive.
  Keywords: custom theme shadcn, brand theme, brand color shadcn,
  dark mode toggle, dark mode shadcn, next-themes, theme provider,
  ThemeProvider, useTheme, mode-toggle component, ModeToggle, theme
  builder, ui.shadcn.com themes, how do I customize colors, how do I
  change the primary color, how do I add dark mode, theme flash
  unstyled content, FOUC, flash of unstyled content, hydration
  mismatch theme, suppressHydrationWarning, shadcn brand color,
  custom palette, per-component override shadcn, data-slot,
  data-slot card, oklch theme, HSL theme, paste theme CSS, globals.css
  theme, replace primary color, override token shadcn,
  ThemeProvider attribute class, system theme preference, light dark
  system toggle, useTheme hook, setTheme function, dark variant not
  working, my theme did not change, theme not applying, lucide Sun
  Moon icons toggle, DropdownMenu mode toggle, double ThemeProvider
  nested provider error.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui : Theming Custom (Workflow Recipe)

End-to-end recipe for building, pasting, and wiring a custom theme. This skill is the workflow ; the token mental model (oklch versus HSL, v3 versus v4 wiring, full token catalog, decision trees) lives in `shadcn-core-theming`. ALWAYS read that skill first when in doubt about a token format or wiring location.

## Quick Reference : The three-step happy path

The shortest path from "we want a custom theme" to a working dark-mode-toggling app :

1. **Pick** : open https://ui.shadcn.com/themes, pick a base style (`default`, `new-york`, `sera`, `luma`), pick a radius (`0.0rem`, `0.25rem`, `0.5rem`, `0.75rem`, `1.0rem`), pick a primary color (Zinc, Slate, Stone, Gray, Neutral, Red, Rose, Orange, Green, Blue, Yellow, Violet).
2. **Copy** : click "Copy code". The builder emits a single CSS block containing `:root { ... }` and `.dark { ... }` blocks, formatted for the project's Tailwind generation (oklch for v4, HSL space-separated for v3).
3. **Paste** : open `app/globals.css` (Next.js) or `src/index.css` (Vite). Replace the existing `:root { ... }` and `.dark { ... }` blocks ONLY. Keep the `@import "tailwindcss";` line, the `@custom-variant dark` line, the `@theme inline { ... }` block, and the `@layer base { ... }` block intact.

That is the ENTIRE happy path. If the project already has the `ThemeProvider` mounted, the new colors take effect on the next dev-server reload. If not, follow the framework-specific mount recipe below.

### Five invariants

1. ALWAYS paste only the `:root` and `.dark` blocks from the theme builder ; NEVER paste the `@theme inline` mapping (it already lives in the project).
2. ALWAYS verify the builder output matches the project's Tailwind generation (oklch for v4, HSL space-separated for v3) BEFORE pasting. Wrong format = silent zero-style output.
3. ALWAYS mount exactly ONE `ThemeProvider` at the root layout. Nesting two providers (e.g. next-themes inside a custom Context) produces unpredictable toggle behavior.
4. ALWAYS pass `attribute="class"` to next-themes ; any other value (e.g. `attribute="data-theme"`) disables Tailwind's `dark:` variant in shadcn projects because the `.dark` selector targets a class, not an attribute.
5. ALWAYS put `suppressHydrationWarning` on `<html>` when using next-themes in Next.js. Omitting it produces a hydration mismatch warning and a first-paint flash of the wrong theme.

## Decision Tree 1 : Which workflow do I need?

```
Q1. Goal = "use a built-in palette" (the theme builder output is enough)?
    yes -> Workflow A : theme-builder paste (below)
    no  -> Q2

Q2. Goal = "override one specific token" (e.g. brand primary, brand radius)?
    yes -> Workflow B : surgical token override (below)
    no  -> Q3

Q3. Goal = "build a custom palette from a brand color"?
    yes -> Workflow C : custom palette build (below)
    no  -> Q4

Q4. Goal = "style one component instance differently"?
    yes -> Workflow D : per-component override via data-slot (below)
    no  -> NOT a theming task. Check shadcn-core-theming for token semantics
           or shadcn-impl-component-install for adding components.
```

## Workflow A : Theme-builder paste

The fastest custom theme. Total time : approximately 60 seconds.

```
1. Navigate to https://ui.shadcn.com/themes.
2. In the right panel : pick Style (default / new-york / sera / luma).
3. In the right panel : pick Color (Zinc / Slate / Stone / Gray / Neutral /
   Red / Rose / Orange / Green / Blue / Yellow / Violet).
4. In the right panel : pick Radius (0.0 / 0.25 / 0.5 / 0.75 / 1.0).
5. Click "Copy code" at the top-right of the right panel.
6. Open the project's main CSS file :
   - Next.js : app/globals.css (or src/app/globals.css in monorepos)
   - Vite + React : src/index.css
   - Astro / Remix : project-specific ; usually src/styles/globals.css
7. Locate the existing :root { ... } and .dark { ... } blocks.
8. Replace them with the pasted blocks ONLY.
9. Save. The dev server hot-reloads ; the new theme is live.
```

ALWAYS preserve everything ELSE in `globals.css` : the `@import "tailwindcss";` line, the `@import "tw-animate-css";` line, the `@custom-variant dark (&:is(.dark *));` line, the `@theme inline { --color-background: var(--background); ... }` mapping block, and the `@layer base { ... }` block. The builder output is variable VALUES only ; the wiring layer stays unchanged. Full `globals.css` reference in `references/examples.md`.

### Format-verification check

Before pasting, glance at the first variable line. If it reads `--background: oklch(1 0 0);` the format is v4. If it reads `--background: 0 0% 100%;` the format is v3. Cross-check against the project : `package.json` `"tailwindcss": "^4..."` is v4 ; `"^3..."` is v3. NEVER paste v4 (oklch) values into a v3 project ; the `hsl(var(--background))` wrapper at the mapping site produces `hsl(oklch(...))`, which is invalid and silently drops the utility.

The theme builder has a format toggle in the right panel for projects still on v3. Switch it BEFORE clicking "Copy code".

## Workflow B : Surgical token override

When the goal is "use the default shadcn theme but change ONE thing" (e.g. corporate brand primary), edit ONLY the relevant variables in `:root` and `.dark`. Both blocks MUST be updated in lockstep.

### Brand-primary override (v4 example)

```css
:root {
  /* ... all other shadcn defaults unchanged ... */
  --primary: oklch(0.488 0.243 264.376);            /* brand purple */
  --primary-foreground: oklch(0.985 0 0);            /* near-white text */
}

.dark {
  /* ... all other shadcn defaults unchanged ... */
  --primary: oklch(0.692 0.180 270.000);            /* lighter brand purple for dark mode */
  --primary-foreground: oklch(0.205 0 0);            /* near-black text */
}
```

ALWAYS update BOTH `:root` AND `.dark`. The default theme tunes them for opposite contrast ; overriding only `:root` leaves `.dark` showing the original shadcn primary, producing a brand-mismatch in dark mode.

ALWAYS keep `--primary-foreground` in sync. The pair is contrast-tuned. A brand-purple primary with a `--primary-foreground` of pure black fails WCAG-AA. Run the chosen pair through https://webaim.org/resources/contrastchecker/ before shipping.

### Brand-radius override

```css
:root {
  --radius: 0.75rem;  /* override base radius */
}
```

The `@theme inline` block derives `--radius-sm` through `--radius-4xl` from `--radius` ; no per-step override needed. NEVER hardcode `rounded-[12px]` in components ; respect the token.

## Workflow C : Custom palette build

When the theme builder palettes do not match the brand, build from scratch.

```
1. Identify the brand primary color in any color space (hex / RGB / HSL).
2. Convert to oklch (v4) or HSL space-separated (v3). See references/methods.md
   for conversion guidance.
3. Derive the foreground pair : near-white if the primary is dark,
   near-black if the primary is light. Target WCAG-AA contrast.
4. Optionally derive accent, secondary, muted, destructive from the
   brand palette. ALWAYS keep the foreground pair in sync per token.
5. Write both :root { ... } and .dark { ... } blocks. For each token,
   the .dark value is typically lighter than :root for surface tokens
   (background, card, popover, muted) and inverted for the foreground.
6. Paste into globals.css per Workflow A step 6+.
```

ALWAYS verify contrast for every foreground-pair token (`primary` + `primary-foreground`, `secondary` + `secondary-foreground`, etc.) in BOTH `:root` and `.dark`. A 5-pair palette is a 20-check matrix. Full palette template in `references/examples.md`.

ALWAYS start with the default shadcn palette as a baseline ; modify the 3-5 tokens that need to change. NEVER rewrite all 19 core tokens from scratch unless the brand mandates it ; the default is contrast-tuned.

## Workflow D : Per-component override via data-slot

Every shadcn primitive in v4 ships with a `data-slot` attribute on each subcomponent (e.g. Card has `data-slot="card"`, CardHeader has `data-slot="card-header"`, AccordionTrigger has `data-slot="accordion-trigger"`). This is the canonical hook for styling ONE specific component without touching the global tokens.

### Pattern : Tailwind arbitrary variant

```tsx
// Override Card padding for a specific instance using a wrapper class
<div className="[&_[data-slot=card]]:p-8 [&_[data-slot=card-header]]:pb-2">
  <Card>
    <CardHeader>...</CardHeader>
    <CardContent>...</CardContent>
  </Card>
</div>
```

### Pattern : Global CSS-selector override

```css
/* In globals.css, scoped to a wrapper class */
.brand-pricing [data-slot=card] {
  border-width: 2px;
  border-color: var(--primary);
}
```

ALWAYS prefer `data-slot` selectors over component className overrides when the goal is a project-wide tweak to a specific subcomponent. The `data-slot` attribute is stable across `shadcn add --overwrite` cycles ; an internal className like `rounded-xl` may shift between versions.

ALWAYS scope the override under a wrapper class (`.brand-pricing`, `[data-section=hero]`, etc.) to avoid affecting every Card in the app. A bare `[data-slot=card] { ... }` rule applies globally.

Full override examples in `references/examples.md`.

## Decision Tree 2 : Wiring the dark-mode toggle

```
Q1. Is the project Next.js (App Router or Pages Router)?
    yes -> Workflow E : next-themes mount (below)
    no  -> Q2

Q2. Is the project Vite + React?
    yes -> Workflow F : custom Context mount (below)
    no  -> Astro / Remix / TanStack Start : framework-specific page at
           https://ui.shadcn.com/docs/dark-mode/{astro|remix|tanstack-start}
```

## Workflow E : next-themes mount (Next.js)

Three files. Verified verbatim at https://ui.shadcn.com/docs/dark-mode/next on 2026-05-19.

### Step 1 : install

```bash
npm install next-themes
```

### Step 2 : create `components/theme-provider.tsx`

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

### Step 3 : mount in `app/layout.tsx`

```tsx
import { ThemeProvider } from "@/components/theme-provider"

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body>
        <ThemeProvider
          attribute="class"
          defaultTheme="system"
          enableSystem
          disableTransitionOnChange
        >
          {children}
        </ThemeProvider>
      </body>
    </html>
  )
}
```

ALWAYS pass all four props. ALWAYS keep `suppressHydrationWarning` on `<html>`. ALWAYS put `"use client"` at the top of `theme-provider.tsx` ; next-themes reads `localStorage` and `window.matchMedia`, both client-only APIs.

Full prop semantics and useTheme signature in `references/methods.md`.

## Workflow F : Custom Context mount (Vite)

The Vite distribution uses a custom React Context provider, NOT next-themes. Storage key default `"vite-ui-theme"`. Full source verbatim at https://ui.shadcn.com/docs/dark-mode/vite on 2026-05-19.

### Step 1 : create `src/components/theme-provider.tsx`

See full source code in `references/examples.md`. The provider reads `localStorage`, applies the `.light` or `.dark` class to `document.documentElement`, and exports a `useTheme()` hook.

### Step 2 : mount in `src/main.tsx`

```tsx
import { ThemeProvider } from "@/components/theme-provider"

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode>
    <ThemeProvider defaultTheme="system" storageKey="vite-ui-theme">
      <App />
    </ThemeProvider>
  </React.StrictMode>
)
```

ALWAYS pick a stable `storageKey` per app. NEVER share storage keys across apps on the same domain ; the toggle would propagate globally and confuse users.

NEVER install `next-themes` in a Vite project. It works but adds an unnecessary runtime dependency and breaks parity with the official shadcn docs.

## Pattern : The ModeToggle component

A single DropdownMenu with three items (Light / Dark / System) reading `useTheme()` and calling `setTheme(...)`. Sun and Moon icons from `lucide-react` cross-fade via `dark:` Tailwind variants.

```tsx
"use client"  // only needed in Next.js

import { Moon, Sun } from "lucide-react"
import { useTheme } from "next-themes"  // or "@/components/theme-provider" in Vite
import { Button } from "@/components/ui/button"
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuTrigger,
} from "@/components/ui/dropdown-menu"

export function ModeToggle() {
  const { setTheme } = useTheme()

  return (
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <Button variant="outline" size="icon">
          <Sun className="h-[1.2rem] w-[1.2rem] scale-100 rotate-0 transition-all dark:scale-0 dark:-rotate-90" />
          <Moon className="absolute h-[1.2rem] w-[1.2rem] scale-0 rotate-90 transition-all dark:scale-100 dark:rotate-0" />
          <span className="sr-only">Toggle theme</span>
        </Button>
      </DropdownMenuTrigger>
      <DropdownMenuContent align="end">
        <DropdownMenuItem onClick={() => setTheme("light")}>Light</DropdownMenuItem>
        <DropdownMenuItem onClick={() => setTheme("dark")}>Dark</DropdownMenuItem>
        <DropdownMenuItem onClick={() => setTheme("system")}>System</DropdownMenuItem>
      </DropdownMenuContent>
    </DropdownMenu>
  )
}
```

ALWAYS import `useTheme` from `next-themes` in Next.js projects, from `@/components/theme-provider` in Vite projects. They have the same surface (`{ theme, setTheme, ... }`) but are different modules.

ALWAYS include the System item ; defaulting to system preference is the documented shadcn behavior and respects OS-level user choice.

ALWAYS add `"use client"` at the top in Next.js : the component calls `useTheme()` which reads client-only state.

## Common Pitfalls (full details in references/anti-patterns.md)

NEVER do these. Each is verified from documented anti-patterns and field experience :

1. NEVER ship a Next.js app without `suppressHydrationWarning` on `<html>`. The server renders without the theme class ; the client adds it ; React 18+ logs a hydration mismatch and the first paint flashes the wrong theme.
2. NEVER write HSL comma-separated values in a v3 project (`--background: 0, 0%, 100%`). The `hsl(var(--background))` wrapper breaks silently ; the utility renders the literal string.
3. NEVER paste oklch values into a v3 project. The wrapper produces `hsl(oklch(...))` which is invalid CSS and the utility is dropped.
4. NEVER pass `attribute="data-theme"` to next-themes in a shadcn project. The default style ships `.dark` as a class selector ; `attribute="data-theme"` adds a `data-theme="dark"` attribute that the `.dark` selector and Tailwind's `dark:` variant ignore.
5. NEVER nest two `ThemeProvider`s. A common bug : adding a custom Context provider for app-state on top of next-themes, then nesting next-themes again at a layout boundary. The toggle becomes unpredictable.

## Reference Links

- [references/methods.md](references/methods.md) : next-themes ThemeProvider prop reference, useTheme hook signatures (Next.js and Vite), oklch and HSL conversion guidance, palette-generation matrix
- [references/examples.md](references/examples.md) : full v4 `globals.css` with light + dark + brand-primary override, Next.js `app/layout.tsx` ThemeProvider mount, Vite `App.tsx` + custom ThemeContext, ModeToggle component, per-component data-slot override
- [references/anti-patterns.md](references/anti-patterns.md) : five canonical anti-patterns with WHY each fails and the fix

## Companion Skills

- [shadcn-core-theming](../../shadcn-core/shadcn-core-theming/SKILL.md) : token catalog, v3 versus v4 wiring, oklch versus HSL format, decision trees (the mental model behind this workflow)
- [shadcn-syntax-button](../../shadcn-syntax/shadcn-syntax-button/SKILL.md) : Button variants used by the ModeToggle trigger
- [shadcn-impl-component-install](../shadcn-impl-component-install/SKILL.md) : adding the DropdownMenu and Button needed for the ModeToggle
- [shadcn-errors-tailwind-v3-v4-migration](../../shadcn-errors/shadcn-errors-tailwind-v3-v4-migration/SKILL.md) : full migration path when format-verification fails

## Sources

All claims in this skill trace to URLs in `SOURCES.md` :

- https://ui.shadcn.com/themes (theme builder, style enum, color enum, radius enum, copy-CSS workflow)
- https://ui.shadcn.com/docs/theming (token catalog, @theme inline mapping, custom palette workflow)
- https://ui.shadcn.com/docs/dark-mode/next (next-themes ThemeProvider verbatim source, layout integration, suppressHydrationWarning requirement)
- https://ui.shadcn.com/docs/dark-mode/vite (custom Context ThemeProvider verbatim source, storage key default)
- https://github.com/shadcn-ui/ui (data-slot attribute on every v4 primitive subcomponent)

Verified 2026-05-19.
