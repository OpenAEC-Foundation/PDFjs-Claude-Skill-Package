# PDF.js Claude Skill Package — Skill Index

> 13 deterministic skills for PDF.js (pdfjs-dist 5.x) development with Claude

---

## Core (1 skill)

| Skill | Description |
|-------|-------------|
| [pdfjs-core-architecture](skills/source/pdfjs-core/pdfjs-core-architecture/SKILL.md) | PDF.js three-layer architecture, worker thread model, component hierarchy, pdfjs-dist package structure |

## Syntax (5 skills)

| Skill | Description |
|-------|-------------|
| [pdfjs-syntax-worker-setup](skills/source/pdfjs-syntax/pdfjs-syntax-worker-setup/SKILL.md) | GlobalWorkerOptions.workerSrc configuration, CDN/bundler patterns, CMap and font setup |
| [pdfjs-syntax-document-loading](skills/source/pdfjs-syntax/pdfjs-syntax-document-loading/SKILL.md) | getDocument() with all source types, PDFDocumentProxy/PDFPageProxy APIs, password handling |
| [pdfjs-syntax-page-rendering](skills/source/pdfjs-syntax/pdfjs-syntax-page-rendering/SKILL.md) | page.render() pipeline, RenderTask lifecycle, viewport creation, high-DPI canvas scaling |
| [pdfjs-syntax-text-layer](skills/source/pdfjs-syntax/pdfjs-syntax-text-layer/SKILL.md) | TextLayer class (v5), text content extraction, CSS overlay positioning, text selection |
| [pdfjs-syntax-annotation-layer](skills/source/pdfjs-syntax/pdfjs-syntax-annotation-layer/SKILL.md) | AnnotationLayer class (v5), annotation types, link handling, form fields, AnnotationStorage |

## Implementation (2 skills)

| Skill | Description |
|-------|-------------|
| [pdfjs-impl-custom-viewer](skills/source/pdfjs-impl/pdfjs-impl-custom-viewer/SKILL.md) | Complete PDF viewer with navigation, zoom, search, print, thumbnails, lazy loading |
| [pdfjs-impl-bundler-integration](skills/source/pdfjs-impl/pdfjs-impl-bundler-integration/SKILL.md) | Webpack, Vite, Rollup, Next.js, Nuxt.js configuration for pdfjs-dist |

## Error Handling (3 skills)

| Skill | Description |
|-------|-------------|
| [pdfjs-errors-worker](skills/source/pdfjs-errors/pdfjs-errors-worker/SKILL.md) | Worker loading failures, version mismatch, CORS/CSP issues, fake worker fallback |
| [pdfjs-errors-rendering](skills/source/pdfjs-errors/pdfjs-errors-rendering/SKILL.md) | Blurry text, render task races, memory issues, text/annotation layer problems |
| [pdfjs-errors-document](skills/source/pdfjs-errors/pdfjs-errors-document/SKILL.md) | InvalidPDFException, MissingPDFException, PasswordException, CMap/font errors |

## Agents (2 skills)

| Skill | Description |
|-------|-------------|
| [pdfjs-agents-review](skills/source/pdfjs-agents/pdfjs-agents-review/SKILL.md) | Code validation checklist: worker setup, DPI, cancellation, layers, v5 compliance |
| [pdfjs-agents-project-scaffolder](skills/source/pdfjs-agents/pdfjs-agents-project-scaffolder/SKILL.md) | Generate complete PDF.js projects: vanilla, webpack, Vite, Next.js with TypeScript |

---

## Skill Structure

Each skill contains:
```
skill-name/
├── SKILL.md              # Main skill file (<500 lines)
└── references/
    ├── methods.md         # API signatures and parameters
    ├── examples.md        # Working code examples
    └── anti-patterns.md   # What NOT to do
```

## Version Target

All skills target **pdfjs-dist 5.x** exclusively (current: 5.5.207, March 2026).
