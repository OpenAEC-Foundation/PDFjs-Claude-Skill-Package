# Viewer Patterns Reference (pdfjs-dist 5.x Custom Viewer)

## IntersectionObserver for Lazy Page Rendering

### Observer Setup

```typescript
const observer = new IntersectionObserver(callback, options);
```

### Options for PDF Viewer

```typescript
const observerOptions: IntersectionObserverInit = {
  // The scrollable container holding all page slots
  // ALWAYS set this to the scroll container, NOT the document root
  root: document.getElementById("page-container"),

  // Pre-render buffer -- pages start rendering before becoming visible
  // "200px 0px" means 200px above and below the viewport
  // Increase for faster scrolling users, decrease for memory-constrained environments
  rootMargin: "200px 0px",

  // Threshold for triggering the callback
  // 0 = trigger as soon as any pixel enters the root
  // Use [0, 0.5, 1.0] if you need to track partial visibility
  threshold: 0,
};
```

### Callback Pattern for Page Rendering

```typescript
const observerCallback: IntersectionObserverCallback = (entries) => {
  for (const entry of entries) {
    const slot = entry.target as HTMLElement;
    const pageNum = Number(slot.dataset.pageNumber);

    if (entry.isIntersecting) {
      // Page entered the viewport (or buffer zone)
      // ALWAYS check if already rendered to avoid duplicate renders
      if (!renderedPages.has(pageNum)) {
        renderPage(state, slot as HTMLDivElement, pageNum);
      }
    } else {
      // Page left the viewport AND buffer zone
      // ALWAYS clean up to free memory
      cleanupPage(pageNum, slot as HTMLDivElement);
    }
  }
};
```

### Observer Lifecycle

```typescript
// Create and start observing
const observer = new IntersectionObserver(observerCallback, observerOptions);
container.querySelectorAll(".page-slot").forEach((el) => observer.observe(el));

// Stop observing a specific element (e.g., when removing a page)
observer.unobserve(element);

// Stop observing ALL elements (e.g., when destroying the viewer)
observer.disconnect();
```

---

## Page Render Lifecycle

### Step 1: Render Page with All Layers

```typescript
import { TextLayer } from "pdfjs-dist";
import type { PDFDocumentProxy, RenderTask } from "pdfjs-dist";

async function renderPage(
  state: ViewerState,
  slot: HTMLDivElement,
  pageNum: number
): Promise<void> {
  // Prevent duplicate renders
  if (renderedPages.has(pageNum)) return;
  renderedPages.add(pageNum);

  const page = await state.doc.getPage(pageNum);
  const viewport = page.getViewport({ scale: state.scale });
  const dpr = window.devicePixelRatio || 1;

  // --- Canvas layer ---
  const canvas = document.createElement("canvas");
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;
  canvas.style.position = "absolute";
  canvas.style.top = "0";
  canvas.style.left = "0";
  slot.appendChild(canvas);

  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);

  const renderTask = page.render({ canvasContext: ctx, viewport });
  activeRenders.set(pageNum, renderTask);

  try {
    await renderTask.promise;
  } catch (err: unknown) {
    if (err instanceof Error && err.message === "Rendering cancelled") {
      canvas.remove();
      renderedPages.delete(pageNum);
      return;
    }
    throw err;
  } finally {
    activeRenders.delete(pageNum);
  }

  // --- Text layer (for selection and search) ---
  const textLayerDiv = document.createElement("div");
  textLayerDiv.className = "textLayer";
  textLayerDiv.style.position = "absolute";
  textLayerDiv.style.top = "0";
  textLayerDiv.style.left = "0";
  textLayerDiv.style.width = `${Math.floor(viewport.width)}px`;
  textLayerDiv.style.height = `${Math.floor(viewport.height)}px`;
  slot.appendChild(textLayerDiv);

  const textContent = await page.getTextContent();
  const textLayer = new TextLayer({
    container: textLayerDiv,
    textContentSource: textContent,
    viewport: viewport,
  });
  await textLayer.render();
}
```

### Step 2: Cleanup Page

```typescript
function cleanupPage(pageNum: number, slot: HTMLDivElement): void {
  // 1. Cancel any in-progress render
  const task = activeRenders.get(pageNum);
  if (task) {
    task.cancel();
    activeRenders.delete(pageNum);
  }

  // 2. Release canvas GPU memory
  const canvas = slot.querySelector("canvas");
  if (canvas) {
    const ctx = canvas.getContext("2d");
    if (ctx) ctx.clearRect(0, 0, canvas.width, canvas.height);
    canvas.width = 0;  // CRITICAL: setting to 0 releases GPU buffer
    canvas.height = 0;
    canvas.remove();
  }

  // 3. Remove overlay layers
  slot.querySelectorAll(".textLayer, .annotationLayer").forEach((el) => el.remove());

  // 4. Mark as not rendered
  renderedPages.delete(pageNum);
}
```

---

## Page Slot Creation Strategies

### Strategy 1: Query All Pages (accurate, slower for large docs)

```typescript
// ALWAYS use this for documents under 100 pages
async function createSlots(
  doc: PDFDocumentProxy,
  container: HTMLElement,
  scale: number
): Promise<void> {
  for (let i = 1; i <= doc.numPages; i++) {
    const page = await doc.getPage(i);
    const viewport = page.getViewport({ scale });
    createSlotElement(container, i, viewport.width, viewport.height);
  }
}
```

### Strategy 2: Assume Uniform Size (fast, approximate)

```typescript
// Use this for documents over 100 pages where speed matters
// ALWAYS get at least the first page for accurate dimensions
async function createSlotsUniform(
  doc: PDFDocumentProxy,
  container: HTMLElement,
  scale: number
): Promise<void> {
  const firstPage = await doc.getPage(1);
  const viewport = firstPage.getViewport({ scale });

  for (let i = 1; i <= doc.numPages; i++) {
    createSlotElement(container, i, viewport.width, viewport.height);
  }
}
```

### Slot Element Helper

```typescript
function createSlotElement(
  container: HTMLElement,
  pageNum: number,
  width: number,
  height: number
): HTMLDivElement {
  const slot = document.createElement("div");
  slot.className = "page-slot";
  slot.dataset.pageNumber = String(pageNum);
  slot.style.width = `${Math.floor(width)}px`;
  slot.style.height = `${Math.floor(height)}px`;
  slot.style.position = "relative";
  slot.style.marginBottom = "8px";
  slot.style.backgroundColor = "#e0e0e0";
  slot.style.willChange = "transform";
  container.appendChild(slot);
  return slot;
}
```

---

## Scale Calculation Methods

### Fit-to-Width

```typescript
function calculateFitToWidth(
  page: PDFPageProxy,
  containerWidth: number
): number {
  const baseViewport = page.getViewport({ scale: 1.0 });
  return containerWidth / baseViewport.width;
}
```

### Fit-to-Page

```typescript
function calculateFitToPage(
  page: PDFPageProxy,
  containerWidth: number,
  containerHeight: number
): number {
  const baseViewport = page.getViewport({ scale: 1.0 });
  const scaleW = containerWidth / baseViewport.width;
  const scaleH = containerHeight / baseViewport.height;
  return Math.min(scaleW, scaleH);
}
```

### Scale Clamping

```typescript
// ALWAYS clamp scale to prevent extreme values
function clampScale(scale: number): number {
  const MIN_SCALE = 0.1;
  const MAX_SCALE = 10.0;
  return Math.max(MIN_SCALE, Math.min(MAX_SCALE, scale));
}
```

---

## Scroll Position Utilities

### Get Current Page from Scroll Position

```typescript
function getCurrentPageFromScroll(container: HTMLElement): number {
  const containerRect = container.getBoundingClientRect();
  const centerY = containerRect.top + containerRect.height / 2;
  const slots = container.querySelectorAll(".page-slot");

  for (const slot of slots) {
    const rect = slot.getBoundingClientRect();
    if (rect.top <= centerY && rect.bottom >= centerY) {
      return Number((slot as HTMLElement).dataset.pageNumber);
    }
  }
  return 1; // Fallback to first page
}
```

### Scroll to Page

```typescript
function scrollToPage(
  container: HTMLElement,
  pageNum: number,
  behavior: ScrollBehavior = "smooth"
): void {
  const slot = container.querySelector(`[data-page-number="${pageNum}"]`);
  if (slot) {
    slot.scrollIntoView({ behavior, block: "start" });
  }
}
```

### Preserve Scroll Position During Zoom

```typescript
function zoomPreservingPosition(
  state: ViewerState,
  container: HTMLElement,
  newScale: number
): void {
  // Record current position
  const currentPage = getCurrentPageFromScroll(container);
  const slot = container.querySelector(`[data-page-number="${currentPage}"]`);
  const slotRect = slot!.getBoundingClientRect();
  const containerRect = container.getBoundingClientRect();
  const relativeOffset = (containerRect.top - slotRect.top) / slotRect.height;

  // Apply new scale
  setScale(state, container, newScale);

  // Restore position after DOM updates
  requestAnimationFrame(() => {
    const updatedSlot = container.querySelector(`[data-page-number="${currentPage}"]`);
    if (updatedSlot) {
      updatedSlot.scrollIntoView({ block: "start" });
      const newSlotRect = updatedSlot.getBoundingClientRect();
      container.scrollTop += relativeOffset * newSlotRect.height;
    }
  });
}
```

---

## Viewer Cleanup / Destroy

```typescript
function destroyViewer(
  observer: IntersectionObserver,
  container: HTMLElement,
  doc: PDFDocumentProxy
): void {
  // 1. Stop observing
  observer.disconnect();

  // 2. Cancel all active renders
  for (const [pageNum, task] of activeRenders) {
    task.cancel();
  }
  activeRenders.clear();

  // 3. Release all canvases
  container.querySelectorAll("canvas").forEach((canvas) => {
    const ctx = canvas.getContext("2d");
    if (ctx) ctx.clearRect(0, 0, canvas.width, canvas.height);
    canvas.width = 0;
    canvas.height = 0;
  });

  // 4. Clear container
  container.innerHTML = "";
  renderedPages.clear();

  // 5. Destroy the document
  doc.destroy();
}
```
