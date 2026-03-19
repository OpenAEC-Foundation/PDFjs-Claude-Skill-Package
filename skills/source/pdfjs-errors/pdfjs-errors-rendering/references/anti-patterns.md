# Rendering Anti-Patterns (pdfjs-dist 5.x)

Common mistakes that cause PDF.js rendering errors and how to fix them.

---

## 1. Not Scaling Canvas by devicePixelRatio (Blurry Output)

**Severity**: High -- the #1 most common rendering complaint.

### Wrong

```typescript
import type { PDFPageProxy } from "pdfjs-dist";

function renderPage(page: PDFPageProxy, canvas: HTMLCanvasElement) {
  const viewport = page.getViewport({ scale: 1.5 });

  // BROKEN: Canvas dimensions match CSS pixels, not physical pixels
  canvas.width = viewport.width;
  canvas.height = viewport.height;

  const ctx = canvas.getContext("2d")!;
  page.render({ canvasContext: ctx, viewport });
}
```

### Correct

```typescript
import type { PDFPageProxy } from "pdfjs-dist";

function renderPage(page: PDFPageProxy, canvas: HTMLCanvasElement) {
  const viewport = page.getViewport({ scale: 1.5 });
  const dpr = window.devicePixelRatio || 1;

  // Physical pixel dimensions for sharp rendering
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);

  // CSS dimensions for correct layout
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;

  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);

  page.render({ canvasContext: ctx, viewport });
}
```

**Why**: On a 2x Retina display, `devicePixelRatio` is 2. A canvas that is 800 CSS pixels wide needs to be 1600 physical pixels to render sharply. Without this scaling, the browser upscales the 800-pixel canvas to fill 1600 physical pixels, producing blurry output.

---

## 2. Not Cancelling Previous Render Before Starting New One

**Severity**: High -- causes `RenderingCancelledException`, visual corruption, or crashes.

### Wrong

```typescript
import type { PDFPageProxy } from "pdfjs-dist";

// BROKEN: Each zoom level starts a new render without cancelling the old one
function onZoom(page: PDFPageProxy, canvas: HTMLCanvasElement, scale: number) {
  const viewport = page.getViewport({ scale });
  const ctx = canvas.getContext("2d")!;

  // Previous render is still running -- this corrupts the canvas!
  page.render({ canvasContext: ctx, viewport });
}
```

### Correct

```typescript
import type { PDFPageProxy, RenderTask } from "pdfjs-dist";

let activeTask: RenderTask | null = null;

async function onZoom(
  page: PDFPageProxy,
  canvas: HTMLCanvasElement,
  scale: number
) {
  // ALWAYS cancel the previous render first
  if (activeTask) {
    activeTask.cancel();
    activeTask = null;
  }

  const viewport = page.getViewport({ scale });
  const dpr = window.devicePixelRatio || 1;
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;

  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);

  activeTask = page.render({ canvasContext: ctx, viewport });

  try {
    await activeTask.promise;
  } catch (err: unknown) {
    if (err instanceof Error && err.name === "RenderingCancelledException") {
      return; // Expected cancellation
    }
    throw err;
  } finally {
    activeTask = null;
  }
}
```

**Why**: PDF.js does not support multiple concurrent renders on the same canvas. The second `render()` call conflicts with the first, producing visual artifacts, partial renders, or `RenderingCancelledException`. ALWAYS cancel the previous task and wait for cancellation to complete before starting a new render.

---

## 3. Rendering All Pages at Once (Memory Explosion)

**Severity**: Critical -- crashes browser tabs on large documents.

### Wrong

```typescript
import { getDocument } from "pdfjs-dist";
import type { PDFDocumentProxy } from "pdfjs-dist";

// BROKEN: Renders every page simultaneously
async function renderAllPages(
  doc: PDFDocumentProxy,
  container: HTMLElement
) {
  for (let i = 1; i <= doc.numPages; i++) {
    const page = await doc.getPage(i);
    const viewport = page.getViewport({ scale: 1.5 });
    const canvas = document.createElement("canvas");
    canvas.width = viewport.width;
    canvas.height = viewport.height;
    container.appendChild(canvas);

    const ctx = canvas.getContext("2d")!;
    page.render({ canvasContext: ctx, viewport }); // NOT awaited!
  }
}
```

### Correct

```typescript
import type { PDFDocumentProxy, PDFPageProxy } from "pdfjs-dist";

// CORRECT: Render only visible pages, clean up off-screen pages
async function renderVisiblePages(
  doc: PDFDocumentProxy,
  container: HTMLElement,
  visibleRange: { first: number; last: number },
  canvasMap: Map<number, HTMLCanvasElement>
) {
  // Clean up pages that are no longer visible
  for (const [pageNum, canvas] of canvasMap) {
    if (pageNum < visibleRange.first - 1 || pageNum > visibleRange.last + 1) {
      canvas.width = 0;  // Releases GPU memory
      canvas.height = 0;
      canvasMap.delete(pageNum);
    }
  }

  // Render only visible pages
  for (let i = visibleRange.first; i <= visibleRange.last; i++) {
    if (canvasMap.has(i)) continue; // Already rendered

    const page = await doc.getPage(i);
    const viewport = page.getViewport({ scale: 1.5 });
    const dpr = window.devicePixelRatio || 1;

    const canvas = document.createElement("canvas");
    canvas.width = Math.floor(viewport.width * dpr);
    canvas.height = Math.floor(viewport.height * dpr);
    canvas.style.width = `${Math.floor(viewport.width)}px`;
    canvas.style.height = `${Math.floor(viewport.height)}px`;

    const ctx = canvas.getContext("2d")!;
    ctx.scale(dpr, dpr);

    await page.render({ canvasContext: ctx, viewport }).promise;
    container.appendChild(canvas);
    canvasMap.set(i, canvas);
  }
}
```

**Why**: Each rendered canvas consumes GPU memory proportional to its pixel area. A 100-page PDF rendered at 1.5x scale on a 2x DPR display allocates approximately 100 * (900 * 1200 * 4 * 4) = ~1.7 GB of canvas memory. Most browsers will either crash the tab or silently discard older canvases (causing black/blank pages). ALWAYS render only visible pages and release off-screen canvases.

---

## 4. Using Different Viewports for Canvas and Text Layer

**Severity**: High -- causes text selection to be offset from visible text.

### Wrong

```typescript
import { renderTextLayer } from "pdfjs-dist";
import type { PDFPageProxy } from "pdfjs-dist";

async function render(page: PDFPageProxy, canvas: HTMLCanvasElement, textDiv: HTMLElement) {
  // BROKEN: Two separate viewport objects with potentially different floating-point values
  const canvasViewport = page.getViewport({ scale: 1.5 });
  const textViewport = page.getViewport({ scale: 1.5 }); // Different object!

  // ... render canvas with canvasViewport ...

  const textContent = await page.getTextContent();
  await renderTextLayer({
    textContentSource: textContent,
    container: textDiv,
    viewport: textViewport, // WRONG: different viewport object
  }).promise;
}
```

### Correct

```typescript
import { renderTextLayer } from "pdfjs-dist";
import type { PDFPageProxy } from "pdfjs-dist";

async function render(page: PDFPageProxy, canvas: HTMLCanvasElement, textDiv: HTMLElement) {
  // CORRECT: ONE viewport object used everywhere
  const viewport = page.getViewport({ scale: 1.5 });
  const dpr = window.devicePixelRatio || 1;

  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;

  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);
  await page.render({ canvasContext: ctx, viewport }).promise;

  const textContent = await page.getTextContent();
  await renderTextLayer({
    textContentSource: textContent,
    container: textDiv,
    viewport: viewport, // SAME viewport object as canvas
  }).promise;
}
```

**Why**: Even though two `getViewport({ scale: 1.5 })` calls use the same parameters, they create separate objects. Internal floating-point calculations may produce slightly different transform matrices, causing sub-pixel misalignment between the canvas content and the text layer overlay. ALWAYS reuse the same viewport instance for all layers.

---

## 5. Missing CSS for Text Layer Positioning

**Severity**: High -- text spans pile up at top-left corner instead of overlaying rendered text.

### Wrong

```html
<!-- BROKEN: No text layer CSS -- all text spans stack at (0,0) -->
<div class="textLayer">
  <!-- Text spans have position:absolute but no transform -->
</div>
```

### Correct

```typescript
// ALWAYS import the PDF.js text layer stylesheet
// Option 1: Import in your CSS/SCSS
// @import "pdfjs-dist/web/pdf_viewer.css";

// Option 2: Import in your JavaScript (bundler will handle it)
import "pdfjs-dist/web/pdf_viewer.css";
```

**Why**: PDF.js `renderTextLayer()` creates `<span>` elements with CSS `transform` properties for positioning. Without the PDF.js stylesheet, these spans lack required base styles (`position: absolute`, `white-space: pre`, `color: transparent`) and pile up at the origin. ALWAYS import the PDF.js CSS or replicate its essential text layer styles.

---

## 6. Setting pointer-events: none on Annotation Layer

**Severity**: Medium -- annotations are visible but not clickable.

### Wrong

```css
/* BROKEN: Prevents all click/hover events on annotations */
.annotationLayer {
  position: absolute;
  pointer-events: none; /* Links and form fields cannot be clicked! */
}
```

### Correct

```css
.annotationLayer {
  position: absolute;
  inset: 0;
  z-index: 3;
  pointer-events: auto; /* NEVER set to none */
}
```

**Why**: Developers sometimes set `pointer-events: none` on overlay layers to prevent them from intercepting scroll or click events meant for the canvas. This disables all annotation interactivity. ALWAYS keep `pointer-events: auto` on the annotation layer. If you need to allow scroll-through, handle it with targeted event listeners instead.

---

## 7. Not Awaiting render().promise (White Pages)

**Severity**: Critical -- pages appear blank with no error.

### Wrong

```typescript
import type { PDFPageProxy } from "pdfjs-dist";

function renderPage(page: PDFPageProxy, canvas: HTMLCanvasElement) {
  const viewport = page.getViewport({ scale: 1.5 });
  const ctx = canvas.getContext("2d")!;

  // BROKEN: render() is not awaited -- function returns before render completes
  page.render({ canvasContext: ctx, viewport });

  // Canvas is still blank here!
  doSomethingWithCanvas(canvas); // Operates on incomplete render
}
```

### Correct

```typescript
import type { PDFPageProxy } from "pdfjs-dist";

async function renderPage(page: PDFPageProxy, canvas: HTMLCanvasElement) {
  const viewport = page.getViewport({ scale: 1.5 });
  const dpr = window.devicePixelRatio || 1;

  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;

  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);

  // ALWAYS await the render promise
  await page.render({ canvasContext: ctx, viewport }).promise;

  // Canvas content is now fully rendered
  doSomethingWithCanvas(canvas);
}
```

**Why**: `page.render()` returns a `RenderTask` object, not a Promise. The actual rendering is asynchronous. Without awaiting `renderTask.promise`, the canvas remains blank or partially drawn. ALWAYS await the `.promise` property to ensure rendering is complete before using the canvas.

---

## 8. Canvas Dimensions Set to Zero (Silent Blank Page)

**Severity**: Critical -- no error, just a blank page.

### Wrong

```typescript
import type { PDFPageProxy } from "pdfjs-dist";

async function renderPage(page: PDFPageProxy, canvas: HTMLCanvasElement) {
  const viewport = page.getViewport({ scale: 1.5 });
  const ctx = canvas.getContext("2d")!;

  // BROKEN: Canvas width/height never set -- defaults to 300x150
  // or may be 0x0 if previously cleared
  await page.render({ canvasContext: ctx, viewport }).promise;
}
```

### Correct

```typescript
import type { PDFPageProxy } from "pdfjs-dist";

async function renderPage(page: PDFPageProxy, canvas: HTMLCanvasElement) {
  const viewport = page.getViewport({ scale: 1.5 });
  const dpr = window.devicePixelRatio || 1;

  // ALWAYS set canvas dimensions BEFORE calling render()
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;

  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);

  await page.render({ canvasContext: ctx, viewport }).promise;
}
```

**Why**: If canvas dimensions are not set, the canvas defaults to 300x150 pixels. If the canvas was previously cleared by setting `width = 0`, it remains at 0x0. In both cases, PDF.js renders into the wrong-sized canvas, producing either a tiny clipped render or a completely blank output. ALWAYS set canvas dimensions from the viewport BEFORE calling `render()`.

---

## 9. Excessive Scale Without Limits (Tab Crash)

**Severity**: High -- causes tab crash on zoom.

### Wrong

```typescript
// BROKEN: User can zoom to any level, creating enormous canvases
function onZoom(page: PDFPageProxy, canvas: HTMLCanvasElement, scale: number) {
  const viewport = page.getViewport({ scale }); // scale could be 20+
  const dpr = window.devicePixelRatio || 1;

  // At scale=20 on 2x DPR: canvas could be 24000x34000 = 816M pixels
  // This WILL crash the browser
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  // ...
}
```

### Correct

```typescript
function onZoom(page: PDFPageProxy, canvas: HTMLCanvasElement, scale: number) {
  const dpr = window.devicePixelRatio || 1;

  // ALWAYS cap the effective scale to prevent oversized canvases
  const MAX_SCALE = 4;
  const effectiveScale = Math.min(scale, MAX_SCALE / dpr);

  const viewport = page.getViewport({ scale: effectiveScale });

  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;

  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);

  page.render({ canvasContext: ctx, viewport });
}
```

**Why**: Canvas dimensions are limited by browser GPU memory. Safari has particularly low limits (~67M pixels total). At high zoom levels multiplied by DPR, canvas dimensions easily exceed these limits, causing silent failures or tab crashes. ALWAYS cap the maximum effective scale (including DPR) to a safe limit, typically 3x-4x.
