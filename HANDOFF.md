# Handoff : shadcn-ui-Claude-Skill-Package

> Last updated : 2026-05-19
> Generated from Skill-Package-Workflow-Template BOOTSTRAP-RUNBOOK

## Status

- **Phase** : Phase 5 in progress (Batch 1 done, B2-B14 pending)
- **Skills** : 3 / 42 (B1 complete : architecture, stack, cli)
- **GitHub remote** : https://github.com/OpenAEC-Foundation/shadcn-ui-Claude-Skill-Package (to be created)
- **Last commit** : a04b7e5 feat(skill): shadcn-core-cli
- **Compliance score** : n/a (validation runs in Phase 6)

## What is done

- Phase 1 raw masterplan + template substitution (commit e7e960f)
- Phase 2 deep research vooronderzoek-shadcn.md (5613 words, 59 components, commit 1dc9e31)
- Phase 3 masterplan refinement (20 decisions, 42 skills, 14 batches, commit 53e2377)
- Phase 5 Batch 1 : shadcn-core-architecture (e2ef290), shadcn-core-stack (2c54e29), shadcn-core-cli (a04b7e5)
- All B1 SKILL.md < 500 lines, validate-frontmatter + validate-line-count + em-dash check : OK
- DECISIONS.md D-008 + D-009 logged

## What is open

- Phase 5 B2 : core-theming, core-registry, core-blocks
- Phase 5 B3-B8 : syntax skills (18 total, 6 batches)
- Phase 5 B9-B11 : impl skills (7 total, 3 batches)
- Phase 5 B12-B14 : errors + agents (11 total, 3 batches)
- Phase 6/7 : validation + publication (deferred per user-instruction "STOP na Phase 5")

## Next-session entry point

Open this workspace in VS Code and run :

```
Lees BOOTSTRAP-RUNBOOK.md van Skill-Package-Workflow-Template en hervat Phase 5 vanaf Batch 2.
```

## Active orchestration (Phase 5)

**External orchestrator running** : workers worker-1/2/3 are shared via cross-workspace tmux-orchestration with TailwindCSS-Claude-Skill-Package. B1 was completed via that orchestrator. Workers currently dispatching Tailwind batches.

| Worker | Last skill done | Status | tmo task ID |
|--------|-----------------|--------|-------------|
| worker-1 | shadcn-core-architecture | working on Tailwind tasks | T-1 done |
| worker-2 | shadcn-core-stack | working on Tailwind tasks | T-2 done |
| worker-3 | shadcn-core-cli | working on Tailwind tasks | T-3 done |

## Decisions made this session (DECISIONS.md)

- D-008 : Skip Phase 4 topic-research for foundational core batch B1 (vooronderzoek already covers extensively)
- D-009 : tmux-orchestration as Phase 5 execution backbone

## How to resume B2

Option A : continue via existing tmux workers (if external orchestrator dispatches shadcn batches)
Option B : dispatch B2 via in-process Agent tool (3 parallel opus agents, faster, no tmux overhead)
Option C : kill external orchestration + re-spawn dedicated shadcn workers

Recommended : Option B for predictable per-session progress.

## Decisions blocking next step

- (none active : Phase 2 can start immediately)

## Special notes

- shadcn ui is CLI-driven, NOT a library. Components are copied via `npx shadcn@latest add <name>` into the consumer project. Skills MUST teach this paradigm explicitly.
- Stack-composition matters : Radix UI primitives + cva + tailwind-merge + Tailwind CSS. Skills MUST cite which underlying primitive the component derives from.
- Cross-pkg refs (companion-skills) : Tailwind-pkg + React-pkg available locally, NEVER auto-load. Reference via Read-on-demand only.
- Em-dash ban applies to section headings within all SKILL.md files.

---

**Anti-pattern caveat** : HANDOFF.md MOET synchroon blijven met ROADMAP.md. Bij elke phase-completion : update beide in dezelfde commit. Cross-Tech L-016 toonde dat drift tussen HANDOFF en ROADMAP leidt tot foute aannames.
