# PDF.js Skill Package — Definitive Masterplan

## Status

Phase 3 complete. Finalized from raw masterplan after research review.
Date: 2026-03-19

---

## Decisions Made During Refinement

| # | Decision | Rationale |
|---|----------|-----------|
| D-01 | **Merged** `core-rendering-pipeline` into `syntax-page-rendering` | Rendering pipeline is best understood in context of page rendering. Separate core skill would duplicate viewport/canvas content. |
| D-02 | **Merged** `impl-text-search` into `impl-custom-viewer` | Text search is a viewer feature, not standalone. Custom viewer skill covers all viewer features including search. |
| D-03 | **Merged** `impl-form-handling` into `syntax-annotation-layer` | Form fields are Widget annotations. AnnotationStorage is part of the annotation API. Separate skill too thin. |
| D-04 | **Merged** `impl-print` and `impl-thumbnails` into `impl-custom-viewer` | Print and thumbnails are viewer features. Custom viewer is the right home. |
| D-05 | **Added** `pdfjs-syntax-typescript` | Research revealed TypeScript setup with pdfjs-dist needs dedicated coverage: type imports, generic patterns, typed getDocument parameters. |
| D-06 | **Updated** version target from 4.x to 5.x | Research discovered pdfjs-dist is at v5.5.207. v4.x unmaintained. D-008 supersedes D-007. |
| D-07 | **Reordered** batches: worker-setup before document-loading | Worker MUST be configured before any getDocument() call. Natural dependency order. |

**Result**: 17 raw skills → **13 definitive skills** (4 merges, 1 addition, 1 removal — see D-015).

---

## Definitive Skill Inventory (13 skills — see D-015 for typescript removal)

### pdfjs-core/ (1 skill)

| Name | Scope | Key APIs | Complexity | Dependencies |
|------|-------|----------|------------|-------------|
| `pdfjs-core-architecture` | PDF.js architecture; three layers (Core/Display/Viewer); worker thread model; pdfjs-dist package structure (build/, cmaps/, standard_fonts/); component hierarchy (PDFDocumentProxy → PDFPageProxy → layers); version info (5.x) | PDFDocumentProxy, PDFPageProxy, GlobalWorkerOptions, TextLayer, AnnotationLayer | M | None |

### pdfjs-syntax/ (5 skills)

| Name | Scope | Key APIs | Complexity | Dependencies |
|------|-------|----------|------------|-------------|
| `pdfjs-syntax-worker-setup` | GlobalWorkerOptions.workerSrc configuration; CDN URLs (cdnjs, unpkg, jsdelivr); webpack/vite/rollup bundler config; fake worker mode; version matching requirement; CMap and standard font configuration; workerParams | GlobalWorkerOptions, workerSrc, cMapUrl, standardFontDataUrl | M | core-architecture |
| `pdfjs-syntax-document-loading` | getDocument() with all source types (URL, ArrayBuffer, TypedArray, data); DocumentInitParameters; PDFDocumentLoadingTask (progress, promise, destroy); PDFDocumentProxy (numPages, getPage, getMetadata, getOutline, getData, getDownloadInfo, destroy); PDFPageProxy (getViewport, render, getTextContent, getAnnotations, cleanup); password-protected PDFs | getDocument(), PDFDocumentLoadingTask, PDFDocumentProxy, PDFPageProxy | L | syntax-worker-setup |
| `pdfjs-syntax-page-rendering` | page.render() with canvas context; RenderTask lifecycle (promise, cancel); viewport creation (scale, rotation, offsetX, offsetY); high-DPI canvas scaling with devicePixelRatio; render cancellation pattern; OffscreenCanvas; rendering pipeline (canvas → text → annotation layer stacking) | page.render(), page.getViewport(), RenderTask, PageViewport, CanvasRenderingContext2D | M | syntax-document-loading |
| `pdfjs-syntax-text-layer` | TextLayer class (v5); constructor params (textContentSource, container, viewport); render()/update()/cancel() methods; getTextContent() and streamTextContent(); TextContent structure (items, styles); CSS overlay positioning; text selection; plain text extraction | TextLayer, page.getTextContent(), page.streamTextContent(), TextContent, TextItem | M | syntax-page-rendering |
| `pdfjs-syntax-annotation-layer` | AnnotationLayer class (v5); render()/update()/cancel(); getAnnotations({intent}); annotation types (Link, Text, Widget, Popup, Highlight); link handling (internal + external); form field rendering; AnnotationStorage; AnnotationEditorLayer for editing; layer stacking order | AnnotationLayer, page.getAnnotations(), AnnotationStorage, AnnotationEditorLayer | M | syntax-page-rendering |

### pdfjs-impl/ (2 skills)

| Name | Scope | Key APIs | Complexity | Dependencies |
|------|-------|----------|------------|-------------|
| `pdfjs-impl-custom-viewer` | Complete PDF viewer from scratch; page navigation (prev/next/goto); zoom controls (in/out/fit-page/fit-width); scroll-based lazy loading with IntersectionObserver; virtual scrolling; thumbnail generation; text search with highlighting; print functionality; NEVER render all pages at once | All viewer patterns, IntersectionObserver, CSS @media print | L | syntax-page-rendering, syntax-text-layer, syntax-annotation-layer |
| `pdfjs-impl-bundler-integration` | Webpack config for worker (worker-loader/asset module); Vite config (vite-plugin-static-copy or import.meta.url); Rollup config; Next.js/Nuxt.js patterns; CMap/font file copying; tree-shaking considerations | Bundler configs, worker resolution, static asset handling | M | syntax-worker-setup |

### pdfjs-errors/ (3 skills)

| Name | Scope | Key APIs | Complexity | Dependencies |
|------|-------|----------|------------|-------------|
| `pdfjs-errors-worker` | Worker loading failures; version mismatch ("API version does not match Worker version"); CORS errors; CSP violations; fake worker fallback; missing worker diagnostics; worker initialization timeout | Worker error messages, GlobalWorkerOptions, version strings | M | syntax-worker-setup |
| `pdfjs-errors-rendering` | Canvas rendering errors; blurry text (missing devicePixelRatio); render task race conditions; memory issues from rendering all pages; concurrent render conflicts; text layer positioning errors; annotation layer z-index issues | RenderTask errors, canvas errors, CSS issues | M | syntax-page-rendering |
| `pdfjs-errors-document` | Document loading failures; InvalidPDFException; MissingPDFException; PasswordException; network/CORS errors; missing CMap data (CJK fonts); font loading failures; corrupt PDF recovery | InvalidPDFException, MissingPDFException, PasswordException, UnknownErrorException | M | syntax-document-loading |

### pdfjs-agents/ (2 skills)

| Name | Scope | Key APIs | Complexity | Dependencies |
|------|-------|----------|------------|-------------|
| `pdfjs-agents-review` | Validation checklist for generated PDF.js code; worker setup verification; render task cancellation check; DPI handling check; layer stacking order; memory management; anti-pattern detection; v5 API compliance | All validation rules from all skills | M | ALL syntax + impl skills |
| `pdfjs-agents-project-scaffolder` | Generate complete PDF.js project; configure worker for chosen bundler; set up rendering pipeline; configure text/annotation layers; HTML/CSS template; TypeScript setup; package.json with correct pdfjs-dist version | All scaffolding patterns | L | ALL core + syntax skills |

---

## Batch Execution Plan (DEFINITIVE)

| Batch | Skills | Count | Dependencies | Notes |
|-------|--------|-------|-------------|-------|
| 1 | `core-architecture`, `syntax-worker-setup`, `syntax-document-loading` | 3 | None | Foundation: architecture + worker + loading |
| 2 | `syntax-page-rendering`, `syntax-text-layer`, `syntax-annotation-layer` | 3 | Batch 1 | All rendering + layer APIs |
| 3 | `impl-custom-viewer`, `impl-bundler-integration`, `errors-worker` | 3 | Batch 1-2 | Viewer + bundlers + worker errors |
| 4 | `errors-rendering`, `errors-document`, `agents-review` | 3 | Batch 1-3 | Error skills + review agent |
| 5 | `agents-project-scaffolder` | 1 | ALL above | Final: scaffolder references everything |

**Total**: 13 skills across 5 batches (pdfjs-syntax-typescript dropped per D-015).

---

## Per-Skill Agent Prompts

### Constants

```
PROJECT_ROOT = C:\Users\Freek Heijting\Documents\GitHub\PDFjs-Claude-Skill-Package
RESEARCH_FILE = C:\Users\Freek Heijting\Documents\GitHub\PDFjs-Claude-Skill-Package\docs\research\vooronderzoek-pdfjs.md
REQUIREMENTS_FILE = C:\Users\Freek Heijting\Documents\GitHub\PDFjs-Claude-Skill-Package\REQUIREMENTS.md
REFERENCE_SKILL = C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md
```

---

### Batch 1

#### Prompt: pdfjs-core-architecture

```
## Task: Create the pdfjs-core-architecture skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\PDFjs-Claude-Skill-Package\skills\source\pdfjs-core\pdfjs-core-architecture\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (API signatures for PDFDocumentProxy, PDFPageProxy, GlobalWorkerOptions)
3. references/examples.md (working code examples)
4. references/anti-patterns.md (what NOT to do)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: pdfjs-core-architecture
description: "Guides PDF.js architecture including the three-layer model (Core/Display/Viewer), worker thread architecture, component hierarchy, pdfjs-dist package structure, and rendering pipeline overview. Activates when starting a PDF.js project, understanding PDF.js internals, or reasoning about the rendering pipeline."
license: MIT
compatibility: "Designed for Claude Code. Requires pdfjs-dist 5.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Three-layer architecture: Core (pdf.worker.mjs), Display (pdf.mjs), Viewer (viewer.mjs)
- Worker thread model: main thread vs worker thread communication
- Component hierarchy: PDFDocumentProxy → PDFPageProxy → render layers
- pdfjs-dist package structure: build/, cmaps/, standard_fonts/
- Layer stacking: Canvas (visual) → TextLayer (selection) → AnnotationLayer (interactive)
- Key types overview: PDFDocumentProxy, PDFPageProxy, PageViewport, RenderTask, TextLayer, AnnotationLayer
- Version info: pdfjs-dist 5.x (current: 5.5.207)

### Research
Use WebFetch on these URLs to verify content:
- https://mozilla.github.io/pdf.js/getting_started/ — architecture layers
- https://github.com/mozilla/pdf.js/blob/master/src/display/api.js — core API classes
- https://github.com/mozilla/pdf.js/wiki/Frequently-Asked-Questions — common setup issues

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language, not "you should" or "consider"
- All code examples must target pdfjs-dist 5.x
- Include Critical Warnings section with NEVER rules
- Include version annotations (pdfjs-dist 5.x)
```

#### Prompt: pdfjs-syntax-worker-setup

```
## Task: Create the pdfjs-syntax-worker-setup skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\PDFjs-Claude-Skill-Package\skills\source\pdfjs-syntax\pdfjs-syntax-worker-setup\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (GlobalWorkerOptions API, all properties)
3. references/examples.md (worker setup for CDN, webpack, vite, rollup)
4. references/anti-patterns.md (common worker setup mistakes)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: pdfjs-syntax-worker-setup
description: "Configures PDF.js Web Worker for document parsing. Covers GlobalWorkerOptions.workerSrc, CDN URLs, webpack/vite/rollup bundler configuration, fake worker mode, version matching, CMap setup, and standard font configuration. Activates when setting up PDF.js, configuring the worker, fixing worker errors, or integrating with a bundler."
license: MIT
compatibility: "Designed for Claude Code. Requires pdfjs-dist 5.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- GlobalWorkerOptions.workerSrc — setting the worker path
- CDN URLs: cdnjs, unpkg, jsdelivr patterns with version matching
- Webpack configuration: worker-loader, asset/resource module
- Vite configuration: import.meta.url pattern, vite-plugin-static-copy
- Rollup and other bundler patterns
- Fake worker mode: when and how to use workerPort / disable worker
- Version matching requirement: worker version MUST match pdfjs-dist version
- CMap configuration: cMapUrl, cMapPacked for CJK font support
- Standard font configuration: standardFontDataUrl

### Research
Use WebFetch on these URLs to verify content:
- https://mozilla.github.io/pdf.js/getting_started/ — setup instructions
- https://github.com/mozilla/pdf.js/blob/master/src/display/api.js — GlobalWorkerOptions
- https://github.com/mozilla/pdf.js/wiki/Frequently-Asked-Questions — version mismatch FAQ

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- CRITICAL: Worker version MUST match pdfjs-dist version — emphasize this
- Show decision tree: CDN vs bundler vs fake worker
- All code examples must target pdfjs-dist 5.x
```

#### Prompt: pdfjs-syntax-document-loading

```
## Task: Create the pdfjs-syntax-document-loading skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\PDFjs-Claude-Skill-Package\skills\source\pdfjs-syntax\pdfjs-syntax-document-loading\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (getDocument params, PDFDocumentProxy methods, PDFPageProxy methods)
3. references/examples.md (loading from URL, ArrayBuffer, TypedArray, base64)
4. references/anti-patterns.md (loading without worker, not destroying documents)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: pdfjs-syntax-document-loading
description: "Loads PDF documents using getDocument() with all source types. Covers PDFDocumentLoadingTask, PDFDocumentProxy, PDFPageProxy, progress tracking, cancellation, metadata extraction, outline/bookmarks, and document cleanup. Activates when loading PDFs, extracting PDF metadata, getting page count, or accessing PDF document properties."
license: MIT
compatibility: "Designed for Claude Code. Requires pdfjs-dist 5.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- getDocument(source) — DocumentInitParameters: url, data, httpHeaders, withCredentials, password, range, cMapUrl, standardFontDataUrl
- PDFDocumentLoadingTask: promise, destroy(), onProgress callback
- PDFDocumentProxy: numPages, fingerprints, getPage(num), getMetadata(), getOutline(), getData(), getDownloadInfo(), getAttachments(), getFieldObjects(), cleanup(), destroy()
- PDFPageProxy: pageNumber, rotate, userUnit, view, getViewport(), render(), getTextContent(), getAnnotations(), getOperatorList(), cleanup()
- Loading from URL, ArrayBuffer, Uint8Array, base64-encoded data
- Password-protected PDF handling
- Memory management: destroy() vs cleanup()

### Research
Use WebFetch on these URLs to verify content:
- https://github.com/mozilla/pdf.js/blob/master/src/display/api.js — all class methods
- https://mozilla.github.io/pdf.js/getting_started/ — loading examples
- https://github.com/mozilla/pdf.js/wiki/Frequently-Asked-Questions — loading FAQ

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- ALWAYS show worker setup before getDocument() in examples
- Include decision tree: URL vs ArrayBuffer vs TypedArray
- All code examples must target pdfjs-dist 5.x
```

---

### Batch 2

#### Prompt: pdfjs-syntax-page-rendering

```
## Task: Create the pdfjs-syntax-page-rendering skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\PDFjs-Claude-Skill-Package\skills\source\pdfjs-syntax\pdfjs-syntax-page-rendering\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (render params, RenderTask API, PageViewport API)
3. references/examples.md (basic render, high-DPI, re-render with cancel)
4. references/anti-patterns.md (no cancel, no DPI, render all pages)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: pdfjs-syntax-page-rendering
description: "Renders PDF pages to canvas using page.render() and manages the rendering pipeline. Covers RenderTask lifecycle, viewport creation with scale/rotation, high-DPI canvas scaling with devicePixelRatio, render cancellation patterns, and layer stacking order. Activates when rendering PDF pages, handling zoom/rotation, fixing blurry PDF rendering, or managing render tasks."
license: MIT
compatibility: "Designed for Claude Code. Requires pdfjs-dist 5.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- page.getViewport({scale, rotation, offsetX, offsetY, dontFlip}) → PageViewport
- page.render({canvasContext, viewport, transform, background, annotationMode}) → RenderTask
- RenderTask: promise, cancel(), separateAnnots
- Canvas setup: width/height from viewport, getContext('2d')
- High-DPI rendering: devicePixelRatio, canvas.width vs canvas.style.width
- Render cancellation: ALWAYS cancel previous render before starting new one
- Layer stacking: canvas (z:0) → TextLayer (z:1) → AnnotationLayer (z:2)
- OffscreenCanvas support
- Rendering pipeline flow: getDocument → getPage → getViewport → render

### Research
Use WebFetch on these URLs to verify content:
- https://github.com/mozilla/pdf.js/blob/master/src/display/api.js — render() params and RenderTask
- https://github.com/mozilla/pdf.js/wiki/Frequently-Asked-Questions — rendering performance

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- CRITICAL: ALWAYS show devicePixelRatio handling in canvas examples
- CRITICAL: ALWAYS show render task cancellation pattern
- All code examples must target pdfjs-dist 5.x
```

#### Prompt: pdfjs-syntax-text-layer

```
## Task: Create the pdfjs-syntax-text-layer skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\PDFjs-Claude-Skill-Package\skills\source\pdfjs-syntax\pdfjs-syntax-text-layer\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (TextLayer API, getTextContent params, TextContent structure)
3. references/examples.md (text layer rendering, text extraction, streaming)
4. references/anti-patterns.md (old renderTextLayer API, wrong CSS, missing container)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: pdfjs-syntax-text-layer
description: "Creates selectable and searchable text overlays using the TextLayer class. Covers text content extraction, TextLayer rendering, CSS positioning, text selection, plain text extraction, and streaming text content. Activates when adding text selection to PDF viewer, extracting text from PDF, implementing PDF search, or fixing text layer positioning."
license: MIT
compatibility: "Designed for Claude Code. Requires pdfjs-dist 5.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- TextLayer class (v5 API): constructor({textContentSource, container, viewport, images})
- TextLayer methods: render() → Promise, update({viewport, onBefore}), cancel()
- TextLayer properties: textDivs, textContentItemsStr
- TextLayer.cleanup() static method
- page.getTextContent({includeMarkedContent, disableNormalization}) → TextContent
- page.streamTextContent() → ReadableStream
- TextContent structure: items (TextItem[]), styles
- TextItem: str, dir, width, height, transform, fontName, hasEOL
- CSS requirements: position absolute, z-index above canvas, pointer-events
- Text selection CSS setup
- Plain text extraction pattern (concatenating TextItem.str values)
- NEVER use deprecated renderTextLayer() function

### Research
Use WebFetch on these URLs to verify content:
- https://github.com/mozilla/pdf.js/blob/master/src/display/text_layer.js — TextLayer class API
- https://github.com/mozilla/pdf.js/issues/18206 — TextLayer migration from renderTextLayer

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- NEVER use deprecated renderTextLayer() — ALWAYS use TextLayer class
- Show text layer positioned absolutely over canvas
- All code examples must target pdfjs-dist 5.x
```

#### Prompt: pdfjs-syntax-annotation-layer

```
## Task: Create the pdfjs-syntax-annotation-layer skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\PDFjs-Claude-Skill-Package\skills\source\pdfjs-syntax\pdfjs-syntax-annotation-layer\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (AnnotationLayer API, getAnnotations, AnnotationStorage, AnnotationEditorLayer)
3. references/examples.md (annotation rendering, link handling, form fields)
4. references/anti-patterns.md (wrong z-index, missing AnnotationStorage, deprecated APIs)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: pdfjs-syntax-annotation-layer
description: "Renders interactive PDF annotations using the AnnotationLayer class. Covers annotation types (links, text, widgets/forms, popups, highlights), link handling, form field rendering, AnnotationStorage for form data, and AnnotationEditorLayer for editing. Activates when adding annotations to PDF viewer, handling PDF links, rendering PDF forms, or extracting form data."
license: MIT
compatibility: "Designed for Claude Code. Requires pdfjs-dist 5.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- AnnotationLayer class (v5): render(), update(), cancel()
- page.getAnnotations({intent}) → AnnotationData[]
- Annotation types: Link, Text, Widget (form fields), Popup, Highlight, Underline, Squiggly, StrikeOut, Stamp, FileAttachment
- Link annotation handling: internal links (dest), external links (url)
- Widget annotation types: text input, checkbox, radio, select, button
- AnnotationStorage: for storing form field values, getAll(), setValue()
- AnnotationEditorLayer: FreeText, Ink, Stamp, Highlight editor types
- Layer stacking: MUST be on top of TextLayer (highest z-index)
- CSS requirements for annotation layer positioning

### Research
Use WebFetch on these URLs to verify content:
- https://github.com/mozilla/pdf.js/blob/master/src/display/annotation_layer.js — AnnotationLayer API
- https://github.com/mozilla/pdf.js/blob/master/src/display/api.js — getAnnotations method

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Annotation layer MUST be rendered on top of text layer
- Include decision tree for annotation type handling
- All code examples must target pdfjs-dist 5.x
```

---

### Batch 3

#### Prompt: pdfjs-impl-custom-viewer

```
## Task: Create the pdfjs-impl-custom-viewer skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\PDFjs-Claude-Skill-Package\skills\source\pdfjs-impl\pdfjs-impl-custom-viewer\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (viewer patterns, IntersectionObserver API usage)
3. references/examples.md (complete viewer with navigation, zoom, search, print, thumbnails)
4. references/anti-patterns.md (render all pages, no lazy loading, no cleanup)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: pdfjs-impl-custom-viewer
description: "Builds a complete custom PDF viewer with page navigation, zoom controls, text search, print support, and thumbnails. Covers lazy page loading with IntersectionObserver, virtual scrolling, scroll-based page detection, and memory management. Activates when building a PDF viewer, adding PDF viewing to a web app, implementing page navigation, zoom, search, print, or thumbnail generation."
license: MIT
compatibility: "Designed for Claude Code. Requires pdfjs-dist 5.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Page navigation: previous/next, go-to-page, page count display
- Zoom controls: zoom in/out, fit-to-page, fit-to-width, custom scale
- Scroll-based page loading: IntersectionObserver for lazy rendering
- Virtual scrolling: only render visible pages + buffer
- Current page detection from scroll position
- Thumbnail generation: render at reduced scale, caching
- Text search: search across all pages, highlight matches, navigate results
- Print: render at print resolution, CSS @media print, window.print()
- Memory management: cleanup rendered pages when scrolled away
- HTML structure: container → page wrappers → canvas + text + annotation layers
- NEVER render all pages at once — ALWAYS use lazy loading

### Research
Use WebFetch on these URLs to verify content:
- https://github.com/mozilla/pdf.js/tree/master/web — reference viewer implementation
- https://github.com/mozilla/pdf.js/wiki/Frequently-Asked-Questions — performance tips
- https://mozilla.github.io/pdf.js/getting_started/ — basic setup

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/ (especially the complete viewer example)
- Use ALWAYS/NEVER deterministic language
- CRITICAL: NEVER render all pages at once — show lazy loading with IntersectionObserver
- Show complete viewer architecture with all features
- All code examples must target pdfjs-dist 5.x
```

#### Prompt: pdfjs-impl-bundler-integration

```
## Task: Create the pdfjs-impl-bundler-integration skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\PDFjs-Claude-Skill-Package\skills\source\pdfjs-impl\pdfjs-impl-bundler-integration\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (bundler-specific configurations)
3. references/examples.md (complete configs for webpack, vite, rollup, next.js)
4. references/anti-patterns.md (wrong worker paths, missing static assets)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: pdfjs-impl-bundler-integration
description: "Integrates pdfjs-dist with modern JavaScript bundlers. Covers webpack worker configuration, Vite setup with import.meta.url, Rollup configuration, Next.js/Nuxt.js patterns, CMap and font file copying, and tree-shaking considerations. Activates when configuring PDF.js with webpack, vite, rollup, next.js, or any bundler, or when fixing worker loading issues in bundled applications."
license: MIT
compatibility: "Designed for Claude Code. Requires pdfjs-dist 5.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Webpack: worker as asset/resource, copy-webpack-plugin for cmaps/fonts
- Vite: import.meta.url worker pattern, vite-plugin-static-copy, optimizeDeps config
- Rollup: rollup-plugin-copy for static assets
- Next.js: dynamic import, worker path in public/, SSR considerations
- Nuxt.js: client-only component, worker in static/
- CMap file copying: source location in node_modules, destination, cMapUrl config
- Standard font copying: standardFontDataUrl config
- Tree-shaking: what can and cannot be tree-shaken
- Decision tree: which bundler config to use

### Research
Use WebFetch on these URLs to verify content:
- https://github.com/mozilla/pdf.js/wiki/Frequently-Asked-Questions — bundler guidance
- https://mozilla.github.io/pdf.js/getting_started/ — CDN vs local setup

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Include complete, copy-paste-ready configs for each bundler
- All code examples must target pdfjs-dist 5.x
```

#### Prompt: pdfjs-errors-worker

```
## Task: Create the pdfjs-errors-worker skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\PDFjs-Claude-Skill-Package\skills\source\pdfjs-errors\pdfjs-errors-worker\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (error types and GlobalWorkerOptions reference)
3. references/examples.md (error recovery patterns)
4. references/anti-patterns.md (causes of each error)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: pdfjs-errors-worker
description: "Diagnoses and fixes PDF.js Web Worker errors. Covers version mismatch errors, worker loading failures, CORS issues, CSP violations, fake worker fallback, and worker initialization problems. Activates when PDF.js worker fails to load, version mismatch error appears, or worker-related errors occur."
license: MIT
compatibility: "Designed for Claude Code. Requires pdfjs-dist 5.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- "The API version 'X' does not match the Worker version 'Y'" — cause: mismatched files, fix: ensure same version
- Worker failed to load — cause: wrong path, 404, CORS, CSP
- CORS errors when loading worker from CDN — fix: correct CDN URL or self-host
- CSP violations blocking worker — fix: add worker-src to Content-Security-Policy
- Fake worker fallback: when to use, performance implications
- Worker initialization timeout — causes and fixes
- Diagnostic decision tree: error message → likely cause → fix
- Prevention patterns: version pinning, CDN URL templates

### Research
Use WebFetch on these URLs to verify content:
- https://github.com/mozilla/pdf.js/wiki/Frequently-Asked-Questions — version mismatch FAQ
- https://github.com/mozilla/pdf.js/issues — search for common worker errors

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Structure as: Error Message → Cause → Fix → Prevention
- All code examples must target pdfjs-dist 5.x
```

---

### Batch 4

#### Prompt: pdfjs-errors-rendering

```
## Task: Create the pdfjs-errors-rendering skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\PDFjs-Claude-Skill-Package\skills\source\pdfjs-errors\pdfjs-errors-rendering\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (RenderTask error handling, canvas API)
3. references/examples.md (error recovery, proper DPI handling, cancel pattern)
4. references/anti-patterns.md (causes of rendering issues)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: pdfjs-errors-rendering
description: "Diagnoses and fixes PDF.js rendering errors. Covers blurry text from missing devicePixelRatio, render task race conditions, memory issues, concurrent render conflicts, text layer misalignment, and annotation layer z-index problems. Activates when PDF renders blurry, text is misaligned, rendering is slow, or render tasks conflict."
license: MIT
compatibility: "Designed for Claude Code. Requires pdfjs-dist 5.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Blurry rendering — cause: missing devicePixelRatio, fix: scale canvas dimensions
- Render task race conditions — cause: not cancelling previous render, fix: cancel-then-render pattern
- Memory explosion — cause: rendering all pages at once, fix: lazy loading
- Concurrent render conflicts — cause: multiple render() calls on same canvas, fix: queue/cancel
- Text layer misalignment — cause: wrong CSS, viewport mismatch, fix: sync viewport
- Annotation layer not interactive — cause: wrong z-index or pointer-events, fix: CSS stacking
- Canvas context lost — cause: too many canvases, fix: cleanup unused canvases
- White/blank pages — cause: render not awaited, canvas size 0, fix: check dimensions

### Research
Use WebFetch on these URLs to verify content:
- https://github.com/mozilla/pdf.js/wiki/Frequently-Asked-Questions — rendering FAQ
- https://github.com/mozilla/pdf.js/blob/master/src/display/api.js — RenderTask

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Structure as: Symptom → Cause → Fix → Prevention
- All code examples must target pdfjs-dist 5.x
```

#### Prompt: pdfjs-errors-document

```
## Task: Create the pdfjs-errors-document skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\PDFjs-Claude-Skill-Package\skills\source\pdfjs-errors\pdfjs-errors-document\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (exception types, error handling API)
3. references/examples.md (error handling patterns, password prompts, fallbacks)
4. references/anti-patterns.md (swallowing errors, not handling password)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: pdfjs-errors-document
description: "Diagnoses and fixes PDF document loading errors. Covers InvalidPDFException, MissingPDFException, PasswordException, CORS errors, missing CMap data for CJK fonts, font loading failures, and corrupt PDF handling. Activates when PDF fails to load, password prompt needed, CJK text missing, or network errors occur during PDF loading."
license: MIT
compatibility: "Designed for Claude Code. Requires pdfjs-dist 5.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- InvalidPDFException — corrupt or non-PDF file
- MissingPDFException — 404, wrong URL, file not found
- PasswordException — password-protected PDF, PasswordResponses enum
- UnknownErrorException — catch-all for unexpected errors
- CORS/network errors — cross-origin loading, mixed content
- Missing CMap data — CJK fonts not rendering, fix: configure cMapUrl
- Font loading failures — missing standardFontDataUrl, embedded font issues
- Memory errors — extremely large PDFs, fix: range requests, partial loading
- Error handling pattern: try/catch with specific exception types
- Password prompt workflow: catch PasswordException, prompt user, retry with password

### Research
Use WebFetch on these URLs to verify content:
- https://github.com/mozilla/pdf.js/blob/master/src/display/api.js — exception types
- https://github.com/mozilla/pdf.js/wiki/Frequently-Asked-Questions — loading issues

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Structure as: Exception Type → Cause → Fix → Prevention
- All code examples must target pdfjs-dist 5.x
```

#### Prompt: pdfjs-agents-review

```
## Task: Create the pdfjs-agents-review skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\PDFjs-Claude-Skill-Package\skills\source\pdfjs-agents\pdfjs-agents-review\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (complete checklist items with pass/fail criteria)
3. references/examples.md (example review of good and bad PDF.js code)
4. references/anti-patterns.md (all anti-patterns from all other skills consolidated)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: pdfjs-agents-review
description: "Validates generated PDF.js code for correctness and best practices. Checks worker setup, render task cancellation, devicePixelRatio handling, layer stacking, memory management, v5 API compliance, and anti-pattern detection. Activates when reviewing PDF.js code, validating a PDF viewer implementation, or checking PDF.js code quality."
license: MIT
compatibility: "Designed for Claude Code. Requires pdfjs-dist 5.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Worker setup check: GlobalWorkerOptions.workerSrc set before getDocument()
- Version matching check: worker version matches pdfjs-dist version
- Render task cancellation: previous render cancelled before new render
- DPI handling: devicePixelRatio used for canvas scaling
- Layer stacking: canvas → TextLayer → AnnotationLayer in correct z-order
- Memory management: documents destroyed, pages cleaned up, canvases removed
- Lazy loading: pages rendered on-demand, not all at once
- v5 API compliance: TextLayer class (not renderTextLayer), AnnotationLayer class
- Import correctness: correct import paths from pdfjs-dist
- Error handling: exceptions caught and handled appropriately
- Anti-pattern consolidation from all error skills

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Structure as numbered checklist with PASS/FAIL criteria
- All code examples must target pdfjs-dist 5.x
```

---

### Batch 5

#### Prompt: pdfjs-agents-project-scaffolder

```
## Task: Create the pdfjs-agents-project-scaffolder skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\PDFjs-Claude-Skill-Package\skills\source\pdfjs-agents\pdfjs-agents-project-scaffolder\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (scaffolding decision tree and options)
3. references/examples.md (complete project templates: vanilla, webpack, vite)
4. references/anti-patterns.md (scaffolding mistakes)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: pdfjs-agents-project-scaffolder
description: "Generates complete PDF.js project structures with proper worker configuration, rendering pipeline, text and annotation layers, and bundler integration. Covers vanilla JS, webpack, and Vite setups with TypeScript support. Activates when creating a new PDF.js project, scaffolding a PDF viewer, or setting up PDF.js from scratch."
license: MIT
compatibility: "Designed for Claude Code. Requires pdfjs-dist 5.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Decision tree: vanilla CDN vs webpack vs vite vs next.js
- Package.json with pdfjs-dist 5.x dependency
- Worker configuration for chosen bundler
- HTML template with page container, canvas, text layer, annotation layer
- CSS template with layer stacking, responsive sizing
- JavaScript/TypeScript entry point with:
  - Worker setup
  - Document loading
  - Page rendering with DPI handling
  - Text layer setup
  - Annotation layer setup
  - Basic navigation (prev/next/goto)
  - Zoom controls
- tsconfig.json for TypeScript projects
- Bundler config file (webpack.config.js / vite.config.ts)
- Complete file tree output

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/ (especially complete project templates)
- Use ALWAYS/NEVER deterministic language
- Templates must be complete and copy-paste-ready
- All code examples must target pdfjs-dist 5.x
```

---

## Appendix: Skill Directory Structure

```
skills/source/
├── pdfjs-core/
│   └── pdfjs-core-architecture/
│       ├── SKILL.md
│       └── references/
│           ├── methods.md
│           ├── examples.md
│           └── anti-patterns.md
├── pdfjs-syntax/
│   ├── pdfjs-syntax-worker-setup/
│   ├── pdfjs-syntax-document-loading/
│   ├── pdfjs-syntax-page-rendering/
│   ├── pdfjs-syntax-text-layer/
│   └── pdfjs-syntax-annotation-layer/
├── pdfjs-impl/
│   ├── pdfjs-impl-custom-viewer/
│   └── pdfjs-impl-bundler-integration/
├── pdfjs-errors/
│   ├── pdfjs-errors-worker/
│   ├── pdfjs-errors-rendering/
│   └── pdfjs-errors-document/
└── pdfjs-agents/
    ├── pdfjs-agents-review/
    └── pdfjs-agents-project-scaffolder/
```
