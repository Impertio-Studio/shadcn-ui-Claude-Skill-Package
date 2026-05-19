# DataTable Examples

Six worked recipes. Every example assumes `npx shadcn@latest add table` was already run and `@tanstack/react-table` is installed.

---

## Example 1 : Minimal sortable DataTable

`components/columns.tsx` :

```tsx
"use client"

import type { ColumnDef } from "@tanstack/react-table"
import { ArrowUpDown } from "lucide-react"
import { Button } from "@/components/ui/button"

export type Payment = {
  id: string
  amount: number
  status: "pending" | "processing" | "success" | "failed"
  email: string
}

export const columns: ColumnDef<Payment>[] = [
  { accessorKey: "status", header: "Status" },
  {
    accessorKey: "email",
    header: ({ column }) => (
      <Button variant="ghost" onClick={() => column.toggleSorting(column.getIsSorted() === "asc")}>
        Email <ArrowUpDown className="ml-2 h-4 w-4" />
      </Button>
    ),
  },
  {
    accessorKey: "amount",
    header: () => <div className="text-right">Amount</div>,
    cell: ({ row }) => {
      const amount = parseFloat(row.getValue("amount"))
      return <div className="text-right font-medium">{new Intl.NumberFormat("en-US", { style: "currency", currency: "USD" }).format(amount)}</div>
    },
  },
]
```

`components/data-table.tsx` :

```tsx
"use client"

import * as React from "react"
import {
  type ColumnDef, type SortingState,
  flexRender, getCoreRowModel, getSortedRowModel, useReactTable,
} from "@tanstack/react-table"
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from "@/components/ui/table"

interface DataTableProps<TData, TValue> {
  columns: ColumnDef<TData, TValue>[]
  data: TData[]
}

export function DataTable<TData, TValue>({ columns, data }: DataTableProps<TData, TValue>) {
  const [sorting, setSorting] = React.useState<SortingState>([])
  const table = useReactTable({
    data, columns,
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
    onSortingChange: setSorting,
    state: { sorting },
  })

  return (
    <div className="rounded-md border">
      <Table>
        <TableHeader>
          {table.getHeaderGroups().map(hg => (
            <TableRow key={hg.id}>
              {hg.headers.map(h => <TableHead key={h.id}>{h.isPlaceholder ? null : flexRender(h.column.columnDef.header, h.getContext())}</TableHead>)}
            </TableRow>
          ))}
        </TableHeader>
        <TableBody>
          {table.getRowModel().rows.length ? table.getRowModel().rows.map(row => (
            <TableRow key={row.id}>
              {row.getVisibleCells().map(cell => <TableCell key={cell.id}>{flexRender(cell.column.columnDef.cell, cell.getContext())}</TableCell>)}
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

---

## Example 2 : DataTable with filter + pagination

```tsx
"use client"

import * as React from "react"
import {
  type ColumnDef, type ColumnFiltersState, type SortingState,
  flexRender, getCoreRowModel, getFilteredRowModel,
  getPaginationRowModel, getSortedRowModel, useReactTable,
} from "@tanstack/react-table"
import { Button } from "@/components/ui/button"
import { Input } from "@/components/ui/input"
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from "@/components/ui/table"

export function DataTable<TData, TValue>({ columns, data }: { columns: ColumnDef<TData, TValue>[]; data: TData[] }) {
  const [sorting, setSorting] = React.useState<SortingState>([])
  const [columnFilters, setColumnFilters] = React.useState<ColumnFiltersState>([])

  const table = useReactTable({
    data, columns,
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
    getFilteredRowModel: getFilteredRowModel(),
    getPaginationRowModel: getPaginationRowModel(),
    onSortingChange: setSorting,
    onColumnFiltersChange: setColumnFilters,
    state: { sorting, columnFilters },
    initialState: { pagination: { pageSize: 10 } },
  })

  return (
    <div>
      <div className="flex items-center py-4">
        <Input
          placeholder="Filter emails..."
          value={(table.getColumn("email")?.getFilterValue() as string) ?? ""}
          onChange={e => table.getColumn("email")?.setFilterValue(e.target.value)}
          className="max-w-sm"
        />
      </div>
      <div className="rounded-md border">
        <Table>
          {/* header + body same as Example 1 */}
        </Table>
      </div>
      <div className="flex items-center justify-end space-x-2 py-4">
        <Button variant="outline" size="sm" onClick={() => table.previousPage()} disabled={!table.getCanPreviousPage()}>Previous</Button>
        <Button variant="outline" size="sm" onClick={() => table.nextPage()} disabled={!table.getCanNextPage()}>Next</Button>
      </div>
    </div>
  )
}
```

ALWAYS read the current filter value via `table.getColumn("email")?.getFilterValue()` and cast to `string` ; the engine stores `unknown`.

---

## Example 3 : DataTable with row selection + bulk actions

```tsx
"use client"

import * as React from "react"
import {
  type ColumnDef, flexRender, getCoreRowModel, useReactTable,
} from "@tanstack/react-table"
import { Checkbox } from "@/components/ui/checkbox"
import { Button } from "@/components/ui/button"

export type User = { id: string; name: string; email: string }

export const userColumns: ColumnDef<User>[] = [
  {
    id: "select",
    header: ({ table }) => (
      <Checkbox
        checked={table.getIsAllPageRowsSelected() || (table.getIsSomePageRowsSelected() && "indeterminate")}
        onCheckedChange={value => table.toggleAllPageRowsSelected(!!value)}
        aria-label="Select all"
      />
    ),
    cell: ({ row }) => (
      <Checkbox
        checked={row.getIsSelected()}
        onCheckedChange={value => row.toggleSelected(!!value)}
        aria-label="Select row"
      />
    ),
    enableSorting: false,
    enableHiding: false,
  },
  { accessorKey: "name", header: "Name" },
  { accessorKey: "email", header: "Email" },
]

export function UserTable({ data }: { data: User[] }) {
  const [rowSelection, setRowSelection] = React.useState({})

  const table = useReactTable({
    data,
    columns: userColumns,
    getCoreRowModel: getCoreRowModel(),
    enableRowSelection: true,
    getRowId: row => row.id,                       // CRITICAL : prevents index-key collisions
    onRowSelectionChange: setRowSelection,
    state: { rowSelection },
  })

  const selectedIds = table.getSelectedRowModel().rows.map(r => r.original.id)

  async function bulkDelete() {
    if (!selectedIds.length) return
    await fetch("/api/users/bulk-delete", { method: "POST", body: JSON.stringify({ ids: selectedIds }) })
    setRowSelection({})
  }

  return (
    <div>
      {selectedIds.length > 0 && (
        <div className="flex items-center gap-2 py-2">
          <span className="text-sm text-muted-foreground">{selectedIds.length} selected</span>
          <Button variant="destructive" size="sm" onClick={bulkDelete}>Delete selected</Button>
        </div>
      )}
      {/* render Table as in Example 1 */}
    </div>
  )
}
```

ALWAYS set `getRowId: row => row.id` whenever rows have a stable identifier ; otherwise selection state keys on array index and breaks on reorder/refetch.

---

## Example 4 : Server-side data (Next.js) with pageCount

```tsx
"use client"

import * as React from "react"
import { useQuery, keepPreviousData } from "@tanstack/react-query"
import {
  type ColumnDef, type SortingState, type ColumnFiltersState, type PaginationState,
  flexRender, getCoreRowModel, useReactTable,
} from "@tanstack/react-table"

type ApiResponse<T> = { rows: T[]; pageCount: number }

export function ServerDataTable<TData>({ columns, fetchUrl }: { columns: ColumnDef<TData>[]; fetchUrl: string }) {
  const [sorting, setSorting] = React.useState<SortingState>([])
  const [columnFilters, setColumnFilters] = React.useState<ColumnFiltersState>([])
  const [pagination, setPagination] = React.useState<PaginationState>({ pageIndex: 0, pageSize: 10 })

  const { data } = useQuery({
    queryKey: ["table", fetchUrl, sorting, columnFilters, pagination],
    queryFn: async () => {
      const params = new URLSearchParams({
        page: String(pagination.pageIndex + 1),
        pageSize: String(pagination.pageSize),
        sort: sorting.map(s => `${s.desc ? "-" : ""}${s.id}`).join(","),
        filters: JSON.stringify(columnFilters),
      })
      const res = await fetch(`${fetchUrl}?${params}`)
      return (await res.json()) as ApiResponse<TData>
    },
    placeholderData: keepPreviousData,
  })

  const table = useReactTable({
    data: data?.rows ?? [],
    columns,
    pageCount: data?.pageCount ?? -1,            // -1 falls back to "unknown" UI ; supply real count when available
    state: { sorting, columnFilters, pagination },
    onSortingChange: setSorting,
    onColumnFiltersChange: setColumnFilters,
    onPaginationChange: setPagination,
    manualSorting: true,
    manualFiltering: true,
    manualPagination: true,
    getCoreRowModel: getCoreRowModel(),
  })

  // render Table + pagination buttons as in Example 2
  return null
}
```

NEVER add `getSortedRowModel()` or `getFilteredRowModel()` or `getPaginationRowModel()` here. The engine would re-sort the already-sorted server response.

---

## Example 5 : Column visibility toggle (DropdownMenu)

```tsx
"use client"

import * as React from "react"
import {
  type ColumnDef, type VisibilityState,
  flexRender, getCoreRowModel, useReactTable,
} from "@tanstack/react-table"
import { Button } from "@/components/ui/button"
import {
  DropdownMenu, DropdownMenuCheckboxItem,
  DropdownMenuContent, DropdownMenuTrigger,
} from "@/components/ui/dropdown-menu"

export function VisibilityTable<TData>({ columns, data }: { columns: ColumnDef<TData>[]; data: TData[] }) {
  const [columnVisibility, setColumnVisibility] = React.useState<VisibilityState>({})

  const table = useReactTable({
    data, columns,
    getCoreRowModel: getCoreRowModel(),
    onColumnVisibilityChange: setColumnVisibility,
    state: { columnVisibility },
  })

  return (
    <div>
      <DropdownMenu>
        <DropdownMenuTrigger asChild>
          <Button variant="outline" className="ml-auto">Columns</Button>
        </DropdownMenuTrigger>
        <DropdownMenuContent align="end">
          {table.getAllColumns().filter(c => c.getCanHide()).map(column => (
            <DropdownMenuCheckboxItem
              key={column.id}
              className="capitalize"
              checked={column.getIsVisible()}
              onCheckedChange={value => column.toggleVisibility(!!value)}
            >
              {column.id}
            </DropdownMenuCheckboxItem>
          ))}
        </DropdownMenuContent>
      </DropdownMenu>
      {/* render Table here */}
    </div>
  )
}
```

ALWAYS filter by `column.getCanHide()` so the `select` and `actions` columns (which set `enableHiding: false`) are not listed.

---

## Example 6 : DataTable with custom cell renderers + row actions

```tsx
"use client"

import { type ColumnDef } from "@tanstack/react-table"
import { MoreHorizontal } from "lucide-react"
import { Badge } from "@/components/ui/badge"
import { Button } from "@/components/ui/button"
import {
  DropdownMenu, DropdownMenuContent, DropdownMenuItem,
  DropdownMenuLabel, DropdownMenuSeparator, DropdownMenuTrigger,
} from "@/components/ui/dropdown-menu"

export type Order = { id: string; total: number; status: "paid" | "refunded" | "pending" }

export const orderColumns: ColumnDef<Order>[] = [
  {
    accessorKey: "status",
    header: "Status",
    cell: ({ row }) => {
      const status = row.getValue<Order["status"]>("status")
      const variant = status === "paid" ? "default" : status === "refunded" ? "destructive" : "secondary"
      return <Badge variant={variant}>{status}</Badge>
    },
  },
  {
    accessorKey: "total",
    header: "Total",
    cell: ({ row }) => {
      const total = parseFloat(row.getValue("total"))
      return new Intl.NumberFormat("en-US", { style: "currency", currency: "USD" }).format(total)
    },
  },
  {
    id: "actions",
    enableHiding: false,
    cell: ({ row }) => {
      const order = row.original
      return (
        <DropdownMenu>
          <DropdownMenuTrigger asChild>
            <Button variant="ghost" className="h-8 w-8 p-0">
              <span className="sr-only">Open menu</span>
              <MoreHorizontal className="h-4 w-4" />
            </Button>
          </DropdownMenuTrigger>
          <DropdownMenuContent align="end">
            <DropdownMenuLabel>Actions</DropdownMenuLabel>
            <DropdownMenuItem onClick={() => navigator.clipboard.writeText(order.id)}>Copy order ID</DropdownMenuItem>
            <DropdownMenuSeparator />
            <DropdownMenuItem>View customer</DropdownMenuItem>
            <DropdownMenuItem>View order details</DropdownMenuItem>
          </DropdownMenuContent>
        </DropdownMenu>
      )
    },
  },
]
```

The `actions` column uses `id: "actions"` (no `accessorKey`), so it MUST set `id` explicitly and SHOULD set `enableHiding: false`.
