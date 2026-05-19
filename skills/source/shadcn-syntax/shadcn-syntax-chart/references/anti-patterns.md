# Anti-Patterns: shadcn-syntax-chart

Each entry: symptom -> root cause -> fix. Verified against `shadcn-ui/ui` source (`apps/v4/registry/new-york-v4/ui/chart.tsx`) and Recharts v2.x runtime behavior.

## AP-01: ChartContainer Without ChartConfig

**Symptom**: Chart renders, but bars / lines / areas appear in Recharts default colors (gray-blue-green palette). Dark mode produces no change. Hovering produces a tooltip with no series labels.

**Root cause**: `ChartContainer` injects `--color-<dataKey>` CSS vars by iterating `Object.entries(config)`. With an empty or missing `config`, the inner `<ChartStyle>` returns `null` and zero vars are written. Recharts then falls back to its built-in palette because `var(--color-desktop)` resolves to the empty string. Tooltip labels rely on `config[key].label`, so missing config also breaks tooltip text.

**Fix**: ALWAYS pass a `config` object whose keys match every `dataKey` you use in Recharts children. Even single-series charts need it.

```tsx
// WRONG
<ChartContainer config={{}}>
  <BarChart data={data}>
    <Bar dataKey="visitors" fill="var(--color-visitors)" />
  </BarChart>
</ChartContainer>

// RIGHT
const chartConfig = {
  visitors: { label: "Visitors", color: "var(--chart-1)" },
} satisfies ChartConfig

<ChartContainer config={chartConfig}>
  <BarChart data={data}>
    <Bar dataKey="visitors" fill="var(--color-visitors)" />
  </BarChart>
</ChartContainer>
```

## AP-02: Inline Hex / HSL Colors Instead of CSS-Var References

**Symptom**: Chart looks correct in light mode but is illegible or visually clashing in dark mode (same hex value against a near-black background). Theme switches do not affect chart colors.

**Root cause**: `<Bar fill="#3b82f6" />` is a literal CSS value. It does not pass through `ChartConfig`, ignores `--chart-N` tokens, and does not switch when `.dark` is applied to `<html>`. The shadcn theming model relies entirely on CSS custom property indirection; bypassing it with literals breaks the contract.

**Fix**: ALWAYS reference colors as `var(--color-<dataKey>)` in Recharts props and define the actual color in `ChartConfig`. If you need a one-off color not tied to `--chart-N`, use the `theme` discriminator in `ChartConfig`:

```tsx
// WRONG: hardcoded hex
<Bar dataKey="desktop" fill="#3b82f6" radius={4} />

// RIGHT: via ChartConfig
const chartConfig = {
  desktop: {
    label: "Desktop",
    theme: { light: "oklch(0.6 0.2 240)", dark: "oklch(0.75 0.18 240)" },
  },
} satisfies ChartConfig

<Bar dataKey="desktop" fill="var(--color-desktop)" radius={4} />
```

## AP-03: Missing 'use client' on Chart File

**Symptom**: Next.js App Router build fails with:
```
You're importing a component that needs useContext. It only works in a Client Component
but none of its parents are marked with "use client", so they're Server Components by default.
```

**Root cause**: `ChartContainer` uses `React.createContext` + `React.useId()` + `React.useContext`. Recharts `ResponsiveContainer` uses `ResizeObserver`, which does not exist in the Node runtime where RSCs render. The chart wrapper file IS a Client Component file by nature.

**Fix**: ALWAYS add `'use client'` as the first line of any file that imports a chart wrapper. Do NOT add it to `components/ui/chart.tsx` (it is already there). Add it to your consuming file.

```tsx
// WRONG: no directive in consuming file
import { ChartContainer } from "@/components/ui/chart"
export default function Page() { ... }

// RIGHT
"use client"

import { ChartContainer } from "@/components/ui/chart"
export default function Page() { ... }
```

Common workflow: keep the page itself as an RSC (server-rendered), extract the chart into a `*.client.tsx` child component that starts with `'use client'`, and import that child from the server page. See Companion Skill `shadcn-impl-rsc-vs-client-boundaries`.

## AP-04: Calling Chart Inside a React Server Component

**Symptom**: Same as AP-03 (build error). Specifically, the error mentions the file containing the chart, not its parent.

**Root cause**: A page or layout under Next.js `app/` directory defaults to RSC. Importing `ChartContainer` from such a file is rejected because contexts and `useId` are RSC-illegal.

**Fix**: Move the chart into a separate file with `'use client'` at the top, then import it from the server file. The boundary creates a Client Component island inside the otherwise-server page.

```tsx
// WRONG: app/dashboard/page.tsx (RSC by default)
import { BarChart } from "recharts"
import { ChartContainer } from "@/components/ui/chart"

export default async function DashboardPage() {
  const data = await db.query(...)
  return <ChartContainer config={...}>...</ChartContainer>  // FAILS
}

// RIGHT: split into two files
// app/dashboard/chart.client.tsx
"use client"
import { BarChart } from "recharts"
import { ChartContainer, type ChartConfig } from "@/components/ui/chart"
export function DashboardChart({ data }: { data: Row[] }) {
  return <ChartContainer config={chartConfig}>...</ChartContainer>
}

// app/dashboard/page.tsx
import { DashboardChart } from "./chart.client"
export default async function DashboardPage() {
  const data = await db.query(...)
  return <DashboardChart data={data} />
}
```

## AP-05: Using Recharts Directly Without Shadcn Wrappers

**Symptom**: Chart works but has default Recharts visual styling: white background, default font, default tooltip card with browser styling, gray gridlines, no theme awareness. Dark mode does not affect the chart at all.

**Root cause**: Importing `BarChart`, `Bar`, `Tooltip`, `Legend`, `ResponsiveContainer` directly from `recharts` skips the entire shadcn theming layer. There is no `ChartContext`, no `--color-<key>` vars, no `data-chart` selector. The chart becomes an island of un-themed visualization in an otherwise themed app.

**Fix**: ALWAYS wrap with `ChartContainer` and ALWAYS use `ChartTooltip` + `ChartTooltipContent` + `ChartLegend` + `ChartLegendContent` instead of the raw Recharts equivalents.

```tsx
// WRONG: bare Recharts
import { Bar, BarChart, Tooltip, Legend, ResponsiveContainer } from "recharts"

<ResponsiveContainer width="100%" height={300}>
  <BarChart data={data}>
    <Bar dataKey="visitors" fill="#3b82f6" />
    <Tooltip />
    <Legend />
  </BarChart>
</ResponsiveContainer>

// RIGHT: shadcn wrappers
import { Bar, BarChart } from "recharts"
import {
  ChartContainer,
  ChartTooltip,
  ChartTooltipContent,
  ChartLegend,
  ChartLegendContent,
} from "@/components/ui/chart"

<ChartContainer config={chartConfig} className="min-h-[300px] w-full">
  <BarChart data={data}>
    <Bar dataKey="visitors" fill="var(--color-visitors)" />
    <ChartTooltip content={<ChartTooltipContent />} />
    <ChartLegend content={<ChartLegendContent />} />
  </BarChart>
</ChartContainer>
```

## AP-06: Missing Height Constraint on ChartContainer

**Symptom**: Chart renders zero pixels tall. DOM inspection shows an empty `<div data-slot="chart">` with `height: 0`. No error in console.

**Root cause**: Recharts `ResponsiveContainer` measures the parent's height via `ResizeObserver`. If the parent has no intrinsic or explicit height (typical when placed in a flex column or a plain `<div>`), the measurement is zero and the SVG is rendered at zero size. `ChartContainer` does NOT impose a default height.

**Fix**: ALWAYS set either `min-h-[N]` (pixel floor) or `aspect-*` (width-driven ratio) on `ChartContainer`'s `className`.

```tsx
// WRONG: no height
<ChartContainer config={chartConfig} className="w-full">
  <BarChart data={data}>...</BarChart>
</ChartContainer>

// RIGHT: min-height
<ChartContainer config={chartConfig} className="min-h-[240px] w-full">
  <BarChart data={data}>...</BarChart>
</ChartContainer>

// RIGHT: aspect ratio
<ChartContainer config={chartConfig} className="aspect-video w-full">
  <BarChart data={data}>...</BarChart>
</ChartContainer>
```

For pie / donut charts use `aspect-square`. For dashboards typically `min-h-[300px]` or `aspect-[4/3]`.

## AP-07: Double-Wrapping with ResponsiveContainer

**Symptom**: Chart renders at zero size OR the SVG appears at a small fixed size that does not respond to container resize.

**Root cause**: `ChartContainer` already includes `Recharts.ResponsiveContainer` internally (see `chart.tsx` source). Nesting another `ResponsiveContainer` inside its children causes the inner one to measure the parent's zero-sized SVG and propagate that zero up the tree.

**Fix**: NEVER add `<ResponsiveContainer>` inside `ChartContainer`'s children. The Recharts chart root (`BarChart`, `LineChart`, etc.) is the direct child.

```tsx
// WRONG: nested ResponsiveContainer
import { ResponsiveContainer, BarChart, Bar } from "recharts"

<ChartContainer config={chartConfig} className="aspect-video">
  <ResponsiveContainer width="100%" height="100%">
    <BarChart data={data}>
      <Bar dataKey="visitors" fill="var(--color-visitors)" />
    </BarChart>
  </ResponsiveContainer>
</ChartContainer>

// RIGHT: chart root is the direct child
import { BarChart, Bar } from "recharts"

<ChartContainer config={chartConfig} className="aspect-video">
  <BarChart data={data}>
    <Bar dataKey="visitors" fill="var(--color-visitors)" />
  </BarChart>
</ChartContainer>
```

This anti-pattern is especially common when migrating existing Recharts code into shadcn; developers preserve the `ResponsiveContainer` from their old code.

## AP-08: Mismatched ChartConfig Key vs Recharts dataKey

**Symptom**: Chart renders, but one or more series have no color (default gray) and missing labels in tooltip / legend.

**Root cause**: The wrapper generates CSS vars verbatim from `ChartConfig` keys. If your `ChartConfig` has key `"Desktop"` (capital D) and your `Bar` has `dataKey="desktop"`, the generated var is `--color-Desktop` and the reference `var(--color-desktop)` resolves to nothing. CSS custom properties are case-sensitive.

**Fix**: ALWAYS use identical casing and spelling for the `ChartConfig` key, the `dataKey` on the Recharts element, and the `var(--color-X)` reference.

```tsx
// WRONG: capitalization mismatch
const chartConfig = {
  Desktop: { label: "Desktop", color: "var(--chart-1)" },
} satisfies ChartConfig

<Bar dataKey="desktop" fill="var(--color-desktop)" />  // resolves to nothing

// RIGHT: consistent lowercase
const chartConfig = {
  desktop: { label: "Desktop", color: "var(--chart-1)" },
} satisfies ChartConfig

<Bar dataKey="desktop" fill="var(--color-desktop)" />
```
