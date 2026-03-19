# Page Rendering Anti-Patterns (pdfjs-dist 5.x)

Common mistakes when rendering PDF pages and how to fix them.

---

## 1. Not Cancelling Previous Render Task

**Severity**: Critical -- causes visual artifacts, race conditions, and memory leaks.

### Wrong

```typescript
import type { PDFPageProxy } from "pdfjs-dist";

// BROKEN: No cancellation of previous render
async function onZoomChange(
  page: PDFPageProxy,
  canvas: HTMLCanvasElement,
  newScale: number
): Promise<void> {
  const viewport = page.getViewport({ scale: newScale });
  const ctx = canvas.getContext("2d")!;

  // This starts a NEW render while the OLD one is still drawing.
  // Two renders fight over the same canvas pixels simultaneously.
  page.render({ canvasContext: ctx, viewport });
}
```

### Correct

```typescript
import type { PDFPageProxy, RenderTask } from "pdfjs-dist";

let currentRenderTask: RenderTask | null = null;

async function onZoomChange(
  page: PDFPageProxy,
  canvas: HTMLCanvasElement,
  newScale: number
): Promise<void> {
  // ALWAYS cancel the previous render before starting a new one
  if (currentRenderTask) {
    currentRenderTask.cancel();
    currentRenderTask = null;
  }

  const viewport = page.getViewport({ scale: newScale });
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
    if (err instanceof Error && err.message === "Rendering cancelled") {
      return;
    }
    throw err;
  } finally {
    currentRenderTask = null;
  }
}
```

**Why**: When two renders run on the same canvas, their draw operations interleave. The result is a garbled image mixing pixels from two different scales or rotations. Additionally, the uncancelled render task holds references to the canvas context, preventing garbage collection.

---

## 2. Not Handling devicePixelRatio (Blurry Rendering)

**Severity**: High -- every high-DPI display (Retina, 4K) shows blurry output.

### Wrong

```typescript
// BROKEN: Ignores devicePixelRatio entirely
async function renderPage(
  page: PDFPageProxy,
  canvas: HTMLCanvasElement
): Promise<void> {
  const viewport = page.getViewport({ scale: 1.5 });

  // Setting canvas dimensions directly from viewport produces
  // a 1x resolution canvas that the browser stretches to fill
  // the high-DPI screen, causing blur.
  canvas.width = viewport.width;
  canvas.height = viewport.height;

  const ctx = canvas.getContext("2d")!;
  await page.render({ canvasContext: ctx, viewport }).promise;
}
```

### Correct

```typescript
async function renderPage(
  page: PDFPageProxy,
  canvas: HTMLCanvasElement
): Promise<void> {
  const viewport = page.getViewport({ scale: 1.5 });
  const dpr = window.devicePixelRatio || 1;

  // ALWAYS set pixel dimensions to viewport * DPR
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);

  // ALWAYS set CSS dimensions to viewport size
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;

  // ALWAYS scale the context by DPR
  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);

  await page.render({ canvasContext: ctx, viewport }).promise;
}
```

**Why**: On a Retina display (`devicePixelRatio = 2`), the browser maps 1 CSS pixel to 4 physical pixels. Without DPI scaling, the canvas renders at 1x resolution and the browser upscales it, producing visible blur on all text and graphics.

---

## 3. Rendering All Pages at Once (Memory Explosion)

**Severity**: Critical -- causes out-of-memory crashes and UI freezing on large documents.

### Wrong

```typescript
// BROKEN: Renders every page immediately on load
async function renderAllPages(
  doc: PDFDocumentProxy,
  container: HTMLDivElement
): Promise<void> {
  // A 500-page PDF will create 500 canvases and 500 concurrent renders.
  // Each canvas at scale 1.5 on a Retina display consumes ~20-40 MB.
  // Total: 10-20 GB of memory. The browser tab WILL crash.
  for (let i = 1; i <= doc.numPages; i++) {
    const page = await doc.getPage(i);
    const viewport = page.getViewport({ scale: 1.5 });
    const canvas = document.createElement("canvas");
    const dpr = window.devicePixelRatio || 1;
    canvas.width = Math.floor(viewport.width * dpr);
    canvas.height = Math.floor(viewport.height * dpr);
    container.appendChild(canvas);

    const ctx = canvas.getContext("2d")!;
    ctx.scale(dpr, dpr);
    await page.render({ canvasContext: ctx, viewport }).promise;
  }
}
```

### Correct

```typescript
// ALWAYS use IntersectionObserver to render only visible pages
// See examples.md section 7 for the complete lazy loading implementation

async function setupLazyViewer(
  doc: PDFDocumentProxy,
  container: HTMLDivElement
): Promise<void> {
  // Create lightweight placeholder divs for all pages (cheap)
  for (let i = 1; i <= doc.numPages; i++) {
    const page = await doc.getPage(i);
    const viewport = page.getViewport({ scale: 1.5 });

    const placeholder = document.createElement("div");
    placeholder.dataset.pageNumber = String(i);
    placeholder.style.width = `${Math.floor(viewport.width)}px`;
    placeholder.style.height = `${Math.floor(viewport.height)}px`;
    placeholder.style.backgroundColor = "#f0f0f0";
    container.appendChild(placeholder);
  }

  // Only render pages as they scroll into view
  const observer = new IntersectionObserver(
    (entries) => {
      for (const entry of entries) {
        if (entry.isIntersecting) {
          renderSinglePage(doc, entry.target as HTMLDivElement);
        }
      }
    },
    { root: container, rootMargin: "200px" }
  );

  container.querySelectorAll("[data-page-number]").forEach((el) => {
    observer.observe(el);
  });
}
```

**Why**: Each rendered canvas consumes significant GPU and CPU memory. At scale 1.5 on a 2x DPR display, a single A4 page canvas uses approximately 36 MB (1782 x 2520 pixels x 4 bytes x 2 DPR = ~36 MB). Even 50 pages can consume 1.8 GB.

---

## 4. Not Awaiting render Task Promise

**Severity**: High -- leads to reading incomplete canvas data and race conditions.

### Wrong

```typescript
// BROKEN: Does not await the render promise
function renderPage(
  page: PDFPageProxy,
  canvas: HTMLCanvasElement
): void {
  const viewport = page.getViewport({ scale: 1.5 });
  const ctx = canvas.getContext("2d")!;

  // render() returns a RenderTask, NOT a resolved canvas.
  // The render is asynchronous -- the canvas is EMPTY at this point.
  page.render({ canvasContext: ctx, viewport });

  // BROKEN: Canvas content is incomplete or empty
  const imageData = canvas.toDataURL("image/png");
  console.log("Captured:", imageData); // Captures a blank or partial image
}
```

### Correct

```typescript
async function renderPage(
  page: PDFPageProxy,
  canvas: HTMLCanvasElement
): Promise<void> {
  const viewport = page.getViewport({ scale: 1.5 });
  const dpr = window.devicePixelRatio || 1;
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;

  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);

  const renderTask = page.render({ canvasContext: ctx, viewport });

  // ALWAYS await the promise before accessing canvas content
  await renderTask.promise;

  // NOW the canvas is fully rendered
  const imageData = canvas.toDataURL("image/png");
  console.log("Captured:", imageData); // Contains the complete rendered page
}
```

**Why**: `page.render()` starts an asynchronous rendering operation. The returned `RenderTask` contains a `promise` property that resolves only when all drawing operations are complete. Without awaiting it, you read a canvas that is either blank or partially drawn.

---

## 5. Wrong Canvas Dimension Setup

**Severity**: Medium -- causes distorted or clipped rendering.

### Wrong: Setting only pixel dimensions without CSS dimensions

```typescript
// BROKEN: Missing CSS dimensions
const viewport = page.getViewport({ scale: 1.5 });
const dpr = window.devicePixelRatio || 1;

canvas.width = Math.floor(viewport.width * dpr);
canvas.height = Math.floor(viewport.height * dpr);
// Missing: canvas.style.width and canvas.style.height

// Result: The canvas displays at its pixel dimensions, which are
// DPR times too large. The page appears oversized.
```

### Wrong: Using floating-point canvas dimensions

```typescript
// BROKEN: Canvas dimensions MUST be integers
const viewport = page.getViewport({ scale: 1.37 });
const dpr = window.devicePixelRatio || 1;

// viewport.width might be 893.67 -- canvas cannot handle fractional pixels
canvas.width = viewport.width * dpr;   // Fractional value!
canvas.height = viewport.height * dpr;  // Fractional value!
// Result: Sub-pixel rendering artifacts, potential visual glitches
```

### Correct

```typescript
const viewport = page.getViewport({ scale: 1.5 });
const dpr = window.devicePixelRatio || 1;

// ALWAYS use Math.floor() for integer pixel dimensions
canvas.width = Math.floor(viewport.width * dpr);
canvas.height = Math.floor(viewport.height * dpr);

// ALWAYS set both CSS dimensions for proper display sizing
canvas.style.width = `${Math.floor(viewport.width)}px`;
canvas.style.height = `${Math.floor(viewport.height)}px`;

const ctx = canvas.getContext("2d")!;
ctx.scale(dpr, dpr);
```

**Why**: Canvas `width` and `height` attributes MUST be integers -- the browser silently truncates fractional values which can cause 1-pixel misalignment. CSS dimensions control how large the canvas appears on screen and MUST be set separately from the pixel dimensions to achieve proper DPI scaling.

---

## 6. Concurrent Renders on Same Canvas

**Severity**: Critical -- produces corrupted visual output.

### Wrong

```typescript
// BROKEN: Multiple rapid calls without synchronization
async function handleRapidZoom(
  page: PDFPageProxy,
  canvas: HTMLCanvasElement,
  scales: number[]
): Promise<void> {
  // Fires multiple renders on the same canvas simultaneously
  const promises = scales.map((scale) => {
    const viewport = page.getViewport({ scale });
    const ctx = canvas.getContext("2d")!;
    return page.render({ canvasContext: ctx, viewport }).promise;
  });

  // All renders draw to the same canvas at the same time -- visual corruption
  await Promise.all(promises);
}
```

### Also Wrong: Fire-and-forget from event listeners

```typescript
// BROKEN: Each scroll event fires a new render without cancelling
document.addEventListener("scroll", () => {
  const scale = calculateZoomFromScroll();
  const viewport = page.getViewport({ scale });
  const ctx = canvas.getContext("2d")!;

  // Every scroll event starts a new render.
  // At 60fps scrolling, this creates 60 concurrent renders per second.
  page.render({ canvasContext: ctx, viewport });
});
```

### Correct: Serialize renders with cancellation

```typescript
import type { PDFPageProxy, RenderTask } from "pdfjs-dist";

let pendingRender: RenderTask | null = null;

async function serializedRender(
  page: PDFPageProxy,
  canvas: HTMLCanvasElement,
  scale: number
): Promise<void> {
  // ALWAYS cancel any in-flight render
  if (pendingRender) {
    pendingRender.cancel();
    pendingRender = null;
  }

  const viewport = page.getViewport({ scale });
  const dpr = window.devicePixelRatio || 1;
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;

  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);

  pendingRender = page.render({ canvasContext: ctx, viewport });

  try {
    await pendingRender.promise;
  } catch (err: unknown) {
    if (err instanceof Error && err.message === "Rendering cancelled") {
      return;
    }
    throw err;
  } finally {
    pendingRender = null;
  }
}

// Debounce rapid events to reduce unnecessary render cycles
let debounceTimer: ReturnType<typeof setTimeout>;

document.addEventListener("scroll", () => {
  clearTimeout(debounceTimer);
  debounceTimer = setTimeout(() => {
    const scale = calculateZoomFromScroll();
    serializedRender(page, canvas, scale);
  }, 50); // 50ms debounce
});
```

**Why**: A canvas has a single 2D context. When multiple `page.render()` calls draw to the same context simultaneously, their draw commands interleave unpredictably. The result is a garbled mix of different render states. ALWAYS ensure only one `RenderTask` is active per canvas at any time.

---

## 7. Not Cleaning Up RenderTask References

**Severity**: Medium -- causes memory leaks in long-running applications.

### Wrong

```typescript
// BROKEN: RenderTask reference is never cleared
class PageView {
  private renderTask: RenderTask | null = null;

  async render(page: PDFPageProxy, canvas: HTMLCanvasElement): Promise<void> {
    const viewport = page.getViewport({ scale: 1.5 });
    const ctx = canvas.getContext("2d")!;

    this.renderTask = page.render({ canvasContext: ctx, viewport });
    await this.renderTask.promise;
    // renderTask is never set to null after completion.
    // It holds references to the canvas context and page data.
  }
}
```

### Correct

```typescript
class PageView {
  private renderTask: RenderTask | null = null;

  async render(page: PDFPageProxy, canvas: HTMLCanvasElement): Promise<void> {
    if (this.renderTask) {
      this.renderTask.cancel();
      this.renderTask = null;
    }

    const viewport = page.getViewport({ scale: 1.5 });
    const dpr = window.devicePixelRatio || 1;
    canvas.width = Math.floor(viewport.width * dpr);
    canvas.height = Math.floor(viewport.height * dpr);
    canvas.style.width = `${Math.floor(viewport.width)}px`;
    canvas.style.height = `${Math.floor(viewport.height)}px`;

    const ctx = canvas.getContext("2d")!;
    ctx.scale(dpr, dpr);

    this.renderTask = page.render({ canvasContext: ctx, viewport });

    try {
      await this.renderTask.promise;
    } finally {
      // ALWAYS clear the reference after completion or failure
      this.renderTask = null;
    }
  }
}
```

**Why**: A completed `RenderTask` still holds internal references to the canvas context and rendering data. Setting it to `null` in a `finally` block allows garbage collection to reclaim that memory. In a viewer that renders hundreds of pages during a session, these leaks accumulate.
