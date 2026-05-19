# Command : Canonical Examples

Five working recipes. Every snippet is verified against `apps/v4/registry/new-york-v4/ui/command.tsx`, https://ui.shadcn.com/docs/components/radix/command, and https://cmdk.paco.me (2026-05-19).

All examples assume :
- shadcn ui evergreen-2026 (registry style `new-york-v4`)
- Tailwind v4 (no `tailwind.config.js` ; tokens via `@theme inline`)
- React 19 (no explicit `forwardRef` needed)
- The command file `components/ui/command.tsx` has `"use client"` at the top.
- The Dialog file `components/ui/dialog.tsx` exists (CommandDialog depends on it).

## Example 1 : Cmd+K command palette (the canonical recipe)

The global command palette. Press Cmd+K (macOS) or Ctrl+K (Windows / Linux) to open. Esc or outside-click closes.

```tsx
"use client"

import { useEffect, useState } from "react"
import {
  CommandDialog,
  CommandInput,
  CommandList,
  CommandEmpty,
  CommandGroup,
  CommandItem,
  CommandShortcut,
  CommandSeparator,
} from "@/components/ui/command"

export function CommandMenu() {
  const [open, setOpen] = useState(false)

  useEffect(() => {
    function onKeyDown(e: KeyboardEvent) {
      if (e.key === "k" && (e.metaKey || e.ctrlKey)) {
        e.preventDefault()
        setOpen((current) => !current)
      }
    }
    document.addEventListener("keydown", onKeyDown)
    return () => document.removeEventListener("keydown", onKeyDown)
  }, [])

  function runCommand(callback: () => void) {
    setOpen(false)
    callback()
  }

  return (
    <CommandDialog open={open} onOpenChange={setOpen}>
      <CommandInput placeholder="Type a command or search..." />
      <CommandList>
        <CommandEmpty>No results found.</CommandEmpty>
        <CommandGroup heading="Suggestions">
          <CommandItem
            value="calendar"
            onSelect={() => runCommand(() => console.log("open calendar"))}
          >
            Calendar
          </CommandItem>
          <CommandItem
            value="search-emoji"
            keywords={["smiley", "sticker", "icon"]}
            onSelect={() => runCommand(() => console.log("open emoji picker"))}
          >
            Search Emoji
          </CommandItem>
        </CommandGroup>
        <CommandSeparator />
        <CommandGroup heading="Settings">
          <CommandItem
            value="profile"
            onSelect={() => runCommand(() => console.log("open profile"))}
          >
            Profile
            <CommandShortcut>Ctrl+P</CommandShortcut>
          </CommandItem>
          <CommandItem
            value="billing"
            onSelect={() => runCommand(() => console.log("open billing"))}
          >
            Billing
            <CommandShortcut>Ctrl+B</CommandShortcut>
          </CommandItem>
        </CommandGroup>
      </CommandList>
    </CommandDialog>
  )
}
```

Notes :
- The `useEffect` listener checks BOTH `e.metaKey` (macOS Cmd) AND `e.ctrlKey` (Windows / Linux Ctrl) so the palette responds on every platform.
- `e.preventDefault()` runs BEFORE the state toggle ; Chrome maps Cmd+K to the location-bar suggestions panel and would otherwise swallow the key.
- The cleanup `return () => document.removeEventListener(...)` is mandatory ; without it the listener leaks on every remount and the toggles compound.
- Every CommandItem has a stable, unique `value`. `runCommand` always closes the dialog BEFORE invoking the action so the user sees the close animation while the route changes.
- The `keywords` array on the Emoji item lets the user type "smiley" and still match.
- CommandShortcut is a pure span ; it does NOT bind the actual shortcut, it only renders the hint pill. Bind the real shortcut in your app's keymap.

## Example 2 : Async server-side search with CommandLoading

The user types in the input ; debounced fetch hits `/api/search` ; results render as they arrive. `shouldFilter={false}` disables cmdk's built-in filter because the server has already ranked the results.

```tsx
"use client"

import { useEffect, useState } from "react"
import { Command as CommandPrimitive } from "cmdk"
import {
  Command,
  CommandInput,
  CommandList,
  CommandEmpty,
  CommandItem,
} from "@/components/ui/command"

const CommandLoading = CommandPrimitive.Loading

type Result = { id: string; label: string; url: string }

export function ServerSearchCommand() {
  const [query, setQuery] = useState("")
  const [results, setResults] = useState<Result[]>([])
  const [loading, setLoading] = useState(false)

  useEffect(() => {
    if (!query) {
      setResults([])
      setLoading(false)
      return
    }

    const controller = new AbortController()
    const timer = setTimeout(async () => {
      setLoading(true)
      try {
        const res = await fetch(
          `/api/search?q=${encodeURIComponent(query)}`,
          { signal: controller.signal }
        )
        const data = (await res.json()) as Result[]
        setResults(data)
      } catch (err) {
        // AbortError when the user keeps typing : ignore.
      } finally {
        setLoading(false)
      }
    }, 250)

    return () => {
      controller.abort()
      clearTimeout(timer)
    }
  }, [query])

  return (
    <Command shouldFilter={false} className="rounded-lg border shadow-md">
      <CommandInput
        value={query}
        onValueChange={setQuery}
        placeholder="Search the docs..."
      />
      <CommandList>
        {loading && <CommandLoading>Searching…</CommandLoading>}
        {!loading && query && results.length === 0 && (
          <CommandEmpty>No results for "{query}".</CommandEmpty>
        )}
        {results.map((r) => (
          <CommandItem
            key={r.id}
            value={r.id}
            onSelect={() => (window.location.href = r.url)}
          >
            {r.label}
          </CommandItem>
        ))}
      </CommandList>
    </Command>
  )
}
```

Notes :
- `shouldFilter={false}` is REQUIRED. Without it cmdk would try to re-filter the server response against the local input, producing zero matches whenever the server's ranking differs from cmdk's fuzzy score.
- `value={query}` + `onValueChange={setQuery}` on CommandInput makes the search controlled, so the effect can react to it.
- `CommandLoading` is imported from cmdk directly ; shadcn's `command.tsx` does NOT re-export it.
- The debounce timer cancels via `clearTimeout` AND the in-flight fetch cancels via `controller.abort()` on every keystroke change ; both cleanups are required to avoid stale results overwriting newer ones.
- CommandEmpty is gated on `!loading && query && results.length === 0` because with `shouldFilter={false}` cmdk does not auto-decide when to render it.
- The `key` on CommandItem is `r.id` (stable across re-renders), the `value` is `r.id` too so cmdk has a unique identity per row.

## Example 3 : Palette with groups, separators, icons, and shortcuts

A more visually rich palette with grouped sections, dividers, and icon-prefixed items.

```tsx
"use client"

import {
  Command,
  CommandInput,
  CommandList,
  CommandEmpty,
  CommandGroup,
  CommandItem,
  CommandSeparator,
  CommandShortcut,
} from "@/components/ui/command"
import {
  CalendarIcon, SmileIcon, CalculatorIcon,
  UserIcon, CreditCardIcon, SettingsIcon,
} from "lucide-react"

export function RichPalette() {
  return (
    <Command className="rounded-lg border shadow-md md:min-w-[450px]">
      <CommandInput placeholder="Type a command or search..." />
      <CommandList>
        <CommandEmpty>No results found.</CommandEmpty>
        <CommandGroup heading="Suggestions">
          <CommandItem value="calendar">
            <CalendarIcon />
            <span>Calendar</span>
          </CommandItem>
          <CommandItem value="search-emoji">
            <SmileIcon />
            <span>Search Emoji</span>
          </CommandItem>
          <CommandItem value="calculator" disabled>
            <CalculatorIcon />
            <span>Calculator</span>
          </CommandItem>
        </CommandGroup>
        <CommandSeparator />
        <CommandGroup heading="Settings">
          <CommandItem value="profile">
            <UserIcon />
            <span>Profile</span>
            <CommandShortcut>Ctrl+P</CommandShortcut>
          </CommandItem>
          <CommandItem value="billing">
            <CreditCardIcon />
            <span>Billing</span>
            <CommandShortcut>Ctrl+B</CommandShortcut>
          </CommandItem>
          <CommandItem value="settings">
            <SettingsIcon />
            <span>Settings</span>
            <CommandShortcut>Ctrl+S</CommandShortcut>
          </CommandItem>
        </CommandGroup>
      </CommandList>
    </Command>
  )
}
```

Notes :
- Icons inside a CommandItem are auto-styled by shadcn's `[&_svg]` selectors (size-4, muted-foreground colour). NO extra className needed on the icon.
- `disabled` on Calculator removes it from keyboard nav and dims it via `data-[disabled=true]:opacity-50`.
- CommandSeparator is hidden automatically while the user types (cmdk applies the `hidden` attribute) ; pass `alwaysRender` to override.
- Wrap text in `<span>` so the icon + label + shortcut layout (`flex items-center gap-2 ... ml-auto`) sorts naturally.

## Example 4 : Command inside Popover (the Combobox recipe)

The shadcn Combobox is documented as "Popover + Command". Use it when the palette is anchored to a specific trigger button (form fields, filter dropdowns, multi-select inputs).

```tsx
"use client"

import { useState } from "react"
import { Check, ChevronsUpDown } from "lucide-react"
import { cn } from "@/lib/utils"
import { Button } from "@/components/ui/button"
import {
  Command,
  CommandInput,
  CommandList,
  CommandEmpty,
  CommandGroup,
  CommandItem,
} from "@/components/ui/command"
import {
  Popover,
  PopoverContent,
  PopoverTrigger,
} from "@/components/ui/popover"

const frameworks = [
  { value: "next", label: "Next.js" },
  { value: "vite", label: "Vite" },
  { value: "remix", label: "Remix" },
  { value: "astro", label: "Astro" },
]

export function FrameworkCombobox() {
  const [open, setOpen] = useState(false)
  const [value, setValue] = useState("")

  return (
    <Popover open={open} onOpenChange={setOpen}>
      <PopoverTrigger asChild>
        <Button
          variant="outline"
          role="combobox"
          aria-expanded={open}
          className="w-[200px] justify-between"
        >
          {value
            ? frameworks.find((f) => f.value === value)?.label
            : "Select framework..."}
          <ChevronsUpDown className="ml-2 size-4 shrink-0 opacity-50" />
        </Button>
      </PopoverTrigger>
      <PopoverContent className="w-[200px] p-0">
        <Command>
          <CommandInput placeholder="Search framework..." />
          <CommandList>
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
- `PopoverContent` gets `p-0` so the Command surface owns the padding ; without it the rounded corners clip awkwardly.
- The `role="combobox"` + `aria-expanded` on the Button satisfies the combobox ARIA pattern that the Button itself does not auto-wire.
- `onSelect` receives the TRIMMED, LOWERCASED `value` ("next", not "Next.js"). Match against your data using the same casing or transform on read.
- `setOpen(false)` inside `onSelect` closes the popover after pick ; the user expects this in a combobox UX.
- The Check icon is opacity-toggled so the column width stays stable (no layout shift between selected / unselected rows).

## Example 5 : Nested / multi-page command palette

Selecting "Search projects..." replaces the visible items with a sub-route ("project A", "project B"). Backspace on an empty search OR Esc pops back to the parent page. This recipe comes verbatim from the cmdk README.

```tsx
"use client"

import { useState } from "react"
import {
  Command,
  CommandInput,
  CommandList,
  CommandItem,
} from "@/components/ui/command"

type Page = "root" | "projects" | "teams"

export function MultiPagePalette() {
  const [search, setSearch] = useState("")
  const [pages, setPages] = useState<Page[]>(["root"])
  const currentPage = pages[pages.length - 1]

  return (
    <Command
      className="rounded-lg border shadow-md max-w-md"
      onKeyDown={(e) => {
        // Esc OR Backspace-on-empty-search pops the stack.
        if (e.key === "Escape" || (e.key === "Backspace" && !search)) {
          if (pages.length > 1) {
            e.preventDefault()
            setPages((current) => current.slice(0, -1))
          }
        }
      }}
    >
      <CommandInput
        value={search}
        onValueChange={setSearch}
        placeholder={
          currentPage === "root"
            ? "Type a command..."
            : `Searching ${currentPage}...`
        }
      />
      <CommandList>
        {currentPage === "root" && (
          <>
            <CommandItem
              value="search-projects"
              onSelect={() => {
                setSearch("")
                setPages((current) => [...current, "projects"])
              }}
            >
              Search projects…
            </CommandItem>
            <CommandItem
              value="join-team"
              onSelect={() => {
                setSearch("")
                setPages((current) => [...current, "teams"])
              }}
            >
              Join a team…
            </CommandItem>
          </>
        )}

        {currentPage === "projects" && (
          <>
            <CommandItem value="project-a">Project A</CommandItem>
            <CommandItem value="project-b">Project B</CommandItem>
          </>
        )}

        {currentPage === "teams" && (
          <>
            <CommandItem value="team-1">Team 1</CommandItem>
            <CommandItem value="team-2">Team 2</CommandItem>
          </>
        )}
      </CommandList>
    </Command>
  )
}
```

Notes :
- The `pages` state is a STACK. Push to go deeper, pop to go back.
- `onKeyDown` on the Command root catches Esc + Backspace-on-empty-search. cmdk does not provide a built-in "back" key ; this is the canonical workaround per the cmdk README.
- `e.preventDefault()` in the Esc branch prevents cmdk from also dismissing the highlight ; the parent-page pop is the only action that should happen.
- `setSearch("")` runs on every page transition so the new page does not inherit a stale filter from the previous one.
- For a modal version, wrap the same body in `<CommandDialog open onOpenChange>` and gate the pop on `pages.length > 1` versus `pages.length === 1` (close the dialog at the root level).
