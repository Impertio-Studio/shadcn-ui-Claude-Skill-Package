---
name: shadcn-agents-rsc-boundary-validator
description: >
  Use when auditing a Next.js App Router codebase that consumes shadcn ui
  for misplaced or missing `"use client"` directives, when reviewing a
  pull request that adds shadcn primitives and you must confirm the
  boundary is correct before merging, when a build fails with the
  `useState is not a function` / `Cannot read properties of null
  (reading useContext)` Radix hydration crash, when `components.json`
  has been edited (toggled `rsc:true` <-> `rsc:false`, or added to a
  Vite / Astro / Pages-Router project where the flag must be `false`),
  when an AI-generated patch has slapped `"use client"` on `app/layout.tsx`
  or on every component file, when a Server Component tries to pass
  `onClick` / `onChange` / a class instance into a client child, when a
  Provider (next-themes ThemeProvider, TanStack Query QueryClientProvider,
  jotai Provider, zustand Provider, Sonner Toaster) has been imported
  directly into a server layout instead of through a thin client wrapper,
  or when a code-review checklist must enforce the RSC boundary rules
  before a release.
  Prevents the over-clientification anti-pattern of marking
  `app/layout.tsx` or `app/page.tsx` as `"use client"` (kills RSC for
  every descendant, defeats App Router bundle savings), the missing-
  directive crash where a file imports Dialog / Sheet / Drawer / Popover
  / Tooltip / HoverCard / Form / Select / Combobox / Command / Sonner /
  Tabs / Accordion / Carousel / Sidebar / Calendar without a `"use
  client"` header and Radix throws on render, the silently-ignored
  conditional directive (template literal `` `use client` ``, `if (...)
  "use client"`, comments before the directive) that the bundler treats
  as a normal string, the boundary-violation pattern where a server
  component passes an inline event handler into a client child, the
  forgotten Provider wrapper where next-themes / TanStack Query /
  Sonner is consumed directly in a server layout, the non-serialisable-
  prop crossing (Date with custom methods, Decimal, class instance,
  Map<string, classInstance>) that silently mis-renders after hydration,
  and the inverted-flag anti-pattern where `rsc:false` is set on an App
  Router project (every primitive ends up client) or `rsc:true` is set
  on a Vite project (primitives ship without the directive they need).
  Covers the validator workflow (scan file -> classify components ->
  verify directive presence + placement + form -> flag boundary
  violations -> report verdict), the 6-point project-level checklist
  (components.json `rsc` flag matches framework, every file-with-
  client-primitive has the directive, no `app/layout.tsx` carries the
  directive, no conditional or quoted-wrong directive, every Provider
  lives in a dedicated client wrapper file, no server-to-client
  function or class-instance prop), the per-primitive RSC compatibility
  matrix for the full 60-primitive evergreen-2026 catalogue (RSC-safe
  vs client-required vs mixed) inherited from the boundary skill, the
  grep patterns + ripgrep one-liners that flag each violation class,
  the worked validation scenarios (broken layout + missing Dialog
  directive vs corrected server-shell-plus-client-island), and the
  exact hand-off rules to the related shadcn skills.
  Keywords: rsc validator, validate use client, server component
  validator, boundary validator, components.json rsc check, layout
  use client warning, validate Next.js App Router shadcn, server
  passing onClick, validate Provider boundary, RSC code review,
  use client missing, use client conditional, hydration error
  validator, ThemeProvider in server layout, useState is not a
  function shadcn, useContext is null radix, server passes function
  to client, Dialog without use client, Sheet without use client,
  Form without use client, Sonner Toaster without use client,
  Sidebar without use client, rsc:true should be true, rsc:false on
  app router, rsc:true on vite, App Router shadcn audit, shadcn pr
  review, shadcn lint, why does my app crash on hydration, why is
  my dialog not opening, ship less javascript audit, smallest leaf
  use client, suppressHydrationWarning missing, non-serialisable
  prop, class instance prop crossing, Decimal across boundary,
  Date across boundary, Map across boundary.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026, Next.js App Router (14+ / 15.x). For Vite, Astro, Remix, Pages Router the validator enforces `rsc:false`."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui : RSC Boundary Validator (Agent / Validator)

This skill is an AGENT skill. It does NOT teach how to compose RSC vs
client trees (that is `shadcn-impl-rsc-vs-client-boundaries`, B10). It
teaches an automated AUDIT : given a project tree, classify every shadcn
import, verify the `"use client"` directive is in the right files, in
the right form, in the right place, and that the boundary crossing
follows the React Server Components rules.

ALWAYS run the validator as a six-point checklist in a fixed order.
NEVER stop at the first failure ; a project usually has several boundary
defects layered together, and the checklist surfaces all of them in one
pass.

## Companion Skills

- `shadcn-impl-rsc-vs-client-boundaries` (B10) : the rulebook that defines
  WHICH primitives need the directive and WHY ; this validator enforces
  those rules. The full per-primitive matrix lives in B10
  `references/methods.md` (60 rows).
- `shadcn-core-cli` : `components.json` schema including the `rsc` flag,
  `shadcn add --overwrite` for regenerating a file whose directive is
  wrong.
- `shadcn-impl-component-install` : the install workflow that emits files
  with or without the directive ; use this skill if the validator
  reports a missing-directive file and you want to re-add cleanly.
- `shadcn-impl-framework-integration` : per-framework defaults (App
  Router -> `rsc:true`, Vite -> `rsc:false`, Astro client island ->
  `rsc:false`).

## Quick Reference : Validator Workflow

```
INPUT  : a repo path with components.json + components/ui/* + app/ (or pages/)
PHASE 1 : scan components.json -> read `rsc` flag, detect framework
PHASE 2 : classify every file under components/ui/* and app/ using
          the per-primitive matrix (RSC-safe / client-required / mixed)
PHASE 3 : verify the `"use client"` directive is present where required,
          absent where forbidden, well-formed where present
PHASE 4 : scan for boundary-violation props (onClick / function / class
          instance passed from server file to client file)
PHASE 5 : scan for Provider-in-server-layout anti-pattern
PHASE 6 : emit a verdict report : PASS / WARN / FAIL with file + line
OUTPUT : 6-point checklist with PASS / FAIL per item + remediation hint
```

ALWAYS run all six phases even on small projects ; the cheap phases
(grep) cost milliseconds and catch the silently-broken cases.

## Quick Reference : 6-Point Checklist

```
[1] components.json `rsc` flag matches the framework :
    App Router -> rsc:true.
    Pages Router / Vite / Astro / Remix / TanStack Start -> rsc:false.

[2] Every file importing a CLIENT-REQUIRED primitive starts with the
    directive :
    Dialog / Sheet / Drawer / AlertDialog / Popover / Tooltip /
    HoverCard / Form / Select / Combobox / Command / Sonner (Toaster
    mount) / Tabs / Accordion / Carousel / Sidebar / Calendar /
    DropdownMenu / ContextMenu / Menubar / NavigationMenu / Switch /
    Slider / Checkbox / RadioGroup / Collapsible / ScrollArea /
    InputOTP / Toggle / ToggleGroup / Resizable / Chart / Progress.

[3] No `app/layout.tsx` / `app/page.tsx` / `app/template.tsx` carries
    the directive (defeats RSC for the whole subtree).

[4] Every `"use client"` is :
    - the FIRST non-whitespace / non-comment line of the file,
    - double-quoted or single-quoted (NEVER backticks, NEVER a
      template literal, NEVER inside an `if`),
    - file-level (no per-function or per-component attempt).

[5] Every context Provider (next-themes ThemeProvider, TanStack Query
    QueryClientProvider, jotai Provider, zustand Provider, Sonner
    Toaster, OverlayProvider, NuqsAdapter) lives in a dedicated
    `"use client"` wrapper file ; never imported directly into
    `app/layout.tsx` or any server component.

[6] No server file (no `"use client"` directive) passes a function
    (`onClick`, `onChange`, `onSubmit`, `onSelect`, ...) or a non-
    serialisable instance (class instance, Decimal, Date with custom
    methods, Map<key, classInstance>) to a client child.
```

For ripgrep / find one-liners that implement each check see
`references/methods.md` §3.

## Quick Reference : Per-Primitive Lookup Matrix

The validator inherits the canonical per-primitive table from
`shadcn-impl-rsc-vs-client-boundaries` `references/methods.md` §3
(60 rows). The condensed lookup used by this validator :

```
CLIENT-REQUIRED (must carry "use client") :
  Accordion, AlertDialog, Calendar, Carousel, Chart, Checkbox,
  Collapsible, Combobox, Command, ContextMenu, Dialog, Drawer,
  DropdownMenu, Form (react-hook-form), HoverCard, InputOTP, Menubar,
  NavigationMenu, Popover, Progress, RadioGroup, Resizable,
  ScrollArea, Select, Sheet, Sidebar, Slider, Sonner Toaster,
  Switch, Tabs, Toggle, ToggleGroup, Tooltip.

RSC-SAFE (must NOT carry "use client" unless the parent file also
imports a client primitive) :
  Alert, AspectRatio, Avatar, Badge, Breadcrumb, Button, Card,
  Empty, Field, Input, InputGroup, Item, Kbd, Label, Native Select
  (without onChange), Pagination, Separator, Skeleton, Spinner,
  Table primitives (Table / TableRow / ...), Textarea, Typography.

MIXED (the export bundle includes both a client root and pure-HTML
subcomponents ; the emitted file is client) :
  AlertDialog (Trigger + Content + body), Direction.
```

Rule of thumb the validator uses : if the source file imports anything
from `@radix-ui/*` (other than `@radix-ui/react-label` and `@radix-ui/
react-separator`) or from `cmdk` / `embla-carousel-react` / `vaul` /
`react-day-picker` / `react-resizable-panels` / `react-hook-form` /
`sonner` / `@tanstack/react-table` / `recharts`, the file MUST carry
the directive. See `references/methods.md` §2 for the full grep
pattern and §5 for the matrix snapshot.

## Quick Reference : Three Verdict Levels

```
PASS  : all six checks succeed ; the project's RSC boundary is sound.
WARN  : at least one rule fired on a RSC-safe primitive (cosmetic
        over-clientification : Card / Badge / Button with the directive
        for no reason ; ships extra JavaScript but does not crash).
FAIL  : at least one rule fired on a CLIENT-REQUIRED primitive (the
        project WILL crash at render with `useState is not a function`
        or similar) ; a Provider was found in a server layout ; a
        server file passed a function to a client child ; the `rsc`
        flag mismatches the framework.
```

ALWAYS report ALL failures in a single verdict ; never short-circuit.

## Decision Tree : Verifying a Single File

```
For file F :

1. Read first non-comment, non-blank line.
2. If F is `app/layout.tsx` / `app/page.tsx` / `app/template.tsx` :
     and line 1 == `"use client"` -> FAIL (rule 3)
3. Else if line 1 == `"use client"` :
     verify the directive is well-formed (rule 4)
     mark F as a CLIENT file ; continue.
4. Else (no directive on line 1) :
     scan imports for CLIENT-REQUIRED primitives or for any
     `@radix-ui/*` package (minus Label / Separator) or any
     cmdk / vaul / embla / react-day-picker / react-hook-form /
     sonner / @tanstack/react-table / recharts dependency.
     If any match -> FAIL (rule 2 ; directive missing).
     If only RSC-safe primitives -> file is OK as server.
5. For every JSX child element rendered in F :
     if the element is a known client component (imported from a
     "use client" file) AND F is server (no directive) AND a
     prop on that element is a function literal, an arrow function,
     a class instance, a Date with custom methods, or a Map ->
     FAIL (rule 6).
```

See `references/examples.md` for two worked traces (broken vs
correct) of this decision tree.

## Pattern : Running the Validator

```bash
# Tier 1 : two-second grep audit. Run in every shadcn project before merge.
grep -L '^"use client"' \
  components/ui/{accordion,alert-dialog,calendar,carousel,chart,checkbox,collapsible,combobox,command,context-menu,dialog,drawer,dropdown-menu,form,hover-card,input-otp,menubar,navigation-menu,popover,progress,radio-group,resizable,scroll-area,select,sheet,sidebar,slider,sonner,switch,tabs,toggle,toggle-group,tooltip}.tsx 2>/dev/null
# Expected output : EMPTY. Any filename printed is FAIL.

grep -l '^"use client"' \
  components/ui/{alert,aspect-ratio,avatar,badge,breadcrumb,button,card,empty,field,input,input-group,item,kbd,label,pagination,separator,skeleton,spinner,table,textarea}.tsx 2>/dev/null
# Expected output : EMPTY. Any filename printed is WARN.

grep -l '^"use client"' app/layout.tsx app/page.tsx app/template.tsx 2>/dev/null
# Expected output : EMPTY. Any filename printed is FAIL (rule 3).

jq '.rsc' components.json
# Expected : true for App Router. false for Vite / Pages / Astro / Remix.
```

ALWAYS treat the grep output as authoritative for rules 2, 3, and the
RSC-safe over-clientification check (WARN). For rules 4 (well-formed
directive), 5 (Provider wrappers), and 6 (boundary crossing) use the
extended ripgrep patterns documented in `references/methods.md` §3.

## Pattern : Reporting Format

```
RSC Boundary Audit : <repo-name>

[1] components.json rsc flag      :  PASS  (rsc:true matches framework=app-router)
[2] Client primitive directives    :  FAIL
      MISSING in components/ui/dialog.tsx       (imports @radix-ui/react-dialog)
      MISSING in components/ui/sheet.tsx        (imports @radix-ui/react-dialog)
[3] Layout / page / template not client : FAIL
      OVER-CLIENT app/layout.tsx
[4] Directive well-formedness      :  PASS  (all 12 directives valid)
[5] Provider wrappers              :  FAIL
      ThemeProvider imported directly from next-themes in app/layout.tsx
      Wrap in components/theme-provider.tsx with "use client"
[6] Boundary props                 :  PASS  (no inline functions across server->client)

Verdict : FAIL. 3 critical defects + 0 warnings.
Run remediation : see shadcn-impl-rsc-vs-client-boundaries B10.
```

ALWAYS print a section per rule with a single PASS / WARN / FAIL keyword
and the offending paths + reason. NEVER aggregate ; per-file detail is
what the developer needs to fix.

## Pattern : Remediation Hand-Off

After a FAIL verdict the validator does NOT auto-fix. It hands off :

| Defect class                              | Remediation skill                                       |
|-------------------------------------------|---------------------------------------------------------|
| Rule 1 : `rsc` flag wrong                 | `shadcn-core-cli` : edit `components.json`, rerun `add` |
| Rule 2 : missing directive on client file | `shadcn-impl-component-install` : `shadcn add <name> --overwrite` |
| Rule 3 : layout marked client             | `shadcn-impl-rsc-vs-client-boundaries` §"Server Shell + Client Island" |
| Rule 4 : directive form wrong             | manual fix : move to first line, double-quote it        |
| Rule 5 : Provider in server layout        | `shadcn-impl-rsc-vs-client-boundaries` §"Provider Wrapper" |
| Rule 6 : function/instance across boundary| `shadcn-impl-rsc-vs-client-boundaries` §"Passing Data Across the Boundary" |

ALWAYS forward to the skill ; NEVER inline the remediation steps in the
validator output. Keeps the validator small and the rulebook authoritative.

## When This Skill Does NOT Apply

- The project is a Pages Router project AND `rsc:false` is set : the
  emitted directive on every file is correct ; rule 2 still applies
  (Dialog must have it), rule 3 does not (there is no `app/layout`).
  Run rules 1, 2, 4, 5, 6 ; skip rule 3.
- The project is a Vite + React SPA : there are no server components ;
  every file is client ; rule 3 does not apply. The `rsc:false` flag
  must be set or the CLI omits the directive on Dialog and the app
  silently mis-bundles. Run rules 1, 2, 4 ; skip 3 ; rules 5 and 6
  become advisory (Provider hygiene is still recommended).
- The project is a single-file demo or a Storybook story : the
  validator is overkill ; rely on the build error instead.

## Reference Files

- `references/methods.md` : full ripgrep / find / jq one-liners per rule,
  per-primitive matrix snapshot (60 rows), AST patterns for rule 6
  (server-to-client prop crossing), and a Bash driver script that
  runs all six rules and emits the report block above.
- `references/examples.md` : two worked traces : a BROKEN App Router
  project (layout client + missing Dialog directive + ThemeProvider
  in layout + onClick across boundary) with verdict, and the CORRECT
  refactored version showing the canonical server-shell + client-
  island layout.
- `references/anti-patterns.md` : six anti-patterns the validator
  catches with diagnosis, fix, and verification step. Covers the
  six checklist failure modes plus the suppressHydrationWarning-as-
  bandaid trap.
