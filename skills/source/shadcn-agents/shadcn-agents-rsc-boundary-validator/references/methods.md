# Methods : RSC Boundary Validator Reference

The authoritative per-primitive matrix is in
`shadcn-impl-rsc-vs-client-boundaries` `references/methods.md` §3 (60 rows).
This file holds the validator-specific tooling : the six-rule checklist,
grep / ripgrep / jq one-liners that implement each rule, a condensed
matrix snapshot, AST-level patterns for rule 6, and a Bash driver
script that runs all six rules and emits the verdict block.

Verified against https://ui.shadcn.com/docs/components-json,
https://react.dev/reference/rsc/use-client, and
https://nextjs.org/docs/app/building-your-application/rendering/composition-patterns.
Last verified : 2026-05-19.

---

## 1. The Six-Rule Checklist (Formal Form)

| #  | Rule                                                                 | Verdict on miss |
|----|----------------------------------------------------------------------|-----------------|
| 1  | `components.json` `rsc` flag matches framework (App Router = true, else false). | FAIL            |
| 2  | Every file importing a CLIENT-REQUIRED primitive starts with `"use client"`.    | FAIL            |
| 3  | `app/layout.tsx` / `app/page.tsx` / `app/template.tsx` MUST NOT start with the directive. | FAIL            |
| 4  | Every `"use client"` is on line 1 (after comments / blank lines), double- or single-quoted, file-level. | FAIL            |
| 5  | Every Context Provider lives in a dedicated `"use client"` wrapper file, never imported directly into a server layout. | FAIL            |
| 6  | No server file passes a function or non-serialisable instance into a client child. | FAIL            |
| 2b | RSC-safe primitive (Card / Badge / Button / ...) carries the directive unnecessarily. | WARN            |

The validator emits PASS / WARN / FAIL per rule and a final aggregate
verdict (FAIL if any rule failed, WARN if any rule warned and none
failed, PASS otherwise).

---

## 2. Grep / Ripgrep / jq Patterns per Rule

### Rule 1 : `components.json` `rsc` flag

```bash
# Read the flag :
jq '.rsc' components.json
# -> true  (App Router project, expected)
# -> false (Pages / Vite / Astro / Remix / TanStack Start, expected)

# Detect the framework :
test -d app && echo "framework=app-router"
test -d pages && echo "framework=pages-router"
test -f vite.config.ts -o -f vite.config.js && echo "framework=vite"
test -f astro.config.mjs -o -f astro.config.ts && echo "framework=astro"
test -f remix.config.js -o -f remix.config.ts && echo "framework=remix"

# Cross-check :
FW=$(test -d app && echo app-router || (test -d pages && echo pages-router || echo other))
RSC=$(jq -r '.rsc' components.json)
if [ "$FW" = "app-router" ] && [ "$RSC" != "true" ]; then echo "FAIL rule 1 : App Router needs rsc:true (got $RSC)"; fi
if [ "$FW" != "app-router" ] && [ "$RSC" = "true" ]; then echo "FAIL rule 1 : non-App-Router project must set rsc:false (got $RSC)"; fi
```

### Rule 2 : Missing directive on a client-required file

```bash
# Files that MUST start with "use client" :
CLIENT_FILES=(
  accordion alert-dialog calendar carousel chart checkbox collapsible
  combobox command context-menu dialog drawer dropdown-menu form
  hover-card input-otp menubar navigation-menu popover progress
  radio-group resizable scroll-area select sheet sidebar slider sonner
  switch tabs toggle toggle-group tooltip
)

for f in "${CLIENT_FILES[@]}"; do
  path="components/ui/${f}.tsx"
  [ -f "$path" ] || continue
  if ! head -1 "$path" | grep -qE '^["'"'"']use client["'"'"']'; then
    echo "FAIL rule 2 : missing directive in $path"
  fi
done
```

Equivalent one-liner :

```bash
grep -L '^"use client"' \
  components/ui/{accordion,alert-dialog,calendar,carousel,chart,checkbox,collapsible,combobox,command,context-menu,dialog,drawer,dropdown-menu,form,hover-card,input-otp,menubar,navigation-menu,popover,progress,radio-group,resizable,scroll-area,select,sheet,sidebar,slider,sonner,switch,tabs,toggle,toggle-group,tooltip}.tsx 2>/dev/null
```

### Rule 2b (WARN) : Over-clientified RSC-safe file

```bash
grep -l '^"use client"' \
  components/ui/{alert,aspect-ratio,avatar,badge,breadcrumb,button,card,empty,field,input,input-group,item,kbd,label,pagination,separator,skeleton,spinner,table,textarea,typography}.tsx 2>/dev/null
```

Any file printed is over-clientified (ships unnecessary JavaScript) ;
WARN-level, NOT a FAIL.

### Rule 3 : Layout / page / template marked client

```bash
for f in app/layout.tsx app/page.tsx app/template.tsx; do
  [ -f "$f" ] || continue
  if head -1 "$f" | grep -qE '^["'"'"']use client["'"'"']'; then
    echo "FAIL rule 3 : $f is marked client (defeats RSC for the subtree)"
  fi
done

# Recursive variant for nested route segments :
rg --files-with-matches -n '^"use client"' app | grep -E '/(layout|page|template)\.tsx?$'
```

### Rule 4 : Well-formed directive

```bash
# A : directive must be on line 1 (or line 1 after a license / @ts-* comment block)
rg -n '"use client"' --no-heading | awk -F: '$2 != 1 { print "FAIL rule 4 (placement) :", $0 }'

# B : NEVER backticked (template literal -> ignored by bundler)
rg -n '`use client`' --no-heading | awk '{ print "FAIL rule 4 (backticks) :", $0 }'

# C : NEVER conditional / inside if-statement
rg -n 'if\s*\([^)]+\)\s*["'"'"']use client["'"'"']' --no-heading | awk '{ print "FAIL rule 4 (conditional) :", $0 }'

# D : NEVER inside a function body
rg -B0 -A0 -n '^\s+"use client"' --no-heading | awk '{ print "FAIL rule 4 (indented, not file-level) :", $0 }'
```

### Rule 5 : Provider in server layout

```bash
# Each line that imports one of the known provider libs DIRECTLY in a
# file that is NOT marked "use client" :

PROVIDER_PATTERNS=(
  'from\s+["'"'"']next-themes["'"'"']'
  'from\s+["'"'"']@tanstack/react-query["'"'"']'
  'from\s+["'"'"']jotai["'"'"']'
  'from\s+["'"'"']zustand["'"'"']'
  'from\s+["'"'"']sonner["'"'"']'
  'from\s+["'"'"']nuqs["'"'"']'
)

for f in $(rg --files-with-matches -n 'ThemeProvider|QueryClientProvider|JotaiProvider|Toaster|NuqsAdapter|<Provider' app); do
  if ! head -1 "$f" | grep -qE '^["'"'"']use client["'"'"']'; then
    echo "FAIL rule 5 : provider imported in server file $f"
  fi
done
```

A clean project will have ZERO matches : every provider lives in
`components/providers/*.tsx` or `components/theme-provider.tsx`, each
of which carries the directive on line 1.

### Rule 6 : Function / non-serialisable prop across boundary

This rule needs an AST or a heuristic grep. The validator runs the
heuristic and surfaces candidates for human review (false-positive
rate is low when scoped to JSX expression contexts).

```bash
# Heuristic A : inline arrow function inside JSX inside a server file :
for f in $(rg --files-with-matches -l "" app components --glob '*.tsx' | xargs -I{} sh -c 'head -1 {} | grep -q "use client" || echo {}'); do
  rg -n '<[A-Z][A-Za-z]+\s+[^>]*on[A-Z][A-Za-z]+=\{[^}]*=>' "$f" \
    | awk -v F="$f" '{ print "FAIL rule 6 (inline-arrow) :", F, $0 }'
done

# Heuristic B : `new <ClassName>(...)` passed as a prop :
rg -n '<[A-Z][A-Za-z]+\s+[^>]*=\{new\s+[A-Z]' app components --glob '*.tsx'

# Heuristic C : Date with custom methods, Decimal, BigInt across boundary :
rg -n '<[A-Z][A-Za-z]+\s+[^>]*=\{[^}]*(new\s+Date|Decimal|BigInt|new\s+Map\(|new\s+Set\()' app components --glob '*.tsx'
```

These are heuristics ; the validator surfaces them as `FAIL rule 6
(candidate)` and asks the operator to confirm. AST-precise detection
requires a TypeScript-language-server pass : when available, run
`tsc --noEmit` and look for the canonical Next.js error :

```
Error: Functions cannot be passed directly to Client Components
unless you explicitly expose it by marking it with "use server".
```

ALWAYS treat that compiler error as authoritative ; the heuristic is
only used when running outside `tsc`.

---

## 3. Bash Driver : `validate-rsc.sh`

```bash
#!/usr/bin/env bash
# validate-rsc.sh : run all six RSC boundary checks against a shadcn project.
# Usage : ./validate-rsc.sh [repo-root]
set -e
ROOT="${1:-.}"
cd "$ROOT"

FAILS=0
WARNS=0
echo "RSC Boundary Audit : $(basename "$ROOT")"
echo

# Detect framework
if [ -d app ]; then FW="app-router"
elif [ -d pages ]; then FW="pages-router"
elif [ -f vite.config.ts ] || [ -f vite.config.js ]; then FW="vite"
elif [ -f astro.config.mjs ] || [ -f astro.config.ts ]; then FW="astro"
elif [ -f remix.config.js ] || [ -f remix.config.ts ]; then FW="remix"
else FW="other"
fi

# Rule 1
RSC=$(jq -r '.rsc' components.json 2>/dev/null || echo "missing")
if [ "$FW" = "app-router" ] && [ "$RSC" != "true" ]; then
  echo "[1] components.json rsc flag       :  FAIL  (App Router needs rsc:true, got $RSC)"; FAILS=$((FAILS+1))
elif [ "$FW" != "app-router" ] && [ "$RSC" = "true" ]; then
  echo "[1] components.json rsc flag       :  FAIL  (framework=$FW needs rsc:false, got true)"; FAILS=$((FAILS+1))
else
  echo "[1] components.json rsc flag       :  PASS  (rsc:$RSC matches framework=$FW)"
fi

# Rule 2
MISSING=$(grep -L '^"use client"' \
  components/ui/{accordion,alert-dialog,calendar,carousel,chart,checkbox,collapsible,combobox,command,context-menu,dialog,drawer,dropdown-menu,form,hover-card,input-otp,menubar,navigation-menu,popover,progress,radio-group,resizable,scroll-area,select,sheet,sidebar,slider,sonner,switch,tabs,toggle,toggle-group,tooltip}.tsx 2>/dev/null || true)
if [ -n "$MISSING" ]; then
  echo "[2] Client primitive directives    :  FAIL"; FAILS=$((FAILS+1))
  echo "$MISSING" | sed 's/^/      MISSING in /'
else
  echo "[2] Client primitive directives    :  PASS"
fi

# Rule 2b (WARN)
OVER=$(grep -l '^"use client"' \
  components/ui/{alert,aspect-ratio,avatar,badge,breadcrumb,button,card,empty,field,input,input-group,item,kbd,label,pagination,separator,skeleton,spinner,table,textarea,typography}.tsx 2>/dev/null || true)
if [ -n "$OVER" ]; then
  echo "[2b] RSC-safe over-clientification :  WARN"; WARNS=$((WARNS+1))
  echo "$OVER" | sed 's/^/      OVER-CLIENT /'
fi

# Rule 3
LAYOUT_FAIL=""
for f in app/layout.tsx app/page.tsx app/template.tsx; do
  [ -f "$f" ] || continue
  head -1 "$f" | grep -qE '^"use client"' && LAYOUT_FAIL="$LAYOUT_FAIL $f"
done
if [ -n "$LAYOUT_FAIL" ]; then
  echo "[3] Layout/page/template not client:  FAIL"; FAILS=$((FAILS+1))
  for f in $LAYOUT_FAIL; do echo "      OVER-CLIENT $f"; done
else
  echo "[3] Layout/page/template not client:  PASS"
fi

# Rule 4 (well-formed)
BAD4=$(rg -n '`use client`' app components 2>/dev/null || true)
if [ -n "$BAD4" ]; then
  echo "[4] Directive well-formedness      :  FAIL  (backticked / template literal)"; FAILS=$((FAILS+1))
  echo "$BAD4" | sed 's/^/      /'
else
  echo "[4] Directive well-formedness      :  PASS"
fi

# Rule 5 (Provider in server file)
PROV=$(rg --files-with-matches -n 'from ["'"'"']next-themes["'"'"']|from ["'"'"']@tanstack/react-query["'"'"']|from ["'"'"']sonner["'"'"']' app 2>/dev/null || true)
PROV_FAIL=""
for f in $PROV; do
  head -1 "$f" | grep -qE '^"use client"' || PROV_FAIL="$PROV_FAIL $f"
done
if [ -n "$PROV_FAIL" ]; then
  echo "[5] Provider wrappers              :  FAIL"; FAILS=$((FAILS+1))
  for f in $PROV_FAIL; do echo "      PROVIDER in server file $f"; done
else
  echo "[5] Provider wrappers              :  PASS"
fi

# Rule 6 (boundary props ; heuristic)
B6=$(rg -n '<[A-Z][A-Za-z]+\s+[^>]*on[A-Z][A-Za-z]+=\{[^}]*=>' app components --glob '*.tsx' 2>/dev/null || true)
if [ -n "$B6" ]; then
  echo "[6] Boundary props                 :  CANDIDATE (review manually)"
  echo "$B6" | head -5 | sed 's/^/      /'
else
  echo "[6] Boundary props                 :  PASS"
fi

echo
if [ $FAILS -gt 0 ]; then echo "Verdict : FAIL. $FAILS critical defects + $WARNS warnings."
elif [ $WARNS -gt 0 ]; then echo "Verdict : WARN. $WARNS warnings."
else echo "Verdict : PASS."
fi
```

ALWAYS keep the driver bash-only and dependency-free (jq + rg + grep
are present in every CI runner). NEVER bake the validator into a Node
script ; that adds an npm-install hop and breaks the
ten-second pre-commit budget.

---

## 4. Per-Primitive Matrix Snapshot (60 rows)

Full table : `shadcn-impl-rsc-vs-client-boundaries` `references/methods.md` §3.
Snapshot used by the validator :

| #  | Primitive            | Class         |
|----|----------------------|---------------|
| 1  | Accordion            | client        |
| 2  | Alert                | RSC-safe      |
| 3  | AlertDialog          | client        |
| 4  | AspectRatio          | RSC-safe      |
| 5  | Avatar               | RSC-safe      |
| 6  | Badge                | RSC-safe      |
| 7  | Breadcrumb           | RSC-safe      |
| 8  | Button               | RSC-safe      |
| 9  | Calendar             | client        |
| 10 | Card                 | RSC-safe      |
| 11 | Carousel             | client        |
| 12 | Chart                | client        |
| 13 | Checkbox             | client        |
| 14 | Collapsible          | client        |
| 15 | Combobox             | client        |
| 16 | Command              | client        |
| 17 | ContextMenu          | client        |
| 18 | Data Table recipe    | client        |
| 19 | DatePicker recipe    | client        |
| 20 | Dialog               | client        |
| 21 | Direction            | client (safe) |
| 22 | Drawer               | client        |
| 23 | DropdownMenu         | client        |
| 24 | Empty                | RSC-safe      |
| 25 | Field                | RSC-safe      |
| 26 | Form                 | client        |
| 27 | HoverCard            | client        |
| 28 | Input                | RSC-safe      |
| 29 | InputGroup           | RSC-safe      |
| 30 | InputOTP             | client        |
| 31 | Item                 | RSC-safe      |
| 32 | Kbd                  | RSC-safe      |
| 33 | Label                | RSC-safe      |
| 34 | Menubar              | client        |
| 35 | Native Select        | RSC-safe      |
| 36 | NavigationMenu       | client        |
| 37 | Pagination           | RSC-safe      |
| 38 | Popover              | client        |
| 39 | Progress             | client        |
| 40 | RadioGroup           | client        |
| 41 | Resizable            | client        |
| 42 | ScrollArea           | client        |
| 43 | Select               | client        |
| 44 | Separator            | RSC-safe      |
| 45 | Sheet                | client        |
| 46 | Sidebar              | client        |
| 47 | Skeleton             | RSC-safe      |
| 48 | Slider               | client        |
| 49 | Sonner (Toaster)     | client        |
| 50 | Spinner              | RSC-safe      |
| 51 | Switch               | client        |
| 52 | Table primitives     | RSC-safe      |
| 53 | Tabs                 | client        |
| 54 | Textarea             | RSC-safe      |
| 55 | Toggle               | client        |
| 56 | ToggleGroup          | client        |
| 57 | Tooltip              | client        |
| 58 | Typography           | RSC-safe      |
| 59 | InputGroup (alias)   | RSC-safe      |
| 60 | NativeSelect (alias) | RSC-safe      |

---

## 5. Detecting Imports That Force Client

The driver's rule 2 lookup uses filename matching. The richer check
uses import scanning (catches third-party components that wrap a Radix
package under a custom name) :

```bash
rg -n --no-heading -e '@radix-ui/(react-(accordion|alert-dialog|checkbox|collapsible|context-menu|dialog|dropdown-menu|hover-card|menubar|navigation-menu|popover|progress|radio-group|scroll-area|select|slider|switch|tabs|toggle|toggle-group|tooltip|toast))' \
   -e 'from ["'"'"']cmdk["'"'"']' \
   -e 'from ["'"'"']vaul["'"'"']' \
   -e 'from ["'"'"']embla-carousel-react["'"'"']' \
   -e 'from ["'"'"']react-day-picker["'"'"']' \
   -e 'from ["'"'"']react-resizable-panels["'"'"']' \
   -e 'from ["'"'"']react-hook-form["'"'"']' \
   -e 'from ["'"'"']sonner["'"'"']' \
   -e 'from ["'"'"']@tanstack/react-table["'"'"']' \
   -e 'from ["'"'"']recharts["'"'"']' \
   app components | \
while IFS=: read -r file line _; do
  head -1 "$file" | grep -qE '^"use client"' || echo "FAIL rule 2 (import-scan) : $file:$line needs directive"
done
```

This catches the case where a developer creates
`components/my-dialog.tsx` that wraps Radix Dialog locally ; the file
must carry the directive even though it is not under `components/ui/`.

---

## 6. False-Positive Guard for Native Select / Pagination

Native `<select>` and `<Pagination>` are RSC-safe IF and ONLY IF the
consumer does NOT bind an event handler. The validator surfaces them
as candidates when the JSX includes `onChange=` or `onClick=` :

```bash
rg -n '<select\b[^>]*onChange=' app components --glob '*.tsx'
rg -n '<Pagination[^>]*onClick=' app components --glob '*.tsx'
```

ALWAYS treat hits as FAIL rule 2 (the consumer file needs the directive).

---

## 7. CI / Pre-Commit Wiring

```yaml
# .github/workflows/rsc-boundary.yml
name: RSC Boundary
on: [pull_request, push]
jobs:
  rsc-audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install ripgrep + jq
        run: sudo apt-get install -y ripgrep jq
      - name: Run validate-rsc.sh
        run: bash scripts/validate-rsc.sh .
```

```bash
# .husky/pre-commit
#!/usr/bin/env bash
bash scripts/validate-rsc.sh . || {
  echo "RSC boundary validation failed. Commit aborted." >&2
  exit 1
}
```

ALWAYS run the validator in CI on every PR ; the build itself catches
some of these (missing directive on Dialog crashes the server render)
but rule 3 (layout marked client) compiles fine and silently kills RSC.

---

## 8. References

- https://ui.shadcn.com/docs/components-json : the `rsc` field schema.
- https://react.dev/reference/rsc/use-client : the directive contract.
- https://nextjs.org/docs/app/building-your-application/rendering/composition-patterns : server/client composition.
- https://nextjs.org/docs/messages/react-client-component-async : the
  "Functions cannot be passed directly to Client Components" error.
- `shadcn-impl-rsc-vs-client-boundaries` (B10) : the rulebook this
  validator enforces.
