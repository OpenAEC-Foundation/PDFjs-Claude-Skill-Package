# Worker Anti-Patterns (pdfjs-dist 5.x)

Common mistakes that cause PDF.js worker errors and how to fix them.

---

## 1. Hardcoded CDN Version (Version Mismatch)

**Severity**: Critical -- causes `The API version "X" does not match the Worker version "Y"` on every update.

### Wrong

```typescript
import { GlobalWorkerOptions } from "pdfjs-dist";

// BROKEN: Hardcoded version drifts from installed pdfjs-dist
GlobalWorkerOptions.workerSrc =
  "https://cdnjs.cloudflare.com/ajax/libs/pdf.js/4.9.155/pdf.worker.min.mjs";
```

### Correct

```typescript
import { GlobalWorkerOptions, version } from "pdfjs-dist";

// ALWAYS interpolate the version from the installed package
GlobalWorkerOptions.workerSrc =
  `https://cdnjs.cloudflare.com/ajax/libs/pdf.js/${version}/pdf.worker.min.mjs`;
```

**Why**: When you update pdfjs-dist via npm, the API version changes but the hardcoded CDN URL still points to the old worker file. This creates a version mismatch that prevents all PDF operations from working.

---

## 2. Setting workerSrc After getDocument()

**Severity**: Critical -- causes `No "GlobalWorkerOptions.workerSrc" specified` or triggers fake worker fallback.

### Wrong

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";

// BROKEN: Worker config happens AFTER document loading starts
async function loadPdf(url: string) {
  const doc = await getDocument({ url }).promise; // <- Worker not configured yet!

  // Too late -- getDocument() already tried to create a worker
  GlobalWorkerOptions.workerSrc = new URL(
    "pdfjs-dist/build/pdf.worker.min.mjs",
    import.meta.url
  ).toString();

  return doc;
}
```

### Correct

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";

// ALWAYS set workerSrc at module initialization, before any getDocument() call
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

async function loadPdf(url: string) {
  return await getDocument({ url }).promise;
}
```

**Why**: `getDocument()` immediately begins worker initialization. If `workerSrc` is not set at that point, PDF.js either throws an error or falls back to fake worker mode (main thread), causing severe performance degradation.

---

## 3. Using @latest or Floating Version in CDN URL

**Severity**: High -- causes intermittent version mismatches after CDN cache updates.

### Wrong

```typescript
import { GlobalWorkerOptions } from "pdfjs-dist";

// BROKEN: "latest" resolves to whatever version the CDN has cached
GlobalWorkerOptions.workerSrc =
  "https://unpkg.com/pdfjs-dist@latest/build/pdf.worker.min.mjs";

// ALSO BROKEN: Semver range can resolve to different version than installed
GlobalWorkerOptions.workerSrc =
  "https://cdn.jsdelivr.net/npm/pdfjs-dist@^5.0.0/build/pdf.worker.min.mjs";
```

### Correct

```typescript
import { GlobalWorkerOptions, version } from "pdfjs-dist";

// ALWAYS pin to the exact installed version
GlobalWorkerOptions.workerSrc =
  `https://unpkg.com/pdfjs-dist@${version}/build/pdf.worker.min.mjs`;
```

**Why**: CDN caches update independently from your application. When `@latest` resolves to a newer version than your installed pdfjs-dist, the version mismatch error appears. This is especially insidious because it works during development and breaks in production days later when the CDN updates.

---

## 4. Mixing Module Formats (.mjs API with .js Worker)

**Severity**: High -- causes silent worker loading failures or cryptic errors.

### Wrong

```typescript
// Application uses ES modules
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist"; // Uses pdf.mjs

// BROKEN: Loading IIFE worker with ES module API
GlobalWorkerOptions.workerSrc = "/pdf.worker.min.js"; // IIFE format
```

### Correct

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";

// ALWAYS match module format: .mjs API needs .mjs worker
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();
```

**Why**: The ES module build (`pdf.min.mjs`) and the IIFE build (`pdf.min.js`) may have subtle differences in their worker communication protocol. Mixing formats can cause the worker handshake to fail silently, resulting in documents that never load.

---

## 5. Not Handling Worker Destruction Race Conditions

**Severity**: Medium -- causes `Worker was destroyed` errors during component unmount.

### Wrong

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";
import type { PDFDocumentProxy } from "pdfjs-dist";

// BROKEN: No cleanup on component unmount
class PdfViewer {
  private doc: PDFDocumentProxy | null = null;

  async load(url: string): void {
    this.doc = await getDocument({ url }).promise;
  }

  // Called when component is removed from DOM
  destroy(): void {
    // BROKEN: What if load() is still in progress?
    // The promise will reject with "Worker was destroyed"
    this.doc?.destroy();
  }
}
```

### Correct

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";
import type { PDFDocumentProxy, PDFDocumentLoadingTask } from "pdfjs-dist";

class PdfViewer {
  private doc: PDFDocumentProxy | null = null;
  private loadingTask: PDFDocumentLoadingTask | null = null;
  private destroyed = false;

  async load(url: string): Promise<void> {
    // Cancel any previous loading
    if (this.loadingTask) {
      this.loadingTask.destroy();
      this.loadingTask = null;
    }

    this.loadingTask = getDocument({ url });

    try {
      this.doc = await this.loadingTask.promise;
    } catch (err: unknown) {
      if (this.destroyed) return; // Expected during teardown
      throw err;
    } finally {
      this.loadingTask = null;
    }
  }

  async destroy(): Promise<void> {
    this.destroyed = true;

    // Cancel in-flight loading first
    if (this.loadingTask) {
      this.loadingTask.destroy();
      this.loadingTask = null;
    }

    // Then destroy the loaded document
    if (this.doc) {
      await this.doc.destroy();
      this.doc = null;
    }
  }
}
```

**Why**: In SPA frameworks (React, Vue, Angular), components can be destroyed while PDF loading is still in progress. Without proper cancellation, the pending promise rejects with "Worker was destroyed", which appears as an unhandled promise rejection in production error tracking.

---

## 6. Reusing a Worker Port for Multiple PDFWorker Instances

**Severity**: Medium -- causes `Cannot use more than one PDFWorker per port`.

### Wrong

```typescript
import { GlobalWorkerOptions } from "pdfjs-dist";

// BROKEN: Sharing a single Worker port across multiple document loads
const sharedWorker = new Worker("/pdf.worker.min.mjs", { type: "module" });
GlobalWorkerOptions.workerPort = sharedWorker;

// First load works fine
const doc1 = await getDocument({ url: "a.pdf" }).promise;

// Second load fails: "Cannot use more than one PDFWorker per port"
const doc2 = await getDocument({ url: "b.pdf" }).promise;
```

### Correct

```typescript
import { GlobalWorkerOptions } from "pdfjs-dist";

// CORRECT: Use workerSrc instead of workerPort for multiple documents
// PDF.js creates a new Worker internally for each document
GlobalWorkerOptions.workerSrc = "/pdf.worker.min.mjs";

const doc1 = await getDocument({ url: "a.pdf" }).promise;
const doc2 = await getDocument({ url: "b.pdf" }).promise; // Works fine
```

**Why**: Each PDFWorker instance needs exclusive control over its Worker port for message passing. When `workerPort` is set globally, every `getDocument()` call tries to create a PDFWorker using the same port, causing a conflict after the first one. Use `workerSrc` instead, which lets PDF.js create a fresh Worker for each document.

---

## 7. Ignoring Worker Errors in Production

**Severity**: High -- users see blank pages with no error information.

### Wrong

```typescript
// BROKEN: No error handling -- user sees a blank page
async function showPdf(url: string, canvas: HTMLCanvasElement) {
  const doc = await getDocument({ url }).promise;
  const page = await doc.getPage(1);
  const viewport = page.getViewport({ scale: 1.5 });
  const ctx = canvas.getContext("2d")!;
  await page.render({ canvasContext: ctx, viewport }).promise;
}
```

### Correct

```typescript
import { getDocument, GlobalWorkerOptions, version } from "pdfjs-dist";

async function showPdf(
  url: string,
  canvas: HTMLCanvasElement,
  onError: (message: string) => void
) {
  try {
    const doc = await getDocument({ url }).promise;
    const page = await doc.getPage(1);
    const viewport = page.getViewport({ scale: 1.5 });
    const dpr = window.devicePixelRatio || 1;

    canvas.width = Math.floor(viewport.width * dpr);
    canvas.height = Math.floor(viewport.height * dpr);
    canvas.style.width = `${Math.floor(viewport.width)}px`;
    canvas.style.height = `${Math.floor(viewport.height)}px`;

    const ctx = canvas.getContext("2d")!;
    ctx.scale(dpr, dpr);

    await page.render({ canvasContext: ctx, viewport }).promise;
  } catch (err: unknown) {
    if (!(err instanceof Error)) {
      onError("Unknown PDF loading error");
      return;
    }

    if (err.message.includes("does not match the Worker version")) {
      onError(
        "PDF viewer version conflict. Please refresh the page. " +
        `(API: ${version}, Worker: ${GlobalWorkerOptions.workerSrc})`
      );
    } else if (err.message.includes("GlobalWorkerOptions.workerSrc")) {
      onError("PDF viewer is not properly configured.");
    } else if (err.message.includes("fake worker failed")) {
      onError("PDF viewer failed to initialize. Please try a different browser.");
    } else {
      onError(`Failed to load PDF: ${err.message}`);
    }
  }
}
```

**Why**: Worker errors prevent PDF rendering entirely. Without error handling, users see a blank page with no indication of what went wrong. ALWAYS catch worker-related errors and display actionable messages that help both users and developers diagnose the issue.

---

## 8. Relative Worker Path That Breaks with Client-Side Routing

**Severity**: Medium -- worker loads on some pages but fails on others.

### Wrong

```typescript
// BROKEN: Relative path resolves differently based on current URL
GlobalWorkerOptions.workerSrc = "./pdf.worker.min.mjs";

// On https://app.com/ → resolves to https://app.com/pdf.worker.min.mjs ✓
// On https://app.com/docs/123 → resolves to https://app.com/docs/pdf.worker.min.mjs ✗ (404)
```

### Correct

```typescript
// ALWAYS use absolute paths or import.meta.url resolution
GlobalWorkerOptions.workerSrc = "/pdf.worker.min.mjs"; // Absolute from root

// OR use import.meta.url which resolves based on module location, not page URL
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();
```

**Why**: Relative URLs resolve based on the browser's current URL, which changes with client-side routing. A worker that loads successfully on the homepage will 404 on a nested route. ALWAYS use absolute paths or `import.meta.url` resolution to ensure the worker loads regardless of the current route.
