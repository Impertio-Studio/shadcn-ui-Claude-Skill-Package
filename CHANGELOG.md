# Changelog

All notable changes to the shadcn ui Skill Package.

Format follows [Keep a Changelog](https://keepachangelog.com/).

## [1.0.0] - 2026-05-19

### Added
- 42 deterministic Claude skills for shadcn ui evergreen-2026
  - 6 core skills : architecture, CLI, stack, theming, registry, blocks
  - 18 syntax skills : variant-cva, button, form, field, dialog, sheet, drawer, selectors, command, menu-primitives, popover-tooltip-hovercard, toast-sonner, table, chart, calendar-datepicker, sidebar, layout-primitives, input-otp
  - 7 impl skills : component-install, form-validation, data-table, theming-custom, responsive-dialog-drawer, rsc-vs-client-boundaries, framework-integration
  - 7 errors skills : cli-sync-mismatch, styling-conflicts, radix-controlled, form-state, tailwind-v3-v4-migration, react-day-picker-v9, cmdk-version-drift
  - 4 agents skills : component-selector, cva-validator, form-validator, rsc-boundary-validator
- All SKILL.md files under 500 lines, YAML folded scalar, "Use when..." opener, Keywords-regel with technical + symptom-based + plain-language terms
- 3 reference files per skill (methods.md, examples.md, anti-patterns.md)
- 100% compliance audit score
- Discovery manifests : package.json agents.skills[], agents/openai.yaml
- vooronderzoek-shadcn.md research document (5613 words, 59 components catalogued)
- 20 Refinement Decisions documented in masterplan
- 10 Architectural Decisions in DECISIONS.md

### Methodology
- 7-phase research-first development (per OpenAEC Foundation methodology)
- WebFetch-verified against ui.shadcn.com/docs, github.com/shadcn-ui/ui, radix-ui.com/primitives, cva.style/docs, react-day-picker.js.org, cmdk.paco.me

## [Unreleased]
