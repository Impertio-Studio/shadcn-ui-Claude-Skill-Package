---
name: shadcn-syntax-dialog
description: >
  Use when adding, debugging, or refactoring a shadcn ui Dialog (a focus-
  trapped modal that overlays the page), composing its ten primitives
  (Dialog / DialogTrigger / DialogPortal / DialogOverlay / DialogContent
  / DialogHeader / DialogFooter / DialogTitle / DialogDescription /
  DialogClose), deciding between controlled (`open` + `onOpenChange`)
  and uncontrolled (`defaultOpen` or no state at all) operation,
  visually hiding the title with an sr-only span while keeping screen-
  reader compliance, closing the dialog after a form submit, swapping
  the default close button via `showCloseButton={false}` on
  DialogContent, retargeting the portal container, or wrapping a
  custom button via `<DialogClose asChild>`.
  Prevents the canonical Dialog failures : passing `open` without
  `onOpenChange` (the dialog becomes read-only and uncloseable),
  omitting DialogTitle (Radix dev-mode throws an a11y warning + axe
  flags it as critical), rendering DialogContent outside DialogPortal
  (z-index conflicts and stacking-context bugs in transformed
  ancestors), using `<DialogClose asChild>` with a non-button child
  whose component swallows merged props, missing the synchronous
  `onOpenChange(false)` call inside a form-submit handler so the
  dialog stays open after success, and stacking nested dialogs without
  separate portal containers so the inner dialog ends up beneath the
  outer overlay.
  Covers the ten dialog primitives verbatim from `apps/v4/registry/
  new-york-v4/ui/dialog.tsx`, the controlled-state contract that
  Radix enforces, the implicit Portal+Overlay wrap inside
  DialogContent, the v4 `showCloseButton` prop on DialogContent and
  DialogFooter, the mandatory `"use client"` directive, the
  `data-state="open|closed"` attribute used for enter/exit animation,
  the sr-only DialogTitle pattern via VisuallyHidden, the
  `aria-describedby={undefined}` opt-out, the `container` prop on
  DialogPortal for stacking-context fixes, and the pointer to the
  responsive Dialog-on-desktop / Drawer-on-mobile recipe.
  Keywords: shadcn dialog, modal, popup, DialogTitle, DialogTrigger,
  DialogContent, DialogPortal, DialogOverlay, DialogClose,
  DialogHeader, DialogFooter, DialogDescription, controlled dialog,
  uncontrolled dialog, open onOpenChange, defaultOpen, showCloseButton,
  modal prop, focus trap, Esc to close, sr-only DialogTitle,
  VisuallyHidden Title, aria-describedby undefined, dialog
  accessibility warning, missing DialogTitle warning, Radix dialog
  console warning, asChild dialog close, dialog form submit, close
  dialog after submit, dialog z-index, dialog Portal stacking context,
  Portal container, nested dialogs, how do I open a modal, how do I
  close a modal, modal not closing, dialog stays open after submit,
  dialog cannot be closed, dialog read-only, dialog open prop
  required, dialog stuck open, dialog blank screen.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui : Dialog Syntax

The Dialog component is shadcn's reference focus-trapped modal. It wraps the Radix Dialog primitive, adds default styling, and exposes a ten-part composition surface. Every dialog-shaped primitive in the catalogue (AlertDialog, Sheet, Drawer, CommandDialog, the mobile Sidebar variant) is either a thin extension of this surface or a parallel implementation following the same controlled-state and a11y contract. Mastering Dialog means mastering the whole family.

## Quick Reference

### The 10 primitives

| Primitive | Purpose |
|-----------|---------|
| `Dialog` | Root state container ; owns `open` / `onOpenChange` / `defaultOpen` / `modal` |
| `DialogTrigger` | Element that opens the dialog ; `asChild`-aware |
| `DialogPortal` | Renders Overlay + Content inside `document.body` (or `container` prop) |
| `DialogOverlay` | Backdrop ; styled by shadcn with `bg-black/50` + fade animation |
| `DialogContent` | Main surface ; AUTO-WRAPS itself in `DialogPortal` + `DialogOverlay` ; exposes `showCloseButton` |
| `DialogHeader` | Layout div for Title + Description ; pure Tailwind, no Radix part |
| `DialogFooter` | Layout div for action buttons ; pure Tailwind ; exposes optional `showCloseButton` |
| `DialogTitle` | Accessible name (REQUIRED for screen-reader compliance) |
| `DialogDescription` | Accessible description ; OPTIONAL but recommended |
| `DialogClose` | Closes the dialog ; `asChild`-aware ; wrap any button |

### Minimal Dialog (uncontrolled)

```tsx
"use client"

import {
  Dialog, DialogTrigger, DialogContent,
  DialogHeader, DialogFooter,
  DialogTitle, DialogDescription, DialogClose,
} from "@/components/ui/dialog"
import { Button } from "@/components/ui/button"

<Dialog>
  <DialogTrigger asChild>
    <Button>Open</Button>
  </DialogTrigger>
  <DialogContent>
    <DialogHeader>
      <DialogTitle>Are you sure?</DialogTitle>
      <DialogDescription>This will permanently delete the file.</DialogDescription>
    </DialogHeader>
    <DialogFooter>
      <DialogClose asChild>
        <Button variant="outline">Cancel</Button>
      </DialogClose>
      <Button variant="destructive">Delete</Button>
    </DialogFooter>
  </DialogContent>
</Dialog>
```

Notice : no `open` / `onOpenChange` props. Radix manages state internally. DialogTrigger and DialogClose update it.

### Five invariants

1. ALWAYS provide a `DialogTitle`. NEVER omit it ; Radix logs a dev-mode warning and axe-core flags missing accessible-name as `critical`.
2. ALWAYS either control BOTH `open` and `onOpenChange`, or control NEITHER. NEVER pass `open` alone (the dialog becomes a frozen prop with no way to close it).
3. ALWAYS keep the `"use client"` directive at the top of `components/ui/dialog.tsx`. NEVER strip it ; Dialog uses Radix client-only hooks and a Portal.
4. ALWAYS render `<DialogContent>` directly (it wraps itself in `DialogPortal` + `DialogOverlay`). NEVER hand-wrap `DialogContent` in your own portal unless you are deliberately retargeting `container`, in which case use the explicit `<DialogPortal container={el}>` form.
5. ALWAYS wrap a non-default close affordance via `<DialogClose asChild>` so a single child receives the click handler. NEVER place two children inside `<DialogClose asChild>` (Radix Slot rejects multi-child).

## Decision Tree 1 : Controlled or uncontrolled?

```
Q1. Do you need the dialog state to MIRROR a value outside the dialog
    (e.g., a `selectedId` URL param, a Zustand store, a parent form
    state, an effect that opens the dialog on data-arrival)?
    no  -> uncontrolled. Use <Dialog> with no props.
    yes -> Q2

Q2. Are you willing to OWN both `open` and `onOpenChange` (every render
    you write must pass both, and onOpenChange must call setOpen)?
    no  -> stay uncontrolled and lift state via a ref or event callback
           on the trigger element ; do NOT pass `open` alone.
    yes -> ALWAYS pass <Dialog open={state} onOpenChange={setState}>.
           Implement onOpenChange as a STABLE setter that accepts a
           boolean and writes it to your state container.
```

NEVER pass `open` without `onOpenChange`. Radix treats the prop as authoritative and ignores Trigger/Close/Esc, leaving the dialog frozen in its initial open state.

NEVER pass `onOpenChange` without `open`. The callback fires but the displayed state is still uncontrolled, so your handler runs against stale closed state.

For a one-shot "default to open on mount" without ongoing control, use `<Dialog defaultOpen>` and skip controlled state entirely.

## Decision Tree 2 : Where should DialogContent render?

```
Q1. Is the dialog inside a transformed ancestor (any element with
    `transform`, `filter`, `perspective`, `will-change`, or a CSS
    container query that creates a new stacking context)?
    no  -> default. <DialogContent> auto-portals to document.body.
           Nothing extra to do.
    yes -> Q2

Q2. Does the dialog NEED to be visually anchored inside the
    transformed ancestor (rare ; almost always a design mistake) ?
    no  -> default. Auto-portal escapes the stacking context and
           the dialog renders correctly on top of everything.
    yes -> Use the EXPLICIT portal form :
           <DialogPortal container={hostElement}>
             <DialogContent>...</DialogContent>
           </DialogPortal>
           and accept that the dialog is constrained to the
           ancestor's stacking context.
```

DialogContent already calls `DialogPortal` + `DialogOverlay` internally (verified verbatim from `apps/v4/registry/new-york-v4/ui/dialog.tsx`). The explicit `<DialogPortal>` is only required when you must override `container` or `forceMount`.

## Decision Tree 3 : Hide the title visually but keep a11y?

```
Q1. Does the dialog need a VISIBLE title?
    yes -> <DialogTitle>Heading</DialogTitle>. Done.
    no  -> Q2

Q2. Does the dialog still need to ANNOUNCE a name to screen readers?
    (yes : every dialog does, per WAI-ARIA Dialog pattern)
    -> ALWAYS wrap DialogTitle in `sr-only` :
       <DialogTitle className="sr-only">Settings</DialogTitle>
       OR use the Radix VisuallyHidden primitive :
       <VisuallyHidden asChild><DialogTitle>Settings</DialogTitle></VisuallyHidden>
       NEVER drop DialogTitle entirely.
```

The `sr-only` className route is the shadcn-canonical pattern and what the Sidebar mobile variant uses internally (per issue #5746, 49 reactions).

## The 10 primitives in detail

### Dialog (Root)

Owns the state. Props :

- `open?: boolean` : controlled visibility. PAIR WITH `onOpenChange`.
- `defaultOpen?: boolean` : uncontrolled initial state. EXCLUDES `open`.
- `onOpenChange?: (open: boolean) => void` : called when Radix wants to change visibility (Trigger click, Close click, Esc, pointer-down-outside, programmatic).
- `modal?: boolean` (default `true`) : when true, focus is trapped and the rest of the page is `aria-hidden` ; when false, the page stays interactive (rarely useful).

ALWAYS choose either `defaultOpen` or `open`+`onOpenChange`. NEVER both ; Radix prefers the controlled prop and ignores the default.

### DialogTrigger

Element that toggles `open` to true. Default render is a `<button type="button">`. Use `asChild` to wrap any focusable element. Receives the `data-state="open|closed"` attribute for styling.

```tsx
<DialogTrigger asChild>
  <Button variant="outline">Edit profile</Button>
</DialogTrigger>
```

NEVER wrap two siblings in `<DialogTrigger asChild>` ; Radix Slot rejects multi-child.

### DialogPortal

Renders Overlay + Content inside `document.body` by default. Props :

- `container?: HTMLElement` : alternate render target. Useful for shadow DOM, iframe roots, deliberately-constrained scopes.
- `forceMount?: boolean` : keep mounted even when `open` is false (lets you animate exit yourself).

DialogContent calls DialogPortal internally with no props. To customise, use the explicit form.

### DialogOverlay

Backdrop. Styled by shadcn with `fixed inset-0 z-50 bg-black/50` + fade animation via `data-[state=open]:animate-in` / `data-[state=closed]:animate-out`.

NEVER set `pointer-events-none` on the overlay : Radix needs pointer events to fire `onPointerDownOutside`, which closes the dialog when modal is true.

### DialogContent

The main surface. Auto-portals (wraps itself in `DialogPortal` + `DialogOverlay`). Props :

- `showCloseButton?: boolean` (default `true`) : shadcn-only addition ; toggles the built-in top-right XIcon close.
- `onOpenAutoFocus?: (event) => void` : called when focus moves into the content on open. `event.preventDefault()` to keep focus where it was.
- `onCloseAutoFocus?: (event) => void` : called when focus moves back on close.
- `onEscapeKeyDown?: (event) => void` : called when Esc is pressed. `event.preventDefault()` to keep open.
- `onPointerDownOutside?: (event) => void` : called when user clicks outside. `event.preventDefault()` to keep open.
- `onInteractOutside?: (event) => void` : union of pointer-down-outside + focus-outside.
- `forceMount?: boolean` : keep mounted for custom exit animation.

ALWAYS render exactly ONE `<DialogContent>` per `<Dialog>`. NEVER conditionally render two and toggle which is shown.

### DialogHeader

Pure layout div with `flex flex-col gap-2 text-center sm:text-left`. Not a Radix part ; safe to omit if your layout differs.

### DialogFooter

Layout div with `flex flex-col-reverse gap-2 sm:flex-row sm:justify-end`. shadcn evergreen-2026 adds an optional `showCloseButton?: boolean` (default `false`) that auto-renders a `<DialogClose asChild><Button variant="outline">Close</Button></DialogClose>` at the end of the children.

### DialogTitle

REQUIRED. Renders an `<h2>` with the dialog's accessible name. Without it, Radix logs `DialogContent requires a DialogTitle for the component to be accessible for screen reader users.` and axe-core flags the dialog as missing an accessible name (rule `aria-dialog-name`, severity critical).

To hide visually : `<DialogTitle className="sr-only">…</DialogTitle>` or wrap with `<VisuallyHidden asChild>`.

### DialogDescription

Optional supplemental text. If you intentionally omit it, ALSO pass `aria-describedby={undefined}` to `<DialogContent>` to suppress the Radix warning about a missing description element.

### DialogClose

Closes the dialog. Default render is a `<button type="button">`. `asChild` to wrap any focusable element. Useful in DialogFooter for an explicit "Cancel" button.

```tsx
<DialogClose asChild>
  <Button variant="outline">Cancel</Button>
</DialogClose>
```

## a11y requirement : DialogTitle is non-optional

The WAI-ARIA Dialog pattern requires every modal dialog to have an accessible name. Radix Dialog implements this by reading `<DialogTitle>` and wiring it via `aria-labelledby`. If DialogTitle is missing :

1. Radix logs a development warning to the console.
2. axe-core / axe DevTools / Lighthouse a11y audit flag the dialog as critical (no accessible name).
3. Screen-reader users hear "dialog" with no further context.

This applies to EVERY dialog, including the mobile Sidebar variant (issue #5746, 49 reactions) and CommandDialog from cmdk.

The fix is always the same : add a DialogTitle, with `sr-only` if invisible. See `references/examples.md` for the visually-hidden recipe.

## Portal rendering and z-index

`<DialogPortal>` mounts its children inside `document.body` by default. This means :

- The dialog escapes any `overflow: hidden`, `transform`, `filter`, or `will-change` ancestor that would otherwise clip it.
- The dialog renders at the top of the DOM, so a `z-50` from the shadcn DialogOverlay sits above any non-portalled `z-*` value below 50 in the rest of the tree.
- Nested dialogs MUST each use their own portal scope. The inner dialog's overlay automatically stacks above the outer (later DOM order = later in `document.body`), as long as both go through DialogPortal.

If you need to retarget (rare) :

```tsx
<DialogPortal container={hostRef.current}>
  <DialogOverlay />
  <DialogContent>…</DialogContent>
</DialogPortal>
```

Note : when you take over `DialogPortal` manually, you ALSO render `<DialogOverlay />` yourself ; DialogContent only auto-wraps if you let it.

NEVER place a `<DialogContent>` directly without either auto-portal or explicit DialogPortal. Z-index ordering becomes ancestor-dependent and unpredictable.

## RSC and the `"use client"` directive

The shadcn `dialog.tsx` ships with `"use client"` at the top. This is non-negotiable :

- Radix Dialog uses React Context internally.
- Dialog state, focus trapping, and Portal mounting all run client-side.
- The `data-state` attribute toggles via React state, not server render.

ALWAYS keep `"use client"` in `components/ui/dialog.tsx`. NEVER strip it ; the file imports `radix-ui` which requires a client boundary.

Components that consume `<Dialog>` may live in Server Components, but the dialog file itself draws the client boundary. See `shadcn-impl-rsc-vs-client-boundaries` for the broader RSC contract.

## Form-inside-dialog : close after submit

The canonical pattern when the dialog body contains a form :

```tsx
const [open, setOpen] = useState(false)
function onSubmit(values) {
  await saveProfile(values)
  setOpen(false) // ALWAYS synchronously after the await
}
```

ALWAYS call `setOpen(false)` (or whatever your `onOpenChange` setter is) inside the success branch of the submit handler. NEVER rely on the form's submit-button being a `<DialogClose>` : if validation fails, `<DialogClose>` still fires and closes the dialog before the user sees the error.

Full recipe in `references/examples.md`.

## Companion Skills

- [shadcn-syntax-sheet](../shadcn-syntax-sheet/SKILL.md) : Dialog with a `side` prop ; same controlled-state contract, side-panel UX, larger content. Read when the surface should slide in from an edge instead of centring.
- [shadcn-syntax-drawer](../shadcn-syntax-drawer/SKILL.md) : Vaul-backed bottom-sheet for mobile ; same a11y contract (DrawerTitle required) but different primitive. Read when the surface is mobile-first.
- [shadcn-impl-responsive-dialog-drawer](../../shadcn-impl/shadcn-impl-responsive-dialog-drawer/SKILL.md) : Dialog-on-desktop / Drawer-on-mobile recipe using `useMediaQuery`. Read when the same UI must adapt across viewports.
- [shadcn-errors-radix-controlled](../../shadcn-errors/shadcn-errors-radix-controlled/SKILL.md) : the controlled-state failure catalogue across all Radix primitives, including `open`-without-`onOpenChange`, Slot multi-child crashes, and Portal stacking-context surprises.
- [shadcn-syntax-button](../shadcn-syntax-button/SKILL.md) : the `asChild` pattern used by DialogTrigger and DialogClose.

## Reference Links

- [references/methods.md](references/methods.md) : full prop signatures for all ten primitives, controlled-state pair, `showCloseButton` toggle, data-attributes.
- [references/examples.md](references/examples.md) : six canonical recipes (uncontrolled, controlled, form-in-dialog, sr-only title, custom portal container, asChild DialogClose).
- [references/anti-patterns.md](references/anti-patterns.md) : six anti-patterns with WHY and FIX.

## Sources

All claims in this skill trace to URLs in `SOURCES.md` :

- https://ui.shadcn.com/docs/components/radix/dialog (canonical Dialog docs, prop tables, showCloseButton documentation)
- https://www.radix-ui.com/primitives/docs/components/dialog (Radix Dialog primitive API, controlled-state contract, accessibility notes, keyboard interactions)
- https://github.com/shadcn-ui/ui (apps/v4/registry/new-york-v4/ui/dialog.tsx, verbatim v4 source ; showCloseButton implementation ; auto-portal wrapping)
- https://github.com/shadcn-ui/ui/issues/5746 (Sidebar mobile DialogTitle requirement, 49 reactions, open ; demonstrates the sr-only DialogTitle pattern)
- https://www.w3.org/WAI/ARIA/apg/patterns/dialog-modal/ (WAI-ARIA Dialog pattern, accessible-name requirement)

Verified 2026-05-19.
