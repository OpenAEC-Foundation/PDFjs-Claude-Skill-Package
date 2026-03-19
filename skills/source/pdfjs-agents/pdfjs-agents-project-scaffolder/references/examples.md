# Complete Project Templates (pdfjs-dist 5.x)

## 1. Vanilla CDN Setup

Zero build step. Copy these files and open `index.html` in a browser.

### index.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>PDF.js Viewer (Vanilla)</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { font-family: system-ui, sans-serif; background: #525659; }

    #toolbar {
      position: sticky; top: 0; z-index: 10;
      background: #323639; color: #fff;
      padding: 8px 16px; display: flex; align-items: center; gap: 8px;
    }
    #toolbar button { padding: 4px 12px; cursor: pointer; }
    #toolbar input { width: 60px; padding: 4px; }

    #viewer-container {
      display: flex; justify-content: center; padding: 20px 0; overflow: auto;
    }
    #page-container {
      position: relative; box-shadow: 0 2px 10px rgba(0,0,0,0.3); background: #fff;
    }
    .textLayer { z-index: 1; }
    .annotationLayer { z-index: 2; }
  </style>
  <!-- ALWAYS load the viewer CSS from CDN for text/annotation layer styles -->
  <link rel="stylesheet"
    href="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/5.0.375/pdf_viewer.min.css" />
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

  <!-- ALWAYS load pdf.mjs BEFORE pdf.worker.min.mjs -->
  <script type="module">
    // Use the ESM build from CDN
    import {
      getDocument,
      GlobalWorkerOptions,
    } from "https://cdnjs.cloudflare.com/ajax/libs/pdf.js/5.0.375/pdf.min.mjs";

    // ALWAYS set workerSrc to the matching version
    GlobalWorkerOptions.workerSrc =
      "https://cdnjs.cloudflare.com/ajax/libs/pdf.js/5.0.375/pdf.worker.min.mjs";

    // State
    let pdfDoc = null;
    let currentPage = 1;
    let currentScale = 1.5;
    let currentRenderTask = null;

    // DOM
    const container = document.getElementById("page-container");
    const pageNum = document.getElementById("page-num");
    const pageCount = document.getElementById("page-count");

    async function loadDocument(url) {
      pdfDoc = await getDocument({ url }).promise;
      pageCount.textContent = String(pdfDoc.numPages);
      await renderPage(currentPage);
    }

    async function renderPage(num) {
      if (!pdfDoc) return;

      // ALWAYS cancel previous render
      if (currentRenderTask) {
        currentRenderTask.cancel();
        currentRenderTask = null;
      }

      const page = await pdfDoc.getPage(num);
      const viewport = page.getViewport({ scale: currentScale });
      const dpr = window.devicePixelRatio || 1;

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

      const ctx = canvas.getContext("2d");
      ctx.scale(dpr, dpr);

      currentRenderTask = page.render({ canvasContext: ctx, viewport });

      try {
        await currentRenderTask.promise;
      } catch (err) {
        if (err.message === "Rendering cancelled") return;
        throw err;
      } finally {
        currentRenderTask = null;
      }

      pageNum.textContent = String(num);
    }

    // Navigation
    document.getElementById("prev-btn").addEventListener("click", () => {
      if (currentPage <= 1) return;
      currentPage--;
      renderPage(currentPage);
    });

    document.getElementById("next-btn").addEventListener("click", () => {
      if (!pdfDoc || currentPage >= pdfDoc.numPages) return;
      currentPage++;
      renderPage(currentPage);
    });

    document.getElementById("goto-btn").addEventListener("click", () => {
      const input = document.getElementById("page-input");
      const num = parseInt(input.value, 10);
      if (!pdfDoc || num < 1 || num > pdfDoc.numPages) return;
      currentPage = num;
      renderPage(currentPage);
    });

    // Zoom
    document.getElementById("zoom-in-btn").addEventListener("click", () => {
      currentScale *= 1.25;
      renderPage(currentPage);
    });

    document.getElementById("zoom-out-btn").addEventListener("click", () => {
      currentScale /= 1.25;
      renderPage(currentPage);
    });

    // Load a sample PDF -- replace with your PDF URL
    loadDocument("https://mozilla.github.io/pdf.js/web/compressed.tracemonkey-pldi-09.pdf");
  </script>
</body>
</html>
```

---

## 2. Webpack Setup (TypeScript)

### File Tree

```
pdfjs-webpack-viewer/
├── package.json
├── tsconfig.json
├── webpack.config.js
├── src/
│   ├── index.html
│   ├── styles.css
│   └── main.ts
```

### package.json

```json
{
  "name": "pdfjs-webpack-viewer",
  "private": true,
  "version": "1.0.0",
  "type": "module",
  "dependencies": {
    "pdfjs-dist": "^5.0.0"
  },
  "devDependencies": {
    "webpack": "^5.90.0",
    "webpack-cli": "^5.1.0",
    "webpack-dev-server": "^5.0.0",
    "html-webpack-plugin": "^5.6.0",
    "copy-webpack-plugin": "^12.0.0",
    "ts-loader": "^9.5.0",
    "css-loader": "^7.1.0",
    "style-loader": "^4.0.0",
    "typescript": "^5.5.0"
  },
  "scripts": {
    "dev": "webpack serve --mode development",
    "build": "webpack --mode production"
  }
}
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"]
  },
  "include": ["src"]
}
```

### webpack.config.js

```javascript
import path from "node:path";
import { fileURLToPath } from "node:url";
import HtmlWebpackPlugin from "html-webpack-plugin";
import CopyWebpackPlugin from "copy-webpack-plugin";

const __dirname = path.dirname(fileURLToPath(import.meta.url));

export default {
  entry: "./src/main.ts",
  output: {
    path: path.resolve(__dirname, "dist"),
    filename: "bundle.js",
    clean: true,
  },
  resolve: {
    extensions: [".ts", ".js"],
  },
  module: {
    rules: [
      {
        test: /\.ts$/,
        use: "ts-loader",
        exclude: /node_modules/,
      },
      {
        test: /\.css$/,
        use: ["style-loader", "css-loader"],
      },
    ],
  },
  plugins: [
    new HtmlWebpackPlugin({
      template: "./src/index.html",
    }),
    // ALWAYS copy the worker file -- it MUST NOT be bundled into the main chunk
    new CopyWebpackPlugin({
      patterns: [
        {
          from: "node_modules/pdfjs-dist/build/pdf.worker.min.mjs",
          to: "pdf.worker.min.mjs",
        },
      ],
    }),
  ],
  devServer: {
    port: 3000,
    hot: true,
  },
};
```

### src/index.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>PDF.js Viewer (Webpack)</title>
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
</body>
</html>
```

### src/styles.css

```css
@import "pdfjs-dist/web/pdf_viewer.css";

* { margin: 0; padding: 0; box-sizing: border-box; }
body { font-family: system-ui, sans-serif; background: #525659; }

#toolbar {
  position: sticky; top: 0; z-index: 10;
  background: #323639; color: #fff;
  padding: 8px 16px; display: flex; align-items: center; gap: 8px;
}
#toolbar button { padding: 4px 12px; cursor: pointer; }
#toolbar input { width: 60px; padding: 4px; }

#viewer-container {
  display: flex; justify-content: center; padding: 20px 0; overflow: auto;
}
#page-container {
  position: relative; box-shadow: 0 2px 10px rgba(0,0,0,0.3); background: #fff;
}
.textLayer { z-index: 1; }
.annotationLayer { z-index: 2; }
```

### src/main.ts

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";
import { TextLayer, AnnotationLayer } from "pdfjs-dist";
import type { PDFDocumentProxy, RenderTask } from "pdfjs-dist";
import "./styles.css";

// ALWAYS configure worker BEFORE any document loading
// Uses the file copied by copy-webpack-plugin
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

let pdfDoc: PDFDocumentProxy | null = null;
let currentPage = 1;
let currentScale = 1.5;
let currentRenderTask: RenderTask | null = null;

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

  if (currentRenderTask) {
    currentRenderTask.cancel();
    currentRenderTask = null;
  }

  const page = await pdfDoc.getPage(num);
  const viewport = page.getViewport({ scale: currentScale });
  const dpr = window.devicePixelRatio || 1;

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
document.getElementById("prev-btn")!.addEventListener("click", () => {
  if (currentPage <= 1) return;
  currentPage--;
  renderPage(currentPage);
});

document.getElementById("next-btn")!.addEventListener("click", () => {
  if (!pdfDoc || currentPage >= pdfDoc.numPages) return;
  currentPage++;
  renderPage(currentPage);
});

document.getElementById("goto-btn")!.addEventListener("click", () => {
  const input = document.getElementById("page-input") as HTMLInputElement;
  const num = parseInt(input.value, 10);
  if (!pdfDoc || num < 1 || num > pdfDoc.numPages) return;
  currentPage = num;
  renderPage(currentPage);
});

document.getElementById("zoom-in-btn")!.addEventListener("click", () => {
  currentScale *= 1.25;
  renderPage(currentPage);
});

document.getElementById("zoom-out-btn")!.addEventListener("click", () => {
  currentScale /= 1.25;
  renderPage(currentPage);
});

// Load PDF -- replace with your PDF path or URL
loadDocument("/sample.pdf");
```

---

## 3. Vite Setup (TypeScript)

### File Tree

```
pdfjs-vite-viewer/
├── package.json
├── tsconfig.json
├── vite.config.ts
├── index.html
├── src/
│   ├── styles.css
│   └── main.ts
```

### package.json

```json
{
  "name": "pdfjs-vite-viewer",
  "private": true,
  "version": "1.0.0",
  "type": "module",
  "dependencies": {
    "pdfjs-dist": "^5.0.0"
  },
  "devDependencies": {
    "vite": "^6.0.0",
    "typescript": "^5.5.0"
  },
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  }
}
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"]
  },
  "include": ["src"]
}
```

### vite.config.ts

```typescript
import { defineConfig } from "vite";

export default defineConfig({
  // Vite handles import.meta.url natively -- no special config needed for the worker
  optimizeDeps: {
    // ALWAYS include pdfjs-dist in optimized deps for faster dev startup
    include: ["pdfjs-dist"],
  },
  build: {
    // Ensure the worker file is not inlined
    assetsInlineLimit: 0,
  },
});
```

### index.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>PDF.js Viewer (Vite)</title>
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
  <script type="module" src="/src/main.ts"></script>
</body>
</html>
```

### src/styles.css

```css
@import "pdfjs-dist/web/pdf_viewer.css";

* { margin: 0; padding: 0; box-sizing: border-box; }
body { font-family: system-ui, sans-serif; background: #525659; }

#toolbar {
  position: sticky; top: 0; z-index: 10;
  background: #323639; color: #fff;
  padding: 8px 16px; display: flex; align-items: center; gap: 8px;
}
#toolbar button { padding: 4px 12px; cursor: pointer; }
#toolbar input { width: 60px; padding: 4px; }

#viewer-container {
  display: flex; justify-content: center; padding: 20px 0; overflow: auto;
}
#page-container {
  position: relative; box-shadow: 0 2px 10px rgba(0,0,0,0.3); background: #fff;
}
.textLayer { z-index: 1; }
.annotationLayer { z-index: 2; }
```

### src/main.ts

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";
import { TextLayer, AnnotationLayer } from "pdfjs-dist";
import type { PDFDocumentProxy, RenderTask } from "pdfjs-dist";
import "./styles.css";

// Vite handles import.meta.url natively -- no plugins needed
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

let pdfDoc: PDFDocumentProxy | null = null;
let currentPage = 1;
let currentScale = 1.5;
let currentRenderTask: RenderTask | null = null;

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

  if (currentRenderTask) {
    currentRenderTask.cancel();
    currentRenderTask = null;
  }

  const page = await pdfDoc.getPage(num);
  const viewport = page.getViewport({ scale: currentScale });
  const dpr = window.devicePixelRatio || 1;

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
document.getElementById("prev-btn")!.addEventListener("click", () => {
  if (currentPage <= 1) return;
  currentPage--;
  renderPage(currentPage);
});

document.getElementById("next-btn")!.addEventListener("click", () => {
  if (!pdfDoc || currentPage >= pdfDoc.numPages) return;
  currentPage++;
  renderPage(currentPage);
});

document.getElementById("goto-btn")!.addEventListener("click", () => {
  const input = document.getElementById("page-input") as HTMLInputElement;
  const num = parseInt(input.value, 10);
  if (!pdfDoc || num < 1 || num > pdfDoc.numPages) return;
  currentPage = num;
  renderPage(currentPage);
});

document.getElementById("zoom-in-btn")!.addEventListener("click", () => {
  currentScale *= 1.25;
  renderPage(currentPage);
});

document.getElementById("zoom-out-btn")!.addEventListener("click", () => {
  currentScale /= 1.25;
  renderPage(currentPage);
});

// Load PDF -- replace with your PDF path or URL
loadDocument("/sample.pdf");
```

---

## 4. Next.js Setup (TypeScript)

### File Tree

```
pdfjs-nextjs-viewer/
├── package.json
├── tsconfig.json
├── next.config.js
├── public/
│   └── pdf.worker.min.mjs      (copied from node_modules/pdfjs-dist/build/)
├── app/
│   ├── layout.tsx
│   ├── page.tsx
│   └── globals.css
└── components/
    └── PdfViewer.tsx
```

### package.json

```json
{
  "name": "pdfjs-nextjs-viewer",
  "private": true,
  "version": "1.0.0",
  "dependencies": {
    "next": "^15.0.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "pdfjs-dist": "^5.0.0"
  },
  "devDependencies": {
    "@types/react": "^19.0.0",
    "@types/node": "^22.0.0",
    "typescript": "^5.5.0"
  },
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "postinstall": "cp node_modules/pdfjs-dist/build/pdf.worker.min.mjs public/pdf.worker.min.mjs"
  }
}
```

### next.config.js

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  // Webpack config to handle pdfjs-dist correctly
  webpack: (config) => {
    // Prevent webpack from trying to bundle the worker
    config.resolve.alias.canvas = false;
    return config;
  },
};

export default nextConfig;
```

### app/layout.tsx

```tsx
import type { Metadata } from "next";
import "./globals.css";

export const metadata: Metadata = {
  title: "PDF.js Viewer (Next.js)",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}
```

### app/globals.css

```css
* { margin: 0; padding: 0; box-sizing: border-box; }
body { font-family: system-ui, sans-serif; background: #525659; }

.toolbar {
  position: sticky; top: 0; z-index: 10;
  background: #323639; color: #fff;
  padding: 8px 16px; display: flex; align-items: center; gap: 8px;
}
.toolbar button { padding: 4px 12px; cursor: pointer; }
.toolbar input { width: 60px; padding: 4px; }

.viewer-container {
  display: flex; justify-content: center; padding: 20px 0; overflow: auto;
}
.page-container {
  position: relative; box-shadow: 0 2px 10px rgba(0,0,0,0.3); background: #fff;
}
.textLayer { z-index: 1; }
.annotationLayer { z-index: 2; }
```

### app/page.tsx

```tsx
import dynamic from "next/dynamic";

// ALWAYS use dynamic import with ssr: false for PDF.js components
// PDF.js requires browser APIs (canvas, window) that do not exist on the server
const PdfViewer = dynamic(() => import("../components/PdfViewer"), {
  ssr: false,
  loading: () => <p style={{ color: "#fff", textAlign: "center", padding: "20px" }}>Loading viewer...</p>,
});

export default function Home() {
  return <PdfViewer url="/sample.pdf" />;
}
```

### components/PdfViewer.tsx

```tsx
"use client";

import { useEffect, useRef, useState, useCallback } from "react";
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";
import { TextLayer, AnnotationLayer } from "pdfjs-dist";
import type { PDFDocumentProxy, RenderTask } from "pdfjs-dist";
// ALWAYS import the viewer CSS for text/annotation layer styles
import "pdfjs-dist/web/pdf_viewer.css";

// ALWAYS set workerSrc BEFORE any document loading
// Uses the file copied to public/ by the postinstall script
GlobalWorkerOptions.workerSrc = "/pdf.worker.min.mjs";

interface PdfViewerProps {
  url: string;
}

export default function PdfViewer({ url }: PdfViewerProps) {
  const containerRef = useRef<HTMLDivElement>(null);
  const renderTaskRef = useRef<RenderTask | null>(null);
  const [pdfDoc, setPdfDoc] = useState<PDFDocumentProxy | null>(null);
  const [currentPage, setCurrentPage] = useState(1);
  const [totalPages, setTotalPages] = useState(0);
  const [scale, setScale] = useState(1.5);
  const [pageInput, setPageInput] = useState("1");

  // Load document
  useEffect(() => {
    let cancelled = false;

    async function load() {
      const doc = await getDocument({ url }).promise;
      if (cancelled) return;
      setPdfDoc(doc);
      setTotalPages(doc.numPages);
    }

    load();
    return () => { cancelled = true; };
  }, [url]);

  // Render page
  const renderPage = useCallback(async (num: number, renderScale: number) => {
    if (!pdfDoc || !containerRef.current) return;

    // ALWAYS cancel previous render
    if (renderTaskRef.current) {
      renderTaskRef.current.cancel();
      renderTaskRef.current = null;
    }

    const page = await pdfDoc.getPage(num);
    const viewport = page.getViewport({ scale: renderScale });
    const dpr = window.devicePixelRatio || 1;
    const container = containerRef.current;

    container.innerHTML = "";
    container.style.width = `${Math.floor(viewport.width)}px`;
    container.style.height = `${Math.floor(viewport.height)}px`;

    // Canvas
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

    renderTaskRef.current = page.render({ canvasContext: ctx, viewport });

    try {
      await renderTaskRef.current.promise;
    } catch (err: unknown) {
      if (err instanceof Error && err.message === "Rendering cancelled") return;
      throw err;
    } finally {
      renderTaskRef.current = null;
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
  }, [pdfDoc]);

  // Re-render on page or scale change
  useEffect(() => {
    if (pdfDoc) renderPage(currentPage, scale);
  }, [pdfDoc, currentPage, scale, renderPage]);

  return (
    <>
      <div className="toolbar">
        <button onClick={() => setCurrentPage((p) => Math.max(1, p - 1))}>Previous</button>
        <span>
          Page {currentPage} / {totalPages}
        </span>
        <button onClick={() => setCurrentPage((p) => Math.min(totalPages, p + 1))}>Next</button>
        <input
          type="number"
          min={1}
          max={totalPages}
          value={pageInput}
          onChange={(e) => setPageInput(e.target.value)}
        />
        <button
          onClick={() => {
            const num = parseInt(pageInput, 10);
            if (num >= 1 && num <= totalPages) setCurrentPage(num);
          }}
        >
          Go
        </button>
        <button onClick={() => setScale((s) => s * 1.25)}>Zoom In</button>
        <button onClick={() => setScale((s) => s / 1.25)}>Zoom Out</button>
      </div>
      <div className="viewer-container">
        <div ref={containerRef} className="page-container" />
      </div>
    </>
  );
}
```

---

## Setup Commands Quick Reference

### Vanilla CDN
```bash
# No setup needed -- just open index.html in a browser
# For a local server:
npx serve .
```

### Webpack
```bash
npm init -y
npm install pdfjs-dist
npm install -D webpack webpack-cli webpack-dev-server html-webpack-plugin copy-webpack-plugin ts-loader css-loader style-loader typescript
npm run dev
```

### Vite
```bash
npm init -y
npm install pdfjs-dist
npm install -D vite typescript
npm run dev
```

### Next.js
```bash
npx create-next-app@latest pdfjs-nextjs-viewer --typescript --app
cd pdfjs-nextjs-viewer
npm install pdfjs-dist
# Copy worker to public folder
cp node_modules/pdfjs-dist/build/pdf.worker.min.mjs public/
npm run dev
```
