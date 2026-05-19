# Table : six anti-patterns with WHY and FIX

Each anti-pattern below has been observed in real shadcn-ui projects (GitHub issues, Discord, code review feedback). Each entry follows the WHY / FIX shape so you can both recognise the smell and apply the correct remediation.

For the positive guidance, see SKILL.md "Five invariants" and the three decision trees.

---

## AP-001 : Using `Table` for sortable / filterable / paginated / selectable data

**Smell** :

```tsx
// WRONG : hand-rolled sort header on the primitive
import { Table, TableHeader, TableRow, TableHead, TableBody, TableCell } from "@/components/ui/table"
import { ChevronsUpDown } from "lucide-react"

function ProductTable({ rows }: { rows: Product[] }) {
  const [sortKey, setSortKey] = useState<"name" | "price">("name")
  const [sortDir, setSortDir] = useState<"asc" | "desc">("asc")

  const sortedRows = [...rows].sort((a, b) => {
    // hand-rolled sort logic ...
  })

  return (
    <Table>
      <TableHeader>
        <TableRow>
          <TableHead onClick={() => setSortKey("name")}>
            Name <ChevronsUpDown />
          </TableHead>
          ...
        </TableRow>
      </TableHeader>
      ...
    </Table>
  )
}
```

**WHY this is wrong** :

This skill (`shadcn-syntax-table`) is the STYLING PRIMITIVE only. It has no sort logic, no filter logic, no pagination logic, no row selection logic. The moment you reach for `useState` + a sort handler on a `<TableHead>`, you are reinventing TanStack Table v8 incorrectly.

Common consequences :

- Sort handlers do not handle locale-aware string compare (your numeric column sorts as text).
- Multi-column sort (Shift-click for secondary sort) is missing.
- Sort direction is not synced to URL or to a parent's `searchParams`, so refreshes lose state.
- Accessible sort announcement (`aria-sort="ascending"`) is not wired.
- Pagination cursor is stuck in component state, lost on navigation.

**FIX** :

ALWAYS switch to the TanStack Table v8 recipe documented in `shadcn-impl-data-table` (the B9 skill). That skill teaches the canonical shape :

```tsx
import { ColumnDef, useReactTable, getCoreRowModel, getSortedRowModel, flexRender } from "@tanstack/react-table"
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from "@/components/ui/table"

// ColumnDef[] declares sort behaviour ; useReactTable returns the table API ;
// flexRender pipes headers + cells through TableHead and TableCell.
```

The recipe COMPOSES this primitive : you still write `<Table>`, `<TableHeader>`, `<TableCell>`, but the surrounding logic lives in `useReactTable`. ALWAYS go through the recipe ; NEVER fork half of it onto the primitive.

---

## AP-002 : Shipping a Table with no accessible name

**Smell** :

```tsx
// WRONG : no TableCaption, no aria-label, no aria-labelledby
<Table>
  <TableHeader>
    <TableRow>
      <TableHead>Name</TableHead>
      <TableHead>Email</TableHead>
    </TableRow>
  </TableHeader>
  <TableBody>...</TableBody>
</Table>
```

**WHY this is wrong** :

Screen readers announce a `<table>` with no caption and no `aria-label` as simply "table". The user has no idea what data they are reading. WCAG 2.1 SC 1.3.1 (Info and Relationships) and SC 2.4.6 (Headings and Labels) both require that tabular data carry a meaningful name.

This anti-pattern is invisible in QA : the table looks fine to sighted developers and never produces a runtime warning. It only surfaces when a screen-reader user encounters it.

**FIX** :

ALWAYS provide exactly one of three accessible-name strategies (see SKILL.md "Accessibility") :

```tsx
// Approach A : visible caption (footnote style)
<Table>
  <TableCaption>Recent invoices, March 2026.</TableCaption>
  ...
</Table>

// Approach B : invisible label (when caption would clutter the design)
<Table aria-label="Recent invoices">
  ...
</Table>

// Approach C : referenced label (when surrounding chrome already names the data)
<Card>
  <CardHeader>
    <CardTitle id="recent-invoices-title">Recent invoices</CardTitle>
  </CardHeader>
  <CardContent className="p-0">
    <Table aria-labelledby="recent-invoices-title">
      ...
    </Table>
  </CardContent>
</Card>
```

NEVER combine approaches : if both `<TableCaption>` and `aria-label` are present, screen readers announce both and the message becomes redundant.

---

## AP-003 : Putting `TableHead` inside `TableBody` (or `TableCell` inside `TableHeader`)

**Smell** :

```tsx
// WRONG : TableHead inside TableBody
<Table>
  <TableBody>
    <TableRow>
      <TableHead>Row label</TableHead>
      <TableCell>data</TableCell>
    </TableRow>
    <TableRow>
      <TableHead>Another label</TableHead>
      <TableCell>data</TableCell>
    </TableRow>
  </TableBody>
</Table>
```

**WHY this is wrong** :

`<TableHead>` renders a `<th>` element. `<TableCell>` renders a `<td>` element. The HTML spec is unambiguous : `<th>` cells belong inside `<thead>` (column headers) OR `<tbody>` AS ROW HEADERS WITH `scope="row"`. Just sticking `<TableHead>` inside `<TableBody>` without `scope="row"` produces a `<th>` that screen readers cannot interpret correctly.

The shadcn `<TableHead>` styling (`h-10 px-2 font-medium text-foreground whitespace-nowrap`) is sized for column titles, not for body rows. Using it as a row label produces visually inconsistent height (the row is taller than its siblings).

**FIX** :

Two correct approaches depending on intent :

```tsx
// Option 1 : it is a column header. Move it to <TableHeader>.
<Table>
  <TableHeader>
    <TableRow>
      <TableHead>Column title</TableHead>
      <TableHead>Other column</TableHead>
    </TableRow>
  </TableHeader>
  <TableBody>
    <TableRow>
      <TableCell>data</TableCell>
      <TableCell>data</TableCell>
    </TableRow>
  </TableBody>
</Table>

// Option 2 : it is a ROW header (left column labels each row).
// Use a styled <TableCell> with font-medium ; do NOT use <TableHead>.
<Table>
  <TableBody>
    <TableRow>
      <TableCell className="font-medium">Row label</TableCell>
      <TableCell>data</TableCell>
    </TableRow>
  </TableBody>
</Table>
// If you need TRUE row-header semantics (screen readers announce "row header"),
// drop to the native HTML : <th scope="row">Row label</th> inside <TableRow>.
```

ALWAYS keep `<TableHead>` confined to `<TableHeader>`. NEVER let a single `<TableHead>` leak into `<TableBody>` ; either move it to the header or replace it with `<TableCell className="font-medium">`.

---

## AP-004 : Hand-rolling sort-header chevrons that toggle on click without TanStack integration

**Smell** :

```tsx
// WRONG : custom sort chevron with local state
import { ChevronUp, ChevronDown } from "lucide-react"

function SortableHeader({ label, sortKey, currentSort, onSort }: {...}) {
  const direction = currentSort.key === sortKey ? currentSort.dir : null
  return (
    <TableHead>
      <button onClick={() => onSort(sortKey)} className="flex items-center gap-2">
        {label}
        {direction === "asc" && <ChevronUp />}
        {direction === "desc" && <ChevronDown />}
      </button>
    </TableHead>
  )
}
```

**WHY this is wrong** :

This is a special case of AP-001. The visible smell is the chevron and the click handler. The hidden smell is everything you DID NOT implement : `aria-sort` on the `<th>` (so screen readers announce "sorted ascending"), keyboard activation (Enter / Space on the button to sort), multi-column sort (Shift-click for secondary), sort-direction round-tripping through URL `searchParams`, and the locale-aware compare function.

TanStack Table v8 wires ALL of this for you via `column.getCanSort()`, `column.getToggleSortingHandler()`, `column.getIsSorted()`, and the `sortingFns` library (text, alphanumeric, basic, datetime).

**FIX** :

ALWAYS use the TanStack recipe :

```tsx
// In shadcn-impl-data-table : the canonical sort-header pattern
const columns: ColumnDef<Product>[] = [
  {
    accessorKey: "name",
    header: ({ column }) => (
      <Button
        variant="ghost"
        onClick={() => column.toggleSorting(column.getIsSorted() === "asc")}
      >
        Name
        <ArrowUpDown className="ml-2 size-4" />
      </Button>
    ),
  },
]
```

The recipe handles `aria-sort`, keyboard activation, sort-fn selection, and state. NEVER reinvent it on the primitive. If your project is using this skill (`shadcn-syntax-table`) for sort-headers, STOP and migrate to `shadcn-impl-data-table`.

---

## AP-005 : Storing pagination state inside the primitive

**Smell** :

```tsx
// WRONG : pagination logic glued to the styling primitive
function PaginatedTable({ allRows }: { allRows: Row[] }) {
  const [page, setPage] = useState(0)
  const pageSize = 10
  const visible = allRows.slice(page * pageSize, (page + 1) * pageSize)

  return (
    <>
      <Table>
        <TableBody>
          {visible.map((r) => <TableRow key={r.id}><TableCell>{r.name}</TableCell></TableRow>)}
        </TableBody>
      </Table>
      <div className="flex justify-end gap-2 mt-4">
        <Button onClick={() => setPage((p) => Math.max(0, p - 1))}>Prev</Button>
        <Button onClick={() => setPage((p) => p + 1)}>Next</Button>
      </div>
    </>
  )
}
```

**WHY this is wrong** :

The Table primitive has NO concept of pagination. The moment you wire `useState` + a `slice()` window + Prev/Next buttons around it, you are building half of `getPaginationRowModel()` by hand and missing critical features :

- Page size selector ("Rows per page : 10 / 25 / 50 / 100").
- "Page 1 of 12" indicator.
- Disabled state on Prev when at page 0 / Next when at last page.
- Total row count display.
- URL-synced page (refresh-resilient pagination).
- Server-side pagination handshake (when the data set exceeds memory).

Worse : if you later need to also sort or filter, the slice() interacts incorrectly with the sort (you sort the visible page, not the full dataset) and you get rows-jumping-between-pages bugs.

**FIX** :

ALWAYS use TanStack Table v8's `getPaginationRowModel()` via `shadcn-impl-data-table`. The recipe handles all the above and exposes `table.getCanPreviousPage()`, `table.getCanNextPage()`, `table.previousPage()`, `table.nextPage()`, `table.setPageSize()` as a single coherent API.

If the dataset is small (under ~50 rows) and pagination is not actually needed for usability, ALWAYS prefer rendering all rows in this primitive with no pagination at all. NEVER add pagination "just in case" ; it adds UI complexity that real users do not need.

---

## AP-006 : Wrapping `<Table>` in your own overflow container

**Smell** :

```tsx
// WRONG : double scroll container
<div className="overflow-x-auto rounded-md border">
  <Table>
    <TableHeader>...</TableHeader>
    <TableBody>...</TableBody>
  </Table>
</div>
```

**WHY this is wrong** :

The shadcn `<Table>` ALREADY wraps the native `<table>` in `<div data-slot="table-container" className="relative w-full overflow-x-auto">` (verbatim v4 source). Adding your own `overflow-x-auto` div on the outside creates a nested scroll container : on narrow viewports the user gets two horizontal scrollbars stacked, the inner one inside the outer one, both moving the same content.

The visible symptom : when the user scrolls horizontally, sometimes only one scrollbar moves, sometimes both, the user is confused, and on touch devices the gesture handling becomes inconsistent.

**FIX** :

ALWAYS apply layout / overflow / border / rounded utilities DIRECTLY to `<Table>` via `className`. The `cn()` call inside the primitive routes them through twMerge so they merge correctly with the existing classes.

```tsx
// CORRECT : single scroll container, border on the inner <table>
<Table className="rounded-md border">
  <TableHeader>...</TableHeader>
  <TableBody>...</TableBody>
</Table>
```

If you need a border around the OUTER container (the scroll container div), the shadcn primitive does not expose that directly. The accepted pattern is to copy `components/ui/table.tsx` and edit the wrapper div's className. Remember : copy-not-install means you ALWAYS own the source ; edit it freely. NEVER edit `node_modules` ; the file does not live there.

NEVER wrap `<Table>` in your own `overflow-x-auto` div. NEVER wrap it in your own `overflow-auto` div. NEVER wrap it in your own `max-h-*` + `overflow-y-auto` div hoping for vertical sticky-header behaviour ; sticky headers require `<thead>` to be `position: sticky`, which is a separate concern not handled by either the primitive or the DataTable recipe out of the box. (For sticky table headers, see the upstream Tailwind documentation on `sticky top-0` applied to `<TableHeader>`.)

---

## Quick lookup table

| Anti-pattern | Trigger | Fix |
|--------------|---------|-----|
| AP-001 | Hand-rolled sort / filter / paginate / select on this primitive | Use `shadcn-impl-data-table` (TanStack recipe) |
| AP-002 | No TableCaption AND no aria-label AND no aria-labelledby | Add exactly one of the three |
| AP-003 | TableHead in TableBody, or TableCell in TableHeader | Move TableHead to TableHeader OR use TableCell font-medium for row labels |
| AP-004 | Custom chevron sort-button on TableHead with local state | Switch to TanStack column.getToggleSortingHandler() via shadcn-impl-data-table |
| AP-005 | useState page + slice() pagination glued to Table | Switch to TanStack getPaginationRowModel() via shadcn-impl-data-table |
| AP-006 | Outer overflow-x-auto div wrapping `<Table>` | Apply className directly to `<Table>` ; the primitive already includes its own scroll container |

When in doubt : if the table needs ANY interactivity beyond hover, you are out of scope for this skill. Read `shadcn-impl-data-table` instead.
