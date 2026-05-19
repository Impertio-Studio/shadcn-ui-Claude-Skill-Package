# ROADMAP — shadcn ui Skill Package

## Current Status

| Phase | Description | Status | Progress |
|-------|------------|--------|----------|
| Phase 1 | Raw Masterplan | ✅ Done | 100% |
| Phase 2 | Deep Research (Vooronderzoek) | ✅ Done | 100% |
| Phase 3 | Masterplan Refinement | ✅ Done | 100% |
| Phase 4 | Topic-Specific Research | ⏭ Skipped for B1 (D-008), per-batch from B3+ | 7% |
| Phase 5 | Skill Creation | 🔄 In progress (B1+B2 done, B3-B14 pending) | 14% |
| Phase 6 | Validation | ⏳ Pending | 0% |
| Phase 7 | Publication | ⏳ Pending | 0% |

**Overall Progress**: 43% (Phase 1-3 complete, B1+B2 skill-creation done, 6/42 skills built, core/ category 100%)

## Next Steps

1. Phase 5 batch B2 : core-theming, core-registry, core-blocks
   - 3 tmux workers via skill-builder role
   - QG every reply
2. Phase 5 batch B3+ : syntax skills (require Phase 4 topic-research per BOOTSTRAP §6.3, NOT skipped from B3 onwards)
3. Continue through B14 (agents)

**External orchestrator note** : workers (worker-1/2/3 tmux sessions) are shared across shadcn ui + Tailwind CSS skill packages via cross-workspace orchestration. B1 was completed by external dispatch. Coordinate with main orchestrator session for B2 dispatch timing.

## Skill Summary

| Category | Estimated | Created | Validated |
|----------|-----------|---------|-----------|
| core/ | 6 | 6 | 6 |
| syntax/ | 18 | 0 | 0 |
| impl/ | 7 | 0 | 0 |
| errors/ | 7 | 0 | 0 |
| agents/ | 4 | 0 | 0 |
| **Total** | **42** | **6** | **6** |

## Changelog

### Phase 5 : Batch 2 done (2026-05-19)
- 3 shadcn-core skills built via in-process Agent dispatch (D-010 deviation)
- shadcn-core-theming (SHA 1dbce94, 325 lines + 3 refs, full token catalog v3+v4)
- shadcn-core-registry (SHA 4ff2558, 380 lines + 3 refs, components.json schema deep-dive)
- shadcn-core-blocks (SHA f1fdb86, 295 lines + 3 refs, blocks as distinct distribution surface)
- All validate-frontmatter + validate-line-count + em-dash check: OK
- core/ category 100% complete (6/6)

### Phase 5 : Batch 1 done (2026-05-19)
- 3 shadcn-core skills built + committed
- shadcn-core-architecture (SHA e2ef290, 289 lines + 3 refs)
- shadcn-core-stack (SHA 2c54e29, 338 lines + 3 refs)
- shadcn-core-cli (SHA a04b7e5, 427 lines + 3 refs)
- All validate-frontmatter.js + validate-line-count.js exit 0
- No em-dash in headings
- Workers built via cross-workspace orchestration (shared with Tailwind CSS pkg)

### Phase 3 : Masterplan refined (2026-05-19)
- 20 Refinement Decisions (8 ADD / 1 SPLIT / 2 MERGE / 1 RENAME / 1 DROP)
- Final inventory: 42 skills (core 6 / syntax 18 / impl 7 / errors 7 / agents 4)
- 14-batch execution plan with file-scope-per-batch
- Agent-prompt template + sample for shadcn-syntax-button
- D-008 + D-009 (skip Phase 4 for B1, tmux-orchestration backbone)

### Phase 2 : Deep Research complete (2026-05-19)
- vooronderzoek-shadcn.md : 5613 words, 13 sections, 59 components catalogued
- 15 GitHub issues referenced for anti-patterns
- 15 newly-discovered sub-topics
- SOURCES.md : 15 URLs Last-Verified 2026-05-19, 4 new sources added

### Phase 1 : Raw Masterplan complete (2026-05-19)
- Template placeholders substituted in CLAUDE.md, ROADMAP.md, REQUIREMENTS.md, SOURCES.md, HANDOFF.md, README.md
- SOURCES.md populated with 21 approved primary/secondary sources (ui.shadcn.com/docs, github.com/shadcn-ui/ui, radix-ui.com/primitives, github.com/joe-bell/cva, plus integration sources)
- Raw masterplan written in `docs/masterplan/shadcn-masterplan.md` with ~33 skills across 5 categories
- Identified shadcn paradigm note : CLI-driven, NOT library, components copied to consumer project
- Underlying stack composition map captured (Radix + cva + tailwind-merge + clsx + lucide + integrations)

### Phase 0 : Infrastructure (2026-05-19)
- Repository structure created
- Core files initialized (CLAUDE.md, ROADMAP.md, REQUIREMENTS.md, DECISIONS.md, SOURCES.md, WAY_OF_WORK.md, LESSONS.md, CHANGELOG.md)
- Skill category directories created
