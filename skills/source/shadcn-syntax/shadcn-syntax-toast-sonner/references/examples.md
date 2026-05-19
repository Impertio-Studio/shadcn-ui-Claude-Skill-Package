# shadcn-syntax-toast-sonner : Canonical Examples

Seven recipes, each runnable in a fresh shadcn ui project with Sonner installed via `pnpm dlx shadcn@latest add sonner`. Every consumer that calls `toast()` carries `"use client"`. The `<Toaster />` lives once, in the app root.

## 1. Mount the Toaster in the app root

Next.js App Router :

```tsx
// app/layout.tsx
import { Toaster } from "@/components/ui/sonner"
import "./globals.css"

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body>
        {children}
        <Toaster richColors closeButton position="bottom-right" />
      </body>
    </html>
  )
}
```

Vite / React Router :

```tsx
// src/main.tsx
import React from "react"
import ReactDOM from "react-dom/client"
import App from "./App"
import { Toaster } from "@/components/ui/sonner"
import "./index.css"

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode>
    <App />
    <Toaster richColors closeButton position="bottom-right" />
  </React.StrictMode>
)
```

ALWAYS mount exactly ONE `<Toaster />` at the highest reasonable level (root layout / root provider). NEVER mount a second one inside a nested route or page ; every page in your app already sees the root-mounted Toaster.

## 2. Simple toast from a Button onClick

```tsx
"use client"

import { toast } from "sonner"
import { Button } from "@/components/ui/button"

export function CopyLinkButton({ url }: { url: string }) {
  return (
    <Button
      onClick={async () => {
        await navigator.clipboard.writeText(url)
        toast.success("Link copied to clipboard")
      }}
    >
      Copy link
    </Button>
  )
}
```

Notice : `toast.success` (semantic variant), not plain `toast()` ; this lets `richColors` paint the toast green. The component carries `"use client"` because `toast()` is a client-only call.

## 3. Toast with action + cancel buttons

```tsx
"use client"

import { toast } from "sonner"
import { Button } from "@/components/ui/button"

export function DeleteRowButton({ rowId }: { rowId: string }) {
  return (
    <Button
      variant="destructive"
      onClick={() => {
        // Optimistically remove the row from the UI here
        toast("Row deleted", {
          description: "It will be permanently removed in 5 seconds.",
          duration: 5000,
          action: {
            label: "Undo",
            onClick: () => {
              // Restore the row in the UI here
              toast.success("Row restored")
            },
          },
          cancel: {
            label: "Dismiss",
            onClick: () => {
              // No-op ; user just closed the toast
            },
          },
        })
      }}
    >
      Delete
    </Button>
  )
}
```

The `action` object's `onClick` is the canonical place to wire an Undo. Keep the `duration` short enough that the user notices the toast but long enough that the Undo is realistically clickable (3000-5000ms is typical).

## 4. Wrap a fetch with toast.promise

```tsx
"use client"

import { toast } from "sonner"
import { Button } from "@/components/ui/button"

async function saveProfile(payload: { name: string }) {
  const res = await fetch("/api/profile", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(payload),
  })
  if (!res.ok) throw new Error(`HTTP ${res.status}`)
  return (await res.json()) as { id: string; name: string }
}

export function SaveProfileButton({ name }: { name: string }) {
  return (
    <Button
      onClick={() => {
        toast.promise(saveProfile({ name }), {
          loading: "Saving profile...",
          success: (data) => `Profile saved : ${data.name}`,
          error: (err) =>
            err instanceof Error ? err.message : "Save failed",
        })
      }}
    >
      Save
    </Button>
  )
}
```

`toast.promise` resolves the loading toast into the success or error toast on the SAME id. ALWAYS pass all three states (`loading`, `success`, `error`). NEVER omit `error` ; a rejected promise leaves the loading toast spinning forever.

## 5. Manual loading + dismiss with a stable id

Use this when success and error happen in different places (long-running job, WebSocket completion, polling timeout) and `toast.promise` cannot represent the lifecycle.

```tsx
"use client"

import { useEffect } from "react"
import { toast } from "sonner"
import { Button } from "@/components/ui/button"

export function StartLongJob() {
  return (
    <Button
      onClick={() => {
        const id = toast.loading("Starting export...", {
          duration: Infinity,    // stick until we resolve it
          dismissible: false,    // user cannot swipe it away mid-job
        })

        // Imagine this id arrives async from a job-runner :
        const jobId = startExportJob()

        const ws = new WebSocket(`/api/jobs/${jobId}/events`)
        ws.onmessage = (event) => {
          const msg = JSON.parse(event.data)
          if (msg.kind === "progress") {
            toast.loading(`Exporting... ${msg.percent}%`, { id })
          } else if (msg.kind === "done") {
            toast.success("Export ready", {
              id,
              description: msg.downloadUrl,
              action: { label: "Download", onClick: () => location.assign(msg.downloadUrl) },
            })
            ws.close()
          } else if (msg.kind === "error") {
            toast.error("Export failed", { id, description: msg.message })
            ws.close()
          }
        }
      }}
    >
      Start export
    </Button>
  )
}

// dummy
declare function startExportJob(): string
```

Two patterns combined :

- `duration: Infinity` + `dismissible: false` to keep the loading toast pinned.
- Reusing the same `id` across `toast.loading`, `toast.success`, `toast.error` so the same visual slot updates instead of stacking new toasts.

## 6. Custom toast with arbitrary JSX

```tsx
"use client"

import { toast } from "sonner"
import { Button } from "@/components/ui/button"
import { Avatar, AvatarFallback, AvatarImage } from "@/components/ui/avatar"

export function FollowToastButton({
  user,
}: {
  user: { name: string; avatarUrl: string; handle: string }
}) {
  return (
    <Button
      onClick={() => {
        toast.custom((id) => (
          <div className="flex items-center gap-3 rounded-md border bg-background p-3 shadow-md">
            <Avatar>
              <AvatarImage src={user.avatarUrl} alt={user.name} />
              <AvatarFallback>{user.name.slice(0, 2)}</AvatarFallback>
            </Avatar>
            <div className="flex-1">
              <p className="text-sm font-medium">{user.name}</p>
              <p className="text-xs text-muted-foreground">
                started following @{user.handle}
              </p>
            </div>
            <Button
              size="sm"
              variant="outline"
              onClick={() => toast.dismiss(id)}
            >
              Dismiss
            </Button>
          </div>
        ))
      }}
    >
      Trigger follow notification
    </Button>
  )
}
```

`toast.custom` receives the toast id ; pass it to `toast.dismiss(id)` inside the rendered JSX so the user can close the custom toast even when the Toaster's `closeButton` is off.

## 7. Migration : useToast() (pre-2026) -> toast() (Sonner)

The `useToast()` hook and the Radix-based `<Toast>` family were REMOVED per the shadcn 2026 changelog. Below is a side-by-side migration of a typical save handler.

**Before** (pre-2026, DEAD CODE) :

```tsx
"use client"

import { useToast } from "@/components/ui/use-toast"
import { ToastAction } from "@/components/ui/toast"
import { Button } from "@/components/ui/button"

export function SaveButton({ payload }: { payload: { name: string } }) {
  const { toast } = useToast()

  return (
    <Button
      onClick={async () => {
        try {
          await saveProfile(payload)
          toast({
            title: "Saved",
            description: `Profile "${payload.name}" was saved.`,
            action: (
              <ToastAction altText="Undo" onClick={() => undoSave()}>
                Undo
              </ToastAction>
            ),
          })
        } catch (err) {
          toast({
            title: "Save failed",
            description: err instanceof Error ? err.message : "Unknown error",
            variant: "destructive",
          })
        }
      }}
    >
      Save
    </Button>
  )
}
```

Required at the app root (also DEAD) :

```tsx
// app/layout.tsx (pre-2026 DEAD CODE)
import { Toaster } from "@/components/ui/toaster"      // OLD : Radix-based Toaster
// ...
<Toaster />
```

**After** (2026+, Sonner) :

```tsx
"use client"

import { toast } from "sonner"
import { Button } from "@/components/ui/button"

export function SaveButton({ payload }: { payload: { name: string } }) {
  return (
    <Button
      onClick={async () => {
        try {
          await saveProfile(payload)
          toast.success("Saved", {
            description: `Profile "${payload.name}" was saved.`,
            action: { label: "Undo", onClick: () => undoSave() },
          })
        } catch (err) {
          toast.error("Save failed", {
            description: err instanceof Error ? err.message : "Unknown error",
          })
        }
      }}
    >
      Save
    </Button>
  )
}
```

Required at the app root (NEW) :

```tsx
// app/layout.tsx
import { Toaster } from "@/components/ui/sonner"       // NEW : sonner wrapper
// ...
<Toaster richColors closeButton />
```

Step-by-step migration recipe :

1. `pnpm dlx shadcn@latest add sonner` to add `components/ui/sonner.tsx`.
2. Delete `components/ui/use-toast.ts`, `components/ui/toast.tsx`, and `components/ui/toaster.tsx` (the old Radix one).
3. Global find-and-replace : `from "@/components/ui/toaster"` -> `from "@/components/ui/sonner"`.
4. Global find-and-replace : `from "@/components/ui/use-toast"` -> `from "sonner"`.
5. Remove every `const { toast } = useToast()` line ; `toast` is now a top-level import.
6. Rewrite each call site :
   - `toast({ title: X })` -> `toast(X)` or `toast.success(X)` if it was a success message.
   - `toast({ title: X, description: Y })` -> `toast(X, { description: Y })`.
   - `toast({ title: X, variant: "destructive" })` -> `toast.error(X)`.
   - `toast({ ..., action: <ToastAction onClick={fn}>L</ToastAction> })` -> `toast(..., { action: { label: "L", onClick: fn } })`.
7. Remove any remaining `import { ToastAction } from "@/components/ui/toast"` lines (the path is gone).
8. Verify : run the app, click every former toast trigger, confirm toasts appear with `richColors` and the correct semantic variant.

The migration is mechanical for ~90% of call sites. The only two cases that need hand-holding :

- Toasts that relied on the old `useToast()` return value (`{ id, dismiss, update }`). The new equivalent : `const id = toast(...); toast.dismiss(id); toast(..., { id })` (the third call UPDATES instead of adding).
- Custom Radix `ToastProvider` / `ToastViewport` configuration. Discard ; the sonner `<Toaster />` is the single replacement.

## Sources

- https://ui.shadcn.com/docs/components/radix/sonner
- https://sonner.emilkowal.ski (examples gallery + ExternalToast option fields)
- https://github.com/shadcn-ui/ui (apps/v4/registry/new-york-v4/ui/sonner.tsx)
- https://ui.shadcn.com/docs/changelog (useToast removal line)

Verified 2026-05-19.
