---
name: shadcn-syntax-chart
description: >
  Use when building data visualizations with the shadcn Chart primitive that wraps Recharts (Bar, Line, Area, Pie, Radar, Scatter, RadialBar, Composed).
  Prevents the common mistake of using Recharts directly (loses shadcn theming), inlining hex colors (breaks dark mode), forgetting `'use client'` (RSC crash), or omitting `ChartConfig` (no CSS-var color generation).
  Covers the five wrappers (ChartContainer, ChartTooltip, ChartTooltipContent, ChartLegend, ChartLegendContent), the `ChartConfig` type, the `--chart-1..5` and `var(--color-<dataKey>)` color convention, theme-aware light/dark color resolution, and per-chart-type Recharts composition.
  Keywords: shadcn chart, shadcn ui chart, recharts wrapper, ChartContainer, ChartTooltip, ChartTooltipContent, ChartLegend, ChartLegendContent, ChartConfig, bar chart, line chart, area chart, pie chart, radar chart, scatter chart, stacked area chart, chart colors, theme-aware chart, --chart-1 css var, --color-dataKey, dark mode chart, how do I make a chart in shadcn, my chart shows no colors, chart is black and white, recharts not themed, chart blank in production, use client chart, RSC chart error, ResponsiveContainer shadcn, custom tooltip chart, chart legend shadcn, chart not rendering, getting started with charts, plain html chart, what is ChartConfig
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn-syntax-chart

The shadcn Chart primitive is a thin theming and tooltip/legend layer over [Recharts](https://recharts.org). It does NOT replace Recharts. You compose Recharts chart elements as children of `ChartContainer`, and the shadcn wrappers inject CSS variables (`--color-<dataKey>`) derived from a `ChartConfig` object so colors track the active theme (light / dark) without inline hex values.

Install with: `npx shadcn@latest add chart`. This creates `components/ui/chart.tsx` plus the dependency `recharts` in `package.json`.

## Quick Reference

| Wrapper | Source | Role |
|---|---|---|
| `ChartContainer` | shadcn (custom) | Provides `ChartConfig` context, injects `<style>` with `--color-<key>` CSS vars, wraps Recharts `ResponsiveContainer` |
| `ChartTooltip` | re-export of `Recharts.Tooltip` | The Recharts tooltip primitive, slotted via `content` prop |
| `ChartTooltipContent` | shadcn (custom) | Themed tooltip body. Props: `indicator` (`"dot" \| "line" \| "dashed"`), `hideLabel`, `hideIndicator`, `labelKey`, `nameKey`, `labelFormatter`, `formatter` |
| `ChartLegend` | re-export of `Recharts.Legend` | The Recharts legend primitive, slotted via `content` prop |
| `ChartLegendContent` | shadcn (custom) | Themed legend body. Props: `nameKey`, `hideIcon`, `verticalAlign` |
| `ChartConfig` (type) | shadcn (custom) | `Record<string, { label?: ReactNode; icon?: ComponentType } & ({ color?: string } \| { theme: { light: string; dark: string } })>` |

ALWAYS:
- Place `'use client'` at the top of any file that imports a chart wrapper. Recharts uses `ResponsiveContainer` which depends on `ResizeObserver` and is non-serializable across the RSC boundary.
- Wrap a Recharts chart (e.g. `BarChart`, `LineChart`, `PieChart`) inside `ChartContainer` and pass a `config` prop of type `ChartConfig`.
- Set an explicit height contract on `ChartContainer`: either `className="min-h-[200px] w-full"` or `className="aspect-video"`. Recharts `ResponsiveContainer` measures parent dimensions and renders nothing when parent has zero height.
- Reference colors in Recharts child props (e.g. `<Bar fill="var(--color-desktop)" />`) using `var(--color-<dataKey>)` where `<dataKey>` matches a key in `ChartConfig`.
- Set chart-token CSS variables (`--chart-1` through `--chart-5`) in `globals.css` under `:root` and `.dark` so `color: "var(--chart-1)"` in `ChartConfig` resolves per theme. The Companion Skill `shadcn-core-theming` defines these tokens.

NEVER:
- Render a chart wrapper in a React Server Component without `'use client'`. The build will fail with `You're importing a component that needs useContext`.
- Pass inline hex / hsl / oklch strings to `<Bar fill="#3b82f6" />` or `<Line stroke="#e11d48" />` directly. This bypasses `ChartConfig`, breaks dark mode, and loses theme synchronization.
- Omit the `config` prop on `ChartContainer`. Without it, no `--color-<dataKey>` CSS vars are injected and `var(--color-desktop)` resolves to nothing (Recharts falls back to default Recharts palette).
- Use `Recharts.Tooltip` and `Recharts.Legend` directly. They render with default browser styling and ignore shadcn theme tokens. Use `ChartTooltip` + `ChartTooltipContent` and `ChartLegend` + `ChartLegendContent`.
- Wrap `ChartContainer` itself inside `Recharts.ResponsiveContainer`. `ChartContainer` already includes `ResponsiveContainer` internally; double-wrapping produces zero-sized SVG.

## Decision Tree: Which Chart Type?

```
Need to compare discrete categories?      -> BarChart (Recharts.Bar)
Need to show trend over continuous axis?  -> LineChart (Recharts.Line)
Need to show trend + magnitude/cumulative?-> AreaChart (Recharts.Area)
Need to show parts of a whole (<= 7 slices)? -> PieChart (Recharts.Pie)
Need to show multi-dimensional comparison? -> RadarChart (Recharts.Radar)
Need to show correlation between 2 metrics? -> ScatterChart (Recharts.Scatter)
Need to combine bars + lines + areas?     -> ComposedChart (Recharts.ComposedChart)
Need radial / dial style?                  -> RadialBarChart (Recharts.RadialBar)
```

See `references/methods.md` for the per-chart-type Recharts integration table and `references/examples.md` for a complete code example of each.

## ChartConfig Pattern

`ChartConfig` is the single source of truth for series labels, icons, and colors. The wrapper generates one `--color-<key>` CSS custom property per entry.

```ts
import { type ChartConfig } from "@/components/ui/chart"

const chartConfig = {
  desktop: {
    label: "Desktop",
    color: "var(--chart-1)",
  },
  mobile: {
    label: "Mobile",
    color: "var(--chart-2)",
  },
} satisfies ChartConfig
```

Two color modes are mutually exclusive per entry (the TypeScript discriminated union enforces this):

| Mode | Shape | Use when |
|---|---|---|
| Single color | `{ color: "var(--chart-1)" }` | Color is identical in light + dark (theme tokens already adapt) |
| Theme-split color | `{ theme: { light: "oklch(0.7 0.2 30)", dark: "oklch(0.8 0.15 30)" } }` | Light and dark need distinct, hand-picked colors that do NOT come from `--chart-N` tokens |

The wrapper injects:

```css
[data-chart=chart-xyz] {
  --color-desktop: var(--chart-1);
  --color-mobile: var(--chart-2);
}
.dark [data-chart=chart-xyz] {
  --color-desktop: var(--chart-1);  /* same; --chart-1 itself is dark-aware */
  --color-mobile: var(--chart-2);
}
```

Reference the keys in Recharts props: `<Bar dataKey="desktop" fill="var(--color-desktop)" />`. The `dataKey` matches the `ChartConfig` key, and the `fill` references the generated CSS var.

## Color CSS-Var Convention

Two layers of CSS variables interact:

1. **Theme tokens** (`--chart-1` through `--chart-5`): defined in `globals.css` under `:root` and `.dark`. These are the brand palette. ALWAYS define exactly five (the official shadcn theme builder generates five). Owned by the theming skill.
2. **Series tokens** (`--color-<dataKey>`): injected by `ChartContainer` per chart instance, scoped to a `[data-chart=<id>]` selector. Owned by `ChartConfig`.

The pattern: `ChartConfig.color` REFERENCES a theme token, then Recharts props reference the series token. This indirection lets one config drive many charts and lets one palette change cascade.

```css
/* globals.css */
:root {
  --chart-1: oklch(0.646 0.222 41.116);
  --chart-2: oklch(0.6 0.118 184.704);
  --chart-3: oklch(0.398 0.07 227.392);
  --chart-4: oklch(0.828 0.189 84.429);
  --chart-5: oklch(0.769 0.188 70.08);
}
.dark {
  --chart-1: oklch(0.488 0.243 264.376);
  --chart-2: oklch(0.696 0.17 162.48);
  --chart-3: oklch(0.769 0.188 70.08);
  --chart-4: oklch(0.627 0.265 303.9);
  --chart-5: oklch(0.645 0.246 16.439);
}
```

## Chart Type Catalog

| Chart type | Recharts root | Required children | Common series prop |
|---|---|---|---|
| Bar | `BarChart` | `XAxis`, `YAxis` optional, `CartesianGrid` optional, `Bar` (one per series) | `dataKey`, `fill`, `radius`, `stackId` |
| Line | `LineChart` | `XAxis`, `Line` (one per series) | `dataKey`, `stroke`, `type` (`monotone` \| `linear` \| `step`), `dot` |
| Area | `AreaChart` | `XAxis`, `Area` (one per series), `<defs>` with `<linearGradient>` for fill gradient | `dataKey`, `stroke`, `fill`, `stackId`, `type` |
| Pie | `PieChart` | `Pie` (one), `<Cell>` array for per-slice colors | `dataKey`, `nameKey`, `innerRadius`, `outerRadius`, `paddingAngle` |
| Radar | `RadarChart` | `PolarGrid`, `PolarAngleAxis`, `Radar` | `dataKey`, `stroke`, `fill`, `fillOpacity` |
| Scatter | `ScatterChart` | `XAxis`, `YAxis`, `Scatter` | `dataKey`, `fill`, `shape` |
| Composed | `ComposedChart` | mix `Bar`, `Line`, `Area` | per-child |
| RadialBar | `RadialBarChart` | `RadialBar` | `dataKey`, `cornerRadius`, `background` |

ALWAYS import the Recharts elements from `"recharts"` and the wrappers from `"@/components/ui/chart"`. They live in separate import statements.

```tsx
import { Bar, BarChart, CartesianGrid, XAxis } from "recharts"
import {
  ChartContainer,
  ChartTooltip,
  ChartTooltipContent,
  type ChartConfig,
} from "@/components/ui/chart"
```

## Theme-Aware Resolution

Color resolution is one-directional: `Recharts prop` -> `--color-<dataKey>` (per chart) -> `ChartConfig.color` -> `--chart-N` (theme) -> oklch value (light or dark scope).

When the user toggles the theme (via `next-themes` or any other mechanism that sets the `.dark` class on `<html>`), the theme tokens flip, the series tokens cascade automatically, and Recharts re-paints without re-mounting. No JavaScript code path is involved.

For per-instance overrides (e.g. one chart needs different colors than the shared palette), use the `theme` discriminator on the `ChartConfig` entry:

```ts
const chartConfig = {
  revenue: {
    label: "Revenue",
    theme: {
      light: "oklch(0.6 0.2 142)",
      dark: "oklch(0.75 0.18 142)",
    },
  },
} satisfies ChartConfig
```

## 'use client' Requirement

EVERY file that imports any chart wrapper MUST declare `'use client'` on line 1.

Reasons (all three apply):

1. `ChartContainer` uses `React.useId()` and `React.useContext(ChartContext)`. Hooks are illegal in RSCs.
2. Recharts `ResponsiveContainer` registers a `ResizeObserver` on the host element. `ResizeObserver` does not exist in the Node runtime that renders RSCs.
3. SVG `<defs>` with `<linearGradient>` that Area charts use rely on stable client-side `id` allocation.

Symptom of missing `'use client'` in Next.js App Router: build-time error `You're importing a component that needs useContext. It only works in a Client Component but none of its parents are marked with "use client", so they're Server Components by default.`

Recovery pattern: extract the chart into a separate `*.client.tsx` file (e.g. `revenue-chart.client.tsx`) with `'use client'` at the top, then import it from your server-rendered page. The page itself stays a Server Component; only the chart is a Client boundary.

## Custom Tooltip Content

The default `ChartTooltipContent` renders an indicator + label + value triplet for each series. Customization happens via props:

| Prop | Type | Default | Use for |
|---|---|---|---|
| `indicator` | `"dot" \| "line" \| "dashed"` | `"dot"` | Visual marker style. `"line"` for line/area charts, `"dot"` for bars |
| `hideLabel` | `boolean` | `false` | Hide the X-axis label (top row of tooltip) |
| `hideIndicator` | `boolean` | `false` | Hide the colored marker entirely |
| `labelKey` | `string` | `dataKey` of first item | Override which field provides the label string |
| `nameKey` | `string` | `name` of payload item | Override which field provides the series name |
| `labelFormatter` | `(label, payload) => ReactNode` | none | Custom label rendering (e.g. date formatting) |
| `formatter` | `(value, name, item, index, payload) => ReactNode` | none | Full custom row rendering |

For total custom control, replace `ChartTooltipContent` with your own component passed to `content`. The child receives Recharts `TooltipProps` plus access to `ChartConfig` via `useChart()` (exported from `@/components/ui/chart`).

See `references/examples.md` for a custom-tooltip example.

## Companion Skills

Load on demand when their topic surfaces during chart work:

- **shadcn-core-theming**: defines `--chart-1` through `--chart-5` CSS variables, light/dark scoping, oklch color space. Required reading before customizing chart palette.
- **shadcn-impl-component-install**: covers `npx shadcn@latest add chart` workflow, dependency resolution for `recharts`, and post-install adjustments to `components/ui/chart.tsx`.
- **shadcn-impl-rsc-vs-client-boundaries**: covers when to split a page into server / client components. Charts ALWAYS fall on the client side; the boundary is typically the parent page.
- **shadcn-core-stack**: maps Recharts as the underlying visualization library and explains why shadcn wraps rather than reimplements.

## Common Failure Modes

See `references/anti-patterns.md` for the exhaustive list. The top five:

1. `ChartContainer` without `ChartConfig` -> chart renders but with default Recharts colors, no theme awareness.
2. Inline `fill="#3b82f6"` -> chart renders but dark mode shows wrong colors.
3. Missing `'use client'` -> RSC build error.
4. Missing height constraint on `ChartContainer` -> chart renders 0 pixels tall (silent).
5. Importing `Tooltip` / `Legend` from `recharts` instead of `ChartTooltip` / `ChartLegend` from `@/components/ui/chart` -> theming wrappers not applied.

## Reference Files

- `references/methods.md`: full wrapper signatures, ChartConfig type definition, per-chart-type Recharts integration table.
- `references/examples.md`: working code for BarChart (multi-series), LineChart (multi-series), stacked AreaChart, PieChart with legend, custom ChartTooltipContent override.
- `references/anti-patterns.md`: seven anti-patterns with symptoms, root cause, and fix.
