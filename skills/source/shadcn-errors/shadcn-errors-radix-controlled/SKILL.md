---
name: shadcn-errors-radix-controlled
description: >
  Use when a shadcn Dialog / Sheet / Drawer / DropdownMenu / Popover /
  HoverCard / Select / AlertDialog / ContextMenu / Collapsible /
  Accordion / Tabs is stuck open, refuses to close, never opens, fires
  `onOpenChange` at unexpected times, drops focus after a form submit,
  crashes with "React.Children.only expected to receive a single child"
  when wrapped in `asChild`, renders behind another element with higher
  z-index, anchors its popper to the wrong position inside a
  `transform`ed parent, traps focus inside the wrong container, or
  swallows the Escape key. Prevents the half-controlled state trap
  (passing `open` without `onOpenChange` makes the primitive read-only
  and uncloseable), the `asChild` Slot crash (Slot accepts exactly one
  child and that child MUST forward refs), the `asChild` silent-prop-
  swallow (a `<div>` child ignores `onClick` and `aria-*` that Slot
  forwards), the Portal stacking-context trap (a `transform` /
  `filter` / `perspective` / `will-change` ancestor reroots the popper
  coordinate system), the modal=true Popover-with-form focus loss
  (form submit closes the popover before React commits), and the
  controlled-without-close-on-success bug (modal stays open after a
  successful mutation because the parent never calls
  `setOpen(false)`). Covers the open + onOpenChange pair rule, the
  defaultOpen-for-uncontrolled escape hatch, per-primitive controlled
  contracts (Dialog / Sheet / Drawer / DropdownMenu / Popover /
  HoverCard / Select / AlertDialog / ContextMenu / Collapsible /
  Accordion / Tabs / Tooltip), the Slot single-child + forwardRef
  contract, Portal container scoping with the `container` prop, the
  `modal` prop semantics (focus trap, scroll lock, outside-click
  dismissal) for Dialog / Sheet / Drawer / Popover, focus restoration
  via `onOpenAutoFocus` and `onCloseAutoFocus`, the
  close-on-success controlled pattern, and why Tooltip uses
  `TooltipProvider` + delayed state instead of `open`+`onOpenChange`.
  Keywords: radix controlled state, open prop without onOpenChange,
  dialog stuck open, dialog cannot close, dialog read-only, sheet
  will not close, drawer will not close, dropdown menu stuck open,
  popover stuck open, select stuck open, asChild slot crash,
  React.Children.only expected to receive a single child, Slot
  single child, Slot forwardRef, asChild swallows props, asChild
  with div, asChild with Link, asChild with input, portal z-index,
  portal stacking context, transform breaks popper, transformed
  parent radix, will-change radix popper, filter ancestor radix,
  perspective parent popper, modal focus trap, modal scroll lock,
  modal vs non-modal popover, popover form submit focus loss,
  hovercard modal, controlled vs uncontrolled, defaultOpen, why
  is my dialog read-only, why does my dropdown not close, why
  does my popover close on submit, radix state half-binding,
  half controlled radix, close dialog after submit, close
  popover after mutation, set open false after success, escape
  key not closing dialog, escape swallowed, onEscapeKeyDown
  preventDefault, onPointerDownOutside, onInteractOutside,
  onOpenAutoFocus, onCloseAutoFocus, focus restoration, return
  focus to trigger, modal=true popover focus trap, modal=false
  popover, tooltip controlled, TooltipProvider delayDuration,
  AlertDialog cancel button focus, ContextMenu trigger right
  click, Collapsible open onOpenChange, Accordion value
  onValueChange, Tabs value onValueChange, Select value
  onValueChange, radix swallow event, custom keydown handler
  conflict.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026 (Radix UI unified `radix-ui` package, Feb 2026)."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn + Radix Controlled-State Errors : The Open / onOpenChange Pair Rule, asChild Slot, Portal, and modal

Every Radix-wrapped shadcn primitive (Dialog, Sheet, Drawer, DropdownMenu,
Popover, HoverCard, Select, AlertDialog, ContextMenu, Collapsible,
Accordion, Tabs, Tooltip) shares one state-management contract. Break the
contract and the symptom is always one of three : the primitive cannot
close, the primitive crashes on render, or the primitive renders in the
wrong place with the wrong focus. This skill documents the contract and
the six recurring traps that violate it.

Companion skills :

- `shadcn-syntax-dialog` (Batch B4) : Dialog composition surface and
  Radix wrap layers
- `shadcn-syntax-popover-tooltip-hovercard` (Batch B6) : Popover /
  Tooltip / HoverCard differences and `modal` prop semantics
- `shadcn-syntax-drawer` (Batch B5) : Drawer (Vaul) controlled state
- `shadcn-syntax-sheet` (Batch B5) : Sheet (Radix Dialog variant)
- `shadcn-syntax-menu-primitives` (Batch B6) : DropdownMenu /
  ContextMenu / Menubar shared `onOpenChange`
- `shadcn-syntax-selectors` (Batch B6) : Select / Combobox
  `value`+`onValueChange` (a parallel pair to `open`+`onOpenChange`)
- `shadcn-errors-form-state` : controlled form state interacting with
  controlled dialog state (close-on-success pattern)

## Quick Reference : The Open + onOpenChange Pair Rule (ALWAYS)

For EVERY Radix primitive with an `open` prop, the rule is binary :

1. ALWAYS pass BOTH `open` and `onOpenChange` together (controlled).
2. OR pass NEITHER and use `defaultOpen` if a non-default initial
   state is needed (uncontrolled).
3. NEVER pass `open` alone. The primitive becomes read-only : it
   reflects the prop but ignores Escape, the X button, the overlay
   click, and the explicit `DialogClose` element. The user cannot
   close it.

The same rule applies under different prop names :

| Primitive | Pair |
|-----------|------|
| Dialog, AlertDialog, Sheet, Drawer, Popover, HoverCard, DropdownMenu, ContextMenu, Menubar, Collapsible, Tooltip | `open` + `onOpenChange` |
| Select, Combobox | `value` + `onValueChange` |
| Accordion (single) | `value` + `onValueChange` |
| Accordion (multiple), ToggleGroup | `value` (string[]) + `onValueChange` |
| Tabs | `value` + `onValueChange` |
| RadioGroup, Switch, Checkbox | `checked` / `value` + `onCheckedChange` / `onValueChange` |
| Slider | `value` (number[]) + `onValueChange` |

For the `open` family the `defaultOpen` escape hatch is `defaultOpen={true}`.
For the `value` family the equivalent is `defaultValue="..."`.

Verified at https://www.radix-ui.com/primitives/docs/components/dialog
(2026-05-19) : `Root` accepts `open` (boolean), `onOpenChange` (function),
`defaultOpen` (boolean), and `modal` (boolean, defaults to `true`).

## Quick Reference : Why open-alone Makes The Primitive Read-Only

Radix internals : when `open` is provided, the primitive switches into
"controlled mode". In controlled mode, internal close handlers (Escape,
overlay click, `DialogClose` click, X button, focus-restoration) do
NOT mutate state directly. They call `onOpenChange(false)`. The PARENT
is then responsible for setting `open={false}`.

If `onOpenChange` is missing, those internal handlers fire into the
void. The state stays at `open={true}` forever. The dialog is visually
locked open until the page reloads.

The fix is ALWAYS one of :

```tsx
// CORRECT : controlled
const [open, setOpen] = React.useState(false)
<Dialog open={open} onOpenChange={setOpen}>...</Dialog>

// CORRECT : uncontrolled (let Radix manage state)
<Dialog defaultOpen>...</Dialog>
<Dialog>...</Dialog>

// WRONG : read-only, never closes
<Dialog open={true}>...</Dialog>
```

The exact same logic applies to `value` controlled primitives : `value`
without `onValueChange` makes a `Select` or `Tabs` read-only at the
current value.

## Quick Reference : Per-Primitive Controlled Contract

| Primitive | State pair | Default `modal` | Notes |
|-----------|-----------|-----------------|-------|
| Dialog | `open` / `onOpenChange` | `true` | Focus trap + scroll lock + outside-click closes |
| AlertDialog | `open` / `onOpenChange` | `true` (not configurable) | NO outside-click close ; only `Action` / `Cancel` |
| Sheet | `open` / `onOpenChange` | `true` | Dialog under the hood |
| Drawer (Vaul) | `open` / `onOpenChange` | n/a | Snap points, drag-to-close ; not Radix Dialog |
| Popover | `open` / `onOpenChange` | `false` | Non-modal default ; can opt in to `modal={true}` |
| HoverCard | `open` / `onOpenChange` | n/a (always non-modal) | Triggered by hover and focus, NOT click |
| DropdownMenu, ContextMenu, Menubar | `open` / `onOpenChange` | n/a | Auto outside-click close, focus returns to trigger |
| Select | `value` / `onValueChange` AND `open` / `onOpenChange` | n/a | Two separate pairs ; rarely need to control `open` |
| Collapsible | `open` / `onOpenChange` | n/a | No portal, no focus trap |
| Accordion | `value` / `onValueChange` | n/a | `type="single"` or `type="multiple"` |
| Tabs | `value` / `onValueChange` | n/a | Tab list navigation managed internally |
| Tooltip | `open` / `onOpenChange` + `TooltipProvider` `delayDuration` | n/a | Provider-driven delays ; controlled `open` is rare |

`modal={true}` (Dialog default) ALWAYS implies : focus trap, scroll
lock on `<body>`, click-outside-overlay closes, Escape closes. The
trap is implemented via `aria-hidden` on sibling DOM and a focus
sentinel on the content boundary.

`modal={false}` (Popover default) means : NO focus trap, NO scroll
lock, outside click still closes via `onPointerDownOutside`, focus is
NOT restored to the trigger on close unless explicitly handled.

## Quick Reference : asChild Slot Contract

`asChild={true}` on a Radix trigger (DialogTrigger, PopoverTrigger,
DropdownMenuTrigger, etc.) tells the trigger to render its child
INSTEAD of the default `<button>`, merging all the trigger's props
(onClick, aria-expanded, data-state, ref) onto that child.

The contract is strict :

1. EXACTLY ONE child. The Radix Slot uses `React.Children.only` and
   throws "React.Children.only expected to receive a single child"
   if there are zero, two, or more children.
2. The child MUST accept and forward a `ref`. Native DOM elements
   (`<button>`, `<a>`, `<div>`, `<input>`) all do. Custom components
   without `React.forwardRef` do NOT and will silently break focus,
   keyboard navigation, and `data-state` styling.
3. The child SHOULD accept the props the Slot forwards. `<div>` accepts
   `onClick` but ignores `aria-expanded`, `aria-haspopup`, and
   `data-state`. The trigger then has no a11y semantics. ALWAYS use a
   focusable element (`<button>`, `<a>` with href, custom button
   component with forwardRef) as the asChild target.
4. If the child is a custom component, it MUST spread the props onto
   the underlying DOM element. `function MyButton(props) { return
   <button {...props} /> }` works ; `function MyButton({ onClick })
   { return <button onClick={onClick} /> }` swallows `data-state`.

Example of the most common WRONG pattern :

```tsx
// WRONG : Link from Next.js does forwardRef, but the icon is a SECOND child.
// Slot throws "React.Children.only".
<DialogTrigger asChild>
  <Link href="/edit"><Icon /> Edit</Link>
</DialogTrigger>
```

```tsx
// CORRECT : wrap children inside the single asChild target.
<DialogTrigger asChild>
  <Link href="/edit">
    <span><Icon /> Edit</span>
  </Link>
</DialogTrigger>
```

The Slot pattern is documented at
https://www.radix-ui.com/primitives/docs/utilities/slot (verified
2026-05-19). The single-child + forwardRef contract is enforced at
render time, not at type-check time.

## Quick Reference : Portal and Stacking-Context Traps

Every overlay primitive (Dialog, Sheet, AlertDialog, Popover,
DropdownMenu, HoverCard, ContextMenu, Tooltip, Select) renders its
content inside a Radix `Portal` that defaults to `document.body`.
This bypasses any local stacking context EXCEPT when a CSS property
on an ancestor of the `<body>`-attached portal would still affect
positioning anchors.

The five CSS properties that break Radix popper positioning :

1. `transform: anything-non-none` on any ancestor (creates a
   containing block for positioned descendants in CSS, but Radix
   floating-ui anchors are computed against `getBoundingClientRect()`
   which IS still correct ; the symptom is usually z-index, not
   coordinates).
2. `filter` on any ancestor (creates a containing block).
3. `perspective` on any ancestor (creates a containing block).
4. `will-change: transform` (creates a containing block lazily).
5. `contain: paint` or `contain: layout` (creates a containing block).

When any of these are present on an ancestor of the trigger AND the
portal lands at `document.body`, the portal content stacks against
the global stacking context but the trigger lives inside a NEW local
stacking context. The portal content can render BEHIND the trigger's
ancestor if z-index ordering is wrong.

The fix : use the Portal `container` prop to attach the portal to a
specific element you control :

```tsx
<DialogPortal container={containerRef.current}>
  <DialogOverlay />
  <DialogContent>...</DialogContent>
</DialogPortal>
```

Or set the trigger's transformed ancestor to `position: relative` +
`z-index: auto` so it does not steal stacking.

A z-index above `1000` on the Radix overlay (default in shadcn's
`dialog.tsx`) usually wins. If it does not, the ancestor's stacking
context is the problem, not the z-index value.

See `references/anti-patterns.md` for the full trap matrix.

## Quick Reference : Focus Management Contract

For modal primitives (Dialog, AlertDialog, Sheet, Drawer,
DropdownMenu, ContextMenu, Menubar, Select-open, Popover with
`modal={true}`) :

1. ON OPEN : focus moves to the first focusable element inside the
   content. Override via `onOpenAutoFocus={(e) => { e.preventDefault();
   myInputRef.current?.focus() }}`.
2. WHILE OPEN : Tab and Shift+Tab cycle within the content. Focus
   cannot escape (modal trap).
3. ON ESCAPE : `onEscapeKeyDown` fires, then `onOpenChange(false)`
   fires. Calling `e.preventDefault()` in `onEscapeKeyDown` cancels
   the close.
4. ON OUTSIDE POINTER : `onPointerDownOutside` fires, then
   `onOpenChange(false)`. Calling `e.preventDefault()` cancels.
5. ON CLOSE : focus returns to the trigger element (or the element
   that had focus before open). Override via `onCloseAutoFocus={(e)
   => { e.preventDefault(); myReturnRef.current?.focus() }}`.

For non-modal Popover (`modal={false}`, the default) : focus does NOT
trap, outside click still closes, focus is NOT auto-restored on close.

Verified at
https://www.radix-ui.com/primitives/docs/components/dialog#content
(2026-05-19) : the `Content` exposes `onOpenAutoFocus`,
`onCloseAutoFocus`, `onEscapeKeyDown`, `onPointerDownOutside`, and
`onInteractOutside`.

## Quick Reference : Close-On-Success Controlled Pattern

A controlled dialog tied to a form submit MUST set `open={false}`
inside the success branch. The mutation handler is the source of
truth, not the form's `onSubmit`.

```tsx
const [open, setOpen] = React.useState(false)

const mutation = useMutation({
  mutationFn: api.saveUser,
  onSuccess: () => {
    setOpen(false)
    toast.success("Saved.")
  },
})

return (
  <Dialog open={open} onOpenChange={setOpen}>
    <DialogTrigger asChild><Button>Edit</Button></DialogTrigger>
    <DialogContent>
      <form onSubmit={(e) => { e.preventDefault(); mutation.mutate(data) }}>
        ...
        <Button type="submit" disabled={mutation.isPending}>Save</Button>
      </form>
    </DialogContent>
  </Dialog>
)
```

ALWAYS call `setOpen(false)` in `onSuccess`. NEVER call it in the
submit handler's synchronous body : the mutation is async and the
dialog will close before the request resolves, losing the
`isPending` UI affordance.

For Popover-with-form-inside, use `modal={true}` to trap focus AND
prevent the form's first interactive element from being clicked
outside the popover by accident. See
`references/examples.md` for the full pattern.

## Decision Tree : Why Will My Primitive Not Close?

```
Did you pass `open` to the root?
├── No  -> Uncontrolled. Radix handles close internally.
│         If still stuck : check for `e.preventDefault()` in
│         `onEscapeKeyDown` / `onPointerDownOutside` / `onInteractOutside`.
└── Yes -> Did you ALSO pass `onOpenChange`?
        ├── No  -> ROOT CAUSE : half-controlled state. Dialog is
        │          read-only. Add `onOpenChange={setOpen}` (or remove
        │          the `open` prop entirely).
        └── Yes -> Does `onOpenChange` call `setOpen(value)`?
                ├── No (it does nothing) -> Same as half-controlled.
                │   The state pair is broken at the consumer side.
                └── Yes -> Is something calling `e.preventDefault()` in
                           `onEscapeKeyDown` / `onPointerDownOutside`?
                           OR is `open` derived from a memoized value
                           that never re-renders? OR is the mutation
                           callback never firing `setOpen(false)`?
                           Trace the `onOpenChange` call with a console
                           log : if it fires with `false` and the dialog
                           stays open, the consumer state setter is
                           broken.
```

## Patterns : Per-Primitive Quick Checks

- Dialog / AlertDialog / Sheet : if Escape and overlay-click do not
  close, you have `open` without `onOpenChange`.
- Drawer (Vaul) : same pair rule ; additionally `dismissible={false}`
  blocks drag-to-close even when state is correct.
- Popover : default is non-modal. If you put a form inside, set
  `modal={true}` to prevent focus-loss on first click.
- DropdownMenu / ContextMenu / Menubar : `onOpenChange` fires on
  outside click, Escape, and item-select with `onSelect`. If items
  do not close the menu on click, check for
  `e.preventDefault()` in the item's `onSelect`.
- Tooltip : do NOT control `open` unless you need programmatic
  show-on-error patterns. Use `TooltipProvider` `delayDuration` and
  `skipDelayDuration` instead.
- Select : `value` + `onValueChange` is the controlled pair (not
  `open`). The popper `open` state is internal and rarely needs
  control.

## Patterns : The `data-state` Hook For Debugging

Every Radix overlay primitive sets `data-state="open"` or
`data-state="closed"` on its trigger and content. Inspect the DOM :

```html
<button data-state="open" aria-expanded="true" data-slot="dialog-trigger">...</button>
```

If `data-state="open"` but the dialog is invisible, the issue is
CSS / z-index / portal stacking, not state. If `data-state="closed"`
but the dialog is visible, you have stale styles (a closing-animation
that never finishes) or a stuck `forceMount` Portal.

## Reference Links

- Companion : `shadcn-syntax-dialog` (Dialog composition surface)
- Companion : `shadcn-syntax-popover-tooltip-hovercard` (modal prop deep dive)
- Companion : `shadcn-syntax-drawer` (Vaul drawer controlled state)
- Companion : `shadcn-syntax-sheet` (Sheet as Dialog variant)
- Companion : `shadcn-syntax-menu-primitives` (DropdownMenu / ContextMenu)
- Companion : `shadcn-errors-form-state` (form + controlled dialog)
- Detailed signatures : `references/methods.md`
- Worked examples : `references/examples.md`
- Anti-patterns to avoid : `references/anti-patterns.md`
- Radix Dialog docs : https://www.radix-ui.com/primitives/docs/components/dialog
- Radix Popover docs : https://www.radix-ui.com/primitives/docs/components/popover
- Radix Slot utility : https://www.radix-ui.com/primitives/docs/utilities/slot
- shadcn Dialog reference : https://ui.shadcn.com/docs/components/dialog
- shadcn Popover reference : https://ui.shadcn.com/docs/components/popover
