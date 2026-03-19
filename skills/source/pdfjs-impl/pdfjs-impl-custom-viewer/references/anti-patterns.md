# Custom Viewer Anti-Patterns (pdfjs-dist 5.x)

Common mistakes when building a custom PDF viewer and how to fix them.

---

## 1. Rendering All Pages at Once

**Severity**: Critical -- causes out-of-memory crashes, UI freezing, and tab crashes on large documents.

### Wrong

```typescript
// BROKEN: Renders every page on load
async function loadAllPages(
  doc: PDFDocumentProxy,
  container: HTMLElement
): Promise<void> {
  for (let i = 1; i <= doc.numPages; i++) {
    const page = await doc.getPage(i);
    const viewport = page.getViewport({ scale: 1.5 });
    const dpr = window.devicePixelRatio || 1;

    const canvas = document.createElement("canvas");
    canvas.width = Math.floor(viewport.width * dpr);
    canvas.height = Math.floor(viewport.height * dpr);
    container.appendChild(canvas);

    const ctx = canvas.getContext("2d")!;
    ctx.scale(dpr, dpr);
    await page.render({ canvasContext: ctx, viewport }).promise;
  }
  // A 200-page PDF at scale 1.5 on 2x DPR = ~7.2 GB of canvas memory
}
```

### Correct

```typescript
// ALWAYS use IntersectionObserver to render only visible pages
function setupLazyRendering(container: HTMLElement): IntersectionObserver {
  return new IntersectionObserver(
    (entries) => {
      for (const entry of entries) {
        const pageNum = Number((entry.target as HTMLElement).dataset.pageNumber);
        if (entry.isIntersecting && !renderedPages.has(pageNum)) {
          renderPage(pageNum, entry.target as HTMLDivElement);
        } else if (!entry.isIntersecting) {
          cleanupPage(pageNum, entry.target as HTMLDivElement);
        }
      }
    },
    { root: container, rootMargin: "200px 0px" }
  );
}
```

**Why**: Each canvas at scale 1.5 on a 2x DPR display consumes ~36 MB of GPU memory. Even 50 pages total 1.8 GB. ALWAYS use lazy loading with IntersectionObserver and ALWAYS clean up pages that scroll out of view.

---

## 2. No Memory Cleanup When Pages Scroll Out of View

**Severity**: Critical -- memory grows linearly as user scrolls, eventually crashing.

### Wrong

```typescript
// BROKEN: Pages are rendered but never cleaned up
const observer = new IntersectionObserver((entries) => {
  for (const entry of entries) {
    if (entry.isIntersecting) {
      renderPage(entry.target as HTMLDivElement);
    }
    // When page scrolls OUT of view: nothing happens.
    // Canvas stays in DOM consuming GPU memory forever.
  }
});
```

### Correct

```typescript
const observer = new IntersectionObserver((entries) => {
  for (const entry of entries) {
    const pageNum = Number((entry.target as HTMLElement).dataset.pageNumber);

    if (entry.isIntersecting) {
      if (!renderedPages.has(pageNum)) {
        renderPage(entry.target as HTMLDivElement);
      }
    } else {
      // ALWAYS clean up when page leaves viewport
      cleanupPage(pageNum, entry.target as HTMLDivElement);
    }
  }
});

function cleanupPage(pageNum: number, slot: HTMLDivElement): void {
  // Cancel in-progress render
  const task = activeRenders.get(pageNum);
  if (task) {
    task.cancel();
    activeRenders.delete(pageNum);
  }

  // CRITICAL: Set canvas dimensions to 0 to release GPU buffer
  const canvas = slot.querySelector("canvas");
  if (canvas) {
    canvas.width = 0;
    canvas.height = 0;
    canvas.remove();
  }

  // Remove text/annotation layers
  slot.querySelectorAll(".textLayer, .annotationLayer").forEach((el) => el.remove());
  renderedPages.delete(pageNum);
}
```

**Why**: Setting `canvas.width = 0` and `canvas.height = 0` is the ONLY reliable way to release the GPU memory buffer associated with a canvas. Simply removing the canvas from the DOM does NOT immediately free GPU memory -- the browser may keep the buffer alive for potential reuse. ALWAYS zero the dimensions before removing.

---

## 3. Calling getPage() for All Pages During Initialization

**Severity**: High -- blocks UI thread for seconds on large documents.

### Wrong

```typescript
// BROKEN: Fetches all page objects upfront
async function initViewer(doc: PDFDocumentProxy): Promise<void> {
  const allPages: PDFPageProxy[] = [];

  // For a 500-page document, this loop takes 5-10 seconds
  // The UI is completely frozen during this time
  for (let i = 1; i <= doc.numPages; i++) {
    allPages.push(await doc.getPage(i));
  }

  // Now create the viewer...
}
```

### Correct

```typescript
// ALWAYS fetch pages lazily when needed
async function initViewer(doc: PDFDocumentProxy): Promise<void> {
  // Get only the first page for initial dimensions
  const firstPage = await doc.getPage(1);
  const baseViewport = firstPage.getViewport({ scale: 1.0 });

  // For large documents (100+ pages), assume uniform page sizes
  // Create slots using first page dimensions
  for (let i = 1; i <= doc.numPages; i++) {
    createSlotWithDimensions(i, baseViewport.width, baseViewport.height);
  }

  // Pages are fetched individually when they scroll into view
}
```

**Why**: `getPage()` involves parsing page metadata from the PDF. For large documents this is not instantaneous. Fetching pages lazily means the viewer is interactive immediately after loading -- pages are fetched only when IntersectionObserver triggers rendering.

---

## 4. Not Debouncing Scroll Events for Page Detection

**Severity**: Medium -- causes layout thrashing and dropped frames.

### Wrong

```typescript
// BROKEN: Runs expensive DOM queries on every scroll event (60x/sec)
container.addEventListener("scroll", () => {
  const slots = container.querySelectorAll(".page-slot");
  for (const slot of slots) {
    const rect = slot.getBoundingClientRect(); // Forces layout recalculation
    // ... determine current page
  }
  // At 60fps, this runs getBoundingClientRect() on EVERY page 60 times per second
  // 500 pages = 30,000 getBoundingClientRect() calls per second
});
```

### Correct

```typescript
// ALWAYS debounce scroll-based calculations
let scrollTimer: ReturnType<typeof setTimeout>;

container.addEventListener("scroll", () => {
  clearTimeout(scrollTimer);
  scrollTimer = setTimeout(() => {
    const centerY = container.getBoundingClientRect().top + container.clientHeight / 2;
    const slots = container.querySelectorAll(".page-slot");

    for (const slot of slots) {
      const rect = slot.getBoundingClientRect();
      if (rect.top <= centerY && rect.bottom >= centerY) {
        updateCurrentPage(Number((slot as HTMLElement).dataset.pageNumber));
        break; // Stop after finding the current page
      }
    }
  }, 50); // 50ms debounce
});
```

**Why**: `getBoundingClientRect()` forces the browser to synchronously calculate layout. Calling it inside an un-debounced scroll handler causes layout thrashing -- the browser cannot batch layout calculations, leading to dropped frames and janky scrolling.

---

## 5. Not Cancelling Renders During Zoom Change

**Severity**: High -- causes visual artifacts and wasted GPU cycles.

### Wrong

```typescript
// BROKEN: Zoom changes without cancelling existing renders
function onZoomChange(newScale: number): void {
  const slots = container.querySelectorAll(".page-slot");

  slots.forEach(async (slot) => {
    const pageNum = Number((slot as HTMLElement).dataset.pageNumber);
    const page = await doc.getPage(pageNum);
    const viewport = page.getViewport({ scale: newScale });
    const canvas = slot.querySelector("canvas")!;
    const ctx = canvas.getContext("2d")!;

    // Old render is still writing to this canvas!
    // New render starts drawing over the old one -- visual corruption
    page.render({ canvasContext: ctx, viewport });
  });
}
```

### Correct

```typescript
function onZoomChange(newScale: number): void {
  const slots = container.querySelectorAll(".page-slot");

  slots.forEach((slot) => {
    const pageNum = Number((slot as HTMLElement).dataset.pageNumber);

    // ALWAYS cancel in-progress render first
    const task = activeRenders.get(pageNum);
    if (task) {
      task.cancel();
      activeRenders.delete(pageNum);
    }

    // Remove old canvas, mark as not rendered
    cleanupPage(pageNum, slot as HTMLDivElement);
  });

  // Resize slots -- IntersectionObserver will trigger re-renders for visible pages
  // ... resize logic
}
```

**Why**: When zoom changes, ALL visible pages need to be re-rendered at the new scale. If the old renders are not cancelled first, the old and new renders draw to the same canvases simultaneously, producing garbled output. ALWAYS cancel before re-rendering.

---

## 6. Not Handling Window Resize

**Severity**: Medium -- fit-to-width breaks when user resizes browser window.

### Wrong

```typescript
// BROKEN: Scale is calculated once and never updated
const scale = container.clientWidth / baseViewport.width;
// If user resizes browser window, content overflows or has excessive whitespace
```

### Correct

```typescript
// ALWAYS recalculate scale on resize (debounced)
let resizeTimer: ReturnType<typeof setTimeout>;

window.addEventListener("resize", () => {
  clearTimeout(resizeTimer);
  resizeTimer = setTimeout(() => {
    const baseViewport = firstPage.getViewport({ scale: 1.0 });
    const newScale = container.clientWidth / baseViewport.width;
    applyZoom(state, container, newScale);
  }, 200);
});
```

**Why**: When the user resizes the browser, the container width changes but the viewer scale stays fixed. The result is either horizontal overflow (window shrunk) or wasted whitespace (window enlarged). ALWAYS listen for `resize` and recalculate the scale. Debounce at 200ms to avoid excessive re-renders during drag-resize.

---

## 7. Using innerHTML to Clear Page Slots

**Severity**: Medium -- causes memory leaks by not releasing canvas GPU buffers.

### Wrong

```typescript
// BROKEN: Clears DOM but does not release GPU memory
function cleanupPage(slot: HTMLDivElement): void {
  slot.innerHTML = ""; // Canvas removed from DOM but GPU buffer NOT freed
}
```

### Correct

```typescript
function cleanupPage(slot: HTMLDivElement): void {
  const canvas = slot.querySelector("canvas");
  if (canvas) {
    // ALWAYS zero canvas dimensions before removal
    const ctx = canvas.getContext("2d");
    if (ctx) ctx.clearRect(0, 0, canvas.width, canvas.height);
    canvas.width = 0;  // Releases GPU buffer
    canvas.height = 0;
    canvas.remove();
  }
  // Then remove other elements
  slot.querySelectorAll(".textLayer, .annotationLayer").forEach((el) => el.remove());
}
```

**Why**: `innerHTML = ""` removes elements from the DOM but does NOT trigger GPU memory release for canvas elements. The browser's GPU process keeps the framebuffer allocated until garbage collection eventually runs. Setting `width = 0` and `height = 0` explicitly releases the GPU buffer immediately. In a viewer where users scroll through hundreds of pages, this difference prevents gigabytes of leaked GPU memory.

---

## 8. Extracting Text for Search Without Caching

**Severity**: Medium -- repeated searches re-parse every page, causing multi-second delays.

### Wrong

```typescript
// BROKEN: Re-extracts text on every search query
async function search(doc: PDFDocumentProxy, query: string): Promise<void> {
  for (let i = 1; i <= doc.numPages; i++) {
    const page = await doc.getPage(i);
    const textContent = await page.getTextContent(); // Called EVERY search
    // ... search logic
  }
}
```

### Correct

```typescript
// ALWAYS cache text content per page
const textCache = new Map<number, TextContent>();

async function search(doc: PDFDocumentProxy, query: string): Promise<void> {
  for (let i = 1; i <= doc.numPages; i++) {
    let textContent = textCache.get(i);
    if (!textContent) {
      const page = await doc.getPage(i);
      textContent = await page.getTextContent();
      textCache.set(i, textContent);
    }
    // ... search logic using cached textContent
  }
}
```

**Why**: `page.getTextContent()` parses the PDF's text operators for each page. For a 200-page document, this takes 2-5 seconds. Without caching, every keystroke in the search box triggers this full parse. ALWAYS cache the result -- text content does not change during the viewer session.

---

## 9. Print Without intent: "print"

**Severity**: Low -- some annotations meant only for print will not appear.

### Wrong

```typescript
// BROKEN: Renders for screen instead of print
await page.render({
  canvasContext: ctx,
  viewport: viewport,
  // Missing intent: "print"
  // Print-only annotations (like form field hints) will not render
}).promise;
```

### Correct

```typescript
// ALWAYS use intent: "print" when rendering for print output
await page.render({
  canvasContext: ctx,
  viewport: viewport,
  intent: "print",
}).promise;
```

**Why**: PDFs can contain annotations flagged as "print only" or "screen only" via the annotation flags. Using `intent: "display"` (the default) omits print-only content. ALWAYS set `intent: "print"` when rendering pages for the print workflow.
