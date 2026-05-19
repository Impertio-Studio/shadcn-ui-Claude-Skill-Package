# Methods Reference: shadcn-syntax-chart

Verified against `apps/v4/registry/new-york-v4/ui/chart.tsx` in `shadcn-ui/ui` (evergreen-2026) and `recharts` v2.x API documentation.

## ChartConfig Type Definition

```ts
type ChartConfig = Record<
  string,
  {
    label?: React.ReactNode
    icon?: React.ComponentType
  } & (
    | { color?: string; theme?: never }
    | { color?: never; theme: Record<"light" | "dark", string> }
  )
>
```

Discriminated union: `color` and `theme` are mutually exclusive per entry. `label` and `icon` are independent and optional.

Usage with type-narrowing:

```ts
import { type ChartConfig } from "@/components/ui/chart"

const chartConfig = {
  desktop: { label: "Desktop", color: "var(--chart-1)" },
  mobile:  { label: "Mobile",  color: "var(--chart-2)" },
} satisfies ChartConfig
```

The `satisfies` operator preserves the literal key types so `keyof typeof chartConfig` resolves to `"desktop" | "mobile"`. ALWAYS prefer `satisfies` over `as ChartConfig` (the latter erases the literal keys).

## Wrapper Signatures

### ChartContainer

```tsx
function ChartContainer({
  id,
  className,
  children,
  config,
  initialDimension = { width: 320, height: 200 },
  ...props
}: React.ComponentProps<"div"> & {
  config: ChartConfig
  children: React.ComponentProps<typeof Recharts.ResponsiveContainer>["children"]
  initialDimension?: { width: number; height: number }
}): JSX.Element
```

Behavior:
- Generates a unique `chart-<id>` selector via `React.useId()`.
- Provides `ChartContext` with `{ config }`.
- Injects an internal `<ChartStyle>` that writes `--color-<key>` CSS vars under `[data-chart=chart-<id>]` for `:root` and `.dark`.
- Wraps `children` in `Recharts.ResponsiveContainer`.

The `children` prop is constrained to a single Recharts chart element (`BarChart`, `LineChart`, `AreaChart`, `PieChart`, `RadarChart`, `ScatterChart`, `ComposedChart`, `RadialBarChart`). It cannot be a fragment or an array.

### ChartTooltip

```ts
const ChartTooltip = Recharts.Tooltip
```

Direct re-export. Pass `ChartTooltipContent` (or a custom component) via the `content` prop:

```tsx
<ChartTooltip content={<ChartTooltipContent />} />
```

Recharts `Tooltip` props that apply: `cursor`, `active`, `payload`, `coordinate`, `position`, `filterNull`, `defaultIndex`, `wrapperStyle`.

### ChartTooltipContent

```tsx
function ChartTooltipContent({
  active,
  payload,
  className,
  indicator = "dot",
  hideLabel = false,
  hideIndicator = false,
  label,
  labelFormatter,
  labelClassName,
  formatter,
  color,
  nameKey,
  labelKey,
}: React.ComponentProps<typeof Recharts.Tooltip>
   & React.ComponentProps<"div">
   & {
     hideLabel?: boolean
     hideIndicator?: boolean
     indicator?: "line" | "dot" | "dashed"
     nameKey?: string
     labelKey?: string
   }): JSX.Element | null
```

Returns `null` when `active === false` or `payload` is empty.

Indicator visual mapping:
- `"dot"` -> 10x10px filled square, `items-center` aligned (default for bars, pies)
- `"line"` -> 4px wide tall bar (use for line / area charts)
- `"dashed"` -> 1.5px dashed border, transparent fill

### ChartLegend

```ts
const ChartLegend = Recharts.Legend
```

Direct re-export. Pass `ChartLegendContent` via `content`:

```tsx
<ChartLegend content={<ChartLegendContent />} />
```

Recharts `Legend` props that apply: `verticalAlign` (`"top" | "middle" | "bottom"`), `align` (`"left" | "center" | "right"`), `layout` (`"horizontal" | "vertical"`), `iconType`.

### ChartLegendContent

```tsx
function ChartLegendContent({
  className,
  hideIcon = false,
  payload,
  verticalAlign = "bottom",
  nameKey,
}: React.ComponentProps<"div"> & Pick<
  Recharts.LegendProps,
  "payload" | "verticalAlign"
> & {
  hideIcon?: boolean
  nameKey?: string
}): JSX.Element | null
```

Returns `null` when `payload` is empty. Renders each payload item with an icon (from `ChartConfig.icon` if defined, else a colored square) plus the label.

## useChart Hook

```ts
function useChart(): { config: ChartConfig }
```

Throws `"useChart must be used within a <ChartContainer />"` if called outside the provider. Use ONLY inside custom tooltip / legend content components to access the active `ChartConfig`.

## Per-Chart-Type Recharts Integration

| Chart type | Root component | Required children | Optional children | Series prop name |
|---|---|---|---|---|
| Bar | `BarChart` | `Bar` | `XAxis`, `YAxis`, `CartesianGrid`, `Tooltip`, `Legend` | `Bar.dataKey` |
| Line | `LineChart` | `Line` | `XAxis`, `YAxis`, `CartesianGrid`, `Tooltip`, `Legend` | `Line.dataKey` |
| Area | `AreaChart` | `Area` | `XAxis`, `YAxis`, `CartesianGrid`, `Tooltip`, `Legend`, `<defs>` | `Area.dataKey` |
| Pie | `PieChart` | `Pie` | `Tooltip`, `Legend` | `Pie.dataKey` + `Pie.nameKey` |
| Radar | `RadarChart` | `PolarGrid`, `PolarAngleAxis`, `Radar` | `PolarRadiusAxis`, `Tooltip`, `Legend` | `Radar.dataKey` |
| Scatter | `ScatterChart` | `XAxis`, `YAxis`, `Scatter` | `ZAxis`, `CartesianGrid`, `Tooltip`, `Legend` | `Scatter.dataKey` |
| Composed | `ComposedChart` | at least one of `Bar`, `Line`, `Area` | `XAxis`, `YAxis`, `CartesianGrid`, `Tooltip`, `Legend` | per-child |
| RadialBar | `RadialBarChart` | `RadialBar` | `PolarAngleAxis`, `Tooltip`, `Legend` | `RadialBar.dataKey` |

## Recharts Series Props (reference)

```tsx
// Bar
<Bar
  dataKey="desktop"
  fill="var(--color-desktop)"   // CSS-var ref, NOT hex
  radius={[4, 4, 0, 0]}          // tuple: [tl, tr, br, bl]
  stackId="a"                    // group bars on same X for stacking
/>

// Line
<Line
  dataKey="visitors"
  stroke="var(--color-visitors)"
  type="monotone"                // "monotone" | "linear" | "step" | "natural"
  dot={false}                    // hide per-point dots
  strokeWidth={2}
/>

// Area (requires <defs> + linearGradient for proper fill)
<Area
  dataKey="revenue"
  stroke="var(--color-revenue)"
  fill="url(#fillRevenue)"       // refers to <defs> gradient id
  fillOpacity={0.4}
  stackId="a"
  type="natural"
/>

// Pie (use <Cell> children for per-slice colors)
<Pie
  data={chartData}
  dataKey="visitors"
  nameKey="browser"
  innerRadius={60}                // donut hole
  outerRadius={80}
  paddingAngle={2}
>
  {chartData.map((entry) => (
    <Cell key={entry.browser} fill={`var(--color-${entry.browser})`} />
  ))}
</Pie>

// Radar
<Radar
  dataKey="skill"
  stroke="var(--color-skill)"
  fill="var(--color-skill)"
  fillOpacity={0.6}
/>

// Scatter
<Scatter
  dataKey="latency"
  fill="var(--color-latency)"
/>
```

## ResponsiveContainer Behavior

`ChartContainer` ALWAYS wraps its `children` in `Recharts.ResponsiveContainer` internally. Consequences:

1. Do NOT add another `<ResponsiveContainer>` inside `ChartContainer`'s children. The chart will measure zero.
2. The parent of `ChartContainer` MUST have a measurable height. If `ChartContainer` itself uses `aspect-video`, height derives from width. If it uses `min-h-[X]`, height is at least X.
3. The `initialDimension` prop seeds Recharts before the first `ResizeObserver` callback fires. Default `{ width: 320, height: 200 }` prevents flash-of-empty-chart during hydration.

## chartConfig Key Naming Rules

The keys in `ChartConfig` MUST match `dataKey` values used in Recharts children OR the values referenced via `nameKey` (Pie/Radial). The wrapper generates `--color-<exactKey>` verbatim:

| ChartConfig key | Generated CSS var | Reference in Recharts |
|---|---|---|
| `desktop` | `--color-desktop` | `fill="var(--color-desktop)"` |
| `mobile-2024` | `--color-mobile-2024` | `fill="var(--color-mobile-2024)"` |
| `q1Revenue` | `--color-q1Revenue` | `fill="var(--color-q1Revenue)"` |

ALWAYS use kebab-case OR camelCase consistently. CSS custom properties are case-sensitive; `--color-Desktop` and `--color-desktop` are different vars.
