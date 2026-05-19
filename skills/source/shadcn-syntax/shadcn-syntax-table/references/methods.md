# Table : the eight subcomponent signatures (verbatim)

All signatures below are taken VERBATIM from `apps/v4/registry/new-york-v4/ui/table.tsx` in the shadcn-ui/ui repository (v4, evergreen-2026). v3 deltas annotated per subcomponent.

Source URLs :

- v4 : https://github.com/shadcn-ui/ui/blob/main/apps/v4/registry/new-york-v4/ui/table.tsx
- v3 : https://github.com/shadcn-ui/ui/blob/main/apps/v4/public/r/styles/default/table.json

The v4 file is preceded by a `"use client"` directive (line 1). The v3 file is NOT. See SKILL.md "RSC compatibility" section.

---

## 1. `Table`

Signature :

```tsx
function Table({ className, ...props }: React.ComponentProps<"table">): JSX.Element
```

v4 body :

```tsx
function Table({ className, ...props }: React.ComponentProps<"table">) {
  return (
    <div
      data-slot="table-container"
      className="relative w-full overflow-x-auto"
    >
      <table
        data-slot="table"
        className={cn("w-full caption-bottom text-sm", className)}
        {...props}
      />
    </div>
  )
}
```

Props : all native `<table>` props (HTMLAttributes<HTMLTableElement> + table-specific). The `className` prop is routed through `cn()` and merged onto the inner `<table>`, NOT the outer scroll-container `<div>`.

v3 delta :

- v3 signature is `React.forwardRef<HTMLTableElement, React.HTMLAttributes<HTMLTableElement>>`.
- v3 outer div uses `overflow-auto` (both axes) instead of v4's `overflow-x-auto` (horizontal only).
- v3 outer div has NO `data-slot` attribute. v3 inner `<table>` has NO `data-slot` attribute.
- v3 has `Table.displayName = "Table"`.

Data attributes (v4 only) :

- outer `<div>` : `data-slot="table-container"`
- inner `<table>` : `data-slot="table"`

---

## 2. `TableHeader`

Signature :

```tsx
function TableHeader({ className, ...props }: React.ComponentProps<"thead">): JSX.Element
```

v4 body :

```tsx
function TableHeader({ className, ...props }: React.ComponentProps<"thead">) {
  return (
    <thead
      data-slot="table-header"
      className={cn("[&_tr]:border-b", className)}
      {...props}
    />
  )
}
```

Props : all native `<thead>` props (HTMLAttributes<HTMLTableSectionElement>).

Class purpose : `[&_tr]:border-b` applies a bottom border to every descendant `<tr>` inside the header. This is the visual separator between the header row(s) and the body.

v3 delta :

- v3 signature is `React.forwardRef<HTMLTableSectionElement, React.HTMLAttributes<HTMLTableSectionElement>>`.
- v3 has NO `data-slot` attribute.
- v3 has `TableHeader.displayName = "TableHeader"`.
- Class string is identical.

---

## 3. `TableBody`

Signature :

```tsx
function TableBody({ className, ...props }: React.ComponentProps<"tbody">): JSX.Element
```

v4 body :

```tsx
function TableBody({ className, ...props }: React.ComponentProps<"tbody">) {
  return (
    <tbody
      data-slot="table-body"
      className={cn("[&_tr:last-child]:border-0", className)}
      {...props}
    />
  )
}
```

Props : all native `<tbody>` props.

Class purpose : `[&_tr:last-child]:border-0` STRIPS the bottom border on the last data row, so the body does not double-border against the footer or the page edge.

v3 delta :

- v3 is a `React.forwardRef` ; v3 has `TableBody.displayName = "TableBody"`.
- v3 has NO `data-slot`.
- Class string is identical.

---

## 4. `TableFooter`

Signature :

```tsx
function TableFooter({ className, ...props }: React.ComponentProps<"tfoot">): JSX.Element
```

v4 body :

```tsx
function TableFooter({ className, ...props }: React.ComponentProps<"tfoot">) {
  return (
    <tfoot
      data-slot="table-footer"
      className={cn(
        "border-t bg-muted/50 font-medium [&>tr]:last:border-b-0",
        className
      )}
      {...props}
    />
  )
}
```

Props : all native `<tfoot>` props.

Class purpose :

- `border-t` : top border separates the footer from the body.
- `bg-muted/50` : muted background at 50% opacity signals "summary row".
- `font-medium` : medium weight for total / sum content.
- `[&>tr]:last:border-b-0` : strips the bottom border on the last footer row.

v3 delta :

- v3 is a `React.forwardRef` ; v3 has `TableFooter.displayName = "TableFooter"`.
- v3 has NO `data-slot`.
- Class string is identical.

---

## 5. `TableRow`

Signature :

```tsx
function TableRow({ className, ...props }: React.ComponentProps<"tr">): JSX.Element
```

v4 body :

```tsx
function TableRow({ className, ...props }: React.ComponentProps<"tr">) {
  return (
    <tr
      data-slot="table-row"
      className={cn(
        "border-b transition-colors hover:bg-muted/50 has-aria-expanded:bg-muted/50 data-[state=selected]:bg-muted",
        className
      )}
      {...props}
    />
  )
}
```

Props : all native `<tr>` props.

Class purpose :

- `border-b` : bottom border (stripped on last body row by `TableBody`'s `[&_tr:last-child]:border-0`).
- `transition-colors` : smooth hover and selection transitions.
- `hover:bg-muted/50` : hover-state background.
- `has-aria-expanded:bg-muted/50` : (v4 only) background when a DESCENDANT element carries `aria-expanded="true"`. Useful when a row contains an expandable detail.
- `data-[state=selected]:bg-muted` : background when the DataTable recipe sets `data-state="selected"` on the row (used by TanStack rowSelection).

v3 delta :

- v3 is a `React.forwardRef` ; v3 has `TableRow.displayName = "TableRow"`.
- v3 has NO `data-slot`.
- v3 LACKS `has-aria-expanded:bg-muted/50`. Otherwise identical.

---

## 6. `TableHead`

Signature :

```tsx
function TableHead({ className, ...props }: React.ComponentProps<"th">): JSX.Element
```

v4 body :

```tsx
function TableHead({ className, ...props }: React.ComponentProps<"th">) {
  return (
    <th
      data-slot="table-head"
      className={cn(
        "h-10 px-2 text-left align-middle font-medium whitespace-nowrap text-foreground [&:has([role=checkbox])]:pr-0 [&>[role=checkbox]]:translate-y-[2px]",
        className
      )}
      {...props}
    />
  )
}
```

Props : all native `<th>` props (ThHTMLAttributes<HTMLTableCellElement>). Native attributes include `scope`, `abbr`, `colSpan`, `rowSpan`, `headers`.

Class purpose :

- `h-10 px-2` : fixed height 40px, horizontal padding 8px (v4 size).
- `text-left align-middle` : left-aligned text, vertically centred.
- `font-medium` : medium weight (column titles).
- `whitespace-nowrap` : single-line headers (the scroll container handles overflow).
- `text-foreground` : (v4) full-strength text colour for column titles.
- `[&:has([role=checkbox])]:pr-0` : strips right padding when the cell contains a checkbox (so the checkbox aligns to the right edge in select-all scenarios).
- `[&>[role=checkbox]]:translate-y-[2px]` : 2px downward nudge for checkboxes inside the cell, aligning the checkbox vertical centre with text descenders.

v3 delta :

- v3 is a `React.forwardRef` ; v3 has `TableHead.displayName = "TableHead"`.
- v3 has NO `data-slot`.
- v3 uses `h-12 px-4` (taller, wider) instead of v4's `h-10 px-2`.
- v3 uses `text-muted-foreground` (dimmer headers) instead of v4's `text-foreground`.
- v3 LACKS `whitespace-nowrap` and `[&>[role=checkbox]]:translate-y-[2px]`.

---

## 7. `TableCell`

Signature :

```tsx
function TableCell({ className, ...props }: React.ComponentProps<"td">): JSX.Element
```

v4 body :

```tsx
function TableCell({ className, ...props }: React.ComponentProps<"td">) {
  return (
    <td
      data-slot="table-cell"
      className={cn(
        "p-2 align-middle whitespace-nowrap [&:has([role=checkbox])]:pr-0 [&>[role=checkbox]]:translate-y-[2px]",
        className
      )}
      {...props}
    />
  )
}
```

Props : all native `<td>` props (TdHTMLAttributes<HTMLTableCellElement>). Native attributes include `colSpan`, `rowSpan`, `headers`.

Class purpose :

- `p-2` : padding 8px on all sides (v4).
- `align-middle` : vertical centre alignment.
- `whitespace-nowrap` : single-line cells (the scroll container handles overflow).
- `[&:has([role=checkbox])]:pr-0` and `[&>[role=checkbox]]:translate-y-[2px]` : checkbox alignment, mirror of TableHead.

v3 delta :

- v3 is a `React.forwardRef` ; v3 has `TableCell.displayName = "TableCell"`.
- v3 has NO `data-slot`.
- v3 uses `p-4` (much more padding) instead of v4's `p-2`.
- v3 LACKS `whitespace-nowrap` and `[&>[role=checkbox]]:translate-y-[2px]`.

---

## 8. `TableCaption`

Signature :

```tsx
function TableCaption({ className, ...props }: React.ComponentProps<"caption">): JSX.Element
```

v4 body :

```tsx
function TableCaption({
  className,
  ...props
}: React.ComponentProps<"caption">) {
  return (
    <caption
      data-slot="table-caption"
      className={cn("mt-4 text-sm text-muted-foreground", className)}
      {...props}
    />
  )
}
```

Props : all native `<caption>` props.

Class purpose :

- `mt-4` : top margin (works in tandem with the `caption-bottom` class on the parent `<table>` so the caption sits 16px below the body).
- `text-sm text-muted-foreground` : small muted text (footnote tone).

v3 delta :

- v3 is a `React.forwardRef` ; v3 has `TableCaption.displayName = "TableCaption"`.
- v3 has NO `data-slot`.
- Class string is identical.

---

## Export shape (v4)

```tsx
export {
  Table,
  TableHeader,
  TableBody,
  TableFooter,
  TableHead,
  TableRow,
  TableCell,
  TableCaption,
}
```

Eight named exports, alphabetical-but-grouped. ALWAYS import with named imports : `import { Table, TableHeader, ... } from "@/components/ui/table"`. NEVER default-import : the file has no default export.

## Type ergonomics

Every subcomponent uses `React.ComponentProps<"X">` (v4) or `React.HTMLAttributes<HTMLXElement>` (v3). The v4 form is broader : `React.ComponentProps<"table">` includes `cellPadding`, `cellSpacing`, `summary`, `width`, `align`, plus the standard HTMLAttributes. ALWAYS prefer `React.ComponentProps<"X">` over `React.HTMLAttributes<HTMLXElement>` when extending the components, for full prop coverage.

## Source verification

The verbatim source quotes above were retrieved 2026-05-19 from the URLs listed at the top of this file. If shadcn upgrades the registry (which it does occasionally without bumping a public version), re-fetch via :

```bash
gh api repos/shadcn-ui/ui/contents/apps/v4/registry/new-york-v4/ui/table.tsx | jq -r '.content' | base64 -d
```

NEVER paste the source into your project from this skill. ALWAYS run `npx shadcn@latest add table`, which fetches the live registry version, applies your project's style + baseColor + cssVariables choices, and writes the file to `components/ui/table.tsx` for you.
