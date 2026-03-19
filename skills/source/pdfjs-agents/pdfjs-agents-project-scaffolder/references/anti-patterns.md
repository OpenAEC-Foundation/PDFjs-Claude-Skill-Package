# Project Scaffolding Anti-Patterns (pdfjs-dist 5.x)

Common mistakes when setting up a PDF.js project and how to fix them.

---

## 1. Bundling the Worker into the Main Bundle

**Severity**: Critical -- blocks the UI thread, defeats the entire purpose of the worker architecture.

### Wrong

```typescript
// BROKEN: Importing the worker directly bundles it into main.js
import "pdfjs-dist/build/pdf.worker.min.mjs";

// Or even worse:
import pdfjsWorker from "pdfjs-dist/build/pdf.worker.min.mjs";
GlobalWorkerOptions.workerSrc = pdfjsWorker;
```

### Correct

```typescript
// The worker MUST be loaded as a separate file via URL
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();
```

**Why**: The worker file contains the PDF parsing engine. If bundled into the main bundle, all PDF parsing runs on the UI thread, causing the page to freeze during document loading and rendering. The worker MUST run in a separate Web Worker thread.

---

## 2. Worker Version Mismatch

**Severity**: Critical -- causes cryptic deserialization errors and silent failures.

### Wrong

```html
<!-- BROKEN: CDN worker version does not match installed pdfjs-dist version -->
<script>
  // Installed: pdfjs-dist@5.0.375
  // Worker: loading version 4.x from CDN
  GlobalWorkerOptions.workerSrc =
    "https://cdnjs.cloudflare.com/ajax/libs/pdf.js/4.0.0/pdf.worker.min.mjs";
</script>
```

### Correct

```typescript
// ALWAYS ensure the worker version matches the installed package version
// Option 1: Use import.meta.url (bundler resolves correct version)
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

// Option 2: CDN with exact matching version
// Check: npm list pdfjs-dist → shows installed version
GlobalWorkerOptions.workerSrc =
  "https://cdnjs.cloudflare.com/ajax/libs/pdf.js/5.0.375/pdf.worker.min.mjs";
```

**Why**: The main library and worker communicate via a serialization protocol. When versions mismatch, the protocol is incompatible, causing errors like "Invalid PDF structure" or "Unable to deserialize cloned data" that do not mention version mismatch.

---

## 3. Not Configuring Worker Before getDocument()

**Severity**: Critical -- document loading fails or falls back to slow synchronous parsing.

### Wrong

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";

// BROKEN: Loading document BEFORE setting workerSrc
const doc = await getDocument("/sample.pdf").promise;

// Setting it AFTER has no effect on the already-started loading task
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();
```

### Correct

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";

// ALWAYS set workerSrc FIRST
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

// THEN load documents
const doc = await getDocument("/sample.pdf").promise;
```

**Why**: `GlobalWorkerOptions.workerSrc` is read at the time `getDocument()` is called. If not set, PDF.js either falls back to fake worker mode (synchronous, blocking UI) or throws an error depending on the version and environment.

---

## 4. Using CommonJS Build in ESM Projects

**Severity**: High -- causes bundler errors, tree-shaking failures, or runtime crashes.

### Wrong

```typescript
// BROKEN in Vite or webpack 5 with ESM:
const pdfjsLib = require("pdfjs-dist");

// Also wrong: explicitly importing the CJS build
import pdfjsLib from "pdfjs-dist/build/pdf.js"; // CJS build
```

### Correct

```typescript
// ALWAYS use the ESM imports for Vite and webpack 5+
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";
import type { PDFDocumentProxy } from "pdfjs-dist";
```

**Why**: pdfjs-dist 5.x ships both CJS and ESM builds. Vite and webpack 5 in ESM mode expect the `.mjs` entry points. Using the CJS build causes dual-package hazard errors, breaks tree-shaking, and may fail with "require is not defined" in ESM contexts.

---

## 5. Missing PDF.js Viewer CSS

**Severity**: High -- text layer is invisible, annotation layer is mispositioned.

### Wrong

```typescript
// BROKEN: No CSS imported for text/annotation layers
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";
import { TextLayer, AnnotationLayer } from "pdfjs-dist";

// Text layer renders but is invisible (default opacity is 0 in the CSS)
// Annotation layer renders but positioned incorrectly
```

### Correct

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";
import { TextLayer, AnnotationLayer } from "pdfjs-dist";

// ALWAYS import the viewer CSS when using text or annotation layers
import "pdfjs-dist/web/pdf_viewer.css";
```

Or in CSS:

```css
/* ALWAYS import when using text/annotation layers */
@import "pdfjs-dist/web/pdf_viewer.css";
```

Or via CDN (vanilla setup):

```html
<link rel="stylesheet"
  href="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/5.0.375/pdf_viewer.min.css" />
```

**Why**: The official CSS controls text span positioning, opacity, and selection behavior for the text layer, as well as annotation element positioning. Without it, the text layer appears as invisible positioned spans, and annotations render in the wrong locations.

---

## 6. Not Handling devicePixelRatio in Setup

**Severity**: High -- all PDFs render blurry on Retina and 4K displays.

### Wrong

```typescript
// BROKEN: Canvas setup without DPI scaling
const viewport = page.getViewport({ scale: 1.5 });
canvas.width = viewport.width;
canvas.height = viewport.height;

const ctx = canvas.getContext("2d")!;
await page.render({ canvasContext: ctx, viewport }).promise;
// Result: Blurry on any screen with devicePixelRatio > 1
```

### Correct

```typescript
const viewport = page.getViewport({ scale: 1.5 });
const dpr = window.devicePixelRatio || 1;

// ALWAYS scale canvas pixel dimensions by DPR
canvas.width = Math.floor(viewport.width * dpr);
canvas.height = Math.floor(viewport.height * dpr);

// ALWAYS set CSS dimensions separately
canvas.style.width = `${Math.floor(viewport.width)}px`;
canvas.style.height = `${Math.floor(viewport.height)}px`;

const ctx = canvas.getContext("2d")!;
ctx.scale(dpr, dpr);

await page.render({ canvasContext: ctx, viewport }).promise;
```

**Why**: On Retina displays (DPR = 2), a 600px CSS width canvas has 1200 physical pixels. Without DPR scaling, the canvas renders at 600 pixels and the browser stretches it to 1200, causing visible blur on all text and graphics.

---

## 7. Using PDF.js Server-Side Without Guards (Next.js)

**Severity**: Critical -- crashes the server with "document is not defined" or "window is not defined".

### Wrong

```tsx
// BROKEN: Server component trying to use PDF.js
// app/page.tsx (this is a server component by default in Next.js App Router)
import { getDocument } from "pdfjs-dist";

export default async function Page() {
  // CRASHES: getDocument uses browser APIs not available on the server
  const doc = await getDocument("/sample.pdf").promise;
  return <div>Pages: {doc.numPages}</div>;
}
```

### Correct

```tsx
// app/page.tsx -- server component delegates to client component
import dynamic from "next/dynamic";

// ALWAYS use dynamic import with ssr: false for PDF.js
const PdfViewer = dynamic(() => import("../components/PdfViewer"), {
  ssr: false,
});

export default function Page() {
  return <PdfViewer url="/sample.pdf" />;
}
```

```tsx
// components/PdfViewer.tsx -- client component
"use client";

import { useEffect, useState } from "react";
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";

// ALWAYS guard browser-only code with "use client" directive
GlobalWorkerOptions.workerSrc = "/pdf.worker.min.mjs";

export default function PdfViewer({ url }: { url: string }) {
  // ... rendering logic using useEffect and useRef
}
```

**Why**: PDF.js requires `window`, `document`, `canvas`, and other browser APIs. In Next.js App Router, components are server components by default. ALWAYS use `"use client"` directive and dynamic imports with `ssr: false` to ensure PDF.js only runs in the browser.

---

## 8. Incorrect Layer Stacking (No Position Relative on Container)

**Severity**: Medium -- layers render but overlap incorrectly or appear outside the container.

### Wrong

```typescript
// BROKEN: Container has no position: relative
const container = document.getElementById("page-container")!;
// container.style.position is "static" (default)

const canvas = document.createElement("canvas");
canvas.style.position = "absolute"; // Absolute relative to what?
canvas.style.top = "0";
container.appendChild(canvas);
// Canvas positions itself relative to the nearest positioned ancestor,
// which may be the <body> instead of the container.
```

### Correct

```typescript
const container = document.getElementById("page-container")!;
// ALWAYS set position: relative on the layer container
container.style.position = "relative";
container.style.width = `${Math.floor(viewport.width)}px`;
container.style.height = `${Math.floor(viewport.height)}px`;

const canvas = document.createElement("canvas");
canvas.style.position = "absolute";
canvas.style.top = "0";
canvas.style.left = "0";
container.appendChild(canvas);
// Canvas now correctly positions within the container bounds
```

**Why**: CSS `position: absolute` positions an element relative to its nearest ancestor with `position: relative`, `absolute`, or `fixed`. Without setting `position: relative` on the container, all layers position themselves relative to a distant ancestor, breaking the visual alignment between canvas, text layer, and annotation layer.

---

## 9. Not Copying Worker File in Webpack

**Severity**: Critical -- worker fails to load, document loading silently fails or blocks.

### Wrong

```javascript
// webpack.config.js -- missing copy-webpack-plugin
export default {
  entry: "./src/main.ts",
  // ... no CopyWebpackPlugin for the worker file
};
```

```typescript
// main.ts -- references worker that was never copied
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();
// In development: 404 error when trying to load the worker
// In production: worker URL resolves but file does not exist in dist/
```

### Correct

```javascript
// webpack.config.js
import CopyWebpackPlugin from "copy-webpack-plugin";

export default {
  // ...
  plugins: [
    // ALWAYS copy the worker file to the output directory
    new CopyWebpackPlugin({
      patterns: [
        {
          from: "node_modules/pdfjs-dist/build/pdf.worker.min.mjs",
          to: "pdf.worker.min.mjs",
        },
      ],
    }),
  ],
};
```

**Why**: Unlike Vite, webpack does not automatically resolve and serve files referenced via `import.meta.url` from `node_modules`. The worker file MUST be explicitly copied to the output directory so it is available at runtime. Without it, the browser gets a 404 when trying to load the worker, and PDF.js either fails or falls back to synchronous mode.

---

## 10. Forgetting to Clean Up on Re-render

**Severity**: Medium -- causes memory leaks and stale layers accumulating in the DOM.

### Wrong

```typescript
// BROKEN: Appending new layers without removing old ones
async function renderPage(num: number): Promise<void> {
  const page = await pdfDoc!.getPage(num);
  const viewport = page.getViewport({ scale });
  const container = document.getElementById("page-container")!;

  // Creates a NEW canvas on every render without removing the old one
  const canvas = document.createElement("canvas");
  container.appendChild(canvas);
  // After 10 page navigations: 10 canvases stacked on top of each other
}
```

### Correct

```typescript
async function renderPage(num: number): Promise<void> {
  const page = await pdfDoc!.getPage(num);
  const viewport = page.getViewport({ scale });
  const container = document.getElementById("page-container")!;

  // ALWAYS clear previous content before rendering
  container.innerHTML = "";

  const canvas = document.createElement("canvas");
  container.appendChild(canvas);
  // Only one canvas exists at a time
}
```

**Why**: Each canvas consumes significant GPU memory. Appending without removing creates an ever-growing stack of hidden canvases underneath the current one, consuming memory proportional to the number of page navigations. ALWAYS clear the container before creating new layers.
