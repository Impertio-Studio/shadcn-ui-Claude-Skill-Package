# Architectural Decisions

Numbered decisions (D-XXX) with rationale. Immutable once recorded — new decisions can supersede but never delete old ones.

---

## D-001: English-Only Skills

- **Date**: 2026-05-19
- **Decision**: All skill content MUST be in English only.
- **Rationale**: Skills are instructions FOR Claude, not for end users. Claude reads English and responds in the user's language. Bilingual skills double maintenance with zero functional benefit. Proven in ERPNext (28 skills) and Blender-Bonsai (73 skills).
- **Consequence**: No translations needed. All descriptions, code comments, and documentation in English.

---

## D-002: MIT License

- **Date**: 2026-05-19
- **Decision**: Project uses MIT License.
- **Rationale**: Most permissive license, maximizes adoption. Consistent with OpenAEC Foundation standards.
- **Consequence**: No commercial restrictions. Community-friendly.

---

## D-003: SKILL.md Under 500 Lines

- **Date**: 2026-05-19
- **Decision**: SKILL.md files MUST be under 500 lines.
- **Rationale**: Keeps main skill focused on decision trees and quick reference. Heavy content belongs in references/ directory. Proven optimal in ERPNext (180-427 lines per skill).
- **Consequence**: Complex topics split between SKILL.md (quick reference + patterns) and references/ (complete API, examples, anti-patterns).

---

## D-004: 7-Phase Research-First Methodology

- **Date**: 2026-05-19
- **Decision**: Follow the 7-phase research-first development methodology.
- **Rationale**: Proven in ERPNext (28 skills), Blender-Bonsai (73 skills), and Tauri 2 (27 skills). Research prevents hallucination. Deterministic skills require deep understanding.
- **Consequence**: No skill creation without prior deep research. Phases are sequential with defined exit criteria.

---

## D-005: ROADMAP.md as Single Source of Truth

- **Date**: 2026-05-19
- **Decision**: ROADMAP.md is the ONLY place for project status.
- **Rationale**: Multiple status locations cause drift and "which is current?" confusion. Single source of truth enables reliable session recovery.
- **Consequence**: Never duplicate status in CLAUDE.md or other files. All status references point to ROADMAP.md.

---

## D-006: WebFetch for Research Verification

- **Date**: 2026-05-19
- **Decision**: Use WebFetch to verify all code examples against latest official documentation.
- **Rationale**: Technology APIs evolve. Training data may be stale. WebFetch ensures latest official docs are consulted, not outdated cached knowledge.
- **Consequence**: All code examples must be verified against current official documentation before inclusion in skills.

---

## D-007: GitHub Publication Under OpenAEC Foundation

- **Date**: 2026-05-19
- **Decision**: Publish all skill packages under the OpenAEC Foundation GitHub organization.
- **Rationale**: Centralized, consistent branding. Community ownership. Discoverability.
- **Consequence**: All repos follow OpenAEC naming conventions and include social preview banners with OpenAEC branding.

---

## D-008: Skip Phase 4 topic-research for foundational core batches (B1)

- **Date**: 2026-05-19
- **Decision**: Skip Phase 4 topic-research for Batch 1 (`shadcn-core-architecture`, `shadcn-core-stack`, `shadcn-core-cli`).
- **Rationale**: vooronderzoek-shadcn.md (5613 words) extensively covers architecture (§1), stack composition (§1 table), and CLI surface (§4). Per BOOTSTRAP §6.3 skip-criteria ("vooronderzoek >40 doc-pages + helder onderbouwd: skip Phase 4 voor die batch"), additional topic-research adds no information. Workers reference vooronderzoek directly + WebFetch primary URLs as needed.
- **Consequence**: B1 workers receive bundle pointing to vooronderzoek-shadcn.md sections + SOURCES.md primary URLs. NO `docs/research/topic-research/shadcn-core-*-research.md` files for B1.
- **Re-evaluate**: Phase 4 topic-research re-enabled from B3 onwards (component-specific syntax skills) where vooronderzoek depth per component is shallower.

---

## D-010: In-process Agent dispatch for Phase 5 B2-B14 (one-time deviation)

- **Date**: 2026-05-19
- **Decision**: Use in-process Agent tool (3 parallel opus agents per batch) for Phase 5 batches B2-B14 instead of tmux-orchestration workers.
- **Rationale**: tmux workers worker-1/2/3 are shared via cross-workspace orchestration with TailwindCSS-Claude-Skill-Package. External orchestrator controls dispatch timing for those workers, making predictable shadcn batch progression impossible in this session. In-process Agent tool delivers same parallel-execution model (3 agents per batch, separate file-scopes) without the cross-workspace contention.
- **Consequence**: B2 onwards uses Agent tool. tmo task tracking deprecated for these batches (replaced by TaskCreate). Quality-gate (APPROVE / RE-INSTRUCT / REPLACE) still mandatory per batch. File-scope isolation still enforced via per-skill output directory. Supersedes D-009 for B2-B14 only.
- **Re-evaluate**: D-009 (tmux backbone) remains valid for future skill packages where workers are not externally shared.

---

## D-009: tmux-orchestration as Phase 5 execution backbone

- **Date**: 2026-05-19
- **Decision**: All Phase 5 skill creation runs via tmux-orchestration skill with 3 persistent workers (skill-builder role) in VS Code panels.
- **Rationale**: BOOTSTRAP §6 hard requirement. Persistent workers with file-scope-isolation + quality-gate-loop is proven scalable (Blender-Bonsai 73 skills, Frappe 61 skills). In-process Agent tool lacks file-scope-lock guarantees.
- **Consequence**: state/ dir gitignored, roles/skill-builder.md in state/, .vscode/tasks.json tracked for shared spawn config, workers respond in Nederlands per user-language setting.
