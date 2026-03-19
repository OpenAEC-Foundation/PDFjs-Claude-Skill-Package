# DECISIONS

Architectural and process decisions with rationale. Each decision is numbered and immutable once recorded. New decisions may supersede old ones but old ones are never deleted.

---

## D-001: 7-Phase Research-First Methodology
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Need a structured approach to build high-quality skills
**Decision**: Adopt the 7-phase methodology proven in the ERPNext, Blender-Bonsai, and Tauri 2 Skill Packages
**Rationale**: ERPNext project successfully produced 28 domain skills, Blender-Bonsai produced 73 skills, Tauri 2 produced 27 skills with this approach. Research-first prevents hallucinated content.
**Reference**: https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package/blob/main/WAY_OF_WORK.md

## D-002: Single Technology Package
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: PDF.js is one technology — Mozilla's PDF viewer/renderer library
**Decision**: No per-technology separation needed. All skills share the `pdfjs-` prefix under a single `skills/source/` tree.
**Rationale**: PDF.js is a single library. The viewer API, worker API, and rendering pipeline are all part of the same framework, not separate technologies.

## D-003: English-Only Skills
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Team works primarily in Dutch, skills target international audience
**Decision**: ALL skill content in English only
**Rationale**: Skills are instructions for Claude, not end-user documentation. Claude reads English and responds in any language. Bilingual skills double maintenance with zero functional benefit. Proven in ERPNext, Blender, and Tauri projects.

## D-004: Claude Code Agent Tool for Orchestration
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Need to produce skills efficiently. Windows environment, no oa-cli available.
**Decision**: Use Claude Code Agent tool for parallel execution instead of oa-cli
**Rationale**: Windows environment does not support oa-cli (requires WSL/Linux). Claude Code Agent tool provides native parallelism within the Claude Code session. Simpler setup, no tmux/fcntl dependencies.

## D-005: MIT License
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Need to choose open-source license
**Decision**: MIT License
**Rationale**: Most permissive, maximizes adoption. Consistent with OpenAEC Foundation philosophy.

## D-006: ROADMAP.md as Single Source of Truth
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Need to track project status across multiple sessions and agents
**Decision**: ROADMAP.md is the ONLY place where project status is tracked
**Rationale**: Multiple status locations cause drift and confusion. Single source prevents "which is current?" questions. Proven in ERPNext, Blender, and Tauri projects.

## D-007: pdfjs-dist 4.x Only
**Date**: 2026-03-19
**Status**: SUPERSEDED by D-008
**Context**: PDF.js has multiple major versions with different APIs and distribution methods
**Decision**: All code targets pdfjs-dist 4.x exclusively. No legacy version coverage.
**Rationale**: pdfjs-dist 4.x is the current major version with significant API changes from v3 and earlier. The rendering API, worker setup, and layer APIs have evolved substantially. Supporting older versions would dilute quality and create confusion.

## D-008: pdfjs-dist 5.x Target
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Phase 2 research revealed pdfjs-dist is now at v5.5.207 (March 2026). Version 4.x is no longer maintained. Key API changes: `renderTextLayer()` deprecated in favor of `TextLayer` class, `AnnotationLayer` follows same pattern, private class fields adopted, deprecated options removed.
**Decision**: All code targets pdfjs-dist 5.x exclusively. No v4 or earlier coverage.
**Rationale**: v5.x is the current major version (latest: 5.5.207). v4.x is unmaintained. The class-based TextLayer/AnnotationLayer API is the only supported approach. Targeting v5 ensures skills stay relevant.

## D-009: SKILL.md < 500 Lines
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Skills need to be scannable and focused
**Decision**: SKILL.md files must be under 500 lines. Heavy content goes in references/ directory.
**Rationale**: Long files reduce skill effectiveness. The references/ pattern separates quick-reference from deep-dive content.

## D-010: WebFetch Verification Required
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: AI training data may contain outdated or incorrect API information
**Decision**: All code examples must be verified against official documentation via WebFetch before inclusion in skills
**Rationale**: PDF.js API changes frequently between versions. Training data often contains v2/v3 patterns that are incorrect for v5.x.

## D-011: Merge core-rendering-pipeline into syntax-page-rendering
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Masterplan refinement — rendering pipeline content overlaps with page rendering
**Decision**: Merge `core-rendering-pipeline` into `syntax-page-rendering`. No standalone core rendering skill.
**Rationale**: Rendering pipeline is best understood in context of page rendering. Separate core skill would duplicate viewport/canvas content.

## D-012: Merge impl-text-search into impl-custom-viewer
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Masterplan refinement — text search is a viewer feature
**Decision**: Merge `impl-text-search` into `impl-custom-viewer`.
**Rationale**: Text search is not standalone; it's part of the viewer feature set.

## D-013: Merge impl-form-handling into syntax-annotation-layer
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Masterplan refinement — form fields are Widget annotations
**Decision**: Merge `impl-form-handling` into `syntax-annotation-layer`.
**Rationale**: Form fields are Widget annotations. AnnotationStorage is part of the annotation API. Separate skill too thin.

## D-014: Merge impl-print and impl-thumbnails into impl-custom-viewer
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Masterplan refinement — print and thumbnails are viewer features
**Decision**: Merge `impl-print` and `impl-thumbnails` into `impl-custom-viewer`.
**Rationale**: Print and thumbnails are viewer features, not standalone implementation skills.

## D-015: Drop pdfjs-syntax-typescript skill
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Masterplan refinement added a TypeScript skill (D-05 in masterplan), but during Phase 5 execution it was determined that TypeScript patterns are adequately covered within each individual skill's code examples
**Decision**: Drop `pdfjs-syntax-typescript` as a standalone skill. TypeScript patterns are integrated into all other skills. Final count: 13 skills (not 14).
**Rationale**: Every skill already includes TypeScript code examples with proper type imports. A standalone TypeScript skill would only repeat content already present across the package.

## D-016: Topic research conducted inline during skill creation
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Phase 4 topic research was done inline via WebFetch during Phase 5 agent execution rather than as separate research documents
**Decision**: Topic-specific research was conducted inline during skill creation via WebFetch, not as separate documents in docs/research/topic-research/.
**Rationale**: The comprehensive vooronderzoek (1623 lines) provided sufficient foundation. Agents used WebFetch for topic-specific verification during skill writing. This is consistent with the workflow which allows Phase 4 to run concurrently with Phase 5.
