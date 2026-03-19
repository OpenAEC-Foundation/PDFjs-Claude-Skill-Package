---
name: pdfjs-syntax-page-rendering
description: "Renders PDF pages to canvas using page.render() and manages the rendering pipeline. Covers RenderTask lifecycle, viewport creation with scale/rotation, high-DPI canvas scaling with devicePixelRatio, render cancellation patterns, and layer stacking order. Activates when rendering PDF pages, handling zoom/rotation, fixing blurry PDF rendering, or managing render tasks."
license: MIT
compatibility: "Designed for Claude Code. Requires pdfjs-dist 5.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# pdfjs-syntax-page-rendering

## Quick Reference

### Rendering Pipeline

| Step | Method | Output |
|------|--------|--------|
| 1. Load document | `getDocument({ url }).promise` | `PDFDocumentProxy` |
| 2. Get page | `doc.getPage(pageNumber)` | `PDFPageProxy` |
| 3. Create viewport | `page.getViewport({ scale })` | `PageViewport` |
| 4. Setup canvas | Set `canvas.width/height` with DPI scaling | `<canvas>` element |
| 5. Render | `page.render({ canvasContext, viewport })` | `RenderTask` |
| 6. Await completion | `renderTask.promise` | Rendered canvas |

### Layer Stacking Order

| Layer | z-index | Purpose |
|-------|---------|---------|
| Canvas | 0 | PDF page pixels |
| TextLayer | 1 | Selectable/searchable text overlay |
| AnnotationLayer | 2 | Links, forms, annotations |

### Critical Warnings

**NEVER** start a new render without cancelling the previous RenderTask -- this causes race conditions where two renders fight over the same canvas, producing visual artifacts and memory leaks.

**ALWAYS** handle `devicePixelRatio` -- without it, PDF renders blurry on high-DPI screens (Retina, 4K). The canvas MUST have its pixel dimensions scaled by the device pixel ratio while keeping CSS dimensions at viewport size.

**NEVER** set `canvas.width/height` equal to `viewport.width/height` without DPI scaling -- this produces blurry output on any screen with `devicePixelRatio > 1`.

**ALWAYS** set BOTH canvas pixel dimensions AND CSS style dimensions -- pixel dimensions control resolution, CSS dimensions control display size.

**NEVER** render all pages at once -- use lazy loading with `IntersectionObserver` to render only visible pages. Rendering all pages simultaneously causes massive memory consumption and UI freezing.

**ALWAYS** await `renderTask.promise` before accessing the canvas content -- the render is asynchronous and the canvas is incomplete until the promise resolves.

---

## Essential Patterns

### Basic Page Rendering (with DPI handling)

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";

// ALWAYS configure worker before loading documents
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

async function renderPage(
  url: string,
  pageNumber: number,
  canvas: HTMLCanvasElement,
  scale: number = 1.5
): Promise<void> {
  const doc = await getDocument(url).promise;
  const page = await doc.getPage(pageNumber);
  const viewport = page.getViewport({ scale });

  // ALWAYS apply devicePixelRatio for sharp rendering
  const dpr = window.devicePixelRatio || 1;
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;

  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);

  const renderTask = page.render({
    canvasContext: ctx,
    viewport: viewport,
  });

  await renderTask.promise;
}
```

### Viewport Creation

```typescript
// Basic viewport with scale
const viewport = page.getViewport({ scale: 1.5 });

// Viewport with rotation (0, 90, 180, or 270 degrees)
const rotatedViewport = page.getViewport({ scale: 1.5, rotation: 90 });

// Viewport with offset
const offsetViewport = page.getViewport({
  scale: 1.0,
  offsetX: 10,
  offsetY: 20,
});

// Clone viewport with different parameters
const zoomedViewport = viewport.clone({ scale: 2.0 });
const rotatedClone = viewport.clone({ rotation: 180 });
```

### Cancel-Then-Render Pattern (Zoom / Rotation)

```typescript
import type { RenderTask } from "pdfjs-dist";

let currentRenderTask: RenderTask | null = null;

async function reRenderPage(
  page: PDFPageProxy,
  canvas: HTMLCanvasElement,
  scale: number,
  rotation: number = 0
): Promise<void> {
  // ALWAYS cancel previous render before starting a new one
  if (currentRenderTask) {
    currentRenderTask.cancel();
    currentRenderTask = null;
  }

  const viewport = page.getViewport({ scale, rotation });

  const dpr = window.devicePixelRatio || 1;
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;

  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);

  currentRenderTask = page.render({
    canvasContext: ctx,
    viewport: viewport,
  });

  try {
    await currentRenderTask.promise;
  } catch (err: unknown) {
    // RenderTask.cancel() causes the promise to reject -- this is expected
    if (err instanceof Error && err.message === "Rendering cancelled") {
      return; // Normal cancellation, not an error
    }
    throw err; // Re-throw actual errors
  } finally {
    currentRenderTask = null;
  }
}
```

### Render Parameters

```typescript
const renderTask = page.render({
  // REQUIRED
  canvasContext: ctx,             // CanvasRenderingContext2D
  viewport: viewport,            // PageViewport from getViewport()

  // OPTIONAL
  transform: [2, 0, 0, 2, 0, 0], // Additional CSS transform matrix
  background: "rgba(0,0,0,0)",    // Canvas background (default: white)
  annotationMode: 2,              // 0=DISABLE, 1=ENABLE, 2=ENABLE_FORMS, 3=ENABLE_STORAGE
  intent: "display",              // "display" | "print" | "any"
});
```

### annotationMode Values

| Value | Constant | Behavior |
|-------|----------|----------|
| 0 | `AnnotationMode.DISABLE` | No annotations rendered |
| 1 | `AnnotationMode.ENABLE` | Render annotations (read-only) |
| 2 | `AnnotationMode.ENABLE_FORMS` | Render with interactive forms |
| 3 | `AnnotationMode.ENABLE_STORAGE` | Render with persistent form data |

---

## Common Operations

### Zoom Implementation

```typescript
let currentScale = 1.5;

function zoomIn(page: PDFPageProxy, canvas: HTMLCanvasElement): void {
  currentScale *= 1.25;
  reRenderPage(page, canvas, currentScale);
}

function zoomOut(page: PDFPageProxy, canvas: HTMLCanvasElement): void {
  currentScale /= 1.25;
  reRenderPage(page, canvas, currentScale);
}

function fitToWidth(
  page: PDFPageProxy,
  canvas: HTMLCanvasElement,
  containerWidth: number
): void {
  const baseViewport = page.getViewport({ scale: 1.0 });
  currentScale = containerWidth / baseViewport.width;
  reRenderPage(page, canvas, currentScale);
}
```

### Rotation Implementation

```typescript
let currentRotation = 0;

function rotate(
  page: PDFPageProxy,
  canvas: HTMLCanvasElement,
  degrees: number
): void {
  // Rotation MUST be a multiple of 90
  currentRotation = (currentRotation + degrees) % 360;
  if (currentRotation < 0) currentRotation += 360;
  reRenderPage(page, canvas, currentScale, currentRotation);
}
```

### OffscreenCanvas Rendering (Web Worker)

```typescript
// Inside a Web Worker
async function renderToOffscreen(
  page: PDFPageProxy,
  scale: number
): Promise<ImageBitmap> {
  const viewport = page.getViewport({ scale });
  const dpr = 1; // Workers typically use DPR=1 or receive it from main thread
  const offscreen = new OffscreenCanvas(
    Math.floor(viewport.width * dpr),
    Math.floor(viewport.height * dpr)
  );
  const ctx = offscreen.getContext("2d")!;
  ctx.scale(dpr, dpr);

  const renderTask = page.render({
    canvasContext: ctx,
    viewport: viewport,
  });

  await renderTask.promise;
  return offscreen.transferToImageBitmap();
}
```

---

## Decision Tree: Canvas Setup

```
Need to render a PDF page?
├── Is this a new render (no previous render on this canvas)?
│   ├── YES → Create viewport → Setup canvas with DPI → Render
│   └── NO → Cancel previous RenderTask FIRST → Then re-render
│
├── Is this for screen display?
│   ├── YES → Use window.devicePixelRatio for DPI scaling
│   └── NO (printing/export) → Use intent: "print", consider DPR=2 or higher
│
├── Need zoom?
│   └── Change scale → New viewport → Cancel old render → Re-render
│
├── Need rotation?
│   └── Change rotation → New viewport → Cancel old render → Re-render
│
└── Rendering in Web Worker?
    └── Use OffscreenCanvas instead of HTMLCanvasElement
```

---

## Reference Links

- [references/methods.md](references/methods.md) -- Complete API signatures for PageViewport, RenderTask, and render parameters
- [references/examples.md](references/examples.md) -- Working code examples for basic render, high-DPI, zoom, rotation, and lazy loading
- [references/anti-patterns.md](references/anti-patterns.md) -- Common rendering mistakes and their fixes

### Official Sources

- https://mozilla.github.io/pdf.js/api/ -- PDF.js API reference
- https://github.com/mozilla/pdf.js/tree/master/examples -- Official examples
- https://github.com/mozilla/pdf.js -- Source code and types
