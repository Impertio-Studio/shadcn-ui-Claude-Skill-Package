# shadcn CLI Sync Mismatch : Anti-Patterns

Every entry pairs a concrete WRONG pattern with the WHY (root cause)
and the right thing to do instead. Verified against
https://ui.shadcn.com/docs/cli and field reports cross-referenced in
the project's vooronderzoek §9-10.

## 1. Running `add <name> --overwrite` Without `--diff` First

WRONG :

```bash
pnpm dlx shadcn@latest add button --overwrite
```

Why this fails : `--overwrite` replaces `components/ui/button.tsx`
byte-for-byte with the current registry version. Local edits are gone
with no undo prompt, no confirmation summary, and no diff print. If
you have any custom variants, custom default props, custom imports,
or a custom comment header, they vanish silently. The CLI exits with
status 0 and a "Updated 1 file" line. The destruction is invisible
unless you `git diff` immediately after, and if your tree was dirty
the diff conflates your own pre-overwrite changes with the CLI's
write.

Right thing : ALWAYS run `--dry-run` then `--diff` BEFORE `--overwrite`.

```bash
pnpm dlx shadcn@latest add button --dry-run     # list files touched
pnpm dlx shadcn@latest add button --diff        # show the actual content diff
# review. decide. only then :
pnpm dlx shadcn@latest add button --overwrite
```

The diff-first habit takes ten seconds and prevents the entire class
of "the CLI ate my edits" bug reports.

## 2. Modifying `components/ui/<name>.tsx` Directly and Never Re-Running the CLI

WRONG : edit `components/ui/dialog.tsx` inline (new variants, custom
icon, tweaked layout), then refuse to ever re-run `shadcn add dialog`
out of fear of losing edits.

Why this fails : you become disconnected from upstream. shadcn ships
real fixes : the Sidebar mobile DialogTitle a11y fix (issue #5746),
the cmdk break recovery (issue #2944), the react-day-picker v9
adaptation in Calendar (issue #4366), Tailwind v4 keyframe imports
(issue #6925). A perma-snapshotted component will MISS these fixes,
and the failure modes are subtle (silent a11y warnings, runtime
TypeErrors only on certain inputs, missing animations). For a
security-relevant component (anything that wraps an event handler
that crosses user-input boundaries), the snapshot path is unacceptable.

Right thing : use the custom-variants-in-a-separate-file pattern (see
SKILL.md and examples.md Example 2). Keep `components/ui/<name>.tsx`
re-add-safe. Custom code lives in `components/ui/<name>-extensions.tsx`,
which the CLI never touches. Or, for heavy customization, use the fork
strategy : copy the upstream to `lib/components/<name>/`, customize
the fork, keep the upstream file as a reference for diffs.

If you genuinely choose vendor pattern (snapshot forever), document it
explicitly with a top-of-file `VENDORED YYYY-MM-DD` comment and
schedule periodic upstream-review checkpoints. Snapshot-by-accident
is the failure mode this anti-pattern names ; snapshot-by-decision
with documentation is acceptable for low-risk components.

## 3. Adding Custom Variants Inline in the Copied File

WRONG :

```tsx
// components/ui/button.tsx   (the file shadcn add wrote)
const buttonVariants = cva("...", {
  variants: {
    variant: {
      default: "...",
      destructive: "...",
      outline: "...",
      secondary: "...",
      ghost: "...",
      link: "...",
      warning: "bg-amber-500 text-amber-50",       // inline custom
      success: "bg-emerald-600 text-emerald-50",   // inline custom
    },
    size: { ... },
  },
})
```

Why this fails : the `warning` and `success` entries are inside the
shadcn-managed file. The next `shadcn add button --overwrite` (run by
you, a teammate, a CI script, or `migrate icons` which rewrites the
file) deletes them silently. Every call site that used
`variant="warning"` now type-errors at build time (or, worse, falls
through to the default variant at runtime). The call sites are spread
across dozens of files. The recovery is "search the git log for the
last commit that had `warning`, copy the cva block back". Expensive
and error-prone.

Right thing : the custom-variants-in-a-separate-file pattern. Create
`components/ui/button-extensions.tsx` exporting an `ExtButton`
component that composes the base `Button` with additional cva
variants on a wrapper. Import call sites use `ExtButton intent="warning"`
instead of `Button variant="warning"`. The base `button.tsx` stays
byte-identical to the registry version and survives every overwrite.

See SKILL.md "Custom Variants in a Separate File" and examples.md
Example 2 for full code.

## 4. Assuming `components.json` Tracks Per-Component Versions

WRONG : reading the `components.json` schema and looking for
`installedVersion`, `componentLocks`, `dependencies.button`, or a
`components.lock.json` companion file. Concluding that "shadcn must
have some lockfile somewhere" and writing CI checks based on that
assumption.

Why this fails : there is NO per-component version pin. `components.json`
records configuration (style, baseColor, rsc, aliases, registries),
not component state. The registry is evergreen ; each `add` call
fetches the current registry item. The only "version" you have is the
file content on disk. CI checks that compare `package.json` versions
to detect "shadcn updates" cannot work because shadcn-the-CLI version
in `package.json` is unrelated to "what version of Button is in
components/ui/button.tsx".

Right thing : version-pin via git. Treat `components/ui/<name>.tsx`
like any other source file you own. The git commit is the version
pin. Add a top-of-file comment recording the date and CLI version
used :

```tsx
// components/ui/button.tsx
// shadcn-add 2026-05-19 via shadcn@4.7.0
```

Then the audit becomes : "show the git log for components/ui/, sorted
by last-modified, with date and CLI-version annotations". For
projects with strong audit needs, automate a pre-commit hook that
prepends/refreshes the annotation when `add` touches the file.

## 5. Forgetting Clean Git Status Before `add --overwrite`

WRONG :

```bash
# (uncommitted edits to components/ui/button.tsx in working tree)
pnpm dlx shadcn@latest add button --overwrite
# (CLI overwrites the file ; your uncommitted edits are erased)
git diff HEAD -- components/ui/button.tsx
# (shows ONLY the diff between previous-commit and CLI-write,
#  NOT between previous-commit + your edits + CLI-write,
#  because your edits never made it to a commit)
```

Why this fails : `--overwrite` doesn't stop for uncommitted changes.
Without a commit, your work-in-progress edits are not in git history.
After the overwrite there is NO recovery path : `git stash` was never
run, `git checkout HEAD~1 -- <file>` brings back the pre-edit version
not your in-progress version, and editor undo history typically does
NOT span across an external file replacement. The destruction is
permanent.

Right thing : enforce a pre-overwrite hygiene routine :

```bash
git status                       # MUST show clean tree, OR
git add components/ui/ && git commit -m "wip: pre-shadcn-sync snapshot"
                                 # commit the WIP edits first

# now safe :
pnpm dlx shadcn@latest add button --diff
pnpm dlx shadcn@latest add button --overwrite
```

For extra safety, tag the pre-sync commit (`git tag pre-sync-...`) so
rollback is a single `git reset --hard <tag>`. Optionally encode the
hygiene rule as a pre-`shadcn` script in `package.json` :

```json
{
  "scripts": {
    "shadcn:safe-add": "git diff-index --quiet HEAD -- || (echo 'tree not clean' && exit 1) ; pnpm dlx shadcn@latest add"
  }
}
```

Then `pnpm shadcn:safe-add button --diff` refuses to run on a dirty
tree.

## 6. Treating `shadcn diff` as a Subcommand

WRONG :

```bash
pnpm dlx shadcn@latest diff button
# error: unknown command 'diff'
```

Why this fails : the diff capability is a FLAG on `add`, not a
subcommand. Verified at https://ui.shadcn.com/docs/cli (2026-05-19) :
the documented `add` flags include `--diff [path]` ; there is no
top-level `diff` command. Likewise there is no `shadcn update <name>`
subcommand : "update" workflow IS `add <name> --diff` then `add <name>
--overwrite`. Older blog posts and copilot-style autocompletes
suggest the wrong shape often enough that the mistake is endemic.

Right thing : use the FLAG.

```bash
pnpm dlx shadcn@latest add button --diff           # preview diff
pnpm dlx shadcn@latest add button --diff path.tsx  # diff against a specific local file
pnpm dlx shadcn@latest add button --dry-run        # list files touched, no diff
pnpm dlx shadcn@latest add button --overwrite      # apply
```

## 7. Combining `--all` with `--overwrite` and `--yes`

WRONG :

```bash
pnpm dlx shadcn@latest add --all --overwrite --yes
```

Why this fails : `--all` adds EVERY component in the registry,
`--overwrite` replaces files, and `--yes` skips confirmations. The
three together produce a project-wide nuke of `components/ui/` with
zero prompts. Every customization, every comment header, every
extension that lives in `components/ui/` (which is wrong but
common), every minor inline fix is gone. A team that runs this once
spends an afternoon piecing the project back together from git
history.

Right thing : NEVER combine those three flags. If you genuinely want
all registry components installed (rare ; usually a sign that
component selection has gotten out of hand), use them ONE AT A TIME
with `--diff` review. Reserve `--yes` and `--all` for fresh-project
bootstraps where there is nothing to overwrite.

## 8. Running `migrate icons` or `migrate radix` on a Dirty Tree

WRONG :

```bash
git status
# Changes not staged for commit:
#   modified:   components/ui/dialog.tsx
#   modified:   components/ui/button.tsx
pnpm dlx shadcn@latest migrate icons -y
```

Why this fails : `migrate icons` rewrites import lines across EVERY
`components/ui/*.tsx` file, including ones you have uncommitted edits
in. The migration is well-behaved (it only touches imports and icon
symbols), but `-y` skips confirmation and the rewrite is mixed with
your WIP. The resulting `git diff` is a hopeless tangle of "my edit"
and "migrate's edit" on adjacent lines. The recovery is to `git
stash`, re-run the migrate, then unstash and re-resolve.

Right thing : commit or stash WIP first, then run with `-y` only
after a non-`-y` dry pass on one file.

```bash
git stash push -m "wip"
pnpm dlx shadcn@latest migrate icons --list      # see options
pnpm dlx shadcn@latest migrate icons             # run interactively first time
git diff HEAD                                    # review
pnpm test
git add -A && git commit -m "chore: migrate icon library"
git stash pop
```

For projects with many `components/ui/` files, the interactive prompt
is preferable to `-y` even when the tree is clean : the prompt prints
which files will change so a misconfigured run can be aborted before
any write.

## 9. Forking Without Pinning Imports

WRONG : copy `components/ui/sidebar.tsx` to `lib/components/sidebar/sidebar.tsx`,
customize the fork, but leave call sites importing from
`@/components/ui/sidebar`. Then run `shadcn add sidebar --overwrite`
to "stay current".

Why this fails : the fork at `lib/components/sidebar/` is never
called. Every import in the app still resolves to the upstream-managed
`components/ui/sidebar.tsx`. The fork drifts into dead code. The team
thinks the customizations are live ; they are not. Bug reports
surface as "but I added that variant" and the answer is "you added it
to a file no one imports".

Right thing : when forking, ALWAYS update import paths atomically with
the fork creation. Run a project-wide replace :

```bash
# update imports
git grep -l '@/components/ui/sidebar' | xargs sed -i 's|@/components/ui/sidebar|@/lib/components/sidebar/sidebar|g'
git diff --stat
pnpm test
git add -A && git commit -m "refactor: fork Sidebar to lib/components/sidebar"
```

Verify with a grep that no consumer still imports from the upstream
path :

```bash
git grep '@/components/ui/sidebar' || echo 'fork imports clean'
```

Document the fork policy in `components/ui/README.md` so the next
maintainer (or future-you) knows the upstream file is reference-only.

## 10. Trusting the CLI Confirmation Prompt to Catch Mistakes

WRONG : running `pnpm dlx shadcn@latest add button` without
`--overwrite`, getting a confirmation prompt "components/ui/button.tsx
exists. Overwrite?", clicking yes by reflex, then complaining the
overwrite happened.

Why this fails : the confirmation prompt is a single yes/no. It does
NOT show a diff. It does NOT distinguish "trivial" from "extensive"
local changes. Once accepted, the file is replaced. The prompt is a
speed bump, not a safety net. Worse : running `add button -y` skips
the prompt entirely, and CI scripts that run `shadcn add` invariably
include `-y` to avoid hanging on the prompt.

Right thing : ALWAYS use `--diff` BEFORE accepting any overwrite, even
the interactive prompt's. The prompt is not the review step ; the
`--diff` output is. Treat the interactive prompt as equivalent to
`--overwrite` for safety analysis.

Project-level enforcement : disable the interactive prompt in team
scripts by always passing `--dry-run` then `--diff` then `--overwrite`
explicitly. The explicit three-step ritual is harder to misclick than
a single y/N prompt.
