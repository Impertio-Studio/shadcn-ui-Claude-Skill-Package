# Examples: shadcn-syntax-chart

Every example below assumes:
- `npx shadcn@latest add chart` has been run (creates `components/ui/chart.tsx`).
- `--chart-1` through `--chart-5` are defined in `globals.css` under `:root` and `.dark`. See Companion Skill `shadcn-core-theming`.
- The file starts with `'use client'`.

## Example 1: BarChart with ChartConfig (Multi-Series)

```tsx
"use client"

import { Bar, BarChart, CartesianGrid, XAxis } from "recharts"
import {
  ChartContainer,
  ChartTooltip,
  ChartTooltipContent,
  ChartLegend,
  ChartLegendContent,
  type ChartConfig,
} from "@/components/ui/chart"

const chartData = [
  { month: "January",  desktop: 186, mobile:  80 },
  { month: "February", desktop: 305, mobile: 200 },
  { month: "March",    desktop: 237, mobile: 120 },
  { month: "April",    desktop:  73, mobile: 190 },
  { month: "May",      desktop: 209, mobile: 130 },
  { month: "June",     desktop: 214, mobile: 140 },
]

const chartConfig = {
  desktop: { label: "Desktop", color: "var(--chart-1)" },
  mobile:  { label: "Mobile",  color: "var(--chart-2)" },
} satisfies ChartConfig

export function VisitorsBarChart() {
  return (
    <ChartContainer config={chartConfig} className="min-h-[200px] w-full">
      <BarChart accessibilityLayer data={chartData}>
        <CartesianGrid vertical={false} />
        <XAxis
          dataKey="month"
          tickLine={false}
          tickMargin={10}
          axisLine={false}
          tickFormatter={(value) => value.slice(0, 3)}
        />
        <ChartTooltip content={<ChartTooltipContent indicator="dot" />} />
        <ChartLegend content={<ChartLegendContent />} />
        <Bar dataKey="desktop" fill="var(--color-desktop)" radius={4} />
        <Bar dataKey="mobile"  fill="var(--color-mobile)"  radius={4} />
      </BarChart>
    </ChartContainer>
  )
}
```

Key points: `dataKey` on each `Bar` matches a `ChartConfig` key; `fill` references the wrapper-generated `--color-<key>` var; `accessibilityLayer` enables Recharts keyboard navigation; `radius={4}` rounds the top corners of bars.

## Example 2: LineChart with Multiple Series

```tsx
"use client"

import { CartesianGrid, Line, LineChart, XAxis } from "recharts"
import {
  ChartContainer,
  ChartTooltip,
  ChartTooltipContent,
  type ChartConfig,
} from "@/components/ui/chart"

const chartData = [
  { month: "Jan", revenue: 12400, cost:  8200, profit:  4200 },
  { month: "Feb", revenue: 14800, cost:  9100, profit:  5700 },
  { month: "Mar", revenue: 16200, cost:  9800, profit:  6400 },
  { month: "Apr", revenue: 13900, cost:  9600, profit:  4300 },
  { month: "May", revenue: 18600, cost: 10800, profit:  7800 },
  { month: "Jun", revenue: 21200, cost: 11900, profit:  9300 },
]

const chartConfig = {
  revenue: { label: "Revenue", color: "var(--chart-1)" },
  cost:    { label: "Cost",    color: "var(--chart-2)" },
  profit:  { label: "Profit",  color: "var(--chart-3)" },
} satisfies ChartConfig

export function FinancialsLineChart() {
  return (
    <ChartContainer config={chartConfig} className="aspect-video w-full">
      <LineChart accessibilityLayer data={chartData} margin={{ left: 12, right: 12 }}>
        <CartesianGrid vertical={false} />
        <XAxis
          dataKey="month"
          tickLine={false}
          axisLine={false}
          tickMargin={8}
        />
        <ChartTooltip content={<ChartTooltipContent indicator="line" />} />
        <Line dataKey="revenue" stroke="var(--color-revenue)" type="monotone" strokeWidth={2} dot={false} />
        <Line dataKey="cost"    stroke="var(--color-cost)"    type="monotone" strokeWidth={2} dot={false} />
        <Line dataKey="profit"  stroke="var(--color-profit)"  type="monotone" strokeWidth={2} dot={false} />
      </LineChart>
    </ChartContainer>
  )
}
```

Note `indicator="line"` on the tooltip suits line charts (matches the visual metaphor). `type="monotone"` produces smooth curves; `type="linear"` produces straight segments.

## Example 3: Stacked AreaChart with Gradient Fills

```tsx
"use client"

import { Area, AreaChart, CartesianGrid, XAxis } from "recharts"
import {
  ChartContainer,
  ChartTooltip,
  ChartTooltipContent,
  type ChartConfig,
} from "@/components/ui/chart"

const chartData = [
  { date: "2026-01-01", desktop: 222, mobile: 150, tablet: 90 },
  { date: "2026-01-02", desktop: 197, mobile: 180, tablet: 75 },
  { date: "2026-01-03", desktop: 167, mobile: 120, tablet: 60 },
  { date: "2026-01-04", desktop: 242, mobile: 260, tablet: 110 },
  { date: "2026-01-05", desktop: 373, mobile: 290, tablet: 140 },
  { date: "2026-01-06", desktop: 301, mobile: 340, tablet: 130 },
]

const chartConfig = {
  desktop: { label: "Desktop", color: "var(--chart-1)" },
  mobile:  { label: "Mobile",  color: "var(--chart-2)" },
  tablet:  { label: "Tablet",  color: "var(--chart-3)" },
} satisfies ChartConfig

export function TrafficAreaChart() {
  return (
    <ChartContainer config={chartConfig} className="aspect-video w-full">
      <AreaChart accessibilityLayer data={chartData}>
        <defs>
          <linearGradient id="fillDesktop" x1="0" y1="0" x2="0" y2="1">
            <stop offset="5%"  stopColor="var(--color-desktop)" stopOpacity={0.8} />
            <stop offset="95%" stopColor="var(--color-desktop)" stopOpacity={0.1} />
          </linearGradient>
          <linearGradient id="fillMobile" x1="0" y1="0" x2="0" y2="1">
            <stop offset="5%"  stopColor="var(--color-mobile)" stopOpacity={0.8} />
            <stop offset="95%" stopColor="var(--color-mobile)" stopOpacity={0.1} />
          </linearGradient>
          <linearGradient id="fillTablet" x1="0" y1="0" x2="0" y2="1">
            <stop offset="5%"  stopColor="var(--color-tablet)" stopOpacity={0.8} />
            <stop offset="95%" stopColor="var(--color-tablet)" stopOpacity={0.1} />
          </linearGradient>
        </defs>
        <CartesianGrid vertical={false} />
        <XAxis
          dataKey="date"
          tickLine={false}
          axisLine={false}
          tickMargin={8}
          tickFormatter={(value) =>
            new Date(value).toLocaleDateString("en-US", { month: "short", day: "numeric" })
          }
        />
        <ChartTooltip content={<ChartTooltipContent indicator="line" />} />
        <Area dataKey="tablet"  type="natural" fill="url(#fillTablet)"  stroke="var(--color-tablet)"  stackId="a" />
        <Area dataKey="mobile"  type="natural" fill="url(#fillMobile)"  stroke="var(--color-mobile)"  stackId="a" />
        <Area dataKey="desktop" type="natural" fill="url(#fillDesktop)" stroke="var(--color-desktop)" stackId="a" />
      </AreaChart>
    </ChartContainer>
  )
}
```

Stacking requires identical `stackId` on every `Area`. The `<defs>` block with `<linearGradient>` is mandatory for proper top-to-bottom fading fills; without it, areas render as solid blocks.

## Example 4: PieChart with Legend

```tsx
"use client"

import { Cell, Pie, PieChart } from "recharts"
import {
  ChartContainer,
  ChartTooltip,
  ChartTooltipContent,
  ChartLegend,
  ChartLegendContent,
  type ChartConfig,
} from "@/components/ui/chart"

const chartData = [
  { browser: "chrome",   visitors: 275, fill: "var(--color-chrome)" },
  { browser: "safari",   visitors: 200, fill: "var(--color-safari)" },
  { browser: "firefox",  visitors: 187, fill: "var(--color-firefox)" },
  { browser: "edge",     visitors: 173, fill: "var(--color-edge)" },
  { browser: "other",    visitors:  90, fill: "var(--color-other)" },
]

const chartConfig = {
  visitors: { label: "Visitors" },
  chrome:   { label: "Chrome",   color: "var(--chart-1)" },
  safari:   { label: "Safari",   color: "var(--chart-2)" },
  firefox:  { label: "Firefox",  color: "var(--chart-3)" },
  edge:     { label: "Edge",     color: "var(--chart-4)" },
  other:    { label: "Other",    color: "var(--chart-5)" },
} satisfies ChartConfig

export function BrowsersPieChart() {
  return (
    <ChartContainer config={chartConfig} className="mx-auto aspect-square max-h-[300px]">
      <PieChart>
        <ChartTooltip content={<ChartTooltipContent nameKey="visitors" hideLabel />} />
        <Pie
          data={chartData}
          dataKey="visitors"
          nameKey="browser"
          innerRadius={60}
          outerRadius={100}
          paddingAngle={2}
        >
          {chartData.map((entry) => (
            <Cell key={entry.browser} fill={entry.fill} />
          ))}
        </Pie>
        <ChartLegend
          content={<ChartLegendContent nameKey="browser" />}
          verticalAlign="bottom"
        />
      </PieChart>
    </ChartContainer>
  )
}
```

Note `aspect-square` instead of `aspect-video` for circular charts. The `nameKey="browser"` on the legend tells the wrapper to look up labels under `chartConfig.browser`, `chartConfig.chrome`, etc. The `fill` is computed in the data itself referencing the wrapper-generated CSS var. `innerRadius={60}` produces a donut; set to `0` for a solid pie.

## Example 5: Custom ChartTooltipContent Override

When the default tooltip layout (label / indicator / name / value) does not fit, replace it with your own component. Use `useChart()` to access the config.

```tsx
"use client"

import { Bar, BarChart, XAxis } from "recharts"
import {
  ChartContainer,
  ChartTooltip,
  useChart,
  type ChartConfig,
} from "@/components/ui/chart"

const chartData = [
  { stage: "Lead",      count: 1240, value:  98000 },
  { stage: "Qualified", count:  860, value: 124000 },
  { stage: "Proposal",  count:  420, value: 186000 },
  { stage: "Closed",    count:  180, value: 210000 },
]

const chartConfig = {
  count: { label: "Count", color: "var(--chart-1)" },
  value: { label: "Value", color: "var(--chart-2)" },
} satisfies ChartConfig

function CustomTooltip({ active, payload }: { active?: boolean; payload?: any[] }) {
  const { config } = useChart()

  if (!active || !payload?.length) return null

  const row = payload[0].payload

  return (
    <div className="rounded-lg border border-border/50 bg-background p-3 shadow-xl">
      <div className="mb-2 font-semibold">{row.stage}</div>
      <div className="grid gap-1 text-xs">
        <div className="flex justify-between gap-4">
          <span className="text-muted-foreground">{config.count.label}</span>
          <span className="font-mono tabular-nums">{row.count.toLocaleString()}</span>
        </div>
        <div className="flex justify-between gap-4">
          <span className="text-muted-foreground">{config.value.label}</span>
          <span className="font-mono tabular-nums">
            ${row.value.toLocaleString()}
          </span>
        </div>
      </div>
    </div>
  )
}

export function FunnelBarChart() {
  return (
    <ChartContainer config={chartConfig} className="min-h-[240px] w-full">
      <BarChart accessibilityLayer data={chartData}>
        <XAxis dataKey="stage" tickLine={false} axisLine={false} />
        <ChartTooltip content={<CustomTooltip />} cursor={false} />
        <Bar dataKey="count" fill="var(--color-count)" radius={4} />
      </BarChart>
    </ChartContainer>
  )
}
```

`CustomTooltip` MUST be rendered inside `ChartContainer` so `useChart()` resolves. `cursor={false}` removes the gray hover-bar that Recharts draws by default.
