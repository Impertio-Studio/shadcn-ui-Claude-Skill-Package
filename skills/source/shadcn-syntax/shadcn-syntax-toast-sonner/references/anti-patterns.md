# shadcn-syntax-toast-sonner : Anti-Patterns

Six failure modes that surface in real shadcn ui projects using Sonner. Each entry : the symptom you observe, the WHY (root cause), and the FIX (concrete code change). Sourced from the upstream `sonner` issue tracker, the shadcn changelog, and the verbatim shadcn `Toaster` wrapper.

## AP-1 : Calling `toast()` with no `<Toaster />` mounted

**Symptom** : You click a button that calls `toast.success("Saved")`. Nothing happens. No error in the console. The toast never appears.

**Why** : `toast()` enqueues a notification into Sonner's client-side store. The store is read by the `<Toaster />` component, which renders the visible toast region. Without a `<Toaster />` mounted somewhere in the tree, there is no consumer for the queue, so the call is effectively a no-op. Sonner does NOT log a warning in production builds.

**Fix** :

```tsx
// app/layout.tsx  (or src/main.tsx for Vite)
import { Toaster } from "@/components/ui/sonner"

export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>
        {children}
        <Toaster />              {/* the missing piece */}
      </body>
    </html>
  )
}
```

ALWAYS mount `<Toaster />` once at the highest reasonable level (root layout or root render in main.tsx). NEVER assume the shadcn CLI installed it for you ; `shadcn add sonner` only adds `components/ui/sonner.tsx` (the wrapper file), it does NOT mount it.

## AP-2 : Importing or calling the removed `useToast` hook

**Symptom** : TypeScript / build error `Cannot find module "@/components/ui/use-toast"` or runtime error `useToast is not a function`. Some projects also see `Module not found: "@/components/ui/toaster"` because the old Radix-based Toaster file was deleted.

**Why** : The shadcn 2026 changelog explicitly REMOVED both the legacy Radix-based `<Toast>` component and the `useToast()` hook. The registry no longer ships `components/ui/use-toast.ts`, `components/ui/toast.tsx`, or the Radix-based `components/ui/toaster.tsx`. Code copied from pre-2026 tutorials still references these files.

**Fix** :

```tsx
// REMOVE every line like this :
import { useToast } from "@/components/ui/use-toast"
import { Toaster } from "@/components/ui/toaster"
import { ToastAction } from "@/components/ui/toast"

// REPLACE with :
import { toast } from "sonner"
import { Toaster } from "@/components/ui/sonner"
// (ToastAction has no direct equivalent ; use the `action` option on toast())

// REMOVE the hook call :
// const { toast } = useToast()           <-- delete

// Old call site :
// toast({ title: "Saved", variant: "destructive" })
// New call site :
toast.error("Saved")    // or toast.success / toast.info / toast.warning
```

See the full migration table in the parent SKILL.md and the side-by-side recipe in `examples.md` §7. ALWAYS replace the hook destructuring with a top-level import. NEVER try to keep the old `useToast()` call shape by writing a custom wrapper around `toast` ; the hook contract was removed precisely because the imperative API does not need it.

## AP-3 : Mounting two `<Toaster />` components

**Symptom** : Every toast appears twice (or N times). Sometimes one copy renders at the top of the screen and another at the bottom. Dismissing one copy leaves the other visible.

**Why** : Each `<Toaster />` instance instantiates its own subscriber to the sonner queue AND renders its own viewport. A `toast()` call notifies EVERY subscriber, so each viewport renders its own copy. Common causes : mounting `<Toaster />` in `RootLayout` AND in a per-page layout, copy-pasting a layout snippet from a tutorial without removing the old one, or mounting it inside a feature-flagged `<Providers>` wrapper that runs twice during dev hot-reload.

**Fix** :

```tsx
// app/layout.tsx
<Toaster />          {/* keep ONE here */}

// app/(dashboard)/layout.tsx
// <Toaster />        <-- DELETE this duplicate

// app/profile/page.tsx
// <Toaster />        <-- DELETE this duplicate too
```

ALWAYS grep your codebase for `<Toaster` before adding a new mount : `grep -r "Toaster" app/ src/`. Keep exactly one mount, in the highest practical location. NEVER mount per-route ; the single root mount is visible to every page.

## AP-4 : `toast.promise` without all three states

**Symptom** : When a promise rejects, the loading toast (spinner + "Saving..." text) stays visible indefinitely. Or, when the promise resolves successfully, no follow-up toast appears.

**Why** : `toast.promise` resolves the loading state into either the `success` or `error` state on the SAME toast id. If you omit `error`, a rejection has nowhere to land and the toast stays in its loading state forever. If you omit `success`, a resolution removes the toast silently instead of confirming completion.

**Fix** :

```tsx
// BROKEN : missing error
toast.promise(saveProfile(payload), {
  loading: "Saving...",
  success: "Saved",
  // error: ...   <-- MISSING : on reject, the toast spins forever
})

// CORRECT
toast.promise(saveProfile(payload), {
  loading: "Saving...",
  success: (data) => `Profile saved : ${data.name}`,
  error: (err) =>
    err instanceof Error ? err.message : "Save failed",
})
```

ALWAYS pass `loading`, `success`, AND `error`. NEVER assume the promise will only resolve ; even "safe" calls (clipboard, navigator APIs) can reject in edge cases (permissions, browser focus, off-the-network). Treat `toast.promise` as a state machine with three terminal states ; all three must be wired.

## AP-5 : Calling `toast()` from a React Server Component

**Symptom** : `Error : toast can not be used in a server component` thrown at request time. Or, the toast call is silently dropped during SSR and never fires after hydration. Or a confusing build error mentioning client-only modules in a server boundary.

**Why** : Sonner stores its toast queue in a client-side module-level singleton. RSCs run on the server, where there is no DOM, no queue, no subscriber. Calling `toast()` from a server file (a `page.tsx` without `"use client"`, a Server Action body, a `loading.tsx` boundary, or any non-client component) reaches into a store that does not exist on the server.

**Fix** :

```tsx
// BROKEN : page.tsx with no "use client" calls toast()
// app/profile/page.tsx
import { toast } from "sonner"

export default function ProfilePage() {
  toast.success("Welcome back")    // <-- crashes in RSC
  return <div>...</div>
}

// CORRECT : split into a server page + a client component
// app/profile/page.tsx                     (server, no toast call)
import { WelcomeToast } from "./welcome-toast"
export default function ProfilePage() {
  return (
    <>
      <WelcomeToast />
      <div>...</div>
    </>
  )
}

// app/profile/welcome-toast.tsx            (client, fires toast)
"use client"
import { useEffect } from "react"
import { toast } from "sonner"
export function WelcomeToast() {
  useEffect(() => {
    toast.success("Welcome back")
  }, [])
  return null
}
```

ALWAYS fire `toast()` from a `"use client"` component, ideally inside an event handler or a `useEffect`. NEVER call `toast()` directly in the render path of an RSC, Server Action, or top-level page that has no `"use client"` directive.

For Server Actions specifically : return a result from the action and call `toast.*` from the client `onSubmit` / `useActionState` handler that consumed the result. Server Actions cannot fire toasts directly because they execute on the server.

## AP-6 : Missing `"use client"` on a component that calls `toast()`

**Symptom** : Looks similar to AP-5, but the file is a regular component (not a page). Build error `"toast" cannot be imported from a Server Component` or `useToast / toast is only allowed in client components` during a Next.js build. In Vite / SPA setups the symptom is different : the import works but the build chunks balloon, or the call silently no-ops because the bundler stripped the client-only branch.

**Why** : The component file imports `toast` from `"sonner"`, which transitively pulls in client-only code (window references, the in-memory queue, the imperative dispatcher). Without `"use client"` at the top of the file, Next.js treats the file as a Server Component by default and rejects the import. In Vite, the file works but suffers tree-shaking surprises and event-handler hydration glitches because the wider context expects a server-render contract that the component breaks.

**Fix** :

```tsx
// BROKEN : missing "use client" on a component that fires toast()
import { toast } from "sonner"
import { Button } from "@/components/ui/button"

export function CopyButton({ value }: { value: string }) {
  return (
    <Button onClick={() => toast.success("Copied")}>
      Copy
    </Button>
  )
}

// CORRECT
"use client"

import { toast } from "sonner"
import { Button } from "@/components/ui/button"

export function CopyButton({ value }: { value: string }) {
  return (
    <Button onClick={() => toast.success("Copied")}>
      Copy
    </Button>
  )
}
```

ALWAYS place `"use client"` as the very first line (before imports) in any component file that calls `toast.*` OR imports `toast` from `"sonner"`. NEVER rely on the parent component's client boundary to "cover" a child file ; Next.js evaluates the directive per-file, not per-tree. A single missing `"use client"` in a leaf component is enough to break the build.

This same rule applies to the `<Toaster />` wrapper file (`components/ui/sonner.tsx`). The shadcn CLI ships it with `"use client"` at the top. NEVER strip that directive ; the wrapper uses `useTheme()` from `next-themes`, a client-only hook.

## Sources

- https://ui.shadcn.com/docs/changelog (Toast component + useToast hook REMOVED line)
- https://ui.shadcn.com/docs/components/radix/sonner (canonical shadcn Sonner docs)
- https://sonner.emilkowal.ski (Toaster mount rules, toast.promise contract, ExternalToast)
- https://github.com/shadcn-ui/ui (apps/v4/registry/new-york-v4/ui/sonner.tsx ; `"use client"` directive on the wrapper)
- https://github.com/emilkowalski/sonner (upstream issues : silent-no-op without Toaster, duplicate-toaster bug reports)

Verified 2026-05-19.
