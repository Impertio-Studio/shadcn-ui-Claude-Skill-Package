---
name: shadcn-agents-component-selector
description: >
  Use when a user describes a UI requirement in plain English ("a modal to
  edit a record", "a searchable dropdown of countries", "show a non-blocking
  success message", "a right-click menu on a table row", "a sortable table
  of users") and the correct shadcn primitive is not yet obvious, when an
  existing implementation picks the wrong shadcn component for the
  interaction model ("Dialog used as a mobile bottom-sheet", "Select used
  for a 200-item searchable list", "Tooltip used for content with a link
  inside", "Toast (Sonner) used for a destructive confirmation"), when
  validating a code-review or AI-generated draft against the shadcn
  component catalogue, when choosing between Dialog vs Sheet vs Drawer,
  Select vs Combobox vs Command, DropdownMenu vs ContextMenu vs Menubar
  vs NavigationMenu, Tooltip vs HoverCard vs Popover, Button vs Link vs
  asChild composition, Toast (Sonner) vs Alert vs AlertDialog, Card vs
  Skeleton vs Spinner for loading states, Form (react-hook-form) vs Field
  (new 2026 primitive) for form composition, or Table vs DataTable for
  tabular data display.
  Prevents the canonical mis-selection failures : reaching for Dialog on
  mobile where the gesture-aware Drawer (vaul) is the documented choice,
  reaching for Select when the option list exceeds ~10 items or needs
  search where Combobox (Popover + Command) is the documented choice,
  reaching for DropdownMenu when the trigger is a right-click where
  ContextMenu is the correct primitive, reaching for Tooltip to hold
  interactive content (button, link, form) which Tooltip a11y forbids,
  reaching for Toast (Sonner) to confirm a destructive action where
  AlertDialog must block until the user explicitly confirms, reaching
  for the plain Table primitive when the user asked for sorting,
  filtering, pagination, or column-visibility which require the
  TanStack-Table DataTable recipe, reaching for the legacy
  Toast component which has been REMOVED from the catalogue in favour
  of Sonner, and reaching for native Form binding when the form is
  shared across react-hook-form and TanStack Form where the new Field
  primitive is the correct shared a11y layer.
  Covers the full 59-component shadcn ui catalogue grouped by
  interaction model (modal-class, selector-class, menu-class, floating-
  class, notification-class, loading-class, form-class, table-class),
  per-category decision tables with discrimination criteria (trigger
  type, viewport size, search need, blocking behaviour, content
  complexity, data shape), worked validation scenarios mapping a real
  prompt to a wrong-then-right component pick with the rationale, the
  validator workflow (input -> classify -> recommend -> cross-reference
  to syntax skill), and a complete cross-reference matrix from every
  decision back to the relevant shadcn-syntax-* skill in this package.
  Keywords: shadcn component selector, which component, Dialog vs Sheet
  vs Drawer, Select vs Combobox vs Command, DropdownMenu vs ContextMenu,
  Menubar vs NavigationMenu, Tooltip vs HoverCard vs Popover, Button vs
  Link asChild, Toast vs Alert vs AlertDialog, Card vs Skeleton vs
  Spinner, Form vs Field primitive, Table vs DataTable, when to use
  Popover, when to use Drawer, when to use Sheet, mobile bottom sheet,
  searchable dropdown, right-click menu, blocking confirmation,
  non-blocking notification, loading state, sortable table, filterable
  table, component decision tree, choosing shadcn component, validate
  component choice, which modal, which menu, which selector, which
  notification, which loading indicator, which form layer,
  how do I pick a component, recommend shadcn component, wrong component
  shadcn, refactor Dialog to Drawer, refactor Select to Combobox,
  refactor Toast to AlertDialog, shadcn component recommendation, shadcn
  ui selector agent, shadcn validator, component cross-reference matrix.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui : Component Selector (Agent / Validator)

This skill is an AGENT skill. It does NOT teach a single component's API. It teaches a decision-tree validator : given a UI requirement in plain English or a draft implementation, classify the requirement, recommend the correct shadcn primitive, flag wrong picks, and forward-point to the syntax skill that covers the chosen primitive.

The 59 shadcn components cluster into 8 interaction families. ALWAYS classify the requirement into ONE family BEFORE picking a primitive. NEVER reason directly from "this looks like X" because surface similarity (Dialog and Drawer both overlay, Tooltip and Popover both float) hides the discriminating criterion (gesture vs button, hover vs click).

## Quick Reference : The 8 Decision Families

| Family | Question to ask | Primitives | Tie-breaker |
|--------|----------------|------------|-------------|
| Modal-class | Where does the user focus go and what dismisses it ? | Dialog / Sheet / Drawer / AlertDialog | Viewport + gesture vs button + blocking-vs-not |
| Selector-class | How many options and does the user need to search ? | Native Select / Select / Combobox / Command | Option count + search need + multi-select |
| Menu-class | What triggers the menu open ? | DropdownMenu / ContextMenu / Menubar / NavigationMenu | Button click vs right-click vs app-bar vs site-nav |
| Floating-class | Does the floating content contain interactive elements ? | Tooltip / HoverCard / Popover | Hover-only vs hover-preview vs click-with-interaction |
| Notification-class | Must the user acknowledge before continuing ? | Sonner (toast) / Alert (inline) / AlertDialog | Non-blocking ephemeral vs persistent banner vs blocking confirm |
| Loading-class | Is the structure known or unknown ? | Skeleton / Spinner / Card-with-content | Known layout shape vs indeterminate vs already-data-shaped |
| Form-class | Which form library is in play ? | Form (RHF) / Field (library-agnostic) | react-hook-form-locked vs shared/agnostic |
| Table-class | Does the user need sort, filter, pagination, or column-visibility ? | Table / DataTable recipe | Static rows vs any-table-interaction |

ALWAYS run the family-classification step first. NEVER skip to picking a primitive. If the requirement spans two families (e.g., "a popover with a searchable list inside") compose : Popover + Command, never invent a new primitive.

## Validator Workflow

Given a UI requirement R (plain English string OR a draft component tree), execute :

1. **Read R out loud as an interaction**. What does the user DO ? What do they SEE ? What MUST happen before they proceed ?
2. **Classify into ONE family** from the table above. If R spans two families, decompose R into a parent family and a child family.
3. **Apply the per-family decision table** (in `references/methods.md`). Each row gives a discriminating criterion and a primitive recommendation.
4. **Recommend the primitive** AND state WHY (which discriminating criterion drove the pick).
5. **Cross-reference to the syntax skill** that covers the chosen primitive (matrix below).
6. **If R names a wrong primitive**, output a WRONG / RIGHT / WHY triple and forward-point to the syntax skill. ALWAYS cite the discriminating criterion ; NEVER recommend by aesthetic preference.

## Family 1 : Modal-class (Dialog vs Sheet vs Drawer vs AlertDialog)

All four overlay content. They differ on viewport, gesture, and blocking.

| User intent | Primitive | Discriminating criterion |
|-------------|-----------|--------------------------|
| Centered overlay on desktop, small-to-medium content | `Dialog` | Centered. No edge. Closes via X, Esc, overlay-click. NOT gesture-aware. |
| Edge-anchored panel (settings, side-nav, side-edit) on desktop | `Sheet` | Anchored to `side="top|right|bottom|left"`. Closes via X, Esc, overlay-click. Mirrors Dialog API. |
| Mobile bottom-sheet with gesture dismiss (drag-to-close) | `Drawer` | Wraps `vaul`. Gesture-aware. Mobile-first. `direction="top|right|bottom|left"`. Snap-points supported. |
| Destructive confirm ("Delete this record ?") | `AlertDialog` | Forces user choice. NO overlay-click-to-dismiss. Explicit AlertDialogAction / AlertDialogCancel. |

ALWAYS use AlertDialog (NEVER Dialog) for destructive confirmation. AlertDialog disables overlay-click and Esc-to-dismiss patterns that Dialog allows, which is the entire point.

ALWAYS use Drawer (NEVER Dialog) on mobile when the user expects a bottom-sheet with drag-to-close. The documented responsive pattern is `useMediaQuery` -> Dialog on desktop, Drawer on mobile. See `shadcn-impl-responsive-dialog-drawer`.

NEVER use Sheet for a destructive confirmation. NEVER use Drawer on desktop unless the design specifically requires gesture-aware panels (rare). NEVER use Dialog for a mobile bottom-sheet ; the X-button-only dismiss feels wrong against platform conventions.

Cross-references : `shadcn-syntax-dialog`, `shadcn-syntax-sheet`, `shadcn-syntax-drawer`, `shadcn-impl-responsive-dialog-drawer`.

## Family 2 : Selector-class (Native Select vs Select vs Combobox vs Command)

All four pick from a list. They differ on option count, search, and mobile / a11y.

| User intent | Primitive | Discriminating criterion |
|-------------|-----------|--------------------------|
| Small fixed option list (under ~10), prioritise mobile + screen-reader compatibility | `Native Select` | Native `<select>`. Best mobile keyboard. Best screen-reader. No styling beyond Tailwind. |
| Small-to-medium option list, custom styling needed (icons in options, descriptions) | `Select` | Radix Select. Custom rendering. Keyboard-navigable. NOT searchable. |
| Medium-to-large list, user needs to search by typing | `Combobox` | Popover + Command composition since 2026 has a dedicated Combobox API. Searchable. |
| Command palette (Cmd+K, fuzzy across many commands) | `Command` | `cmdk`-based. Global keyboard launcher. Groups, separators, shortcuts. |

ALWAYS use Combobox (NEVER Select) when option count exceeds ~10 OR when search is needed. The Select primitive has no built-in search ; forcing search into Select requires re-implementing the Combobox composition.

ALWAYS use Native Select when the option list is small AND mobile / accessibility is the priority. Radix Select renders a custom popper that is harder for some assistive tech than the native control.

NEVER use Command as a regular form field selector. Command is intended as a command-palette launcher (typically inside a CommandDialog triggered by Cmd+K), not as an in-form dropdown. The in-form searchable selector is Combobox.

Cross-references : `shadcn-syntax-selectors` (covers Native Select + Select + Combobox), `shadcn-syntax-command`.

## Family 3 : Menu-class (DropdownMenu vs ContextMenu vs Menubar vs NavigationMenu)

All four open a menu. They differ on the trigger and on the role.

| User intent | Primitive | Discriminating criterion |
|-------------|-----------|--------------------------|
| Button-click reveals a menu of actions | `DropdownMenu` | Left-click trigger. Most common. Actions, checkboxes, radio, submenus. |
| Right-click on a region reveals contextual actions | `ContextMenu` | Right-click (or long-press on touch). API mirrors DropdownMenu. |
| Desktop-app-style horizontal menu bar (File / Edit / View) | `Menubar` | Persistent bar with hover-over-trigger expansion. App-shell only. |
| Top-level site navigation with rich-content flyouts | `NavigationMenu` | Hover-and-click site nav. Wide flyout content with grid panels and links. |

ALWAYS use ContextMenu (NEVER DropdownMenu) when the trigger semantic is right-click. Right-click is a platform gesture ; reusing DropdownMenu loses the platform contract.

ALWAYS use NavigationMenu (NEVER DropdownMenu) for top-level site nav with rich content. DropdownMenu is for actions (verbs) ; NavigationMenu is for navigation (nouns / destinations).

NEVER use Menubar for a one-off button-triggered menu. Menubar is a persistent application chrome surface ; using it as a single button is over-engineering.

Cross-references : `shadcn-syntax-menu-primitives` (covers DropdownMenu + ContextMenu + Menubar + NavigationMenu).

## Family 4 : Floating-class (Tooltip vs HoverCard vs Popover)

All three float above the page on a trigger. They differ on trigger event and on whether the floating content is interactive.

| User intent | Primitive | Discriminating criterion |
|-------------|-----------|--------------------------|
| Short non-interactive hint on hover (label, abbreviation) | `Tooltip` | Hover-only. Content MUST NOT be interactive (a11y forbids). String / short fragment only. |
| Rich hover-preview (user card, link-preview) NON-interactive primary content | `HoverCard` | Hover with delay. Static rich content. May contain links but primary intent is preview. |
| Click-triggered floating content with form, buttons, complex interaction | `Popover` | Click trigger. Anchored content. Fully interactive. Forms, date-pickers, filter-builders. |

ALWAYS use Popover (NEVER Tooltip) when the floating content contains a button, link, form, or any other interactive element. Tooltip is hover-only and screen-readers expose its content as a label, not as an interactive region. Putting a button inside Tooltip violates ARIA.

ALWAYS use HoverCard (NEVER Tooltip) when the hover-preview is a rich card (avatar + bio + stats). Tooltip is constrained to short string content.

NEVER use Popover where Tooltip works. Popover requires an intentional click and has heavier popper machinery ; reaching for it just to show "Save" on a button is over-engineered.

Cross-references : `shadcn-syntax-popover-tooltip-hovercard`.

## Family 5 : Button-class (Button vs Link vs asChild composition)

Buttons and links share visual style in shadcn. They differ semantically and on Slot-composition.

| User intent | Primitive | Discriminating criterion |
|-------------|-----------|--------------------------|
| Action that changes state or submits data | `<Button onClick>` | Verb. No href. Pure action. |
| Action that semantically navigates | `<Button asChild><Link href /></Button>` | Noun / destination. asChild forwards Slot props to Link. Preserves prefetch + middle-click. |
| Text-link inline in prose | `Button variant="link"` or styled `<a>` | If interactive within text, variant="link" matches button focus-state. |

ALWAYS use `asChild` + `<Link>` (NEVER a bare `<button onClick={() => router.push()}>`) when the action is navigation. The bare-onClick variant loses Next.js prefetch, breaks middle-click open-in-new-tab, breaks keyboard "open link" semantics, and breaks SEO crawl.

NEVER pass two children to `<Button asChild>`. Slot.Root accepts exactly one. Anti-pattern : `<Button asChild><Link>label</Link><Icon/></Button>` will throw. Compose : `<Button asChild><Link>{icon}label</Link></Button>`.

Cross-references : `shadcn-syntax-button`, `shadcn-syntax-variant-cva`.

## Family 6 : Notification-class (Sonner vs Alert vs AlertDialog)

All three communicate status. They differ on persistence and on whether the user MUST acknowledge.

| User intent | Primitive | Discriminating criterion |
|-------------|-----------|--------------------------|
| Non-blocking transient confirmation ("Saved", "Copied") | `Sonner` (`toast.success`) | Auto-dismiss. Bottom or top corner. Stacks. Not in user flow. |
| Inline persistent status (form-level error banner, page-level warning) | `Alert` | In document flow. No auto-dismiss. `variant="default | destructive"`. |
| Blocking confirmation before destructive action | `AlertDialog` | Modal. Disables overlay-dismiss. Requires explicit Action / Cancel. |

ALWAYS use AlertDialog (NEVER Sonner toast) for destructive confirmation. Toast is dismissible by ignoring it ; a destructive action must require explicit confirmation. The legacy Toast component has been REMOVED from the catalogue ; new projects MUST use Sonner.

ALWAYS use Alert (NEVER Sonner) for persistent status that belongs in document flow. Toast is ephemeral ; a form-level "3 fields are invalid" banner must remain visible while the user fixes them.

NEVER use Sonner for any flow where the user might miss the message and continue (payment confirmation, irreversible delete, account-deletion success). Pair Sonner only with reversible, low-consequence actions.

Cross-references : `shadcn-syntax-toast-sonner`, `shadcn-errors-form-state` (for Alert in form context), Dialog (for blocking choice that is not destructive).

## Family 7 : Loading-class (Skeleton vs Spinner vs Card / Sonner)

All three communicate "work in progress". They differ on whether the final shape is known.

| User intent | Primitive | Discriminating criterion |
|-------------|-----------|--------------------------|
| The final layout shape is known (avatar + 3 lines, card grid) | `Skeleton` | Layout-matching placeholder. Reduces CLS. Best perceived performance. |
| The final shape is unknown OR very small (button, inline) | `Spinner` (new 2026) | Indeterminate animation. Use inline in a button, or as a single page-level indicator. |
| Background long-running task with progress feedback | `Sonner` + `toast.promise()` | Progress messaging without blocking. |

ALWAYS use Skeleton (NEVER Spinner) when the final shape is known. Skeleton matches the post-load layout so the page does not jump (cumulative layout shift). Spinner gives no preview.

NEVER fill an entire page with a centered Spinner when you know the layout shape ahead of time. Skeleton with the exact layout is the documented choice.

NEVER mount a Skeleton for an action that completes in under ~200 ms. Below that threshold the Skeleton flashes and looks jankier than no indicator at all.

Cross-references : `shadcn-syntax-button` (loading button pattern), `shadcn-syntax-toast-sonner` (toast.promise pattern).

## Family 8 : Form-class (Form vs Field primitive)

Both wire a11y. They differ on form-library coupling.

| User intent | Primitive | Discriminating criterion |
|-------------|-----------|--------------------------|
| react-hook-form + zod project, all forms use RHF | `Form` (FormField / FormItem / FormLabel / FormControl / FormDescription / FormMessage) | Composes RHF Controller. Auto-wires aria. RHF-locked. |
| TanStack Form project, OR shared library across form libs, OR no form-state library at all | `Field` (FieldLabel / FieldDescription / FieldError / FieldGroup / FieldSet / FieldLegend) | New 2026 primitive. a11y wiring decoupled from RHF. Library-agnostic. |

ALWAYS use Form (NEVER Field alone) inside a react-hook-form + zod project. The Form component is the canonical RHF integration ; using Field strips you of the auto-wired Controller binding.

ALWAYS use Field (NEVER Form) when the form is built without react-hook-form, OR when the component must work across react-hook-form AND TanStack Form. Field provides the same a11y wiring without the RHF dependency.

NEVER mix Form and Field for the same logical form. Pick one layer per form. Mixing creates two competing aria-describedby chains.

Cross-references : `shadcn-syntax-form`, `shadcn-syntax-field`, `shadcn-impl-form-validation`, `shadcn-errors-form-state`.

## Family 9 : Table-class (Table vs DataTable recipe)

The shadcn Table is a styled HTML primitive. DataTable is a documented recipe that composes Table with TanStack Table v8.

| User intent | Primitive | Discriminating criterion |
|-------------|-----------|--------------------------|
| Static rows. No sort. No filter. No pagination. No column visibility. | `Table` (Table / TableHeader / TableBody / TableRow / TableCell / TableHead) | Pure HTML. Zero JS state. |
| ANY of : sortable columns, filterable rows, pagination, row-selection, column visibility, virtualisation | `DataTable` recipe | Composes `@tanstack/react-table` v8 with Table primitives. State via useReactTable. |

ALWAYS use the DataTable recipe (NEVER plain Table) the moment the user mentions sort / filter / pagination / column-visibility / row-selection. The plain Table primitive has zero state machinery ; re-implementing sort or pagination by hand is the canonical wrong-component pick.

NEVER reach for DataTable for a 3-row static summary. Plain Table is leaner and renders without `"use client"`.

Cross-references : `shadcn-syntax-table`, `shadcn-impl-data-table`.

## Cross-Reference Matrix : Decision -> Syntax Skill

| Decision domain | Authoritative syntax skill |
|-----------------|----------------------------|
| Dialog / AlertDialog | `shadcn-syntax-dialog` |
| Sheet | `shadcn-syntax-sheet` |
| Drawer | `shadcn-syntax-drawer` |
| Responsive Dialog + Drawer pattern | `shadcn-impl-responsive-dialog-drawer` |
| Native Select + Select + Combobox | `shadcn-syntax-selectors` |
| Command palette (cmdk) | `shadcn-syntax-command` |
| DropdownMenu + ContextMenu + Menubar + NavigationMenu | `shadcn-syntax-menu-primitives` |
| Tooltip + HoverCard + Popover | `shadcn-syntax-popover-tooltip-hovercard` |
| Button + Link + asChild | `shadcn-syntax-button` |
| cva variant API | `shadcn-syntax-variant-cva` |
| Sonner toast | `shadcn-syntax-toast-sonner` |
| Form (react-hook-form + zod) | `shadcn-syntax-form` |
| Field (new 2026 primitive) | `shadcn-syntax-field` |
| Table primitive | `shadcn-syntax-table` |
| DataTable (TanStack Table v8) | `shadcn-impl-data-table` |
| Form validation workflow | `shadcn-impl-form-validation` |
| Form state errors | `shadcn-errors-form-state` |
| Calendar / DatePicker | `shadcn-syntax-calendar-datepicker` |
| Input OTP | `shadcn-syntax-input-otp` |
| Chart (Recharts) | `shadcn-syntax-chart` |
| Sidebar | `shadcn-syntax-sidebar` |
| Layout primitives (ScrollArea, Separator, AspectRatio, Resizable) | `shadcn-syntax-layout-primitives` |
| RSC / use client boundaries | `shadcn-impl-rsc-vs-client-boundaries` |
| Component install / overwrite / diff | `shadcn-impl-component-install` |
| Theming (tokens, dark mode, oklch) | `shadcn-impl-theming-custom`, `shadcn-core-theming` |

## Validator Output Format

When acting as a validator on a user requirement R or a draft tree D, emit :

```
Family: <one of the 8 families>
Recommended primitive: <name>
Reason: <one sentence citing the discriminating criterion>
Wrong picks flagged: <if any : list with WHY each is wrong>
Forward-pointer: <syntax skill name>
```

ALWAYS emit a forward-pointer. NEVER recommend a primitive without naming the syntax skill that covers its API.

## Companion Skills

- `shadcn-syntax-dialog` : Dialog + AlertDialog API and patterns
- `shadcn-syntax-sheet` : Sheet API (Dialog-extends, side prop)
- `shadcn-syntax-drawer` : Drawer (vaul) API and mobile gesture model
- `shadcn-syntax-selectors` : Native Select + Select + Combobox API
- `shadcn-syntax-command` : Command / cmdk palette API
- `shadcn-syntax-menu-primitives` : DropdownMenu + ContextMenu + Menubar + NavigationMenu API
- `shadcn-syntax-popover-tooltip-hovercard` : floating-class API discrimination
- `shadcn-syntax-button` : Button + asChild Slot composition
- `shadcn-syntax-toast-sonner` : Sonner API including toast.promise
- `shadcn-syntax-form` : react-hook-form integration
- `shadcn-syntax-field` : new 2026 Field primitive
- `shadcn-syntax-table` : pure HTML Table primitive
- `shadcn-impl-data-table` : TanStack Table v8 DataTable recipe
- `shadcn-impl-responsive-dialog-drawer` : Dialog-desktop / Drawer-mobile pattern
- `shadcn-impl-form-validation` : end-to-end form validation workflow
- `shadcn-errors-form-state` : form-state error patterns
- `shadcn-impl-rsc-vs-client-boundaries` : per-component "use client" placement

## References

- `references/methods.md` : full decision tables per family with discrimination criteria columns
- `references/examples.md` : 8 worked validation scenarios (WRONG -> RIGHT -> WHY)
- `references/anti-patterns.md` : 8 canonical mis-selection anti-patterns with fixes
