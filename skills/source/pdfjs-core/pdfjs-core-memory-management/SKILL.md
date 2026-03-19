---
name: pdfjs-core-memory-management
description: >
  Use when building PDF viewers that load multiple documents, navigate between pages,
  or run for extended periods. Prevents the #1 production PDF.js issue: memory leaks
  from unreleased PDFDocumentProxy, uncleaned pages, and orphaned canvas contexts.
  Covers destroy/cleanup lifecycle, canvas reuse, render task cancellation, blob URL
  revocation, and large PDF handling strategies.
  Keywords: memory leak, destroy, cleanup, PDFDocumentProxy, canvas, blob URL, garbage collection.
license: MIT
compatibility: "Designed for Claude Code. Requires pdfjs-dist 5.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# pdfjs-core-memory-management

## Quick Reference

### Resource Lifecycle Overview

| Resource | Acquire | Release | When to Release |
|----------|---------|---------|-----------------|
| `PDFDocumentProxy` | `getDocument().promise` | `.destroy()` | When switching documents or unmounting viewer |
| `PDFPageProxy` | `pdfDoc.getPage(n)` | `.cleanup()` | When page scrolls off-screen or is evicted from pool |
| `RenderTask` | `page.render(params)` | `.cancel()` | BEFORE destroying page, re-rendering, or unmounting |
| Canvas context | `canvas.getContext('2d')` | Reset dimensions + `clearRect` | Before reuse or removal from DOM |
| `TextLayer` | `new TextLayer(params)` | `.cancel()` | Before destroying the page container |
| `AnnotationLayer` | `annotationLayer.render()` | `.cancel()` | Before destroying the page container |
| Blob URL | `URL.createObjectURL(blob)` | `URL.revokeObjectURL(url)` | Immediately after passing to `getDocument()` |
| Event listeners | `addEventListener()` | `removeEventListener()` | On destroy/unmount |

### Destroy Order (CRITICAL)

ALWAYS destroy resources in this exact order:

```
1. Cancel all active RenderTasks
2. Cancel TextLayer and AnnotationLayer instances
3. Call PDFPageProxy.cleanup() on all loaded pages
4. Clear and reset canvas elements
5. Remove event listeners
6. Call PDFDocumentProxy.destroy()
7. Revoke any blob URLs
8. Remove DOM elements
```

Violating this order causes errors. Destroying a `PDFDocumentProxy` while a `RenderTask` is active throws an unhandled exception in the worker.

### Critical Warnings

**NEVER** call `PDFDocumentProxy.destroy()` without first cancelling all active `RenderTask` instances -- the worker throws unhandled exceptions for in-flight operations.

**NEVER** keep references to `PDFPageProxy` objects after calling `PDFDocumentProxy.destroy()` -- they become invalid and any method call throws.

**NEVER** create a new canvas for each page render in a single-page viewer -- reuse the same canvas and clear it. Creating canvases without removing old ones leaks GPU memory.

**NEVER** load a PDF from a blob URL without revoking it after `getDocument()` resolves -- each unreleased blob URL holds the entire PDF binary in memory.

**NEVER** rely on garbage collection to clean up PDF.js resources -- the worker thread and internal caches are NOT released by GC. ALWAYS call `destroy()` explicitly.

**ALWAYS** cancel `RenderTask` before starting a new render on the same canvas -- concurrent renders corrupt canvas state.

**ALWAYS** clean up PDF.js resources on SPA route changes -- leaving active workers behind is the most common memory leak in production PDF viewers.

**ALWAYS** use a page pool with a maximum size for multi-page viewers -- keeping every visited page in memory crashes long-running sessions.

---

## Decision Tree: Cleanup Strategy

```
Is the user navigating away from the PDF viewer entirely?
├── YES → Full Cleanup
│   ├── Cancel all RenderTasks
│   ├── Cancel TextLayer/AnnotationLayer
│   ├── Cleanup all PDFPageProxy instances
│   ├── Destroy PDFDocumentProxy
│   ├── Revoke blob URLs
│   └── Remove DOM elements and event listeners
│
└── NO → Is the user switching to a different PDF?
    ├── YES → Document Swap
    │   ├── Cancel active RenderTasks
    │   ├── Destroy PREVIOUS PDFDocumentProxy
    │   ├── Revoke previous blob URL
    │   ├── Clear canvas
    │   └── Load new document
    │
    └── NO → Is the user scrolling/navigating pages?
        ├── YES → Page Pool Management
        │   ├── Use IntersectionObserver for visibility
        │   ├── Cancel RenderTask on off-screen pages
        │   ├── Call cleanup() on evicted pages
        │   ├── Clear off-screen canvases
        │   └── Keep pool size under budget (e.g., 5-10 pages)
        │
        └── NO → Is the user zooming/resizing?
            ├── YES → Re-render Cleanup
            │   ├── Cancel current RenderTask
            │   ├── Clear canvas
            │   ├── Create new viewport
            │   └── Start new render
            │
            └── NO → No cleanup needed
```

---

## PDFDocumentProxy.destroy()

The most important cleanup method. Releases the worker thread, internal page cache, and all document data.

```typescript
let currentDoc: PDFDocumentProxy | null = null;

async function loadDocument(source: string | ArrayBuffer): Promise<void> {
  // ALWAYS destroy previous document before loading new one
  if (currentDoc) {
    await currentDoc.destroy();
    currentDoc = null;
  }

  const loadingTask = getDocument(source);
  currentDoc = await loadingTask.promise;
}
```

`destroy()` returns a `Promise`. ALWAYS `await` it when loading a replacement document to ensure resources are fully released before allocating new ones.

---

## PDFPageProxy.cleanup()

Releases cached rendering data (operator list, image data) for a specific page. The page can be re-fetched via `getPage()` after cleanup -- it is NOT destroyed, just evicted from cache.

```typescript
// Cleanup a page that scrolled off-screen
function evictPage(page: PDFPageProxy): void {
  page.cleanup();
}
```

**Key distinction**: `cleanup()` releases the cached data but the `PDFPageProxy` object remains valid. You can call `render()` again later and it will re-fetch data from the worker. Use this for page pooling, NOT for final teardown.

---

## RenderTask.cancel()

ALWAYS cancel active render tasks before any of these operations:
- Destroying the page or document
- Starting a new render on the same canvas
- Removing the canvas from the DOM
- Navigating to a different page (single-page viewer)

```typescript
let activeRenderTask: RenderTask | null = null;

async function renderPage(page: PDFPageProxy, ctx: CanvasRenderingContext2D, viewport: PageViewport): Promise<void> {
  if (activeRenderTask) {
    activeRenderTask.cancel();
  }

  activeRenderTask = page.render({ canvasContext: ctx, viewport });

  try {
    await activeRenderTask.promise;
  } catch (err: any) {
    if (err.name === 'RenderingCancelledException') {
      return; // Expected -- not an error
    }
    throw err;
  } finally {
    activeRenderTask = null;
  }
}
```

---

## Canvas Cleanup

Canvas elements consume GPU-backed memory. ALWAYS clean up canvases explicitly:

```typescript
function clearCanvas(canvas: HTMLCanvasElement): void {
  const ctx = canvas.getContext('2d');
  if (ctx) {
    ctx.clearRect(0, 0, canvas.width, canvas.height);
  }
  // Reset dimensions to release GPU memory
  canvas.width = 0;
  canvas.height = 0;
}
```

Setting `canvas.width = 0` forces the browser to release the backing store. This is the ONLY reliable way to free canvas GPU memory without removing the element from the DOM.

---

## Blob URL Revocation

When loading PDFs from `File` objects or `Blob` data:

```typescript
async function loadFromFile(file: File): Promise<void> {
  const url = URL.createObjectURL(file);

  try {
    if (currentDoc) {
      await currentDoc.destroy();
    }
    const loadingTask = getDocument(url);
    currentDoc = await loadingTask.promise;
  } finally {
    // ALWAYS revoke after getDocument resolves or rejects
    URL.revokeObjectURL(url);
  }
}
```

PDF.js reads the full data during `getDocument()`. Once the promise resolves, the blob URL is no longer needed. Revoking it immediately releases the duplicate copy held by the URL reference.

---

## IntersectionObserver Page Lifecycle

For multi-page viewers, use `IntersectionObserver` to manage the render/cleanup cycle:

```typescript
const PAGE_POOL_SIZE = 7; // Maximum pages kept rendered
const renderedPages = new Map<number, { page: PDFPageProxy; renderTask: RenderTask | null }>();

const observer = new IntersectionObserver(
  (entries) => {
    for (const entry of entries) {
      const pageNum = Number(entry.target.dataset.pageNum);
      if (entry.isIntersecting) {
        renderPageInView(pageNum, entry.target as HTMLElement);
      } else {
        scheduleEviction(pageNum);
      }
    }
  },
  { rootMargin: '200px' } // Pre-render 200px before visible
);

function scheduleEviction(pageNum: number): void {
  if (renderedPages.size <= PAGE_POOL_SIZE) return;

  const entry = renderedPages.get(pageNum);
  if (!entry) return;

  if (entry.renderTask) {
    entry.renderTask.cancel();
  }
  entry.page.cleanup();

  const canvas = document.querySelector(`[data-page-num="${pageNum}"] canvas`);
  if (canvas) clearCanvas(canvas as HTMLCanvasElement);

  renderedPages.delete(pageNum);
}
```

---

## SPA Route Change Cleanup

In Single Page Applications, ALWAYS register a cleanup function for route changes:

```typescript
// React example
useEffect(() => {
  let doc: PDFDocumentProxy | null = null;
  let renderTask: RenderTask | null = null;

  async function init() {
    doc = await getDocument(pdfUrl).promise;
    const page = await doc.getPage(1);
    renderTask = page.render({ canvasContext: ctx, viewport });
    await renderTask.promise;
  }

  init();

  // Cleanup on unmount / route change
  return () => {
    if (renderTask) renderTask.cancel();
    if (doc) doc.destroy();
  };
}, [pdfUrl]);
```

```typescript
// Vanilla SPA with event listener
function mountPdfViewer(container: HTMLElement): () => void {
  let doc: PDFDocumentProxy | null = null;
  const renderTasks: RenderTask[] = [];

  // ... setup code ...

  // Return cleanup function
  return () => {
    renderTasks.forEach((task) => task.cancel());
    if (doc) doc.destroy();
    container.innerHTML = '';
  };
}

// On route change
const cleanup = mountPdfViewer(container);
router.onNavigate(() => cleanup());
```

---

## Memory Diagnostics

### Chrome DevTools Heap Snapshot

To diagnose PDF.js memory leaks:

1. Open DevTools > Memory > Heap Snapshot
2. Load a PDF, navigate pages, then switch documents
3. Take snapshot > filter for `PDFDocumentProxy`, `PDFPageProxy`, `HTMLCanvasElement`
4. If instances accumulate across document loads, `destroy()` is missing
5. Check Retainers panel to find what holds references

### Key Indicators of Leaks

| Symptom | Likely Cause |
|---------|--------------|
| `PDFDocumentProxy` count grows | Missing `destroy()` on document switch |
| `HTMLCanvasElement` count grows | Canvas created but never removed or cleared |
| Worker thread count grows (Performance tab) | `destroy()` not called -- each `getDocument()` spawns a worker |
| Detached DOM tree (in Heap Snapshot) | Event listeners preventing GC of removed elements |
| ArrayBuffer memory grows | Blob URLs not revoked, or page data not cleaned up |

---

## Reference Links

- [references/methods.md](references/methods.md) -- Cleanup and destroy API signatures
- [references/examples.md](references/examples.md) -- Complete working code examples
- [references/anti-patterns.md](references/anti-patterns.md) -- Memory leak patterns with WRONG/RIGHT code

### Official Sources

- https://mozilla.github.io/pdf.js/api/
- https://github.com/nicolo-ribaudo/pdfjs-dist/blob/master/types/src/display/api.d.ts
- https://github.com/nicolo-ribaudo/pdfjs-dist
