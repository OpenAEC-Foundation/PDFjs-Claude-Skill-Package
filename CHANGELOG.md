# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [1.1.0] - 2026-03-20

### Added
- `pdfjs-core-memory-management` — Resource lifecycle, destroy/cleanup patterns, memory leak prevention, large PDF handling
- `pdfjs-impl-forms-and-save` — Interactive PDF form filling, AnnotationStorage, getFieldObjects(), saveDocument()
- Skill count increased from 13 to 15

### Changed
- Updated INDEX.md, README.md, and social preview banner to reflect 15 skills
- Updated ROADMAP.md, DECISIONS.md, and masterplan

## [1.0.0] - 2026-03-19

### Added
- Project initialized with 7-phase research-first methodology
- Core documentation files (CLAUDE.md, ROADMAP.md, DECISIONS.md, etc.)
- Directory structure for skills and research
- Deep research: vooronderzoek-pdfjs.md (1623 lines)
- Refined masterplan: 13 definitive skills across 5 batches
- **13 complete skills** across 5 categories:
  - `pdfjs-core`: architecture (1 skill)
  - `pdfjs-syntax`: worker-setup, document-loading, page-rendering, text-layer, annotation-layer (5 skills)
  - `pdfjs-impl`: custom-viewer, bundler-integration (2 skills)
  - `pdfjs-errors`: worker, rendering, document (3 skills)
  - `pdfjs-agents`: review, project-scaffolder (2 skills)
- Each skill includes SKILL.md + references/ (methods.md, examples.md, anti-patterns.md)
- All skills target pdfjs-dist 5.x exclusively
- INDEX.md with complete skill catalog
- README.md with installation instructions, skill table, and social preview banner
- Social preview banner (1280x640px)
- Compliance audit and remediation pass
