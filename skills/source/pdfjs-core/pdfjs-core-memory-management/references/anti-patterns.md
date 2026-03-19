# Anti-Patterns: Memory Management

> pdfjs-dist 5.x -- Common memory leak patterns and their corrections.

## AP-001: Never Calling destroy() on Document Switch

**WRONG:**
```typescript
async function openPdf(url: string) {
  const doc = await getDocument(url).promise;
  const page = await doc.getPage(1);
  await page.render({ canvasContext: ctx, viewport }).promise;
  // doc is never destroyed -- worker keeps running, memory accumulates
}

// User opens multiple PDFs -- each call leaks a worker + document data
openPdf('/report-q1.pdf');
openPdf('/report-q2.pdf');
openPdf('/report-q3.pdf');
// Result: 3 active workers, 3 document caches in memory
```

**RIGHT:**
```typescript
let currentDoc: PDFDocumentProxy | null = null;

async function openPdf(url: string) {
  // ALWAYS destroy previous document first
  if (currentDoc) {
    await currentDoc.destroy();
  }

  currentDoc = await getDocument(url).promise;
  const page = await currentDoc.getPage(1);
  await page.render({ canvasContext: ctx, viewport }).promise;
}
```

**WHY:** Each `PDFDocumentProxy` owns a worker thread and caches all fetched page data. Without `destroy()`, these resources accumulate with every document load. In a session where a user opens 20 PDFs, this means 20 active workers and potentially hundreds of megabytes of leaked memory.

---

## AP-002: Creating New Canvas for Every Page Render

**WRONG:**
```typescript
async function showPage(pageNum: number) {
  const page = await doc.getPage(pageNum);
  const viewport = page.getViewport({ scale: 1.5 });

  // Creates a NEW canvas every time the user navigates
  const canvas = document.createElement('canvas');
  canvas.width = viewport.width;
  canvas.height = viewport.height;
  container.appendChild(canvas);

  const ctx = canvas.getContext('2d')!;
  await page.render({ canvasContext: ctx, viewport }).promise;
  // Old canvases stay in the DOM, consuming GPU memory
}
```

**RIGHT:**
```typescript
const canvas = document.getElementById('pdf-canvas') as HTMLCanvasElement;
const ctx = canvas.getContext('2d')!;
let currentRenderTask: RenderTask | null = null;

async function showPage(pageNum: number) {
  // Cancel previous render
  if (currentRenderTask) {
    currentRenderTask.cancel();
  }

  const page = await doc.getPage(pageNum);
  const viewport = page.getViewport({ scale: 1.5 });
  const dpr = window.devicePixelRatio || 1;

  // Reuse the SAME canvas
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;
  ctx.scale(dpr, dpr);

  currentRenderTask = page.render({ canvasContext: ctx, viewport });
  try {
    await currentRenderTask.promise;
  } catch (err: any) {
    if (err.name !== 'RenderingCancelledException') throw err;
  }
}
```

**WHY:** Each canvas allocates a GPU-backed bitmap. A canvas at 1500x2000 pixels at 2x DPR consumes ~24 MB of GPU memory. Navigating through 50 pages without reusing canvases leaks over 1 GB. ALWAYS reuse canvases in single-page viewers. In multi-page viewers, ALWAYS evict off-screen canvases.

---

## AP-003: Forgetting to Revoke Blob URLs

**WRONG:**
```typescript
async function loadFromUpload(file: File) {
  const url = URL.createObjectURL(file);
  const doc = await getDocument(url).promise;
  // url is never revoked -- the entire PDF binary stays in memory twice:
  // once in the blob URL and once in PDF.js internal buffers
  await renderFirstPage(doc);
}
```

**RIGHT:**
```typescript
async function loadFromUpload(file: File) {
  const url = URL.createObjectURL(file);
  try {
    const doc = await getDocument(url).promise;
    await renderFirstPage(doc);
  } finally {
    // ALWAYS revoke after getDocument() completes
    URL.revokeObjectURL(url);
  }
}
```

**WHY:** `URL.createObjectURL()` creates a reference that keeps the entire `Blob` in memory. PDF.js copies the data during `getDocument()`, so the blob URL is redundant after loading. A 50 MB PDF loaded via blob URL without revocation wastes 50 MB indefinitely. Repeated loads compound the waste.

---

## AP-004: Not Cancelling RenderTask Before Destroy

**WRONG:**
```typescript
async function cleanup() {
  // Destroying document while render is still in progress
  await doc.destroy();
  // Throws: worker receives messages for a destroyed document
}
```

**RIGHT:**
```typescript
async function cleanup() {
  // Step 1: Cancel all active renders
  if (currentRenderTask) {
    currentRenderTask.cancel();
    try {
      await currentRenderTask.promise;
    } catch {
      // RenderingCancelledException -- expected
    }
    currentRenderTask = null;
  }

  // Step 2: Destroy document
  await doc.destroy();
}
```

**WHY:** An active `RenderTask` communicates with the worker thread. If the document is destroyed while a render is in progress, the worker receives messages for a document that no longer exists, causing unhandled exceptions. ALWAYS cancel all renders before destroying the document.

---

## AP-005: No Cleanup on SPA Route Change

**WRONG:**
```typescript
// React component -- NO cleanup on unmount
function PdfViewer({ url }: { url: string }) {
  const canvasRef = useRef<HTMLCanvasElement>(null);

  useEffect(() => {
    async function load() {
      const doc = await getDocument(url).promise;
      const page = await doc.getPage(1);
      const viewport = page.getViewport({ scale: 1.5 });
      const ctx = canvasRef.current!.getContext('2d')!;
      page.render({ canvasContext: ctx, viewport });
    }
    load();
    // Missing return cleanup function!
  }, [url]);

  return <canvas ref={canvasRef} />;
}
// When user navigates away, the worker keeps running in the background
```

**RIGHT:**
```typescript
function PdfViewer({ url }: { url: string }) {
  const canvasRef = useRef<HTMLCanvasElement>(null);

  useEffect(() => {
    let doc: PDFDocumentProxy | null = null;
    let renderTask: RenderTask | null = null;
    let cancelled = false;

    async function load() {
      doc = await getDocument(url).promise;
      if (cancelled) { await doc.destroy(); doc = null; return; }

      const page = await doc.getPage(1);
      if (cancelled) { await doc.destroy(); doc = null; return; }

      const viewport = page.getViewport({ scale: 1.5 });
      const ctx = canvasRef.current!.getContext('2d')!;
      renderTask = page.render({ canvasContext: ctx, viewport });

      try {
        await renderTask.promise;
      } catch (err: any) {
        if (err.name !== 'RenderingCancelledException') throw err;
      }
    }
    load();

    // ALWAYS return cleanup function
    return () => {
      cancelled = true;
      if (renderTask) renderTask.cancel();
      if (doc) doc.destroy();
    };
  }, [url]);

  return <canvas ref={canvasRef} />;
}
```

**WHY:** In SPAs, navigating away from a route unmounts components but does NOT automatically stop JavaScript. Worker threads, pending renders, and cached data persist until the tab is closed. This is the #1 source of memory leaks in production PDF viewers. ALWAYS return a cleanup function from `useEffect` (React) or use `onUnmounted` (Vue) or `ngOnDestroy` (Angular).

---

## AP-006: Keeping All Visited Pages in Memory

**WRONG:**
```typescript
const pageCache = new Map<number, { page: PDFPageProxy; canvas: HTMLCanvasElement }>();

async function goToPage(pageNum: number) {
  if (pageCache.has(pageNum)) {
    showCachedPage(pageNum);
    return;
  }

  const page = await doc.getPage(pageNum);
  const canvas = createAndRenderCanvas(page);
  // Cache grows unbounded -- NEVER evicts
  pageCache.set(pageNum, { page, canvas });
}
// After browsing 200 pages: 200 canvases + 200 page caches in memory
```

**RIGHT:**
```typescript
const MAX_CACHED_PAGES = 7;
const pageCache = new Map<number, { page: PDFPageProxy; canvas: HTMLCanvasElement }>();

async function goToPage(pageNum: number) {
  if (pageCache.has(pageNum)) {
    showCachedPage(pageNum);
    return;
  }

  // Evict oldest entries when pool is full
  while (pageCache.size >= MAX_CACHED_PAGES) {
    const oldestKey = pageCache.keys().next().value;
    const entry = pageCache.get(oldestKey)!;
    entry.page.cleanup();
    entry.canvas.width = 0;
    entry.canvas.height = 0;
    entry.canvas.remove();
    pageCache.delete(oldestKey);
  }

  const page = await doc.getPage(pageNum);
  const canvas = createAndRenderCanvas(page);
  pageCache.set(pageNum, { page, canvas });
}
```

**WHY:** Each cached page holds an operator list, decoded images, and a GPU-backed canvas. Without eviction, memory grows linearly with the number of visited pages. A 500-page document where a user scrolls through everything will crash the browser tab. ALWAYS set a maximum pool size and evict pages that are far from the current viewport.

---

## AP-007: Leaking Event Listeners on Viewer Destruction

**WRONG:**
```typescript
class PdfViewer {
  constructor() {
    // Anonymous functions cannot be removed
    window.addEventListener('resize', () => this.onResize());
    document.addEventListener('keydown', (e) => this.onKeydown(e));
    this.container.addEventListener('scroll', () => this.onScroll());
  }

  destroy() {
    // Cannot remove anonymous listeners!
    // The PdfViewer instance is retained in memory via closure references
    this.doc?.destroy();
  }
}
```

**RIGHT:**
```typescript
class PdfViewer {
  private boundResize: () => void;
  private boundKeydown: (e: KeyboardEvent) => void;
  private boundScroll: () => void;

  constructor() {
    // Store bound references so they can be removed
    this.boundResize = this.onResize.bind(this);
    this.boundKeydown = this.onKeydown.bind(this);
    this.boundScroll = this.onScroll.bind(this);

    window.addEventListener('resize', this.boundResize);
    document.addEventListener('keydown', this.boundKeydown);
    this.container.addEventListener('scroll', this.boundScroll);
  }

  destroy() {
    // Remove ALL event listeners
    window.removeEventListener('resize', this.boundResize);
    document.removeEventListener('keydown', this.boundKeydown);
    this.container.removeEventListener('scroll', this.boundScroll);

    this.doc?.destroy();
  }
}
```

**WHY:** Anonymous arrow functions or inline handlers passed to `addEventListener` cannot be removed because `removeEventListener` requires the exact same function reference. The closure retains a reference to the `PdfViewer` instance, which retains references to the `PDFDocumentProxy`, canvases, and DOM elements. This creates a chain of retained objects that the garbage collector cannot release. ALWAYS store bound handler references and remove them on destroy.

---

## AP-008: Not Clearing Canvas Before Re-render at Different Scale

**WRONG:**
```typescript
async function zoom(newScale: number) {
  const page = await doc.getPage(currentPageNum);
  const viewport = page.getViewport({ scale: newScale });

  // Resize canvas but DON'T clear it
  canvas.width = viewport.width;
  canvas.height = viewport.height;

  // If the new render is smaller than the old one, stale pixels remain visible
  // If render fails, the old image at the wrong scale is displayed
  await page.render({ canvasContext: ctx, viewport }).promise;
}
```

**RIGHT:**
```typescript
async function zoom(newScale: number) {
  // Cancel any active render first
  if (currentRenderTask) {
    currentRenderTask.cancel();
  }

  const page = await doc.getPage(currentPageNum);
  const viewport = page.getViewport({ scale: newScale });
  const dpr = window.devicePixelRatio || 1;

  // Clear before resize to prevent ghost images
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;
  ctx.scale(dpr, dpr);

  currentRenderTask = page.render({ canvasContext: ctx, viewport });
  try {
    await currentRenderTask.promise;
  } catch (err: any) {
    if (err.name !== 'RenderingCancelledException') throw err;
  }
}
```

**WHY:** Resizing a canvas without clearing it first can leave stale pixel data visible, especially if the new dimensions are different or the render fails partway through. ALWAYS clear the canvas and cancel any active render task before starting a new render at a different scale.

---

## AP-009: Loading PDFs in a Loop Without Awaiting Destroy

**WRONG:**
```typescript
const urls = ['/doc1.pdf', '/doc2.pdf', '/doc3.pdf'];

for (const url of urls) {
  const doc = await getDocument(url).promise;
  const text = await extractText(doc);
  results.push(text);
  doc.destroy(); // Not awaited! Next getDocument may start before cleanup finishes
}
```

**RIGHT:**
```typescript
const urls = ['/doc1.pdf', '/doc2.pdf', '/doc3.pdf'];

for (const url of urls) {
  const doc = await getDocument(url).promise;
  const text = await extractText(doc);
  results.push(text);
  await doc.destroy(); // ALWAYS await destroy before loading next document
}
```

**WHY:** `destroy()` returns a `Promise` that resolves when the worker thread has fully cleaned up. Without `await`, the next `getDocument()` may start before cleanup finishes, leading to overlapping workers and resource contention. In batch processing scenarios, this compounds rapidly and causes out-of-memory crashes.
