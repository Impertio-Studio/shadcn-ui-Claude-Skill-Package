# Masterplan : shadcn ui

> Status : Phase 3 refined (definitive, post-vooronderzoek)
> Generated : 2026-05-19
> Methodology : 7-phase research-first (BOOTSTRAP-RUNBOOK §5)

## Scope

- Technology : shadcn ui
- Versions : evergreen-2026 (canary), CLI `shadcn@4.7.0` (2026-05-05)
- Languages : TypeScript, React (18+ / 19), Tailwind CSS (v3.4 / v4)
- Prefix : shadcn
- License : MIT
- Description : Deterministic Claude skills for shadcn ui copy-not-install paradigm, full component catalog (59 components), variant API (cva), CSS-token theming (HSL v3 / oklch v4), composition patterns, framework integration, and a13y-correct Radix usage.

## Paradigm note (critical for every skill)

shadcn ui is NOT a library. Consumers run `npx shadcn@latest add <component>` and the CLI copies TypeScript source into `components/ui/` inside the consumer project. There is no runtime dependency on shadcn itself : only on Radix UI primitives, cva, tailwind-merge, clsx, and Tailwind CSS. Every skill MUST respect this ownership model and the AI-Ready pillar : Claude reads the local copy, modifies it, and reasons over it without indirection.

## Refinement Decisions (post-research, hard requirements per BOOTSTRAP §5.1)

| ID | Decision | Reden | Bron |
|----|----------|-------|------|
| RD-01 | ADD `shadcn-core-blocks` as separate skill (was bullet in raw plan) | Blocks are a distinct distribution surface with own registry. Three new style additions (default / new-york / sera / luma) per changelog 2026-Q2. | vooronderzoek §1 + §4 |
| RD-02 | ADD `shadcn-syntax-field` as NEW skill (not in raw plan) | Field primitive added in 2026 as a13y foundation underpinning both react-hook-form and TanStack Form Form integrations. Now sits beneath Form composition. | vooronderzoek §2 + §6 |
| RD-03 | ADD `shadcn-syntax-sidebar` as standalone skill (not in raw plan) | Sidebar has 16 subcomponents and is the most complex single shadcn component. Folding into a multi-component skill would breach the 500-line limit. | vooronderzoek §2 |
| RD-04 | ADD `shadcn-syntax-chart` (not in raw plan) | Chart is a first-class shadcn primitive wrapping Recharts with own ChartContainer / ChartTooltip / ChartLegend wrappers. Distinct from any general component pattern. | vooronderzoek §2 |
| RD-05 | ADD `shadcn-syntax-calendar-datepicker` (not in raw plan) | react-day-picker v9 introduced breaking changes that surface as top-15 issues. Calendar warrants standalone skill. | vooronderzoek §3 + §9 |
| RD-06 | ADD `shadcn-syntax-drawer` (not in raw plan) | Drawer is Vaul-backed (separate from Dialog) with mobile-first semantics. Required for responsive-dialog-drawer impl pattern. | vooronderzoek §2 |
| RD-07 | ADD `shadcn-syntax-input-otp` (not in raw plan) | input-otp is a separate primitive with own slot API. | vooronderzoek §2 |
| RD-08 | SPLIT `shadcn-syntax-data-table` -> `shadcn-syntax-table` (HTML primitive) + `shadcn-impl-data-table` (TanStack recipe) | Conceptual mismatch : Table is a styling primitive, DataTable is a TanStack Table v8 integration recipe. Different audiences, different content. | vooronderzoek §2 + §7 |
| RD-09 | MERGE `shadcn-syntax-context-menu` into `shadcn-syntax-menu-primitives` (with DropdownMenu, Menubar, NavigationMenu) | All four share the Radix Menu primitive surface. Decision-tree fits one skill. | vooronderzoek §2 |
| RD-10 | MERGE `shadcn-syntax-resizable` into `shadcn-syntax-layout-primitives` (with ScrollArea, Separator, AspectRatio) | All four are layout-helper primitives without independent decision logic. | vooronderzoek §2 |
| RD-11 | RENAME `shadcn-syntax-select-combobox` -> `shadcn-syntax-selectors` (add Native Select + Command-as-palette decision-tree) | Selector-choice is the actual decision skill; just two-name made it look smaller than it is. | vooronderzoek §2 |
| RD-12 | ADD `shadcn-impl-responsive-dialog-drawer` (not in raw plan) | Dialog-on-desktop / Drawer-on-mobile is the documented official pattern. Distinct from each individual syntax skill. | vooronderzoek §2 + §11 |
| RD-13 | ADD `shadcn-impl-rsc-vs-client-boundaries` (not in raw plan) | RSC boundary rules differ per primitive (Card server-safe, Dialog client-only). Single biggest source of mid-build errors per top-15 issues. | vooronderzoek §11 |
| RD-14 | ADD `shadcn-errors-tailwind-v3-v4-migration` (not in raw plan) | Tailwind v3 (HSL space-separated, tailwind.config.js) -> v4 (oklch, @theme inline, single CSS file) is the single largest hallucination risk per AI-coding logs. | vooronderzoek §3 + §8 |
| RD-15 | ADD `shadcn-errors-react-day-picker-v9` (not in raw plan) | Calendar v9 breaking change blocks consumers; documented as standalone error skill. | vooronderzoek §9 + §10 |
| RD-16 | ADD `shadcn-errors-cmdk-version-drift` (not in raw plan) | Three top-15 issues map to cmdk drift (filter prop, value normalization, Vaul interaction). Dedicated skill prevents conflation with general styling-conflicts. | vooronderzoek §10 |
| RD-17 | ADD `shadcn-agents-rsc-boundary-validator` (not in raw plan) | Validates the rsc:true components.json flag against per-component "use client" requirements; pairs with impl-rsc-vs-client-boundaries. | vooronderzoek §11 |
| RD-18 | DROP `shadcn-errors-fast-refresh-lint` from candidate list | Niche, low-frequency. Skip per BOOTSTRAP §3.5 scope-discipline. | scope discipline |
| RD-19 | KEEP toast as `shadcn-syntax-toast-sonner` (NOT `shadcn-syntax-toast`) | Toast component REMOVED per changelog ; Sonner is sole toast primitive. Naming the skill `toast-sonner` makes the migration-target explicit. | vooronderzoek §2 changelog |
| RD-20 | ALL styling-related skills MUST scope per Tailwind generation (v3 vs v4) explicitly | Single biggest hallucination risk per top-15 issues. Cross-cutting rule. | vooronderzoek §3 + §8 |

## Final Skill Inventory (42 skills)

### Core (6 skills) : architecture + cross-cutting

1. `shadcn-core-architecture` : copy-not-install paradigm, ownership model, AI-Ready pillar, evergreen versioning
2. `shadcn-core-cli` : init / add / diff / update / build / migrate, --overwrite vs --diff flags, registry resolution
3. `shadcn-core-stack` : Radix + cva + tailwind-merge + clsx + lucide composition map, what each layer owns
4. `shadcn-core-theming` : CSS custom properties, HSL space-separated (v3) vs oklch (v4), --background/foreground/primary/etc exhaustive list, dark mode toggle
5. `shadcn-core-registry` : components.json full schema, default vs custom registries, registries namespace, URL templates + env vars + headers
6. `shadcn-core-blocks` : blocks distribution surface, style enum (default / new-york / sera / luma), block-add workflow

### Syntax (18 skills) : per-component or per-cluster API patterns

7. `shadcn-syntax-variant-cva` : cva(base, { variants, compoundVariants, defaultVariants }), VariantProps inference, `cn()` ALWAYS-rule
8. `shadcn-syntax-button` : variant (default / destructive / outline / secondary / ghost / link), size (default / sm / lg / icon), asChild Slot pattern
9. `shadcn-syntax-form` : Form / FormField / FormItem / FormLabel / FormControl / FormDescription / FormMessage, zodResolver, Controller-vs-register decision
10. `shadcn-syntax-field` : Field a13y primitive (NEW 2026), FieldGroup / FieldLabel / FieldDescription / FieldError, sits beneath Form composition
11. `shadcn-syntax-dialog` : Radix Dialog (Trigger / Portal / Overlay / Content / Header / Footer / Title / Description / Close), controlled vs uncontrolled
12. `shadcn-syntax-sheet` : Dialog-derivative with `side` prop, when Sheet over Dialog (side-panel UX, larger content)
13. `shadcn-syntax-drawer` : Vaul-backed, mobile-first, when Drawer over Sheet (mobile-bottom-sheet semantics)
14. `shadcn-syntax-selectors` : Select (Radix) vs Combobox (Popover + Command) vs Native vs Command-as-palette decision-tree
15. `shadcn-syntax-command` : cmdk underneath, CommandDialog wrapping, CommandInput/List/Empty/Group/Item, async filtering
16. `shadcn-syntax-menu-primitives` : DropdownMenu + ContextMenu + Menubar + NavigationMenu shared API surface + when-which decision-tree
17. `shadcn-syntax-popover-tooltip-hovercard` : decision : Popover (click), Tooltip (hover short), HoverCard (hover rich), delay-duration prop
18. `shadcn-syntax-toast-sonner` : Sonner-only (Toast REMOVED), toast() / toast.success / toast.error / toast.promise patterns, Toaster placement
19. `shadcn-syntax-table` : HTML table primitive, Table / TableHeader / TableBody / TableRow / TableCell composition (styling-only)
20. `shadcn-syntax-chart` : ChartContainer / ChartTooltip / ChartLegend wrappers over Recharts, config-via-CSS-vars pattern
21. `shadcn-syntax-calendar-datepicker` : react-day-picker v9 API, Calendar + Popover composition for DatePicker recipe
22. `shadcn-syntax-sidebar` : SidebarProvider / Sidebar / SidebarTrigger / SidebarMenu / 16-subcomponent surface, collapsible variants
23. `shadcn-syntax-layout-primitives` : Resizable + ScrollArea + Separator + AspectRatio layout helpers
24. `shadcn-syntax-input-otp` : input-otp primitive, InputOTP / InputOTPGroup / InputOTPSlot / InputOTPSeparator

### Implementation (7 skills) : end-to-end workflows

25. `shadcn-impl-component-install` : when add vs custom, components.json full setup, aliases (`@/components/ui` / `@/lib/utils` / etc), `package.json#imports` alternative, post-add manual fixups
26. `shadcn-impl-form-validation` : end-to-end form (zod schema + Form composition + zodResolver + submit handler + error rendering + async validation)
27. `shadcn-impl-data-table` : TanStack Table v8 recipe (ColumnDef + useReactTable + sort / filter / paginate / row-select / column-visibility)
28. `shadcn-impl-theming-custom` : custom theme via CSS-var override, next-themes integration, theme-builder workflow, light / dark / system modes
29. `shadcn-impl-responsive-dialog-drawer` : Dialog-on-desktop / Drawer-on-mobile pattern, useMediaQuery hook, shared content component
30. `shadcn-impl-rsc-vs-client-boundaries` : RSC-safe primitives vs client-required primitives, `'use client'` directive placement, components.json `rsc:true` flag
31. `shadcn-impl-framework-integration` : Vite vs Next.js (app + pages routers) vs Remix vs Astro vs TanStack Start init differences, alias config per bundler

### Errors (7 skills) : anti-patterns + debugging

32. `shadcn-errors-cli-sync-mismatch` : CLI update overwriting local customizations, `shadcn diff <component>` workflow, merge strategy, fork-vs-vendor decision
33. `shadcn-errors-styling-conflicts` : tailwind-merge vs class order, cva variant override pitfalls, `cn()` helper ALWAYS-rule, raw className-concat antipattern
34. `shadcn-errors-radix-controlled` : controlled-state pitfalls (open prop without onOpenChange = read-only), Portal z-index surprises, asChild misuse, Slot single-child requirement
35. `shadcn-errors-form-state` : react-hook-form + Form mistakes (Controller vs register, Watch over-rendering, zod async validation, defaultValues vs values)
36. `shadcn-errors-tailwind-v3-v4-migration` : v3 (HSL space-separated, tailwind.config.js) -> v4 (oklch, @theme inline, single CSS file), `bg-background` class semantics shift
37. `shadcn-errors-react-day-picker-v9` : Calendar v9 breaking changes (renamed props, new style hooks, custom-day-renderer API)
38. `shadcn-errors-cmdk-version-drift` : 3 top-15 issues (filter prop, value normalization, Vaul + cmdk interaction)

### Agents (4 skills) : validators + orchestrators

39. `shadcn-agents-component-selector` : decision-tree validator : Dialog vs Sheet vs Drawer, Select vs Combobox vs Command, DropdownMenu vs ContextMenu vs Menubar, Tooltip vs HoverCard vs Popover
40. `shadcn-agents-cva-validator` : variant API correctness checker (variant shape, compoundVariants order, VariantProps inference, ALWAYS-cn-merge)
41. `shadcn-agents-form-validator` : react-hook-form + zod + shadcn Form integration validator (schema-to-Form mapping, Controller-correctness, error-display completeness)
42. `shadcn-agents-rsc-boundary-validator` : validates `components.json` `rsc:true` flag against per-component "use client" requirements, flags missing 'use client' directives

## Execution Plan : Batches

Dependency chain : core -> syntax -> impl -> errors -> agents. Batch-size = 3 (Claude Code Agent tool optimum + tmux-orchestration default). File-scope per batch : NEVER two skills writing to the same path.

| Batch | Skills | Cat-mix | File-scope | Dependencies | Est. duration |
|-------|--------|---------|------------|--------------|---------------|
| B1 | shadcn-core-architecture, shadcn-core-stack, shadcn-core-cli | core | skills/source/shadcn-core/* | none | ~20 min |
| B2 | shadcn-core-theming, shadcn-core-registry, shadcn-core-blocks | core | skills/source/shadcn-core/* | B1 | ~20 min |
| B3 | shadcn-syntax-variant-cva, shadcn-syntax-button, shadcn-syntax-form | syntax | skills/source/shadcn-syntax/* | B1-2 | ~25 min |
| B4 | shadcn-syntax-field, shadcn-syntax-dialog, shadcn-syntax-sheet | syntax | skills/source/shadcn-syntax/* | B3 | ~25 min |
| B5 | shadcn-syntax-drawer, shadcn-syntax-selectors, shadcn-syntax-command | syntax | skills/source/shadcn-syntax/* | B3 | ~25 min |
| B6 | shadcn-syntax-menu-primitives, shadcn-syntax-popover-tooltip-hovercard, shadcn-syntax-toast-sonner | syntax | skills/source/shadcn-syntax/* | B3 | ~25 min |
| B7 | shadcn-syntax-table, shadcn-syntax-chart, shadcn-syntax-calendar-datepicker | syntax | skills/source/shadcn-syntax/* | B3 | ~25 min |
| B8 | shadcn-syntax-sidebar, shadcn-syntax-layout-primitives, shadcn-syntax-input-otp | syntax | skills/source/shadcn-syntax/* | B3 | ~25 min |
| B9 | shadcn-impl-component-install, shadcn-impl-form-validation, shadcn-impl-data-table | impl | skills/source/shadcn-impl/* | B3-8 | ~30 min |
| B10 | shadcn-impl-theming-custom, shadcn-impl-responsive-dialog-drawer, shadcn-impl-rsc-vs-client-boundaries | impl | skills/source/shadcn-impl/* | B2,4,5 | ~30 min |
| B11 | shadcn-impl-framework-integration, shadcn-errors-cli-sync-mismatch, shadcn-errors-styling-conflicts | impl+errors | skills/source/shadcn-impl/* + skills/source/shadcn-errors/* | B2,3 | ~30 min |
| B12 | shadcn-errors-radix-controlled, shadcn-errors-form-state, shadcn-errors-tailwind-v3-v4-migration | errors | skills/source/shadcn-errors/* | B4,3,2 | ~30 min |
| B13 | shadcn-errors-react-day-picker-v9, shadcn-errors-cmdk-version-drift, shadcn-agents-component-selector | errors+agents | skills/source/shadcn-errors/* + skills/source/shadcn-agents/* | B7,5,ALL-syntax | ~30 min |
| B14 | shadcn-agents-cva-validator, shadcn-agents-form-validator, shadcn-agents-rsc-boundary-validator | agents | skills/source/shadcn-agents/* | ALL | ~30 min |

**Total : 14 batches, 42 skills, ~6 hours wall-clock via tmux-orchestration (3 workers parallel + topic-research overhead + quality-gate loop).**

## Per-Skill Agent Prompts

Each worker receives a prompt-block of the following shape. Sample below for `shadcn-syntax-button`; the other 41 prompt-blocks share this structure and are JIT-generated from the template before each batch dispatches (template + per-skill scope-bullets stored in `docs/masterplan/prompts/{batch-N}/{skill-name}.md`).

### Agent-Prompt TEMPLATE (used per skill)

```
Workspace : /home/freek/GitHub/shadcn-ui-Claude-Skill-Package/
Output dir : skills/source/shadcn-{CATEGORY}/shadcn-{CATEGORY}-{TOPIC}/
Files to create :
  - SKILL.md (max 500 lines, YAML folded scalar `>`, "Use when..." opener, Keywords-regel met technische + symptom-based + plain-language termen)
  - references/methods.md (complete API signatures)
  - references/examples.md (working code examples, version-explicit)
  - references/anti-patterns.md (anti-patterns + "WHY this fails")

Research input :
  - docs/research/vooronderzoek-shadcn.md sections {RELEVANT_SECTIONS}
  - Topic research : docs/research/topic-research/shadcn-{CATEGORY}-{TOPIC}-research.md (Phase 4 output)
  - Approved sources : SOURCES.md primary URLs only

Scope (deterministic bullets) :
  - {scope-bullet-1}
  - {scope-bullet-2}
  - ...

Cross-pkg refs (Companion Skills, Read-on-demand only) :
  - React skill package : {when relevant}
  - Tailwind skill package : {when relevant}

Quality rules (BOOTSTRAP §6.2 skill-builder role) :
  - English-only, deterministic language (ALWAYS X / NEVER Y), NO "you might want to"
  - YAML frontmatter : folded scalar `>`, "Use when..." opener, license:MIT, compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026.", metadata.author: OpenAEC-Foundation, version: "1.0"
  - Keywords-regel : technische termen + symptom-based termen ("nothing shows", "click does nothing") + plain-language termen ("how do I", "what is")
  - License: MIT in frontmatter
  - Section headings : use `:` separator, NEVER em-dash `—`
  - Tailwind scope per generation : code-examples explicitly annotated `v3` or `v4`
  - WebFetch verify all code-snippets against SOURCES.md primary URLs
  - SKILL.md < 500 lines (overflow to references/)
  - 3 reference files: methods.md, examples.md, anti-patterns.md (ALL mandatory)
  - Commit per skill : `feat(skill): shadcn-{CATEGORY}-{TOPIC}`

Quality gate after completion :
  - validate-frontmatter.js exit 0
  - validate-language.js exit 0
  - validate-line-count.js exit 0
  - validate-structure.js exit 0
  - validate-emdash.js exit 0
```

### Sample completed prompt (B3, skill 2 of 3) : shadcn-syntax-button

```
Workspace : /home/freek/GitHub/shadcn-ui-Claude-Skill-Package/
Output dir : skills/source/shadcn-syntax/shadcn-syntax-button/
Files to create :
  - SKILL.md
  - references/methods.md
  - references/examples.md
  - references/anti-patterns.md

Research input :
  - docs/research/vooronderzoek-shadcn.md sections §2 (Component catalog : Button entry), §5 (cva variant API), §11 (RSC compat)
  - Topic research : docs/research/topic-research/shadcn-syntax-button-research.md (Phase 4 output)
  - Approved sources : https://ui.shadcn.com/docs/components/radix/button, https://cva.style/docs, https://github.com/dcastil/tailwind-merge

Scope (deterministic bullets) :
  - Variant API : default / destructive / outline / secondary / ghost / link via cva
  - Size variants : default / sm / lg / icon
  - asChild Slot pattern : when to use (e.g. wrapping <Link/> from Next.js)
  - asChild single-child requirement + common errors when multiple children passed
  - VariantProps<typeof buttonVariants> typing pattern
  - ALWAYS use `cn()` for className override, NEVER raw template-string concat
  - Tailwind v3 vs v4 ring-utility shift (`focus-visible:ring-2` -> `focus-visible:ring-3` per v4 docs)

Cross-pkg refs :
  - React skill package : asChild Slot pattern relates to React component composition
  - Tailwind skill package : ring + focus-visible utilities

Quality rules : (per TEMPLATE above)

Quality gate after completion : (per TEMPLATE above)
```

## Cross-package companion skills (Read-on-demand, NEVER auto-load)

- **React skill package** at /home/freek/GitHub/React-Claude-Skill-Package : reference for hooks, controlled state, Server Components, asChild Slot semantics
- **Tailwind CSS skill package** at /home/freek/GitHub/TailwindCSS-Claude-Skill-Package : reference for utility classes, custom-properties, dark-mode, v3 -> v4 migration (note: Tailwind pkg currently empty per skill-radar, fallback to tailwindcss.com/docs WebFetch)

## Cross-cutting hard rules (apply to EVERY skill)

1. **Tailwind generation scoping** : every code-example explicitly annotated `v3` or `v4`. Skills that touch styling MUST cover both.
2. **`cn()` ALWAYS rule** : ALWAYS use `cn(...)` helper for className composition. NEVER raw template-string or string-concat for class merging.
3. **Toast removal** : `useToast` hook is REMOVED. Skills mentioning toast MUST use Sonner (`shadcn-syntax-toast-sonner`).
4. **RSC discipline** : every component-skill annotates whether the component is RSC-safe or client-only. Cross-references `shadcn-impl-rsc-vs-client-boundaries`.
5. **Em-dash ban** : section headings use `:` separator, NEVER `—`. Run `validate-emdash.js` after every batch.
6. **WebFetch verification** : every API claim cites an approved SOURCES.md URL. NO training-data assumptions.

## Phase 5 execution model (tmux-orchestration)

- 3 workers spawned in VS Code panels (`worker-1`, `worker-2`, `worker-3`)
- Custom role : `skill-builder` (BOOTSTRAP §6.2)
- QG cadence : every worker reply
- Reply language : Nederlands
- State dir : /home/freek/GitHub/shadcn-ui-Claude-Skill-Package/state/
- File-scope per worker : SET PER BATCH (re-instruct on batch transition, do NOT re-spawn)
- Per batch flow :
  1. Generate 3 topic-research files in parallel via 3 in-process opus agents
  2. Verify topic-research files exist
  3. Bundle 3 prompt-blocks (one per worker) via masterplan + topic-research reference
  4. Inject via `tmo task add` + `tmo send`
  5. QG loop : APPROVE / RE-INSTRUCT / REPLACE per worker
  6. Wait all 3 workers status=done
  7. Run validators on batch-output
  8. Commit batch : `feat(phase-5): batch-{N} skills [{A},{B},{C}]`
  9. Update ROADMAP.md + HANDOFF.md
  10. Reflection checkpoint (BOOTSTRAP §6.4.10)

## Next : User checkpoint (BOOTSTRAP §5.4)

Show batch-tabel + decisions-tabel + sample agent-prompt above. WAIT for user :
- `ja` -> proceed to Phase 4+5 via tmux-orchestration
- `fix: <correctie>` -> apply correction, re-checkpoint
- `pauze` -> commit Phase 3, stop, resume next session
