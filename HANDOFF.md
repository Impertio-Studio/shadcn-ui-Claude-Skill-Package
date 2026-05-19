# Handoff : shadcn-ui-Claude-Skill-Package

> Last updated : 2026-05-19
> Generated from Skill-Package-Workflow-Template BOOTSTRAP-RUNBOOK

## Status

- **Phase** : Phase 1 complete (raw masterplan written, ready for Phase 2 deep research)
- **Skills** : 0 / ~28-32 planned
- **GitHub remote** : https://github.com/OpenAEC-Foundation/shadcn-ui-Claude-Skill-Package (to be created)
- **Last commit** : (see git log)
- **Compliance score** : n/a (validation runs in Phase 6)

## What is done

- Workspace bootstrap (governance files + skill category dirs)
- Phase 1 raw masterplan with topic inventory across 5 categories
- Template placeholder substitution in core files (CLAUDE.md, ROADMAP.md, REQUIREMENTS.md, SOURCES.md)
- SOURCES.md populated with 21 approved primary/secondary sources
- Initial commit pushed to bootstrap

## What is open

- Phase 2 : deep research single-agent dispatch -> docs/research/vooronderzoek-shadcn.md
- Phase 3 : masterplan refinement + user-checkpoint before Phase 4/5
- Phase 4+5 : topic-research + skill creation via tmux-orchestration (3 workers)
- Phase 6/7 : validation + publication (deferred per user-instruction "STOP na Phase 5")

## Next-session entry point

Open this workspace in VS Code and run :

```
Lees START-PROMPT.md en hervat vanaf Phase 2 (deep research).
```

Of expliciet :

```
Lees BOOTSTRAP-RUNBOOK.md van Skill-Package-Workflow-Template en hervat phase 2.
```

## Active batch (if in Phase 5)

| Worker | Skill | Status | tmo task ID |
|--------|-------|--------|-------------|
| worker-1 | (none yet) | n/a | n/a |
| worker-2 | (none yet) | n/a | n/a |
| worker-3 | (none yet) | n/a | n/a |

## Decisions blocking next step

- (none active : Phase 2 can start immediately)

## Special notes

- shadcn ui is CLI-driven, NOT a library. Components are copied via `npx shadcn@latest add <name>` into the consumer project. Skills MUST teach this paradigm explicitly.
- Stack-composition matters : Radix UI primitives + cva + tailwind-merge + Tailwind CSS. Skills MUST cite which underlying primitive the component derives from.
- Cross-pkg refs (companion-skills) : Tailwind-pkg + React-pkg available locally, NEVER auto-load. Reference via Read-on-demand only.
- Em-dash ban applies to section headings within all SKILL.md files.

---

**Anti-pattern caveat** : HANDOFF.md MOET synchroon blijven met ROADMAP.md. Bij elke phase-completion : update beide in dezelfde commit. Cross-Tech L-016 toonde dat drift tussen HANDOFF en ROADMAP leidt tot foute aannames.
