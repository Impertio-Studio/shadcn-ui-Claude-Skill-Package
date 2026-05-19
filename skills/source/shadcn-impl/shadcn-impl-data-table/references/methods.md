# DataTable Methods Reference

Verified against https://ui.shadcn.com/docs/components/radix/data-table and https://tanstack.com/table/latest/docs.

## ColumnDef

```ts
import type { ColumnDef } from "@tanstack/react-table"

type ColumnDef<TData, TValue = unknown> = {
  // Identity (one of accessorKey OR accessorFn is REQUIRED for data cells)
  id?: string
  accessorKey?: keyof TData & string
  accessorFn?: (row: TData, rowIndex: number) => TValue

  // Rendering
  header?: string | ((ctx: HeaderContext<TData, TValue>) => ReactNode)
  cell?: (ctx: CellContext<TData, TValue>) => ReactNode
  footer?: string | ((ctx: HeaderContext<TData, TValue>) => ReactNode)

  // Feature flags (per-column override of table-level config)
  enableSorting?: boolean
  enableColumnFilter?: boolean
  enableGlobalFilter?: boolean
  enableHiding?: boolean
  enableResizing?: boolean
  enableMultiSort?: boolean

  // Sizing (for column resizing)
  size?: number       // default
  minSize?: number    // floor
  maxSize?: number    // ceiling

  // Grouping into a header tree
  columns?: ColumnDef<TData, TValue>[]

  // Filter
  filterFn?: FilterFn<TData> | BuiltInFilterFn
  sortingFn?: SortingFn<TData> | BuiltInSortingFn

  // Custom meta passthrough
  meta?: Record<string, unknown>
}
```

Rule : `id` is REQUIRED when neither `accessorKey` nor `accessorFn` is set (display-only columns like `select` or `actions`).

## useReactTable

```ts
import { useReactTable } from "@tanstack/react-table"

const table = useReactTable<TData>(options: TableOptions<TData>): Table<TData>
```

### TableOptions (most-used fields)

```ts
type TableOptions<TData> = {
  // Required
  data: TData[]
  columns: ColumnDef<TData, any>[]
  getCoreRowModel: () => RowModel<TData>

  // Optional row models (one per feature you want enabled)
  getSortedRowModel?: () => RowModel<TData>
  getFilteredRowModel?: () => RowModel<TData>
  getPaginationRowModel?: () => RowModel<TData>
  getExpandedRowModel?: () => RowModel<TData>
  getGroupedRowModel?: () => RowModel<TData>
  getFacetedRowModel?: () => RowModel<TData>

  // Controlled state (each is independent)
  state?: Partial<TableState>
  initialState?: Partial<TableState>
  onStateChange?: (state: TableState) => void
  onSortingChange?: OnChangeFn<SortingState>
  onColumnFiltersChange?: OnChangeFn<ColumnFiltersState>
  onColumnVisibilityChange?: OnChangeFn<VisibilityState>
  onRowSelectionChange?: OnChangeFn<RowSelectionState>
  onPaginationChange?: OnChangeFn<PaginationState>
  onGlobalFilterChange?: OnChangeFn<unknown>

  // Server-side flags
  manualSorting?: boolean
  manualFiltering?: boolean
  manualPagination?: boolean
  manualGrouping?: boolean
  manualExpanding?: boolean
  pageCount?: number          // REQUIRED if manualPagination: true (or supply rowCount)
  rowCount?: number           // alternative to pageCount

  // Row selection
  enableRowSelection?: boolean | ((row: Row<TData>) => boolean)
  enableMultiRowSelection?: boolean
  enableSubRowSelection?: boolean
  getRowId?: (row: TData, index: number, parent?: Row<TData>) => string

  // Column resizing
  enableColumnResizing?: boolean
  columnResizeMode?: "onChange" | "onEnd"
  columnResizeDirection?: "ltr" | "rtl"

  // Global feature toggles
  enableSorting?: boolean
  enableMultiSort?: boolean
  enableFilters?: boolean
  enableColumnFilters?: boolean
  enableGlobalFilter?: boolean
  enableHiding?: boolean
}
```

## State Type Signatures

```ts
import type {
  SortingState, ColumnFiltersState, VisibilityState,
  RowSelectionState, PaginationState,
} from "@tanstack/react-table"

type SortingState        = { id: string; desc: boolean }[]
type ColumnFiltersState  = { id: string; value: unknown }[]
type VisibilityState     = Record<string, boolean>
type RowSelectionState   = Record<string, boolean>
type PaginationState     = { pageIndex: number; pageSize: number }
```

## Table Instance Methods (most-used)

### Header / row iteration

```ts
table.getHeaderGroups(): HeaderGroup<TData>[]
table.getFooterGroups(): HeaderGroup<TData>[]
table.getRowModel(): RowModel<TData>             // { rows, flatRows, rowsById }
table.getAllColumns(): Column<TData>[]
table.getAllLeafColumns(): Column<TData>[]
```

### Sorting

```ts
table.setSorting(updater: Updater<SortingState>): void
column.toggleSorting(desc?: boolean, multi?: boolean): void
column.getIsSorted(): false | "asc" | "desc"
column.clearSorting(): void
```

### Filtering

```ts
table.setColumnFilters(updater: Updater<ColumnFiltersState>): void
table.setGlobalFilter(updater: Updater<unknown>): void
column.setFilterValue(updater: Updater<unknown>): void
column.getFilterValue(): unknown
```

### Pagination

```ts
table.setPageIndex(updater: Updater<number>): void
table.setPageSize(updater: Updater<number>): void
table.nextPage(): void
table.previousPage(): void
table.firstPage(): void
table.lastPage(): void
table.getCanNextPage(): boolean
table.getCanPreviousPage(): boolean
table.getPageCount(): number
table.getRowCount(): number
table.getState().pagination: PaginationState
```

### Row selection

```ts
table.toggleAllPageRowsSelected(value?: boolean): void
table.toggleAllRowsSelected(value?: boolean): void
table.getIsAllPageRowsSelected(): boolean
table.getIsSomePageRowsSelected(): boolean
table.getSelectedRowModel(): RowModel<TData>
row.toggleSelected(value?: boolean): void
row.getIsSelected(): boolean
row.getCanSelect(): boolean
```

### Column visibility

```ts
table.setColumnVisibility(updater: Updater<VisibilityState>): void
table.getIsAllColumnsVisible(): boolean
column.getCanHide(): boolean
column.getIsVisible(): boolean
column.toggleVisibility(value?: boolean): void
```

### Column resizing

```ts
header.getSize(): number
header.getStart(): number
column.getCanResize(): boolean
header.getResizeHandler(): (event: unknown) => void
column.resetSize(): void
```

## flexRender

```ts
import { flexRender } from "@tanstack/react-table"

flexRender(
  component: ReactNode | ((ctx: any) => ReactNode),
  context: HeaderContext<TData, TValue> | CellContext<TData, TValue>,
): ReactNode
```

Use for ANY `header` / `cell` / `footer` definition that may be a function ; flexRender handles both static nodes and function definitions uniformly.

## Server-Side Contract

When `manualX: true`, the engine SKIPS that pipeline stage. You MUST :

| Flag | What you must provide | What you must omit |
|------|------------------------|--------------------|
| `manualSorting: true` | sorted `data` from server | `getSortedRowModel()` |
| `manualFiltering: true` | filtered `data` from server | `getFilteredRowModel()` |
| `manualPagination: true` | paged `data` + `pageCount` (or `rowCount`) | `getPaginationRowModel()` |

Refetch trigger : `onSortingChange`, `onColumnFiltersChange`, and `onPaginationChange` fire on every UI interaction. Refetch in a `useEffect` keyed on the relevant state slices, or pass them as TanStack Query keys.

## Imports Cheat Sheet

```ts
import {
  // Hook
  useReactTable,
  // Row-model functions
  getCoreRowModel, getSortedRowModel, getFilteredRowModel,
  getPaginationRowModel, getExpandedRowModel,
  // Rendering helper
  flexRender,
  // Types
  type ColumnDef, type SortingState, type ColumnFiltersState,
  type VisibilityState, type RowSelectionState, type PaginationState,
  type Row, type Column, type Header, type Table,
} from "@tanstack/react-table"
```

ALWAYS import from `@tanstack/react-table` (v8). NEVER import from `react-table` (v7, deprecated). NEVER mix v7 and v8 packages.
