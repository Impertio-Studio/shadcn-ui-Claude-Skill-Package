# Validation Scenarios : WRONG -> RIGHT -> WHY

Eight worked validation scenarios. Each starts with a real user requirement, names the WRONG primitive that an AI or developer commonly picks, names the RIGHT primitive, and explains WHY in terms of the family's discriminating criterion.

ALWAYS read the family classification step out loud before judging the pick. NEVER recommend by aesthetic preference.

---

## Scenario 1 : Mobile bottom-sheet for record edit

**Requirement** : "On mobile, when the user taps a record, slide up a panel from the bottom where they can edit the fields and drag it down to dismiss."

**Wrong pick** : `Dialog`

```tsx
// WRONG
<Dialog open={open} onOpenChange={setOpen}>
  <DialogContent>
    <DialogHeader><DialogTitle>Edit record</DialogTitle></DialogHeader>
    <Form ... />
  </DialogContent>
</Dialog>
```

**Right pick** : `Drawer` (with `Dialog` on desktop via responsive media query)

```tsx
// RIGHT
const isMobile = useMediaQuery("(max-width: 768px)")

return isMobile ? (
  <Drawer open={open} onOpenChange={setOpen}>
    <DrawerContent>
      <DrawerHeader><DrawerTitle>Edit record</DrawerTitle></DrawerHeader>
      <Form ... />
    </DrawerContent>
  </Drawer>
) : (
  <Dialog open={open} onOpenChange={setOpen}>
    <DialogContent>
      <DialogHeader><DialogTitle>Edit record</DialogTitle></DialogHeader>
      <Form ... />
    </DialogContent>
  </Dialog>
)
```

**Why** : Family = Modal-class. Discriminating criteria : viewport = mobile AND user expects drag-to-dismiss. Dialog is centered, button-dismissed, NOT gesture-aware. Drawer wraps `vaul` for gesture-aware bottom-sheet behaviour and is the documented mobile-first choice. The responsive pattern (Drawer on mobile, Dialog on desktop) is itself the documented shadcn recipe (see `shadcn-impl-responsive-dialog-drawer`).

Forward-pointer : `shadcn-syntax-drawer`, `shadcn-impl-responsive-dialog-drawer`.

---

## Scenario 2 : Searchable country dropdown

**Requirement** : "A form field where the user picks one of the ~200 countries. They should be able to type a few letters to filter the list."

**Wrong pick** : `Select`

```tsx
// WRONG
<Select onValueChange={setCountry}>
  <SelectTrigger><SelectValue placeholder="Country" /></SelectTrigger>
  <SelectContent>
    {countries.map((c) => (
      <SelectItem key={c.code} value={c.code}>{c.name}</SelectItem>
    ))}
  </SelectContent>
</Select>
```

**Right pick** : `Combobox` (Popover + Command, or 2026 dedicated Combobox)

```tsx
// RIGHT
<Popover open={open} onOpenChange={setOpen}>
  <PopoverTrigger asChild>
    <Button variant="outline" role="combobox" aria-expanded={open}>
      {country ? countries.find((c) => c.code === country)?.name : "Country"}
      <ChevronsUpDown className="ml-2 size-4 opacity-50" />
    </Button>
  </PopoverTrigger>
  <PopoverContent className="p-0">
    <Command>
      <CommandInput placeholder="Search country..." />
      <CommandEmpty>No country found.</CommandEmpty>
      <CommandList>
        {countries.map((c) => (
          <CommandItem key={c.code} value={c.name} onSelect={() => { setCountry(c.code); setOpen(false) }}>
            {c.name}
          </CommandItem>
        ))}
      </CommandList>
    </Command>
  </PopoverContent>
</Popover>
```

**Why** : Family = Selector-class. Discriminating criteria : option count > ~10 AND user MUST search. Select has no built-in search. Forcing search into Select means re-implementing Combobox by hand. Combobox is exactly Popover + Command for this purpose.

Forward-pointer : `shadcn-syntax-selectors`, `shadcn-syntax-command`.

---

## Scenario 3 : Right-click menu on a file row

**Requirement** : "When the user right-clicks on a row in the file list, show actions : Open, Rename, Duplicate, Delete."

**Wrong pick** : `DropdownMenu` with a hidden trigger or a manual right-click binding

```tsx
// WRONG
<div onContextMenu={(e) => { e.preventDefault(); setOpen(true) }}>
  <DropdownMenu open={open} onOpenChange={setOpen}>
    <DropdownMenuTrigger asChild><div /></DropdownMenuTrigger>
    <DropdownMenuContent>...</DropdownMenuContent>
  </DropdownMenu>
</div>
```

**Right pick** : `ContextMenu`

```tsx
// RIGHT
<ContextMenu>
  <ContextMenuTrigger asChild>
    <Row file={file} />
  </ContextMenuTrigger>
  <ContextMenuContent>
    <ContextMenuItem onSelect={() => open(file)}>Open</ContextMenuItem>
    <ContextMenuItem onSelect={() => rename(file)}>Rename</ContextMenuItem>
    <ContextMenuItem onSelect={() => duplicate(file)}>Duplicate</ContextMenuItem>
    <ContextMenuSeparator />
    <ContextMenuItem className="text-destructive" onSelect={() => del(file)}>Delete</ContextMenuItem>
  </ContextMenuContent>
</ContextMenu>
```

**Why** : Family = Menu-class. Discriminating criterion : trigger = right-click (a platform gesture). ContextMenu's entire purpose is right-click ; it mirrors DropdownMenu's API surface but binds to the contextmenu event natively and handles long-press on touch. Reusing DropdownMenu loses the platform contract and skips touch handling.

Forward-pointer : `shadcn-syntax-menu-primitives`.

---

## Scenario 4 : Interactive avatar hover-preview

**Requirement** : "On hover over an avatar in a comments list, show a card with the user's name, bio, and a Follow button."

**Wrong pick** : `Tooltip`

```tsx
// WRONG : Tooltip cannot legally contain a button (a11y)
<Tooltip>
  <TooltipTrigger asChild><Avatar>...</Avatar></TooltipTrigger>
  <TooltipContent>
    <div>{user.name}</div>
    <div>{user.bio}</div>
    <Button onClick={follow}>Follow</Button>
  </TooltipContent>
</Tooltip>
```

**Right pick** : `HoverCard` if the Follow button is optional or rare, else `Popover` on click

```tsx
// RIGHT (hover preview WITH a primary action -> Popover on click is cleaner ;
// HoverCard works only if the action is secondary)
<Popover>
  <PopoverTrigger asChild><Avatar>...</Avatar></PopoverTrigger>
  <PopoverContent>
    <div className="font-semibold">{user.name}</div>
    <p className="text-sm text-muted-foreground">{user.bio}</p>
    <Button size="sm" onClick={follow}>Follow</Button>
  </PopoverContent>
</Popover>
```

**Why** : Family = Floating-class. Discriminating criterion : floating content contains an interactive element (Button). Tooltip's ARIA role is `tooltip` ; assistive tech reads it as a label, not as an interactive region. Putting a Button inside Tooltip means keyboard / screen-reader users cannot reach the button. HoverCard tolerates LINKS but its primary intent is preview ; if there is a primary action button, Popover with click trigger is the documented choice.

Forward-pointer : `shadcn-syntax-popover-tooltip-hovercard`.

---

## Scenario 5 : Delete confirmation via toast

**Requirement** : "When the user clicks Delete on an account, show a toast asking them to confirm by clicking an Undo button before the delete commits."

**Wrong pick** : `Sonner` (`toast(...)` with an action button)

```tsx
// WRONG : toast is dismissible by ignoring it ; user can lose the message
function onDelete() {
  toast("Account will be deleted.", {
    action: { label: "Undo", onClick: cancelDelete },
  })
  setTimeout(commitDelete, 5000)
}
```

**Right pick** : `AlertDialog`

```tsx
// RIGHT
<AlertDialog>
  <AlertDialogTrigger asChild>
    <Button variant="destructive">Delete account</Button>
  </AlertDialogTrigger>
  <AlertDialogContent>
    <AlertDialogHeader>
      <AlertDialogTitle>Delete this account ?</AlertDialogTitle>
      <AlertDialogDescription>
        This is irreversible. All data will be permanently removed.
      </AlertDialogDescription>
    </AlertDialogHeader>
    <AlertDialogFooter>
      <AlertDialogCancel>Cancel</AlertDialogCancel>
      <AlertDialogAction onClick={commitDelete} className="bg-destructive">
        Delete
      </AlertDialogAction>
    </AlertDialogFooter>
  </AlertDialogContent>
</AlertDialog>
```

**Why** : Family = Notification-class. Discriminating criterion : action is destructive / irreversible AND user MUST acknowledge. Toast is ephemeral and dismissible by ignoring it ; a destructive confirmation that the user can miss is a real-world data-loss bug. AlertDialog disables overlay-click and Esc-to-dismiss patterns ; it forces an explicit Action or Cancel.

Forward-pointer : `shadcn-syntax-dialog` (AlertDialog), `shadcn-syntax-toast-sonner`.

---

## Scenario 6 : Loading state for a known dashboard layout

**Requirement** : "While the dashboard data is loading, show a placeholder for the 4-card KPI grid and the chart below."

**Wrong pick** : centered `Spinner`

```tsx
// WRONG : page jumps when data arrives (CLS), shape unknown to user
{loading ? (
  <div className="grid h-screen place-items-center"><Spinner /></div>
) : (
  <Dashboard data={data} />
)}
```

**Right pick** : `Skeleton` shaped like the final layout

```tsx
// RIGHT
{loading ? (
  <div className="grid grid-cols-4 gap-4">
    {Array.from({ length: 4 }).map((_, i) => (
      <Card key={i}>
        <CardHeader>
          <Skeleton className="h-4 w-24" />
        </CardHeader>
        <CardContent>
          <Skeleton className="h-8 w-32" />
          <Skeleton className="mt-2 h-3 w-20" />
        </CardContent>
      </Card>
    ))}
    <Skeleton className="col-span-4 h-64 w-full" />
  </div>
) : (
  <Dashboard data={data} />
)}
```

**Why** : Family = Loading-class. Discriminating criterion : final shape is known. Skeleton matches the post-load layout, eliminates cumulative layout shift, and gives the user perceived speed because the page does not snap into existence. Spinner gives no preview and worsens perceived performance for known layouts.

Forward-pointer : `shadcn-syntax-toast-sonner` (toast.promise for background load progress), syntax skill for Card.

---

## Scenario 7 : Sortable, filterable users table

**Requirement** : "A table of users where columns can sort, rows can be filtered by an email search input, and the user can toggle which columns are visible."

**Wrong pick** : plain `Table` with hand-rolled sort / filter / visibility state

```tsx
// WRONG : reinvents what TanStack Table provides
const [sortKey, setSortKey] = useState("email")
const sorted = useMemo(() => [...users].sort(...), [users, sortKey])
const filtered = useMemo(() => sorted.filter(...), [sorted, filter])
// then 60+ lines wiring per-column header onClick handlers, etc.
return <Table>...</Table>
```

**Right pick** : DataTable recipe (`useReactTable` + Table primitives)

```tsx
// RIGHT
const table = useReactTable({
  data: users,
  columns,
  getCoreRowModel: getCoreRowModel(),
  getSortedRowModel: getSortedRowModel(),
  getFilteredRowModel: getFilteredRowModel(),
  getPaginationRowModel: getPaginationRowModel(),
  onSortingChange: setSorting,
  onColumnFiltersChange: setColumnFilters,
  onColumnVisibilityChange: setColumnVisibility,
  state: { sorting, columnFilters, columnVisibility },
})

return (
  <Table>
    <TableHeader>
      {table.getHeaderGroups().map((hg) => (
        <TableRow key={hg.id}>
          {hg.headers.map((h) => (
            <TableHead key={h.id} onClick={h.column.getToggleSortingHandler()}>
              {flexRender(h.column.columnDef.header, h.getContext())}
            </TableHead>
          ))}
        </TableRow>
      ))}
    </TableHeader>
    <TableBody>...</TableBody>
  </Table>
)
```

**Why** : Family = Table-class. Discriminating criterion : ANY of {sort, filter, pagination, column-visibility, row-selection}. Once any of these is requested, the plain Table primitive is the wrong layer ; the DataTable recipe composes TanStack Table v8 with the Table primitives to give all of these for free.

Forward-pointer : `shadcn-syntax-table`, `shadcn-impl-data-table`.

---

## Scenario 8 : Shared form a11y across react-hook-form and TanStack Form

**Requirement** : "We are migrating from react-hook-form to TanStack Form. We want a single set of form components that work for both during the migration."

**Wrong pick** : `Form` (FormField / FormItem ...)

```tsx
// WRONG : Form is locked to react-hook-form's Controller ; TanStack Form
// cannot drive these without re-implementing the integration
<Form {...rhfForm}>
  <FormField name="email" control={rhfForm.control} render={...} />
</Form>
// ... then later in a TanStack form ...
<Form {...tanstackForm}>  {/* shape mismatch */}
  <FormField ... />        {/* breaks */}
</Form>
```

**Right pick** : `Field` (FieldLabel / FieldDescription / FieldError ...)

```tsx
// RIGHT : Field is decoupled from any form lib ; wire to RHF or TanStack
// Form per usage
function EmailField({ value, onChange, error, description }: Props) {
  return (
    <Field>
      <FieldLabel htmlFor="email">Email</FieldLabel>
      <Input id="email" value={value} onChange={onChange} aria-invalid={!!error} />
      <FieldDescription>{description}</FieldDescription>
      {error && <FieldError>{error}</FieldError>}
    </Field>
  )
}
```

**Why** : Family = Form-class. Discriminating criterion : the same component must work across two form libraries. Form is the canonical react-hook-form integration ; its FormField is hard-wired to RHF's `Controller`. The new 2026 Field primitive provides identical a11y wiring (aria-describedby, aria-invalid, label htmlFor) without any form-library coupling, so it can be driven by RHF, TanStack Form, or no form library at all.

Forward-pointer : `shadcn-syntax-field`, `shadcn-syntax-form`, `shadcn-impl-form-validation`.

---

## Summary Table : Scenarios

| # | Requirement | Wrong | Right | Family | Discriminating criterion |
|---|-------------|-------|-------|--------|--------------------------|
| 1 | Mobile edit panel with drag-to-close | Dialog | Drawer (+ Dialog on desktop) | Modal | viewport = mobile + gesture |
| 2 | Searchable country dropdown | Select | Combobox | Selector | option count + search |
| 3 | Right-click row actions | DropdownMenu | ContextMenu | Menu | trigger = right-click |
| 4 | Hover-preview with Follow button | Tooltip | Popover (or HoverCard) | Floating | interactive content |
| 5 | Confirm destructive delete | Sonner toast | AlertDialog | Notification | destructive + must acknowledge |
| 6 | Loading state for known dashboard | Spinner | Skeleton | Loading | final shape known |
| 7 | Sortable / filterable user table | Table | DataTable recipe | Table | any sort/filter/paginate |
| 8 | Form shared across form libs | Form | Field | Form | form-lib coupling |
