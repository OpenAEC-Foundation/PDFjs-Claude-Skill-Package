# Lessons Learned

Observations and findings captured during skill package development.

---

## L-001: Single Technology Simplifies Package Structure

- **Date**: 2026-03-19
- **Context**: PDF.js is one library with a viewer/renderer API surface.
- **Finding**: No need for per-technology separation. `skills/source/pdfjs-{category}/` is sufficient. The entire package can use a flat category structure without layered technology namespacing.

---

## L-002: Worker Setup is the Entry Gate

- **Date**: 2026-03-19
- **Context**: PDF.js requires a Web Worker for PDF parsing and rendering.
- **Finding**: Every skill that involves document loading MUST include worker setup. `GlobalWorkerOptions.workerSrc` MUST be set before any `getDocument()` call. The worker version MUST match the pdfjs-dist version exactly. This is the most common source of "PDF.js not working" issues. Skills must emphasize this pattern consistently.

---

## L-003: Render Task Lifecycle is Critical

- **Date**: 2026-03-19
- **Context**: PDF.js uses `RenderTask` objects for canvas rendering.
- **Finding**: ALWAYS cancel a previous render task before starting a new one. Failing to cancel causes memory leaks, visual artifacts, and race conditions. Every rendering skill MUST show the cancel-then-render pattern.

---

## L-004: pdfjs-dist is at v5.x, not v4.x

- **Date**: 2026-03-19
- **Context**: Phase 2 research discovered pdfjs-dist latest is 5.5.207 (March 2026). v4.x is no longer maintained.
- **Finding**: Initial D-007 decision targeted v4.x — WRONG. Corrected to D-008 targeting v5.x. Key v5 changes: `renderTextLayer()` replaced by `TextLayer` class with `render()` method, `AnnotationLayer` follows same class-based pattern, private class fields adopted, deprecated options removed. ALWAYS verify version before writing skills.

---

## L-005: TextLayer and AnnotationLayer are Class-Based in v5

- **Date**: 2026-03-19
- **Context**: PDF.js v5 replaced function-based APIs with class-based ones.
- **Finding**: Old `renderTextLayer({...})` is deprecated. New pattern: `new TextLayer({textContentSource, viewport, container})` then `await textLayer.render()`. Same pattern for `AnnotationLayer`. Skills MUST use the class-based API exclusively. The `update()` method handles viewport changes without full re-render.

---

## L-006: PDF.js Has Three Architectural Layers

- **Date**: 2026-03-19
- **Context**: Research on PDF.js architecture revealed clear layering.
- **Finding**: Core (pdf.worker.mjs — binary parsing, no public API), Display (pdf.mjs — getDocument, PDFDocumentProxy, PDFPageProxy), Viewer (viewer.mjs — UI components). Skills should focus on Display layer API. Core layer is internal. Viewer layer is reference implementation only.
