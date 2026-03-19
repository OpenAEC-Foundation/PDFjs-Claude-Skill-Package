# Anti-Patterns: Core Architecture

> pdfjs-dist 5.x -- Common mistakes and their corrections.

## AP-001: Loading Without Worker Configuration

**WRONG:**
```typescript
import { getDocument } from 'pdfjs-dist';

// Missing GlobalWorkerOptions.workerSrc setup!
const pdfDoc = await getDocument({ url: '/doc.pdf' }).promise;
// Error: "No "GlobalWorkerOptions.workerSrc" specified"
```

**RIGHT:**
```typescript
import { GlobalWorkerOptions, getDocument } from 'pdfjs-dist';

GlobalWorkerOptions.workerSrc = new URL(
  'pdfjs-dist/build/pdf.worker.mjs',
  import.meta.url
).toString();

const pdfDoc = await getDocument({ url: '/doc.pdf' }).promise;
```

**WHY:** PDF.js requires the worker script to perform PDF parsing. Without `workerSrc`, the library cannot spawn the worker thread and throws an error immediately.

---

## AP-002: Rendering All Pages At Once

**WRONG:**
```typescript
const pdfDoc = await getDocument({ url: '/500-page-doc.pdf' }).promise;

// Rendering all 500 pages simultaneously
for (let i = 1; i <= pdfDoc.numPages; i++) {
  const page = await pdfDoc.getPage(i);
  const viewport = page.getViewport({ scale: 2.0 });
  const canvas = document.createElement('canvas');
  canvas.width = viewport.width;
  canvas.height = viewport.height;
  await page.render({ canvasContext: canvas.getContext('2d')!, viewport }).promise;
  container.appendChild(canvas);
}
// Result: 500 canvases in memory, potential OOM crash
```

**RIGHT:**
```typescript
// Use IntersectionObserver to render only visible pages
// See references/examples.md "Lazy Page Rendering" for full implementation
const observer = new IntersectionObserver((entries) => {
  for (const entry of entries) {
    if (entry.isIntersecting) {
      renderPage(pdfDoc, entry.target, pageNum, scale);
      observer.unobserve(entry.target);
    }
  }
}, { rootMargin: '200px' });
```

**WHY:** Each canvas consumes significant GPU and CPU memory. Rendering hundreds of pages simultaneously exhausts browser memory limits and crashes the tab. ALWAYS use lazy loading with IntersectionObserver.

---

## AP-003: Ignoring devicePixelRatio

**WRONG:**
```typescript
const viewport = page.getViewport({ scale: 1.5 });
canvas.width = viewport.width;
canvas.height = viewport.height;
// Looks blurry on HiDPI/Retina displays
```

**RIGHT:**
```typescript
const viewport = page.getViewport({ scale: 1.5 });
const dpr = window.devicePixelRatio || 1;
canvas.width = Math.floor(viewport.width * dpr);
canvas.height = Math.floor(viewport.height * dpr);
canvas.style.width = `${Math.floor(viewport.width)}px`;
canvas.style.height = `${Math.floor(viewport.height)}px`;
context.scale(dpr, dpr);
```

**WHY:** On HiDPI displays (Retina, 4K monitors), `devicePixelRatio` is 2 or higher. Without scaling, the canvas renders at half the physical resolution, producing visibly blurry text and graphics.

---

## AP-004: Concurrent Renders on Same Canvas

**WRONG:**
```typescript
async function onPageChange(pageNum: number) {
  const page = await pdfDoc.getPage(pageNum);
  const viewport = page.getViewport({ scale: 1.5 });
  // Starting new render without cancelling previous one
  page.render({ canvasContext: context, viewport });
}
```

**RIGHT:**
```typescript
let currentRenderTask: RenderTask | null = null;

async function onPageChange(pageNum: number) {
  // ALWAYS cancel previous render first
  if (currentRenderTask) {
    currentRenderTask.cancel();
  }

  const page = await pdfDoc.getPage(pageNum);
  const viewport = page.getViewport({ scale: 1.5 });
  currentRenderTask = page.render({ canvasContext: context, viewport });

  try {
    await currentRenderTask.promise;
  } catch (err: any) {
    if (err.name !== 'RenderingCancelledException') {
      throw err;
    }
    // Cancellation is expected, not an error
  }
}
```

**WHY:** The canvas 2D context has internal state. Two concurrent render operations write to the same state machine, producing corrupted output and throwing `RenderingCancelledException`. ALWAYS cancel before re-rendering.

---

## AP-005: Forgetting to Destroy PDFDocumentProxy

**WRONG:**
```typescript
async function viewPdf(url: string) {
  const pdfDoc = await getDocument({ url }).promise;
  const page = await pdfDoc.getPage(1);
  await page.render({ canvasContext: ctx, viewport }).promise;
  // pdfDoc is never destroyed -- worker thread keeps running
}

// Called repeatedly as user opens different PDFs
viewPdf('/doc1.pdf');
viewPdf('/doc2.pdf');
viewPdf('/doc3.pdf');
// Result: 3 worker threads running, memory accumulating
```

**RIGHT:**
```typescript
let currentDoc: PDFDocumentProxy | null = null;

async function viewPdf(url: string) {
  // ALWAYS destroy previous document first
  if (currentDoc) {
    await currentDoc.destroy();
  }

  currentDoc = await getDocument({ url }).promise;
  const page = await currentDoc.getPage(1);
  await page.render({ canvasContext: ctx, viewport }).promise;
}
```

**WHY:** Each `PDFDocumentProxy` holds references to a worker thread and cached page data. Without `destroy()`, these resources accumulate and cause memory leaks that grow with each document loaded.

---

## AP-006: Using Deprecated renderTextLayer()

**WRONG:**
```typescript
import { renderTextLayer } from 'pdfjs-dist';

// Deprecated in pdfjs-dist 4.x, removed in 5.x
renderTextLayer({
  textContent: await page.getTextContent(),
  container: textLayerDiv,
  viewport,
});
```

**RIGHT:**
```typescript
import { TextLayer } from 'pdfjs-dist';

const textLayer = new TextLayer({
  textContentSource: page.streamTextContent(),
  container: textLayerDiv,
  viewport,
});
await textLayer.render();
```

**WHY:** The `renderTextLayer()` function was deprecated in pdfjs-dist 4.x and removed in 5.x. The `TextLayer` class provides streaming support, better performance, and an `update()` method for viewport changes.

---

## AP-007: Version Mismatch Between pdf.mjs and pdf.worker.mjs

**WRONG:**
```typescript
// pdf.mjs from pdfjs-dist@5.5.207 (installed via npm)
import { GlobalWorkerOptions } from 'pdfjs-dist';

// Worker from a different version (e.g., CDN with outdated URL)
GlobalWorkerOptions.workerSrc = 'https://cdn.example.com/pdf.worker.mjs'; // v4.8.0
```

**RIGHT:**
```typescript
import { GlobalWorkerOptions } from 'pdfjs-dist';

// ALWAYS use the worker from the same package
GlobalWorkerOptions.workerSrc = new URL(
  'pdfjs-dist/build/pdf.worker.mjs',
  import.meta.url
).toString();
```

**WHY:** The main library and worker communicate via an internal protocol. Version mismatches cause silent data corruption, missing features, or explicit "version mismatch" errors. ALWAYS ensure both come from the same pdfjs-dist installation.

---

## AP-008: Using the Viewer Layer as a Library

**WRONG:**
```typescript
// Trying to import viewer components directly
import { PDFViewer, EventBus } from 'pdfjs-dist/web/pdf_viewer.mjs';

// Building on top of Mozilla's reference viewer implementation
const viewer = new PDFViewer({ container, eventBus });
```

**RIGHT:**
```typescript
// Use the Display API to build your own viewer
import { GlobalWorkerOptions, getDocument, TextLayer } from 'pdfjs-dist';

// Build custom viewer logic using getDocument(), render(), TextLayer, etc.
```

**WHY:** The Viewer layer (`pdf_viewer.mjs`) is Mozilla's reference implementation, not a supported library API. It has no stability guarantees, its internals change between versions, and it tightly couples to Mozilla's UI structure. ALWAYS build custom viewers using the Display layer API.

---

## AP-009: Not Handling Password-Protected PDFs

**WRONG:**
```typescript
try {
  const pdfDoc = await getDocument({ url: '/encrypted.pdf' }).promise;
} catch (err) {
  console.error('Failed to load PDF'); // Unhelpful error handling
}
```

**RIGHT:**
```typescript
const loadingTask = getDocument({ url: '/encrypted.pdf' });

loadingTask.onPassword = (callback, reason) => {
  const password = prompt(
    reason === 1 ? 'Enter password:' : 'Wrong password. Try again:'
  );
  if (password) {
    callback(password);
  } else {
    loadingTask.destroy();
  }
};

const pdfDoc = await loadingTask.promise;
```

**WHY:** Without the `onPassword` callback, encrypted PDFs throw a `PasswordException` that crashes the loading flow. ALWAYS set `onPassword` when loading PDFs from untrusted sources where encryption is possible.

---

## AP-010: Using 0-Based Page Numbers

**WRONG:**
```typescript
// Pages are NOT 0-based
const firstPage = await pdfDoc.getPage(0); // Throws error!
```

**RIGHT:**
```typescript
// Pages are 1-based
const firstPage = await pdfDoc.getPage(1);
const lastPage = await pdfDoc.getPage(pdfDoc.numPages);
```

**WHY:** PDF.js uses 1-based page numbering, matching the PDF specification. `getPage(0)` throws an error. ALWAYS use page numbers from 1 to `numPages`.
