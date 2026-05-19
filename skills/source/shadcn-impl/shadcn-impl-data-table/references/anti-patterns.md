# DataTable Anti-Patterns

Six concrete failure modes with their causes and fixes. Verified against https://ui.shadcn.com/docs/components/radix/data-table and https://tanstack.com/table/latest/docs.

---

## 1. Forgetting `getCoreRowModel` : zero rows render

### Wrong

```tsx
const table = useReactTable({ data, columns })
// table.getRowModel().rows -> [] (empty, even though data is populated)
```

### Symptom

The DataTable shows headers but the body is empty. `data.length` is non-zero. The empty-state row (`No results.`) renders for a populated dataset.

### Root cause

`getCoreRowModel` is the pipeline stage that turns `data` into `Row<TData>[]`. Without it, NO row model is built. Every feature row-model function (sorted, filtered, paginated) DEPENDS on the core row model and chains off it.

### Fix

```tsx
import { getCoreRowModel, useReactTable } from "@tanstack/react-table"

const table = useReactTable({
  data, columns,
  getCoreRowModel: getCoreRowModel(),            // ALWAYS, no exceptions
})
```

ALWAYS pass `getCoreRowModel: getCoreRowModel()` on EVERY `useReactTable` call, even if you do not need any other feature.

---

## 2. Mixing client-side and server-side state : sorting fires twice

### Wrong

```tsx
const table = useReactTable({
  data: serverFetchedSortedRows,
  columns,
  manualSorting: true,
  state: { sorting },
  onSortingChange: setSorting,
  getCoreRowModel: getCoreRowModel(),
  getSortedRowModel: getSortedRowModel(),       // BUG : already sorted by server, this re-sorts client-side
})
```

### Symptom

User clicks a header. Sort indicator flips. Visible row order reverts to the server's order, then jumps to the client-sorted order on the next render. Console shows two consecutive renders. With reversed comparators or different collations, the final order CONTRADICTS the API response.

### Root cause

`manualSorting: true` tells the engine "the data is ALREADY sorted, do not sort again." But the presence of `getSortedRowModel()` adds a sorted-row-model stage to the pipeline, which sorts a second time client-side, using the engine's default comparator. The two stages can disagree (case sensitivity, locale, null position).

### Fix

```tsx
const table = useReactTable({
  data: serverFetchedSortedRows,
  columns,
  manualSorting: true,                         // server sorts
  state: { sorting },
  onSortingChange: setSorting,
  getCoreRowModel: getCoreRowModel(),
  // getSortedRowModel : OMITTED. Same rule for filtering and pagination.
})
```

Rule : `manualX: true` and `getXRowModel()` are MUTUALLY EXCLUSIVE. Pick one strategy per feature.

---

## 3. `getRowId` missing : checkbox selection collides on refetch

### Wrong

```tsx
const table = useReactTable({
  data, columns,
  enableRowSelection: true,
  state: { rowSelection },
  onRowSelectionChange: setRowSelection,
  getCoreRowModel: getCoreRowModel(),
  // getRowId missing : engine falls back to row-array-index as key ("0", "1", "2", ...)
})
```

### Symptom

User selects row 3. Data refetches (server-side pagination, search, reorder, or live update). User now sees a different user/payment/order in slot 3 ; the checkbox is still checked. Selecting "page 1, row 3" then navigating to page 2 selects "page 2, row 3" too because both share index `"3"`.

### Root cause

`RowSelectionState` is `Record<string, boolean>`. The string key defaults to `row.index` (the array position in `data`). When data changes shape, indexes point to different items. Selection is bound to position instead of identity.

### Fix

```tsx
const table = useReactTable({
  data, columns,
  enableRowSelection: true,
  state: { rowSelection },
  onRowSelectionChange: setRowSelection,
  getRowId: row => row.id,                     // stable per-record key
  getCoreRowModel: getCoreRowModel(),
})
```

ALWAYS provide `getRowId` whenever rows have a stable identifier (id, uuid, slug, primary key). NEVER rely on array-index keys for selection that survives a refetch.

---

## 4. `manualPagination` without `pageCount` : infinite "next page"

### Wrong

```tsx
const table = useReactTable({
  data,
  columns,
  manualPagination: true,
  state: { pagination },
  onPaginationChange: setPagination,
  getCoreRowModel: getCoreRowModel(),
  // pageCount missing
})

// table.getPageCount() returns -1
// table.getCanNextPage() returns true forever
```

### Symptom

The "Next" button stays enabled past the actual end of data. Clicking it advances `pageIndex`, the API returns an empty array, the table goes blank, but `Next` is still clickable. Page indicators show `Page 5 of -1`.

### Root cause

When `manualPagination: true` the engine does not know the total. It defaults `pageCount` to `-1`, which means "unknown count." `getCanNextPage()` returns `true` because there is no upper bound to compare against.

### Fix

```tsx
const table = useReactTable({
  data: data?.rows ?? [],
  columns,
  pageCount: data?.pageCount ?? -1,           // total page count from API
  // alternatively: rowCount: data?.total
  manualPagination: true,
  state: { pagination },
  onPaginationChange: setPagination,
  getCoreRowModel: getCoreRowModel(),
})
```

ALWAYS supply EITHER `pageCount` OR `rowCount` whenever `manualPagination: true`. The API response from your server MUST return one of them.

---

## 5. `'use client'` missing on DataTable component : hydration error

### Wrong

```tsx
// components/ui/data-table.tsx (NO 'use client' directive)
import * as React from "react"
import { useReactTable } from "@tanstack/react-table"

export function DataTable<TData>({ ... }) {
  const [sorting, setSorting] = React.useState([])
  const table = useReactTable({ /* ... */ })
  return <Table>{/* ... */}</Table>
}
```

### Symptom (Next.js app router)

Build error : `useState only works in Client Components.` Or runtime error : `Cannot read properties of null (reading 'useReducer')`. Or silent hydration mismatch where the server-rendered headers differ from client-rendered rows.

### Root cause

The TanStack engine (`useReactTable`) calls `useReducer` internally. State (sorting, filters, selection, visibility, pagination) is React state. None of this works in a server component. The Next.js app router defaults every file to "server" until `"use client"` opts it in.

### Fix

```tsx
"use client"                                  // FIRST line of the file

import * as React from "react"
import { useReactTable } from "@tanstack/react-table"

export function DataTable<TData>({ ... }) { /* ... */ }
```

ALWAYS put `"use client"` on `components/ui/data-table.tsx` and on the `columns.tsx` file IF columns use cell-rendering functions that need browser-only APIs (clipboard, dropdowns, dialogs). The PAGE that imports `<DataTable />` may stay a server component.

---

## 6. `ColumnDef` without `accessorKey` or `accessorFn` : cells render undefined

### Wrong

```ts
export const columns: ColumnDef<Payment>[] = [
  { header: "Status" },                       // BUG : no accessor
  { header: "Amount", cell: ({ row }) => row.getValue("amount") },  // BUG : "amount" key does not exist in any column
]
```

### Symptom

Cells render as empty strings or `undefined`. `row.getValue("amount")` returns `undefined`. Console warning : `Column with id "amount" does not exist.` Sorting by these columns does nothing. Filters never match.

### Root cause

Without `accessorKey` or `accessorFn`, TanStack cannot extract the cell value from `TData`. The column becomes a "display column" (which is valid for `select` and `actions` columns) but `row.getValue(id)` has nothing to return for it.

### Fix

For data columns :

```ts
{ accessorKey: "status", header: "Status" }                                       // string key into TData
{ accessorFn: row => row.amount * 1.21, id: "total_incl_vat", header: "Total" }   // computed value
```

For display-only columns (no underlying data) :

```ts
{
  id: "actions",                              // REQUIRED when no accessor : explicit id
  enableHiding: false,
  cell: ({ row }) => <RowActions row={row.original} />,
}
```

Rule : EVERY column needs ONE OF `accessorKey`, `accessorFn`, or an explicit `id`. The first two also make `row.getValue(id)` work ; the last makes the column a display-only column.

---

## Decision rules cheat sheet

| Failure | Rule |
|---------|------|
| Empty body | ALWAYS pass `getCoreRowModel: getCoreRowModel()`. |
| Double-sort, sort flicker | `manualX` flag and `getXRowModel()` are MUTUALLY EXCLUSIVE per feature. |
| Wrong row selected after refetch | ALWAYS pass `getRowId: row => row.id` for stable selection. |
| Infinite "next page" | ALWAYS pass `pageCount` (or `rowCount`) when `manualPagination: true`. |
| Hydration / `useState` error | ALWAYS put `"use client"` on the DataTable component file. |
| Cells render undefined | EVERY column needs `accessorKey`, `accessorFn`, or explicit `id`. |
