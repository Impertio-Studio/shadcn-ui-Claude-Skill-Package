# Methods Reference: Responsive Dialog + Drawer

All signatures below are TypeScript. Files are owned by the consumer project (shadcn ownership model). Names match the official shadcn ui `drawer-dialog` example unless explicitly noted.

## useMediaQuery (hook signature)

Location: `src/hooks/use-media-query.ts`.

```ts
function useMediaQuery(query: string): boolean
```

- Parameter `query`: a CSS media query string. The canonical value for this skill is `"(min-width: 768px)"`, matching the official shadcn ui example and Tailwind's `md:` breakpoint.
- Return: `true` when the query matches the current viewport, `false` otherwise.
- Implementation MUST use `React.useSyncExternalStore` (React 18+). The `useState` + `useEffect` variant is disallowed in this skill because it causes a Next.js App Router hydration mismatch on first paint.
- The hook is client-only. Callers MUST place it inside a `"use client"` component. On the server, `useSyncExternalStore` falls back to `getServerSnapshot`, which throws by design.
- The query string is the `useCallback` dependency. Passing a literal string is safe; passing a dynamically-built string requires `useMemo` upstream to avoid re-subscribing every render.

## ResponsiveModal (component signature)

Location: `src/components/responsive-modal.tsx`.

```ts
type ResponsiveModalProps = {
  open: boolean
  onOpenChange: (open: boolean) => void
  title: string
  description?: string
  trigger?: React.ReactNode
  children: React.ReactNode
  breakpoint?: string
}

export function ResponsiveModal(props: ResponsiveModalProps): JSX.Element
```

Contract:
- `open` + `onOpenChange` are forwarded to BOTH `Dialog` and `Drawer`. Callers MUST own the state via `useState`. Uncontrolled mode (`defaultOpen`) is NOT exposed because controlled state is required to close the surface from inside `children` (e.g. after a successful submit).
- `title` becomes `DialogTitle` on desktop and `DrawerTitle` on mobile. REQUIRED for screen-reader compliance. Empty string is allowed by Radix Dialog only if `aria-labelledby` is overridden upstream; this skill assumes the simpler labelled path and requires a non-empty title.
- `description` becomes `DialogDescription` / `DrawerDescription`. Optional but RECOMMENDED (Radix Dialog warns when absent).
- `trigger` is rendered inside `DialogTrigger asChild` / `DrawerTrigger asChild`. Omit it to open the modal imperatively via `onOpenChange(true)` from elsewhere.
- `children` renders inside `DialogContent` (no automatic padding wrapper) and inside `DrawerContent` below `DrawerHeader` (no automatic padding wrapper). Callers MUST handle their own padding, typically with the pattern `className="px-4 md:px-0"` on the inner element, because `DialogContent` already pads horizontally but `DrawerContent` does not.
- `breakpoint` defaults to `"(min-width: 768px)"`. Override only when the design system mandates a different switch point.

## Shared content component contract

The component rendered inside `children` MUST satisfy three rules:

1. Accept an OPTIONAL `className` prop and merge it via `cn(...)`. The Drawer branch passes `className="px-4"` to compensate for `DrawerContent`'s missing horizontal padding; the Dialog branch passes nothing.
2. Render IDENTICAL field sets, identical labels, and identical validation messages on both branches. NEVER show fewer fields on one branch.
3. Receive callbacks (`onDone`, `onCancel`, `onSubmit`) as props so the orchestrator can close the modal via `setOpen(false)` after a successful action. The component itself MUST NOT call `setOpen` directly; it does not know which surface is active.

Recommended TypeScript shape:

```ts
type SharedContentProps = {
  className?: string
  onDone?: () => void
}
```

The official example's `ProfileForm` uses `React.ComponentProps<"form">` so the form element passes through `id`, `name`, `onSubmit` etc. without additional plumbing.

## Open-state contract

ONE `useState` pair for the surface, shared by Dialog and Drawer:

```ts
const [open, setOpen] = React.useState(false)
```

ANTI-CONTRACT (do NOT do this):

```ts
const [dialogOpen, setDialogOpen] = React.useState(false)
const [drawerOpen, setDrawerOpen] = React.useState(false)
```

The two-state variant breaks when the viewport crosses the breakpoint while the modal is open (rotate device, drag window): the now-active branch reads `false` because only the other branch's flag was updated, so the user's open modal silently disappears.

## Breakpoint constant (recommended)

When `ResponsiveModal` is used in more than three places, extract the breakpoint to a single constant so a design-system change requires one edit:

```ts
// src/lib/breakpoints.ts
export const MEDIA_DESKTOP = "(min-width: 768px)"
```

The orchestrator then reads `useMediaQuery(MEDIA_DESKTOP)` and accepts an override via the `breakpoint` prop for the rare exception.

## DrawerClose vs DialogClose

Both close the modal when clicked. Both REQUIRE the `asChild` pattern when wrapping a `Button`:

```tsx
<DrawerClose asChild>
  <Button variant="outline">Cancel</Button>
</DrawerClose>
```

Without `asChild`, the close primitive renders its own button and nests it inside the consumer's button, producing invalid HTML (`<button>` inside `<button>`).

`DialogContent` ALSO ships a built-in top-right close affordance, controlled by the `showCloseButton` prop (defaults to `true`). `DrawerContent` does NOT. This is why the canonical pattern adds a `DrawerFooter` with a `DrawerClose` Cancel button on the mobile branch but not on the desktop branch.

## Responsive Combobox variant signature

The same pattern applies to the Combobox composition (Popover + Command). The desktop branch renders `<Popover>`; the mobile branch renders `<Drawer>` with the same `<Command>` body inside.

```ts
function ResponsiveCombobox<TOption>(props: {
  value: TOption | null
  onValueChange: (next: TOption | null) => void
  options: TOption[]
  getLabel: (opt: TOption) => string
  placeholder?: string
}): JSX.Element
```

The implementation lives in `examples.md`. The decision to switch to a Drawer on mobile is driven by the same UX rule: a Popover anchored to a trigger button is unusable on a phone because the on-screen keyboard covers it and the list cannot scroll into view.
