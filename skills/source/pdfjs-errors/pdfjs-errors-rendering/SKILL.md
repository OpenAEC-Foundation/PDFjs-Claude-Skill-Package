---
name: pdfjs-errors-rendering
description: >
  Use when debugging canvas rendering errors, blurry text, render task race conditions,
  or memory issues from PDF page rendering. Prevents blurry rendering by enforcing
  devicePixelRatio scaling and avoids memory leaks from concurrent render conflicts.
  Covers canvas errors, text layer positioning, annotation layer z-index issues,
  and memory management for multi-page rendering.
  Keywords: blurry PDF, devicePixelRatio, RenderTask, canvas error, memory leak, z-index.
license: MIT
compatibility: "Designed for Claude Code. Requires pdfjs-dist 5.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# pdfjs-errors-rendering

## Quick Reference

### Rendering Error Types

| Symptom | Likely Cause | Severity |
|---------|-------------|----------|
| Blurry/fuzzy text and images | Missing `devicePixelRatio` scaling | High |
| White or blank page | `render()` not awaited or canvas dimensions are 0 | Critical |
| `RenderingCancelledException` | Previous render not cancelled before starting new one | Medium |
| Page renders then disappears | Concurrent `render()` calls on same canvas | High |
| Text layer offset from rendered text | Viewport mismatch between canvas and text layer | High |
| Annotations not clickable | Wrong `z-index` or `pointer-events: none` on annotation layer | Medium |
| Browser tab crashes / OOM | All pages rendered simultaneously without cleanup | Critical |
| Canvas context lost (black canvas) | Too many active canvases exceeding browser GPU limit | High |
| Extremely slow rendering | Rendering at excessive scale without throttling | Medium |

### Critical Warnings

**ALWAYS** multiply canvas `width` and `height` by `window.devicePixelRatio` and apply a CSS transform or style to compensate -- failing to do this is the #1 cause of blurry PDF rendering.

**ALWAYS** cancel the previous `RenderTask` before starting a new `render()` call on the same canvas -- concurrent renders on the same canvas produce corrupted output or `RenderingCancelledException`.

**NEVER** render all pages of a multi-page PDF simultaneously -- this allocates massive canvas memory and will crash the browser tab on documents with more than ~50 pages.

**ALWAYS** await `renderTask.promise` before performing any further operations on the canvas -- not awaiting causes white/blank pages because the render has not completed.

**NEVER** set canvas dimensions to 0 or leave them unset -- a canvas with `width: 0` or `height: 0` produces a blank page with no error message.

**ALWAYS** use the same `viewport` for the canvas, text layer, and annotation layer -- mismatched viewports cause text selection and annotations to appear offset from the rendered content.

---

## Diagnostic Decision Tree

```
PDF rendering problem?
|
+-- Blurry / fuzzy output
|   +-- devicePixelRatio not applied? -> Scale canvas by DPR (see Fix 1)
|   +-- CSS width/height not set? -> Set CSS dimensions to logical size
|   +-- Scale too low? -> Increase viewport scale parameter
|
+-- White / blank page
|   +-- render() not awaited? -> ALWAYS await renderTask.promise
|   +-- Canvas width or height is 0? -> Set dimensions from viewport BEFORE render()
|   +-- Canvas not in DOM? -> Ensure canvas is attached before render()
|   +-- getContext("2d") returns null? -> Canvas already has WebGL context or is detached
|
+-- RenderingCancelledException
|   +-- User scrolled / zoomed quickly? -> Expected; cancel previous render first
|   +-- Multiple render() on same canvas? -> Implement cancel-then-render pattern (see Fix 2)
|   +-- Component re-rendered? -> Cancel render in cleanup/unmount
|
+-- Text layer misaligned
|   +-- Different viewport for text vs canvas? -> Use SAME viewport object (see Fix 4)
|   +-- Missing CSS from pdfjs-dist? -> Import text layer stylesheet
|   +-- Container has padding/margin? -> Use position:relative container, no padding
|
+-- Annotations not clickable
|   +-- Annotation layer behind canvas? -> Set z-index above canvas (see Fix 5)
|   +-- pointer-events: none? -> Remove or override to pointer-events: auto
|   +-- Annotation layer mispositioned? -> Use same viewport as canvas
|
+-- Browser tab crashes / very slow
|   +-- All pages rendered at once? -> Implement lazy rendering (see Fix 3)
|   +-- Scale too high? -> Cap scale at 3x-4x DPR
|   +-- Canvases not cleaned up? -> Destroy off-screen canvases
|
+-- Canvas goes black / context lost
    +-- Too many canvases? -> Limit active canvases to ~6-8 (see Fix 6)
    +-- Very large canvas? -> Check canvas area < browser limit (~16M pixels)
    +-- GPU memory exhausted? -> Release canvases for off-screen pages
```

---

## Essential Fixes

### Fix 1: Blurry Rendering (devicePixelRatio)

**Symptom**: PDF text and images appear blurry, especially on Retina/HiDPI displays.

**Cause**: Canvas `width`/`height` attributes match CSS pixels instead of physical pixels.

```typescript
import type { PDFPageProxy } from "pdfjs-dist";

function renderPageSharp(
  page: PDFPageProxy,
  canvas: HTMLCanvasElement,
  scale: number = 1.5
): Promise<void> {
  const viewport = page.getViewport({ scale });
  const dpr = window.devicePixelRatio || 1;

  // Physical pixel dimensions (ALWAYS multiply by DPR)
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);

  // CSS dimensions (logical pixels for layout)
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;

  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);

  return page.render({ canvasContext: ctx, viewport }).promise;
}
```

**Prevention**: ALWAYS apply `devicePixelRatio` scaling. NEVER set canvas `width`/`height` equal to viewport dimensions without multiplying by DPR.

### Fix 2: Concurrent Render Conflicts (Cancel-Then-Render)

**Symptom**: `RenderingCancelledException`, corrupted rendering, or page flickers.

**Cause**: A new `render()` call starts while a previous one is still running on the same canvas.

```typescript
import { getDocument } from "pdfjs-dist";
import type { PDFPageProxy, RenderTask } from "pdfjs-dist";

let currentRenderTask: RenderTask | null = null;

async function renderPage(
  page: PDFPageProxy,
  canvas: HTMLCanvasElement,
  scale: number
): Promise<void> {
  // ALWAYS cancel the previous render before starting a new one
  if (currentRenderTask) {
    currentRenderTask.cancel();
    currentRenderTask = null;
  }

  const viewport = page.getViewport({ scale });
  const dpr = window.devicePixelRatio || 1;
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;

  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);

  currentRenderTask = page.render({ canvasContext: ctx, viewport });

  try {
    await currentRenderTask.promise;
  } catch (err: unknown) {
    if (err instanceof Error && err.name === "RenderingCancelledException") {
      // Expected when cancel() was called -- not an error
      return;
    }
    throw err;
  } finally {
    currentRenderTask = null;
  }
}
```

**Prevention**: ALWAYS store the `RenderTask` reference and call `cancel()` before starting a new render on the same canvas.

### Fix 3: Memory Explosion (Lazy Page Rendering)

**Symptom**: Browser tab crashes, excessive memory usage, or OOM errors on large documents.

**Cause**: All pages are rendered to canvases simultaneously.

```typescript
import type { PDFDocumentProxy } from "pdfjs-dist";

// CORRECT: Only render pages that are visible in the viewport
function getVisiblePages(
  container: HTMLElement,
  pageHeight: number,
  totalPages: number
): { first: number; last: number } {
  const scrollTop = container.scrollTop;
  const viewportHeight = container.clientHeight;
  const buffer = pageHeight; // One page buffer above and below

  const first = Math.max(1, Math.floor((scrollTop - buffer) / pageHeight) + 1);
  const last = Math.min(
    totalPages,
    Math.ceil((scrollTop + viewportHeight + buffer) / pageHeight)
  );

  return { first, last };
}

// ALWAYS clean up off-screen canvases to free memory
function cleanupOffScreenPages(
  canvasMap: Map<number, HTMLCanvasElement>,
  visibleFirst: number,
  visibleLast: number
): void {
  for (const [pageNum, canvas] of canvasMap) {
    if (pageNum < visibleFirst - 1 || pageNum > visibleLast + 1) {
      canvas.width = 0;
      canvas.height = 0;
      canvasMap.delete(pageNum);
    }
  }
}
```

**Prevention**: NEVER call `render()` on all pages at once. ALWAYS implement an intersection observer or scroll-based visibility check to render only visible pages plus a small buffer.

### Fix 4: Text Layer Misalignment

**Symptom**: Selecting text highlights the wrong area, or text overlay is offset from rendered content.

**Cause**: The text layer uses a different viewport or the container CSS is incorrect.

See [references/examples.md](references/examples.md) for the complete text layer alignment pattern.

**Prevention**: ALWAYS use the exact same `viewport` object for both canvas rendering and text layer rendering. ALWAYS import the PDF.js text layer CSS. ALWAYS use a `position: relative` container with no padding.

### Fix 5: Annotation Layer Not Interactive

**Symptom**: Links, form fields, and annotations in the PDF are not clickable.

**Cause**: The annotation layer `<div>` sits behind the canvas in the stacking order, or has `pointer-events: none`.

See [references/examples.md](references/examples.md) for the complete annotation layer stacking pattern.

**Prevention**: ALWAYS set the annotation layer `z-index` higher than the canvas. NEVER apply `pointer-events: none` to the annotation layer container.

### Fix 6: Canvas Context Lost

**Symptom**: Previously rendered pages turn black or blank after scrolling.

**Cause**: The browser has a limit on simultaneous canvas contexts (typically 8-16 for hardware-accelerated contexts). Exceeding this limit causes older canvases to lose their context silently.

```typescript
const MAX_ACTIVE_CANVASES = 6;
const activeCanvases: Map<number, HTMLCanvasElement> = new Map();

function releaseCanvas(pageNum: number): void {
  const canvas = activeCanvases.get(pageNum);
  if (canvas) {
    // Setting dimensions to 0 releases GPU memory
    canvas.width = 0;
    canvas.height = 0;
    activeCanvases.delete(pageNum);
  }
}

function enforceCanvasLimit(currentPage: number): void {
  if (activeCanvases.size <= MAX_ACTIVE_CANVASES) return;

  // Release canvases furthest from the current page
  const sorted = [...activeCanvases.keys()].sort(
    (a, b) => Math.abs(a - currentPage) - Math.abs(b - currentPage)
  );

  while (sorted.length > MAX_ACTIVE_CANVASES) {
    const pageToRelease = sorted.pop()!;
    releaseCanvas(pageToRelease);
  }
}
```

**Prevention**: ALWAYS limit the number of active canvases. ALWAYS set `canvas.width = 0; canvas.height = 0;` to release GPU memory for off-screen pages.

---

## Prevention Patterns

### DPR-Aware Render Helper

```typescript
import type { PDFPageProxy, RenderTask } from "pdfjs-dist";

interface RenderResult {
  task: RenderTask;
  viewport: ReturnType<PDFPageProxy["getViewport"]>;
}

function renderWithDPR(
  page: PDFPageProxy,
  canvas: HTMLCanvasElement,
  scale: number = 1.5
): RenderResult {
  const viewport = page.getViewport({ scale });
  const dpr = window.devicePixelRatio || 1;

  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;

  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);

  const task = page.render({ canvasContext: ctx, viewport });
  return { task, viewport };
}
```

### Safe Page Render Queue

```typescript
import type { PDFPageProxy, RenderTask } from "pdfjs-dist";

class PageRenderQueue {
  private activeTasks = new Map<number, RenderTask>();

  async render(
    pageNum: number,
    page: PDFPageProxy,
    canvas: HTMLCanvasElement,
    scale: number
  ): Promise<void> {
    // Cancel existing render for this page
    const existing = this.activeTasks.get(pageNum);
    if (existing) {
      existing.cancel();
    }

    const viewport = page.getViewport({ scale });
    const dpr = window.devicePixelRatio || 1;
    canvas.width = Math.floor(viewport.width * dpr);
    canvas.height = Math.floor(viewport.height * dpr);
    canvas.style.width = `${Math.floor(viewport.width)}px`;
    canvas.style.height = `${Math.floor(viewport.height)}px`;

    const ctx = canvas.getContext("2d")!;
    ctx.scale(dpr, dpr);

    const task = page.render({ canvasContext: ctx, viewport });
    this.activeTasks.set(pageNum, task);

    try {
      await task.promise;
    } catch (err: unknown) {
      if (err instanceof Error && err.name === "RenderingCancelledException") {
        return; // Expected cancellation
      }
      throw err;
    } finally {
      this.activeTasks.delete(pageNum);
    }
  }

  cancelAll(): void {
    for (const task of this.activeTasks.values()) {
      task.cancel();
    }
    this.activeTasks.clear();
  }
}
```

---

## Reference Links

- [references/methods.md](references/methods.md) -- RenderTask API, canvas context methods, viewport configuration
- [references/examples.md](references/examples.md) -- Complete error recovery patterns, DPI handling, text/annotation layer alignment
- [references/anti-patterns.md](references/anti-patterns.md) -- Common causes of every rendering issue

### Official Sources

- https://mozilla.github.io/pdf.js/api/ -- PDF.js API reference
- https://github.com/mozilla/pdf.js -- Source code (rendering pipeline)
- https://github.com/mozilla/pdf.js/tree/master/examples -- Official rendering examples
