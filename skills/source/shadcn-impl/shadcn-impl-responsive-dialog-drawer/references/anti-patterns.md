# Anti-patterns: Responsive Dialog + Drawer

Seven recurring failure modes that this skill prevents. Each entry pairs the wrong pattern with the correct pattern.

## 1. Rendering BOTH Dialog AND Drawer simultaneously

The single most common mistake: avoiding the `useMediaQuery` hook by mounting both surfaces and toggling visibility via Tailwind responsive utilities.

### Wrong

```tsx
export function EditProfile() {
  const [open, setOpen] = React.useState(false)
  return (
    <>
      <div className="hidden md:block">
        <Dialog open={open} onOpenChange={setOpen}>
          <DialogContent>
            <DialogTitle>Edit profile</DialogTitle>
            <ProfileForm />
          </DialogContent>
        </Dialog>
      </div>
      <div className="md:hidden">
        <Drawer open={open} onOpenChange={setOpen}>
          <DrawerContent>
            <DrawerTitle>Edit profile</DrawerTitle>
            <ProfileForm />
          </DrawerContent>
        </Drawer>
      </div>
    </>
  )
}
```

Why this fails:
- Both Radix Dialog (via `@radix-ui/react-dialog`) and Vaul (the Drawer engine) mount portal nodes with `role="dialog"` and `aria-modal="true"`. The accessibility tree now contains TWO concurrent modal dialogs; screen readers announce both, NVDA in particular reads the second one first.
- Both surfaces try to install a focus trap on `open=true`. Focus ping-pongs between the two trapped regions.
- `ProfileForm` is mounted TWICE. Each instance has its own internal state, so typing in the visually-hidden form silently fills a phantom copy. When the viewport resizes mid-edit, the user sees the other (empty) instance.
- The `hidden` Tailwind utility uses `display: none`, which Radix and Vaul cannot detect; they still process keyboard events globally. Pressing `Escape` closes both `open` flags but only one was visually active.

### Right

Single source, early return:

```tsx
const isDesktop = useMediaQuery("(min-width: 768px)")
if (isDesktop) {
  return <Dialog ...>...</Dialog>
}
return <Drawer ...>...</Drawer>
```

Only ONE surface mounts. The content component instantiates exactly once per resize that crosses the breakpoint.

## 2. `useMediaQuery` without an SSR guard (Next.js hydration mismatch)

A homegrown hook based on `useState` + `useEffect` "feels" simpler but breaks Next.js App Router.

### Wrong

```tsx
export function useMediaQuery(query: string) {
  const [matches, setMatches] = React.useState(
    typeof window !== "undefined" && window.matchMedia(query).matches
  )
  React.useEffect(() => {
    const mql = window.matchMedia(query)
    setMatches(mql.matches)
    const onChange = () => setMatches(mql.matches)
    mql.addEventListener("change", onChange)
    return () => mql.removeEventListener("change", onChange)
  }, [query])
  return matches
}
```

Why this fails:
- Server render: `typeof window !== "undefined"` is `false`, initial state is `false`, the server returns the Drawer branch HTML.
- Client first paint: still uses `false` because React reads the initial state. Returns Drawer.
- `useEffect` runs AFTER first paint: state flips to `true` on a desktop browser, React re-renders, the Drawer unmounts and the Dialog mounts.
- Result: visible flicker, console warning `Warning: Expected server HTML to contain a matching <button> in <body>` or similar hydration mismatches when the trigger button has different children per branch.

### Right

Use `useSyncExternalStore` with a `getServerSnapshot` that throws (see `examples.md` Example 1). The hook is then explicitly client-only; combined with `"use client"` on the consumer component, the server never tries to render either branch and the first client paint reads the real `matchMedia` value synchronously.

If the user really cannot tolerate any flash (they have a server-rendered shell), set the trigger button to be identical across both branches and place the branching INSIDE the modal content only.

## 3. Separate `useState` for Dialog and Drawer

Looks defensive but introduces a viewport-resize bug.

### Wrong

```tsx
const [dialogOpen, setDialogOpen] = React.useState(false)
const [drawerOpen, setDrawerOpen] = React.useState(false)
const isDesktop = useMediaQuery("(min-width: 768px)")

if (isDesktop) {
  return <Dialog open={dialogOpen} onOpenChange={setDialogOpen}>...</Dialog>
}
return <Drawer open={drawerOpen} onOpenChange={setDrawerOpen}>...</Drawer>
```

Why this fails:
- The user opens the Drawer on a phone: `drawerOpen=true`, `dialogOpen=false`.
- They rotate the device to landscape OR resize the browser past 768px (DevTools, foldable phones, Samsung DeX, iPadOS Stage Manager).
- `isDesktop` flips to `true`, the if-branch returns `<Dialog open={dialogOpen=false}>`, which is closed.
- The modal silently disappears mid-edit. Form state inside the (now unmounted) Drawer is lost.

### Right

ONE state pair, shared:

```tsx
const [open, setOpen] = React.useState(false)
```

Both branches use the same `open` and `setOpen`. The modal stays open across viewport changes. Form state in the shared content component persists because the content component is the SAME component instance whether it lives inside Dialog or Drawer (assuming both branches pass the same children identity).

CAVEAT: even with one state, React unmounts/remounts the content component when the parent surface changes (Dialog -> Drawer). To preserve form state across resize, lift the form state up (to a parent `useState` or react-hook-form `useForm` instance OUTSIDE the modal), and pass `defaultValues` or controlled props down.

## 4. Hard-coded breakpoint inline instead of `useMediaQuery` hook

Every component re-implements the same breakpoint check, each slightly different.

### Wrong

```tsx
// In EditProfile.tsx
const [isDesktop, setIsDesktop] = React.useState(window.innerWidth > 768)

// In DeleteAccount.tsx
const [isDesktop, setIsDesktop] = React.useState(window.innerWidth >= 768)

// In ShareSheet.tsx
const isDesktop = typeof window !== "undefined" && window.matchMedia("(min-width: 640px)").matches
```

Why this fails:
- The first variant uses `>`, the second `>=`, so at exactly 768px viewport one component shows Dialog and the other shows Drawer. The user sees inconsistent modal styles on the same page.
- The third variant uses 640px (Tailwind `sm:` boundary) so a 700px tablet gets a Dialog from one feature and a Drawer from another.
- None of the variants react to live viewport changes. Resize the window, the modals do not switch.
- `window.innerWidth` is `undefined` during SSR, the first two variants throw "window is not defined" the moment a server render runs.

### Right

One hook, one constant, one default:

```ts
// src/lib/breakpoints.ts
export const MEDIA_DESKTOP = "(min-width: 768px)"

// In every consumer
const isDesktop = useMediaQuery(MEDIA_DESKTOP)
```

Changing the breakpoint across the app is then a one-line edit. The hook handles SSR safety and resize listeners.

## 5. Different content per breakpoint (UX inconsistency)

Tempting to "optimize for mobile" by trimming fields, but this introduces silent data loss.

### Wrong

```tsx
if (isDesktop) {
  return (
    <Dialog ...>
      <DialogContent>
        <ProfileFormFull /> {/* shows email, username, bio, avatar upload */}
      </DialogContent>
    </Dialog>
  )
}
return (
  <Drawer ...>
    <DrawerContent>
      <ProfileFormCompact /> {/* shows only email + username */}
    </DrawerContent>
  </Drawer>
)
```

Why this fails:
- A user opens the Drawer on a phone, fills email + username, taps Save. The save handler submits ONLY those two fields. The user's existing bio and avatar are now sent as `undefined` or empty strings, depending on serialization, and the backend overwrites them.
- Conversely, a user opens the Dialog on desktop, fills everything, rotates an iPad mini below 768px: the Dialog unmounts, the Drawer remounts with `ProfileFormCompact`, the previously typed bio is gone with no warning.
- Accessibility audits flag this as a content-disparity violation (WCAG 1.4.4 requires content not to be hidden by viewport).

### Right

ONE content component for BOTH branches. If the form is genuinely too long for a phone, reduce the form length for all screens or split it into a multi-step wizard. The mobile form is the desktop form; the surface changes, the content does not.

## 6. Missing `DialogTitle` / `DrawerTitle` (screen-reader violation)

Skipping the title because "the description already says what this is".

### Wrong

```tsx
<DialogContent>
  <p>Make changes to your profile here.</p>
  <ProfileForm />
</DialogContent>
```

Why this fails:
- Radix Dialog logs: `Warning: Missing 'DialogTitle' for the 'DialogContent'. Add a 'DialogTitle' to make the content accessible to screen readers.` Vaul logs an equivalent warning for Drawer.
- Screen readers announce the modal as "dialog" with no name. The user has no idea what just opened.
- Some audit tools (axe-core, Lighthouse a11y score) fail the page with `aria-dialog-name`.

### Right

ALWAYS include both Title and Description, on BOTH branches:

```tsx
<DialogHeader>
  <DialogTitle>Edit profile</DialogTitle>
  <DialogDescription>Make changes to your profile here.</DialogDescription>
</DialogHeader>
```

If the design genuinely requires a hidden title, wrap it in shadcn's `VisuallyHidden` slot rather than omitting it.

## 7. `DrawerClose` wrapping a `Button` WITHOUT `asChild`

The Drawer Cancel button does not actually close the Drawer, or it renders nested `<button>` elements.

### Wrong

```tsx
<DrawerFooter>
  <DrawerClose>
    <Button variant="outline">Cancel</Button>
  </DrawerClose>
</DrawerFooter>
```

Why this fails:
- Without `asChild`, `DrawerClose` renders ITS OWN button and places `<Button>Cancel</Button>` as a child. The resulting DOM is `<button><button>Cancel</button></button>`, which is invalid HTML. Browsers handle it inconsistently: clicking the inner button may or may not bubble to the outer one.
- React DevTools shows the duplicated focus ring; keyboard tab order hits both buttons.
- Form-wide `type="submit"` semantics get confused because nested buttons each default to `type="submit"`.

### Right

```tsx
<DrawerFooter>
  <DrawerClose asChild>
    <Button variant="outline">Cancel</Button>
  </DrawerClose>
</DrawerFooter>
```

`asChild` uses Radix's `Slot` pattern: `DrawerClose` merges its own props (the close handler, the `data-state` attribute) onto the child `Button` instead of rendering a wrapper. The result is ONE `<button>` element with the correct semantics.

The same rule applies to `DrawerTrigger`, `DialogTrigger`, `DialogClose`, `PopoverTrigger`, and any other Radix `*Trigger` / `*Close` primitive wrapping a real interactive element.
