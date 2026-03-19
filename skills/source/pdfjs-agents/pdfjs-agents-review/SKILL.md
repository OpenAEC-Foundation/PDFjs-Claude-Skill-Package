---
name: pdfjs-agents-review
description: >
  Use when reviewing or validating generated PDF.js code for correctness, best
  practices, and common mistakes. Prevents shipping code with missing worker setup,
  uncancelled render tasks, incorrect DPI handling, or wrong layer stacking order.
  Covers worker setup verification, render task lifecycle, memory management,
  v5 API compliance, and anti-pattern detection checklist.
  Keywords: code review, validation, checklist, anti-pattern, PDF.js review, quality.
license: MIT
compatibility: "Designed for Claude Code. Requires pdfjs-dist 5.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# pdfjs-agents-review

## Quick Reference

### Review Checklist Summary

| # | Check | Severity | Category |
|---|-------|----------|----------|
| 1 | Worker setup before getDocument() | Critical | Worker |
| 2 | Worker version matches pdfjs-dist | Critical | Worker |
| 3 | Previous render cancelled before new render | Critical | Rendering |
| 4 | devicePixelRatio applied to canvas | High | Rendering |
| 5 | Layer stacking order correct | High | Layers |
| 6 | Documents destroyed on cleanup | Critical | Memory |
| 7 | Pages loaded lazily (not all at once) | Critical | Memory |
| 8 | v5 API used (TextLayer class, not renderTextLayer) | High | API |
| 9 | Import paths correct for pdfjs-dist 5.x | High | Imports |
| 10 | Errors caught and handled | High | Errors |

### Severity Levels

| Level | Meaning | Action |
|-------|---------|--------|
| Critical | Causes crashes, data corruption, or total failure | MUST fix before shipping |
| High | Causes visible bugs or silent performance degradation | MUST fix before review approval |
| Medium | Causes minor issues or technical debt | SHOULD fix |
| Low | Style or optimization suggestion | MAY fix |

---

## Review Checklist

### Check 1: Worker Setup Before getDocument()

**PASS** if `GlobalWorkerOptions.workerSrc` is set BEFORE any `getDocument()` call.

**FAIL** if:
- `getDocument()` is called without prior `workerSrc` configuration
- `workerSrc` is set AFTER the first `getDocument()` call
- Worker is imported directly into the main bundle (`import 'pdfjs-dist/build/pdf.worker.mjs'`) in production code

**Look for**:
```typescript
// MUST appear before any getDocument() call
GlobalWorkerOptions.workerSrc = ...
```

---

### Check 2: Worker Version Matches pdfjs-dist

**PASS** if the worker URL uses the same version as the installed pdfjs-dist package.

**FAIL** if:
- CDN URL has a hardcoded version that may drift from npm package
- CDN URL uses `@latest` or a semver range (`@^5.0.0`)
- Worker file uses `.js` extension instead of `.mjs` (pdfjs-dist 5.x ships `.mjs` only)

**Best practice**: Interpolate version from the package:
```typescript
import { GlobalWorkerOptions, version } from "pdfjs-dist";
GlobalWorkerOptions.workerSrc =
  `https://cdnjs.cloudflare.com/ajax/libs/pdf.js/${version}/pdf.worker.min.mjs`;
```

---

### Check 3: Render Task Cancellation

**PASS** if every code path that calls `page.render()` cancels any previous `RenderTask` on the same canvas first.

**FAIL** if:
- A new `page.render()` starts without cancelling the previous one
- `RenderTask` reference is not stored for later cancellation
- Scroll/zoom event handlers fire `page.render()` without debouncing or cancellation
- Cancellation error (`"Rendering cancelled"`) is not caught in the try/catch

**Pattern to verify**:
```typescript
if (currentRenderTask) {
  currentRenderTask.cancel();
  currentRenderTask = null;
}
// ... then start new render
```

---

### Check 4: devicePixelRatio Handling

**PASS** if canvas pixel dimensions are scaled by `devicePixelRatio` AND CSS dimensions are set separately.

**FAIL** if:
- `canvas.width` / `canvas.height` are set directly from `viewport.width` / `viewport.height` without DPI multiplier
- CSS `style.width` / `style.height` are not set on the canvas
- `ctx.scale(dpr, dpr)` is missing after setting canvas dimensions
- `Math.floor()` is not used (canvas dimensions MUST be integers)

**Three-step pattern**:
```typescript
const dpr = window.devicePixelRatio || 1;
canvas.width = Math.floor(viewport.width * dpr);    // 1. Pixel dimensions
canvas.style.width = `${Math.floor(viewport.width)}px`; // 2. CSS dimensions
ctx.scale(dpr, dpr);                                 // 3. Context scale
```

---

### Check 5: Layer Stacking Order

**PASS** if layers are stacked: canvas (z-index 0) < TextLayer (1) < AnnotationLayer (2).

**FAIL** if:
- AnnotationLayer has a lower z-index than TextLayer (links/forms not clickable)
- Text layer or annotation layer lacks `position: absolute`
- Parent container lacks `position: relative`
- Layers have mismatched dimensions (not using the same viewport)
- PDF.js viewer CSS (`pdfjs-dist/web/pdf_viewer.css`) is not imported

---

### Check 6: Memory Management

**PASS** if documents are destroyed, pages are cleaned up, and canvases are properly released.

**FAIL** if:
- `PDFDocumentProxy.destroy()` is never called when switching documents
- Previous loading task is not cancelled before starting a new one
- Canvas dimensions are not zeroed (`canvas.width = 0; canvas.height = 0`) before removal
- `RenderTask` references are not cleared after completion (set to `null` in `finally` block)
- `textLayer.cancel()` or `annotationLayer.cancel()` is not called before removal
- `TextLayer.cleanup()` is called while active instances still exist

---

### Check 7: Lazy Loading

**PASS** if pages are rendered on-demand (e.g., via `IntersectionObserver`).

**FAIL** if:
- All pages are rendered in a loop on document load
- All `PDFPageProxy` objects are fetched upfront with `getPage()` in a loop
- Pages that scroll out of view are not cleaned up
- No `rootMargin` buffer is used on the IntersectionObserver (causes visible pop-in)

---

### Check 8: v5 API Compliance

**PASS** if code uses pdfjs-dist 5.x APIs exclusively.

**FAIL** if:
- `renderTextLayer()` function is used (removed in v5 -- use `TextLayer` class)
- `.js` file extensions are used instead of `.mjs`
- Legacy UMD imports are used instead of ESM
- `textDivs` array parameter is passed (old API)
- `textContentSource` is missing from `TextLayer` constructor (use instead of `textContent`)

---

### Check 9: Import Correctness

**PASS** if imports use correct paths for pdfjs-dist 5.x.

**FAIL** if:
- Importing from wrong paths (e.g., `pdfjs-dist/es5/` which does not exist in v5)
- Using named imports that do not exist (`renderTextLayer`, `AnnotationLayerBuilder`)
- Importing `PDFViewer` or `EventBus` from `pdfjs-dist/web/pdf_viewer.mjs` (not a stable API)
- Missing type imports for TypeScript (`PDFPageProxy`, `RenderTask`, `PageViewport`)

**Valid imports for pdfjs-dist 5.x**:
```typescript
import { getDocument, GlobalWorkerOptions, TextLayer, AnnotationLayer, version } from "pdfjs-dist";
import type { PDFDocumentProxy, PDFPageProxy, RenderTask, PageViewport } from "pdfjs-dist";
```

---

### Check 10: Error Handling

**PASS** if PDF.js exceptions are caught and handled with actionable messages.

**FAIL** if:
- `getDocument().promise` has no error handling
- Password-protected PDFs are not handled (`onPassword` callback or `PasswordException` catch)
- Errors are silently swallowed (`catch (e) { console.log(e) }`)
- Version mismatch errors are not detected or explained to users
- `renderTask.promise` rejection from cancellation is not distinguished from real errors

---

## Decision Tree: Review Workflow

```
Reviewing PDF.js code?
|
+-- Step 1: Check IMPORTS
|   +-- Are all imports valid for pdfjs-dist 5.x? (Check 9)
|   +-- Is TextLayer imported as a class? (Check 8)
|   +-- Are type imports present for TypeScript? (Check 9)
|
+-- Step 2: Check WORKER SETUP
|   +-- Is workerSrc set before getDocument()? (Check 1)
|   +-- Does worker version match package version? (Check 2)
|
+-- Step 3: Check RENDERING
|   +-- Is devicePixelRatio handled? (Check 4)
|   +-- Is previous render cancelled before new render? (Check 3)
|   +-- Are layers stacked correctly? (Check 5)
|
+-- Step 4: Check MEMORY
|   +-- Are documents destroyed? (Check 6)
|   +-- Are pages loaded lazily? (Check 7)
|   +-- Are canvases zeroed before removal? (Check 6)
|
+-- Step 5: Check ERROR HANDLING
|   +-- Are exceptions caught? (Check 10)
|   +-- Is render cancellation handled gracefully? (Check 3, 10)
|   +-- Are password PDFs handled? (Check 10)
|
+-- Step 6: Check ANTI-PATTERNS
    +-- See references/anti-patterns.md for consolidated list
```

---

## Review Output Format

When reporting review results, use this format for each finding:

```
[FAIL] Check #N: <check name>
Severity: Critical | High | Medium | Low
Location: <file>:<line>
Issue: <what is wrong>
Fix: <what to do instead>
```

When all checks pass:

```
[PASS] All 10 checks passed. No anti-patterns detected.
```

---

## Reference Links

- [references/methods.md](references/methods.md) -- Complete checklist items with detailed PASS/FAIL criteria
- [references/examples.md](references/examples.md) -- Example review of good and bad PDF.js code
- [references/anti-patterns.md](references/anti-patterns.md) -- Consolidated anti-patterns from all PDF.js skills

### Official Sources

- https://mozilla.github.io/pdf.js/api/ -- PDF.js API reference
- https://github.com/mozilla/pdf.js/tree/master/examples -- Official examples
- https://github.com/mozilla/pdf.js -- Source code and types
