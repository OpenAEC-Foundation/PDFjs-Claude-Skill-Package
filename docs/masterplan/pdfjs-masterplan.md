# PDF.js Skill Package — Raw Masterplan

## Status

Phase 1 raw masterplan. Pending refinement after vooronderzoek (Phase 3).
Date: 2026-03-19

---

## Preliminary Skill Inventory (17 skills)

### pdfjs-core/ (2 skills)

| Name | Scope | Key APIs | Complexity | Dependencies |
|------|-------|----------|------------|-------------|
| `pdfjs-core-architecture` | PDF.js architecture; worker thread model; component hierarchy; pdfjs-dist package structure; version info | PDFDocumentProxy, PDFPageProxy, GlobalWorkerOptions | M | None |
| `pdfjs-core-rendering-pipeline` | Rendering pipeline overview; canvas vs SVG; viewport system; layer stacking (canvas → text → annotation); DPI handling | PageViewport, RenderTask, devicePixelRatio | M | core-architecture |

### pdfjs-syntax/ (5 skills)

| Name | Scope | Key APIs | Complexity | Dependencies |
|------|-------|----------|------------|-------------|
| `pdfjs-syntax-document-loading` | getDocument() with all source types; PDFDocumentLoadingTask; PDFDocumentProxy properties/methods; PDFPageProxy properties/methods; worker setup requirement | getDocument(), PDFDocumentProxy, PDFPageProxy, PDFDocumentLoadingTask | L | core-architecture |
| `pdfjs-syntax-page-rendering` | page.render() with canvas context; RenderTask lifecycle; viewport creation; high-DPI canvas scaling; render cancellation pattern | page.render(), page.getViewport(), RenderTask, CanvasRenderingContext2D | M | syntax-document-loading |
| `pdfjs-syntax-text-layer` | TextLayer class; getTextContent(); TextContent structure; CSS overlay positioning; text selection; text extraction | TextLayer, page.getTextContent(), TextContent, TextItem | M | syntax-page-rendering |
| `pdfjs-syntax-annotation-layer` | AnnotationLayer class; getAnnotations(); annotation types; link handling; AnnotationEditorLayer; AnnotationStorage | AnnotationLayer, page.getAnnotations(), AnnotationStorage, AnnotationEditorLayer | M | syntax-page-rendering |
| `pdfjs-syntax-worker-setup` | GlobalWorkerOptions.workerSrc; CDN URLs; webpack/vite/rollup config; fake worker mode; version matching; CMap and font config | GlobalWorkerOptions, workerSrc, CMaps, standard_fonts | M | core-architecture |

### pdfjs-impl/ (5 skills)

| Name | Scope | Key APIs | Complexity | Dependencies |
|------|-------|----------|------------|-------------|
| `pdfjs-impl-custom-viewer` | Building a complete viewer; page navigation; zoom controls; scroll-based loading; lazy rendering; virtual scrolling | All viewer patterns, IntersectionObserver | L | syntax-page-rendering, syntax-text-layer, syntax-annotation-layer |
| `pdfjs-impl-text-search` | Text search across pages; highlighting matches; search navigation; FindController patterns | getTextContent(), text matching, highlight overlay | M | syntax-text-layer |
| `pdfjs-impl-print` | Print functionality; print-quality rendering; CSS print media; page formatting; print dialog integration | High-resolution canvas rendering, CSS @media print | M | syntax-page-rendering |
| `pdfjs-impl-thumbnails` | Thumbnail generation; thumbnail caching; thumbnail navigation; lazy thumbnail rendering | page.render() at reduced scale, canvas thumbnails | S | syntax-page-rendering |
| `pdfjs-impl-form-handling` | Interactive form fields; AnnotationStorage; form data extraction; form field rendering; form submission patterns | AnnotationStorage, Widget annotations, form field types | M | syntax-annotation-layer |

### pdfjs-errors/ (3 skills)

| Name | Scope | Key APIs | Complexity | Dependencies |
|------|-------|----------|------------|-------------|
| `pdfjs-errors-worker` | Worker loading failures; version mismatch errors; CORS issues; CSP violations; fake worker fallback; missing worker diagnostics | Worker error messages, GlobalWorkerOptions | M | syntax-worker-setup |
| `pdfjs-errors-rendering` | Canvas rendering errors; DPI issues; blurry text; render task failures; memory issues; concurrent render conflicts | RenderTask errors, canvas errors | M | syntax-page-rendering |
| `pdfjs-errors-document` | Document loading failures; corrupt PDFs; password-protected PDFs; network errors; missing CMap errors; font loading issues | PasswordException, InvalidPDFException, MissingPDFException | M | syntax-document-loading |

### pdfjs-agents/ (2 skills)

| Name | Scope | Key APIs | Complexity | Dependencies |
|------|-------|----------|------------|-------------|
| `pdfjs-agents-review` | Validation checklist for generated PDF.js code; worker setup verification; render task cancellation check; DPI handling check; anti-pattern detection | All validation rules | M | ALL syntax + impl skills |
| `pdfjs-agents-project-scaffolder` | Generate complete PDF.js project structure; configure worker; set up rendering pipeline; configure text/annotation layers; bundler integration | All scaffolding patterns | L | ALL core + syntax skills |

---

## Preliminary Batch Plan

| Batch | Skills | Count | Dependencies | Notes |
|-------|--------|-------|-------------|-------|
| 1 | `core-architecture`, `core-rendering-pipeline`, `syntax-worker-setup` | 3 | None | Foundation skills |
| 2 | `syntax-document-loading`, `syntax-page-rendering` | 2 | Batch 1 | Core syntax patterns |
| 3 | `syntax-text-layer`, `syntax-annotation-layer`, `impl-thumbnails` | 3 | Batch 2 | Layer APIs + simple impl |
| 4 | `impl-custom-viewer`, `impl-text-search`, `impl-print` | 3 | Batch 2-3 | Implementation patterns |
| 5 | `impl-form-handling`, `errors-worker`, `errors-rendering` | 3 | Batch 2-3 | Forms + error skills |
| 6 | `errors-document`, `agents-review`, `agents-project-scaffolder` | 3 | ALL above | Final batch |

**Total**: 17 skills across 6 batches.

---

## Notes for Phase 3 Refinement

After vooronderzoek, review these decisions:
1. Should `core-rendering-pipeline` merge into `syntax-page-rendering`? (May overlap)
2. Is `impl-form-handling` warranted or is annotation-layer coverage sufficient?
3. Should `impl-text-search` merge into `impl-custom-viewer`?
4. Are there PDF.js v4 API changes that require additional skills?
5. Does AnnotationEditorLayer warrant its own skill?
6. Are there Node.js/server-side patterns worth covering? (REQUIREMENTS says browser-focused)

---

## Reference: Proven Masterplan Format

After Phase 2 vooronderzoek and Phase 3 refinement, this masterplan will be expanded to include:
- Decisions table (merges, additions, removals)
- Exact scope per skill with bullet points
- Research section references with line numbers
- Complete agent prompts for every skill
- Following template: `C:\Users\Freek Heijting\Documents\GitHub\Skill-Package-Workflow-Template\templates\masterplan.md.template`
