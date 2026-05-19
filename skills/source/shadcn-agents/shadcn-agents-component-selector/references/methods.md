# Decision Tables : Per-Family Discrimination Criteria

The validator workflow classifies a requirement into ONE of 8 families, then applies the family's decision table to pick a primitive. Each table below lists the discriminating criteria as columns. ALWAYS read every column ; NEVER pick a primitive after matching only one criterion when multiple distinguish the row.

## Family 1 : Modal-class

| Primitive | Viewport | Trigger | Dismiss methods | Blocking | Content size | Anchor | Underlying lib | Use when |
|-----------|----------|---------|-----------------|----------|--------------|--------|----------------|----------|
| `Dialog` | desktop + mobile (not gesture-aware) | button click | X button, Esc, overlay click | non-blocking | small to medium | centered | Radix Dialog | centered modal on desktop ; small editor or confirmation |
| `Sheet` | desktop + mobile | button click | X button, Esc, overlay click | non-blocking | medium to large | edge (`side="top|right|bottom|left"`) | Radix Dialog (extended) | side-panel settings, side-nav, side-edit ; mirrors Dialog API |
| `Drawer` | mobile-first (also desktop) | button click + drag gesture | X button, Esc, overlay click, **drag-to-close** | non-blocking | medium to full | edge (`direction="top|right|bottom|left"`) | `vaul` | mobile bottom-sheet, drag-aware panel, responsive Dialog/Drawer pattern |
| `AlertDialog` | desktop + mobile | button click | **only AlertDialogAction or AlertDialogCancel** | **blocking** | small | centered | Radix AlertDialog | destructive confirmation ; user MUST choose |

Discrimination order : (1) is this destructive ? -> AlertDialog. (2) is this mobile with gesture expected ? -> Drawer. (3) is this edge-anchored ? -> Sheet. (4) otherwise -> Dialog.

## Family 2 : Selector-class

| Primitive | Option count | Searchable | Custom item rendering | Mobile UX | Underlying lib | Use when |
|-----------|--------------|------------|-----------------------|-----------|----------------|----------|
| `Native Select` | small (under ~10) | n/a (no in-control search) | none (text only) | best (platform-native popup) | native `<select>` | small fixed list, mobile / a11y priority |
| `Select` | small to medium | no | yes (icons, descriptions in SelectItem) | good | Radix Select | styled dropdown without search |
| `Combobox` | medium to large (any size) | yes (filter by typing) | yes | good | Popover + Command, or 2026 dedicated Combobox | user MUST search ; option list grows |
| `Command` | any (typically large) | yes (fuzzy + groups) | yes | n/a (palette UX) | `cmdk` | global command palette, Cmd+K launcher |

Discrimination order : (1) command palette launcher ? -> Command. (2) needs search ? -> Combobox. (3) small list + mobile priority ? -> Native Select. (4) otherwise -> Select.

## Family 3 : Menu-class

| Primitive | Trigger | Role | Persistent UI ? | Underlying lib | Use when |
|-----------|---------|------|-----------------|----------------|----------|
| `DropdownMenu` | left-click on trigger button | actions | no (closes on select) | Radix DropdownMenu | menu of actions opened by a button click |
| `ContextMenu` | right-click (or long-press on touch) | actions | no (closes on select) | Radix ContextMenu | contextual actions on a region ; reuses DropdownMenu API surface |
| `Menubar` | hover or click on horizontal bar items | application chrome | **yes (persistent bar)** | Radix Menubar | desktop-app horizontal menu (File / Edit / View) |
| `NavigationMenu` | hover + click on top-level item | site navigation | yes (persistent top nav) | Radix NavigationMenu | top-level site nav with rich flyout panels |

Discrimination order : (1) right-click trigger ? -> ContextMenu. (2) persistent application bar ? -> Menubar. (3) site navigation with rich flyouts ? -> NavigationMenu. (4) one-off action menu ? -> DropdownMenu.

## Family 4 : Floating-class

| Primitive | Trigger | Interactive content allowed ? | Delay | Content type | Underlying lib | Use when |
|-----------|---------|-------------------------------|-------|--------------|----------------|----------|
| `Tooltip` | hover (or focus) | **no (a11y forbids)** | `delayDuration` | string / short fragment | Radix Tooltip | label, abbreviation expansion, button hint |
| `HoverCard` | hover | links allowed ; primary intent NOT-interactive | `openDelay` / `closeDelay` | rich preview card | Radix HoverCard | user-card preview, link-preview, hover-rich-preview |
| `Popover` | click (or controlled `open`) | **yes (forms, buttons, anything)** | none | rich interactive content | Radix Popover | click-to-open floating form, date picker, filter builder |

Discrimination order : (1) content contains a button / form / link with primary interactive intent ? -> Popover. (2) hover-rich-preview ? -> HoverCard. (3) hover-only string hint ? -> Tooltip.

## Family 5 : Button-class

| Composition | Semantic | DOM root | Use when |
|-------------|----------|----------|----------|
| `<Button onClick>` | action / verb | `<button>` | click changes state, submits, opens another component |
| `<Button asChild><Link href /></Button>` | navigation / noun | `<a>` (via Slot) | click navigates ; preserves Next.js prefetch, middle-click, keyboard, SEO |
| `<Button variant="link">` | inline-link visual style | `<button>` or via asChild `<a>` | text-style button matching link visual; pair with asChild for true navigation |

Slot rules :
- `asChild` requires EXACTLY one child element
- That child MUST forward refs and accept arbitrary props (`<Link>`, `<a>`, custom forwardRef components)
- NEVER pass two children inside `<Button asChild>`

## Family 6 : Notification-class

| Primitive | Persistence | Acknowledgement required ? | Position | Use when |
|-----------|-------------|----------------------------|----------|----------|
| `Sonner` (`toast(...)`, `toast.success`, `toast.error`, `toast.promise`, etc.) | ephemeral (auto-dismiss) | **no** | corner (top / bottom + left / center / right) | non-blocking transient ("Saved", "Copied", "Sent") |
| `Alert` | persistent in document flow | depends (no built-in dismiss) | inline (flows with layout) | form-level error, page-level warning banner |
| `AlertDialog` | modal, until acknowledged | **yes (explicit Action / Cancel)** | centered modal | destructive confirmation, irreversible action |

Discrimination order : (1) destructive / irreversible ? -> AlertDialog. (2) status that belongs in document flow ? -> Alert. (3) ephemeral non-critical feedback ? -> Sonner.

The legacy `Toast` Radix-based component has been REMOVED from the shadcn catalogue. NEVER use it ; ALWAYS use Sonner.

## Family 7 : Loading-class

| Primitive | Indicates | When final shape is known ? | Best for | Use when |
|-----------|-----------|----------------------------|----------|----------|
| `Skeleton` | layout placeholder | yes | full-page or large-region loads | the post-load layout shape is known (avatar + 3 lines, card grid, table rows) |
| `Spinner` (new 2026) | indeterminate progress | no | small inline / single-page-level | shape unknown, or loading is small / inline (in-button loading) |
| `Sonner` with `toast.promise()` | background task progress | n/a | non-blocking background actions | long-running task user can ignore while it completes |

Discrimination order : (1) is the final shape known and the load >= 200 ms ? -> Skeleton. (2) tiny inline indicator inside a Button ? -> Spinner. (3) background task with start / success / error states ? -> toast.promise.

## Family 8 : Form-class

| Primitive | Form library coupling | a11y wiring | Use when |
|-----------|------------------------|-------------|----------|
| `Form` (FormField, FormItem, FormLabel, FormControl, FormDescription, FormMessage) | **locked to react-hook-form + zod** | auto-wired via Controller | RHF + zod project ; canonical shadcn form layer |
| `Field` (FieldLabel, FieldDescription, FieldError, FieldGroup, FieldSet, FieldLegend, FieldContent, FieldSeparator, FieldTitle) | **decoupled from any form lib** | manual but consistent | TanStack Form, no-form-lib, or shared component across multiple form libs |

NEVER use both Form and Field for the same logical form. Pick one a11y layer per form. Mixing produces two competing aria-describedby chains.

## Family 9 : Table-class

| Primitive | State machinery | Required deps | Use when |
|-----------|-----------------|---------------|----------|
| `Table` (Table, TableHeader, TableBody, TableRow, TableHead, TableCell, TableCaption, TableFooter) | none (pure HTML + Tailwind) | none | static rows ; no sort, no filter, no pagination |
| `DataTable` (recipe) | `useReactTable` ; SortingState, ColumnFiltersState, VisibilityState, row-selection state | `@tanstack/react-table` v8 | sortable columns, filterable rows, pagination, column visibility toggles, row selection, virtualisation |

The DataTable is NOT a single primitive. It is a documented recipe that composes the Table primitives with `useReactTable`. Treat ANY of {sort, filter, paginate, column-visibility, row-select} as triggering the recipe.

## Composition Patterns (when families combine)

Some requirements span two families. Compose, never invent a new primitive :

| Composite requirement | Parent family | Child family | Composition |
|-----------------------|---------------|--------------|-------------|
| Popover with a searchable option list | Floating | Selector | `Popover` + `Command` (this is the classic Combobox recipe pre-2026) |
| Right-click menu with a submenu of checkbox items | Menu | Menu (sub) | `ContextMenu` + `ContextMenuSub` + `ContextMenuCheckboxItem` |
| Sheet containing a form | Modal | Form | `Sheet` + `Form` (RHF) ; close Sheet on submit success |
| Sortable table inside a Card | Loading shell | Table | `Card` + DataTable recipe inside `CardContent` |
| Date picker | Floating | Selector | `Popover` + `Calendar` + trigger `Button` ; documented DatePicker recipe |
| Command palette opened by global Cmd+K | Modal-like | Selector | `CommandDialog` (wraps Dialog + Command) |
| Mobile bottom-sheet containing a form | Modal | Form | `Drawer` + `Form` ; pair with responsive switch to `Dialog` on desktop |

ALWAYS decompose composite requirements before picking primitives. The parent (modal / floating) frames the layout ; the child (selector / form / table) fills the content.
