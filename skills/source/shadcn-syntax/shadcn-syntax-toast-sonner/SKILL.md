---
name: shadcn-syntax-toast-sonner
description: >
  Use when adding, debugging, or migrating notification toasts in a
  shadcn ui project, mounting the top-level `<Toaster />` in your app
  root, calling `toast()` / `toast.success` / `toast.error` /
  `toast.warning` / `toast.info` / `toast.loading` / `toast.promise`,
  configuring the Toaster's `position`, `duration`, `expand`,
  `richColors`, or `closeButton` props, dismissing a toast
  programmatically via `toast.dismiss(id)`, wrapping a fetch /
  mutation with `toast.promise` for the loading -> success / error
  state machine, or removing dead `useToast()` hook calls left over
  from a pre-2026 shadcn project.
  Prevents the canonical toast failures : trying to import
  `useToast` from `@/components/ui/use-toast` (the file no longer
  exists ; the hook + the Radix-based Toast component were REMOVED
  per the shadcn 2026 changelog), calling `toast()` before any
  `<Toaster />` is mounted (the call is silently dropped), mounting
  two `<Toaster />` instances and getting duplicated stacks,
  calling `toast()` from inside a React Server Component (Sonner is
  client-only), forgetting `"use client"` on the component that
  calls `toast()`, and using `toast.promise` with only two of the
  three required states (loading + success without error, or
  success without loading) so the toast hangs or never settles.
  Covers the shadcn `Toaster` wrapper verbatim from `apps/v4/
  registry/new-york-v4/ui/sonner.tsx` (icons, theme integration via
  next-themes, the CSS-variable bridge to shadcn's `--popover` /
  `--border` / `--radius` tokens), the upstream `sonner` package
  API (`toast`, `toast.success`, `toast.error`, `toast.warning`,
  `toast.info`, `toast.loading`, `toast.promise`, `toast.dismiss`,
  `toast.custom`), the six `position` values (top-left, top-center,
  top-right, bottom-left, bottom-center, bottom-right ; default
  bottom-right), the `duration` prop (default 4000ms, `Infinity`
  for sticky), `expand` (default true), `richColors` (semantic
  per-type colors), `closeButton` (X affordance on hover), the
  one-toaster-per-app rule, and the migration table from
  `useToast()` -> `toast()`.
  Keywords: shadcn toast, sonner, Toaster, useToast removed,
  useToast deprecated, where is useToast, migrate from useToast,
  toast.success, toast.error, toast.warning, toast.info,
  toast.loading, toast.promise, toast.dismiss, toast.custom,
  richColors, closeButton, position prop, duration prop, expand
  prop, top-right toast, bottom-right toast, sticky toast,
  Infinity duration, sonner not working, toast not showing,
  toast silent, toast nothing happens, no toast appears, Toaster
  not mounted, two toasters, duplicate toasts, toast in server
  component, RSC toast, "use client" toast, how do I show a
  notification, how do I show an error message, how do I notify
  the user, snackbar shadcn, notification shadcn, alert popup
  shadcn, fetch with toast, toast around fetch, loading toast
  pattern, optimistic ui toast.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026. The legacy Radix-based `Toast` component and the `useToast` hook were REMOVED per the 2026 changelog ; Sonner is the sole toast primitive."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui : Toast (Sonner) Syntax

Sonner is the sole notification-toast primitive in shadcn ui as of 2026. The previous Radix-based `Toast` component and its companion `useToast()` hook were REMOVED per the shadcn changelog. The shadcn registry now ships a single `Toaster` wrapper around the upstream `sonner` package (see `apps/v4/registry/new-york-v4/ui/sonner.tsx`), and all toast calls go through the imperative `toast()` API imported directly from `sonner`.

## REMOVAL NOTICE : useToast is gone

If your project has any of the following, it is dead code from a pre-2026 shadcn project and MUST be removed :

- `import { useToast } from "@/components/ui/use-toast"` : the file `components/ui/use-toast.ts` no longer exists in the registry. Delete the import.
- `import { Toast, ToastProvider, ToastViewport } from "@/components/ui/toast"` : the file `components/ui/toast.tsx` (Radix-based) no longer exists. Delete the import.
- `const { toast } = useToast()` : the hook is gone. Replace with `import { toast } from "sonner"` (a top-level import, NOT a hook).
- `<Toaster />` from `@/components/ui/toaster` : the old file at `components/ui/toaster.tsx` (the Radix viewport) no longer exists. Replace with `<Toaster />` from `@/components/ui/sonner`.

See the migration table at the bottom of this skill for line-by-line replacement.

## Quick Reference

### The two surfaces

| Surface | Purpose |
|---------|---------|
| `<Toaster />` (mounted once, in the app root) | Renders the toast viewport ; without it, every `toast()` call is silently dropped |
| `toast(...)` (imported from `sonner`) | Imperative API ; call from anywhere in a client component to enqueue a toast |

### Minimal setup

**`app/layout.tsx`** (Next.js App Router, RSC-friendly) :

```tsx
import { Toaster } from "@/components/ui/sonner"

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        {children}
        <Toaster />
      </body>
    </html>
  )
}
```

`<Toaster />` from `@/components/ui/sonner` is itself a client component (the shadcn wrapper carries `"use client"`), so it is safe to mount inside a Server Component layout.

**Any client component that needs to fire a toast** :

```tsx
"use client"

import { toast } from "sonner"
import { Button } from "@/components/ui/button"

export function CopyLink() {
  return (
    <Button
      onClick={() => {
        toast.success("Link copied to clipboard")
      }}
    >
      Copy link
    </Button>
  )
}
```

ALWAYS keep `"use client"` at the top of any component that calls `toast()`. `toast()` reaches into a client-side store, so calling it from a RSC throws `Error : toast can not be used in a server component`.

### Five invariants

1. ALWAYS mount exactly ONE `<Toaster />` per app, typically in the root layout. NEVER mount it twice ; the two instances each render their own queue and you see every toast doubled.
2. ALWAYS import `toast` directly : `import { toast } from "sonner"`. NEVER import it from `"@/components/ui/sonner"` (only `Toaster` is re-exported there) and NEVER expect a `useToast()` hook (it was removed).
3. ALWAYS keep `"use client"` on any component that calls `toast()`. NEVER call `toast()` inside a Server Component, Server Action, or RSC render path ; the call has no client store to reach.
4. ALWAYS pass all three states to `toast.promise` : `loading`, `success`, `error`. NEVER omit `error` ; an unhandled rejection leaves the loading toast spinning forever.
5. ALWAYS treat `toast()` as fire-and-forget ; it returns a numeric id you can pass to `toast.dismiss(id)`. NEVER `await` `toast()` ; it is synchronous and has no promise contract.

## The toast() API surface

```ts
toast(message: string | ReactNode, options?: ExternalToast): number | string
toast.success(message, options?)
toast.error(message, options?)
toast.warning(message, options?)
toast.info(message, options?)
toast.loading(message, options?)
toast.message(message, options?)            // alias for default neutral toast
toast.promise(promise, { loading, success, error })
toast.custom((id) => <YourNode />, options?)
toast.dismiss(id?)                            // omit id to dismiss all
```

`options` is an `ExternalToast` object. The most common fields :

| Field | Type | Use |
|-------|------|-----|
| `id` | `number \| string` | Stable id ; if a toast with this id already exists it is UPDATED in place instead of appended (useful for progressive updates) |
| `description` | `string \| ReactNode` | Secondary line under the main message |
| `duration` | `number` | Override the Toaster's default ; pass `Infinity` for a sticky toast that only the user can dismiss |
| `action` | `{ label, onClick }` | Renders a button on the right edge ; `onClick` receives the click event |
| `cancel` | `{ label, onClick }` | Secondary action ; renders to the left of `action` |
| `onDismiss` | `(t) => void` | Fired when the user dismisses |
| `onAutoClose` | `(t) => void` | Fired when duration elapses |
| `closeButton` | `boolean` | Force an X button on this toast only ; overrides the Toaster default |
| `dismissible` | `boolean` | When `false`, user cannot swipe / click it away (still respects `duration`) |
| `icon` | `ReactNode` | Override the per-type icon |
| `className` / `style` | string / object | Per-toast styling override |
| `position` | one of 6 anchors | Override the Toaster default for this toast |

Full signatures in `references/methods.md`.

## Decision Tree 1 : Which variant of toast() do I call?

```
Q1. Does the message represent a SUCCESSFUL completion of a user-initiated action
    (saved, copied, sent, uploaded)?
    yes -> toast.success("...")
    no  -> Q2

Q2. Does the message represent a FAILURE the user must notice
    (save failed, network error, validation error)?
    yes -> toast.error("...")
    no  -> Q3

Q3. Is the message a soft caution but not a hard failure
    (rate-limit close, deprecation notice, partial success)?
    yes -> toast.warning("...")
    no  -> Q4

Q4. Is the operation IN PROGRESS and you want the toast to update on completion?
    yes -> toast.promise(thePromise, { loading, success, error })
           OR keep the manual route :
              const id = toast.loading("Saving...")
              await save()
              toast.success("Saved", { id })
    no  -> Q5

Q5. Is the message purely informational (FYI, no semantic colour) ?
    yes -> toast.info("...")    OR plain toast("...") for neutral
    no  -> Q6

Q6. Do you need to render arbitrary JSX (a row of buttons, an avatar, a progress bar)?
    yes -> toast.custom((id) => <YourNode />)
    no  -> default toast("...") with description option.
```

Pick the semantic variant whenever possible : with `richColors` enabled on the Toaster, the variants render with green / red / orange / blue accent colors that screen-readers ALSO surface via `role="status"` / `role="alert"`.

## Decision Tree 2 : Should I use toast.promise or manual loading?

```
Q1. Do you have a single, already-existing Promise (e.g., a fetch call,
    an `await` you can wrap) and do all three outcomes resolve in the
    SAME callsite?
    yes -> toast.promise(thePromise, { loading, success, error })
    no  -> Q2

Q2. Do success and error happen in DIFFERENT places (e.g., a long-running
    job where success arrives via WebSocket, error arrives via a polling
    timeout) ?
    yes -> use the MANUAL route with a stable id :
              const id = toast.loading("Processing")
              ...
              toast.success("Done", { id })       // later, anywhere
              // or
              toast.error("Failed", { id })       // later, anywhere
    no  -> Q3

Q3. Do you need the loading toast to update its TEXT during the operation
    (progress percentages, multi-step) ?
    yes -> manual route ; call toast.loading("New text...", { id }) to
           update in place. NEVER call toast.promise then try to push
           interim updates ; toast.promise owns the lifecycle.
    no  -> toast.promise. Simplest API surface, smallest call site.
```

NEVER nest `toast.promise` calls. The inner promise's loading toast clobbers the outer's queue position.

## The `<Toaster />` props

The shadcn `<Toaster />` wrapper passes ALL props through to upstream `sonner`'s `Toaster`. Most-used :

| Prop | Default | Notes |
|------|---------|-------|
| `position` | `"bottom-right"` | Six anchors : `top-left`, `top-center`, `top-right`, `bottom-left`, `bottom-center`, `bottom-right` |
| `duration` | `4000` (ms) | Default for all toasts ; per-toast override via the `duration` option on each `toast()` call. Pass `Infinity` for sticky |
| `expand` | `true` | When `true`, hovering the stack reveals every toast individually ; when `false`, only the top is visible until dismissed |
| `richColors` | `false` | When `true`, `.success` / `.error` / `.warning` / `.info` toasts use background-colored panels with white text instead of neutral popover background |
| `closeButton` | `false` | When `true`, every toast renders a small X close button on hover |
| `visibleToasts` | `3` | Maximum visible stack size ; older toasts are queued |
| `theme` | inherited from `next-themes` | The shadcn wrapper wires this automatically via `useTheme()` ; do NOT pass `theme` manually unless you bypass next-themes |
| `gap` | `14` (px) | Vertical gap between stacked toasts |
| `offset` | `32` (px) | Distance from the chosen viewport corner |
| `hotkey` | `["altKey", "KeyT"]` | Keyboard shortcut to focus the toast region for accessibility |
| `dir` | `"auto"` | `ltr` / `rtl` / `auto` ; respect the document direction |

ALWAYS configure these on the SINGLE `<Toaster />` instance at the app root. NEVER pass them per-toast except via the per-call options above ; the Toaster props are global defaults.

## The shadcn Toaster wrapper, line by line

The shadcn registry adds a thin opinionated wrapper around `sonner`'s `Toaster`. Verbatim from `apps/v4/registry/new-york-v4/ui/sonner.tsx` :

- `"use client"` at the top : non-negotiable, `useTheme()` from `next-themes` is a client hook.
- Imports `Toaster as Sonner` from `"sonner"`, plus five Lucide icons (`CircleCheckIcon`, `InfoIcon`, `OctagonXIcon`, `TriangleAlertIcon`, `Loader2Icon`).
- Reads `theme` from `useTheme()` (default `"system"`) and forwards it as `theme={theme as ToasterProps["theme"]}`. This auto-syncs the toast color scheme to the rest of the shadcn theme without any extra config.
- Sets `className="toaster group"`.
- Replaces the default sonner icons with shadcn-themed Lucide icons sized via `size-4` (`animate-spin` on the loading icon).
- Bridges sonner's internal CSS variables to shadcn theme tokens :
  - `--normal-bg: var(--popover)`
  - `--normal-text: var(--popover-foreground)`
  - `--normal-border: var(--border)`
  - `--border-radius: var(--radius)`
  This is what makes the toast visually identical to a `<Popover>` or `<HoverCard>` panel.
- Spreads `{...props}` LAST, so any prop you pass to `<Toaster />` overrides the wrapper defaults.

ALWAYS use `<Toaster />` from `@/components/ui/sonner` instead of importing `Toaster` directly from `"sonner"`. The shadcn wrapper carries the theme integration and icon set ; the bare upstream `Toaster` does not.

## `next-themes` integration

The wrapper calls `useTheme()` from `next-themes` to forward the current theme (`light` / `dark` / `system`) to sonner. If your project does NOT use `next-themes` :

- Either install it (`pnpm add next-themes`) and add a `<ThemeProvider>` per shadcn dark-mode docs.
- OR strip the `useTheme()` call from `components/ui/sonner.tsx` and hardcode `theme="system"`.

Without `next-themes` mounted, `useTheme()` returns `{ theme: undefined }` and sonner falls back to its own light/dark detection, which may not match your global theme.

## Migration table : useToast() -> toast()

| Pre-2026 (REMOVED) | 2026+ (Sonner) |
|--------------------|----------------|
| `import { useToast } from "@/components/ui/use-toast"` | `import { toast } from "sonner"` |
| `import { Toaster } from "@/components/ui/toaster"` | `import { Toaster } from "@/components/ui/sonner"` |
| `const { toast } = useToast()` | (delete this line ; `toast` is now a top-level import) |
| `toast({ title: "Saved" })` | `toast.success("Saved")` |
| `toast({ title: "Error", variant: "destructive" })` | `toast.error("Error")` |
| `toast({ title: "X", description: "Y" })` | `toast("X", { description: "Y" })` |
| `toast({ title: "X", action: <ToastAction onClick={...}>Undo</ToastAction> })` | `toast("X", { action: { label: "Undo", onClick: () => ... } })` |
| `const { id, dismiss } = toast({...})` then `dismiss()` | `const id = toast(...)` then `toast.dismiss(id)` |

Side-by-side full code samples in `references/examples.md` (last section).

Files that can be DELETED from a migrated project :
- `components/ui/use-toast.ts`
- `components/ui/toast.tsx`
- `components/ui/toaster.tsx` (the old Radix-based one)

Run `pnpm dlx shadcn@latest add sonner` to install the new `components/ui/sonner.tsx`. Then delete the three old files above. Then global-find-and-replace per the migration table.

## Companion Skills

- [shadcn-impl-form-validation](../../shadcn-impl/shadcn-impl-form-validation/SKILL.md) : The canonical place to fire `toast.success` after a form submit and `toast.error` on submit failure ; covers `react-hook-form` integration patterns where the toast becomes the user feedback channel.
- [shadcn-syntax-button](../shadcn-syntax-button/SKILL.md) : Most `toast()` calls fire from a Button onClick handler ; the `asChild` + onClick patterns there pair with this skill.
- [shadcn-syntax-popover-tooltip-hovercard](../shadcn-syntax-popover-tooltip-hovercard/SKILL.md) : Read when deciding between a transient toast and a hover-anchored Tooltip / HoverCard for a piece of UI feedback. Toasts are for ASYNCHRONOUS notifications ; Tooltip / HoverCard are for SYNCHRONOUS, hover-pinned info.
- [shadcn-errors-radix-controlled](../../shadcn-errors/shadcn-errors-radix-controlled/SKILL.md) : Cross-references the RSC / client-boundary trap (which also bites `toast()` calls in RSC files).

## Reference Links

- [references/methods.md](references/methods.md) : Full prop tables for `<Toaster />` and every `toast.*` method, including `ExternalToast` option fields.
- [references/examples.md](references/examples.md) : Seven canonical recipes (root-layout Toaster, simple toast, action+cancel buttons, `toast.promise` around fetch, manual loading with stable id, JSX via `toast.custom`, side-by-side useToast -> sonner migration).
- [references/anti-patterns.md](references/anti-patterns.md) : Six anti-patterns with WHY and FIX.

## Sources

All claims in this skill trace to URLs in `SOURCES.md` :

- https://ui.shadcn.com/docs/components/radix/sonner (canonical shadcn Sonner docs ; Toaster wrapper installation ; usage examples)
- https://sonner.emilkowal.ski (upstream sonner docs ; full `toast.*` API, ExternalToast option fields, Toaster props)
- https://github.com/shadcn-ui/ui (apps/v4/registry/new-york-v4/ui/sonner.tsx ; verbatim shadcn Toaster wrapper, next-themes integration, CSS-variable bridge, Lucide icon set)
- https://ui.shadcn.com/docs/changelog (Toast component + useToast hook REMOVED in 2026 line)
- https://github.com/emilkowalski/sonner (upstream source ; ExternalToast type, position values, theme prop contract)

Verified 2026-05-19.
