# Examples : Radix Controlled-State Patterns

Every example pairs a WRONG version (the trap) with a RIGHT version
(the fix). Verified against the shadcn `new-york-v4` Dialog / Popover
/ DropdownMenu / Drawer / Select / Sheet sources and the Radix UI
docs, 2026-05-19.

## Example 1 : open Without onOpenChange (the Half-Controlled Trap)

The single most common Radix bug. A developer wires `open` to a
boolean derived from a parent prop or a route segment, forgets the
setter, and the dialog locks shut. The X button, Escape, and the
overlay all stop working.

### WRONG : open prop alone, dialog is read-only

```tsx
"use client"
import { Dialog, DialogContent, DialogTitle, DialogTrigger } from "@/components/ui/dialog"
import { Button } from "@/components/ui/button"

export function EditDialog({ user }: { user: { id: string; name: string } }) {
  // This boolean controls whether the dialog appears.
  const isEditing = Boolean(user.id)

  return (
    <Dialog open={isEditing}>
      <DialogTrigger asChild><Button>Edit</Button></DialogTrigger>
      <DialogContent>
        <DialogTitle>Edit user</DialogTitle>
        <p>The Escape key, overlay click, and X button all do nothing.</p>
      </DialogContent>
    </Dialog>
  )
}
```

The internal close handlers fire `onOpenChange(false)` into the void.
The dialog stays open until the page refreshes.

### RIGHT : pair `open` with `onOpenChange`, sync to parent state

```tsx
"use client"
import * as React from "react"
import { Dialog, DialogContent, DialogTitle, DialogTrigger } from "@/components/ui/dialog"
import { Button } from "@/components/ui/button"

export function EditDialog({ user, onClose }: {
  user: { id: string; name: string }
  onClose: () => void
}) {
  const [open, setOpen] = React.useState(Boolean(user.id))

  return (
    <Dialog
      open={open}
      onOpenChange={(next) => {
        setOpen(next)
        if (!next) onClose()
      }}
    >
      <DialogTrigger asChild><Button>Edit</Button></DialogTrigger>
      <DialogContent>
        <DialogTitle>Edit user</DialogTitle>
        <p>Escape, overlay click, and the X button all close it.</p>
      </DialogContent>
    </Dialog>
  )
}
```

### RIGHT (alternative) : drop the open prop, use defaultOpen

If no external code needs to read or set the open state, prefer
uncontrolled :

```tsx
<Dialog defaultOpen={Boolean(user.id)}>
  ...
</Dialog>
```

Radix manages state internally. The dialog closes correctly via
Escape, overlay, X button.

## Example 2 : Controlled Sync With Form Submit (close on success)

The mutation handler is the source of truth. The dialog closes on
`onSuccess`, NOT in the synchronous submit handler.

```tsx
"use client"
import * as React from "react"
import { useMutation } from "@tanstack/react-query"
import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import { z } from "zod"
import { toast } from "sonner"

import {
  Dialog, DialogContent, DialogDescription, DialogFooter,
  DialogHeader, DialogTitle, DialogTrigger,
} from "@/components/ui/dialog"
import { Button } from "@/components/ui/button"
import { Form, FormControl, FormField, FormItem, FormLabel, FormMessage }
  from "@/components/ui/form"
import { Input } from "@/components/ui/input"

const schema = z.object({ name: z.string().min(1) })
type Data = z.infer<typeof schema>

export function UserDialog({ saveUser }: { saveUser: (d: Data) => Promise<void> }) {
  const [open, setOpen] = React.useState(false)

  const form = useForm<Data>({
    resolver: zodResolver(schema),
    defaultValues: { name: "" },
  })

  const mutation = useMutation({
    mutationFn: saveUser,
    onSuccess: () => {
      setOpen(false)            // close ONLY after the server confirms
      form.reset()
      toast.success("Saved.")
    },
    onError: (e) => toast.error((e as Error).message),
  })

  return (
    <Dialog open={open} onOpenChange={setOpen}>
      <DialogTrigger asChild><Button>New user</Button></DialogTrigger>
      <DialogContent>
        <DialogHeader>
          <DialogTitle>New user</DialogTitle>
          <DialogDescription>Required fields are marked.</DialogDescription>
        </DialogHeader>
        <Form {...form}>
          <form
            onSubmit={form.handleSubmit((data) => mutation.mutate(data))}
            className="grid gap-4"
          >
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
              <Button
                variant="outline"
                type="button"
                onClick={() => setOpen(false)}
              >
                Cancel
              </Button>
              <Button type="submit" disabled={mutation.isPending}>
                {mutation.isPending ? "Saving..." : "Save"}
              </Button>
            </DialogFooter>
          </form>
        </Form>
      </DialogContent>
    </Dialog>
  )
}
```

Key rules :

1. `setOpen(false)` lives ONLY inside `onSuccess`. Never in the
   submit handler's synchronous body.
2. The Save button is disabled while pending. The user sees the
   in-flight state.
3. `form.reset()` runs alongside `setOpen(false)` so the next open
   starts clean.

## Example 3 : asChild With Next.js Link (forwardRef-compat)

```tsx
"use client"
import Link from "next/link"
import { DialogTrigger } from "@/components/ui/dialog"

// CORRECT : single child, Link forwards refs since v13
export function EditTrigger() {
  return (
    <DialogTrigger asChild>
      <Link href="/users/123" prefetch={false}>
        Edit user 123
      </Link>
    </DialogTrigger>
  )
}
```

The Slot merges `onClick` (the dialog open handler), `data-state`,
and `aria-expanded` onto the `<a>` rendered by `Link`. The ref is
composed correctly because Next.js `Link` uses `React.forwardRef`
since v13.

### WRONG : two children inside asChild

```tsx
// Slot crashes : "React.Children.only expected to receive a single React element child"
<DialogTrigger asChild>
  <Link href="/edit"><Icon /></Link>
  <span>Edit</span>
</DialogTrigger>
```

### RIGHT : wrap inside the single child

```tsx
<DialogTrigger asChild>
  <Link href="/edit">
    <Icon />
    <span>Edit</span>
  </Link>
</DialogTrigger>
```

## Example 4 : Portal Scoped To A Specific Container

When a transformed ancestor breaks the default portal placement,
provide a `container` prop to anchor the portal inside a div you
control.

```tsx
"use client"
import * as React from "react"
import {
  Dialog, DialogContent, DialogOverlay, DialogPortal,
  DialogTitle, DialogTrigger,
} from "@/components/ui/dialog"
import { Button } from "@/components/ui/button"

export function ScopedPortalDialog() {
  // Portal host lives inside our component tree, so we control its
  // stacking context and z-index.
  const hostRef = React.useRef<HTMLDivElement | null>(null)
  const [open, setOpen] = React.useState(false)

  return (
    <div className="relative isolate z-0">
      <div
        ref={hostRef}
        className="fixed inset-0 z-50 pointer-events-none"
        // pointer-events-none so the empty host does not block clicks
        // when the dialog is closed. The portal children re-enable
        // pointer events inside the overlay.
      />
      <Dialog open={open} onOpenChange={setOpen}>
        <DialogTrigger asChild>
          <Button>Open scoped dialog</Button>
        </DialogTrigger>
        <DialogPortal container={hostRef.current}>
          <DialogOverlay />
          <DialogContent>
            <DialogTitle>Scoped portal</DialogTitle>
            <p>Rendered inside our own host div, not document.body.</p>
          </DialogContent>
        </DialogPortal>
      </Dialog>
    </div>
  )
}
```

Use this when :

- A `transform` / `filter` / `perspective` / `will-change` ancestor
  breaks the default body-portal stacking.
- You need to control the z-index relative to other portals (toast,
  combobox dropdown, command palette).
- The dialog must live inside a Shadow DOM host that
  `document.body` does not see.

## Example 5 : Popover modal=true With Form Inside

The default Popover is non-modal. A form inside a non-modal Popover
will lose focus the instant the user clicks outside the input area.
Set `modal={true}` to trap focus.

```tsx
"use client"
import * as React from "react"
import { Popover, PopoverContent, PopoverTrigger } from "@/components/ui/popover"
import { Button } from "@/components/ui/button"
import { Input } from "@/components/ui/input"
import { Label } from "@/components/ui/label"

export function FilterPopover() {
  const [open, setOpen] = React.useState(false)
  const [q, setQ] = React.useState("")

  return (
    <Popover open={open} onOpenChange={setOpen} modal={true}>
      <PopoverTrigger asChild>
        <Button variant="outline">Filter</Button>
      </PopoverTrigger>
      <PopoverContent className="w-80" align="end">
        <form
          onSubmit={(e) => {
            e.preventDefault()
            // ALWAYS close AFTER the mutation if it is async.
            // For this synchronous filter, close after the parent
            // state has been updated.
            applyFilter(q)
            setOpen(false)
          }}
          className="grid gap-3"
        >
          <Label htmlFor="filter-q">Query</Label>
          <Input
            id="filter-q"
            value={q}
            onChange={(e) => setQ(e.target.value)}
            autoFocus
          />
          <Button type="submit" size="sm">Apply</Button>
        </form>
      </PopoverContent>
    </Popover>
  )
}

function applyFilter(q: string) {
  // app-specific
}
```

With `modal={true}` :

- Tab cycles inside the form.
- Outside-pointer-down closes the popover via
  `onPointerDownOutside`.
- Body scroll is locked so the popover stays anchored.
- Focus returns to the Filter button on close.

WITHOUT `modal={true}` :

- A click on a sibling card (outside the popover) takes focus AND
  fires the sibling's `onClick` simultaneously, which can be a
  navigation event. The popover closes but the user has already
  left the page.

## Example 6 : Focus Restoration Override (onCloseAutoFocus)

The trigger is being removed by the close action (e.g., a "delete
this row" dialog where the row itself is the trigger). Radix would
try to return focus to a now-unmounted element. Override the
return target :

```tsx
"use client"
import * as React from "react"
import {
  AlertDialog, AlertDialogAction, AlertDialogCancel,
  AlertDialogContent, AlertDialogDescription, AlertDialogFooter,
  AlertDialogHeader, AlertDialogTitle, AlertDialogTrigger,
} from "@/components/ui/alert-dialog"
import { Button } from "@/components/ui/button"

export function DeleteRow({
  rowId,
  onDelete,
  returnFocusRef,
}: {
  rowId: string
  onDelete: (id: string) => void
  returnFocusRef: React.RefObject<HTMLButtonElement>
}) {
  return (
    <AlertDialog>
      <AlertDialogTrigger asChild>
        <Button variant="destructive" size="sm">Delete</Button>
      </AlertDialogTrigger>
      <AlertDialogContent
        onCloseAutoFocus={(e) => {
          // The trigger row will be unmounted on confirm. Return focus
          // to the table's "Add row" button instead.
          e.preventDefault()
          returnFocusRef.current?.focus()
        }}
      >
        <AlertDialogHeader>
          <AlertDialogTitle>Delete row</AlertDialogTitle>
          <AlertDialogDescription>
            This cannot be undone.
          </AlertDialogDescription>
        </AlertDialogHeader>
        <AlertDialogFooter>
          <AlertDialogCancel>Cancel</AlertDialogCancel>
          <AlertDialogAction onClick={() => onDelete(rowId)}>
            Delete
          </AlertDialogAction>
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
  )
}
```

## Example 7 : Cancelling Escape With onEscapeKeyDown

A wizard dialog should NOT close on Escape because the user is
mid-flow. Cancel the Escape :

```tsx
<DialogContent
  onEscapeKeyDown={(e) => {
    if (formIsDirty) {
      e.preventDefault()
      // Show confirmation toast instead of closing.
      toast("Please save or discard your changes first.")
    }
  }}
  onPointerDownOutside={(e) => {
    if (formIsDirty) e.preventDefault()
  }}
>
  ...
</DialogContent>
```

Symmetrically cancel `onPointerDownOutside` so overlay clicks also
respect the dirty-form guard.

## Example 8 : DropdownMenu That Stays Open On Item Click

By default `DropdownMenuItem` closes the menu on click. Override
with `e.preventDefault()` inside `onSelect` for items that should
NOT close (e.g., a checkbox item that toggles state without
dismissing the menu) :

```tsx
<DropdownMenu>
  <DropdownMenuTrigger asChild><Button>Settings</Button></DropdownMenuTrigger>
  <DropdownMenuContent>
    <DropdownMenuCheckboxItem
      checked={notifications}
      onCheckedChange={setNotifications}
      onSelect={(e) => e.preventDefault()}
    >
      Notifications
    </DropdownMenuCheckboxItem>
    <DropdownMenuItem onSelect={() => router.push("/profile")}>
      Profile
    </DropdownMenuItem>
  </DropdownMenuContent>
</DropdownMenu>
```

The checkbox item stays open (`preventDefault`), the Profile item
closes (default behavior).

## Example 9 : Select Two-Pair Controlled State

Both `value`+`onValueChange` AND `open`+`onOpenChange` exist on
`Select.Root`. Usually you only control `value` :

```tsx
"use client"
import * as React from "react"
import {
  Select, SelectContent, SelectItem, SelectTrigger, SelectValue,
} from "@/components/ui/select"

export function CountrySelect() {
  const [country, setCountry] = React.useState<string | undefined>()

  return (
    <Select value={country} onValueChange={setCountry}>
      <SelectTrigger className="w-[180px]">
        <SelectValue placeholder="Pick a country" />
      </SelectTrigger>
      <SelectContent>
        <SelectItem value="nl">Netherlands</SelectItem>
        <SelectItem value="be">Belgium</SelectItem>
        <SelectItem value="de">Germany</SelectItem>
      </SelectContent>
    </Select>
  )
}
```

Avoid controlling `open` unless you have a specific reason (e.g.,
opening the popper programmatically when a sibling triggers
validation). Half-binding `value` (passing without `onValueChange`)
locks the Select to the initial value with no recovery path.
