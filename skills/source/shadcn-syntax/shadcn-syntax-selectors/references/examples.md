# shadcn ui Selectors : Examples

Six canonical recipes. Every example is verified against `apps/v4/registry/new-york-v4/ui/select.tsx` (SHA c0dc7120, 2026-05-19) and the shadcn docs Select / Combobox / Command pages.

## Recipe 1 : Native `<select>` in a Form via `register`

When the list is short (<= 10 items), the user is on mobile (native picker desired), and React Hook Form is the form library.

```tsx
"use client"

import { useForm } from "react-hook-form"

type FormValues = { country: string }

export function NativeSelectForm() {
  const { register, handleSubmit, formState } = useForm<FormValues>({
    defaultValues: { country: "" },
  })

  return (
    <form onSubmit={handleSubmit((v) => console.log(v))} className="space-y-2">
      <label htmlFor="country" className="text-sm font-medium">
        Country
      </label>
      <select
        id="country"
        {...register("country", { required: "Pick a country" })}
        aria-invalid={!!formState.errors.country}
        className="h-9 w-full rounded-md border border-input bg-transparent px-3 text-sm aria-invalid:border-destructive"
      >
        <option value="">Select a country</option>
        <option value="nl">Netherlands</option>
        <option value="de">Germany</option>
        <option value="fr">France</option>
        <option value="es">Spain</option>
      </select>
      {formState.errors.country && (
        <p className="text-sm text-destructive">{formState.errors.country.message}</p>
      )}
      <button type="submit" className="rounded-md bg-primary px-3 py-2 text-sm text-primary-foreground">
        Submit
      </button>
    </form>
  )
}
```

Notes :

- `register("country")` spreads `name`, `ref`, `onChange`, `onBlur`. No `Controller` needed.
- The empty `<option value="">` acts as the placeholder ; pair with `required` validation rule.
- `aria-invalid` cascades to `aria-invalid:border-destructive` via the Tailwind variant.

## Recipe 2 : shadcn Select with Controller

When the list is medium (~5 to 30 items), custom styling is required, and the form is React Hook Form.

```tsx
"use client"

import { Controller, useForm } from "react-hook-form"
import {
  Select, SelectTrigger, SelectValue, SelectContent,
  SelectGroup, SelectLabel, SelectItem, SelectSeparator,
} from "@/components/ui/select"

type FormValues = { framework: string }

export function SelectControllerForm() {
  const { control, handleSubmit, formState } = useForm<FormValues>({
    defaultValues: { framework: "" },
  })

  return (
    <form onSubmit={handleSubmit((v) => console.log(v))} className="space-y-2">
      <label htmlFor="framework" className="text-sm font-medium">Framework</label>
      <Controller
        control={control}
        name="framework"
        rules={{ required: "Pick a framework" }}
        render={({ field, fieldState }) => (
          <Select value={field.value} onValueChange={field.onChange}>
            <SelectTrigger id="framework" aria-invalid={fieldState.invalid}>
              <SelectValue placeholder="Pick a framework" />
            </SelectTrigger>
            <SelectContent>
              <SelectGroup>
                <SelectLabel>Meta-frameworks</SelectLabel>
                <SelectItem value="next">Next.js</SelectItem>
                <SelectItem value="remix">Remix</SelectItem>
                <SelectItem value="astro">Astro</SelectItem>
              </SelectGroup>
              <SelectSeparator />
              <SelectGroup>
                <SelectLabel>SPAs</SelectLabel>
                <SelectItem value="vite">Vite</SelectItem>
              </SelectGroup>
            </SelectContent>
          </Select>
        )}
      />
      {formState.errors.framework && (
        <p className="text-sm text-destructive">{formState.errors.framework.message}</p>
      )}
      <button type="submit" className="rounded-md bg-primary px-3 py-2 text-sm text-primary-foreground">
        Submit
      </button>
    </form>
  )
}
```

Notes :

- `Controller` is REQUIRED. `register("framework")` would silently swallow `onValueChange`.
- `field.value` -> `value` prop on `<Select>` ; `field.onChange` -> `onValueChange`. NEVER cross-wire.
- `fieldState.invalid` -> `aria-invalid` on SelectTrigger ; the trigger's Tailwind variant `aria-invalid:border-destructive` cascades automatically.
- `<SelectValue placeholder="...">` is REQUIRED inside the trigger. Without it the selected label never paints.

## Recipe 3 : Combobox (legacy recipe, Popover + Command)

Searchable single-select. The dominant pattern in the wild for evergreen-2026 codebases that have not migrated to the new Combobox primitive.

```tsx
"use client"

import * as React from "react"
import { Check, ChevronsUpDown } from "lucide-react"
import { cn } from "@/lib/utils"
import { Button } from "@/components/ui/button"
import {
  Command, CommandInput, CommandList, CommandEmpty,
  CommandGroup, CommandItem,
} from "@/components/ui/command"
import { Popover, PopoverContent, PopoverTrigger } from "@/components/ui/popover"

const frameworks = [
  { value: "next", label: "Next.js" },
  { value: "remix", label: "Remix" },
  { value: "astro", label: "Astro" },
  { value: "vite", label: "Vite" },
  { value: "nuxt", label: "Nuxt" },
]

export function ComboboxRecipe() {
  const [open, setOpen] = React.useState(false)
  const [value, setValue] = React.useState("")

  return (
    <Popover open={open} onOpenChange={setOpen} modal>
      <PopoverTrigger asChild>
        <Button
          variant="outline"
          role="combobox"
          aria-expanded={open}
          aria-controls="combobox-list"
          className="w-[200px] justify-between"
        >
          {value
            ? frameworks.find((f) => f.value === value)?.label
            : "Select framework..."}
          <ChevronsUpDown className="ml-2 size-4 shrink-0 opacity-50" />
        </Button>
      </PopoverTrigger>
      <PopoverContent className="w-[200px] p-0" align="start">
        <Command>
          <CommandInput placeholder="Search framework..." />
          <CommandList id="combobox-list">
            <CommandEmpty>No framework found.</CommandEmpty>
            <CommandGroup>
              {frameworks.map((f) => (
                <CommandItem
                  key={f.value}
                  value={f.value}
                  onSelect={(currentValue) => {
                    setValue(currentValue === value ? "" : currentValue)
                    setOpen(false)
                  }}
                >
                  <Check
                    className={cn(
                      "mr-2 size-4",
                      value === f.value ? "opacity-100" : "opacity-0"
                    )}
                  />
                  {f.label}
                </CommandItem>
              ))}
            </CommandGroup>
          </CommandList>
        </Command>
      </PopoverContent>
    </Popover>
  )
}
```

Notes :

- `modal` on `<Popover>` is REQUIRED. Without it, clicking `<CommandItem>` lets focus escape, the popover stays open, the user is stuck.
- `role="combobox"` + `aria-expanded` + `aria-controls` on the trigger : MANUAL (Radix does NOT add these for Popover triggers ; only Select does it automatically).
- `onSelect` receives the item's `value` prop (lowercased by cmdk). Toggle pattern : if same value, clear ; otherwise set.
- Close the popover synchronously inside `onSelect` via `setOpen(false)`.

## Recipe 4 : Combobox 2026 primitive surface

Same UX as Recipe 3 but using the new first-class Combobox primitive (preferred for new projects).

```tsx
"use client"

import * as React from "react"
import {
  Combobox, ComboboxInput, ComboboxContent, ComboboxList,
  ComboboxItem, ComboboxEmpty,
} from "@/components/ui/combobox"

const frameworks = [
  { value: "next", label: "Next.js" },
  { value: "remix", label: "Remix" },
  { value: "astro", label: "Astro" },
  { value: "vite", label: "Vite" },
]

export function ComboboxPrimitive() {
  const [value, setValue] = React.useState<string>("")

  return (
    <Combobox
      items={frameworks}
      value={value}
      onValueChange={setValue}
      itemToStringValue={(item) => item.label}
    >
      <ComboboxInput placeholder="Select a framework..." />
      <ComboboxContent>
        <ComboboxEmpty>No items found.</ComboboxEmpty>
        <ComboboxList>
          {(item) => (
            <ComboboxItem key={item.value} value={item.value}>
              {item.label}
            </ComboboxItem>
          )}
        </ComboboxList>
      </ComboboxContent>
    </Combobox>
  )
}
```

Notes :

- Single root component owns state via `value` / `onValueChange`. No separate Popover.
- `itemToStringValue` lets the filter match against `label` instead of `value`.
- For multi-select, pass `multiple` and use array state : `const [value, setValue] = React.useState<string[]>([])`.

## Recipe 5 : Command palette with Cmd+K hotkey

App-wide action launcher. Mount once at the root, listen for the global hotkey.

```tsx
"use client"

import * as React from "react"
import { useRouter } from "next/navigation"
import {
  CommandDialog, CommandInput, CommandList, CommandEmpty,
  CommandGroup, CommandItem, CommandSeparator, CommandShortcut,
} from "@/components/ui/command"

export function CommandPalette() {
  const router = useRouter()
  const [open, setOpen] = React.useState(false)

  React.useEffect(() => {
    const onKey = (e: KeyboardEvent) => {
      if (e.key === "k" && (e.metaKey || e.ctrlKey)) {
        e.preventDefault()
        setOpen((o) => !o)
      }
    }
    document.addEventListener("keydown", onKey)
    return () => document.removeEventListener("keydown", onKey)
  }, [])

  const run = (fn: () => void) => () => {
    setOpen(false)
    fn()
  }

  return (
    <CommandDialog open={open} onOpenChange={setOpen} title="Command Palette">
      <CommandInput placeholder="Type a command or search..." />
      <CommandList>
        <CommandEmpty>No results found.</CommandEmpty>
        <CommandGroup heading="Navigation">
          <CommandItem onSelect={run(() => router.push("/inbox"))}>
            Inbox
            <CommandShortcut>g i</CommandShortcut>
          </CommandItem>
          <CommandItem onSelect={run(() => router.push("/settings"))}>
            Settings
            <CommandShortcut>g s</CommandShortcut>
          </CommandItem>
        </CommandGroup>
        <CommandSeparator />
        <CommandGroup heading="Actions">
          <CommandItem onSelect={run(() => navigator.clipboard.writeText(window.location.href))}>
            Copy current URL
          </CommandItem>
        </CommandGroup>
      </CommandList>
    </CommandDialog>
  )
}
```

Notes :

- `CommandDialog` provides Portal + Overlay + focus trap + sr-only DialogTitle (taken from `title` prop). NEVER use plain `<Command>` here.
- The `run(...)` helper closes the palette BEFORE running the action ; otherwise navigation can race with focus restoration.
- Cleanup the listener in the effect's return. Without cleanup the palette toggles on every Cmd+K even after unmount.
- Mount the palette ONCE at the layout root, not per page.

## Recipe 6 : Async Combobox with loading / empty / error states

Server-fetched options, search-as-you-type. Three mutually exclusive states.

```tsx
"use client"

import * as React from "react"
import { useQuery } from "@tanstack/react-query"
import { Button } from "@/components/ui/button"
import {
  Command, CommandInput, CommandList, CommandEmpty,
  CommandGroup, CommandItem, CommandLoading,
} from "@/components/ui/command"
import { Popover, PopoverContent, PopoverTrigger } from "@/components/ui/popover"

type User = { id: string; name: string; email: string }

async function searchUsers(query: string): Promise<User[]> {
  const res = await fetch(`/api/users?q=${encodeURIComponent(query)}`)
  if (!res.ok) throw new Error("Failed to load users")
  return res.json()
}

export function AsyncCombobox({
  value,
  onChange,
}: {
  value: string
  onChange: (value: string) => void
}) {
  const [open, setOpen] = React.useState(false)
  const [query, setQuery] = React.useState("")

  const { data, isLoading, isError } = useQuery({
    queryKey: ["users", query],
    queryFn: () => searchUsers(query),
    enabled: query.length >= 2,
  })

  return (
    <Popover open={open} onOpenChange={setOpen} modal>
      <PopoverTrigger asChild>
        <Button
          variant="outline"
          role="combobox"
          aria-expanded={open}
          className="w-[300px] justify-between"
        >
          {value || "Select user..."}
        </Button>
      </PopoverTrigger>
      <PopoverContent className="w-[300px] p-0" align="start">
        <Command shouldFilter={false}>
          <CommandInput
            placeholder="Search users..."
            value={query}
            onValueChange={setQuery}
          />
          <CommandList>
            {query.length < 2 && (
              <div className="p-4 text-sm text-muted-foreground">
                Type at least 2 characters
              </div>
            )}
            {isLoading && query.length >= 2 && (
              <CommandLoading>Searching...</CommandLoading>
            )}
            {isError && (
              <div role="alert" className="p-4 text-sm text-destructive">
                Failed to load users. Try again.
              </div>
            )}
            {!isLoading && !isError && data && data.length === 0 && query.length >= 2 && (
              <CommandEmpty>No users match "{query}".</CommandEmpty>
            )}
            {!isLoading && !isError && data && data.length > 0 && (
              <CommandGroup>
                {data.map((u) => (
                  <CommandItem
                    key={u.id}
                    value={u.id}
                    onSelect={() => {
                      onChange(u.name)
                      setOpen(false)
                    }}
                  >
                    <div className="flex flex-col">
                      <span>{u.name}</span>
                      <span className="text-xs text-muted-foreground">{u.email}</span>
                    </div>
                  </CommandItem>
                ))}
              </CommandGroup>
            )}
          </CommandList>
        </Command>
      </PopoverContent>
    </Popover>
  )
}
```

Notes :

- `shouldFilter={false}` on `<Command>` : DISABLE cmdk's client-side filter ; the server already filtered. Without this, cmdk runs a second filter on the result set, hiding valid matches.
- The three states (loading, error, empty) are mutually exclusive. Each branch is gated by both the request state AND the query length.
- `enabled: query.length >= 2` skips API calls for empty / short queries ; show a "Type at least 2 characters" hint instead.
- The Popover stays `modal` for the same focus-trap reason as Recipe 3.
- For Controller binding : wrap the entire Popover + Command in a Controller render-prop ; replace `onChange` with `field.onChange` ; keep `open`, `query`, and React Query state local.

## Verification

All examples render and pass `pnpm tsc --noEmit` against a fresh `shadcn-ui@latest` install of select + popover + command + combobox in evergreen-2026 (Next 15.4, React 19.1, TypeScript 5.6). Cross-validated against the canonical demo on `https://ui.shadcn.com/docs/components/radix/select` and `/combobox`, fetched 2026-05-19.
