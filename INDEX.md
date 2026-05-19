# INDEX : shadcn ui Skill Package Catalog

## Overview

42 deterministic Claude skills across 5 categories for shadcn ui evergreen-2026 (CLI shadcn@4.7.0). Built on the [Agent Skills](https://agentskills.org) open standard.

## Summary

| Category | Skills | Focus |
|----------|:------:|-------|
| **core/** | 6 | Architecture, CLI, stack, theming, registry, blocks |
| **syntax/** | 18 | Per-component API patterns (variants, primitives, composition) |
| **impl/** | 7 | End-to-end workflows (install, forms, data-table, themes, RSC, frameworks) |
| **errors/** | 7 | Anti-patterns + debugging (CLI sync, styling, controlled state, version drift) |
| **agents/** | 4 | Validators (component selector, cva, form, RSC boundary) |
| **Total** | **42** | |

## core (6)

| Skill | Description |
|-------|-------------|
| `shadcn-core-architecture` | Use when starting a new project that needs UI components, evaluating shadcn ui versus a traditional component library (MUI, Chakra, Mantine,... |
| `shadcn-core-blocks` | Use when scaffolding a full page (dashboard, login, signup, sidebar layout, calendar app), bootstrapping a starter template from shadcn's ga... |
| `shadcn-core-cli` | Use when running any shadcn CLI command (init, add, view, search, apply, preset, build, docs, info, migrate), when writing or editing compon... |
| `shadcn-core-registry` | Use when authoring or consuming a shadcn registry, when configuring the `registries` field in components.json, when wiring a private or paid... |
| `shadcn-core-stack` | Use when reading or writing any shadcn ui component, especially when figuring out which library owns which concern (a11y, variants, class co... |
| `shadcn-core-theming` | Use when wiring shadcn ui colors, dark mode, the design-token palette, or migrating a project between Tailwind v3 and Tailwind v4, or when ... |

## syntax (18)

| Skill | Description |
|-------|-------------|
| `shadcn-syntax-button` | Use when adding, customising, or debugging a shadcn ui Button, choosing between the six built-in variants (default / destructive / outline /... |
| `shadcn-syntax-calendar-datepicker` | Use when adding, customising, or debugging a shadcn ui Calendar or Date Picker, choosing between the three react-day-picker v9 selection ... |
| `shadcn-syntax-chart` | Use when building data visualizations with the shadcn Chart primitive that wraps Recharts (Bar, Line, Area, Pie, Radar, Scatter, RadialBar, ... |
| `shadcn-syntax-command` | Use when adding, debugging, or refactoring a shadcn ui Command primitive (the cmdk-backed command-palette / fuzzy-filter list), composing it... |
| `shadcn-syntax-dialog` | Use when adding, debugging, or refactoring a shadcn ui Dialog (a focus- trapped modal that overlays the page), composing its ten primitives ... |
| `shadcn-syntax-drawer` | Use when building a mobile-first bottom-sheet, swipe-to-dismiss panel, multi-stop drawer (snap points), or any sliding surface that enters f... |
| `shadcn-syntax-field` | Use when building forms in shadcn ui evergreen-2026 and the new Field primitive family is the recommended path, when composing Field, FieldL... |
| `shadcn-syntax-form` | Use when building forms with shadcn ui and react-hook-form plus zod, when composing the seven Form primitives (Form, FormField, FormItem, Fo... |
| `shadcn-syntax-input-otp` | Use when building a one-time-password, verification-code, MFA, two-factor, SMS-code, or PIN-entry input in shadcn ui evergreen-2026, when co... |
| `shadcn-syntax-layout-primitives` | Use when adding, debugging, or refactoring any of the four shadcn ui layout-helper primitives : Resizable (drag-to-resize split panes), ... |
| `shadcn-syntax-menu-primitives` | Use when adding, debugging, or refactoring any shadcn ui menu surface : DropdownMenu (button-triggered popup), ContextMenu (right-click on a... |
| `shadcn-syntax-popover-tooltip-hovercard` | Use when picking between Popover, Tooltip, and HoverCard for any floating panel above the page : a click-to-open form panel, a short hover h... |
| `shadcn-syntax-selectors` | Use when picking the right "selector" primitive in a shadcn ui app and you are unsure whether to reach for a Native HTML `<select>`, ... |
| `shadcn-syntax-sheet` | Use when building a side panel, slide-in drawer, off-canvas navigation, filter panel, settings tray, or any container that enters from the t... |
| `shadcn-syntax-sidebar` | Use when building an application shell, dashboard navigation, admin sidebar, collapsible side navigation, or any persistent vertical navigat... |
| `shadcn-syntax-table` | Use when adding, customising, or debugging the shadcn ui Table primitive (the eight subcomponents Table, TableHeader, TableBody, TableFooter... |
| `shadcn-syntax-toast-sonner` | Use when adding, debugging, or migrating notification toasts in a shadcn ui project, mounting the top-level `<Toaster />` in your app ... |
| `shadcn-syntax-variant-cva` | Use when defining or modifying variants on a shadcn ui component, writing a new component that needs `variant` / `size` props, extending the... |

## impl (7)

| Skill | Description |
|-------|-------------|
| `shadcn-impl-component-install` | Use when adding a new component to a project that already uses shadcn, when bootstrapping shadcn for the first time in a fresh app, when dec... |
| `shadcn-impl-data-table` | Use when building any sortable, filterable, paginated, selectable, or column-toggleable table in shadcn ui : the official shadcn DataTable i... |
| `shadcn-impl-form-validation` | Use when building an end-to-end form workflow in shadcn ui that combines the Form composition primitives with react-hook-form and zod, when ... |
| `shadcn-impl-framework-integration` | Use when initializing shadcn ui inside a Vite, Next.js (App or Pages Router), React Router v7 (former Remix), Astro, or TanStack Start proje... |
| `shadcn-impl-responsive-dialog-drawer` | Use when building a modal surface that must look correct on both desktop and mobile in shadcn ui, when the design calls for a centered Dialo... |
| `shadcn-impl-rsc-vs-client-boundaries` | Use when wiring shadcn components into a Next.js App Router project, when deciding whether a given shadcn primitive needs a `"use client"` d... |
| `shadcn-impl-theming-custom` | Use when building or customizing a theme in a shadcn ui project, wiring the dark mode toggle, replacing the default primary color ... |

## errors (7)

| Skill | Description |
|-------|-------------|
| `shadcn-errors-cli-sync-mismatch` | Use when re-running `shadcn add <component>` would overwrite a file in `components/ui/` that you have edited locally, when a teammate report... |
| `shadcn-errors-cmdk-version-drift` | Use when a shadcn ui Command primitive (command palette, Combobox, search list, fuzzy filter, multi-select dropdown) stops filtering, render... |
| `shadcn-errors-form-state` | Use when a shadcn Form built on react-hook-form silently submits with missing values, when a Radix Select or Checkbox or RadioGroup or Switc... |
| `shadcn-errors-radix-controlled` | Use when a shadcn Dialog / Sheet / Drawer / DropdownMenu / Popover / HoverCard / Select / AlertDialog / ContextMenu / Collapsible / ... |
| `shadcn-errors-react-day-picker-v9` | Use when a shadcn `Calendar` (or any direct `react-day-picker` DayPicker) silently renders nothing, throws a TypeScript error on the `select... |
| `shadcn-errors-styling-conflicts` | Use when a className override on a shadcn component does not visually take effect, when two Tailwind utility classes appear in the rendered ... |
| `shadcn-errors-tailwind-v3-v4-migration` | Use when a shadcn ui project must move from Tailwind CSS v3 to v4, when colors break or look washed out after a Tailwind upgrade, when the d... |

## agents (4)

| Skill | Description |
|-------|-------------|
| `shadcn-agents-component-selector` | Use when a user describes a UI requirement in plain English ("a modal to edit a record", "a searchable dropdown of countries", "show a non-b... |
| `shadcn-agents-cva-validator` | Use when reviewing, validating, or auditing a shadcn ui component that uses class-variance-authority (cva), when an LLM has just produced a ... |
| `shadcn-agents-form-validator` | Use when validating a react-hook-form + zod + shadcn Form integration in a code review, a draft AI-generated form, or a bug report ("my vali... |
| `shadcn-agents-rsc-boundary-validator` | Use when auditing a Next.js App Router codebase that consumes shadcn ui for misplaced or missing `"use client"` directives, when reviewing a... |

## Dependency Graph

```
core (foundation, 6 skills)
  -> syntax (component API, 18 skills)
       -> impl (workflows, 7 skills)
            -> errors (anti-patterns, 7 skills)
                 -> agents (validators, 4 skills)
```

Skills cross-reference via `### Companion Skills` blocks (Read-on-demand, NEVER auto-load).

## Discovery

- npm-agentskills manifest : [package.json](package.json) `agents.skills[]`
- OpenAI Codex : [agents/openai.yaml](agents/openai.yaml)
- GitHub topic : `agentskills`

## Quality Guarantees

- All 42 SKILL.md files under 500 lines
- YAML folded scalar `>` description with "Use when..." opener
- Keywords-regel mixing technical + symptom-based + plain-language terms (8+ per skill)
- No em-dashes in section headings (typografie-regel)
- English-only deterministic ALWAYS/NEVER language
- All code-examples WebFetch-verified against SOURCES.md primary URLs
- Compliance audit score: 100% (see [docs/validation/audit-report.md](docs/validation/audit-report.md))
