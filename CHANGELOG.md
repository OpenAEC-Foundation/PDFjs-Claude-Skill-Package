# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added
- Project initialized with 7-phase research-first methodology
- Core documentation files (CLAUDE.md, ROADMAP.md, DECISIONS.md, etc.)
- Directory structure for skills and research
- Raw masterplan with 17 preliminary skills
- Refined masterplan: 13 definitive skills across 5 batches
- **13 complete skills** across 5 categories:
  - `pdfjs-core`: architecture (1 skill)
  - `pdfjs-syntax`: worker-setup, document-loading, page-rendering, text-layer, annotation-layer (5 skills)
  - `pdfjs-impl`: custom-viewer, bundler-integration (2 skills)
  - `pdfjs-errors`: worker, rendering, document (3 skills)
  - `pdfjs-agents`: review, project-scaffolder (2 skills)
- Each skill includes SKILL.md + references/ (methods.md, examples.md, anti-patterns.md)
- All skills target pdfjs-dist 5.x exclusively
