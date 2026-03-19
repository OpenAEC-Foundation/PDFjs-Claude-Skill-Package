# Document Loading — Anti-Patterns

> Common mistakes when loading PDFs with pdfjs-dist 5.x, with explanations and fixes.

---

## AP-01: Loading Without Setting workerSrc

### Wrong

```typescript
import { getDocument } from 'pdfjs-dist';

// MISSING: GlobalWorkerOptions.workerSrc = ...
const doc = await getDocument('/document.pdf').promise;
```

### Why It Fails

PDF.js falls back to a "fake worker" that runs parsing on the main thread. This blocks the UI during loading and parsing, causing visible freezes — especially on large documents. No error is thrown, making this a silent performance bug.

### Correct

```typescript
import { getDocument, GlobalWorkerOptions } from 'pdfjs-dist';

GlobalWorkerOptions.workerSrc = new URL(
  'pdfjs-dist/build/pdf.worker.min.mjs',
  import.meta.url
).toString();

const doc = await getDocument('/document.pdf').promise;
```

---

## AP-02: Not Destroying Documents (Memory Leak)

### Wrong

```typescript
async function showPageCount(url: string): Promise<number> {
  const doc = await getDocument(url).promise;
  return doc.numPages;
  // doc is never destroyed — fonts, images, and parsed data stay in memory
}
```

### Why It Fails

Each `PDFDocumentProxy` holds parsed page trees, font data, and image caches. In a single-page app that loads multiple PDFs, memory grows continuously. The worker thread also retains state for undestroyed documents.

### Correct

```typescript
async function showPageCount(url: string): Promise<number> {
  const doc = await getDocument(url).promise;
  const count = doc.numPages;
  await doc.destroy();
  return count;
}
```

### Correct (with try/finally)

```typescript
async function extractInfo(url: string): Promise<object> {
  const doc = await getDocument(url).promise;
  try {
    const { info } = await doc.getMetadata();
    return { pages: doc.numPages, title: info.Title };
  } finally {
    await doc.destroy();
  }
}
```

---

## AP-03: Not Handling Password Errors

### Wrong

```typescript
async function loadPdf(url: string) {
  // No error handling — crashes silently on protected PDFs
  const doc = await getDocument(url).promise;
  return doc;
}
```

### Why It Fails

User-uploaded PDFs may be password-protected. Without handling `PasswordException`, the promise rejects with a non-descriptive error, and users see no prompt to enter a password.

### Correct

```typescript
import { getDocument, PasswordResponses } from 'pdfjs-dist';

async function loadPdf(url: string, password?: string) {
  try {
    const params: { url: string; password?: string } = { url };
    if (password) params.password = password;
    return await getDocument(params).promise;
  } catch (error) {
    if (error instanceof Error && error.name === 'PasswordException') {
      // Let the UI prompt for a password and retry
      throw new Error('PDF requires a password');
    }
    throw error;
  }
}
```

---

## AP-04: Using 0-Based Page Numbers

### Wrong

```typescript
const doc = await getDocument(url).promise;
// WRONG: pages are 1-indexed, not 0-indexed
const firstPage = await doc.getPage(0); // Throws "Page number out of range"
```

### Correct

```typescript
const doc = await getDocument(url).promise;
const firstPage = await doc.getPage(1); // First page is 1
const lastPage = await doc.getPage(doc.numPages); // Last page
```

---

## AP-05: Loading All Pages Synchronously

### Wrong

```typescript
const doc = await getDocument(url).promise;
const pages = [];
for (let i = 1; i <= doc.numPages; i++) {
  const page = await doc.getPage(i);
  const viewport = page.getViewport({ scale: 1.5 });
  // Render each page...
  pages.push(page);
}
// Loads ALL pages into memory at once — crashes on 1000-page PDFs
```

### Why It Fails

Loading all pages sequentially blocks the async queue and consumes memory proportional to the total page count. For large documents (hundreds or thousands of pages), this causes out-of-memory errors or extreme latency.

### Correct: Lazy Loading (load pages on demand)

```typescript
const doc = await getDocument(url).promise;

async function renderPage(pageNum: number, canvas: HTMLCanvasElement) {
  const page = await doc.getPage(pageNum);
  const viewport = page.getViewport({ scale: 1.5 });
  canvas.width = viewport.width;
  canvas.height = viewport.height;
  await page.render({
    canvasContext: canvas.getContext('2d')!,
    viewport,
  }).promise;
  // Release rendering caches for this page when done
  page.cleanup();
}

// Only render visible pages
renderPage(currentPageNumber, canvasElement);
```

### Correct: Batch Loading (bounded concurrency)

```typescript
async function loadPagesBatched(
  doc: PDFDocumentProxy,
  batchSize = 5
): Promise<void> {
  for (let start = 1; start <= doc.numPages; start += batchSize) {
    const end = Math.min(start + batchSize, doc.numPages + 1);
    const batch = [];
    for (let i = start; i < end; i++) {
      batch.push(doc.getPage(i));
    }
    const pages = await Promise.all(batch);
    // Process batch...
    pages.forEach((page) => page.cleanup());
  }
}
```

---

## AP-06: Passing Raw String Data to getDocument

### Wrong

```typescript
// Fetching PDF and passing as string — CORRUPTS binary data
const response = await fetch('/document.pdf');
const text = await response.text(); // WRONG: text encoding corrupts binary
const doc = await getDocument({ data: text }).promise;
```

### Why It Fails

PDF files are binary data. Converting to a string via `.text()` applies UTF-8 decoding, which corrupts binary byte sequences. The resulting data is not a valid PDF.

### Correct

```typescript
const response = await fetch('/document.pdf');
const arrayBuffer = await response.arrayBuffer();
const doc = await getDocument({ data: arrayBuffer }).promise;
```

---

## AP-07: Not Cancelling Previous Loads

### Wrong

```typescript
// User clicks "Load" rapidly — multiple documents load simultaneously
async function onLoadClick(url: string) {
  const doc = await getDocument(url).promise;
  renderDocument(doc);
}
```

### Why It Fails

Each `getDocument()` call starts a new worker task. If the user triggers multiple loads (e.g., clicking different PDF links rapidly), multiple documents load and render simultaneously, causing flickering, wasted bandwidth, and memory pressure.

### Correct

```typescript
let currentTask: PDFDocumentLoadingTask | null = null;
let currentDoc: PDFDocumentProxy | null = null;

async function onLoadClick(url: string) {
  // Cancel any in-progress load
  if (currentTask) {
    await currentTask.destroy();
  }
  // Destroy previous document
  if (currentDoc) {
    await currentDoc.destroy();
    currentDoc = null;
  }

  currentTask = getDocument(url);
  try {
    currentDoc = await currentTask.promise;
    renderDocument(currentDoc);
  } catch (error) {
    if (error instanceof Error && error.message !== 'Loading aborted') {
      throw error; // Real error, not a cancellation
    }
  }
}
```

---

## AP-08: Using doc/page After destroy()

### Wrong

```typescript
const doc = await getDocument(url).promise;
await doc.destroy();

// WRONG: document is destroyed — all methods will reject
const page = await doc.getPage(1); // Rejects with error
```

### Correct

ALWAYS finish all operations on a document BEFORE calling `destroy()`. If you need to reload, create a new `getDocument()` call.

---

## AP-09: Missing CMap Configuration for CJK Documents

### Wrong

```typescript
// Chinese/Japanese/Korean text appears as blank or garbled
const doc = await getDocument('/chinese-report.pdf').promise;
```

### Why It Fails

CJK fonts use character maps (CMaps) that are NOT bundled with pdfjs-dist by default. Without configuring `cMapUrl`, text extraction and rendering for CJK characters silently fails.

### Correct

```typescript
const doc = await getDocument({
  url: '/chinese-report.pdf',
  cMapUrl: '/node_modules/pdfjs-dist/cmaps/',
  cMapPacked: true,
}).promise;
```

---

## AP-10: Wrapping getDocument in Unnecessary try/catch Without Rethrowing

### Wrong

```typescript
async function loadPdf(url: string) {
  try {
    return await getDocument(url).promise;
  } catch (error) {
    console.log('Failed to load PDF');
    // Error is swallowed — caller has no idea loading failed
    return null;
  }
}
```

### Why It Fails

Swallowing errors without rethrowing or returning a meaningful error state hides problems. The caller receives `null` and has no way to distinguish between "PDF not found", "network error", "password required", or "corrupted file".

### Correct

```typescript
async function loadPdf(url: string): Promise<PDFDocumentProxy> {
  try {
    return await getDocument(url).promise;
  } catch (error) {
    if (error instanceof Error) {
      if (error.name === 'PasswordException') {
        throw new Error('PDF requires a password');
      }
      if (error.name === 'InvalidPDFException') {
        throw new Error('File is not a valid PDF');
      }
      if (error.name === 'MissingPDFException') {
        throw new Error('PDF file not found');
      }
    }
    throw error; // ALWAYS rethrow unknown errors
  }
}
```
