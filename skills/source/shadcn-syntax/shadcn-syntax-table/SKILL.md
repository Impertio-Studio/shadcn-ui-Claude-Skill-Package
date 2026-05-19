---
name: shadcn-syntax-table
description: >
  Use when adding, customising, or debugging the shadcn ui Table primitive
  (the eight subcomponents Table, TableHeader, TableBody, TableFooter,
  TableRow, TableHead, TableCell, TableCaption), rendering a static,
  read-only table for product lists, invoice line-items, settings rows,
  metric summaries, or any tabular data that does NOT need sorting,
  filtering, pagination, or row selection, deciding between the styling
  primitive (this skill) and the TanStack-Table-driven DataTable recipe
  (shadcn-impl-data-table), composing a TableFooter sum row, adding a
  TableCaption for screen-reader access, dropping a shadcn Table inside
  a Card / Dialog / Drawer / Sheet surface, or diagnosing why a Table
  renders unstyled, breaks alignment, or refuses to scroll horizontally
  on mobile.
  Prevents the canonical Table-primitive failures : reaching for the
  Table primitive when the design actually needs sortable / filterable
  / paginated columns (those require the TanStack recipe in
  shadcn-impl-data-table, NOT this skill), shipping a Table with no
  TableCaption / no aria-label so screen readers announce only "table"
  with no purpose, putting TableHead cells inside TableBody (head cells
  belong in thead, not in tbody), hand-rolling sort-header chevrons that
  flip on click without wiring through TanStack getSortedRowModel(),
  and storing pagination cursors in component state inside this
  primitive (pagination is a recipe concern, not a primitive concern).
  Covers the eight subcomponents verbatim from the canonical v4 registry
  source (apps/v4/registry/new-york-v4/ui/table.tsx), the v3 vs v4
  source differences (forwardRef removed, data-slot attributes added,
  has-aria-expanded row state, TableHead h-12 -> h-10, TableCell p-4
  -> p-2, overflow-auto -> overflow-x-auto, mandatory "use client" in v4),
  the caption-bottom default and how to flip to caption-top, the
  Table-vs-DataTable decision tree, RSC-compatibility nuances (v4 file
  is marked "use client" even though it has no hooks ; that affects
  how you import it from a server component), and the composition
  pattern for embedding a Table inside Card or Dialog.
  Keywords: shadcn table, html table styling, Table component, TableHeader,
  TableBody, TableFooter, TableRow, TableHead, TableCell, TableCaption,
  simple table, static data table, read-only table, product table,
  invoice table, settings table, sum row footer, caption-bottom,
  caption-top, table styling, table inside card, table in dialog,
  Table vs DataTable, when not to use DataTable, when to use TanStack
  Table, sortable table, filterable table, paginated table, row selection,
  how do I style a table, how do I make a simple table, table overflow,
  horizontal scroll table, mobile table, screen reader table,
  table accessibility, table aria-label, data-slot table.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui : Table Syntax (HTML primitive)

Table is shadcn's STYLING PRIMITIVE for HTML tables. It is NOT a data grid. It does NOT sort, filter, paginate, or select rows. It is eight thin wrappers around `<table>`, `<thead>`, `<tbody>`, `<tfoot>`, `<tr>`, `<th>`, `<td>`, `<caption>` with cohesive Tailwind classes and `data-slot` attributes.

For anything that requires sorting, filtering, pagination, or selection, you ALWAYS use the TanStack Table v8 recipe documented in `shadcn-impl-data-table`. That recipe COMPOSES this primitive ; it does NOT replace it.

## Quick Reference

### Canonical call

```tsx
import {
  Table,
  TableBody,
  TableCaption,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from "@/components/ui/table"

<Table>
  <TableCaption>A list of recent invoices.</TableCaption>
  <TableHeader>
    <TableRow>
      <TableHead>Invoice</TableHead>
      <TableHead>Status</TableHead>
      <TableHead className="text-right">Amount</TableHead>
    </TableRow>
  </TableHeader>
  <TableBody>
    <TableRow>
      <TableCell>INV001</TableCell>
      <TableCell>Paid</TableCell>
      <TableCell className="text-right">EUR 250.00</TableCell>
    </TableRow>
  </TableBody>
</Table>
```

`Table` wraps the native `<table>` element in a `<div data-slot="table-container">` with `overflow-x-auto`, so horizontal scroll on narrow viewports is automatic. NEVER wrap Table in your own overflow container ; you will end up with two scrollbars.

### The eight subcomponents

| Subcomponent | Wraps | Required position | Purpose |
|--------------|-------|-------------------|---------|
| `Table` | `<table>` inside a scroll container `<div>` | Root of every table | Outer wrapper + horizontal scroll |
| `TableHeader` | `<thead>` | Direct child of `Table` | Column titles row(s) |
| `TableBody` | `<tbody>` | Direct child of `Table` | Data rows |
| `TableFooter` | `<tfoot>` | Direct child of `Table` (after body) | Summary / total row(s) |
| `TableRow` | `<tr>` | Inside `TableHeader`, `TableBody`, or `TableFooter` | One row |
| `TableHead` | `<th>` | Inside a `TableRow` that is inside `TableHeader` | One column-title cell |
| `TableCell` | `<td>` | Inside a `TableRow` that is inside `TableBody` or `TableFooter` | One data cell |
| `TableCaption` | `<caption>` | Direct child of `Table`, semantically a sibling of header / body | Accessible table description |

### Five invariants

1. ALWAYS import from the local alias `@/components/ui/table`. NEVER from `shadcn-ui` or `@shadcn/ui` ; shadcn is copy-not-install (see `shadcn-core-architecture`).
2. ALWAYS use `TableHead` for column titles INSIDE `TableHeader`, and `TableCell` for data INSIDE `TableBody`. NEVER swap : putting `TableHead` inside `TableBody` produces invalid HTML and breaks screen-reader column-header announcement.
3. ALWAYS provide an accessible name : either `TableCaption` (visible) or `aria-label` on `<Table>` (invisible). NEVER ship a Table with no name ; screen readers announce only "table" with no purpose.
4. ALWAYS use this skill ONLY for static / read-only data. If the design needs sortable headers, search-filter, pagination, or row selection, STOP and switch to `shadcn-impl-data-table` (the TanStack Table v8 recipe).
5. ALWAYS keep TableFooter AFTER TableBody (the source order). The DOM order is `<thead>` -> `<tbody>` -> `<tfoot>`, even though `<tfoot>` is sometimes positioned visually at the bottom by default browser styling regardless.

## Decision Tree 1 : Table primitive or DataTable recipe?

This is the single most important decision the skill makes. Get it wrong and you ship the wrong tool.

```
Q1. Do users need to SORT columns by clicking a header (asc / desc toggle)?
    yes -> shadcn-impl-data-table (TanStack getSortedRowModel)
    no  -> Q2

Q2. Do users need to FILTER rows via a search input or per-column filter UI?
    yes -> shadcn-impl-data-table (TanStack getFilteredRowModel)
    no  -> Q3

Q3. Does the data set exceed ~50 rows AND need pagination
    (prev / next, page-size selector, "1 of N" indicator)?
    yes -> shadcn-impl-data-table (TanStack getPaginationRowModel)
    no  -> Q4

Q4. Do users need to SELECT rows (checkboxes, bulk actions, selection state)?
    yes -> shadcn-impl-data-table (TanStack rowSelection)
    no  -> Q5

Q5. Do users need column-visibility toggles, column-resizing,
    or column-reordering?
    yes -> shadcn-impl-data-table (TanStack columnVisibility, etc.)
    no  -> shadcn-syntax-table (THIS skill) ; you have a static table
```

If even one of Q1 through Q5 is "yes", you ALWAYS use the DataTable recipe. The TanStack recipe COMPOSES this primitive ; you still write `<Table>`, `<TableHeader>`, `<TableCell>` etc., but the surrounding logic lives in the recipe. Mixing the two manually (e.g. hand-rolling sort chevrons on a Table primitive) is an anti-pattern (see `references/anti-patterns.md` AP-001).

## Decision Tree 2 : TableCaption or aria-label?

```
Q1. Is the table embedded in a larger surface (Card, Dialog, Drawer, Sheet)
    that already names the data (CardTitle "Recent invoices",
    DialogTitle "Pending orders")?
    yes -> aria-labelledby on <Table> pointing to the existing title id
    no  -> Q2

Q2. Is a visible caption acceptable in the design
    (small muted text under the table)?
    yes -> <TableCaption>A list of recent invoices.</TableCaption>
    no  -> aria-label="Recent invoices" on <Table> (invisible but announced)
```

ALWAYS pick exactly one. NEVER use both TableCaption AND aria-label : screen readers announce both and the message becomes redundant.

## Decision Tree 3 : Where does the caption visually sit?

The default v4 class on `<table>` is `caption-bottom`. The native `<caption>` defaults to top. shadcn flips it to bottom because the bottom position reads as "footnote" (less visually heavy than a heading-style top caption).

```
Q1. Do you want the caption at the bottom (shadcn default)?
    yes -> <Table><TableCaption>...</TableCaption>...</Table>
           (no overrides needed)
    no  -> Q2

Q2. Do you want the caption at the top (native HTML default)?
    yes -> <Table className="caption-top">
             <TableCaption>...</TableCaption>
             ...
           </Table>
```

The Tailwind `caption-side` utility (`caption-top` / `caption-bottom`) controls this. NEVER set `caption-side` on the `<TableCaption>` itself ; the utility belongs on the `<table>` (which is what `<Table>` renders). Routing `className="caption-top"` through `<Table>` passes the class through `cn()` and twMerge correctly overrides the default.

## Section : The eight subcomponents in detail

### `Table`

Wraps `<table>` in a scroll container. Verbatim v4 (`apps/v4/registry/new-york-v4/ui/table.tsx`) :

```tsx
<div data-slot="table-container" className="relative w-full overflow-x-auto">
  <table data-slot="table" className={cn("w-full caption-bottom text-sm", className)} {...props} />
</div>
```

Two `data-slot` hooks (`table-container` and `table`) give you CSS handles without target-selector gymnastics. ALWAYS apply layout / overflow overrides to `<Table>` itself ; the wrapper `<div>` will inherit appropriately. NEVER target the wrapper `<div>` from outside ; it is an implementation detail.

v3 difference : the v3 source uses `overflow-auto` (both axes), the v4 source uses `overflow-x-auto` (horizontal only). v4 also has no `forwardRef` (React 19 forwards refs implicitly) ; v3 wraps in `React.forwardRef<HTMLTableElement, ...>`.

### `TableHeader`

```tsx
<thead data-slot="table-header" className={cn("[&_tr]:border-b", className)} {...props} />
```

A `<thead>`. Bottom border on every descendant `<tr>` for separation from the body. Position : direct child of `<Table>`, before `<TableBody>`.

### `TableBody`

```tsx
<tbody data-slot="table-body" className={cn("[&_tr:last-child]:border-0", className)} {...props} />
```

A `<tbody>`. Strips the bottom border on the LAST data row so the body does not double-border against the footer or page edge. Position : direct child of `<Table>`, after `<TableHeader>`.

### `TableFooter`

```tsx
<tfoot data-slot="table-footer" className={cn("border-t bg-muted/50 font-medium [&>tr]:last:border-b-0", className)} {...props} />
```

A `<tfoot>`. Muted background + medium font weight signals "summary row". Position : direct child of `<Table>`, after `<TableBody>` in source order.

ALWAYS use `<TableFooter>` for sum rows / total rows / status summaries. NEVER put a "Total" row inside `<TableBody>` ; the muted-background distinction is lost and semantics break.

### `TableRow`

v4 source :

```tsx
<tr data-slot="table-row" className={cn("border-b transition-colors hover:bg-muted/50 has-aria-expanded:bg-muted/50 data-[state=selected]:bg-muted", className)} {...props} />
```

Bottom border, hover background, `has-aria-expanded` background (when a descendant element has `aria-expanded="true"` , e.g. an expanded inline-edit menu), and `data-state=selected` background (the DataTable recipe sets this when a row is selected). v3 source is identical EXCEPT it lacks `has-aria-expanded:bg-muted/50`.

The `data-state=selected` selector exists in BOTH v3 and v4. This skill does NOT set `data-state` ; that is the DataTable recipe's job. The primitive merely exposes the styling hook.

### `TableHead`

v4 source :

```tsx
<th data-slot="table-head" className={cn("h-10 px-2 text-left align-middle font-medium whitespace-nowrap text-foreground [&:has([role=checkbox])]:pr-0 [&>[role=checkbox]]:translate-y-[2px]", className)} {...props} />
```

v3 difference : `h-12 px-4` (taller and wider) instead of v4's `h-10 px-2`, and `text-muted-foreground` instead of `text-foreground` (so v3 column titles are dimmer than v4). The `[&:has([role=checkbox])]:pr-0` and `[&>[role=checkbox]]:translate-y-[2px]` selectors exist in BOTH, anticipating a Checkbox inside the header for "select all".

Position : ALWAYS inside a `<TableRow>` that is inside `<TableHeader>`. NEVER inside `<TableBody>`.

### `TableCell`

v4 source :

```tsx
<td data-slot="table-cell" className={cn("p-2 align-middle whitespace-nowrap [&:has([role=checkbox])]:pr-0 [&>[role=checkbox]]:translate-y-[2px]", className)} {...props} />
```

v3 difference : `p-4` (much more padding) instead of v4's `p-2`. The checkbox selectors are identical to TableHead.

Position : ALWAYS inside a `<TableRow>` that is inside `<TableBody>` or `<TableFooter>`. NEVER inside `<TableHeader>` (use `<TableHead>` there).

### `TableCaption`

```tsx
<caption data-slot="table-caption" className={cn("mt-4 text-sm text-muted-foreground", className)} {...props} />
```

A `<caption>`. Small muted text positioned by the parent table's `caption-side` (default `caption-bottom`). Position : direct child of `<Table>`. Source order does not strictly matter for HTML caption, but place it first for readability.

## Section : Accessibility

Tables MUST have an accessible name. Without one, screen readers announce only "table" and the user has no idea what they are reading.

The two valid approaches :

```tsx
// Approach A : visible caption (preferred when caption-as-footnote suits the design)
<Table>
  <TableCaption>Recent invoices, March 2026.</TableCaption>
  <TableHeader>...</TableHeader>
  <TableBody>...</TableBody>
</Table>

// Approach B : invisible label (preferred when surrounding chrome already names the data)
<Table aria-label="Recent invoices">
  <TableHeader>...</TableHeader>
  <TableBody>...</TableBody>
</Table>

// Approach C : referenced label (preferred when a CardTitle / DialogTitle already exists)
<Card>
  <CardHeader>
    <CardTitle id="recent-invoices-title">Recent invoices</CardTitle>
  </CardHeader>
  <CardContent>
    <Table aria-labelledby="recent-invoices-title">
      <TableHeader>...</TableHeader>
      <TableBody>...</TableBody>
    </Table>
  </CardContent>
</Card>
```

ALWAYS pick exactly one of A, B, or C. NEVER combine them ; redundancy clutters the announcement.

`TableHeader` `<th>` cells are AUTOMATICALLY announced as column headers by screen readers (that is the entire point of `<th>` vs `<td>`). The skill does NOT need to add explicit `scope="col"` for a simple single-row header ; the native `<th>` inside `<thead>` is sufficient. For more complex header structures (multi-row headers, row-group headers), add `scope="col"` / `scope="row"` / `scope="colgroup"` / `scope="rowgroup"` as needed ; native HTML semantics, not a shadcn concern.

## Section : RSC compatibility (the v4 quirk)

The v4 source file ships with a `"use client"` directive on line 1 (verified at https://github.com/shadcn-ui/ui/blob/main/apps/v4/registry/new-york-v4/ui/table.tsx). This is unusual : the file has no hooks, no event handlers, no client-only React features. Why does it carry the directive?

Because shadcn ships a SINGLE source file for both server and client use, and the `"use client"` directive provides a safe upper bound : you can ALWAYS import Table into a client component, and importing into a server component still works (RSC simply treats the Table chunk as a client chunk).

Implication : in a Next.js App Router project, importing `<Table>` into a server-only file causes the Table primitive to be bundled in the client chunk. For static content (a marketing-page invoice table) this is wasted bytes. ALWAYS-allowed workaround : copy the source into your project and DELETE the `"use client"` line. The component still works in both contexts (it has nothing client-only inside), and you save the client-chunk allocation.

NEVER edit `node_modules` to remove the directive ; shadcn is copy-not-install precisely so you can edit the file. Edit `components/ui/table.tsx` in your repo.

v3 difference : the v3 source has NO `"use client"` directive. v3 Table is unambiguously RSC-safe.

## Section : Common compositions

### Table inside Card

```tsx
<Card>
  <CardHeader>
    <CardTitle>Recent invoices</CardTitle>
    <CardDescription>Last 7 days.</CardDescription>
  </CardHeader>
  <CardContent className="p-0">
    <Table aria-labelledby="...">
      <TableHeader>...</TableHeader>
      <TableBody>...</TableBody>
    </Table>
  </CardContent>
</Card>
```

ALWAYS set `className="p-0"` on `<CardContent>` when embedding a Table : default CardContent padding combines with Table padding and the result is over-padded. The Table's internal `p-2` (v4) or `p-4` (v3) cell padding is sufficient.

### Table with TableFooter sum row

```tsx
<Table>
  <TableHeader>
    <TableRow>
      <TableHead>Line</TableHead>
      <TableHead className="text-right">Amount</TableHead>
    </TableRow>
  </TableHeader>
  <TableBody>
    <TableRow><TableCell>Item A</TableCell><TableCell className="text-right">100</TableCell></TableRow>
    <TableRow><TableCell>Item B</TableCell><TableCell className="text-right">250</TableCell></TableRow>
  </TableBody>
  <TableFooter>
    <TableRow>
      <TableCell>Total</TableCell>
      <TableCell className="text-right">350</TableCell>
    </TableRow>
  </TableFooter>
</Table>
```

The `TableFooter` class `bg-muted/50 font-medium` automatically visually distinguishes the row. NEVER add a manual `font-bold` to compete with the cva-style class ; the medium weight is the design choice.

## Anti-patterns (summary ; full catalogue in references/anti-patterns.md)

NEVER do these :

1. NEVER use this primitive for sortable / filterable / paginated / selectable tables. ALWAYS switch to `shadcn-impl-data-table` (TanStack Table v8 recipe). The primitive has no sort, no filter, no paginate, no select.
2. NEVER ship a Table with no `TableCaption` and no `aria-label` / `aria-labelledby`. Screen readers announce only "table" and the user has no clue what they are reading.
3. NEVER put `<TableHead>` inside `<TableBody>` (or `<TableCell>` inside `<TableHeader>`). Cells in the head are `<th>` ; cells in the body are `<td>`. Swapping breaks header semantics for screen readers and invalidates the HTML.
4. NEVER hand-roll a sort header (a `<TableHead><Button onClick={() => sort('name')}><ChevronUpDown /></Button></TableHead>` pattern) on this primitive. You will reinvent half of TanStack Table v8 incorrectly. ALWAYS use the DataTable recipe ; its `column.getCanSort()` + `column.getToggleSortingHandler()` API is the documented contract.
5. NEVER store pagination state (page index, page size, current rows) inside the primitive. The primitive is presentational. Pagination is a recipe concern ; if you need it, you do NOT need this skill.
6. NEVER wrap `<Table>` in your own `<div className="overflow-auto">`. The primitive already wraps `<table>` in `<div data-slot="table-container" className="relative w-full overflow-x-auto">`. You will end up with nested scrollbars.

See `references/anti-patterns.md` for the full WHY and FIX of each.

## Companion Skills

- [shadcn-impl-data-table](../../shadcn-impl/shadcn-impl-data-table/SKILL.md) : the TanStack Table v8 recipe that COMPOSES this primitive. ALWAYS read this if the table needs sort / filter / paginate / select. (B9 in the masterplan.)
- [shadcn-core-architecture](../../shadcn-core/shadcn-core-architecture/SKILL.md) : copy-not-install paradigm ; explains why the primitive lives at `components/ui/table.tsx` in your repo and is freely editable.
- [shadcn-core-stack](../../shadcn-core/shadcn-core-stack/SKILL.md) : Tailwind utilities, cn() helper, twMerge ; explains how the verbatim cn() calls in the Table source merge caller classes.
- [shadcn-impl-rsc-vs-client-boundaries](../../shadcn-impl/shadcn-impl-rsc-vs-client-boundaries/SKILL.md) : why the v4 Table file carries "use client" even with no hooks, and the safe edit to remove it.
- [shadcn-errors-tailwind-v3-v4-migration](../../shadcn-errors/shadcn-errors-tailwind-v3-v4-migration/SKILL.md) : v3 -> v4 source-shape differences (forwardRef removal, data-slot additions, padding shifts in TableHead / TableCell).

## Reference Links

- [references/methods.md](references/methods.md) : the eight subcomponent signatures verbatim from the v4 registry source, with v3 deltas annotated.
- [references/examples.md](references/examples.md) : five canonical recipes : static product table, sum-row footer table, table-with-caption, header-less data-only table, Table inside Card composition.
- [references/anti-patterns.md](references/anti-patterns.md) : six anti-patterns with WHY and FIX.

## Sources

All claims in this skill trace to URLs in `SOURCES.md` :

- https://ui.shadcn.com/docs/components/radix/table (canonical Table docs page, subcomponent list, caption-bottom default)
- https://github.com/shadcn-ui/ui/blob/main/apps/v4/registry/new-york-v4/ui/table.tsx (verbatim v4 source : function components, data-slot, has-aria-expanded, "use client")
- https://github.com/shadcn-ui/ui/blob/main/apps/v4/public/r/styles/default/table.json (verbatim v3 source : forwardRef components, no data-slot, no "use client", h-12 / p-4 sizing)
- https://tanstack.com/table/latest/docs (TanStack Table v8 : the recipe that composes this primitive, covered in shadcn-impl-data-table)
- https://www.w3.org/WAI/tutorials/tables/ (WAI accessibility tutorial : caption vs aria-label, when to use which)

Verified 2026-05-19.
