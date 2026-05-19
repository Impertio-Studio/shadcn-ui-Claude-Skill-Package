# Table : five canonical recipes

Each example is COMPLETE and copy-pasteable. All imports are shown. All recipes ALWAYS use this skill's primitive only and do NOT depend on TanStack Table. If your design needs sorting / filtering / paginating / selection, see `shadcn-impl-data-table` instead.

---

## Example 1 : static product table

A read-only product catalogue with three columns and five rows. Caption announces the table to screen readers.

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

const products = [
  { sku: "SKU-001", name: "Wireless mouse",    price: "EUR  29.95" },
  { sku: "SKU-002", name: "Mechanical keyboard", price: "EUR 129.00" },
  { sku: "SKU-003", name: "USB-C hub",          price: "EUR  49.00" },
  { sku: "SKU-004", name: "Desk mat",           price: "EUR  19.95" },
  { sku: "SKU-005", name: "Webcam 1080p",       price: "EUR  79.00" },
]

export function ProductTable() {
  return (
    <Table>
      <TableCaption>Product catalogue, evergreen 2026.</TableCaption>
      <TableHeader>
        <TableRow>
          <TableHead className="w-[140px]">SKU</TableHead>
          <TableHead>Product</TableHead>
          <TableHead className="text-right">Price</TableHead>
        </TableRow>
      </TableHeader>
      <TableBody>
        {products.map((p) => (
          <TableRow key={p.sku}>
            <TableCell className="font-mono">{p.sku}</TableCell>
            <TableCell>{p.name}</TableCell>
            <TableCell className="text-right">{p.price}</TableCell>
          </TableRow>
        ))}
      </TableBody>
    </Table>
  )
}
```

Key choices :

- `TableCaption` provides the accessible name (Approach A from SKILL.md "Accessibility").
- `TableHead className="w-[140px]"` fixes the SKU column width via the arbitrary-value Tailwind utility. ALWAYS apply column widths to `<TableHead>` ; `<TableCell>` cells in the same column inherit the column width from `<colgroup>`-style native sizing.
- `className="font-mono"` on the SKU cell uses Tailwind's monospace family for fixed-width identifiers.
- `className="text-right"` on both the price `<TableHead>` and every price `<TableCell>` right-aligns the numeric column. ALWAYS align numeric columns right.
- The product array is mapped inline. The component is RSC-safe (no client features) ; if the surrounding file does not carry `"use client"`, this remains a server component.

---

## Example 2 : invoice table with TableFooter sum row

Demonstrates the canonical use of `TableFooter` for a total / sum row. The footer's `bg-muted/50 font-medium` class makes the row visually distinct.

```tsx
import {
  Table,
  TableBody,
  TableCaption,
  TableCell,
  TableFooter,
  TableHead,
  TableHeader,
  TableRow,
} from "@/components/ui/table"

const invoice = [
  { line: "Hosting (March 2026)",    qty: 1, unit: "EUR 100.00", total: "EUR 100.00" },
  { line: "Domain renewal",          qty: 1, unit: "EUR  12.50", total: "EUR  12.50" },
  { line: "SSL certificate",         qty: 1, unit: "EUR  50.00", total: "EUR  50.00" },
  { line: "Email mailbox (5 seats)", qty: 5, unit: "EUR   2.95", total: "EUR  14.75" },
]

export function InvoiceTable() {
  return (
    <Table>
      <TableCaption>Invoice INV-2026-031, due 2026-04-15.</TableCaption>
      <TableHeader>
        <TableRow>
          <TableHead>Line</TableHead>
          <TableHead className="text-right">Qty</TableHead>
          <TableHead className="text-right">Unit price</TableHead>
          <TableHead className="text-right">Total</TableHead>
        </TableRow>
      </TableHeader>
      <TableBody>
        {invoice.map((row) => (
          <TableRow key={row.line}>
            <TableCell>{row.line}</TableCell>
            <TableCell className="text-right">{row.qty}</TableCell>
            <TableCell className="text-right">{row.unit}</TableCell>
            <TableCell className="text-right">{row.total}</TableCell>
          </TableRow>
        ))}
      </TableBody>
      <TableFooter>
        <TableRow>
          <TableCell colSpan={3}>Total</TableCell>
          <TableCell className="text-right">EUR 177.25</TableCell>
        </TableRow>
      </TableFooter>
    </Table>
  )
}
```

Key choices :

- `TableFooter` is the LAST direct child of `<Table>` (after `<TableBody>`). ALWAYS in source order ; native HTML positions `<tfoot>` correctly regardless.
- `<TableCell colSpan={3}>Total</TableCell>` merges the first three columns into one label cell. `colSpan` is a native `<td>` attribute and works directly on shadcn's `<TableCell>`.
- ALWAYS keep numeric formatting consistent between body rows and the footer total ; mixing "EUR" with bare numbers reads as inconsistent.
- NEVER add `font-bold` to the footer cells. The `font-medium` from the `<TableFooter>` class is the deliberate weight ; bold breaks the visual hierarchy with `<TableHead>` cells (which are also `font-medium`).

---

## Example 3 : table with visible caption flipped to top

Demonstrates how to override the shadcn default `caption-bottom` and place the caption at the top of the table.

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

export function SettingsTable() {
  return (
    <Table className="caption-top">
      <TableCaption className="mb-4 mt-0">User permissions overview</TableCaption>
      <TableHeader>
        <TableRow>
          <TableHead>Permission</TableHead>
          <TableHead>Description</TableHead>
          <TableHead className="text-right">Granted</TableHead>
        </TableRow>
      </TableHeader>
      <TableBody>
        <TableRow>
          <TableCell>read:invoices</TableCell>
          <TableCell>View invoice list and details.</TableCell>
          <TableCell className="text-right">Yes</TableCell>
        </TableRow>
        <TableRow>
          <TableCell>write:invoices</TableCell>
          <TableCell>Create, update, or void invoices.</TableCell>
          <TableCell className="text-right">No</TableCell>
        </TableRow>
        <TableRow>
          <TableCell>admin:billing</TableCell>
          <TableCell>Manage billing settings and payment methods.</TableCell>
          <TableCell className="text-right">No</TableCell>
        </TableRow>
      </TableBody>
    </Table>
  )
}
```

Key choices :

- `<Table className="caption-top">` overrides the default `caption-bottom` via Tailwind's `caption-side` utility. The `cn()` + twMerge in `<Table>` resolves the conflict correctly : caller class wins.
- `<TableCaption className="mb-4 mt-0">` flips the spacing : margin-top zero (since the caption is now ABOVE the table) and margin-bottom 16px (matching the original `mt-4` offset but on the opposite side).
- ALWAYS pair the `caption-top` class on `<Table>` with a margin flip on `<TableCaption>`. Without it, the caption would have 16px gap below the table heading, which looks wrong when the caption sits at the top.

---

## Example 4 : header-less data-only table

Some compact summary tables (e.g. key-value summaries on a profile card) do NOT need column titles. Omit `<TableHeader>` and structure the data as label / value pairs.

```tsx
import {
  Table,
  TableBody,
  TableCell,
  TableRow,
} from "@/components/ui/table"

export function MetricSummary() {
  return (
    <Table aria-label="Account metrics summary">
      <TableBody>
        <TableRow>
          <TableCell className="font-medium">Total revenue</TableCell>
          <TableCell className="text-right">EUR 12,450.00</TableCell>
        </TableRow>
        <TableRow>
          <TableCell className="font-medium">Open invoices</TableCell>
          <TableCell className="text-right">7</TableCell>
        </TableRow>
        <TableRow>
          <TableCell className="font-medium">Overdue invoices</TableCell>
          <TableCell className="text-right">2</TableCell>
        </TableRow>
        <TableRow>
          <TableCell className="font-medium">Average days to pay</TableCell>
          <TableCell className="text-right">14</TableCell>
        </TableRow>
      </TableBody>
    </Table>
  )
}
```

Key choices :

- NO `<TableHeader>`, NO `<TableCaption>`. ALWAYS provide an accessible name another way : here `aria-label="Account metrics summary"` on `<Table>` (Approach B from SKILL.md "Accessibility").
- The left column's `font-medium` mimics a label ; the right column is the value. Two-column key-value tables are a common compact pattern.
- ALWAYS omit `<TableHeader>` entirely when there is no header row ; NEVER ship an empty `<TableHeader><TableRow></TableRow></TableHeader>`, it produces an empty `<thead>` with no semantic content.

---

## Example 5 : Table inside Card composition

The most common shadcn composition : a Card with a header and a Table inside its content. The Card provides the visual surface and the title ; the Table provides the data.

```tsx
import {
  Card,
  CardContent,
  CardDescription,
  CardHeader,
  CardTitle,
} from "@/components/ui/card"
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from "@/components/ui/table"

const recentLogins = [
  { user: "alice@example.com",   when: "2026-05-19 09:14", ip: "203.0.113.7"   },
  { user: "bob@example.com",     when: "2026-05-19 08:51", ip: "198.51.100.42" },
  { user: "carol@example.com",   when: "2026-05-18 22:03", ip: "203.0.113.7"   },
  { user: "dave@example.com",    when: "2026-05-18 17:45", ip: "192.0.2.18"    },
]

export function RecentLoginsCard() {
  return (
    <Card>
      <CardHeader>
        <CardTitle id="recent-logins-title">Recent logins</CardTitle>
        <CardDescription>Last 24 hours.</CardDescription>
      </CardHeader>
      <CardContent className="p-0">
        <Table aria-labelledby="recent-logins-title">
          <TableHeader>
            <TableRow>
              <TableHead className="pl-6">User</TableHead>
              <TableHead>When</TableHead>
              <TableHead className="pr-6">IP</TableHead>
            </TableRow>
          </TableHeader>
          <TableBody>
            {recentLogins.map((row) => (
              <TableRow key={row.user + row.when}>
                <TableCell className="pl-6 font-mono">{row.user}</TableCell>
                <TableCell>{row.when}</TableCell>
                <TableCell className="pr-6 font-mono">{row.ip}</TableCell>
              </TableRow>
            ))}
          </TableBody>
        </Table>
      </CardContent>
    </Card>
  )
}
```

Key choices :

- `<CardContent className="p-0">` STRIPS the default Card content padding. ALWAYS do this when embedding a Table in a Card : the Table's internal cell padding is sufficient and the Card padding would cause an inset gap that looks visually broken.
- `<TableHead className="pl-6">` and `<TableHead className="pr-6">` (mirrored on `<TableCell>`) restore HORIZONTAL inset on the first and last columns, so cell content is not flush against the Card edge. This is the canonical pattern for Table-in-Card.
- `<Table aria-labelledby="recent-logins-title">` references the `<CardTitle>` by id (Approach C from SKILL.md "Accessibility"). The Card title doubles as the table's accessible name ; no `<TableCaption>` is needed and no redundant `aria-label` is set.
- ALWAYS pair the `aria-labelledby` on `<Table>` with `id="..."` on the corresponding `<CardTitle>`. NEVER set `aria-label` AND `aria-labelledby` simultaneously ; aria-labelledby wins, aria-label becomes dead code.

---

## Pattern summary

| Recipe | Caption strategy | Footer? | Header? | Use when |
|--------|------------------|---------|---------|----------|
| Example 1 (product table) | `<TableCaption>` | no | yes | Standalone read-only listing, no surrounding chrome |
| Example 2 (invoice with sum) | `<TableCaption>` | yes | yes | Line-items with a total / sum |
| Example 3 (caption-top) | `<TableCaption>` + `caption-top` override | no | yes | Heading-style caption preferred over footnote |
| Example 4 (header-less) | `aria-label` on `<Table>` | no | no | Compact label / value summary |
| Example 5 (Table in Card) | `aria-labelledby` referencing `<CardTitle>` | no | yes | Table embedded in a labelled surface (Card, Dialog, Drawer) |

ALWAYS pick exactly one accessible-name strategy per table. NEVER combine them.

For tables that need sorting / filtering / pagination / selection, ALL five recipes above are the wrong starting point. STOP and read `shadcn-impl-data-table` (the TanStack Table v8 recipe).
