# Masterplan : shadcn ui

> Status : Phase 1 raw : pre-research
> Generated : 2026-05-19
> Methodology : 7-phase research-first (BOOTSTRAP-RUNBOOK §3.5)

## Scope

- Technology : shadcn ui
- Versions : evergreen-2026 (canary)
- Languages : TypeScript, React (18+ / 19), Tailwind CSS (v3.4 / v4)
- Prefix : shadcn
- License : MIT
- Description : Deterministic Claude skills for shadcn ui component selection, variant API (cva), theming tokens, composition patterns, and CLI-driven installation flow.

## Paradigm note (critical for every skill)

shadcn ui is NOT a library. Consumers run `npx shadcn@latest add <component>` and the CLI copies TypeScript source into `components/ui/` inside the consumer project. There is no runtime dependency on shadcn itself : only on Radix UI primitives, cva, tailwind-merge, and Tailwind CSS. Every skill must respect this ownership model.

## Underlying stack (verify per skill)

| Layer | Package | Role |
|-------|---------|------|
| Headless primitives | @radix-ui/react-* | a11y + state management for Dialog, DropdownMenu, Select, Popover, Tooltip, etc. |
| Variant API | class-variance-authority (cva) | type-safe variant/size composition |
| Class merging | tailwind-merge | conflict-resolution for Tailwind utility class strings |
| Class joining | clsx | conditional class composition |
| Styling | Tailwind CSS v3.4 or v4 | utility classes + design tokens via CSS custom properties |
| Icons | lucide-react | default icon set |
| Form | react-hook-form + @hookform/resolvers + zod | Form component integration |
| DataTable | @tanstack/react-table v8 | DataTable component integration |
| Toast | sonner | replacement for old useToast hook |
| Command palette | cmdk | Command primitive |
| Resizable panels | react-resizable-panels | Resizable component |

## Identified Topics (raw, pre-research)

Brainstormed from prompt scope + Radix component catalog. Counts are estimates : refinement happens in Phase 3 after deep research.

### Core (architecture, cross-cutting concerns) : ~5 skills

- `shadcn-core-architecture` : copy-not-install philosophy, components.json schema, ownership model, why-no-npm-install
- `shadcn-core-cli` : init / add / diff / update / build commands, registry-resolution, components.json fields
- `shadcn-core-stack` : Radix + cva + tailwind-merge + clsx + lucide composition map, what each layer owns
- `shadcn-core-theming` : CSS custom properties as design tokens, HSL space-separated format, --background / --foreground / --primary / --muted / --accent / --destructive / --border / --input / --ring, dark mode toggle pattern, theme generation
- `shadcn-core-registry` : default registry vs private registries, registry.json schema, custom-registry hosting

### Syntax (component API patterns) : ~14 skills

- `shadcn-syntax-button` : variant API via cva (default / destructive / outline / secondary / ghost / link), size (default / sm / lg / icon), asChild Slot pattern
- `shadcn-syntax-dialog` : Radix Dialog primitive, Trigger / Portal / Overlay / Content / Header / Footer / Title / Description / Close, controlled (open + onOpenChange) vs uncontrolled
- `shadcn-syntax-sheet` : Dialog-derivative with `side` prop (top / right / bottom / left), when Sheet over Dialog (large content, side-panel UX)
- `shadcn-syntax-form` : react-hook-form + zodResolver, Form / FormField / FormItem / FormLabel / FormControl / FormDescription / FormMessage composition, Controller pattern
- `shadcn-syntax-select-combobox` : Select (Radix) vs Combobox (Popover + Command), when each : Select for known short list, Combobox for searchable, Command for command-palette UX
- `shadcn-syntax-command` : cmdk underneath, CommandDialog wrapping, CommandInput / List / Empty / Group / Item / Separator / Shortcut, async filtering pattern
- `shadcn-syntax-data-table` : TanStack Table v8 integration, ColumnDef typing, useReactTable hook, sorting / filtering / pagination / row-selection / column-visibility patterns
- `shadcn-syntax-toast-sonner` : Sonner replacement of old toast, toast() / toast.success / toast.error / toast.promise patterns, position prop, Toaster placement
- `shadcn-syntax-dropdown-menu` : Radix DropdownMenu, Trigger / Content / Item / Sub / SubTrigger / SubContent / Group / Label / Separator / CheckboxItem / RadioGroup / RadioItem / Shortcut
- `shadcn-syntax-popover-tooltip-hovercard` : decision : Popover (click), Tooltip (hover, short text), HoverCard (hover, rich content), delay-duration / openDelay / closeDelay
- `shadcn-syntax-navigation-menu` : NavigationMenu primitive, NavigationMenuList / Item / Trigger / Content / Link / Indicator / Viewport
- `shadcn-syntax-context-menu` : right-click pattern, ContextMenu primitive, same API surface as DropdownMenu
- `shadcn-syntax-resizable` : react-resizable-panels, ResizablePanelGroup / Panel / Handle, direction (horizontal / vertical), defaultSize / minSize / maxSize
- `shadcn-syntax-variant-cva` : cva() primitive : base classes + variants + compoundVariants + defaultVariants, VariantProps<typeof X> type, when to use over inline conditional classes

### Implementation (end-to-end workflows) : ~6 skills

- `shadcn-impl-component-install` : when to add vs build custom, components.json config setup, aliases (`@/components/ui`), path-resolution, post-add manual fixups
- `shadcn-impl-form-validation` : end-to-end form with react-hook-form + zod schema + Form component + submit handler + error rendering
- `shadcn-impl-data-table-build` : end-to-end DataTable with ColumnDef + useReactTable + sort / filter / paginate / row-select / column-toggle
- `shadcn-impl-theming-custom` : build custom theme via CSS-var override, dark-mode toggle (next-themes pattern), theme-builder workflow
- `shadcn-impl-blocks` : use shadcn.com/blocks pre-built compositions (sidebar / dashboard / authentication / login), add via CLI, customize after copy
- `shadcn-impl-framework-integration` : Vite vs Next.js (app router + pages router) vs Remix vs Astro vs TanStack Start : init differences, alias setup per bundler

### Errors (anti-patterns + debugging) : ~5 skills

- `shadcn-errors-cli-sync-mismatch` : out-of-sync components after CLI update overwrites local customizations, diff workflow (`shadcn diff <component>`), merge strategy
- `shadcn-errors-styling-conflicts` : tailwind-merge vs class order, cva variant override pitfalls, className prop merging via `cn()` helper
- `shadcn-errors-radix-controlled` : Radix controlled-state pitfalls (open prop without onOpenChange = read-only), Portal-rendering surprises, asChild-with-child-mismatch
- `shadcn-errors-form-state` : react-hook-form + Form component mistakes : Controller vs register, Watch over-rendering, zod async validation, defaultValues vs values
- `shadcn-errors-theming-tokens` : CSS-var format (HSL space-separated, NOT comma), Tailwind config-side referencing pattern, oklch upgrade path (v4), --background usage in `bg-background` class

### Agents (validators + orchestrators) : ~3 skills

- `shadcn-agents-component-selector` : decision-tree validator : Dialog vs Sheet vs Drawer, Select vs Combobox vs Command, DropdownMenu vs ContextMenu, Tooltip vs HoverCard vs Popover
- `shadcn-agents-cva-validator` : variant API correctness checker : variant definition shape, compoundVariants order, VariantProps inference, ALWAYS-cn-merge rule
- `shadcn-agents-form-validator` : react-hook-form + zod + shadcn Form integration validator : schema-to-Form-mapping, Controller-correctness, error-display-completeness

## Estimated Skill Count

| Category | Estimated | Notes |
|----------|-----------|-------|
| core | 5 | architecture, CLI, stack, theming, registry |
| syntax | 14 | per major component or component-cluster |
| impl | 6 | workflows + framework integration |
| errors | 5 | CLI sync, styling, controlled state, form state, theming tokens |
| agents | 3 | selector + cva + form validators |
| **Total** | **~33** | refinement in Phase 3 may merge / drop / split |

## Cross-package companion skills (Read-on-demand, NEVER auto-load)

- React skill package : reference for component patterns, hooks, controlled state
- Tailwind CSS skill package : reference for utility classes, custom-properties, dark-mode, v3 -> v4 migration

## Next : Phase 2 Deep Research

After this raw masterplan is committed, dispatch single opus research-agent. Output : `docs/research/vooronderzoek-shadcn.md` (min 2000 words, WebFetch-verified against SOURCES.md primary URLs, Last-Verified dates updated).

Refinement (Phase 3) will revise this inventory based on research findings : expect at least 1 MERGE / DROP / SPLIT decision (BOOTSTRAP §5.1 hard requirement).
