# Bundler Integration Anti-Patterns (pdfjs-dist 5.x)

Common mistakes when integrating pdfjs-dist with bundlers and their fixes.

---

## 1. Hardcoded Worker Path

**Severity**: Critical -- worker fails to load in production.

### Wrong

```typescript
import { GlobalWorkerOptions } from "pdfjs-dist";

// BROKEN: Hardcoded path does not survive bundling
// Bundlers change output filenames (hashing, chunk splitting)
GlobalWorkerOptions.workerSrc = "/pdf.worker.js";
```

### Correct

```typescript
// Option A: Use webpack.mjs (zero-config, handles worker automatically)
import * as pdfjsLib from "pdfjs-dist/webpack.mjs";

// Option B: Use import.meta.url for dynamic resolution
import { GlobalWorkerOptions } from "pdfjs-dist";
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();
```

**Why**: Bundlers rename and hash output files. A hardcoded path like `"/pdf.worker.js"` only works if you manually copy the file AND never change the output path. The `import.meta.url` pattern lets the bundler resolve the correct path at build time.

---

## 2. Importing Worker into Main Bundle

**Severity**: Critical -- blocks UI thread, defeats purpose of Web Worker.

### Wrong

```typescript
// BROKEN: This imports the worker CODE into the main bundle
// It runs on the main thread, blocking the UI
import "pdfjs-dist/build/pdf.worker.mjs";

import { getDocument } from "pdfjs-dist";
const doc = await getDocument("/document.pdf").promise;
```

### Correct

```typescript
// The worker MUST be loaded as a separate file in a Worker thread
import * as pdfjsLib from "pdfjs-dist/webpack.mjs";

// webpack.mjs creates: new Worker(new URL("./build/pdf.worker.mjs", import.meta.url))
// This runs the worker in a separate thread
const doc = await pdfjsLib.getDocument("/document.pdf").promise;
```

**Why**: The PDF.js worker performs heavy PDF parsing and rendering calculations. If imported into the main bundle, all this work runs on the main thread, causing the UI to freeze during PDF operations.

---

## 3. Missing optimizeDeps.exclude in Vite

**Severity**: High -- worker fails to load after Vite pre-bundles pdfjs-dist.

### Wrong

```typescript
// vite.config.ts
import { defineConfig } from "vite";

export default defineConfig({
  // No optimizeDeps configuration
  // Vite pre-bundles pdfjs-dist, breaking import.meta.url resolution
});
```

### Correct

```typescript
// vite.config.ts
import { defineConfig } from "vite";

export default defineConfig({
  optimizeDeps: {
    exclude: ["pdfjs-dist"],
  },
});
```

**Why**: Vite's dependency pre-bundling transforms `import.meta.url` references, breaking the worker URL resolution. Excluding pdfjs-dist from pre-bundling preserves the original `new URL()` pattern that Vite's production build handles correctly.

---

## 4. Not Copying CMap Files

**Severity**: High -- CJK PDFs render with missing or garbled characters.

### Wrong

```typescript
// No CMap files in the static/public directory
// No cMapUrl configured
const doc = await pdfjsLib.getDocument("/chinese-document.pdf").promise;
// Chinese characters appear as squares or are missing entirely
```

### Correct

```typescript
// 1. Copy cmaps/ to your public directory (see webpack/vite configs)
// 2. Configure cMapUrl when loading the document
const doc = await pdfjsLib.getDocument({
  url: "/chinese-document.pdf",
  cMapUrl: "/cmaps/",
  cMapPacked: true, // ALWAYS true -- pdfjs-dist ships .bcmap (binary packed)
}).promise;
```

**Why**: CJK (Chinese, Japanese, Korean) PDFs use character maps to translate character codes to Unicode. These `.bcmap` files are 169 binary files that cannot be bundled -- they must be served as static assets and fetched at runtime by the worker.

---

## 5. Setting cMapPacked to false

**Severity**: Medium -- CMap loading fails silently or errors.

### Wrong

```typescript
const doc = await pdfjsLib.getDocument({
  url: "/document.pdf",
  cMapUrl: "/cmaps/",
  cMapPacked: false, // BROKEN: pdfjs-dist ships binary .bcmap files, not plain text
}).promise;
```

### Correct

```typescript
const doc = await pdfjsLib.getDocument({
  url: "/document.pdf",
  cMapUrl: "/cmaps/",
  cMapPacked: true, // ALWAYS true when using pdfjs-dist
}).promise;
```

**Why**: The pdfjs-dist package ships binary-compressed `.bcmap` files. Setting `cMapPacked: false` tells PDF.js to expect plain-text `.cmap` files, which do not exist in the package.

---

## 6. Using .js Extension Instead of .mjs

**Severity**: High -- import fails, module not found.

### Wrong

```typescript
// BROKEN: pdfjs-dist 5.x ships .mjs files only (modern build)
GlobalWorkerOptions.workerSrc = "pdfjs-dist/build/pdf.worker.js";
// Error: Module not found
```

### Correct

```typescript
// pdfjs-dist 5.x uses .mjs extensions for ESM modules
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();
```

**Why**: pdfjs-dist 5.x dropped `.js` files from the modern build. All files use `.mjs` extension. The legacy build at `pdfjs-dist/legacy/build/` also uses `.mjs`. There are no `.js` or `.cjs` files.

---

## 7. Importing pdfjs-dist at Top Level in SSR Frameworks

**Severity**: Critical -- server-side error, application crashes.

### Wrong (Next.js)

```typescript
// BROKEN: Top-level import runs on the server during SSR
// pdfjs-dist uses browser APIs (Worker, Canvas, window) that don't exist on the server
import * as pdfjsLib from "pdfjs-dist";

export default function PdfViewer() {
  // ReferenceError: Worker is not defined (server-side)
}
```

### Correct (Next.js)

```typescript
"use client";

import { useEffect } from "react";

export default function PdfViewer({ url }: { url: string }) {
  useEffect(() => {
    async function render() {
      // ALWAYS use dynamic import in SSR frameworks
      const pdfjsLib = await import("pdfjs-dist");
      pdfjsLib.GlobalWorkerOptions.workerSrc = "/pdf.worker.min.mjs";
      // ... render logic
    }
    render();
  }, [url]);
}
```

**Why**: pdfjs-dist is a browser-only library. It references `window`, `Worker`, `document`, and `Canvas` at module evaluation time. In SSR frameworks (Next.js, Nuxt.js), modules imported at the top level run on the server first, where these browser APIs do not exist.

---

## 8. Version Mismatch Between Library and Worker

**Severity**: Critical -- silent failures, corrupted rendering.

### Wrong

```html
<!-- BROKEN: Library is v5.5.207 but worker is v4.x -->
<script type="module">
  import * as pdfjsLib from "https://cdn.jsdelivr.net/npm/pdfjs-dist@5.5.207/build/pdf.min.mjs";
  pdfjsLib.GlobalWorkerOptions.workerSrc =
    "https://cdn.jsdelivr.net/npm/pdfjs-dist@4.0.0/build/pdf.worker.min.mjs";
</script>
```

### Correct

```html
<!-- ALWAYS use the same version for library and worker -->
<script type="module">
  import * as pdfjsLib from "https://cdn.jsdelivr.net/npm/pdfjs-dist@5.5.207/build/pdf.min.mjs";
  pdfjsLib.GlobalWorkerOptions.workerSrc =
    "https://cdn.jsdelivr.net/npm/pdfjs-dist@5.5.207/build/pdf.worker.min.mjs";
</script>
```

**Why**: The main library and worker communicate via a message protocol. Different versions have different message formats, causing silent failures, incorrect rendering, or outright errors.

---

## 9. Forgetting Worker Setup Entirely

**Severity**: High -- falls back to fake worker (main thread), causes UI freezing.

### Wrong

```typescript
import { getDocument } from "pdfjs-dist";

// No worker configured at all
// PDF.js falls back to running worker code on the main thread
const doc = await getDocument("/large-document.pdf").promise;
// UI freezes during PDF parsing of large documents
```

### Correct

```typescript
import * as pdfjsLib from "pdfjs-dist/webpack.mjs";

// Worker is auto-configured -- heavy work runs in separate thread
const doc = await pdfjsLib.getDocument("/large-document.pdf").promise;
// UI stays responsive
```

**Why**: Without worker configuration, pdfjs-dist 5.x falls back to `"./pdf.worker.mjs"` as the default workerSrc, which usually fails to resolve. If it fails, PDF.js silently falls back to a "fake worker" that runs all parsing on the main thread. This works but causes UI freezing for large PDFs.

---

## 10. Using Webpack 4 with webpack.mjs

**Severity**: Critical -- build fails.

### Wrong

```typescript
// BROKEN: Webpack 4 does not support new URL(..., import.meta.url)
import * as pdfjsLib from "pdfjs-dist/webpack.mjs";
// Build error: Unexpected token 'import.meta'
```

### Correct (Webpack 4)

```javascript
// webpack.config.js -- Webpack 4 requires explicit worker entry
module.exports = {
  entry: {
    main: "./src/index.js",
    "pdf.worker": "pdfjs-dist/build/pdf.worker.min.mjs",
  },
  output: {
    filename: "[name].bundle.js",
    path: path.resolve(__dirname, "dist"),
  },
};
```

```typescript
// Application code -- set workerSrc to the explicit bundle path
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";
GlobalWorkerOptions.workerSrc = "/pdf.worker.bundle.js";
```

**Why**: The `new URL(..., import.meta.url)` pattern used in `webpack.mjs` requires webpack 5+. Webpack 4 does not understand `import.meta` and will fail at build time. For webpack 4, create a separate entry point for the worker and set `workerSrc` to the output bundle path.
