# Consolidated Anti-Patterns (All PDF.js Skills)

Complete list of anti-patterns collected from all PDF.js skills in this package. Organized by category for use during code review.

---

## Worker Setup Anti-Patterns

### W-01: Calling getDocument() Before Setting workerSrc

**Severity**: Critical

PDF.js silently falls back to fake worker mode (main thread parsing), blocking the UI. No error is thrown.

**Fix**: ALWAYS set `GlobalWorkerOptions.workerSrc` at module initialization, before any `getDocument()` call.

---

### W-02: Version Mismatch Between API and Worker

**Severity**: Critical

Causes `"API version does not match Worker version"`. Not recoverable -- document cannot be loaded.

**Fix**: ALWAYS interpolate version from package: `` `.../${version}/pdf.worker.min.mjs` `` or let the bundler resolve from the same `node_modules`.

---

### W-03: Hardcoded CDN Version

**Severity**: Critical

Version drifts from installed npm package on update, causing version mismatch errors.

**Fix**: Use `import { version } from "pdfjs-dist"` and interpolate into CDN URL.

---

### W-04: Using @latest or Semver Range in CDN URL

**Severity**: High

CDN resolves independently from your app. Works during development, breaks in production when CDN updates.

**Fix**: ALWAYS pin to exact version using the `version` export.

---

### W-05: Relative Worker Path Without Understanding Base URL

**Severity**: Medium

Worker URL resolves relative to the page URL (not the script), breaks with client-side routing.

**Fix**: Use absolute paths (`/pdf.worker.min.mjs`) or `import.meta.url` resolution.

---

### W-06: Using .js Extension Instead of .mjs

**Severity**: High

pdfjs-dist 5.x ships only `.mjs` files. Using `.js` causes 404 errors.

**Fix**: ALWAYS use `.mjs` extension for pdfjs-dist 5.x.

---

### W-07: Loading Worker Over HTTP on HTTPS Page

**Severity**: Medium

Browser blocks mixed content silently. Worker fails, falls back to fake worker.

**Fix**: ALWAYS use HTTPS for CDN URLs.

---

### W-08: Using Fake Worker in Production

**Severity**: High

Importing `pdfjs-dist/build/pdf.worker.min.mjs` directly runs parsing on the main thread. Blocks UI.

**Fix**: ONLY use fake worker for testing or SSR. Use `workerSrc` in production.

---

### W-09: Setting workerSrc Multiple Times

**Severity**: Low

Redundant and risks race conditions if different values are set.

**Fix**: Set `workerSrc` ONCE in a central setup module.

---

### W-10: Creating a New PDFWorker Per Document

**Severity**: Medium

Creates and leaks Web Worker threads.

**Fix**: Let PDF.js manage workers via `GlobalWorkerOptions.workerSrc`.

---

### W-11: Reusing a Worker Port for Multiple Documents

**Severity**: Medium

Causes `"Cannot use more than one PDFWorker per port"`.

**Fix**: Use `workerSrc` (not `workerPort`) when loading multiple documents.

---

### W-12: Mixing Module Formats (.mjs API with .js Worker)

**Severity**: High

Worker handshake fails silently, documents never load.

**Fix**: ALWAYS match module format -- `.mjs` API needs `.mjs` worker.

---

## Rendering Anti-Patterns

### R-01: Not Cancelling Previous Render Task

**Severity**: Critical

Two renders fight over the same canvas, producing visual artifacts and memory leaks.

**Fix**: ALWAYS cancel previous `RenderTask` before starting a new render on the same canvas.

---

### R-02: Not Handling devicePixelRatio (Blurry Rendering)

**Severity**: High

Canvas renders at 1x resolution on Retina/4K displays. Browser upscales, causing blur.

**Fix**: Scale canvas pixel dimensions by `devicePixelRatio`, set CSS dimensions separately, and call `ctx.scale(dpr, dpr)`.

---

### R-03: Rendering All Pages at Once

**Severity**: Critical

Each canvas at scale 1.5 on 2x DPR uses ~36 MB. 50 pages = 1.8 GB. Crashes the tab.

**Fix**: ALWAYS use `IntersectionObserver` for lazy page rendering.

---

### R-04: Not Awaiting renderTask.promise

**Severity**: High

Canvas is incomplete or empty when accessed. Capturing canvas content returns blank image.

**Fix**: ALWAYS `await renderTask.promise` before accessing canvas content.

---

### R-05: Wrong Canvas Dimension Setup

**Severity**: Medium

Missing CSS dimensions causes oversized display. Floating-point values cause sub-pixel artifacts.

**Fix**: Use `Math.floor()` for integer dimensions. Set BOTH pixel and CSS dimensions.

---

### R-06: Concurrent Renders on Same Canvas

**Severity**: Critical

Multiple `page.render()` calls on the same context produce garbled output.

**Fix**: Serialize renders with cancellation. NEVER have more than one active render per canvas.

---

### R-07: Not Cleaning Up RenderTask References

**Severity**: Medium

Completed `RenderTask` holds references to canvas context and rendering data. Memory leak.

**Fix**: ALWAYS set `RenderTask` reference to `null` in a `finally` block.

---

### R-08: Fire-and-Forget from Event Listeners

**Severity**: High

Scroll events at 60fps create 60 concurrent renders per second.

**Fix**: Debounce event handlers (50ms minimum) and cancel previous renders.

---

### R-09: Print Without intent: "print"

**Severity**: Low

Print-only annotations will not appear.

**Fix**: ALWAYS use `intent: "print"` when rendering for print output.

---

## Text Layer Anti-Patterns

### T-01: Using Deprecated renderTextLayer() Function

**Severity**: High

Removed in pdfjs-dist 5.x. Will throw `renderTextLayer is not a function`.

**Fix**: ALWAYS use `new TextLayer({ textContentSource, container, viewport })`.

---

### T-02: Missing Absolute Positioning on Text Layer Container

**Severity**: High

Text spans appear below the canvas instead of overlapping it.

**Fix**: Set `position: absolute; top: 0; left: 0` on text layer div. Parent MUST have `position: relative`.

---

### T-03: Missing Text Layer CSS (Visible Black Text)

**Severity**: High

Text renders as visible black text on top of the canvas, creating double-rendered text.

**Fix**: Import `pdfjs-dist/web/pdf_viewer.css` or set `color: transparent` on `.textLayer span`.

---

### T-04: Viewport Mismatch Between Canvas and Text Layer

**Severity**: High

Text selection areas are shifted or scaled incorrectly relative to the visible text.

**Fix**: ALWAYS use the SAME viewport object for both canvas render and TextLayer.

---

### T-05: Not Cancelling Text Layer Before Re-creating

**Severity**: Medium

Previous `render()` still processing causes duplicate spans and memory leaks.

**Fix**: Call `textLayer.cancel()` before creating a new TextLayer. Prefer `textLayer.update()` for viewport changes.

---

### T-06: Accessing TextItem.str Without Type Guard

**Severity**: Medium

Crashes when item is `TextMarkedContent` (has no `str` property).

**Fix**: ALWAYS check `if ("str" in item)` when `includeMarkedContent` is true.

---

### T-07: Missing Parent Container Relative Positioning

**Severity**: High

Absolutely-positioned layers find `<body>` as ancestor, rendering at wrong location.

**Fix**: Parent container MUST have `position: relative`.

---

### T-08: Calling TextLayer.cleanup() While Instances Are Active

**Severity**: Medium

Destroys shared caches that active instances need.

**Fix**: ONLY call `TextLayer.cleanup()` after ALL text layer instances are destroyed.

---

## Annotation Layer Anti-Patterns

### A-01: Wrong Z-Index (Annotations Behind Text Layer)

**Severity**: High

Links and form fields visible but not clickable.

**Fix**: ALWAYS stack: canvas (0) < textLayer (1) < annotationLayer (2) < annotationEditorLayer (3).

---

### A-02: Missing AnnotationStorage for Forms

**Severity**: High

Form input is lost when scrolling away and back, or when printing.

**Fix**: ALWAYS pass a shared `AnnotationStorage` instance when `renderForms: true`.

---

### A-03: Missing CSS Import for Annotations

**Severity**: Medium

Annotations appear at wrong sizes and positions.

**Fix**: ALWAYS import `pdfjs-dist/web/pdf_viewer.css`.

---

### A-04: Not Using position: absolute on Annotation Layer

**Severity**: High

Annotation layer renders below the canvas instead of overlapping it.

**Fix**: Use `position: absolute` on annotation div. Parent MUST have `position: relative`.

---

### A-05: Wrong Intent for getAnnotations()

**Severity**: Medium

Print-only annotations appear on screen, or display-only annotations appear in print.

**Fix**: Use `intent: "display"` for screen, `intent: "print"` for print. NEVER mix.

---

### A-06: Creating Multiple AnnotationStorage Instances

**Severity**: Medium

Form data entered on one page is not available on another page.

**Fix**: ALWAYS create ONE `AnnotationStorage` instance per document, shared across all pages.

---

### A-07: Forgetting renderForms: true

**Severity**: Medium

Form fields appear as static images. Users cannot interact.

**Fix**: ALWAYS set `renderForms: true` for interactive form fields.

---

### A-08: Not Cancelling Before Removing Annotation Layer

**Severity**: Medium

Memory leaks and orphaned event listeners.

**Fix**: ALWAYS call `annotationLayer.cancel()` before removing the div from the DOM.

---

### A-09: Re-rendering Instead of Updating on Viewport Change

**Severity**: Medium

Slow zoom/rotation, form field input lost.

**Fix**: Use `annotationLayer.update({ viewport })` instead of destroy + rebuild.

---

### A-10: Not Validating External Link URLs

**Severity**: High (Security)

Malicious PDFs can execute JavaScript via `javascript:` URLs.

**Fix**: ONLY allow `http:` and `https:` protocols. NEVER allow `javascript:` or `data:` URLs.

---

### A-11: Using AnnotationMode.ENABLE Instead of ENABLE_FORMS

**Severity**: Medium

Form fields render as static, non-interactive images.

**Fix**: Use `AnnotationMode.ENABLE_FORMS` (2) or `ENABLE_STORAGE` (3) for interactive forms.

---

## Document Loading Anti-Patterns

### D-01: Not Destroying Documents (Memory Leak)

**Severity**: Critical

Worker thread and caches accumulate with each loaded document.

**Fix**: ALWAYS call `doc.destroy()` when switching documents or unmounting.

---

### D-02: Not Handling Password-Protected PDFs

**Severity**: Medium

`PasswordException` crashes the loading flow with no user feedback.

**Fix**: Set `loadingTask.onPassword` callback or catch `PasswordException`.

---

### D-03: Using 0-Based Page Numbers

**Severity**: Medium

`getPage(0)` throws an error. PDF.js uses 1-based page numbers.

**Fix**: Use page numbers from 1 to `doc.numPages`.

---

### D-04: Passing Raw String Data to getDocument

**Severity**: High

`.text()` applies UTF-8 decoding, corrupting binary PDF data.

**Fix**: Use `response.arrayBuffer()` to get binary data, pass as `{ data: arrayBuffer }`.

---

### D-05: Not Cancelling Previous Loads

**Severity**: Medium

Multiple concurrent loads cause flickering, wasted bandwidth, and memory pressure.

**Fix**: Track `PDFDocumentLoadingTask` and call `destroy()` on it before starting a new load.

---

### D-06: Using Document/Page After destroy()

**Severity**: Medium

All methods reject with errors after `destroy()`.

**Fix**: ALWAYS finish all operations BEFORE calling `destroy()`.

---

### D-07: Missing CMap Configuration for CJK

**Severity**: High

CJK characters render as blank rectangles or tofu.

**Fix**: ALWAYS set `cMapUrl` and `cMapPacked: true` when loading CJK documents.

---

### D-08: Forgetting cMapPacked: true

**Severity**: Medium

pdfjs-dist ships binary CMaps. Without `cMapPacked: true`, parsing fails.

**Fix**: ALWAYS pair `cMapUrl` with `cMapPacked: true`.

---

### D-09: Swallowing Errors Without Rethrowing

**Severity**: Medium

Caller receives `null` with no way to distinguish error types.

**Fix**: Catch specific exceptions, provide typed errors, ALWAYS rethrow unknown errors.

---

## Memory Management Anti-Patterns

### M-01: No Memory Cleanup When Pages Scroll Out of View

**Severity**: Critical

Memory grows linearly as user scrolls. Eventually crashes.

**Fix**: Clean up pages when they leave the viewport. Zero canvas dimensions before removal.

---

### M-02: Using innerHTML to Clear Page Slots

**Severity**: Medium

Removes DOM elements but does NOT release GPU memory for canvases.

**Fix**: Set `canvas.width = 0; canvas.height = 0` BEFORE removing from DOM.

---

### M-03: Not Calling page.cleanup()

**Severity**: Medium

Page rendering caches accumulate in memory.

**Fix**: Call `page.cleanup()` for pages that are no longer needed.

---

### M-04: Extracting Text Without Caching

**Severity**: Medium

Repeated searches re-parse every page, causing multi-second delays.

**Fix**: Cache `getTextContent()` results per page. Text content does not change during a session.

---

## Bundler Anti-Patterns

### B-01: Hardcoded Worker Path in Bundled App

**Severity**: Critical

Bundlers rename and hash output files. Hardcoded path fails in production.

**Fix**: Use `import.meta.url` resolution or `pdfjs-dist/webpack.mjs`.

---

### B-02: Importing Worker into Main Bundle

**Severity**: Critical

Worker code runs on main thread, blocking UI during PDF parsing.

**Fix**: Worker MUST be loaded as a separate file in a Worker thread.

---

### B-03: Missing optimizeDeps.exclude in Vite

**Severity**: High

Vite pre-bundles pdfjs-dist, breaking `import.meta.url` resolution for the worker.

**Fix**: Add `optimizeDeps: { exclude: ["pdfjs-dist"] }` to vite.config.ts.

---

### B-04: Not Copying CMap Files to Public Directory

**Severity**: High

CJK PDFs render with missing characters.

**Fix**: Copy `node_modules/pdfjs-dist/cmaps/` to your public/static directory.

---

### B-05: Importing pdfjs-dist at Top Level in SSR Frameworks

**Severity**: Critical

Server-side error: `Worker is not defined`, `document is not defined`.

**Fix**: Use dynamic `import()` inside client-only code paths (useEffect, onMounted).

---

### B-06: Using Webpack 4 with webpack.mjs

**Severity**: Critical

Webpack 4 does not support `import.meta.url`. Build fails.

**Fix**: Create explicit worker entry point in Webpack 4 config.

---

## Architecture Anti-Patterns

### X-01: Using the Viewer Layer as a Library

**Severity**: High

`PDFViewer` and `EventBus` from `pdfjs-dist/web/pdf_viewer.mjs` are not a stable API. Internals change between versions.

**Fix**: Build custom viewers using the Display layer API (`getDocument`, `TextLayer`, `AnnotationLayer`).

---

### X-02: Not Debouncing Scroll Events

**Severity**: Medium

Layout thrashing from `getBoundingClientRect()` calls on every scroll event (60x/sec).

**Fix**: Debounce scroll handlers at 50ms minimum. Use `IntersectionObserver` instead of manual scroll detection.

---

### X-03: Not Handling Window Resize

**Severity**: Medium

Fit-to-width breaks when user resizes browser window.

**Fix**: Listen for `resize` event (debounced at 200ms) and recalculate scale.

---

### X-04: Calling getPage() for All Pages During Initialization

**Severity**: High

Blocks UI for seconds on large documents.

**Fix**: Fetch only the first page for dimensions. Fetch remaining pages lazily on demand.
