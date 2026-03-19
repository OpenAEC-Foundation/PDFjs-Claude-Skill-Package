# Review Checklist: Detailed PASS/FAIL Criteria

Complete reference for each review check with exact patterns to look for, code signatures to verify, and edge cases to consider.

---

## Check 1: Worker Setup Before getDocument()

### PASS Criteria

- `GlobalWorkerOptions.workerSrc` is assigned at module initialization or application startup
- Assignment occurs in a central setup file imported before any PDF loading code
- For bundlers: `import * as pdfjsLib from "pdfjs-dist/webpack.mjs"` handles worker automatically

### FAIL Criteria

- `getDocument()` appears in the same function/scope as `workerSrc` assignment, with `workerSrc` AFTER `getDocument()`
- `workerSrc` is set inside a component's render function or event handler (may run after first load)
- `workerSrc` is set conditionally and the condition may not be met before first use
- Worker code is imported directly: `import 'pdfjs-dist/build/pdf.worker.min.mjs'` in production (runs on main thread)

### Code Signatures to Check

```typescript
// CORRECT: Setup at module level
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

// CORRECT: Auto-configured via webpack entry
import * as pdfjsLib from "pdfjs-dist/webpack.mjs";

// WRONG: Setup AFTER getDocument
const doc = await getDocument(url).promise;
GlobalWorkerOptions.workerSrc = "..."; // Too late
```

### Edge Cases

- Setting `workerSrc` in multiple locations (redundant, risk of race conditions)
- Using `GlobalWorkerOptions.workerPort` with a pre-created Worker (valid but NEVER reuse for multiple documents)
- SSR frameworks: worker setup MUST be inside a client-only code path (useEffect, onMounted, etc.)

---

## Check 2: Worker Version Matches pdfjs-dist

### PASS Criteria

- CDN URL contains the exact same version as the installed `pdfjs-dist` npm package
- Version is interpolated from `pdfjs-dist` package at runtime: `` `.../${version}/...` ``
- Worker file is resolved by the bundler from the same `node_modules/pdfjs-dist` package
- File extension is `.mjs` (not `.js`)

### FAIL Criteria

- Hardcoded version string in CDN URL (will drift on npm update)
- `@latest` or semver range in CDN URL (`@^5.0.0`, `@~5.0.0`)
- Worker from a different major version than the API (v4 worker with v5 API)
- `.js` extension used (pdfjs-dist 5.x ships `.mjs` only)

### Version Verification Pattern

```typescript
import { version } from "pdfjs-dist";
// version is "5.5.207" (or whatever is installed)

// BEST: Interpolate at runtime
GlobalWorkerOptions.workerSrc =
  `https://cdnjs.cloudflare.com/ajax/libs/pdf.js/${version}/pdf.worker.min.mjs`;

// ALSO CORRECT: Let bundler resolve from same package
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();
```

---

## Check 3: Render Task Cancellation

### PASS Criteria

- A `RenderTask` reference variable is maintained per canvas
- Before calling `page.render()`, the previous `RenderTask` is cancelled via `.cancel()`
- After cancellation, the reference is set to `null`
- The `catch` block distinguishes cancellation (`"Rendering cancelled"`) from real errors
- `finally` block clears the `RenderTask` reference

### FAIL Criteria

- `page.render()` is called without checking for or cancelling a previous render
- `RenderTask` is not stored (fire-and-forget)
- Scroll or zoom event handlers call `page.render()` without debouncing
- Cancellation rejection is not caught (appears as unhandled promise rejection)
- `RenderTask` reference is never cleared after completion (memory leak)

### Required Pattern

```typescript
let currentRenderTask: RenderTask | null = null;

// Before every render:
if (currentRenderTask) {
  currentRenderTask.cancel();
  currentRenderTask = null;
}

currentRenderTask = page.render({ canvasContext, viewport });

try {
  await currentRenderTask.promise;
} catch (err: unknown) {
  if (err instanceof Error && err.message === "Rendering cancelled") {
    return; // Expected, not an error
  }
  throw err;
} finally {
  currentRenderTask = null;
}
```

---

## Check 4: devicePixelRatio Handling

### PASS Criteria

All three steps are present:

1. Canvas pixel dimensions: `canvas.width = Math.floor(viewport.width * dpr)`
2. Canvas CSS dimensions: `canvas.style.width = \`${Math.floor(viewport.width)}px\``
3. Context scaling: `ctx.scale(dpr, dpr)`

The `dpr` value defaults to 1: `const dpr = window.devicePixelRatio || 1`

### FAIL Criteria

- `canvas.width = viewport.width` without DPI multiplier
- CSS dimensions (`style.width`, `style.height`) not set
- `ctx.scale()` not called after setting canvas dimensions
- Floating point values used for canvas dimensions (MUST use `Math.floor()`)
- DPI scaling applied but CSS dimensions also include DPI (double-scaling)

### Common Mistake

```typescript
// WRONG: CSS dimensions include DPR (displays 2x too large)
canvas.width = Math.floor(viewport.width * dpr);
canvas.style.width = `${Math.floor(viewport.width * dpr)}px`; // Should NOT include dpr
```

---

## Check 5: Layer Stacking Order

### PASS Criteria

- Canvas has z-index 0 (or no z-index, as it is the base layer)
- TextLayer container has z-index 1
- AnnotationLayer container has z-index 2
- AnnotationEditorLayer (if present) has z-index 3
- All layers use `position: absolute` with `top: 0; left: 0`
- Parent container uses `position: relative`
- All layers use the SAME viewport object
- `pdfjs-dist/web/pdf_viewer.css` is imported

### FAIL Criteria

- AnnotationLayer z-index <= TextLayer z-index (links/forms not clickable)
- Any layer missing `position: absolute`
- Parent missing `position: relative`
- Viewport mismatch between canvas and text/annotation layers
- TextLayer CSS not imported (visible black text instead of transparent overlay)
- Text layer spans missing `color: transparent`

---

## Check 6: Memory Management

### PASS Criteria

- `PDFDocumentProxy.destroy()` is called when switching documents or unmounting
- `PDFDocumentLoadingTask.destroy()` is called to cancel in-progress loads
- Canvas dimensions are zeroed before removal: `canvas.width = 0; canvas.height = 0`
- `RenderTask` references are cleared in `finally` blocks
- `textLayer.cancel()` is called before removing text layer
- `annotationLayer.cancel()` is called before removing annotation layer
- `TextLayer.cleanup()` is called ONLY after ALL text layer instances are destroyed
- `page.cleanup()` is called for pages no longer needed

### FAIL Criteria

- `doc.destroy()` never called (worker thread and caches accumulate)
- Canvases removed via `innerHTML = ""` without zeroing dimensions (GPU memory leak)
- `RenderTask` held after completion without `null` assignment
- Loading tasks not cancelled before starting new ones (parallel loads)
- Component unmount does not check for in-progress operations

---

## Check 7: Lazy Loading

### PASS Criteria

- `IntersectionObserver` (or equivalent scroll-based mechanism) triggers page rendering
- Only visible pages (plus a buffer zone) are rendered
- Pages that scroll out of view are cleaned up (canvas zeroed and removed)
- `rootMargin` is set on the observer (e.g., `"200px"`) to preload nearby pages
- Page slots/placeholders are created with correct dimensions but no canvas

### FAIL Criteria

- `for (let i = 1; i <= doc.numPages; i++)` loop renders all pages
- All `PDFPageProxy` objects fetched upfront via `getPage()` in a loop
- IntersectionObserver only renders but never cleans up out-of-view pages
- No `rootMargin` buffer (pages appear blank momentarily during scroll)

---

## Check 8: v5 API Compliance

### PASS Criteria

- `TextLayer` is used as a class: `new TextLayer({ textContentSource, container, viewport })`
- `TextLayer.render()` is awaited
- `TextLayer.update()` is used for viewport changes (not destroy + recreate)
- `AnnotationLayer` is used as a class
- `.mjs` file extensions used throughout
- ESM imports used (`import { ... } from "pdfjs-dist"`)

### FAIL Criteria

- `renderTextLayer()` function used (removed in v5)
- `textContent` parameter used instead of `textContentSource` in TextLayer constructor
- `textDivs` array passed as parameter (old API)
- `.js` file extensions in import paths or workerSrc
- UMD/IIFE script tags used instead of ESM

---

## Check 9: Import Correctness

### Valid Imports (pdfjs-dist 5.x)

```typescript
// Runtime imports
import { getDocument, GlobalWorkerOptions, version } from "pdfjs-dist";
import { TextLayer } from "pdfjs-dist";
import { AnnotationLayer, AnnotationStorage } from "pdfjs-dist";
import { AnnotationEditorLayer, AnnotationEditorType } from "pdfjs-dist";
import { AnnotationMode, PasswordResponses } from "pdfjs-dist";

// Type imports
import type { PDFDocumentProxy, PDFDocumentLoadingTask } from "pdfjs-dist";
import type { PDFPageProxy, PageViewport } from "pdfjs-dist";
import type { RenderTask, TextContent, TextItem } from "pdfjs-dist";

// Worker (Vite)
import workerUrl from "pdfjs-dist/build/pdf.worker.min.mjs?url";

// Auto-config (Webpack 5)
import * as pdfjsLib from "pdfjs-dist/webpack.mjs";
```

### Invalid Imports

```typescript
// WRONG: Removed in v5
import { renderTextLayer } from "pdfjs-dist";

// WRONG: Not a stable API
import { PDFViewer, EventBus } from "pdfjs-dist/web/pdf_viewer.mjs";

// WRONG: Does not exist in v5
import { getDocument } from "pdfjs-dist/es5/build/pdf.js";

// WRONG: .js extension
import { getDocument } from "pdfjs-dist/build/pdf.js";
```

---

## Check 10: Error Handling

### PASS Criteria

- `getDocument().promise` is wrapped in try/catch
- `PasswordException` is detected and handled (prompt user or throw typed error)
- `InvalidPDFException` and `MissingPDFException` are detected
- Version mismatch errors are detected (contains `"does not match the Worker version"`)
- Render cancellation is distinguished from real render errors
- Errors are NOT silently swallowed (ALWAYS rethrow unknown errors)
- Loading task destruction errors are handled during component unmount

### FAIL Criteria

- No try/catch around PDF.js async operations
- Errors caught and logged but not re-thrown or surfaced to the user
- All errors handled with a generic message (no distinction between error types)
- `renderTask.promise` rejection not caught (unhandled promise rejection in production)
- `catch (e) { return null }` pattern (swallows all errors)

### Error Detection Pattern

```typescript
try {
  const doc = await getDocument(url).promise;
} catch (error) {
  if (error instanceof Error) {
    if (error.name === "PasswordException") { /* prompt password */ }
    if (error.name === "InvalidPDFException") { /* not a valid PDF */ }
    if (error.name === "MissingPDFException") { /* file not found */ }
    if (error.message.includes("does not match the Worker version")) { /* version mismatch */ }
  }
  throw error; // ALWAYS rethrow unknown errors
}
```
