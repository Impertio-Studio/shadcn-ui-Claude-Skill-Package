# Anti-Patterns : Canonical Mis-Selections in the shadcn Catalogue

Each anti-pattern describes a wrong-component pick that recurs in real codebases and AI-generated drafts, names the failure mode, and points to the correct primitive plus the syntax skill that documents it. ALWAYS treat the named anti-pattern as a hard-fail in validation ; NEVER soften the recommendation to preserve the wrong choice.

---

## AP-1 : Tooltip used to hold interactive content

**Symptom** : a button, link, input, or form inside `<TooltipContent>`.

**Why it fails** :

- Tooltip's ARIA role is `tooltip`. Assistive tech exposes the content as a label string, NOT as an interactive region. Keyboard / screen-reader users cannot reach the interactive child.
- Tooltip dismisses on pointer-leave. Moving the pointer toward the button often dismisses Tooltip before the click lands.
- Radix Tooltip explicitly documents this constraint at https://www.radix-ui.com/primitives/docs/components/tooltip.

**Fix** : if the floating content has a primary action, use `Popover` (click-triggered, fully interactive). If the floating content is a rich preview with at most a secondary link, use `HoverCard`.

Forward-pointer : `shadcn-syntax-popover-tooltip-hovercard`.

---

## AP-2 : Dialog used as a mobile bottom-sheet

**Symptom** : on mobile, a centered Dialog appears for what UX expects to be a draggable bottom-sheet.

**Why it fails** :

- Dialog has NO gesture awareness. The user expects drag-to-close ; the Dialog only closes on X / Esc / overlay.
- Dialog centers the content. On a narrow viewport the X-button position and the keyboard-on-focus interaction feel wrong against platform conventions (iOS, Android).
- The documented responsive pattern in shadcn is `useMediaQuery` -> Dialog on desktop, Drawer on mobile.

**Fix** : use `Drawer` (which wraps `vaul`) on mobile, with `Dialog` on desktop via a media-query switch.

Forward-pointer : `shadcn-syntax-drawer`, `shadcn-impl-responsive-dialog-drawer`.

---

## AP-3 : Select used for a searchable or large option list

**Symptom** : Select with > ~10 SelectItem entries, OR Select with a hand-rolled SelectInput at the top to filter items.

**Why it fails** :

- Select has no built-in search. Forcing search into Select means re-implementing Combobox by hand (filter state, keyboard navigation, no-results-empty state, escape-to-close-on-empty).
- Long Select lists are slow to scroll, hard to skim, and pose mobile usability problems (full-screen popper on touch).

**Fix** : use `Combobox` (Popover + Command composition, or 2026 dedicated Combobox API). CommandInput provides search, CommandList virtualises results, CommandEmpty handles no-results.

Forward-pointer : `shadcn-syntax-selectors`, `shadcn-syntax-command`.

---

## AP-4 : DropdownMenu rebound to right-click

**Symptom** : `<div onContextMenu={...}>` wrapping a DropdownMenu, OR manual right-click event bindings to open a DropdownMenu.

**Why it fails** :

- Right-click is a platform gesture. ContextMenu is purpose-built for it ; it handles long-press on touch, suppresses the native browser context menu correctly, and uses the same API as DropdownMenu so there is zero learning cost.
- Hand-rolling `onContextMenu` skips touch handling. Mobile users with long-press will not get the menu (or worse, will get both the native menu AND the custom one).
- A custom right-click rebind is brittle : keyboard equivalent (Shift+F10, Menu key) is missed.

**Fix** : use `ContextMenu` directly. Its trigger fires on right-click and long-press automatically.

Forward-pointer : `shadcn-syntax-menu-primitives`.

---

## AP-5 : Sonner toast used for a destructive confirmation

**Symptom** : `toast(...)` with an "Undo" action button, followed by a setTimeout that commits the destructive operation.

**Why it fails** :

- Toast is dismissible by ignoring it. Users routinely miss toasts because they are not in the user's focus path.
- A destructive action that the user can MISS the confirmation for is a real-world data-loss bug. "I clicked Delete and never saw the Undo toast" is a frequent support ticket.
- AlertDialog disables overlay-click and Esc-to-dismiss patterns specifically so the user is forced to choose Action or Cancel.

**Fix** : use `AlertDialog` (NEVER `Sonner` toast) for any destructive / irreversible action. Reserve toast for non-blocking confirmations of REVERSIBLE actions ("Saved", "Copied").

Forward-pointer : `shadcn-syntax-dialog` (AlertDialog section), `shadcn-syntax-toast-sonner`.

---

## AP-6 : Plain Table used where the user asked for sorting or filtering

**Symptom** : `<TableHead onClick={() => setSortKey('name')}>` plus a `useMemo` over the sorted array, plus a filter input wired by hand.

**Why it fails** :

- Re-implementing TanStack Table by hand grows quickly into hundreds of lines, fails to support multi-column sorting correctly, breaks on tie-breaking, and tends to lose typing.
- The shadcn docs explicitly route any non-static table to the DataTable recipe (`useReactTable` + the Table primitives). The plain Table is for static rows only.
- A hand-rolled solution will not get pagination, column visibility, row selection, or virtualisation later without further reinvention.

**Fix** : the moment the requirement names sort, filter, pagination, column-visibility, row-selection, or virtualisation, switch to the DataTable recipe.

Forward-pointer : `shadcn-syntax-table`, `shadcn-impl-data-table`.

---

## AP-7 : Legacy Toast component used instead of Sonner

**Symptom** : `import { useToast } from "@/hooks/use-toast"` and `<Toaster />` from a deprecated path.

**Why it fails** :

- The Radix-based Toast component and its `useToast` hook have been REMOVED from the shadcn catalogue. New `shadcn add toast` installs Sonner.
- Old AI training data and stale tutorials still emit `useToast` calls. They will fail to import in a fresh project.

**Fix** : use `Sonner` (`<Toaster />` mounted at root, then `toast(...)`, `toast.success`, `toast.error`, `toast.info`, `toast.warning`, `toast.promise`).

Forward-pointer : `shadcn-syntax-toast-sonner`.

---

## AP-8 : Form (RHF) and Field mixed in the same form

**Symptom** : a single logical form uses some FormField wrappers from the Form layer and some Field wrappers from the Field layer.

**Why it fails** :

- Both layers wire `aria-describedby`. Mixing produces two competing chains and inconsistent screen-reader output ("Email Email is invalid is invalid").
- Both layers wire `aria-invalid`. Mixed errors may not surface to assistive tech on the right element.
- The validator layers diverge : FormField is bound to RHF's `Controller` ; Field is library-agnostic. Mixing means the form-state and the displayed error can drift.

**Fix** : pick ONE layer per form. Use `Form` (FormField / FormItem ...) inside a react-hook-form project. Use `Field` (FieldLabel / FieldDescription / FieldError ...) when the form is shared across libraries or uses TanStack Form or no form library.

Forward-pointer : `shadcn-syntax-form`, `shadcn-syntax-field`, `shadcn-impl-form-validation`.

---

## AP-9 : Menubar used for a one-off action menu

**Symptom** : a single button-triggered menu implemented with Menubar / MenubarMenu / MenubarTrigger.

**Why it fails** :

- Menubar is desktop-app application chrome. Its visual model is a PERSISTENT horizontal bar (File / Edit / View). Using it for one button looks wrong, takes too much vertical space, and confuses screen-reader users (the bar role announces as `menubar`, expected to be persistent).
- The chosen primitive should match the interaction model, not the visual default.

**Fix** : use `DropdownMenu` for a one-off action menu. Reserve `Menubar` for true application-shell scenarios.

Forward-pointer : `shadcn-syntax-menu-primitives`.

---

## AP-10 : Popover used where Tooltip works

**Symptom** : every short string hint is implemented as a Popover with a click trigger.

**Why it fails** :

- Tooltip is hover/focus-triggered ; users expect labels to surface on hover with no click required.
- Popover requires an intentional click and renders heavier popper machinery. Using it for a 6-character label is over-engineered.
- Screen-reader expectations differ : tooltip role announces as the trigger's accessible name supplement ; popover role does not.

**Fix** : use `Tooltip` for short non-interactive hint content. Reserve `Popover` for click-triggered interactive content (forms, date pickers, filter builders).

Forward-pointer : `shadcn-syntax-popover-tooltip-hovercard`.

---

## Summary Table

| # | Anti-pattern | Wrong primitive | Right primitive | Forward pointer |
|---|--------------|-----------------|-----------------|-----------------|
| AP-1 | Tooltip holding interactive content | Tooltip | Popover (or HoverCard for previews) | shadcn-syntax-popover-tooltip-hovercard |
| AP-2 | Dialog on mobile bottom-sheet | Dialog | Drawer (+ responsive Dialog on desktop) | shadcn-syntax-drawer, shadcn-impl-responsive-dialog-drawer |
| AP-3 | Select for searchable / large list | Select | Combobox | shadcn-syntax-selectors |
| AP-4 | DropdownMenu rebound to right-click | DropdownMenu | ContextMenu | shadcn-syntax-menu-primitives |
| AP-5 | Sonner toast for destructive confirm | Sonner | AlertDialog | shadcn-syntax-dialog, shadcn-syntax-toast-sonner |
| AP-6 | Plain Table for sortable / filterable data | Table | DataTable recipe | shadcn-syntax-table, shadcn-impl-data-table |
| AP-7 | Legacy Toast instead of Sonner | Toast (legacy) | Sonner | shadcn-syntax-toast-sonner |
| AP-8 | Form + Field mixed in one form | both | pick one layer | shadcn-syntax-form, shadcn-syntax-field |
| AP-9 | Menubar for one-off action menu | Menubar | DropdownMenu | shadcn-syntax-menu-primitives |
| AP-10 | Popover where Tooltip works | Popover | Tooltip | shadcn-syntax-popover-tooltip-hovercard |
