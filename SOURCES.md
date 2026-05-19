# Sources : shadcn ui Skill Package

## Approved Sources

All skill content MUST be verified against these approved sources. No unverified blog posts or AI-generated content.

### Primary Sources

| Source | URL | Type | Last Verified |
|--------|-----|------|---------------|
| shadcn ui Docs | https://ui.shadcn.com/docs | Official Documentation | 2026-05-19 |
| shadcn ui Components | https://ui.shadcn.com/docs/components | Official Documentation (per-component) | 2026-05-19 |
| shadcn ui components.json | https://ui.shadcn.com/docs/components-json | Official Documentation (schema) | 2026-05-19 |
| shadcn ui Theming | https://ui.shadcn.com/docs/theming | Official Documentation | 2026-05-19 |
| shadcn ui Dark Mode | https://ui.shadcn.com/docs/dark-mode | Official Documentation | 2026-05-19 |
| shadcn ui Tailwind v4 | https://ui.shadcn.com/docs/tailwind-v4 | Official Documentation (migration guide) | 2026-05-19 |
| shadcn ui Changelog | https://ui.shadcn.com/docs/changelog | Official Documentation (version history) | 2026-05-19 |
| shadcn ui Installation | https://ui.shadcn.com/docs/installation | Official Documentation (per-framework setup) | 2026-05-19 |
| shadcn ui Themes | https://ui.shadcn.com/themes | Official Documentation (theme builder) | Not yet |
| shadcn ui Blocks | https://ui.shadcn.com/blocks | Official Documentation | Not yet |
| shadcn ui CLI | https://ui.shadcn.com/docs/cli | Official Documentation | 2026-05-19 |
| shadcn ui Registry | https://ui.shadcn.com/docs/registry | Official Documentation | 2026-05-19 |
| shadcn ui GitHub | https://github.com/shadcn-ui/ui | GitHub Repository | 2026-05-19 |
| shadcn ui Releases | https://github.com/shadcn-ui/ui/releases | GitHub Releases | 2026-05-19 |
| Radix UI Primitives | https://www.radix-ui.com/primitives | Official Documentation (underlying primitives) | 2026-05-19 |
| Radix UI Primitives GitHub | https://github.com/radix-ui/primitives | GitHub Repository | Not yet |
| class-variance-authority | https://cva.style/docs | Official Documentation (variant API) | 2026-05-19 |
| class-variance-authority GitHub | https://github.com/joe-bell/cva | GitHub Repository | Not yet |
| tailwind-merge | https://github.com/dcastil/tailwind-merge | GitHub Repository | 2026-05-19 |
| Tailwind CSS Docs | https://tailwindcss.com/docs | Official Documentation (styling foundation) | Not yet |
| React Hook Form | https://react-hook-form.com/docs | Official Documentation (Form integration) | Not yet |
| Zod | https://zod.dev | Official Documentation (Form validation) | Not yet |
| TanStack Table | https://tanstack.com/table/latest/docs | Official Documentation (DataTable integration) | Not yet |
| cmdk | https://cmdk.paco.me | Official Documentation (Command primitive) | Not yet |
| Sonner | https://sonner.emilkowal.ski | Official Documentation (Toast replacement) | Not yet |
| react-resizable-panels | https://github.com/bvaughn/react-resizable-panels | GitHub Repository (Resizable component) | Not yet |
| Lucide Icons | https://lucide.dev | Official Documentation (icon library) | Not yet |
| Vaul (Drawer) | https://github.com/emilkowalski/vaul | GitHub Repository (Drawer underlying lib) | Not yet |
| react-day-picker | https://react-day-picker.js.org | Official Documentation (Calendar underlying lib, v9) | Not yet |
| input-otp | https://github.com/guilhermerodz/input-otp | GitHub Repository (Input OTP underlying lib) | Not yet |
| Recharts | https://recharts.org | Official Documentation (Chart underlying lib) | Not yet |

### Secondary Sources (use only when primary is insufficient)

| Source | URL | Type | Last Verified |
|--------|-----|------|---------------|
| shadcn ui Issues | https://github.com/shadcn-ui/ui/issues | GitHub Issues (anti-patterns + real bugs) | 2026-05-19 |
| shadcn ui Discussions | https://github.com/shadcn-ui/ui/discussions | GitHub Discussions | Not yet |
| Radix UI Issues | https://github.com/radix-ui/primitives/issues | GitHub Issues (primitive bugs) | Not yet |

## Verification Rules

1. **Primary sources ONLY**: Official docs > source code > GitHub issues for anti-patterns
2. **NEVER use**: Random blog posts, unverified StackOverflow answers, AI-generated content without verification
3. **Version-check**: All examples must work against shadcn ui evergreen-2026 (canary)
4. **Date-check**: Note last verification date per source
5. **Cross-reference**: If docs are sparse, verify against source code at github.com/shadcn-ui/ui
6. **WebFetch**: ALWAYS use WebFetch to verify against latest official documentation (D-006)
7. **Stack-coupling**: shadcn ui composes Radix + cva + tailwind-merge. Verify each underlying primitive against its OWN docs (not assumed from shadcn docs).

## Source Addition Protocol

When discovering a new source during research:
1. Verify it's official or maintained by core team
2. Add to appropriate table above with Last Verified date
3. Update the Last Verified date when re-verified during Phase 4 topic research
4. Record significant discoveries in LESSONS.md
