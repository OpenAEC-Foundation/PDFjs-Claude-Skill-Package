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
