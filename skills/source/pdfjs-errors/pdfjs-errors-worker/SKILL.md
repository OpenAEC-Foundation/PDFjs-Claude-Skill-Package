---
name: pdfjs-errors-worker
description: "Diagnoses and fixes PDF.js Web Worker errors. Covers version mismatch errors, worker loading failures, CORS issues, CSP violations, fake worker fallback, and worker initialization problems. Activates when PDF.js worker fails to load, version mismatch error appears, or worker-related errors occur."
license: MIT
compatibility: "Designed for Claude Code. Requires pdfjs-dist 5.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# pdfjs-errors-worker

## Quick Reference

### Worker Error Types

| Error Message | Likely Cause | Severity |
|---------------|-------------|----------|
| `The API version "X" does not match the Worker version "Y"` | Mismatched pdfjs-dist files | Critical |
| `No "GlobalWorkerOptions.workerSrc" specified` | Missing worker configuration | Critical |
| `Setting up fake worker failed` | Fake worker import failure | High |
| `Failed to construct 'Worker'` (browser) | CORS or CSP blocking worker | High |
| `Worker was destroyed` | Premature worker termination | Medium |
| `Cannot use more than one PDFWorker per port` | Port reuse conflict | Medium |
| Network error / 404 on worker file | Wrong workerSrc path | Critical |

### Critical Warnings

**ALWAYS** set `GlobalWorkerOptions.workerSrc` BEFORE calling `getDocument()` -- PDF.js throws `No "GlobalWorkerOptions.workerSrc" specified` if the worker source is not configured when a document load begins.

**ALWAYS** ensure the worker file version matches the pdfjs-dist package version EXACTLY -- even a patch version difference (e.g., API 5.0.1 vs Worker 5.0.0) triggers the version mismatch error.

**NEVER** load the worker file from a different CDN version than your installed pdfjs-dist -- this is the #1 cause of version mismatch errors.

**NEVER** use `pdf.worker.js` (CommonJS) when your application uses ES modules -- use `pdf.worker.min.mjs` instead. Mixing module formats causes silent loading failures.

**ALWAYS** self-host the worker file or use the exact CDN URL matching your pdfjs-dist version -- CDN URLs with "latest" or floating version tags cause intermittent version mismatches after updates.

---

## Diagnostic Decision Tree

```
PDF.js worker error?
│
├── "The API version X does not match the Worker version Y"
│   ├── Using CDN? → Ensure CDN URL version matches pdfjs-dist version exactly
│   ├── Using bundler? → Check that worker file is NOT bundled separately with different version
│   └── Using copy-webpack-plugin? → Verify it copies from correct node_modules/pdfjs-dist path
│
├── "No GlobalWorkerOptions.workerSrc specified"
│   ├── workerSrc set AFTER getDocument()? → Move workerSrc assignment BEFORE getDocument()
│   ├── workerSrc not set at all? → Add GlobalWorkerOptions.workerSrc configuration
│   └── Using workerPort instead? → Verify Worker instance is valid
│
├── 404 / Network Error loading worker
│   ├── Wrong path? → Check workerSrc resolves to actual file location
│   ├── File not copied to public dir? → Copy worker from node_modules to public/static dir
│   └── Bundler not serving it? → Configure bundler to serve .mjs files
│
├── CORS error loading worker
│   ├── CDN worker file? → Use CDN that serves proper CORS headers (cdnjs, unpkg, jsdelivr)
│   ├── Different origin? → Self-host the worker file on same origin
│   └── Still failing? → Use workerPort with Blob URL wrapper (see examples)
│
├── CSP violation blocking worker
│   ├── worker-src missing? → Add worker-src directive to Content-Security-Policy
│   ├── Using Blob URL? → Add blob: to worker-src
│   └── Strict CSP? → Self-host worker and add your domain to worker-src
│
├── "Setting up fake worker failed"
│   ├── Dynamic import blocked? → Check CSP script-src allows dynamic imports
│   ├── Module path wrong? → Verify pdfjs-dist is importable at runtime
│   └── Bundler issue? → Ensure bundler includes the worker code in the bundle
│
├── "Worker was destroyed"
│   ├── Called destroy() too early? → Await all pending operations before destroying
│   ├── Component unmounted? → Cancel loading tasks on unmount before destroy
│   └── Multiple destroy() calls? → Guard with a destroyed flag
│
└── Worker loads but PDF rendering fails silently
    ├── Worker loaded but wrong build? → Ensure pdf.worker.min.mjs matches pdf.min.mjs
    ├── Mixed legacy/modern builds? → Use ONLY the modern (.mjs) or ONLY legacy (.js) build
    └── Check browser console → Worker errors may appear only in worker thread console
```

---

## Essential Fixes

### Fix 1: Version Mismatch Error

**Error**: `The API version "5.0.0" does not match the Worker version "4.9.155"`

**Cause**: The worker file loaded from CDN or disk is a different version than the pdfjs-dist package installed in node_modules.

```typescript
import { GlobalWorkerOptions, version } from "pdfjs-dist";

// WRONG: Hardcoded CDN version that will drift from installed package
// GlobalWorkerOptions.workerSrc = "https://cdnjs.cloudflare.com/ajax/libs/pdf.js/4.9.155/pdf.worker.min.mjs";

// CORRECT: Use import.meta.url to resolve from the installed package
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

// ALTERNATIVE: Pin CDN URL to the exact installed version
GlobalWorkerOptions.workerSrc =
  `https://cdnjs.cloudflare.com/ajax/libs/pdf.js/${version}/pdf.worker.min.mjs`;
```

**Prevention**: ALWAYS use `import.meta.url` resolution or interpolate the `version` export from pdfjs-dist into CDN URLs. NEVER hardcode version numbers in worker URLs.

### Fix 2: Worker Loading Failure (404 / Path Error)

**Error**: Network request for worker file returns 404 or fails to load.

**Cause**: The workerSrc path does not resolve to the actual worker file at runtime.

```typescript
import { GlobalWorkerOptions } from "pdfjs-dist";

// WRONG: Relative path that breaks depending on app routing
// GlobalWorkerOptions.workerSrc = "./pdf.worker.min.mjs";

// CORRECT for Vite: Use URL constructor with import.meta.url
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

// CORRECT for Webpack: Copy worker to public dir and reference it
// (requires copy-webpack-plugin or manual copy)
GlobalWorkerOptions.workerSrc = "/pdf.worker.min.mjs";
```

### Fix 3: CORS Error Loading Worker from CDN

**Error**: `Failed to construct 'Worker': Script at 'https://cdn...' cannot be accessed from origin 'https://myapp.com'`

**Cause**: Browsers block Worker creation from cross-origin scripts unless the server sends proper CORS headers.

```typescript
import { GlobalWorkerOptions } from "pdfjs-dist";

// SOLUTION 1: Self-host the worker file (ALWAYS works)
GlobalWorkerOptions.workerSrc = "/assets/pdf.worker.min.mjs";

// SOLUTION 2: Use a Blob URL wrapper for cross-origin workers
function createWorkerBlobURL(cdnUrl: string): string {
  const workerCode = `importScripts("${cdnUrl}");`;
  const blob = new Blob([workerCode], { type: "application/javascript" });
  return URL.createObjectURL(blob);
}

GlobalWorkerOptions.workerSrc = createWorkerBlobURL(
  "https://cdnjs.cloudflare.com/ajax/libs/pdf.js/5.0.0/pdf.worker.min.js"
);
```

**Note**: The Blob URL approach requires `blob:` in the CSP `worker-src` directive. ALWAYS prefer self-hosting when possible.

### Fix 4: CSP Violation Blocking Worker

**Error**: `Refused to create a worker from 'blob:...' because it violates the Content-Security-Policy directive: "worker-src 'self'"`

**Cause**: The Content-Security-Policy header does not allow worker creation from the specified source.

```html
<!-- WRONG: Missing worker-src directive -->
<meta http-equiv="Content-Security-Policy"
      content="default-src 'self'">

<!-- CORRECT: Add worker-src with required sources -->
<meta http-equiv="Content-Security-Policy"
      content="default-src 'self'; worker-src 'self' blob:">

<!-- CORRECT: If loading from CDN -->
<meta http-equiv="Content-Security-Policy"
      content="default-src 'self'; worker-src 'self' blob: https://cdnjs.cloudflare.com">
```

### Fix 5: Fake Worker Fallback

**When to use**: Environments where Web Workers are unavailable (some testing frameworks, restricted iframes, older environments).

```typescript
import { GlobalWorkerOptions } from "pdfjs-dist";

// Force fake worker mode by NOT setting workerSrc
// and ensuring no workerPort is set.
// PDF.js will fall back to running worker code on the main thread.

// ALTERNATIVE: Explicitly disable worker via workerPort
GlobalWorkerOptions.workerSrc = "";
```

**Performance warning**: Fake worker mode runs all PDF parsing on the main thread. This blocks the UI during document loading and rendering. NEVER use fake worker mode in production unless Web Workers are truly unavailable. Expect 2-5x slower document loading and potential UI freezing on large PDFs.

---

## Prevention Patterns

### Version-Pinned Worker Setup

```typescript
import { GlobalWorkerOptions, version } from "pdfjs-dist";

// Pattern 1: Import-based resolution (RECOMMENDED for bundlers)
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

// Pattern 2: Version-interpolated CDN (RECOMMENDED for CDN usage)
GlobalWorkerOptions.workerSrc =
  `https://cdnjs.cloudflare.com/ajax/libs/pdf.js/${version}/pdf.worker.min.mjs`;

// Pattern 3: Verify at runtime (RECOMMENDED for debugging)
console.log(`PDF.js API version: ${version}`);
console.log(`Worker source: ${GlobalWorkerOptions.workerSrc}`);
```

### Worker Initialization Guard

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";

let workerConfigured = false;

function ensureWorkerConfigured(): void {
  if (workerConfigured) return;

  GlobalWorkerOptions.workerSrc = new URL(
    "pdfjs-dist/build/pdf.worker.min.mjs",
    import.meta.url
  ).toString();

  workerConfigured = true;
}

// ALWAYS call before any document loading
async function loadPdf(url: string) {
  ensureWorkerConfigured();
  return getDocument({ url }).promise;
}
```

---

## Reference Links

- [references/methods.md](references/methods.md) -- GlobalWorkerOptions API, PDFWorker class, and error type reference
- [references/examples.md](references/examples.md) -- Error recovery patterns for every worker error type
- [references/anti-patterns.md](references/anti-patterns.md) -- Common causes of each worker error

### Official Sources

- https://mozilla.github.io/pdf.js/api/ -- PDF.js API reference
- https://github.com/mozilla/pdf.js -- Source code (worker error messages)
- https://github.com/mozilla/pdf.js/tree/master/examples -- Official examples with worker setup
