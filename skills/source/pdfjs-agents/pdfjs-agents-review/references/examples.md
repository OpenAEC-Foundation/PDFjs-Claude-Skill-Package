# Review Examples: Good and Bad PDF.js Code

Side-by-side examples showing code that passes review versus code that fails, with annotations explaining each issue.

---

## Example 1: Complete Viewer Setup (Good vs Bad)

### BAD: Multiple Critical Issues

```typescript
import { getDocument } from "pdfjs-dist";

async function renderPdf(url: string, container: HTMLDivElement) {
  // FAIL Check 1: No workerSrc set before getDocument()
  const doc = await getDocument(url).promise;

  // FAIL Check 7: Renders ALL pages at once
  for (let i = 1; i <= doc.numPages; i++) {
    const page = await doc.getPage(i);
    const viewport = page.getViewport({ scale: 1.5 });
    const canvas = document.createElement("canvas");

    // FAIL Check 4: No devicePixelRatio handling
    canvas.width = viewport.width;
    canvas.height = viewport.height;

    container.appendChild(canvas);
    const ctx = canvas.getContext("2d")!;

    // FAIL Check 3: No render task tracking or cancellation
    page.render({ canvasContext: ctx, viewport });

    // FAIL Check 10: No error handling
  }

  // FAIL Check 6: Document never destroyed
}
```

**Issues found**: 6 FAIL results across Checks 1, 3, 4, 6, 7, 10.

### GOOD: All Checks Pass

```typescript
import { getDocument, GlobalWorkerOptions, TextLayer, version } from "pdfjs-dist";
import type { PDFDocumentProxy, PDFPageProxy, RenderTask } from "pdfjs-dist";

// CHECK 1 PASS: Worker configured at module level, before any getDocument()
// CHECK 2 PASS: Version interpolated from package
GlobalWorkerOptions.workerSrc =
  `https://cdnjs.cloudflare.com/ajax/libs/pdf.js/${version}/pdf.worker.min.mjs`;

// CHECK 9 PASS: Correct imports for pdfjs-dist 5.x
// CHECK 8 PASS: Using TextLayer class (not deprecated renderTextLayer)

const activeRenders = new Map<number, RenderTask>();
const renderedPages = new Set<number>();
let currentDoc: PDFDocumentProxy | null = null;

async function initViewer(
  url: string,
  container: HTMLDivElement
): Promise<void> {
  // CHECK 6 PASS: Previous document destroyed before loading new one
  if (currentDoc) {
    await currentDoc.destroy();
    currentDoc = null;
  }

  // CHECK 10 PASS: Error handling with typed exceptions
  try {
    currentDoc = await getDocument(url).promise;
  } catch (error) {
    if (error instanceof Error) {
      if (error.name === "PasswordException") {
        throw new Error("PDF requires a password");
      }
      if (error.name === "InvalidPDFException") {
        throw new Error("File is not a valid PDF");
      }
    }
    throw error;
  }

  // Get first page for dimensions
  const firstPage = await currentDoc.getPage(1);
  const baseViewport = firstPage.getViewport({ scale: 1.0 });

  // Create placeholder slots for all pages (lightweight)
  container.style.position = "relative";
  for (let i = 1; i <= currentDoc.numPages; i++) {
    const slot = document.createElement("div");
    slot.dataset.pageNumber = String(i);
    slot.style.width = `${Math.floor(baseViewport.width * 1.5)}px`;
    slot.style.height = `${Math.floor(baseViewport.height * 1.5)}px`;
    slot.style.position = "relative";
    slot.style.marginBottom = "8px";
    slot.style.backgroundColor = "#f0f0f0";
    container.appendChild(slot);
  }

  // CHECK 7 PASS: Lazy loading with IntersectionObserver
  const observer = new IntersectionObserver(
    (entries) => {
      for (const entry of entries) {
        const pageNum = Number(
          (entry.target as HTMLElement).dataset.pageNumber
        );
        if (entry.isIntersecting && !renderedPages.has(pageNum)) {
          renderPage(currentDoc!, pageNum, entry.target as HTMLDivElement, 1.5);
        } else if (!entry.isIntersecting) {
          cleanupPage(pageNum, entry.target as HTMLDivElement);
        }
      }
    },
    { root: container, rootMargin: "200px 0px" }
  );

  container.querySelectorAll("[data-page-number]").forEach((el) => {
    observer.observe(el);
  });
}

async function renderPage(
  doc: PDFDocumentProxy,
  pageNum: number,
  slot: HTMLDivElement,
  scale: number
): Promise<void> {
  // CHECK 3 PASS: Cancel previous render for this page
  const existingTask = activeRenders.get(pageNum);
  if (existingTask) {
    existingTask.cancel();
    activeRenders.delete(pageNum);
  }

  const page = await doc.getPage(pageNum);
  const viewport = page.getViewport({ scale });

  // CHECK 4 PASS: devicePixelRatio applied correctly
  const dpr = window.devicePixelRatio || 1;
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
    renderedPages.add(pageNum);

    // CHECK 5 PASS: TextLayer stacked on top of canvas with correct z-index
    const textContent = await page.getTextContent();
    const textLayerDiv = document.createElement("div");
    textLayerDiv.className = "textLayer";
    textLayerDiv.style.position = "absolute";
    textLayerDiv.style.top = "0";
    textLayerDiv.style.left = "0";
    textLayerDiv.style.width = `${Math.floor(viewport.width)}px`;
    textLayerDiv.style.height = `${Math.floor(viewport.height)}px`;
    textLayerDiv.style.zIndex = "1";
    slot.appendChild(textLayerDiv);

    // CHECK 8 PASS: Using TextLayer class (v5 API)
    const textLayer = new TextLayer({
      textContentSource: textContent,
      container: textLayerDiv,
      viewport: viewport,
    });
    await textLayer.render();
  } catch (err: unknown) {
    if (err instanceof Error && err.message === "Rendering cancelled") {
      return; // CHECK 3 PASS: Cancellation handled gracefully
    }
    throw err; // CHECK 10 PASS: Real errors re-thrown
  } finally {
    activeRenders.delete(pageNum); // CHECK 6 PASS: Reference cleared
  }
}

// CHECK 6 PASS: Proper cleanup with canvas dimension zeroing
function cleanupPage(pageNum: number, slot: HTMLDivElement): void {
  const task = activeRenders.get(pageNum);
  if (task) {
    task.cancel();
    activeRenders.delete(pageNum);
  }

  const canvas = slot.querySelector("canvas");
  if (canvas) {
    canvas.width = 0;
    canvas.height = 0;
    canvas.remove();
  }

  slot.querySelectorAll(".textLayer, .annotationLayer").forEach((el) =>
    el.remove()
  );
  renderedPages.delete(pageNum);
}
```

**Result**: All 10 checks PASS.

---

## Example 2: Zoom Handler (Good vs Bad)

### BAD: Race Conditions and Memory Leaks

```typescript
// FAIL Check 3: No cancellation
// FAIL Check 4: No DPI handling
// FAIL Check 6: No cleanup
function onZoomChange(page: PDFPageProxy, canvas: HTMLCanvasElement, newScale: number) {
  const viewport = page.getViewport({ scale: newScale });
  canvas.width = viewport.width;
  canvas.height = viewport.height;
  const ctx = canvas.getContext("2d")!;
  page.render({ canvasContext: ctx, viewport });
}
```

### GOOD: Cancel-Then-Render with DPI

```typescript
let currentRenderTask: RenderTask | null = null;

async function onZoomChange(
  page: PDFPageProxy,
  canvas: HTMLCanvasElement,
  newScale: number
): Promise<void> {
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

---

## Example 3: Text Layer Setup (Good vs Bad)

### BAD: Deprecated API, Missing CSS, Viewport Mismatch

```typescript
// FAIL Check 8: Using deprecated renderTextLayer (removed in v5)
import { renderTextLayer } from "pdfjs-dist";

const canvasViewport = page.getViewport({ scale: 1.5 });
// ... render canvas with canvasViewport ...

// FAIL Check 5: Different viewport for text layer
const textViewport = page.getViewport({ scale: 1.0 });
renderTextLayer({
  textContent: await page.getTextContent(),
  container: textLayerDiv,
  viewport: textViewport,
  textDivs: [], // FAIL Check 8: Old API parameter
});
// FAIL Check 5: No CSS imported, text renders as visible black text
```

### GOOD: v5 API with Correct Positioning

```typescript
import { TextLayer } from "pdfjs-dist";

// Use the SAME viewport as the canvas
const viewport = page.getViewport({ scale: 1.5 });

// Correct absolute positioning
const textLayerDiv = document.createElement("div");
textLayerDiv.className = "textLayer"; // CSS from pdf_viewer.css
textLayerDiv.style.position = "absolute";
textLayerDiv.style.top = "0";
textLayerDiv.style.left = "0";
textLayerDiv.style.width = `${Math.floor(viewport.width)}px`;
textLayerDiv.style.height = `${Math.floor(viewport.height)}px`;
container.appendChild(textLayerDiv);

const textLayer = new TextLayer({
  textContentSource: await page.getTextContent(),
  container: textLayerDiv,
  viewport: viewport, // SAME viewport as canvas
});
await textLayer.render();
```

---

## Example 4: Document Lifecycle (Good vs Bad)

### BAD: Memory Leaks

```typescript
// FAIL Check 6: No cleanup on document switch
async function openPdf(url: string) {
  const doc = await getDocument(url).promise;
  await renderFirstPage(doc);
  // Previous document never destroyed
  // Loading task not tracked for cancellation
}
```

### GOOD: Full Lifecycle Management

```typescript
let currentDoc: PDFDocumentProxy | null = null;
let currentTask: PDFDocumentLoadingTask | null = null;

async function openPdf(url: string): Promise<void> {
  // Cancel in-progress load
  if (currentTask) {
    await currentTask.destroy();
    currentTask = null;
  }

  // Destroy previous document
  if (currentDoc) {
    await currentDoc.destroy();
    currentDoc = null;
  }

  currentTask = getDocument(url);

  try {
    currentDoc = await currentTask.promise;
    await renderFirstPage(currentDoc);
  } catch (error) {
    if (error instanceof Error && error.message === "Loading aborted") {
      return; // Cancellation, not an error
    }
    throw error;
  } finally {
    currentTask = null;
  }
}

// On application shutdown
async function dispose(): Promise<void> {
  if (currentTask) {
    await currentTask.destroy();
  }
  if (currentDoc) {
    await currentDoc.destroy();
  }
}
```

---

## Example 5: Worker Setup (Good vs Bad)

### BAD: Version Drift Risk

```typescript
import { GlobalWorkerOptions, getDocument } from "pdfjs-dist";

// FAIL Check 2: Hardcoded version will drift from npm package
GlobalWorkerOptions.workerSrc =
  "https://cdnjs.cloudflare.com/ajax/libs/pdf.js/5.0.0/pdf.worker.min.mjs";

// FAIL Check 2: Using .js extension (v5 ships .mjs only)
GlobalWorkerOptions.workerSrc = "/pdf.worker.min.js";

// FAIL Check 2: Using @latest (CDN resolves independently)
GlobalWorkerOptions.workerSrc =
  "https://unpkg.com/pdfjs-dist@latest/build/pdf.worker.min.mjs";
```

### GOOD: Version-Safe Setup

```typescript
import { GlobalWorkerOptions, getDocument, version } from "pdfjs-dist";

// Option A: Interpolate version from package
GlobalWorkerOptions.workerSrc =
  `https://cdnjs.cloudflare.com/ajax/libs/pdf.js/${version}/pdf.worker.min.mjs`;

// Option B: Let bundler resolve from same package (Webpack 5)
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

// Option C: Vite-specific
import workerUrl from "pdfjs-dist/build/pdf.worker.min.mjs?url";
GlobalWorkerOptions.workerSrc = workerUrl;
```
