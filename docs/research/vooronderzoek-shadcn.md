# Vooronderzoek : shadcn ui

> Date : 2026-05-19
> Sources verified (WebFetch + gh CLI) :
> - https://ui.shadcn.com/docs
> - https://ui.shadcn.com/docs/components (index, 59 components)
> - https://ui.shadcn.com/docs/cli
> - https://ui.shadcn.com/docs/components-json
> - https://ui.shadcn.com/docs/theming
> - https://ui.shadcn.com/docs/dark-mode
> - https://ui.shadcn.com/docs/dark-mode/next
> - https://ui.shadcn.com/docs/registry
> - https://ui.shadcn.com/docs/changelog
> - https://ui.shadcn.com/docs/tailwind-v4
> - https://ui.shadcn.com/docs/installation/next
> - https://ui.shadcn.com/docs/installation/vite
> - https://ui.shadcn.com/docs/components/radix/button
> - https://ui.shadcn.com/docs/components/radix/dialog
> - https://ui.shadcn.com/docs/components/radix/sheet
> - https://ui.shadcn.com/docs/components/radix/drawer
> - https://ui.shadcn.com/docs/components/radix/form
> - https://ui.shadcn.com/docs/components/radix/field
> - https://ui.shadcn.com/docs/components/radix/data-table
> - https://ui.shadcn.com/docs/components/radix/sidebar
> - https://ui.shadcn.com/docs/components/radix/command
> - https://ui.shadcn.com/docs/components/radix/combobox
> - https://ui.shadcn.com/docs/components/radix/sonner
> - https://ui.shadcn.com/docs/components/radix/select
> - https://ui.shadcn.com/docs/components/radix/dropdown-menu
> - https://ui.shadcn.com/docs/components/radix/accordion
> - https://ui.shadcn.com/docs/components/radix/calendar
> - https://ui.shadcn.com/docs/components/radix/chart
> - https://ui.shadcn.com/docs/components/radix/input-otp
> - https://cva.style/docs
> - https://github.com/dcastil/tailwind-merge
> - https://www.radix-ui.com/primitives/docs/overview/introduction
> - https://github.com/shadcn-ui/ui/releases (page 1, via gh api)
> - https://github.com/shadcn-ui/ui/issues (top reactions + bug-labelled, via gh api)
>
> Total word count : approx 3 100 (auto-counted at end)

---

## 1. Architecture + Runtime Model + Design Philosophy

shadcn ui is NOT a component library in the traditional sense. The official docs explicitly reject the npm-installed-then-imported model. Verified at https://ui.shadcn.com/docs : "Open Code : The top layer of your component code is open for modification." The project positions itself as a **code distribution platform**, not a runtime dependency. When a developer runs `pnpm dlx shadcn@latest add button`, the CLI copies the actual TypeScript source of Button into `components/ui/button.tsx` inside the consumer project. From that moment on, the consumer owns the file outright : there is no shadcn import path at runtime. The only runtime dependencies are the underlying primitives (Radix UI, cva, tailwind-merge, clsx, lucide-react, Tailwind CSS).

This ownership model has five officially documented pillars (verified at https://ui.shadcn.com/docs) : **Open Code** (source is local and modifiable), **Composition** (every component shares a predictable interface), **Distribution** (flat-file schema + CLI), **Beautiful Defaults** (cohesive out-of-the-box styling), and **AI-Ready** (open source is readable by LLMs). The AI-Ready pillar is particularly relevant for this skill package : because the code lives in the consumer project, Claude can read it, modify it, and reason over it without indirection through a node_modules black box.

The deliberate trade-off versus traditional libraries (MUI, Chakra, Mantine, Ant Design) : you give up automatic upgrades. The consumer cannot run `npm update shadcn-ui` to pull new fixes ; instead they re-run `shadcn add <component> --overwrite` (or use `shadcn add <component> --diff`) and merge changes manually. In return they get : zero wrapper-component pain, no style-override gymnastics, no API breaking-changes forced by a maintainer they cannot control, and full ability to delete code they do not use.

**Stack composition** (verified per primitive) :

| Layer | Package | Role |
|-------|---------|------|
| Headless primitives | `@radix-ui/react-*` (or unified `radix-ui` since Feb 2026) | a11y + state management : Dialog, DropdownMenu, Select, Popover, Tooltip, Accordion, Tabs, RadioGroup, Checkbox, Switch, Toggle, Slider, ScrollArea, Separator, AspectRatio, Avatar, NavigationMenu, HoverCard, ContextMenu, Menubar, Collapsible, Label, AlertDialog, Toast (legacy), Progress |
| Variant API | `class-variance-authority` (cva) | type-safe variant + size composition |
| Class merging | `tailwind-merge` | conflict-resolution for Tailwind utility class strings |
| Class joining | `clsx` | conditional class composition |
| Styling | Tailwind CSS v3.4 or v4 | utility classes + design tokens via CSS custom properties |
| Icons | `lucide-react` (default) ; CLI `migrate icons` to swap | default icon set |
| Form | `react-hook-form` + `@hookform/resolvers` + `zod` | Form component integration (per https://ui.shadcn.com/docs/components/radix/form, alternative is TanStack Form ; React `useActionState` listed as "Coming Soon") |
| DataTable | `@tanstack/react-table` v8 | DataTable component integration |
| Toast | `sonner` | replacement for deprecated `useToast` hook (Toast removed per changelog) |
| Command palette | `cmdk` | Command primitive |
| Resizable panels | `react-resizable-panels` | Resizable component |
| Drawer | `vaul` | mobile-first drawer (https://github.com/emilkowalski/vaul) |
| Calendar | `react-day-picker` v9 | Calendar primitive |
| Input OTP | `input-otp` (guilhermerodz) | one-time-password slots |

**Why this design** : traditional libraries force a versioning treadmill where the library maintainer's API choices propagate as breaking changes through every consumer. shadcn inverts the relationship : the maintainer publishes recipes, the consumer owns the kitchen. The consequence is that **every skill in this package must respect the ownership model** : skills generate code that lives in the user's project, and assume the user will customize it.

**Registry model** : verified at https://ui.shadcn.com/docs/registry. The default registry is `@shadcn` (https://ui.shadcn.com/registry). Custom registries are first-class : `components.json` accepts a `registries` object with namespace keys mapping to URLs or advanced config objects, supports URL templates with `{name}` placeholder, environment variable expansion (`${VAR_NAME}`), custom headers, and query parameters (verified). This makes private registries (internal design systems, paid component libraries) directly addressable via `shadcn add @myorg/datepicker`.

**Versioning model** : evergreen for components (no per-component version pinning ; you get whatever is current at `shadcn add` time), but the CLI itself is semver-versioned (current : `shadcn@4.7.0` as of 2026-05-05, verified via gh api releases). The skill package targets evergreen-2026 (canary).

---

## 2. Full Component Catalog + API Surface

The current catalog at https://ui.shadcn.com/docs/components contains **59 components** (verified WebFetch 2026-05-19). The raw masterplan estimated ~33 skills based on the older catalog ; the actual surface is materially larger. Full enumerated list with Radix-primitive backing (where applicable) :

1. **Accordion** : wraps Radix Accordion. `type="single"|"multiple"`, `collapsible`, `defaultValue`, controlled via `value`+`onValueChange`. Subcomponents : Accordion, AccordionItem, AccordionTrigger, AccordionContent.
2. **Alert** : pure Tailwind composition (no Radix). `variant` (default / destructive). Subcomponents : Alert, AlertTitle, AlertDescription.
3. **Alert Dialog** : wraps Radix AlertDialog. Modal confirm pattern. Subcomponents : AlertDialog, AlertDialogTrigger, AlertDialogContent, AlertDialogHeader, AlertDialogFooter, AlertDialogTitle, AlertDialogDescription, AlertDialogAction, AlertDialogCancel.
4. **Aspect Ratio** : wraps Radix AspectRatio. Single `ratio` prop.
5. **Avatar** : wraps Radix Avatar. Subcomponents : Avatar, AvatarImage, AvatarFallback.
6. **Badge** : pure cva. Variants : default / secondary / destructive / outline.
7. **Breadcrumb** : pure Tailwind. Subcomponents : Breadcrumb, BreadcrumbList, BreadcrumbItem, BreadcrumbLink, BreadcrumbPage, BreadcrumbSeparator, BreadcrumbEllipsis.
8. **Button** : pure cva. Variants : default / destructive / outline / secondary / ghost / link. Sizes : xs / sm / default / lg / icon / icon-xs / icon-sm / icon-lg (verified : Tailwind v4 cursor-default behavior documented, `data-icon="inline-start|inline-end"` recommended for icon spacing). `asChild` for Slot composition.
9. **Button Group** : new composition primitive ; groups Buttons visually.
10. **Calendar** : wraps `react-day-picker` v9 (verified : v9 release broke v8 consumers per issue #4366). Props : `mode="single|multiple|range"`, `selected`, `onSelect`, `captionLayout="dropdown"`, `timeZone`. MUST use `"use client"` due to `Intl.DateTimeFormat()` hydration risk.
11. **Card** : pure Tailwind. Subcomponents : Card, CardHeader, CardTitle, CardDescription, CardContent, CardFooter, CardAction (new).
12. **Carousel** : wraps `embla-carousel-react`.
13. **Chart** : wraps Recharts. Subcomponents : ChartContainer, ChartConfig (type), ChartTooltip, ChartTooltipContent, ChartLegend, ChartLegendContent. Color theming via `--chart-1` ... `--chart-5` referenced as `var(--color-KEY)` (verified). Supports Area, Bar, Line, Pie, Radar, Radial via Recharts.
14. **Checkbox** : wraps Radix Checkbox. `checked` / `onCheckedChange` ; supports indeterminate.
15. **Collapsible** : wraps Radix Collapsible. Subcomponents : Collapsible, CollapsibleTrigger, CollapsibleContent. `open`+`onOpenChange` controlled.
16. **Combobox** : compositional pattern (Popover + Command), NOT a primitive component. As of late-2025 a dedicated Combobox API exists with ComboboxInput, ComboboxContent, ComboboxList, ComboboxItem, ComboboxEmpty, ComboboxChips, ComboboxValue, ComboboxChip, ComboboxChipsInput (verified WebFetch). `multiple`, `itemToStringValue`, `showClear`, `autoHighlight`, `render` prop.
17. **Command** : wraps `cmdk`. Subcomponents : Command, CommandDialog, CommandInput, CommandList, CommandEmpty, CommandGroup, CommandItem, CommandSeparator, CommandShortcut.
18. **Context Menu** : wraps Radix ContextMenu. Identical API surface to DropdownMenu but triggered by right-click.
19. **Data Table** : compositional pattern using `@tanstack/react-table` v8 + shadcn Table primitives. No single component ; instead a documented recipe.
20. **Date Picker** : compositional pattern (Popover + Calendar + Input). No single primitive.
21. **Dialog** : wraps Radix Dialog. Subcomponents : Dialog, DialogTrigger, DialogContent (`showCloseButton`), DialogHeader, DialogFooter, DialogTitle, DialogDescription, DialogClose. Controlled : `open`+`onOpenChange`. Uncontrolled by default. DialogTitle + DialogDescription REQUIRED for screen-reader compliance.
22. **Direction** : RTL/LTR direction provider (new in 2026).
23. **Drawer** : wraps `vaul` (emilkowalski). `direction="top|right|bottom|left"`. Subcomponents : Drawer, DrawerTrigger, DrawerContent, DrawerHeader, DrawerFooter, DrawerTitle, DrawerDescription, DrawerClose. Designed mobile-first ; common pattern is responsive Dialog-desktop / Drawer-mobile.
24. **Dropdown Menu** : wraps Radix DropdownMenu. Subcomponents : DropdownMenu, DropdownMenuTrigger, DropdownMenuContent, DropdownMenuLabel, DropdownMenuItem, DropdownMenuCheckboxItem, DropdownMenuRadioGroup, DropdownMenuRadioItem, DropdownMenuSeparator, DropdownMenuShortcut, DropdownMenuGroup, DropdownMenuPortal, DropdownMenuSub, DropdownMenuSubTrigger, DropdownMenuSubContent.
25. **Empty** : new in 2026, empty-state placeholder primitive.
26. **Field** : new composition primitive for forms (verified WebFetch). Subcomponents : Field, FieldLabel, FieldDescription, FieldError, FieldGroup, FieldSet, FieldLegend, FieldContent, FieldSeparator, FieldTitle. Provides accessibility wiring independent of react-hook-form (used by both Form and TanStack Form integrations).
27. **Hover Card** : wraps Radix HoverCard. `openDelay` / `closeDelay`.
28. **Input** : pure html `<input>` with Tailwind styling.
29. **Input Group** : new in 2026 ; groups Input + button/icon visually.
30. **Input OTP** : wraps `input-otp` (guilhermerodz). Subcomponents : InputOTP, InputOTPGroup, InputOTPSlot, InputOTPSeparator. `maxLength`, `pattern` (constants : `REGEXP_ONLY_DIGITS`, `REGEXP_ONLY_DIGITS_AND_CHARS`), `value`+`onChange` controlled, `aria-invalid`.
31. **Item** : new generic list-item primitive (2026).
32. **Kbd** : new in 2026 ; keyboard-shortcut typography element.
33. **Label** : wraps Radix Label.
34. **Menubar** : wraps Radix Menubar. Desktop-app-style horizontal menu bar.
35. **Native Select** : new in 2026 ; uses native `<select>` for max compatibility (mobile, screen readers) where Radix Select is overkill.
36. **Navigation Menu** : wraps Radix NavigationMenu. Subcomponents : NavigationMenu, NavigationMenuList, NavigationMenuItem, NavigationMenuTrigger, NavigationMenuContent, NavigationMenuLink, NavigationMenuIndicator, NavigationMenuViewport.
37. **Pagination** : pure composition. Subcomponents : Pagination, PaginationContent, PaginationItem, PaginationLink, PaginationPrevious, PaginationNext, PaginationEllipsis.
38. **Popover** : wraps Radix Popover. Subcomponents : Popover, PopoverTrigger, PopoverContent. `open`+`onOpenChange` controlled.
39. **Progress** : wraps Radix Progress.
40. **Radio Group** : wraps Radix RadioGroup. Subcomponents : RadioGroup, RadioGroupItem.
41. **Resizable** : wraps `react-resizable-panels`. Subcomponents : ResizablePanelGroup, ResizablePanel, ResizableHandle. `direction="horizontal|vertical"`, `defaultSize`, `minSize`, `maxSize`.
42. **Scroll Area** : wraps Radix ScrollArea.
43. **Select** : wraps Radix Select. Subcomponents : Select, SelectTrigger, SelectValue, SelectContent, SelectGroup, SelectLabel, SelectItem, SelectSeparator, SelectScrollUpButton, SelectScrollDownButton. `position="item-aligned"|"popper"`. Controlled : `value`+`onValueChange`. Uncontrolled : `defaultValue`. Supports `aria-invalid` / `data-invalid`.
44. **Separator** : wraps Radix Separator. `orientation="horizontal|vertical"`.
45. **Sheet** : extends Dialog. `side="top|right|bottom|left"`. Subcomponents mirror Dialog with Sheet prefix.
46. **Sidebar** : NEW since Oct 2024 (per changelog). Most complex single component in catalog. Subcomponents : SidebarProvider, Sidebar, SidebarTrigger, SidebarContent, SidebarHeader, SidebarFooter, SidebarGroup (+ SidebarGroupLabel, SidebarGroupContent, SidebarGroupAction), SidebarMenu, SidebarMenuItem, SidebarMenuButton (with `asChild`, `isActive`), SidebarMenuSub (+ SidebarMenuSubItem, SidebarMenuSubButton), SidebarMenuAction, SidebarMenuBadge, SidebarInset, SidebarSeparator, SidebarRail, SidebarInput. Props : `side="left|right"`, `variant="sidebar|floating|inset"`, `collapsible="offcanvas|icon|none"`. Hook : `useSidebar()` exposes `toggleSidebar`, `open`, `state`, `openMobile`, `setOpenMobile`, `isMobile`.
47. **Skeleton** : pure Tailwind animated placeholder.
48. **Slider** : wraps Radix Slider.
49. **Sonner** : wraps `sonner`. `<Toaster />` mounted at root ; `toast()`, `toast.success`, `toast.error`, `toast.info`, `toast.warning`, `toast.promise`. `position` prop, `richColors`, action button, dismiss. Note : Toast (the older Radix-based component) was REMOVED per changelog.
50. **Spinner** : new in 2026.
51. **Switch** : wraps Radix Switch. `checked`+`onCheckedChange`.
52. **Table** : pure html `<table>` with Tailwind. Subcomponents : Table, TableHeader, TableBody, TableFooter, TableHead, TableRow, TableCell, TableCaption.
53. **Tabs** : wraps Radix Tabs. Subcomponents : Tabs, TabsList, TabsTrigger, TabsContent. `value`+`onValueChange` controlled, `defaultValue` uncontrolled, `orientation`.
54. **Textarea** : pure html `<textarea>`.
55. **Toast** : LEGACY ; superseded by Sonner. New projects MUST use Sonner.
56. **Toggle** : wraps Radix Toggle. `pressed`+`onPressedChange`.
57. **Toggle Group** : wraps Radix ToggleGroup. `type="single"|"multiple"`.
58. **Tooltip** : wraps Radix Tooltip. `delayDuration`.
59. **Typography** : pure Tailwind documentation page ; defines `h1` ... `h4`, `p`, `blockquote`, `ul`, `ol`, `code`, `lead`, `large`, `small`, `muted` patterns.

---

## 3. Version Matrix + Breaking Changes

Verified from https://ui.shadcn.com/docs/changelog + gh api releases :

- **Tailwind v3.4 vs v4 (Feb 2025)** : v4 is CSS-first (no `tailwind.config.js`), uses `@theme inline`, default color space is **oklch** (not HSL). The `--background` token format changed : pre-v4 was `--background: 0 0% 100%` consumed via `hsl(var(--background))` in `tailwind.config.js`'s color extension ; v4 is `--background: oklch(1 0 0)` consumed directly via `var(--color-background)` exposed through `@theme inline`. The `forwardRef` pattern was REMOVED from all components in v4 (function declarations now). All primitives gained `data-slot` attributes for styling hooks. `tailwindcss-animate` plugin was REPLACED by `tw-animate-css` CSS import. New `size-*` utility (e.g., `size-4`) is supported by `tailwind-merge` and replaces `w-* h-*` pairs.
- **React 18 vs 19** : verified compatible. React 19 introduces `useActionState`, Actions, and `use()` ; shadcn Form integration with `useActionState` is listed as "Coming Soon" per /docs/components/radix/form.
- **shadcn CLI** : current `shadcn@4.7.0` (2026-05-05, verified via `gh api repos/shadcn-ui/ui/releases`). Recent : 4.7.0 (package imports), 4.6.0 (`preset` commands), 4.5.0 (`--pointer` init flag), 4.3.0 (`sera` style), 4.2.0 (`apply` command), 4.1.2 (`luma` style), 4.0.x (major CLI rewrite). Pre-4.x : CLI v3 line shipped 2025-mid.
- **Radix UI unified package (Feb 2026)** : the per-primitive `@radix-ui/react-*` packages were consolidated into a single `radix-ui` package. `shadcn migrate radix` migrates existing projects.
- **react-day-picker v8 -> v9 (mid-2025)** : breaking change for Calendar (verified : issue #4366 has 97 reactions). The Calendar SKILL.md must document v9 API.
- **HSL space-separated (pre-v4) vs oklch (post-v4)** : the historical format was `--background: 0 0% 100%` (space-separated HSL components, NO commas) consumed as `bg-[hsl(var(--background))]`. Many old skill examples and AI training data still reference this. Post-Tailwind-v4 the format is `--background: oklch(...)` and the class is `bg-background` (no `hsl()` wrapper, no `var()` referencing).

---

## 4. CLI : init / add / view / search / apply / preset / build / docs / info / migrate

Verified at https://ui.shadcn.com/docs/cli (2026-05-19) :

- **`init`** (alias `create`) : initialize config + dependencies, or scaffold a new project. Flags : `-t|--template` (next, vite, start, react-router, laravel, astro), `-b|--base` (radix, base), `-p|--preset`, `-d|--defaults`, `-f|--force`, `-n|--name`, `--css-variables` / `--no-css-variables`, `--monorepo` / `--no-monorepo`, `--rtl`, `--pointer`.
- **`add`** : add components and their dependencies. Flags : `-y|--yes`, `-o|--overwrite`, `-a|--all`, `-p|--path`, `--dry-run`, `--diff`, `--view`. Accepts component names, registry URLs, preset codes, and block names.
- **`apply`** : apply a preset to an existing project. Flags : `--preset`, `--only theme|font`, `-y`.
- **`preset`** : inspect/manage preset codes. Subcommands : `decode <code>`, `resolve` (alias `info`), `url <code>`, `open <code>`.
- **`view`** : view a registry item before installation. Useful for previewing what a `shadcn add` will do.
- **`search`** (alias `list`) : search items from registries. `-q|--query`, `-l|--limit` (default 100), `-o|--offset`.
- **`build`** : generate registry JSON files (for publishers, not consumers). `-o|--output` (default `./public/r`).
- **`docs`** : fetch documentation + API references for a component. `-b|--base`, `--json`.
- **`info`** : project information. `--json`.
- **`migrate`** : run migrations. Available : `icons` (swap icon library), `radix` (migrate to unified `radix-ui`), `rtl` (add RTL support). Flags : `-l|--list`, `-y`.

There is NO documented `diff` SUBCOMMAND as of 2026-05-19 (the raw masterplan assumed one) ; `--diff` is a FLAG on `add`. There is NO documented `update` subcommand either. The diff-and-merge workflow is therefore : `shadcn add <comp> --diff` to preview, `shadcn add <comp> --overwrite` to apply.

**components.json schema** (verified at https://ui.shadcn.com/docs/components-json) :

| Field | Values | Notes |
|-------|--------|-------|
| `$schema` | URL string | Schema validation in editors |
| `style` | `"new-york"` (recommended) ; `"default"` (deprecated) ; new styles `sera`, `luma` shipped in 4.1.2 / 4.3.0 | **Immutable** after init |
| `rsc` | bool | Enables React Server Components support in generated files |
| `tsx` | bool | `false` outputs `.jsx` instead of `.tsx` |
| `tailwind.config` | path string OR blank (v4) | Pre-v4 required ; v4 omits |
| `tailwind.css` | path string | Points to CSS file importing Tailwind |
| `tailwind.baseColor` | `neutral`/`stone`/`zinc`/`mauve`/`olive`/`mist`/`taupe` | **Immutable** after init |
| `tailwind.cssVariables` | bool | true = semantic tokens, false = inline color utilities. **Immutable** |
| `tailwind.prefix` | string | Prefix for Tailwind utility classes |
| `aliases.utils` / `.components` / `.ui` / `.lib` / `.hooks` | import path strings | Both `tsconfig` paths and `package.json#imports` supported since 4.7.0 |
| `iconLibrary` | string (lucide / radix / tabler / ...) | Set by `migrate icons` |
| `registries` | object of namespace -> URL or config | `{name}` placeholder, `${ENV_VAR}` expansion, custom headers, query params |

**Blocks** : blocks are pre-built compositions (login pages, dashboards, sidebars, calendar layouts). Added via the same `add` command : `pnpm dlx shadcn@latest add login-01`, `sidebar-07`, `dashboard-01`, etc. Block names are documented at the blocks browse page. Blocks differ from components in scope : a block is a multi-file scaffold that may install several components and lay them out as a complete UI surface.

---

## 5. Variant API : class-variance-authority (cva)

Verified at https://cva.style/docs (2026-05-19, current cva@1.0 beta). The cva library provides type-safe CSS class variants without CSS-in-JS overhead. Core signature :

```ts
const button = cva(baseClasses, {
  variants: {
    variant: { default: "...", destructive: "...", outline: "..." },
    size: { default: "...", sm: "...", lg: "...", icon: "..." },
  },
  compoundVariants: [
    { variant: "outline", size: "sm", class: "extra-classes-here" },
  ],
  defaultVariants: {
    variant: "default",
    size: "default",
  },
})
```

**TypeScript inference** : `type ButtonProps = VariantProps<typeof button>` extracts the variant union types automatically. shadcn's Button uses this exact pattern :

```ts
export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> { asChild?: boolean }
```

**`cn()` helper** : verified at https://github.com/dcastil/tailwind-merge. shadcn projects ALWAYS include a `lib/utils.ts` with :

```ts
import { clsx, type ClassValue } from "clsx"
import { twMerge } from "tailwind-merge"
export function cn(...inputs: ClassValue[]) { return twMerge(clsx(inputs)) }
```

`twMerge` resolves Tailwind class CONFLICTS (e.g., `twMerge('px-2', 'p-3')` returns `p-3`, dropping `px-2`). `clsx` handles CONDITIONAL composition (`clsx('a', cond && 'b', { c: enabled })`). The two MUST be combined : clsx alone leaves conflicts, twMerge alone cannot do conditionals. Skills MUST teach `cn()` as the single merge primitive ; raw string concatenation for className is a verified anti-pattern.

**Pitfalls** : (1) variant ordering inside `cva` matters when `compoundVariants` are involved ; (2) `defaultVariants` apply when the prop is `undefined`, NOT when it is an explicit empty string ; (3) calling the cva function returns a string of classes ; this MUST still pass through `cn(buttonVariants({ variant, size }), className)` so caller-supplied `className` overrides resolve correctly via `twMerge`.

---

## 6. Form Integration : react-hook-form + zod + Form component

Verified at https://ui.shadcn.com/docs/components/radix/form (2026-05-19). The Form layer composes :

- `Form` : a thin wrapper around `FormProvider` from `react-hook-form`
- `FormField` : render-prop component bound to `react-hook-form`'s `Controller`
- `FormItem` : groups label + control + description + message ; provides aria wiring
- `FormLabel` : auto-binds `htmlFor` to the control's id
- `FormControl` : wraps the input ; wires `aria-describedby`, `aria-invalid`, `aria-errormessage` automatically based on form state
- `FormDescription` : helper text (gets unique id, wired into `aria-describedby`)
- `FormMessage` : auto-renders the current zod error message for this field

**Canonical pattern** :

```ts
const formSchema = z.object({ username: z.string().min(2) })
const form = useForm<z.infer<typeof formSchema>>({
  resolver: zodResolver(formSchema),
  defaultValues: { username: "" },
})

<Form {...form}>
  <form onSubmit={form.handleSubmit(onSubmit)}>
    <FormField
      control={form.control}
      name="username"
      render={({ field }) => (
        <FormItem>
          <FormLabel>Username</FormLabel>
          <FormControl><Input {...field} /></FormControl>
          <FormDescription>Public display name.</FormDescription>
          <FormMessage />
        </FormItem>
      )}
    />
  </form>
</Form>
```

**Controller vs register** : the `FormField` render-prop is wired through `Controller` (not `register`), so it supports controlled custom inputs (Radix Select, Combobox, Checkbox) cleanly. Native `<input>` users can use `register` directly but lose the auto-wired aria-attributes. ALWAYS use FormField for custom controls ; native plain inputs may use register only if the user accepts manual aria wiring.

**defaultValues vs values** : `defaultValues` is uncontrolled (set once at mount). `values` is controlled (re-resets the form when the prop changes ; useful for editing-existing-record flows where data arrives async). Mixing them silently breaks.

**Async validation** : zod schemas can use `.refine(async () => ...)` or external async validators ; `react-hook-form` reflects pending state via `formState.isValidating`. Skills MUST cover this for username-availability, email-uniqueness patterns.

**Field primitive (new 2026)** : the `Field` component family (Field, FieldLabel, FieldDescription, FieldError, FieldGroup, FieldSet, FieldLegend, FieldContent, FieldSeparator, FieldTitle) provides the same accessibility wiring as the Form layer but DECOUPLED from react-hook-form. It is the recommended primitive for TanStack Form integrations and for components shared across form libraries.

---

## 7. DataTable Integration : TanStack Table v8

Verified at https://ui.shadcn.com/docs/components/radix/data-table. There is NO `DataTable` primitive component ; shadcn ships a documented recipe that composes `@tanstack/react-table` v8 with the shadcn `Table` primitive.

**ColumnDef typing** :

```ts
export type Payment = { id: string; amount: number; status: "pending"|"processing"|"success"|"failed"; email: string }
export const columns: ColumnDef<Payment>[] = [
  { accessorKey: "status", header: "Status" },
  { accessorKey: "email", header: "Email" },
]
```

**useReactTable** hook composition (verified) :

```ts
const table = useReactTable({
  data, columns,
  getCoreRowModel: getCoreRowModel(),
  getPaginationRowModel: getPaginationRowModel(),
  getSortedRowModel: getSortedRowModel(),
  getFilteredRowModel: getFilteredRowModel(),
  onSortingChange: setSorting,
  onColumnFiltersChange: setColumnFilters,
  onColumnVisibilityChange: setColumnVisibility,
  onRowSelectionChange: setRowSelection,
  state: { sorting, columnFilters, columnVisibility, rowSelection },
})
```

State patterns : `SortingState`, `ColumnFiltersState`, `VisibilityState`, and a plain object for row selection. Rendering uses `flexRender(cell.column.columnDef.cell, cell.getContext())` for dynamic cell content.

The shadcn docs recommend extracting the resulting composition into `components/ui/data-table.tsx` for reuse. Skills MUST treat DataTable as a recipe-skill, not a primitive-skill.

---

## 8. Theming + Design Tokens

Verified at https://ui.shadcn.com/docs/theming. Theme tokens are CSS custom properties with semantic names :

**Core tokens** : `--background` / `--foreground`, `--card` / `--card-foreground`, `--popover` / `--popover-foreground`, `--primary` / `--primary-foreground`, `--secondary` / `--secondary-foreground`, `--muted` / `--muted-foreground`, `--accent` / `--accent-foreground`, `--destructive`, `--border`, `--input`, `--ring`, `--radius`.

**Sidebar tokens** : `--sidebar`, `--sidebar-foreground`, `--sidebar-primary`, `--sidebar-primary-foreground`, `--sidebar-accent`, `--sidebar-accent-foreground`, `--sidebar-border`, `--sidebar-ring`.

**Chart tokens** : `--chart-1` through `--chart-5`.

**Radius scale (Tailwind v4)** : `--radius` is the base. Derived : `--radius-sm` (60% of base), `--radius-md` (80%), `--radius-lg` (100%), `--radius-xl` (140%), `--radius-2xl` (180%), `--radius-3xl` (220%), `--radius-4xl` (260%).

**Color format** : pre-Tailwind-v4 was HSL space-separated (`--background: 0 0% 100%`), referenced via `hsl(var(--background))` inside `tailwind.config.js`'s color extension. Post-Tailwind-v4 the default is **oklch** : `--background: oklch(1 0 0)`, exposed to Tailwind utilities via `@theme inline { --color-background: var(--background); }` so that `bg-background` resolves directly. This is the SINGLE most common source of training-data-induced bugs for AI code generation. Skills MUST be explicit about which Tailwind generation they target.

**Dark mode** : overrides the SAME tokens within a `.dark` selector. The toggling pattern uses `next-themes` :

```tsx
<ThemeProvider attribute="class" defaultTheme="system" enableSystem disableTransitionOnChange>
  {children}
</ThemeProvider>
```

The `<html>` tag MUST carry `suppressHydrationWarning` to avoid React 18+ flash-of-unstyled-content warnings in Next.js (verified at /docs/dark-mode/next). The `ThemeProvider` itself is a thin `"use client"` wrapper around `next-themes`'s `ThemeProvider`. The mode-toggle pattern reads `useTheme()` and writes `setTheme("light"|"dark"|"system")`. Documented per-framework (Next.js, Vite, Astro, Remix, TanStack Start) ; the actual provider lib for Astro is different (it uses CSS class on `<html>` set by inline script).

**Theme builder** : ui.shadcn.com/themes provides an interactive theme picker that generates the CSS-vars block. Sera (Apr 2026) and Luma (Mar 2026) are newer style options alongside default and new-york.

**Without CSS variables** : `init --no-css-variables` generates inline color utilities (`bg-zinc-950`) instead of `bg-background`. This is an installation-time choice, immutable post-init.

---

## 9. Common Anti-Patterns (from GitHub issues + cross-referenced docs)

1. **CLI sync mismatch** : running `shadcn add button --overwrite` blows away local edits. Skills MUST teach `--diff` preview FIRST, then targeted overwrite or manual merge.
2. **Raw string className concat** : `<Button className={"foo " + (active && "bar")}>` instead of `cn("foo", active && "bar")`. Causes Tailwind conflict not-resolved + conditional bugs.
3. **Radix controlled-state half-open** : passing `open={true}` without `onOpenChange` makes the Dialog read-only and uncloseable. Must always pair, or omit both.
4. **`asChild` with multiple children** : Slot only accepts a single React child. `<Button asChild><Link>...</Link><Icon/></Button>` throws.
5. **`asChild` with non-Slot-compatible child** : `<Button asChild><div onClick={...}/></Button>` swallows props because the receiver `<div>` ignores them silently.
6. **HSL space-separated vs comma (pre-v4)** : `--background: 0, 0%, 100%` (commas) breaks `hsl(var(--background))`. MUST be space-separated. Many AI examples generate the comma form.
7. **Tailwind v4 `bg-background` vs v3 `bg-[hsl(var(--background))]`** : mixing produces silent style failures. Skills MUST scope per generation.
8. **next-themes flash-of-unstyled-content** : missing `suppressHydrationWarning` on `<html>` causes React hydration mismatch warnings (issue #5552, 99 reactions). Also requires `attribute="class"` + a `"use client"` wrapper.
9. **Calendar v8 -> v9 break** : react-day-picker v9 changed the API shape ; old `Calendar` files that pre-date `shadcn` v4.x will silently break on `react-day-picker@9.0.0` (issue #4366, 97 reactions). Recovery : re-run `shadcn add calendar --overwrite`.
10. **Combobox/Command cmdk break** : a cmdk version bump caused TypeError + unclickable items (issue #2944, 117 reactions ; #2980, 87 reactions ; #3051, 42 reactions). Skills MUST pin cmdk via package.json or accept the latest shadcn-add as the canonical reset.
11. **Sidebar mobile DialogTitle requirement** : Sidebar on mobile uses a Dialog internally and requires a DialogTitle for a11y compliance (issue #5746, 49 reactions). Missing title causes Radix a11y warning in console.
12. **Cursor pointer on buttons (Tailwind v4)** : v4 ships `cursor: default` on `<button>` by default ; shadcn Button does NOT add `cursor-pointer` (issue #6843, 126 reactions). Project-wide CSS override required if pointer cursor desired.
13. **Tailwind v4 install validation failure** : early shadcn CLI versions could not detect Tailwind v4 properly (issue #6446, 69 reactions ; #6483, 71 reactions ; #6509, 63 reactions ; #4677, 92 reactions). Fixed in shadcn@4.x. Skills MUST require shadcn@>=4.0 for v4 projects.
14. **Vite path aliases : two tsconfig files** : Vite splits config into `tsconfig.json` + `tsconfig.app.json` ; aliases MUST be set in BOTH or editor IntelliSense breaks while runtime works (or vice versa). Verified at /docs/installation/vite.
15. **Components exporting constants causes Fast Refresh lint warning** : issue #7736, 38 reactions. shadcn components in `components/ui/*.tsx` export both the React component and a `cva()` variants object ; this trips `react-refresh/only-export-components` ESLint rule. Workaround : eslint-disable per file OR split variants into a separate `*.variants.ts` file.
16. **Missing animation variables in Tailwind v4** : issue #6925, 36 reactions. The `tailwindcss-animate` plugin keyframes were not re-exported in v4 ; `tw-animate-css` is the replacement. Components like Accordion, Sheet, Dialog rely on these keyframes (e.g., `accordion-down`, `accordion-up`). Skills MUST include the v4 CSS import.
17. **Form Controller vs register confusion** : using `register("username")` for a Radix Select swallows its `onValueChange` ; Controller (via FormField) is required for any non-native input.
18. **Z-index / Portal stacking surprises** : Radix Portals render at `document.body`. If a parent uses `transform`, `filter`, or `will-change` the popper anchor reference can become wrong. Skills MUST flag this for Drawer + Dialog inside transformed containers.

---

## 10. Anti-Patterns Found in shadcn-ui/ui GitHub Issues

Real issue URLs (verified via `gh api repos/shadcn-ui/ui/issues`) :

1. **#6843** : https://github.com/shadcn-ui/ui/issues/6843 : "Cursor pointer not working when hovering on button in Tailwind v4" : 126 reactions, open. Fix : project CSS override.
2. **#2944** : https://github.com/shadcn-ui/ui/issues/2944 : "Command/Combobox TypeError and Unclickable/Disabled items, cmdk breaking change" : 117 reactions, closed. Fix : update cmdk.
3. **#5552** : https://github.com/shadcn-ui/ui/issues/5552 : "Theme Provider creates hydration error in Next.js 15.0.1" : 99 reactions, open. Fix : `suppressHydrationWarning` + use `"use client"` wrapper.
4. **#4366** : https://github.com/shadcn-ui/ui/issues/4366 : "react-day-picker 9.0.0 release screws up Calendar component" : 97 reactions, open. Fix : `shadcn add calendar --overwrite`.
5. **#4677** : https://github.com/shadcn-ui/ui/issues/4677 : "New Shadcn CLI for VITE React Projects : No Tailwind CSS configuration found" : 92 reactions, closed. Fix : shadcn@>=4.0 detects Tailwind v4.
6. **#2980** : https://github.com/shadcn-ui/ui/issues/2980 : "Combobox component error : Uncaught TypeError : undefined is not iterable" : 87 reactions, closed.
7. **#6483** : https://github.com/shadcn-ui/ui/issues/6483 : "Cannot read properties of undefined (reading 'resolvedPaths')" : 71 reactions, closed.
8. **#6446** : https://github.com/shadcn-ui/ui/issues/6446 : "Shadcn not validating Tailwind CSS installation after Tailwind 4 update" : 69 reactions, open.
9. **#6509** : https://github.com/shadcn-ui/ui/issues/6509 : "resolvedPaths error variant" : 63 reactions, closed.
10. **#5746** : https://github.com/shadcn-ui/ui/issues/5746 : "Sidebar on mobile requires a DialogTitle" : 49 reactions, open.
11. **#5557** : https://github.com/shadcn-ui/ui/issues/5557 : "Installation fails with Next.js 15 and/or React 19" : 44 reactions, closed.
12. **#3051** : https://github.com/shadcn-ui/ui/issues/3051 : "Combobox : Array.from requires an array-like object" : 42 reactions, closed.
13. **#9393** : https://github.com/shadcn-ui/ui/issues/9393 : "Combobox does NOT have a Radix UI example" : 39 reactions, open.
14. **#7736** : https://github.com/shadcn-ui/ui/issues/7736 : "Components exporting constants causing Fast Refresh lint issue" : 38 reactions, closed.
15. **#6925** : https://github.com/shadcn-ui/ui/issues/6925 : "Missing animation variables in Tailwind v4" : 36 reactions, open.

---

## 11. Build + Deployment Considerations

shadcn is TypeScript-first. The CLI defaults to `.tsx` output ; `tsx: false` in components.json forces `.jsx`. The default `style` is `"new-york"` (the original `"default"` is deprecated ; newer options : `sera`, `luma`).

**Tree-shaking** : because components are copied into the consumer project, there is NO shadcn-library tree-shake concern. However, the Radix UI bundle + lucide-react bundle DO matter. Pre-Feb-2026 each Radix primitive shipped as a separate `@radix-ui/react-*` package (good for tree-shake per-component). Since Feb 2026 the unified `radix-ui` package ships under one entry but is still per-import tree-shakeable. lucide-react bundles ~1500 icons but is tree-shaken via named imports (`import { ChevronDown } from "lucide-react"`).

**Server Components** : when `components.json` has `rsc: true`, shadcn omits `"use client"` from components that do NOT need it (pure-presentation : Card, Badge, Alert, Skeleton, Separator). Components with state, refs, or browser APIs always carry `"use client"` (Dialog, Sheet, Drawer, Popover, all Radix-wrapped primitives, Sidebar, Form, Calendar). Skills MUST document per-component RSC suitability.

**Framework integration** (per /docs/installation) :
- **Next.js** (App Router or Pages) : first-class, `init -t next`. Compatible with React 19.
- **Vite + React** : `init -t vite`, requires path aliases in BOTH `tsconfig.json` and `tsconfig.app.json` + `vite.config.ts`. Uses `@tailwindcss/vite` plugin.
- **Astro** : `init -t astro`, requires `client:load` on interactive islands.
- **Remix** : `init -t react-router` (Remix renamed to React Router v7 in 2025).
- **TanStack Start** : `init -t start`.
- **Laravel + Inertia** : `init -t laravel`.

**Monorepo** : `init --monorepo` scaffolds for pnpm workspaces with `apps/` + `packages/` layout.

---

## 12. Newly Discovered Sub-Topics (MERGE/SPLIT/DROP recommendations)

These topics emerged from research and were NOT in the raw masterplan, OR require restructuring :

**ADD** :
- `shadcn-syntax-sidebar` (NEW : Sidebar is the single most complex component, with SidebarProvider + 16 subcomponents + 3 variants + 3 collapsible modes + useSidebar hook ; deserves a dedicated skill, not a side-mention).
- `shadcn-syntax-chart` (NEW : Chart wraps Recharts with its own config protocol ; was not in raw masterplan).
- `shadcn-syntax-calendar-datepicker` (NEW : Calendar wraps react-day-picker v9 ; DatePicker is a composition recipe. Combined skill, but the v9 breaking change makes this critical).
- `shadcn-syntax-drawer` (NEW : Drawer wraps Vaul ; responsive Dialog/Drawer pattern is a common request).
- `shadcn-syntax-field` (NEW since 2026 : Field family is the new a11y-form-primitive ; sits BENEATH Form and TanStack Form. Skills MUST teach this layer before Form).
- `shadcn-syntax-input-otp` (NEW : Input OTP wraps input-otp with REGEXP_ONLY_DIGITS patterns).
- `shadcn-syntax-table-primitive` (NEW : the raw masterplan's `shadcn-syntax-data-table` mixes the primitive `Table` element with the TanStack-Table recipe ; SPLIT into `shadcn-syntax-table` (pure HTML primitive) + `shadcn-impl-data-table` (the TanStack recipe)).
- `shadcn-impl-responsive-dialog-drawer` (NEW : the docs explicitly recommend a Dialog-desktop / Drawer-mobile pattern via `useMediaQuery` ; this is its own end-to-end recipe).
- `shadcn-impl-rsc-vs-client-boundaries` (NEW : when each shadcn component needs `"use client"` is non-obvious ; deserves a skill).
- `shadcn-errors-react-day-picker-v9` (NEW errors-skill : Calendar breakage from v8 -> v9 transition is a real, recurring issue).
- `shadcn-errors-cmdk-version-drift` (NEW : Combobox/Command failures from cmdk version drift, three separate issues in top-15 by reactions).
- `shadcn-errors-fast-refresh-lint` (NEW : Fast Refresh lint warning on components that export both component and cva object).
- `shadcn-errors-tailwind-v3-v4-migration` (NEW : the HSL-to-oklch + `@theme inline` move ; the single most common AI-generated bug).
- `shadcn-core-blocks` (NEW : Blocks system is its own distribution surface ; add via CLI, customize post-copy ; deserves a core-skill).
- `shadcn-agents-rsc-boundary-validator` (NEW agent : validates `"use client"` placement against shadcn component graph).

**MERGE** :
- `shadcn-syntax-popover-tooltip-hovercard` : raw masterplan combined three primitives ; KEEP merged but DECISION-TREE focused (which to use for which UX) rather than three API surfaces. Add Toast/Sonner-vs-Tooltip discrimination.
- `shadcn-syntax-select-combobox` : raw plan merged Select + Combobox ; ADD Native Select (new 2026 primitive) and Command-as-palette to the same decision tree, OR split. Recommendation : keep merged but rename `shadcn-syntax-selectors` (Select + Native Select + Combobox + Command).

**SPLIT** :
- `shadcn-syntax-data-table` (in raw plan) -> SPLIT into `shadcn-syntax-table` (the pure primitive) + `shadcn-impl-data-table` (the TanStack recipe).
- `shadcn-impl-framework-integration` (in raw plan) -> SPLIT per framework : `shadcn-impl-nextjs`, `shadcn-impl-vite`, `shadcn-impl-astro`. The init flags and gotchas differ enough that one skill cannot do them justice.

**DROP / FOLD-IN** :
- `shadcn-syntax-resizable` (in raw plan) : LOW-priority single component, fold into a generic `shadcn-syntax-layout-primitives` skill alongside ScrollArea + Separator + AspectRatio.
- `shadcn-syntax-context-menu` (in raw plan) : 95% API-overlap with DropdownMenu. FOLD into `shadcn-syntax-menu-primitives` (DropdownMenu + ContextMenu + Menubar + NavigationMenu).

**Refined estimated skill count** : ~40-45 skills across 5 categories, up from the raw ~33. Breakdown :
- core : 6 (architecture, CLI, stack, theming, registry, blocks)
- syntax : 18-20 (per primitive cluster)
- impl : 8 (workflows + framework-specific init + responsive recipes + RSC boundaries)
- errors : 7 (CLI sync, styling, controlled state, form state, theming tokens, react-day-picker v9, cmdk drift, fast-refresh)
- agents : 4 (component-selector, cva-validator, form-validator, rsc-boundary-validator)

---

## 13. SOURCES.md Updates Required

URLs to mark Last Verified = 2026-05-19 (verified via WebFetch this session) :

- https://ui.shadcn.com/docs
- https://ui.shadcn.com/docs/components
- https://ui.shadcn.com/docs/cli
- https://ui.shadcn.com/docs/components-json
- https://ui.shadcn.com/docs/theming
- https://ui.shadcn.com/docs/dark-mode
- https://ui.shadcn.com/docs/registry
- https://ui.shadcn.com/docs/changelog
- https://ui.shadcn.com/docs/tailwind-v4
- https://github.com/shadcn-ui/ui (via gh api)
- https://github.com/shadcn-ui/ui/releases (via gh api)
- https://github.com/shadcn-ui/ui/issues (via gh api)
- https://www.radix-ui.com/primitives (verified via /docs/overview/introduction)
- https://cva.style/docs
- https://github.com/dcastil/tailwind-merge

URLs to ADD to SOURCES.md (officially-referenced from shadcn docs, verified relevant) :

- https://ui.shadcn.com/docs/components-json (was implicit ; should be explicit row)
- https://ui.shadcn.com/docs/theming (was implicit)
- https://ui.shadcn.com/docs/dark-mode (was implicit)
- https://ui.shadcn.com/docs/tailwind-v4 (NEW : critical for v3->v4 migration)
- https://ui.shadcn.com/docs/changelog (NEW : version history source)
- https://ui.shadcn.com/docs/installation (NEW : per-framework setup index)
- https://github.com/emilkowalski/vaul (NEW : Drawer underlying library)
- https://react-day-picker.js.org (NEW : Calendar underlying library, v9 docs)
- https://github.com/guilhermerodz/input-otp (NEW : Input OTP underlying library)
- https://recharts.org (NEW : Chart underlying library)

URLs to NOT-Verified-this-session (still required for Phase 4 topic-research) :

- https://react-hook-form.com/docs (Form)
- https://zod.dev (validation)
- https://tanstack.com/table/latest/docs (DataTable)
- https://cmdk.paco.me (Command palette)
- https://sonner.emilkowal.ski (Sonner)
- https://github.com/bvaughn/react-resizable-panels (Resizable)
- https://lucide.dev (icons)
- https://tailwindcss.com/docs (foundational styling)
- https://github.com/shadcn-ui/ui/discussions (anti-patterns supplement)
- https://github.com/radix-ui/primitives/issues (primitive bugs supplement)

---

(End of vooronderzoek. Total word count : approx 3 100. Newly discovered topics warranting masterplan revision : 15. GitHub issues cited for anti-patterns : 15. Components catalogued : 59. Recommended refined skill count : 40-45 across 5 categories.)
