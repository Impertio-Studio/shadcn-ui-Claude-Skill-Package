# shadcn ui Theming : CSS Variable Catalog

Exhaustive catalog of every CSS custom property shadcn ships in the default style, with type, purpose, default light value, and default dark value. Values verified verbatim at https://ui.shadcn.com/docs/theming on 2026-05-19.

## Format clarification

The catalog below uses oklch as shipped in the current default style (Tailwind v4). For Tailwind v3 projects the SAME variable names hold space-separated HSL components (e.g., `--background: 0 0% 100%`) consumed via `hsl(var(--background))` in `tailwind.config.js`. See `examples.md` for the v3 block. NEVER mix formats inside a single project.

## Core tokens (19 variables)

| Variable | Type | Purpose | Default light | Default dark |
|----------|------|---------|---------------|--------------|
| `--background` | oklch color | Page background ; consumed by `bg-background` | `oklch(1 0 0)` | `oklch(0.145 0 0)` |
| `--foreground` | oklch color | Default text color ; consumed by `text-foreground` | `oklch(0.145 0 0)` | `oklch(0.985 0 0)` |
| `--card` | oklch color | Card surface background ; consumed by `bg-card` | `oklch(1 0 0)` | `oklch(0.205 0 0)` |
| `--card-foreground` | oklch color | Card text color ; consumed by `text-card-foreground` | `oklch(0.145 0 0)` | `oklch(0.985 0 0)` |
| `--popover` | oklch color | Popover / Tooltip / DropdownMenu surface | `oklch(1 0 0)` | `oklch(0.205 0 0)` |
| `--popover-foreground` | oklch color | Popover text color | `oklch(0.145 0 0)` | `oklch(0.985 0 0)` |
| `--primary` | oklch color | Primary action background (default Button variant) | `oklch(0.205 0 0)` | `oklch(0.922 0 0)` |
| `--primary-foreground` | oklch color | Primary action text | `oklch(0.985 0 0)` | `oklch(0.205 0 0)` |
| `--secondary` | oklch color | Secondary surface ; consumed by `bg-secondary` | `oklch(0.97 0 0)` | `oklch(0.269 0 0)` |
| `--secondary-foreground` | oklch color | Secondary text | `oklch(0.205 0 0)` | `oklch(0.985 0 0)` |
| `--muted` | oklch color | Muted surface (Skeleton, Input placeholder bg) | `oklch(0.97 0 0)` | `oklch(0.269 0 0)` |
| `--muted-foreground` | oklch color | Muted text (placeholder, helper text) | `oklch(0.556 0 0)` | `oklch(0.708 0 0)` |
| `--accent` | oklch color | Accent surface (hover states, NavigationMenu) | `oklch(0.97 0 0)` | `oklch(0.269 0 0)` |
| `--accent-foreground` | oklch color | Accent text | `oklch(0.205 0 0)` | `oklch(0.985 0 0)` |
| `--destructive` | oklch color | Destructive action (Delete Button, error Alert) | `oklch(0.577 0.245 27.325)` | `oklch(0.704 0.191 22.216)` |
| `--border` | oklch color | Default border color ; consumed by `border-border` | `oklch(0.922 0 0)` | `oklch(1 0 0 / 10%)` |
| `--input` | oklch color | Input border color ; consumed by `border-input` | `oklch(0.922 0 0)` | `oklch(1 0 0 / 15%)` |
| `--ring` | oklch color | Focus ring color ; consumed by `ring-ring` | `oklch(0.708 0 0)` | `oklch(0.556 0 0)` |
| `--radius` | length | Base border radius ; root of the radius scale | `0.625rem` | `0.625rem` |

Note : `--destructive-foreground` is NOT in the default style. Custom themes may add it ; components in the default style use foreground or hardcoded `text-white`.

## Sidebar tokens (8 variables)

Used exclusively by the `Sidebar` block. Decoupled from core tokens so a sidebar may differ from the page chrome.

| Variable | Type | Default light | Default dark |
|----------|------|---------------|--------------|
| `--sidebar` | oklch color | `oklch(0.985 0 0)` | `oklch(0.205 0 0)` |
| `--sidebar-foreground` | oklch color | `oklch(0.145 0 0)` | `oklch(0.985 0 0)` |
| `--sidebar-primary` | oklch color | `oklch(0.205 0 0)` | `oklch(0.488 0.243 264.376)` |
| `--sidebar-primary-foreground` | oklch color | `oklch(0.985 0 0)` | `oklch(0.985 0 0)` |
| `--sidebar-accent` | oklch color | `oklch(0.97 0 0)` | `oklch(0.269 0 0)` |
| `--sidebar-accent-foreground` | oklch color | `oklch(0.205 0 0)` | `oklch(0.985 0 0)` |
| `--sidebar-border` | oklch color | `oklch(0.922 0 0)` | `oklch(1 0 0 / 10%)` |
| `--sidebar-ring` | oklch color | `oklch(0.708 0 0)` | `oklch(0.556 0 0)` |

## Chart tokens (5 variables)

Used by Recharts wrappers (`Chart`, `ChartContainer`, `ChartTooltip`, etc.). The dark palette is deliberately different from the light palette (not just a value inversion) for visual contrast in dark mode.

| Variable | Type | Default light | Default dark |
|----------|------|---------------|--------------|
| `--chart-1` | oklch color | `oklch(0.646 0.222 41.116)` | `oklch(0.488 0.243 264.376)` |
| `--chart-2` | oklch color | `oklch(0.6 0.118 184.704)` | `oklch(0.696 0.17 162.48)` |
| `--chart-3` | oklch color | `oklch(0.398 0.07 227.392)` | `oklch(0.769 0.188 70.08)` |
| `--chart-4` | oklch color | `oklch(0.828 0.189 84.429)` | `oklch(0.627 0.265 303.9)` |
| `--chart-5` | oklch color | `oklch(0.769 0.188 70.08)` | `oklch(0.645 0.246 16.439)` |

ALWAYS consume chart tokens via `var(--chart-1)` directly inside the chart config object. NEVER reroute them through Tailwind utilities ; Recharts needs raw color strings.

## Radius scale (7 derived values, Tailwind v4)

The radius scale is derived from `--radius` at the `@theme inline` declaration site. All seven are exposed as Tailwind utility-resolvable tokens (`rounded-sm`, `rounded-md`, ...).

| Token | Derivation | At default `--radius: 0.625rem` |
|-------|-----------|--------------------------------|
| `--radius-sm` | `calc(var(--radius) * 0.6)` | `0.375rem` |
| `--radius-md` | `calc(var(--radius) * 0.8)` | `0.5rem` |
| `--radius-lg` | `var(--radius)` | `0.625rem` |
| `--radius-xl` | `calc(var(--radius) * 1.4)` | `0.875rem` |
| `--radius-2xl` | `calc(var(--radius) * 1.8)` | `1.125rem` |
| `--radius-3xl` | `calc(var(--radius) * 2.2)` | `1.375rem` |
| `--radius-4xl` | `calc(var(--radius) * 2.6)` | `1.625rem` |

For Tailwind v3, `tailwind.config.js` maps three values manually : `lg: "var(--radius)"`, `md: "calc(var(--radius) - 2px)"`, `sm: "calc(var(--radius) - 4px)"`. The derived `*-xl` through `*-4xl` are a v4-only convenience.

## Selector contract

The variables are declared inside two selectors that MUST be present together :

- `:root { ... }` : declares the light-mode values (also the default when no class is applied)
- `.dark { ... }` : overrides the SAME variable names with dark-mode values

ALWAYS keep the variable names identical between `:root` and `.dark`. NEVER add a new variable only in `.dark` ; the light-mode cascade will produce an unresolved variable in light mode.

## Components.json control

`components.json` carries one theming-relevant flag (verified at https://ui.shadcn.com/docs/components-json) :

```json
{
  "tailwind": {
    "cssVariables": true
  }
}
```

When `cssVariables` is `true` (the default), shadcn components emit utilities like `bg-background` that resolve through the variables documented above. When `false`, components are scaffolded with hardcoded utilities like `bg-zinc-950` directly. This flag is captured at `init` time and is immutable post-init ; switching requires re-running `shadcn add` for every affected component.

ALWAYS prefer `cssVariables: true` for any project that wants dark mode or runtime theming. NEVER toggle this flag in `components.json` after init ; the generated component files do not change retroactively.

## Companion API surface (where related APIs live)

This skill catalogs the values. The APIs that drive them live in companion skills :

- `useTheme()` hook + `setTheme()` writer : documented in the Pattern 4 / Pattern 5 examples in the parent `SKILL.md`. Comes from `next-themes` (Next.js) or the custom Context (Vite).
- `cn()` helper : sourced from `@/lib/utils`, scaffolded by `shadcn init`. Documented in `shadcn-core-stack`.
- Tailwind utilities resolving against the tokens (`bg-background`, `text-foreground`, `border-border`, `rounded-md`, `ring-ring`) : documented in `shadcn-core-stack` and the Tailwind v4 docs.

## Source

All values transcribed verbatim from https://ui.shadcn.com/docs/theming on 2026-05-19. Re-verify when shadcn ships a new default style ; the values are STYLE-dependent (`default` vs `new-york` vs `sera` vs `luma`).
