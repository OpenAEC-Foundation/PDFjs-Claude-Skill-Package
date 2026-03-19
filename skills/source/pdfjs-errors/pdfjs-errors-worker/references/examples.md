# Worker Error Recovery Patterns (pdfjs-dist 5.x)

## 1. Version Mismatch Recovery

Detect and recover from version mismatch errors at runtime.

```typescript
import { getDocument, GlobalWorkerOptions, version } from "pdfjs-dist";

// ALWAYS log version info during development
console.log(`PDF.js API version: ${version}`);

// RECOMMENDED: Use import.meta.url to guarantee version match
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

async function loadPdfWithVersionCheck(url: string) {
  try {
    return await getDocument({ url }).promise;
  } catch (err: unknown) {
    if (
      err instanceof Error &&
      err.message.includes("does not match the Worker version")
    ) {
      console.error(
        "Version mismatch detected. API version:",
        version,
        "Worker source:",
        GlobalWorkerOptions.workerSrc
      );
      // Recovery: Switch to version-interpolated CDN URL
      GlobalWorkerOptions.workerSrc =
        `https://cdnjs.cloudflare.com/ajax/libs/pdf.js/${version}/pdf.worker.min.mjs`;
      return await getDocument({ url }).promise;
    }
    throw err;
  }
}
```

---

## 2. Worker Loading Failure with Fallback Chain

Try multiple worker sources in order of preference.

```typescript
import { getDocument, GlobalWorkerOptions, version } from "pdfjs-dist";

const WORKER_SOURCES = [
  // Priority 1: Local bundled file
  () =>
    new URL("pdfjs-dist/build/pdf.worker.min.mjs", import.meta.url).toString(),
  // Priority 2: Self-hosted copy
  () => "/assets/pdf.worker.min.mjs",
  // Priority 3: CDN with version pin
  () =>
    `https://cdnjs.cloudflare.com/ajax/libs/pdf.js/${version}/pdf.worker.min.mjs`,
  // Priority 4: Alternative CDN
  () =>
    `https://cdn.jsdelivr.net/npm/pdfjs-dist@${version}/build/pdf.worker.min.mjs`,
];

async function loadPdfWithFallback(url: string) {
  for (const getWorkerSrc of WORKER_SOURCES) {
    try {
      GlobalWorkerOptions.workerSrc = getWorkerSrc();
      return await getDocument({ url }).promise;
    } catch (err: unknown) {
      console.warn(
        `Worker source failed: ${GlobalWorkerOptions.workerSrc}`,
        err
      );
      continue;
    }
  }
  throw new Error("All worker sources failed. Cannot load PDF.");
}
```

---

## 3. CORS-Safe Worker Loading

Handle cross-origin worker loading using a Blob URL wrapper.

```typescript
import { GlobalWorkerOptions, version } from "pdfjs-dist";

/**
 * Creates a same-origin Blob URL that loads a cross-origin worker script.
 * Uses importScripts() which is NOT subject to same-origin restrictions.
 *
 * IMPORTANT: This requires 'blob:' in the CSP worker-src directive.
 * IMPORTANT: This approach only works with IIFE (.js) worker files,
 * NOT with ES module (.mjs) files.
 */
function createCorsWorkerUrl(workerUrl: string): string {
  const workerCode = `importScripts("${workerUrl}");`;
  const blob = new Blob([workerCode], { type: "application/javascript" });
  return URL.createObjectURL(blob);
}

// Usage with IIFE worker file (NOT .mjs)
const cdnUrl = `https://cdnjs.cloudflare.com/ajax/libs/pdf.js/${version}/pdf.worker.min.js`;
GlobalWorkerOptions.workerSrc = createCorsWorkerUrl(cdnUrl);

// ALWAYS clean up the Blob URL when the application shuts down
// to prevent memory leaks
function cleanup(): void {
  if (GlobalWorkerOptions.workerSrc.startsWith("blob:")) {
    URL.revokeObjectURL(GlobalWorkerOptions.workerSrc);
  }
}
```

---

## 4. CSP-Compatible Worker Setup

Configure worker loading for strict Content-Security-Policy environments.

```typescript
import { GlobalWorkerOptions } from "pdfjs-dist";

/**
 * For strict CSP environments, self-host the worker file
 * and reference it from the same origin.
 *
 * Required CSP header:
 *   Content-Security-Policy: worker-src 'self';
 *
 * If using Blob URLs:
 *   Content-Security-Policy: worker-src 'self' blob:;
 */

// RECOMMENDED: Self-hosted worker (simplest CSP)
GlobalWorkerOptions.workerSrc = "/assets/pdf.worker.min.mjs";

// ALTERNATIVE: Use workerPort with explicit Worker creation
// This gives full control over Worker options
const worker = new Worker("/assets/pdf.worker.min.mjs", {
  type: "module",
  name: "pdf.js-worker",
});
GlobalWorkerOptions.workerPort = worker;
```

**CSP Header Examples**:

```
# Self-hosted worker only
Content-Security-Policy: worker-src 'self';

# Self-hosted + Blob URL fallback
Content-Security-Policy: worker-src 'self' blob:;

# Self-hosted + specific CDN
Content-Security-Policy: worker-src 'self' https://cdnjs.cloudflare.com;
```

---

## 5. Fake Worker Mode (Testing / Restricted Environments)

Run PDF.js without a Web Worker when Workers are unavailable.

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";

/**
 * Fake worker mode: PDF.js runs all parsing on the main thread.
 *
 * NEVER use in production -- it blocks the UI thread during:
 * - Document loading (parsing PDF structure)
 * - Page rendering (decoding images, processing fonts)
 * - Text extraction
 *
 * Performance impact:
 * - Small PDFs (< 1 MB): ~2x slower, minor UI lag
 * - Medium PDFs (1-10 MB): ~3x slower, noticeable freezing
 * - Large PDFs (> 10 MB): ~5x slower, UI becomes unresponsive
 *
 * Use ONLY for:
 * - Unit tests (Jest, Vitest) where Worker is not available
 * - Restricted iframes without Worker permission
 * - Server-side rendering (SSR) environments
 */

// Trigger fake worker by not setting workerSrc
// PDF.js automatically falls back to main-thread execution
async function loadPdfFakeWorker(source: ArrayBuffer) {
  // Ensure no workerSrc is set
  GlobalWorkerOptions.workerSrc = "";

  const doc = await getDocument({ data: source }).promise;
  return doc;
}
```

---

## 6. Worker Initialization Timeout Handling

Detect and handle worker initialization that takes too long.

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";
import type { PDFDocumentProxy } from "pdfjs-dist";

GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

/**
 * Load a PDF with a timeout on the entire operation.
 * Worker initialization problems often manifest as hangs
 * rather than explicit errors.
 *
 * Common causes of worker timeouts:
 * - Worker file is very large and takes time to download
 * - Network is slow or intermittent
 * - Worker script has a syntax error (fails silently)
 * - CSP blocks the worker but error is swallowed
 */
async function loadPdfWithTimeout(
  url: string,
  timeoutMs: number = 15000
): Promise<PDFDocumentProxy> {
  const loadingTask = getDocument({ url });

  const timeoutPromise = new Promise<never>((_, reject) => {
    setTimeout(() => {
      loadingTask.destroy();
      reject(new Error(
        `PDF worker initialization timed out after ${timeoutMs}ms. ` +
        "Check: (1) worker file URL is correct, (2) no CORS/CSP blocking, " +
        "(3) network connectivity."
      ));
    }, timeoutMs);
  });

  return Promise.race([loadingTask.promise, timeoutPromise]);
}
```

---

## 7. Robust Worker Setup for Production

Complete production-ready worker initialization with diagnostics.

```typescript
import { getDocument, GlobalWorkerOptions, version } from "pdfjs-dist";
import type { PDFDocumentProxy } from "pdfjs-dist";

interface WorkerDiagnostics {
  apiVersion: string;
  workerSrc: string;
  workerMode: "dedicated" | "port" | "fake";
  timestamp: string;
}

function configureWorker(): WorkerDiagnostics {
  // ALWAYS use import.meta.url for version-safe resolution
  GlobalWorkerOptions.workerSrc = new URL(
    "pdfjs-dist/build/pdf.worker.min.mjs",
    import.meta.url
  ).toString();

  const diagnostics: WorkerDiagnostics = {
    apiVersion: version,
    workerSrc: GlobalWorkerOptions.workerSrc,
    workerMode: GlobalWorkerOptions.workerPort ? "port" : "dedicated",
    timestamp: new Date().toISOString(),
  };

  return diagnostics;
}

async function loadPdfRobust(url: string): Promise<PDFDocumentProxy> {
  const diag = configureWorker();

  try {
    const doc = await getDocument({ url }).promise;
    return doc;
  } catch (err: unknown) {
    if (!(err instanceof Error)) throw err;

    // Provide actionable error messages
    if (err.message.includes("does not match the Worker version")) {
      throw new Error(
        `PDF.js version mismatch. API: ${diag.apiVersion}, ` +
        `Worker: ${diag.workerSrc}. ` +
        "Ensure worker file comes from the same pdfjs-dist version."
      );
    }

    if (err.message.includes("GlobalWorkerOptions.workerSrc")) {
      throw new Error(
        "PDF.js worker not configured. Call configureWorker() before loading PDFs."
      );
    }

    if (err.message.includes("fake worker failed")) {
      throw new Error(
        "PDF.js fake worker fallback failed. " +
        "Ensure pdfjs-dist is installed correctly and importable. " +
        `Diagnostics: ${JSON.stringify(diag)}`
      );
    }

    // Unknown error -- attach diagnostics
    throw new Error(
      `PDF loading failed: ${err.message}. ` +
      `Diagnostics: ${JSON.stringify(diag)}`
    );
  }
}
```

---

## 8. Bundler-Specific Worker Configuration

### Vite

```typescript
// vite.config.ts -- Vite handles .mjs imports natively
// No special configuration needed. Use import.meta.url:
import { GlobalWorkerOptions } from "pdfjs-dist";

GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();
```

### Webpack 5

```typescript
// webpack.config.js
const CopyPlugin = require("copy-webpack-plugin");

module.exports = {
  plugins: [
    new CopyPlugin({
      patterns: [
        {
          from: "node_modules/pdfjs-dist/build/pdf.worker.min.mjs",
          to: "pdf.worker.min.mjs",
        },
      ],
    }),
  ],
};

// In your application code:
import { GlobalWorkerOptions } from "pdfjs-dist";
GlobalWorkerOptions.workerSrc = "/pdf.worker.min.mjs";
```

### Next.js

```typescript
// next.config.js
const CopyPlugin = require("copy-webpack-plugin");

module.exports = {
  webpack: (config) => {
    config.plugins.push(
      new CopyPlugin({
        patterns: [
          {
            from: "node_modules/pdfjs-dist/build/pdf.worker.min.mjs",
            to: "../public/pdf.worker.min.mjs",
          },
        ],
      })
    );
    return config;
  },
};

// In your component:
import { GlobalWorkerOptions } from "pdfjs-dist";
GlobalWorkerOptions.workerSrc = "/pdf.worker.min.mjs";
```
