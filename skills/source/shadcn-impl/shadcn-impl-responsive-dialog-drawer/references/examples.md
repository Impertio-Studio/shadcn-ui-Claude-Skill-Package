# Examples: Responsive Dialog + Drawer

All examples assume TypeScript, React 18+, Next.js App Router (or any framework with `"use client"` boundaries), Tailwind CSS, and shadcn ui components already installed via `npx shadcn@latest add dialog drawer popover command button input label`.

## Example 1: SSR-safe `useMediaQuery` hook

Location: `src/hooks/use-media-query.ts`.

```ts
"use client"

import * as React from "react"

export function useMediaQuery(query: string): boolean {
  const subscribe = React.useCallback(
    (callback: () => void) => {
      const mediaQueryList = window.matchMedia(query)
      mediaQueryList.addEventListener("change", callback)
      return () => {
        mediaQueryList.removeEventListener("change", callback)
      }
    },
    [query],
  )

  const getSnapshot = () => window.matchMedia(query).matches

  const getServerSnapshot = () => {
    // useMediaQuery is a client-only hook. The "use client" directive on the
    // consumer plus useSyncExternalStore's server-snapshot slot keep the hook
    // out of the server render path. Throwing here surfaces accidental
    // server invocations early instead of silently returning false.
    throw new Error("useMediaQuery is a client-only hook")
  }

  return React.useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot)
}
```

Why `useSyncExternalStore` and not `useState` + `useEffect`:
- `useState` + `useEffect` initializes with a fallback value (`false`) and updates on mount, producing a one-frame mismatch between server render and first client paint. Next.js logs `Hydration failed because the initial UI does not match what was rendered on the server`.
- `useSyncExternalStore` is the React-blessed primitive for subscribing to external stores; React 18 schedules the subscription correctly across concurrent renders.

## Example 2: Shared `ProfileForm` content component

```tsx
"use client"

import * as React from "react"
import { cn } from "@/lib/utils"
import { Button } from "@/components/ui/button"
import { Input } from "@/components/ui/input"
import { Label } from "@/components/ui/label"

type ProfileFormProps = React.ComponentProps<"form"> & {
  onDone?: () => void
}

export function ProfileForm({ className, onDone, ...rest }: ProfileFormProps) {
  return (
    <form
      {...rest}
      className={cn("grid items-start gap-6", className)}
      onSubmit={(e) => {
        e.preventDefault()
        // ...persist the data...
        onDone?.()
      }}
    >
      <div className="grid gap-3">
        <Label htmlFor="email">Email</Label>
        <Input type="email" id="email" defaultValue="shadcn@example.com" />
      </div>
      <div className="grid gap-3">
        <Label htmlFor="username">Username</Label>
        <Input id="username" defaultValue="@shadcn" />
      </div>
      <Button type="submit">Save changes</Button>
    </form>
  )
}
```

The `cn("grid items-start gap-6", className)` merge is critical: the Dialog branch passes no `className`, so the form keeps its default spacing; the Drawer branch passes `className="px-4"`, which `cn` (from `tailwind-merge`) appends without conflicting with the grid.

## Example 3: `ResponsiveModal` orchestrator

Location: `src/components/responsive-modal.tsx`.

```tsx
"use client"

import * as React from "react"
import { useMediaQuery } from "@/hooks/use-media-query"
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogHeader,
  DialogTitle,
  DialogTrigger,
} from "@/components/ui/dialog"
import {
  Drawer,
  DrawerContent,
  DrawerDescription,
  DrawerHeader,
  DrawerTitle,
  DrawerTrigger,
} from "@/components/ui/drawer"

const DEFAULT_BREAKPOINT = "(min-width: 768px)"

type ResponsiveModalProps = {
  open: boolean
  onOpenChange: (open: boolean) => void
  title: string
  description?: string
  trigger?: React.ReactNode
  children: React.ReactNode
  breakpoint?: string
}

export function ResponsiveModal({
  open,
  onOpenChange,
  title,
  description,
  trigger,
  children,
  breakpoint = DEFAULT_BREAKPOINT,
}: ResponsiveModalProps) {
  const isDesktop = useMediaQuery(breakpoint)

  if (isDesktop) {
    return (
      <Dialog open={open} onOpenChange={onOpenChange}>
        {trigger ? <DialogTrigger asChild>{trigger}</DialogTrigger> : null}
        <DialogContent className="sm:max-w-[425px]">
          <DialogHeader>
            <DialogTitle>{title}</DialogTitle>
            {description ? (
              <DialogDescription>{description}</DialogDescription>
            ) : null}
          </DialogHeader>
          {children}
        </DialogContent>
      </Dialog>
    )
  }

  return (
    <Drawer open={open} onOpenChange={onOpenChange}>
      {trigger ? <DrawerTrigger asChild>{trigger}</DrawerTrigger> : null}
      <DrawerContent>
        <DrawerHeader className="text-left">
          <DrawerTitle>{title}</DrawerTitle>
          {description ? (
            <DrawerDescription>{description}</DrawerDescription>
          ) : null}
        </DrawerHeader>
        {children}
      </DrawerContent>
    </Drawer>
  )
}
```

## Example 4: Edit-profile feature using `ResponsiveModal`

```tsx
"use client"

import * as React from "react"
import { Button } from "@/components/ui/button"
import { ResponsiveModal } from "@/components/responsive-modal"
import { ProfileForm } from "@/components/profile-form"

export function EditProfilePanel() {
  const [open, setOpen] = React.useState(false)
  return (
    <ResponsiveModal
      open={open}
      onOpenChange={setOpen}
      title="Edit profile"
      description="Make changes to your profile here. Click save when done."
      trigger={<Button variant="outline">Edit Profile</Button>}
    >
      <ProfileForm
        className="px-4 md:px-0"
        onDone={() => setOpen(false)}
      />
    </ResponsiveModal>
  )
}
```

Note the `className="px-4 md:px-0"` pattern: `px-4` is active on the Drawer (mobile), `md:px-0` overrides it to remove padding on the Dialog (desktop) because `DialogContent` already pads. This single Tailwind expression replaces the official example's runtime branching of the `className` prop.

## Example 5: Full delete-confirmation responsive flow

A destructive action benefits from the responsive pattern because the consequences of an accidental confirm are higher on a small screen. The Drawer branch shows a footer with two distinct buttons; the Dialog branch reuses the same pattern.

```tsx
"use client"

import * as React from "react"
import { Button } from "@/components/ui/button"
import { useMediaQuery } from "@/hooks/use-media-query"
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogFooter,
  DialogHeader,
  DialogTitle,
  DialogTrigger,
} from "@/components/ui/dialog"
import {
  Drawer,
  DrawerClose,
  DrawerContent,
  DrawerDescription,
  DrawerFooter,
  DrawerHeader,
  DrawerTitle,
  DrawerTrigger,
} from "@/components/ui/drawer"

export function DeleteAccountButton({ onConfirm }: { onConfirm: () => Promise<void> }) {
  const [open, setOpen] = React.useState(false)
  const [busy, setBusy] = React.useState(false)
  const isDesktop = useMediaQuery("(min-width: 768px)")

  async function handleConfirm() {
    setBusy(true)
    try {
      await onConfirm()
      setOpen(false)
    } finally {
      setBusy(false)
    }
  }

  const Title = "Delete account"
  const Body =
    "This action cannot be undone. Your profile, posts, and saved drafts will be permanently removed."

  if (isDesktop) {
    return (
      <Dialog open={open} onOpenChange={setOpen}>
        <DialogTrigger asChild>
          <Button variant="destructive">Delete account</Button>
        </DialogTrigger>
        <DialogContent className="sm:max-w-[425px]">
          <DialogHeader>
            <DialogTitle>{Title}</DialogTitle>
            <DialogDescription>{Body}</DialogDescription>
          </DialogHeader>
          <DialogFooter>
            <Button variant="outline" onClick={() => setOpen(false)} disabled={busy}>
              Cancel
            </Button>
            <Button variant="destructive" onClick={handleConfirm} disabled={busy}>
              {busy ? "Deleting..." : "Delete account"}
            </Button>
          </DialogFooter>
        </DialogContent>
      </Dialog>
    )
  }

  return (
    <Drawer open={open} onOpenChange={setOpen}>
      <DrawerTrigger asChild>
        <Button variant="destructive">Delete account</Button>
      </DrawerTrigger>
      <DrawerContent>
        <DrawerHeader className="text-left">
          <DrawerTitle>{Title}</DrawerTitle>
          <DrawerDescription>{Body}</DrawerDescription>
        </DrawerHeader>
        <DrawerFooter className="pt-2">
          <Button variant="destructive" onClick={handleConfirm} disabled={busy}>
            {busy ? "Deleting..." : "Delete account"}
          </Button>
          <DrawerClose asChild>
            <Button variant="outline" disabled={busy}>
              Cancel
            </Button>
          </DrawerClose>
        </DrawerFooter>
      </DrawerContent>
    </Drawer>
  )
}
```

Notable details:
- `Title` and `Body` are extracted to local constants so they cannot drift between branches.
- The Drawer puts the destructive button ABOVE the cancel button (standard iOS / Android action-sheet ordering). The Dialog puts cancel on the LEFT and destructive on the RIGHT (standard desktop dialog ordering). This intentional asymmetry follows platform conventions.
- `busy` state is shared across both surfaces, identical loading text.

## Example 6: Responsive Combobox (Popover-on-desktop / Drawer-on-mobile)

The Combobox composition (Popover + Command) breaks on phones because the Popover anchors to the trigger and the on-screen keyboard covers it. The fix: same `Command` body, switch the wrapper from `Popover` to `Drawer`.

```tsx
"use client"

import * as React from "react"
import { Check, ChevronsUpDown } from "lucide-react"
import { cn } from "@/lib/utils"
import { Button } from "@/components/ui/button"
import {
  Command,
  CommandEmpty,
  CommandGroup,
  CommandInput,
  CommandItem,
  CommandList,
} from "@/components/ui/command"
import { Popover, PopoverContent, PopoverTrigger } from "@/components/ui/popover"
import { Drawer, DrawerContent, DrawerTrigger } from "@/components/ui/drawer"
import { useMediaQuery } from "@/hooks/use-media-query"

type Status = { value: string; label: string }

const statuses: Status[] = [
  { value: "backlog", label: "Backlog" },
  { value: "todo", label: "Todo" },
  { value: "in progress", label: "In progress" },
  { value: "done", label: "Done" },
  { value: "canceled", label: "Canceled" },
]

export function StatusCombobox() {
  const [open, setOpen] = React.useState(false)
  const [selected, setSelected] = React.useState<Status | null>(null)
  const isDesktop = useMediaQuery("(min-width: 768px)")

  const Trigger = (
    <Button variant="outline" className="w-[200px] justify-between">
      {selected ? selected.label : "Set status"}
      <ChevronsUpDown className="opacity-50" />
    </Button>
  )

  const Body = (
    <Command>
      <CommandInput placeholder="Filter status..." />
      <CommandList>
        <CommandEmpty>No results found.</CommandEmpty>
        <CommandGroup>
          {statuses.map((s) => (
            <CommandItem
              key={s.value}
              value={s.value}
              onSelect={(v) => {
                setSelected(statuses.find((x) => x.value === v) ?? null)
                setOpen(false)
              }}
            >
              <Check
                className={cn(
                  "mr-2 h-4 w-4",
                  selected?.value === s.value ? "opacity-100" : "opacity-0",
                )}
              />
              {s.label}
            </CommandItem>
          ))}
        </CommandGroup>
      </CommandList>
    </Command>
  )

  if (isDesktop) {
    return (
      <Popover open={open} onOpenChange={setOpen}>
        <PopoverTrigger asChild>{Trigger}</PopoverTrigger>
        <PopoverContent className="w-[200px] p-0" align="start">
          {Body}
        </PopoverContent>
      </Popover>
    )
  }

  return (
    <Drawer open={open} onOpenChange={setOpen}>
      <DrawerTrigger asChild>{Trigger}</DrawerTrigger>
      <DrawerContent>
        <div className="mt-4 border-t">{Body}</div>
      </DrawerContent>
    </Drawer>
  )
}
```

The key reuse trick: `Body` is a JSX expression, NOT a component. Extracting it as a constant inside the render keeps `Command`'s internal state (highlighted item, filter input value) bound to a single instance per render; if it were extracted as a separate component called twice with different parents, the two parents would each get a fresh instance.

The Drawer branch wraps `Body` in `<div className="mt-4 border-t">` because the bare `Command` looks unfinished without a divider above the search input on a bottom sheet. The Popover branch does not need this because `PopoverContent`'s border provides the visual frame.
