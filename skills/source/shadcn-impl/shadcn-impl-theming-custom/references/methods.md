# References : Methods (theming-custom)

API surfaces touched by the custom-theming workflow. The token catalog itself lives in `shadcn-core-theming/references/methods.md` ; this file documents the wiring layer only (ThemeProvider props, useTheme hook, override patterns).

## next-themes : ThemeProvider props

Verified at https://ui.shadcn.com/docs/dark-mode/next and https://github.com/pacocoursey/next-themes (README) on 2026-05-19.

The shadcn-distributed `components/theme-provider.tsx` is a thin `"use client"` wrapper around `next-themes`'s `ThemeProvider`. The full prop surface forwarded :

| Prop | Type | Default | Purpose in shadcn projects |
|------|------|---------|----------------------------|
| `attribute` | `"class"` \| `"data-theme"` \| `string` | `"data-theme"` (next-themes default) | ALWAYS pass `"class"` in shadcn projects. The default style ships `.dark` as a class selector ; any other value disables Tailwind's `dark:` variant and the shadcn `.dark` cascade. |
| `defaultTheme` | `"light"` \| `"dark"` \| `"system"` \| custom | `"system"` (next-themes default) | ALWAYS `"system"` per shadcn docs ; respects OS preference until user makes an explicit choice. |
| `enableSystem` | `boolean` | `true` (next-themes default) | ALWAYS `true`. Disables the System item in the toggle if false. |
| `disableTransitionOnChange` | `boolean` | `false` (next-themes default) | ALWAYS `true` per shadcn docs ; prevents CSS transition flicker during theme switches. |
| `storageKey` | `string` | `"theme"` (next-themes default) | OPTIONAL. Per-app key to avoid cross-app collision when multiple apps share a domain. |
| `themes` | `string[]` | `["light", "dark"]` (when `enableSystem` is true, system is implicit) | OPTIONAL. For multi-theme schemes (e.g. light / dark / sepia / high-contrast). |
| `forcedTheme` | `string` | `undefined` | OPTIONAL. Pins one route to a fixed theme regardless of user preference (e.g. printable pages). |
| `value` | `Record<string, string>` | `undefined` | OPTIONAL. Maps custom theme names to CSS class names (e.g. `{ pastel: "theme-pastel" }`). Rarely needed in shadcn ; the default class names match the `.dark` and `.light` selectors. |

### Minimum prop set (mandatory in shadcn projects)

```tsx
<ThemeProvider
  attribute="class"
  defaultTheme="system"
  enableSystem
  disableTransitionOnChange
>
```

ALWAYS pass all four. Omitting any of the four reintroduces a documented bug (hydration mismatch, dark variants broken, transition flicker, system mode disabled).

## next-themes : useTheme hook

```ts
import { useTheme } from "next-themes"

const {
  theme,         // current explicit user choice : "light" | "dark" | "system"
  setTheme,      // (theme: string) => void : sets the new theme, persists to storage
  resolvedTheme, // resolved theme : "light" | "dark" (system collapsed to actual value)
  systemTheme,   // current OS preference : "light" | "dark"
  themes,        // array of all configured theme names
  forcedTheme,   // forced theme name if forcedTheme prop is set, else undefined
} = useTheme()
```

ALWAYS use `theme` for UI state (which radio is selected in the toggle). ALWAYS use `resolvedTheme` for theme-conditional logic (which logo asset to load). NEVER use `theme` for conditional rendering ; it returns `"system"` and gives no information about the actually-applied theme.

ALWAYS guard against SSR hydration : `useTheme()` returns `theme: undefined` on the server. Display a placeholder or skeleton until `mounted` is true :

```tsx
const [mounted, setMounted] = React.useState(false)
React.useEffect(() => setMounted(true), [])
if (!mounted) return null  // or a skeleton
```

This avoids a hydration mismatch on the toggle button itself.

## Vite custom Context : ThemeProvider props

The Vite distribution at https://ui.shadcn.com/docs/dark-mode/vite (verified 2026-05-19) ships a different surface :

| Prop | Type | Default | Purpose |
|------|------|---------|---------|
| `defaultTheme` | `"light"` \| `"dark"` \| `"system"` | `"system"` | Initial theme if `localStorage` is empty. |
| `storageKey` | `string` | `"vite-ui-theme"` | `localStorage` key for persistence. |
| `children` | `React.ReactNode` | required | Wrapped app. |

The Vite provider has NO `attribute`, `enableSystem`, or `disableTransitionOnChange` props. It applies the class directly via `document.documentElement.classList`. Behavior parity with next-themes is implicit, not configurable.

## Vite custom Context : useTheme hook

```ts
import { useTheme } from "@/components/theme-provider"

const {
  theme,    // current theme : "light" | "dark" | "system"
  setTheme, // (theme: "light" | "dark" | "system") => void
} = useTheme()
```

Smaller surface than next-themes : no `resolvedTheme`, no `systemTheme` exposed. The provider resolves system preference internally inside its `useEffect`. If the app needs `resolvedTheme`, derive it manually :

```ts
const resolvedTheme = theme === "system"
  ? (window.matchMedia("(prefers-color-scheme: dark)").matches ? "dark" : "light")
  : theme
```

ALWAYS throw if `useTheme()` is called outside the provider ; the official source does this. NEVER swallow the error.

## CSS variable override pattern

The general shape for overriding any token :

```css
/* In globals.css, scoped under :root and .dark */

:root {
  --token-name: <value matching project's Tailwind generation>;
}

.dark {
  --token-name: <value matching project's Tailwind generation>;
}
```

Reference for token names, defaults, and purposes : `shadcn-core-theming/references/methods.md`.

ALWAYS update BOTH `:root` AND `.dark` in lockstep. NEVER override only one ; the omitted mode falls back to the shadcn default and produces a brand-mismatch.

ALWAYS keep foreground pairs together. The token names ending in `-foreground` are contrast-tuned for their base. Examples : `--primary` + `--primary-foreground`, `--card` + `--card-foreground`, `--sidebar` + `--sidebar-foreground`.

NEVER add tokens to the variable block without also adding them to the `@theme inline` mapping (v4) or the `tailwind.config.js` color extension (v3). A token without a mapping produces no Tailwind utility.

## Adding a new token (e.g. `--warning`)

The full four-step path to add a brand-new token, e.g. `--warning` for warning banners :

### v4 path

```css
/* 1. Declare in :root and .dark */
:root {
  --warning: oklch(0.795 0.184 86.047);
  --warning-foreground: oklch(0.205 0 0);
}
.dark {
  --warning: oklch(0.795 0.184 86.047);
  --warning-foreground: oklch(0.205 0 0);
}

/* 2. Map in @theme inline */
@theme inline {
  --color-warning: var(--warning);
  --color-warning-foreground: var(--warning-foreground);
}

/* 3. Use in components */
/* <div className="bg-warning text-warning-foreground"> */
```

### v3 path

```css
/* 1. Declare in :root and .dark, HSL space-separated */
@layer base {
  :root {
    --warning: 38 92% 50%;
    --warning-foreground: 0 0% 13%;
  }
  .dark {
    --warning: 38 92% 50%;
    --warning-foreground: 0 0% 13%;
  }
}
```

```js
// 2. Map in tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: {
        warning: "hsl(var(--warning))",
        "warning-foreground": "hsl(var(--warning-foreground))",
      },
    },
  },
}
```

```tsx
// 3. Use in components
// <div className="bg-warning text-warning-foreground">
```

## Color-format conversion guidance

The theme builder outputs in the project's format ; manual palette work needs conversion when the brand color is given in a different space.

### HSL space-separated (Tailwind v3)

Format : `H S% L%` (three components, space-separated, no commas, no `hsl()` wrapper inside the variable).

Conversion from hex / RGB : use any color tool ; common picks are https://www.w3schools.com/colors/colors_picker.asp or `chrome://devtools` color tooltip.

```
#7c3aed (Tailwind violet-600)  ->  263 70% 58%
#ffffff (pure white)           ->  0 0% 100%
#000000 (pure black)           ->  0 0% 0%
```

NEVER write `--primary: hsl(263, 70%, 58%);` in v3. The wrapper at the consumption site (`hsl(var(--primary))`) double-wraps and produces invalid CSS.

### oklch (Tailwind v4)

Format : `oklch(L C H / alpha?)` where L is lightness `[0..1]`, C is chroma `[0..0.4]`, H is hue in degrees `[0..360]`, alpha is optional `[0..1]`.

Conversion : use https://oklch.com/ (interactive tool, accepts hex and outputs oklch). Tailwind v4 ships oklch values for the entire default palette ; pick the visually-closest preset and tweak from there.

```
#7c3aed (Tailwind violet-600)  ->  oklch(0.488 0.243 264.376)
#ffffff (pure white)           ->  oklch(1 0 0)
#000000 (pure black)           ->  oklch(0 0 0)
```

ALWAYS prefer oklch over hex / RGB in v4. The space is perceptually uniform : tweaking L by `0.05` produces a visually-uniform lightness change regardless of hue. This makes dark-mode derivation predictable.

## WCAG-AA contrast check

For every `--token` + `--token-foreground` pair :

1. Convert both values to hex (use https://oklch.com/ or browser devtools color tooltip).
2. Plug both hex codes into https://webaim.org/resources/contrastchecker/.
3. Verify contrast ratio is at least 4.5:1 for normal text, 3:1 for large text (18pt+ or 14pt+ bold).
4. If failing : adjust the foreground L (lightness) value by 0.05-0.10 until passing.

ALWAYS run the check for BOTH `:root` AND `.dark`. A pair that passes in light mode often fails in dark mode if naively inverted.

## ModeToggle component contract

The signature `useTheme()` returns is the only API the ModeToggle needs :

```tsx
const { setTheme } = useTheme()
// onClick handlers : setTheme("light"), setTheme("dark"), setTheme("system")
```

The ModeToggle does NOT need to read `theme` ; the visual state (Sun visible vs Moon visible) is driven by Tailwind's `dark:` variants, which respond to the `.dark` class on `<html>`. Reading `theme` in the toggle introduces a hydration-mismatch risk and is unnecessary.

ALWAYS write the icon cross-fade via `dark:` variants, NEVER via `theme === "dark" ? <Moon/> : <Sun/>` conditional rendering. The conditional approach hydrates with the server-rendered choice, then flips, producing a flicker.

## data-slot selector targeting

Every v4 shadcn primitive subcomponent ships with a stable `data-slot` attribute. The naming convention : kebab-case form of the component name. Examples :

| Component | data-slot value |
|-----------|-----------------|
| `Card` | `card` |
| `CardHeader` | `card-header` |
| `CardTitle` | `card-title` |
| `CardDescription` | `card-description` |
| `CardContent` | `card-content` |
| `CardFooter` | `card-footer` |
| `CardAction` | `card-action` |
| `AccordionTrigger` | `accordion-trigger` |
| `AccordionContent` | `accordion-content` |
| `DialogContent` | `dialog-content` |
| `Button` | `button` |

Verified against the shadcn-ui registry on 2026-05-19 (https://github.com/shadcn-ui/ui/tree/main/apps/v4/registry/new-york-v4/ui).

### Tailwind arbitrary-variant selector

```tsx
<div className="[&_[data-slot=card]]:border-2 [&_[data-slot=card]]:border-primary">
```

The `&_` descendant combinator targets ANY `data-slot=card` element inside the wrapper. ALWAYS scope under a wrapper ; the bare selector is a global override.

### Plain CSS selector

```css
.brand-section [data-slot=card] {
  border-width: 2px;
}
```

ALWAYS prefer `data-slot` selectors over component className overrides. The attribute is stable across `shadcn add --overwrite` cycles ; an internal className like `rounded-xl` may shift between versions and break the override silently.
