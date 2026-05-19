# Dialog : Canonical Examples

Six working recipes. Every snippet is verified against `apps/v4/registry/new-york-v4/ui/dialog.tsx` and https://ui.shadcn.com/docs/components/radix/dialog (2026-05-19).

All examples assume :
- shadcn ui evergreen-2026 (registry style `new-york-v4`)
- Tailwind v4 (no `tailwind.config.js` ; tokens via `@theme inline`)
- React 19 (no explicit `forwardRef` needed)
- The dialog file `components/ui/dialog.tsx` has `"use client"` at the top.

## Example 1 : Minimal Dialog (uncontrolled)

The smallest correct dialog. Radix manages state ; you only render markup.

```tsx
"use client"

import {
  Dialog,
  DialogTrigger,
  DialogContent,
  DialogHeader,
  DialogFooter,
  DialogTitle,
  DialogDescription,
  DialogClose,
} from "@/components/ui/dialog"
import { Button } from "@/components/ui/button"

export function DeleteFileDialog() {
  return (
    <Dialog>
      <DialogTrigger asChild>
        <Button variant="destructive">Delete file</Button>
      </DialogTrigger>
      <DialogContent>
        <DialogHeader>
          <DialogTitle>Delete file ?</DialogTitle>
          <DialogDescription>
            This action permanently removes the file. It cannot be undone.
          </DialogDescription>
        </DialogHeader>
        <DialogFooter>
          <DialogClose asChild>
            <Button variant="outline">Cancel</Button>
          </DialogClose>
          <Button variant="destructive">Delete</Button>
        </DialogFooter>
      </DialogContent>
    </Dialog>
  )
}
```

Notes :
- No `useState`, no `open`, no `onOpenChange`. Radix tracks visibility internally.
- DialogTrigger wraps a Button via `asChild`. The Button receives all merged props (data-state, click handler).
- DialogClose wraps the "Cancel" Button so clicking it closes without a manual handler.
- The "Delete" Button is plain : the parent will handle the actual delete logic and may call `setOpen(false)` if it later switches to controlled state.

## Example 2 : Controlled Dialog with state

Use controlled state when something outside the dialog needs to read or write its open state (URL params, parent form, programmatic open on data-arrival).

```tsx
"use client"

import { useState } from "react"
import {
  Dialog,
  DialogTrigger,
  DialogContent,
  DialogHeader,
  DialogFooter,
  DialogTitle,
  DialogDescription,
  DialogClose,
} from "@/components/ui/dialog"
import { Button } from "@/components/ui/button"

export function ControlledDialog() {
  const [open, setOpen] = useState(false)

  return (
    <>
      <Button onClick={() => setOpen(true)}>Open from anywhere</Button>

      <Dialog open={open} onOpenChange={setOpen}>
        <DialogContent>
          <DialogHeader>
            <DialogTitle>Edit profile</DialogTitle>
            <DialogDescription>
              Changes save when you click "Save".
            </DialogDescription>
          </DialogHeader>
          <DialogFooter>
            <DialogClose asChild>
              <Button variant="outline">Cancel</Button>
            </DialogClose>
            <Button onClick={() => setOpen(false)}>Save</Button>
          </DialogFooter>
        </DialogContent>
      </Dialog>
    </>
  )
}
```

Notes :
- BOTH `open` and `onOpenChange` are passed. NEVER pass `open` without `onOpenChange`.
- The dialog can now be opened from an external Button (not the trigger), from a useEffect, or via URL state. Radix calls `setOpen(false)` on Esc, pointer-down-outside, and the built-in X close button.
- A DialogTrigger is no longer required when external code opens the dialog ; if you want one too, you can include it alongside the external opener.

## Example 3 : Form inside Dialog (close on submit success)

The canonical form-in-dialog pattern. Close after a successful submit ; stay open on validation failure.

```tsx
"use client"

import { useState } from "react"
import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import { z } from "zod"
import {
  Dialog,
  DialogTrigger,
  DialogContent,
  DialogHeader,
  DialogFooter,
  DialogTitle,
  DialogDescription,
  DialogClose,
} from "@/components/ui/dialog"
import {
  Form,
  FormControl,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from "@/components/ui/form"
import { Input } from "@/components/ui/input"
import { Button } from "@/components/ui/button"

const schema = z.object({
  name: z.string().min(2, "Name is too short"),
})

export function EditProfileDialog() {
  const [open, setOpen] = useState(false)
  const form = useForm<z.infer<typeof schema>>({
    resolver: zodResolver(schema),
    defaultValues: { name: "" },
  })

  async function onSubmit(values: z.infer<typeof schema>) {
    await saveProfile(values)
    setOpen(false)              // ALWAYS synchronously after success
    form.reset()
  }

  return (
    <Dialog open={open} onOpenChange={setOpen}>
      <DialogTrigger asChild>
        <Button>Edit profile</Button>
      </DialogTrigger>
      <DialogContent>
        <DialogHeader>
          <DialogTitle>Edit profile</DialogTitle>
          <DialogDescription>Update your display name.</DialogDescription>
        </DialogHeader>
        <Form {...form}>
          <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
            <FormField
              control={form.control}
              name="name"
              render={({ field }) => (
                <FormItem>
                  <FormLabel>Name</FormLabel>
                  <FormControl><Input {...field} /></FormControl>
                  <FormMessage />
                </FormItem>
              )}
            />
            <DialogFooter>
              <DialogClose asChild>
                <Button type="button" variant="outline">Cancel</Button>
              </DialogClose>
              <Button type="submit" disabled={form.formState.isSubmitting}>
                Save
              </Button>
            </DialogFooter>
          </form>
        </Form>
      </DialogContent>
    </Dialog>
  )
}

async function saveProfile(values: { name: string }) {
  // call to backend …
}
```

Notes :
- `setOpen(false)` runs INSIDE the success branch of `onSubmit`. On validation failure, react-hook-form throws and `setOpen(false)` never runs, so the dialog stays open with errors visible.
- The Submit button is a plain `<Button type="submit">`, NEVER a `<DialogClose asChild><Button type="submit"></DialogClose>`. Wrapping submit in DialogClose closes the dialog before validation runs.
- The Cancel button uses `<DialogClose asChild><Button type="button">` ; the explicit `type="button"` prevents it from submitting the surrounding form.

## Example 4 : sr-only DialogTitle (visually hidden but a11y-compliant)

Use when the dialog is a content surface (image lightbox, document preview, embedded map) that has no visible heading but still needs an accessible name.

```tsx
"use client"

import {
  Dialog,
  DialogTrigger,
  DialogContent,
  DialogTitle,
} from "@/components/ui/dialog"
import { Button } from "@/components/ui/button"

export function ImageLightbox({ src, alt }: { src: string; alt: string }) {
  return (
    <Dialog>
      <DialogTrigger asChild>
        <Button variant="ghost">View image</Button>
      </DialogTrigger>
      <DialogContent
        className="max-w-3xl p-0"
        aria-describedby={undefined}
      >
        <DialogTitle className="sr-only">{alt}</DialogTitle>
        <img src={src} alt={alt} className="w-full h-auto rounded-lg" />
      </DialogContent>
    </Dialog>
  )
}
```

Notes :
- `className="sr-only"` keeps the title in the DOM (Radix needs it for `aria-labelledby`) but hides it from sighted users.
- `aria-describedby={undefined}` on DialogContent suppresses the Radix warning about a missing DialogDescription.
- Alternative : `<VisuallyHidden asChild><DialogTitle>{alt}</DialogTitle></VisuallyHidden>` using `import { VisuallyHidden } from "radix-ui"`.
- NEVER omit DialogTitle entirely. Both Radix dev-mode and axe-core flag this as a critical a11y violation (issue #5746).

## Example 5 : Custom Portal container

Use when the dialog must render inside a specific DOM ancestor (shadow DOM root, iframe document body, deliberately-constrained test root). Rare.

```tsx
"use client"

import { useRef, useEffect, useState } from "react"
import {
  Dialog,
  DialogTrigger,
  DialogPortal,
  DialogOverlay,
  DialogContent,
  DialogHeader,
  DialogTitle,
  DialogDescription,
} from "@/components/ui/dialog"
import { Button } from "@/components/ui/button"

export function ScopedDialog() {
  const hostRef = useRef<HTMLDivElement>(null)
  const [hostReady, setHostReady] = useState(false)

  useEffect(() => {
    setHostReady(true)
  }, [])

  return (
    <div className="relative isolate">
      <div ref={hostRef} />
      <Dialog>
        <DialogTrigger asChild>
          <Button>Open scoped dialog</Button>
        </DialogTrigger>
        {hostReady && hostRef.current && (
          <DialogPortal container={hostRef.current}>
            <DialogOverlay />
            <DialogContent>
              <DialogHeader>
                <DialogTitle>Scoped dialog</DialogTitle>
                <DialogDescription>
                  This dialog renders inside the bounded host element,
                  not document.body.
                </DialogDescription>
              </DialogHeader>
            </DialogContent>
          </DialogPortal>
        )}
      </Dialog>
    </div>
  )
}
```

Notes :
- When you take over `<DialogPortal>` manually, ALSO render `<DialogOverlay />` yourself. DialogContent only auto-wraps when used standalone.
- `hostReady` gate prevents rendering before the ref attaches ; without it the first render passes `container={null}` and Radix falls back to `document.body`, defeating the purpose.
- DialogContent then renders WITHOUT its own auto-portal - when the closest `DialogPortalContext` is set, Radix uses it. (Mechanism : DialogContent's auto-portal is at the JSX level ; if you replace the auto-portal with your own, both produce the same `DialogPrimitive.Content` underneath, and Radix de-dupes via context.)

## Example 6 : asChild DialogClose on a custom Button

Use when you need the dialog's close to inherit your Button styling, a tooltip, or an analytics handler. The Slot pattern merges click handlers.

```tsx
"use client"

import {
  Dialog,
  DialogTrigger,
  DialogContent,
  DialogHeader,
  DialogFooter,
  DialogTitle,
  DialogClose,
} from "@/components/ui/dialog"
import { Button } from "@/components/ui/button"
import { Tooltip, TooltipTrigger, TooltipContent } from "@/components/ui/tooltip"

export function ConfirmDialog() {
  function trackCancel() {
    // analytics …
  }

  return (
    <Dialog>
      <DialogTrigger asChild>
        <Button>Open</Button>
      </DialogTrigger>
      <DialogContent>
        <DialogHeader>
          <DialogTitle>Discard changes ?</DialogTitle>
        </DialogHeader>
        <DialogFooter>
          <DialogClose asChild>
            <Tooltip>
              <TooltipTrigger asChild>
                <Button variant="outline" onClick={trackCancel}>
                  Cancel
                </Button>
              </TooltipTrigger>
              <TooltipContent>Discard your edits</TooltipContent>
            </Tooltip>
          </DialogClose>
          <Button variant="destructive">Discard</Button>
        </DialogFooter>
      </DialogContent>
    </Dialog>
  )
}
```

Notes :
- DialogClose receives a single child (the Tooltip composition). Radix Slot merges its own onClick (close the dialog) with the child's onClick (`trackCancel`). Both fire.
- The Tooltip itself uses `asChild` to wrap the Button ; both wrappers compose cleanly because each receives exactly one child.
- NEVER place two children inside `<DialogClose asChild>`. `<DialogClose asChild><Button>Cancel</Button><Icon /></DialogClose>` throws "React.Children.only expected to receive a single React element child".
