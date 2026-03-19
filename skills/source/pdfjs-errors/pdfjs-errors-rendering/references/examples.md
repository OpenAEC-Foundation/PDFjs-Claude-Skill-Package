# Rendering Error Recovery Patterns (pdfjs-dist 5.x)

## 1. DPR-Aware Rendering with Fallback

Handle `devicePixelRatio` correctly across different display types, including dynamic DPR changes (e.g., moving a window between monitors).

```typescript
import type { PDFPageProxy, RenderTask } from "pdfjs-dist";

function renderPageDPRAware(
  page: PDFPageProxy,
  canvas: HTMLCanvasElement,
  scale: number = 1.5
): RenderTask {
  const viewport = page.getViewport({ scale });
  const dpr = window.devicePixelRatio || 1;

  // Physical pixels for sharp rendering
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);

  // CSS pixels for correct layout
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;

  const ctx = canvas.getContext("2d");
  if (!ctx) {
    throw new Error(
      "Cannot get 2D context. Canvas may have a WebGL context or be detached."
    );
  }
  ctx.scale(dpr, dpr);

  return page.render({ canvasContext: ctx, viewport });
}

// Re-render when DPR changes (user moves window to different monitor)
function watchDPRChanges(rerender: () => void): () => void {
  let currentDPR = window.devicePixelRatio;

  const mediaQuery = window.matchMedia(
    `(resolution: ${currentDPR}dppx)`
  );

  function handleChange(): void {
    if (window.devicePixelRatio !== currentDPR) {
      currentDPR = window.devicePixelRatio;
      rerender();
    }
    // Re-register because resolution media query is one-shot
    const newQuery = window.matchMedia(
      `(resolution: ${currentDPR}dppx)`
    );
    newQuery.addEventListener("change", handleChange, { once: true });
  }

  mediaQuery.addEventListener("change", handleChange, { once: true });

  // Return cleanup function
  return () => mediaQuery.removeEventListener("change", handleChange);
}
```

---

## 2. Cancel-Then-Render Pattern (Full Implementation)

Complete implementation of the cancel-before-render pattern for zoom/scroll scenarios.

```typescript
import type { PDFPageProxy, RenderTask } from "pdfjs-dist";

class SafePageRenderer {
  private currentTask: RenderTask | null = null;
  private rendering = false;

  async render(
    page: PDFPageProxy,
    canvas: HTMLCanvasElement,
    scale: number
  ): Promise<boolean> {
    // Step 1: Cancel any in-progress render
    if (this.currentTask) {
      this.currentTask.cancel();
      this.currentTask = null;
    }

    // Step 2: Wait for previous render to fully complete cancellation
    // This prevents race conditions where cancel() is async
    if (this.rendering) {
      await new Promise<void>((resolve) => setTimeout(resolve, 0));
    }

    this.rendering = true;

    // Step 3: Set up canvas with DPR scaling
    const viewport = page.getViewport({ scale });
    const dpr = window.devicePixelRatio || 1;

    canvas.width = Math.floor(viewport.width * dpr);
    canvas.height = Math.floor(viewport.height * dpr);
    canvas.style.width = `${Math.floor(viewport.width)}px`;
    canvas.style.height = `${Math.floor(viewport.height)}px`;

    const ctx = canvas.getContext("2d");
    if (!ctx) {
      this.rendering = false;
      return false;
    }
    ctx.scale(dpr, dpr);

    // Step 4: Start new render
    this.currentTask = page.render({ canvasContext: ctx, viewport });

    try {
      await this.currentTask.promise;
      return true; // Render completed successfully
    } catch (err: unknown) {
      if (err instanceof Error && err.name === "RenderingCancelledException") {
        return false; // Cancelled intentionally
      }
      throw err; // Unexpected error
    } finally {
      this.currentTask = null;
      this.rendering = false;
    }
  }

  cancel(): void {
    if (this.currentTask) {
      this.currentTask.cancel();
      this.currentTask = null;
    }
  }
}
```

---

## 3. Lazy Page Rendering with IntersectionObserver

Render only visible pages using IntersectionObserver for efficient memory usage.

```typescript
import type { PDFDocumentProxy, PDFPageProxy, RenderTask } from "pdfjs-dist";

class LazyPDFRenderer {
  private doc: PDFDocumentProxy;
  private scale: number;
  private renderTasks = new Map<number, RenderTask>();
  private renderedPages = new Set<number>();
  private observer: IntersectionObserver;

  constructor(doc: PDFDocumentProxy, scale: number = 1.5) {
    this.doc = doc;
    this.scale = scale;

    this.observer = new IntersectionObserver(
      (entries) => this.handleIntersection(entries),
      {
        root: null,
        rootMargin: "200px 0px", // Pre-render pages 200px before they become visible
        threshold: 0,
      }
    );
  }

  observe(pageNum: number, canvas: HTMLCanvasElement): void {
    canvas.dataset.pageNum = String(pageNum);
    this.observer.observe(canvas);
  }

  private async handleIntersection(
    entries: IntersectionObserverEntry[]
  ): Promise<void> {
    for (const entry of entries) {
      const canvas = entry.target as HTMLCanvasElement;
      const pageNum = Number(canvas.dataset.pageNum);

      if (entry.isIntersecting && !this.renderedPages.has(pageNum)) {
        await this.renderPage(pageNum, canvas);
      } else if (!entry.isIntersecting && this.renderedPages.has(pageNum)) {
        this.releasePage(pageNum, canvas);
      }
    }
  }

  private async renderPage(
    pageNum: number,
    canvas: HTMLCanvasElement
  ): Promise<void> {
    // Cancel if already rendering this page
    const existingTask = this.renderTasks.get(pageNum);
    if (existingTask) {
      existingTask.cancel();
    }

    const page = await this.doc.getPage(pageNum);
    const viewport = page.getViewport({ scale: this.scale });
    const dpr = window.devicePixelRatio || 1;

    canvas.width = Math.floor(viewport.width * dpr);
    canvas.height = Math.floor(viewport.height * dpr);
    canvas.style.width = `${Math.floor(viewport.width)}px`;
    canvas.style.height = `${Math.floor(viewport.height)}px`;

    const ctx = canvas.getContext("2d");
    if (!ctx) return;
    ctx.scale(dpr, dpr);

    const task = page.render({ canvasContext: ctx, viewport });
    this.renderTasks.set(pageNum, task);

    try {
      await task.promise;
      this.renderedPages.add(pageNum);
    } catch (err: unknown) {
      if (err instanceof Error && err.name === "RenderingCancelledException") {
        return;
      }
      console.error(`Failed to render page ${pageNum}:`, err);
    } finally {
      this.renderTasks.delete(pageNum);
    }
  }

  private releasePage(pageNum: number, canvas: HTMLCanvasElement): void {
    // Free GPU memory by zeroing canvas dimensions
    canvas.width = 0;
    canvas.height = 0;
    this.renderedPages.delete(pageNum);
  }

  destroy(): void {
    this.observer.disconnect();
    for (const task of this.renderTasks.values()) {
      task.cancel();
    }
    this.renderTasks.clear();
    this.renderedPages.clear();
  }
}
```

---

## 4. Text Layer Alignment (Complete Pattern)

Properly align the text selection layer with the rendered canvas content.

```typescript
import { renderTextLayer } from "pdfjs-dist";
import type { PDFPageProxy, TextContent } from "pdfjs-dist";

async function renderWithTextLayer(
  page: PDFPageProxy,
  container: HTMLElement,
  scale: number = 1.5
): Promise<void> {
  // IMPORTANT: Use ONE viewport for everything
  const viewport = page.getViewport({ scale });
  const dpr = window.devicePixelRatio || 1;

  // Clear container
  container.innerHTML = "";
  container.style.position = "relative";
  container.style.width = `${Math.floor(viewport.width)}px`;
  container.style.height = `${Math.floor(viewport.height)}px`;

  // Canvas layer (bottom)
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

  // Render canvas
  await page.render({ canvasContext: ctx, viewport }).promise;

  // Text layer (on top of canvas)
  const textLayerDiv = document.createElement("div");
  textLayerDiv.className = "textLayer";
  textLayerDiv.style.position = "absolute";
  textLayerDiv.style.top = "0";
  textLayerDiv.style.left = "0";
  textLayerDiv.style.width = `${Math.floor(viewport.width)}px`;
  textLayerDiv.style.height = `${Math.floor(viewport.height)}px`;
  container.appendChild(textLayerDiv);

  // Get text content and render text layer
  const textContent: TextContent = await page.getTextContent();

  await renderTextLayer({
    textContentSource: textContent,
    container: textLayerDiv,
    viewport: viewport, // SAME viewport as canvas -- NEVER create a new one
  }).promise;
}
```

### Required CSS for Text Layer

```css
/*
 * ALWAYS import the PDF.js text layer CSS.
 * Without this, text spans will not be positioned correctly.
 *
 * Import from: node_modules/pdfjs-dist/web/pdf_viewer.css
 * Or use these minimal required styles:
 */

.textLayer {
  position: absolute;
  text-align: initial;
  inset: 0;
  overflow: hidden;
  opacity: 1;
  line-height: 1;
  -webkit-text-size-adjust: none;
  text-size-adjust: none;
  forced-color-adjust: none;
  z-index: 2; /* Above canvas (z-index: 1) */
}

.textLayer span,
.textLayer br {
  color: transparent;
  position: absolute;
  white-space: pre;
  cursor: text;
  transform-origin: 0% 0%;
}

.textLayer ::selection {
  background: rgba(0, 0, 255, 0.25);
}
```

---

## 5. Annotation Layer Stacking (Complete Pattern)

Render clickable annotations (links, form fields) above both canvas and text layers.

```typescript
import { AnnotationLayer } from "pdfjs-dist";
import type { PDFPageProxy } from "pdfjs-dist";

async function renderWithAnnotations(
  page: PDFPageProxy,
  container: HTMLElement,
  scale: number = 1.5
): Promise<void> {
  const viewport = page.getViewport({ scale });
  const dpr = window.devicePixelRatio || 1;

  container.innerHTML = "";
  container.style.position = "relative";
  container.style.width = `${Math.floor(viewport.width)}px`;
  container.style.height = `${Math.floor(viewport.height)}px`;

  // Layer 1: Canvas (bottom, z-index: 1)
  const canvas = document.createElement("canvas");
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.cssText = `
    position: absolute; top: 0; left: 0;
    width: ${Math.floor(viewport.width)}px;
    height: ${Math.floor(viewport.height)}px;
    z-index: 1;
  `;
  container.appendChild(canvas);

  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);
  await page.render({ canvasContext: ctx, viewport }).promise;

  // Layer 2: Text layer (middle, z-index: 2)
  // (see example 4 above for text layer code)

  // Layer 3: Annotation layer (top, z-index: 3)
  const annotations = await page.getAnnotations();
  if (annotations.length === 0) return;

  const annotationLayerDiv = document.createElement("div");
  annotationLayerDiv.className = "annotationLayer";
  annotationLayerDiv.style.cssText = `
    position: absolute; top: 0; left: 0;
    width: ${Math.floor(viewport.width)}px;
    height: ${Math.floor(viewport.height)}px;
    z-index: 3;
    pointer-events: auto;
  `;
  container.appendChild(annotationLayerDiv);

  AnnotationLayer.render({
    annotations: annotations,
    div: annotationLayerDiv,
    viewport: viewport, // SAME viewport as canvas
    page: page,
  });
}
```

### Required CSS for Annotation Layer

```css
/*
 * Minimal annotation layer styles to ensure interactivity.
 * Import full styles from: node_modules/pdfjs-dist/web/pdf_viewer.css
 */

.annotationLayer {
  position: absolute;
  inset: 0;
  z-index: 3; /* MUST be above textLayer (2) and canvas (1) */
  pointer-events: auto; /* NEVER set to none */
}

.annotationLayer a {
  position: absolute;
  display: block;
}

.annotationLayer a:hover {
  opacity: 0.2;
  background-color: rgba(255, 255, 0, 0.4);
}
```

---

## 6. Canvas Size Validation Before Render

Prevent silent failures from oversized canvases.

```typescript
import type { PDFPageProxy } from "pdfjs-dist";

const CANVAS_LIMITS = {
  maxArea: 16_777_216,  // 16M pixels -- safe for all browsers including Safari
  maxDimension: 16_384, // 16K pixels -- conservative across browsers
};

function getMaxSafeScale(page: PDFPageProxy, dpr: number): number {
  const unscaled = page.getViewport({ scale: 1.0 });
  const maxScaleByWidth = CANVAS_LIMITS.maxDimension / (unscaled.width * dpr);
  const maxScaleByHeight = CANVAS_LIMITS.maxDimension / (unscaled.height * dpr);
  const maxScaleByArea = Math.sqrt(
    CANVAS_LIMITS.maxArea / (unscaled.width * unscaled.height * dpr * dpr)
  );

  return Math.min(maxScaleByWidth, maxScaleByHeight, maxScaleByArea);
}

function renderPageSafe(
  page: PDFPageProxy,
  canvas: HTMLCanvasElement,
  requestedScale: number
): ReturnType<PDFPageProxy["render"]> {
  const dpr = window.devicePixelRatio || 1;
  const maxScale = getMaxSafeScale(page, dpr);

  // Clamp scale to prevent oversized canvas
  const scale = Math.min(requestedScale, maxScale);

  if (scale < requestedScale) {
    console.warn(
      `Requested scale ${requestedScale} exceeds canvas limits. ` +
      `Clamped to ${scale.toFixed(2)}.`
    );
  }

  const viewport = page.getViewport({ scale });

  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;

  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);

  return page.render({ canvasContext: ctx, viewport });
}
```

---

## 7. White/Blank Page Debugging

Diagnostic helper to identify why a page renders blank.

```typescript
import type { PDFPageProxy } from "pdfjs-dist";

interface RenderDiagnostics {
  canvasInDOM: boolean;
  canvasWidth: number;
  canvasHeight: number;
  contextAvailable: boolean;
  viewportWidth: number;
  viewportHeight: number;
  dpr: number;
  issues: string[];
}

function diagnoseBlankPage(
  page: PDFPageProxy,
  canvas: HTMLCanvasElement,
  scale: number
): RenderDiagnostics {
  const viewport = page.getViewport({ scale });
  const dpr = window.devicePixelRatio || 1;
  const ctx = canvas.getContext("2d");
  const issues: string[] = [];

  if (!document.body.contains(canvas)) {
    issues.push("Canvas is not attached to the DOM");
  }
  if (canvas.width === 0 || canvas.height === 0) {
    issues.push(
      `Canvas dimensions are zero (width: ${canvas.width}, height: ${canvas.height}). ` +
      "Set width/height BEFORE calling render()."
    );
  }
  if (!ctx) {
    issues.push(
      "Cannot get 2D context. Canvas may already have a WebGL context or be detached."
    );
  }
  if (viewport.width === 0 || viewport.height === 0) {
    issues.push("Viewport dimensions are zero. Check the scale parameter.");
  }
  if (scale <= 0) {
    issues.push(`Invalid scale: ${scale}. Scale must be a positive number.`);
  }

  return {
    canvasInDOM: document.body.contains(canvas),
    canvasWidth: canvas.width,
    canvasHeight: canvas.height,
    contextAvailable: ctx !== null,
    viewportWidth: viewport.width,
    viewportHeight: viewport.height,
    dpr,
    issues,
  };
}
```

---

## 8. React Component with Full Error Recovery

Complete React component demonstrating all rendering error prevention patterns.

```typescript
import { useEffect, useRef, useCallback } from "react";
import type { PDFPageProxy, RenderTask } from "pdfjs-dist";

interface PDFPageProps {
  page: PDFPageProxy;
  scale: number;
}

function PDFPage({ page, scale }: PDFPageProps) {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const renderTaskRef = useRef<RenderTask | null>(null);

  const renderPage = useCallback(async () => {
    const canvas = canvasRef.current;
    if (!canvas) return;

    // ALWAYS cancel previous render first
    if (renderTaskRef.current) {
      renderTaskRef.current.cancel();
      renderTaskRef.current = null;
    }

    const viewport = page.getViewport({ scale });
    const dpr = window.devicePixelRatio || 1;

    // ALWAYS set dimensions before render
    canvas.width = Math.floor(viewport.width * dpr);
    canvas.height = Math.floor(viewport.height * dpr);
    canvas.style.width = `${Math.floor(viewport.width)}px`;
    canvas.style.height = `${Math.floor(viewport.height)}px`;

    const ctx = canvas.getContext("2d");
    if (!ctx) return;
    ctx.scale(dpr, dpr);

    renderTaskRef.current = page.render({ canvasContext: ctx, viewport });

    try {
      await renderTaskRef.current.promise;
    } catch (err: unknown) {
      if (err instanceof Error && err.name === "RenderingCancelledException") {
        return; // Expected on re-render or unmount
      }
      console.error("PDF render failed:", err);
    } finally {
      renderTaskRef.current = null;
    }
  }, [page, scale]);

  useEffect(() => {
    renderPage();

    // Cleanup: cancel render on unmount or re-render
    return () => {
      if (renderTaskRef.current) {
        renderTaskRef.current.cancel();
        renderTaskRef.current = null;
      }
    };
  }, [renderPage]);

  return <canvas ref={canvasRef} />;
}
```
