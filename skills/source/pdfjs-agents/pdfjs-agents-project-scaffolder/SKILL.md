---
name: pdfjs-agents-project-scaffolder
description: >
  Use when generating a complete PDF.js project from scratch, scaffolding a new
  PDF viewer application, or setting up the full rendering pipeline with all layers.
  Prevents incomplete project setup by ensuring worker, canvas, text layer, and
  annotation layer are all properly configured.
  Covers bundler-specific worker config, HTML/CSS templates, TypeScript setup,
  package.json with correct pdfjs-dist version, and full rendering pipeline.
  Keywords: scaffold, project setup, boilerplate, PDF viewer template, TypeScript, pdfjs-dist.
license: MIT
compatibility: "Designed for Claude Code. Requires pdfjs-dist 5.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# pdfjs-agents-project-scaffolder

## Quick Reference

### Scaffolding Pipeline

| Step | Action | Output |
|------|--------|--------|
| 1. Choose setup | Select vanilla CDN / webpack / Vite / Next.js | Build strategy |
| 2. Create package.json | Add pdfjs-dist 5.x dependency | package.json |
| 3. Configure worker | Bundler-specific worker setup | Worker config |
| 4. Create HTML template | Page container, canvas, text layer, annotation layer | index.html |
| 5. Create CSS | Layer stacking, responsive sizing | styles.css |
| 6. Create entry point | Worker setup, rendering, navigation, zoom | main.ts / main.js |
| 7. Configure bundler | webpack.config.js / vite.config.ts | Bundler config |
| 8. Add TypeScript (optional) | tsconfig.json with proper types | tsconfig.json |

### Critical Warnings

**ALWAYS** configure `GlobalWorkerOptions.workerSrc` BEFORE calling `getDocument()` -- the worker MUST be initialized first or document loading will fail silently or throw.

**ALWAYS** copy or reference the worker file correctly for your bundler -- the worker version MUST match the pdfjs-dist version exactly. A version mismatch causes cryptic deserialization errors.

**NEVER** import `pdf.worker.min.mjs` directly into your main bundle -- the worker MUST run in a separate thread. Bundling it into the main bundle defeats the purpose and blocks the UI thread during PDF parsing.

**ALWAYS** include the `pdfjs-dist/web/pdf_viewer.css` stylesheet -- without it, the text layer and annotation layer will be incorrectly positioned and invisible or misaligned.

**NEVER** use `pdfjs-dist/build/pdf.js` (CommonJS) in ESM bundler projects -- ALWAYS use `pdfjs-dist/build/pdf.mjs` for webpack 5+ and Vite.

**ALWAYS** handle `devicePixelRatio` in canvas setup -- without it, PDFs render blurry on Retina and 4K displays.

---

## Decision Tree: Which Setup?

```
Starting a new PDF.js project?
├── Quick prototype / no bundler?
│   └── Vanilla CDN → See references/examples.md §1
│       - Zero build step
│       - Script tags from CDN (cdnjs or unpkg)
│       - Good for demos and learning
│
├── Production app with legacy support?
│   └── Webpack → See references/examples.md §2
│       - Full control over worker bundling
│       - Copy-webpack-plugin for worker file
│       - Tree-shaking with webpack 5
│
├── Modern production app?
│   └── Vite → See references/examples.md §3
│       - Fastest dev server
│       - Native ESM, uses import.meta.url for worker
│       - Simplest worker configuration
│
└── SSR / React framework?
    └── Next.js → See references/examples.md §4
        - Dynamic import with ssr: false
        - Worker loaded client-side only
        - Special handling for server components
```

---

## Essential Patterns

### package.json (All Setups)

```json
{
  "dependencies": {
    "pdfjs-dist": "^5.0.0"
  }
}
```

**ALWAYS** pin to major version 5.x. NEVER mix pdfjs-dist versions between the main library and the worker file.

### Worker Configuration by Setup

| Setup | Worker Strategy | Worker Path |
|-------|----------------|-------------|
| Vanilla CDN | CDN URL string | `https://cdnjs.cloudflare.com/ajax/libs/pdf.js/5.x.x/pdf.worker.min.mjs` |
| Webpack | copy-webpack-plugin copies worker to output | `new URL("pdfjs-dist/build/pdf.worker.min.mjs", import.meta.url)` |
| Vite | Native ESM, no plugin needed | `new URL("pdfjs-dist/build/pdf.worker.min.mjs", import.meta.url)` |
| Next.js | CDN or public folder | CDN URL or `/pdf.worker.min.mjs` from public/ |

### Minimal Entry Point (TypeScript)

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";
import { TextLayer, AnnotationLayer } from "pdfjs-dist";
import type { PDFDocumentProxy, PDFPageProxy, RenderTask } from "pdfjs-dist";
import "pdfjs-dist/web/pdf_viewer.css";

// ALWAYS configure worker BEFORE any getDocument() call
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

// State
let pdfDoc: PDFDocumentProxy | null = null;
let currentPage = 1;
let currentScale = 1.5;
let currentRenderTask: RenderTask | null = null;

// DOM elements
const container = document.getElementById("page-container") as HTMLDivElement;
const pageNum = document.getElementById("page-num") as HTMLSpanElement;
const pageCount = document.getElementById("page-count") as HTMLSpanElement;

async function loadDocument(url: string): Promise<void> {
  pdfDoc = await getDocument({ url }).promise;
  pageCount.textContent = String(pdfDoc.numPages);
  await renderPage(currentPage);
}

async function renderPage(num: number): Promise<void> {
  if (!pdfDoc) return;

  // ALWAYS cancel previous render
  if (currentRenderTask) {
    currentRenderTask.cancel();
    currentRenderTask = null;
  }

  const page = await pdfDoc.getPage(num);
  const viewport = page.getViewport({ scale: currentScale });
  const dpr = window.devicePixelRatio || 1;

  // Clear container
  container.innerHTML = "";
  container.style.position = "relative";
  container.style.width = `${Math.floor(viewport.width)}px`;
  container.style.height = `${Math.floor(viewport.height)}px`;

  // Canvas layer
  const canvas = document.createElement("canvas");
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;
  canvas.style.position = "absolute";
  canvas.style.top = "0";
  canvas.style.left = "0";
  container.appendChild(canvas);

  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);

  currentRenderTask = page.render({ canvasContext: ctx, viewport });

  try {
    await currentRenderTask.promise;
  } catch (err: unknown) {
    if (err instanceof Error && err.message === "Rendering cancelled") return;
    throw err;
  } finally {
    currentRenderTask = null;
  }

  // Text layer
  const textDiv = document.createElement("div");
  textDiv.className = "textLayer";
  textDiv.style.position = "absolute";
  textDiv.style.top = "0";
  textDiv.style.left = "0";
  container.appendChild(textDiv);

  const textContent = await page.getTextContent();
  const textLayer = new TextLayer({
    container: textDiv,
    textContentSource: textContent,
    viewport,
  });
  await textLayer.render();

  // Annotation layer
  const annotDiv = document.createElement("div");
  annotDiv.className = "annotationLayer";
  annotDiv.style.position = "absolute";
  annotDiv.style.top = "0";
  annotDiv.style.left = "0";
  container.appendChild(annotDiv);

  const annotations = await page.getAnnotations();
  const annotationLayer = new AnnotationLayer({
    div: annotDiv,
    annotations,
    page,
    viewport,
  });
  await annotationLayer.render({ viewport, annotations });

  pageNum.textContent = String(num);
}

// Navigation
function prevPage(): void {
  if (currentPage <= 1) return;
  currentPage--;
  renderPage(currentPage);
}

function nextPage(): void {
  if (!pdfDoc || currentPage >= pdfDoc.numPages) return;
  currentPage++;
  renderPage(currentPage);
}

function goToPage(num: number): void {
  if (!pdfDoc || num < 1 || num > pdfDoc.numPages) return;
  currentPage = num;
  renderPage(currentPage);
}

// Zoom
function zoomIn(): void {
  currentScale *= 1.25;
  renderPage(currentPage);
}

function zoomOut(): void {
  currentScale /= 1.25;
  renderPage(currentPage);
}
```

### HTML Template

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>PDF.js Viewer</title>
  <link rel="stylesheet" href="styles.css" />
</head>
<body>
  <div id="toolbar">
    <button id="prev-btn">Previous</button>
    <span>Page <span id="page-num">1</span> / <span id="page-count">-</span></span>
    <button id="next-btn">Next</button>
    <input id="page-input" type="number" min="1" />
    <button id="goto-btn">Go</button>
    <button id="zoom-in-btn">Zoom In</button>
    <button id="zoom-out-btn">Zoom Out</button>
  </div>
  <div id="viewer-container">
    <div id="page-container"></div>
  </div>
  <script type="module" src="main.ts"></script>
</body>
</html>
```

### CSS Template

```css
/* ALWAYS import the official PDF.js viewer stylesheet */
@import "pdfjs-dist/web/pdf_viewer.css";

* { margin: 0; padding: 0; box-sizing: border-box; }

body {
  font-family: system-ui, sans-serif;
  background: #525659;
}

#toolbar {
  position: sticky;
  top: 0;
  z-index: 10;
  background: #323639;
  color: #fff;
  padding: 8px 16px;
  display: flex;
  align-items: center;
  gap: 8px;
}

#toolbar button {
  padding: 4px 12px;
  cursor: pointer;
}

#toolbar input {
  width: 60px;
  padding: 4px;
}

#viewer-container {
  display: flex;
  justify-content: center;
  padding: 20px 0;
  overflow: auto;
}

/* ALWAYS ensure layers stack correctly */
#page-container {
  position: relative;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.3);
  background: #fff;
}

.textLayer {
  z-index: 1;
}

.annotationLayer {
  z-index: 2;
}
```

---

## File Tree by Setup

### Vanilla CDN
```
project/
├── index.html
├── styles.css
└── main.js
```

### Webpack (TypeScript)
```
project/
├── package.json
├── tsconfig.json
├── webpack.config.js
├── src/
│   ├── index.html
│   ├── styles.css
│   └── main.ts
└── dist/              (generated)
```

### Vite (TypeScript)
```
project/
├── package.json
├── tsconfig.json
├── vite.config.ts
├── index.html
├── src/
│   ├── styles.css
│   └── main.ts
└── dist/              (generated)
```

### Next.js
```
project/
├── package.json
├── tsconfig.json
├── next.config.js
├── public/
│   └── pdf.worker.min.mjs    (copied from node_modules)
├── app/
│   ├── layout.tsx
│   └── page.tsx
└── components/
    └── PdfViewer.tsx          (client component with "use client")
```

---

## Reference Links

- [references/methods.md](references/methods.md) -- Scaffolding decision tree and configuration options per setup
- [references/examples.md](references/examples.md) -- Complete project templates: vanilla, webpack, Vite, Next.js
- [references/anti-patterns.md](references/anti-patterns.md) -- Common scaffolding mistakes and their fixes

### Official Sources

- https://mozilla.github.io/pdf.js/getting_started/ -- PDF.js getting started guide
- https://mozilla.github.io/pdf.js/api/ -- PDF.js API reference
- https://github.com/mozilla/pdf.js/tree/master/examples -- Official examples
- https://www.npmjs.com/package/pdfjs-dist -- npm package info
