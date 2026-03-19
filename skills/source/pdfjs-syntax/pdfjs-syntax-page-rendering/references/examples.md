# Page Rendering Examples (pdfjs-dist 5.x)

## 1. Complete Rendering Pipeline

The full pipeline from loading a PDF to rendering a page on screen.

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";
import type { PDFDocumentProxy, PDFPageProxy } from "pdfjs-dist";

// ALWAYS configure the worker BEFORE calling getDocument
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

async function renderPdfPage(
  url: string,
  pageNumber: number,
  container: HTMLDivElement
): Promise<void> {
  // Step 1: Load the document
  const loadingTask = getDocument({ url });
  const doc: PDFDocumentProxy = await loadingTask.promise;

  // Step 2: Get the page (1-indexed)
  const page: PDFPageProxy = await doc.getPage(pageNumber);

  // Step 3: Create the viewport
  const scale = 1.5;
  const viewport = page.getViewport({ scale });

  // Step 4: Create and size the canvas with DPI scaling
  const canvas = document.createElement("canvas");
  container.appendChild(canvas);

  const dpr = window.devicePixelRatio || 1;
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;

  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);

  // Step 5: Render the page
  const renderTask = page.render({
    canvasContext: ctx,
    viewport: viewport,
  });

  // Step 6: Await completion
  await renderTask.promise;

  // Canvas now contains the rendered page
}
```

---

## 2. Basic Page Rendering to Canvas

Minimal example for rendering a single page when you already have a canvas element.

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";

GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

async function renderPage(
  url: string,
  canvas: HTMLCanvasElement
): Promise<void> {
  const doc = await getDocument(url).promise;
  const page = await doc.getPage(1);
  const viewport = page.getViewport({ scale: 1.0 });

  // ALWAYS apply devicePixelRatio for sharp rendering
  const dpr = window.devicePixelRatio || 1;
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;

  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);

  await page.render({ canvasContext: ctx, viewport }).promise;
}
```

---

## 3. High-DPI Rendering with devicePixelRatio

Dedicated utility function for DPI-aware canvas setup. Reusable across any rendering scenario.

```typescript
import type { PageViewport } from "pdfjs-dist";

/**
 * Sets up a canvas for high-DPI rendering.
 * ALWAYS call this before page.render() to avoid blurry output.
 *
 * @returns The configured 2D context, already scaled for DPI.
 */
function setupHighDpiCanvas(
  canvas: HTMLCanvasElement,
  viewport: PageViewport
): CanvasRenderingContext2D {
  const dpr = window.devicePixelRatio || 1;

  // Pixel dimensions control actual resolution
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);

  // CSS dimensions control display size on screen
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;

  // Scale the context so all drawing operations use CSS pixel units
  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);

  return ctx;
}

// Usage:
async function renderWithHighDpi(
  page: PDFPageProxy,
  canvas: HTMLCanvasElement,
  scale: number
): Promise<void> {
  const viewport = page.getViewport({ scale });
  const ctx = setupHighDpiCanvas(canvas, viewport);
  await page.render({ canvasContext: ctx, viewport }).promise;
}
```

---

## 4. Cancel-and-Re-render Pattern (Zoom / Rotation Changes)

ALWAYS cancel the previous render before starting a new one. This pattern handles zoom, rotation, and any viewport change.

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";
import type { PDFPageProxy, RenderTask, PageViewport } from "pdfjs-dist";

GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

class PageRenderer {
  private currentRenderTask: RenderTask | null = null;
  private page: PDFPageProxy;
  private canvas: HTMLCanvasElement;

  constructor(page: PDFPageProxy, canvas: HTMLCanvasElement) {
    this.page = page;
    this.canvas = canvas;
  }

  async render(scale: number, rotation: number = 0): Promise<void> {
    // ALWAYS cancel any in-progress render first
    if (this.currentRenderTask) {
      this.currentRenderTask.cancel();
      this.currentRenderTask = null;
    }

    const viewport = this.page.getViewport({ scale, rotation });

    // Setup canvas with DPI handling
    const dpr = window.devicePixelRatio || 1;
    this.canvas.width = Math.floor(viewport.width * dpr);
    this.canvas.height = Math.floor(viewport.height * dpr);
    this.canvas.style.width = `${Math.floor(viewport.width)}px`;
    this.canvas.style.height = `${Math.floor(viewport.height)}px`;

    const ctx = this.canvas.getContext("2d")!;
    ctx.scale(dpr, dpr);

    this.currentRenderTask = this.page.render({
      canvasContext: ctx,
      viewport: viewport,
    });

    try {
      await this.currentRenderTask.promise;
    } catch (err: unknown) {
      // Cancellation causes a rejection -- this is expected behavior
      if (err instanceof Error && err.message === "Rendering cancelled") {
        return; // Not an error, just a cancelled render
      }
      throw err; // Re-throw actual rendering errors
    } finally {
      this.currentRenderTask = null;
    }
  }
}

// Usage:
// const renderer = new PageRenderer(page, canvas);
// await renderer.render(1.5);         // Initial render
// await renderer.render(2.0);         // Zoom in -- cancels previous if still running
// await renderer.render(1.5, 90);     // Rotate -- cancels previous if still running
```

---

## 5. Layer Stacking Setup (Canvas + TextLayer + AnnotationLayer)

ALWAYS stack layers in this order: canvas (bottom), text layer (middle), annotation layer (top). Each layer MUST be absolutely positioned within the same container.

```typescript
import { getDocument, GlobalWorkerOptions, AnnotationMode } from "pdfjs-dist";
import { TextLayer, AnnotationLayer } from "pdfjs-dist";
import type { PDFPageProxy, PageViewport } from "pdfjs-dist";

GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

async function renderPageWithLayers(
  page: PDFPageProxy,
  container: HTMLDivElement,
  scale: number = 1.5
): Promise<void> {
  const viewport = page.getViewport({ scale });
  const dpr = window.devicePixelRatio || 1;

  // Clear previous content
  container.innerHTML = "";

  // Container MUST be positioned relative for absolute children
  container.style.position = "relative";
  container.style.width = `${Math.floor(viewport.width)}px`;
  container.style.height = `${Math.floor(viewport.height)}px`;

  // --- Layer 1: Canvas (rendered PDF pixels) ---
  const canvas = document.createElement("canvas");
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;
  canvas.style.position = "absolute";
  canvas.style.top = "0";
  canvas.style.left = "0";
  container.appendChild(canvas);

  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);

  // Render the page pixels
  await page.render({ canvasContext: ctx, viewport }).promise;

  // --- Layer 2: Text layer (selectable/searchable text) ---
  const textLayerDiv = document.createElement("div");
  textLayerDiv.className = "textLayer";
  textLayerDiv.style.position = "absolute";
  textLayerDiv.style.top = "0";
  textLayerDiv.style.left = "0";
  textLayerDiv.style.width = `${Math.floor(viewport.width)}px`;
  textLayerDiv.style.height = `${Math.floor(viewport.height)}px`;
  container.appendChild(textLayerDiv);

  const textContent = await page.getTextContent();
  const textLayer = new TextLayer({
    container: textLayerDiv,
    textContentSource: textContent,
    viewport: viewport,
  });
  await textLayer.render();

  // --- Layer 3: Annotation layer (links, forms) ---
  const annotationLayerDiv = document.createElement("div");
  annotationLayerDiv.className = "annotationLayer";
  annotationLayerDiv.style.position = "absolute";
  annotationLayerDiv.style.top = "0";
  annotationLayerDiv.style.left = "0";
  annotationLayerDiv.style.width = `${Math.floor(viewport.width)}px`;
  annotationLayerDiv.style.height = `${Math.floor(viewport.height)}px`;
  container.appendChild(annotationLayerDiv);

  const annotations = await page.getAnnotations();
  const annotationLayer = new AnnotationLayer({
    div: annotationLayerDiv,
    annotations: annotations,
    page: page,
    viewport: viewport,
  });
  await annotationLayer.render({ viewport, annotations });
}
```

**Required CSS** (ALWAYS include the PDF.js text layer stylesheet):

```css
/* Import the official PDF.js text layer styles */
@import "pdfjs-dist/web/pdf_viewer.css";

/* Ensure correct stacking order */
.textLayer {
  z-index: 1;
}

.annotationLayer {
  z-index: 2;
}
```

---

## 6. OffscreenCanvas Rendering

Use `OffscreenCanvas` for rendering in a Web Worker or for off-screen thumbnail generation.

```typescript
import type { PDFPageProxy } from "pdfjs-dist";

/**
 * Renders a page to an OffscreenCanvas and returns an ImageBitmap.
 * Use this for Web Worker rendering or generating thumbnails without DOM access.
 *
 * NEVER use window.devicePixelRatio inside a Web Worker -- it is not available.
 * ALWAYS pass the DPR from the main thread if high-DPI output is needed.
 */
async function renderToOffscreenCanvas(
  page: PDFPageProxy,
  scale: number,
  dpr: number = 1
): Promise<ImageBitmap> {
  const viewport = page.getViewport({ scale });

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

// Main thread usage -- transfer result to a visible canvas:
async function renderViaOffscreen(
  page: PDFPageProxy,
  canvas: HTMLCanvasElement,
  scale: number
): Promise<void> {
  const dpr = window.devicePixelRatio || 1;
  const bitmap = await renderToOffscreenCanvas(page, scale, dpr);

  const viewport = page.getViewport({ scale });
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;

  const ctx = canvas.getContext("bitmaprenderer")!;
  ctx.transferFromImageBitmap(bitmap);
}
```

---

## 7. Lazy Loading with IntersectionObserver

NEVER render all pages at once. Use `IntersectionObserver` to render only visible pages.

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";
import type { PDFDocumentProxy, PDFPageProxy, RenderTask } from "pdfjs-dist";

GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

async function setupLazyPdfViewer(
  url: string,
  container: HTMLDivElement
): Promise<void> {
  const doc = await getDocument(url).promise;
  const renderTasks = new Map<number, RenderTask>();

  // Create placeholder containers for all pages
  for (let i = 1; i <= doc.numPages; i++) {
    const page = await doc.getPage(i);
    const viewport = page.getViewport({ scale: 1.5 });

    const pageDiv = document.createElement("div");
    pageDiv.dataset.pageNumber = String(i);
    pageDiv.style.width = `${Math.floor(viewport.width)}px`;
    pageDiv.style.height = `${Math.floor(viewport.height)}px`;
    pageDiv.style.position = "relative";
    pageDiv.style.marginBottom = "10px";
    pageDiv.style.backgroundColor = "#e0e0e0"; // Placeholder color
    container.appendChild(pageDiv);
  }

  // Observe which pages are visible
  const observer = new IntersectionObserver(
    (entries) => {
      for (const entry of entries) {
        const pageNumber = Number(
          (entry.target as HTMLElement).dataset.pageNumber
        );

        if (entry.isIntersecting) {
          renderVisiblePage(doc, entry.target as HTMLDivElement, pageNumber, renderTasks);
        } else {
          // Cancel render if page scrolls out of view
          const task = renderTasks.get(pageNumber);
          if (task) {
            task.cancel();
            renderTasks.delete(pageNumber);
          }
        }
      }
    },
    {
      root: container,
      rootMargin: "200px", // Pre-render pages 200px before they become visible
    }
  );

  // Observe all page placeholders
  container.querySelectorAll("[data-page-number]").forEach((el) => {
    observer.observe(el);
  });
}

async function renderVisiblePage(
  doc: PDFDocumentProxy,
  pageDiv: HTMLDivElement,
  pageNumber: number,
  renderTasks: Map<number, RenderTask>
): Promise<void> {
  // Skip if already rendered
  if (pageDiv.querySelector("canvas")) return;

  const page = await doc.getPage(pageNumber);
  const viewport = page.getViewport({ scale: 1.5 });
  const dpr = window.devicePixelRatio || 1;

  const canvas = document.createElement("canvas");
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;
  pageDiv.appendChild(canvas);

  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);

  const renderTask = page.render({ canvasContext: ctx, viewport });
  renderTasks.set(pageNumber, renderTask);

  try {
    await renderTask.promise;
  } catch (err: unknown) {
    if (err instanceof Error && err.message === "Rendering cancelled") {
      // Remove the canvas if render was cancelled (page scrolled out of view)
      canvas.remove();
      return;
    }
    throw err;
  } finally {
    renderTasks.delete(pageNumber);
  }
}
```

---

## 8. Fit-to-Width Rendering

Calculate the scale dynamically based on container width.

```typescript
import type { PDFPageProxy } from "pdfjs-dist";

function calculateFitToWidthScale(
  page: PDFPageProxy,
  containerWidth: number
): number {
  // Get the page dimensions at scale 1.0
  const baseViewport = page.getViewport({ scale: 1.0 });
  return containerWidth / baseViewport.width;
}

async function renderFitToWidth(
  page: PDFPageProxy,
  canvas: HTMLCanvasElement,
  containerWidth: number
): Promise<void> {
  const scale = calculateFitToWidthScale(page, containerWidth);
  const viewport = page.getViewport({ scale });

  const dpr = window.devicePixelRatio || 1;
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;

  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);

  await page.render({ canvasContext: ctx, viewport }).promise;
}
```
