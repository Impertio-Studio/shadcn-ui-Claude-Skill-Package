---
name: shadcn-impl-data-table
description: >
  Use when building any sortable, filterable, paginated, selectable, or
  column-toggleable table in shadcn ui : the official shadcn DataTable is
  NOT a primitive component but a documented recipe that composes
  TanStack Table v8 (@tanstack/react-table) with the shadcn Table
  primitive. Use this skill any time the task involves columns of data,
  row selection with bulk actions, pagination of a list, server-side
  data loading with sorting or filtering, column-visibility toggles, or
  resizable columns.
  Prevents the recurring bugs : forgetting getCoreRowModel and getting
  zero rows, mixing client-side and server-side state so sorting fires
  twice, missing getRowId so row-selection keys collide, setting
  manualPagination without pageCount so pagination breaks, forgetting
  'use client' on the DataTable component, defining a ColumnDef without
  accessorKey or accessorFn so cells render undefined, and importing
  TanStack helpers from the wrong package.
  Covers the ColumnDef typing pattern, the useReactTable hook composition,
  the six feature row models (core, sorted, filtered, paginated, with
  row selection, with column visibility), the optional column-resizing
  setup, the server-side data contract (manualSorting plus
  manualFiltering plus manualPagination plus pageCount or rowCount), the
  reusable DataTable component composition with shadcn Table primitives,
  flexRender for dynamic cell content, and the 'use client' boundary
  rule.
  Keywords: shadcn data table, TanStack Table v8, @tanstack/react-table,
  ColumnDef, useReactTable, flexRender, sortable table, filterable table,
  paginated table, row selection, column visibility, column resizing,
  server-side data table, manualSorting, manualFiltering,
  manualPagination, pageCount, rowCount, getCoreRowModel,
  getSortedRowModel, getFilteredRowModel, getPaginationRowModel,
  accessorKey, accessorFn, SortingState, ColumnFiltersState,
  VisibilityState, getRowId, enableRowSelection, 'use client' DataTable,
  how do I make a sortable table, how do I add pagination to a table,
  how do I select rows in a table, table is empty no rows showing,
  sorting fires twice, page count infinite, my checkboxes select the
  wrong row, cells are undefined, DataTable recipe, bulk actions table,
  why are my rows not rendering.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026 with TanStack Table v8 (@tanstack/react-table)."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui : DataTable Recipe (TanStack Table v8)

Deterministic mental model for the shadcn DataTable. There is NO `DataTable` primitive in shadcn. The official recipe (https://ui.shadcn.com/docs/components/radix/data-table) composes `@tanstack/react-table` v8 with the shadcn `Table` primitive. ALWAYS treat DataTable as a recipe-skill, not a primitive-skill : you write a reusable `components/ui/data-table.tsx` file in user space.

## Quick Reference

### One mental model

A DataTable is three concerns wired together :

1. **Schema** : a `ColumnDef<TData, TValue>[]` array describes columns (accessor + header + cell).
2. **Headless engine** : `useReactTable({ data, columns, getCoreRowModel, ... })` returns a `table` instance.
3. **Markup** : a `<DataTable />` component renders the shadcn `<Table>` primitive driven by `table.getHeaderGroups()` and `table.getRowModel().rows`.

ALWAYS pass `getCoreRowModel: getCoreRowModel()` or the table renders ZERO rows. ALWAYS mark the DataTable component file `"use client"` ; the headless engine uses React hooks and breaks under server-rendering.

### Install

```bash
npx shadcn@latest add table
npm install @tanstack/react-table
```

ALWAYS install the `table` primitive first ; the recipe imports `Table`, `TableBody`, `TableCell`, `TableHead`, `TableHeader`, `TableRow` from `@/components/ui/table`.

### Six features, six row-model functions

| Feature | Row model fn | State key | Setter prop |
|---------|--------------|-----------|-------------|
| Render rows | `getCoreRowModel()` | : | : |
| Sort | `getSortedRowModel()` | `sorting: SortingState` | `onSortingChange` |
| Filter | `getFilteredRowModel()` | `columnFilters: ColumnFiltersState` | `onColumnFiltersChange` |
| Paginate | `getPaginationRowModel()` | `pagination: PaginationState` | `onPaginationChange` |
| Select rows | `getCoreRowModel()` (no extra fn) | `rowSelection: RowSelectionState` | `onRowSelectionChange` |
| Toggle columns | `getCoreRowModel()` (no extra fn) | `columnVisibility: VisibilityState` | `onColumnVisibilityChange` |

### Client vs server data : the single most important decision

| Question | Answer |
|----------|--------|
| Dataset under ~1000 rows AND loaded once ? | Client-side. Pass full array to `data`. NO `manualX` flags. |
| Dataset over 1000 rows OR loaded per page ? | Server-side. Set `manualSorting: true`, `manualFiltering: true`, `manualPagination: true`. Provide `pageCount` (or `rowCount`). Refetch in a `useEffect` on state change. |
| Mixed (server pagination, client sort) ? | RARE. Only enable the `manualX` flag for what you actually compute server-side. Mismatch causes sorting to fire twice. |

NEVER mix client-side and server-side for the SAME feature. If `manualSorting: true` is set, the engine STOPS sorting client-side ; you MUST refetch sorted data from the server.

### 'use client' boundary

ALWAYS put `"use client"` at the top of `components/ui/data-table.tsx`. The TanStack hook + the state hooks (sorting, filters, etc.) need a client component. The page that USES `<DataTable />` may stay a server component as long as it imports `<DataTable />` from a client module.

## ColumnDef Pattern

### Minimal accessor-based column

```ts
import type { ColumnDef } from "@tanstack/react-table"

export type Payment = {
  id: string
  amount: number
  status: "pending" | "processing" | "success" | "failed"
  email: string
}

export const columns: ColumnDef<Payment>[] = [
  { accessorKey: "status", header: "Status" },
  { accessorKey: "email", header: "Email" },
  { accessorKey: "amount", header: "Amount" },
]
```

ALWAYS provide either `accessorKey` (string key into `TData`) OR `accessorFn` (function returning the cell value). Without one, the cell renders `undefined`.

### Custom cell renderer with flexRender

```ts
import { flexRender } from "@tanstack/react-table"

export const columns: ColumnDef<Payment>[] = [
  {
    accessorKey: "amount",
    header: "Amount",
    cell: ({ row }) => {
      const amount = parseFloat(row.getValue("amount"))
      const formatted = new Intl.NumberFormat("en-US", { style: "currency", currency: "USD" }).format(amount)
      return <div className="text-right font-medium">{formatted}</div>
    },
  },
]
```

ALWAYS use `flexRender(cell.column.columnDef.cell, cell.getContext())` in the body markup so dynamic cells render correctly. NEVER read `cell.value` directly.

### Sortable header

```ts
{
  accessorKey: "email",
  header: ({ column }) => (
    <Button variant="ghost" onClick={() => column.toggleSorting(column.getIsSorted() === "asc")}>
      Email <ArrowUpDown className="ml-2 h-4 w-4" />
    </Button>
  ),
}
```

## useReactTable Setup

### Canonical six-feature client-side composition

```tsx
"use client"

import * as React from "react"
import {
  ColumnDef, ColumnFiltersState, SortingState, VisibilityState,
  flexRender,
  getCoreRowModel, getFilteredRowModel, getPaginationRowModel,
  getSortedRowModel, useReactTable,
} from "@tanstack/react-table"

const [sorting, setSorting] = React.useState<SortingState>([])
const [columnFilters, setColumnFilters] = React.useState<ColumnFiltersState>([])
const [columnVisibility, setColumnVisibility] = React.useState<VisibilityState>({})
const [rowSelection, setRowSelection] = React.useState({})

const table = useReactTable({
  data,
  columns,
  getCoreRowModel: getCoreRowModel(),
  getSortedRowModel: getSortedRowModel(),
  getFilteredRowModel: getFilteredRowModel(),
  getPaginationRowModel: getPaginationRowModel(),
  onSortingChange: setSorting,
  onColumnFiltersChange: setColumnFilters,
  onColumnVisibilityChange: setColumnVisibility,
  onRowSelectionChange: setRowSelection,
  state: { sorting, columnFilters, columnVisibility, rowSelection },
})
```

ALWAYS pair every `onXChange` setter with its `state.X` value, or the table loses controlled state.

## Feature Decision Matrix

| You need | Add to useReactTable | Optional state | Why |
|----------|----------------------|----------------|-----|
| Just render rows | `getCoreRowModel: getCoreRowModel()` | : | Without it, `table.getRowModel().rows` is empty. |
| Click header to sort | `getSortedRowModel: getSortedRowModel()` + `onSortingChange` + `state.sorting` | : | Without controlled state, sorting still works but is uncontrolled and unreadable from outside. |
| Per-column text filter | `getFilteredRowModel: getFilteredRowModel()` + `onColumnFiltersChange` + `state.columnFilters` | : | Drive an `<Input>` that calls `table.getColumn("email")?.setFilterValue(value)`. |
| Pagination | `getPaginationRowModel: getPaginationRowModel()` | `initialState.pagination = { pageSize: 10 }` | Without it, `table.nextPage()`/`previousPage()` no-op. |
| Row selection (checkboxes) | `onRowSelectionChange` + `state.rowSelection` | `enableRowSelection: true`, `getRowId: (row) => row.id` | NEVER omit `getRowId` if rows can be reordered/refetched ; selection collides on array-index keys. |
| Column visibility toggle | `onColumnVisibilityChange` + `state.columnVisibility` | : | Drive a `<DropdownMenu>` listing `table.getAllColumns().filter(c => c.getCanHide())`. |
| Column resizing | `columnResizeMode: "onChange"`, `enableColumnResizing: true` | per-column `enableResizing: false` | OPTIONAL. Requires inline styles on `<TableHead>` with `style={{ width: header.getSize() }}`. |

## Server-Side Data Contract

Server-side mode tells TanStack Table "I will sort, filter, and paginate ; you just render what I give you and surface the state."

### Required wiring

```tsx
const [sorting, setSorting] = React.useState<SortingState>([])
const [columnFilters, setColumnFilters] = React.useState<ColumnFiltersState>([])
const [pagination, setPagination] = React.useState({ pageIndex: 0, pageSize: 10 })

const { data, pageCount } = useQuery({ /* refetches on every state change */ })

const table = useReactTable({
  data: data ?? [],
  columns,
  pageCount,                  // total page count from server
  state: { sorting, columnFilters, pagination },
  onSortingChange: setSorting,
  onColumnFiltersChange: setColumnFilters,
  onPaginationChange: setPagination,
  manualSorting: true,
  manualFiltering: true,
  manualPagination: true,
  getCoreRowModel: getCoreRowModel(),
})
```

ALWAYS provide `pageCount` (or `rowCount`) when `manualPagination: true`. Without it, the engine defaults `pageCount` to `-1` and "next page" is infinite.

ALWAYS refetch on `sorting`, `columnFilters`, and `pagination` change. Use `useQuery` from TanStack Query, `useEffect`, or any data-fetching hook that takes these as dependencies.

NEVER add `getSortedRowModel()`, `getFilteredRowModel()`, or `getPaginationRowModel()` when their `manualX` flag is `true` ; the engine will sort/filter/paginate twice (once in your fetch, again in the model) and the visible order will not match what was fetched.

### Client vs server decision tree

```
Is the full dataset already in memory ?
|--- YES -> client-side, add the get*RowModel for each feature.
|--- NO  -> server-side, set manualX flags, provide pageCount,
            refetch on state change.
```

## DataTable Component Composition

ALWAYS extract the composition into `components/ui/data-table.tsx` for reuse. The component is generic over `<TData, TValue>`.

```tsx
"use client"

import {
  ColumnDef, flexRender,
  getCoreRowModel, useReactTable,
} from "@tanstack/react-table"
import {
  Table, TableBody, TableCell, TableHead, TableHeader, TableRow,
} from "@/components/ui/table"

interface DataTableProps<TData, TValue> {
  columns: ColumnDef<TData, TValue>[]
  data: TData[]
}

export function DataTable<TData, TValue>({ columns, data }: DataTableProps<TData, TValue>) {
  const table = useReactTable({ data, columns, getCoreRowModel: getCoreRowModel() })

  return (
    <div className="rounded-md border">
      <Table>
        <TableHeader>
          {table.getHeaderGroups().map(headerGroup => (
            <TableRow key={headerGroup.id}>
              {headerGroup.headers.map(header => (
                <TableHead key={header.id}>
                  {header.isPlaceholder ? null : flexRender(header.column.columnDef.header, header.getContext())}
                </TableHead>
              ))}
            </TableRow>
          ))}
        </TableHeader>
        <TableBody>
          {table.getRowModel().rows.length ? table.getRowModel().rows.map(row => (
            <TableRow key={row.id} data-state={row.getIsSelected() && "selected"}>
              {row.getVisibleCells().map(cell => (
                <TableCell key={cell.id}>{flexRender(cell.column.columnDef.cell, cell.getContext())}</TableCell>
              ))}
            </TableRow>
          )) : (
            <TableRow><TableCell colSpan={columns.length} className="h-24 text-center">No results.</TableCell></TableRow>
          )}
        </TableBody>
      </Table>
    </div>
  )
}
```

ALWAYS render the empty-state row (`No results.`) so the table never collapses to zero height. ALWAYS spread additional state and setters as props if the parent owns sorting/filtering/selection.

## Reference Links

- shadcn DataTable recipe : https://ui.shadcn.com/docs/components/radix/data-table
- TanStack Table v8 docs : https://tanstack.com/table/latest/docs
- TanStack pagination guide : https://tanstack.com/table/latest/docs/guide/pagination
- TanStack sorting guide : https://tanstack.com/table/latest/docs/guide/sorting
- TanStack row-selection guide : https://tanstack.com/table/latest/docs/guide/row-selection

## Companion Skills

- `shadcn-syntax-table` : the primitive `<Table>` / `<TableHeader>` / `<TableBody>` HTML wrapper this recipe renders into. READ FIRST if you are unsure of the primitive markup.
- `shadcn-syntax-selectors` : the controlled `<Select>` used for page-size dropdowns inside DataTable pagination footers.
- `shadcn-syntax-menu-primitives` : the `<DropdownMenu>` used to drive the column-visibility toggle.

## Detailed References

- `references/methods.md` : full ColumnDef, useReactTable, and per-feature signatures.
- `references/examples.md` : six worked recipes (minimal sortable, filter + paginate, row selection + bulk actions, server-side with Next.js, column visibility toggle, custom cell renderers).
- `references/anti-patterns.md` : six concrete failure modes with fixes.
