# Worker Setup Examples

## 1. CDN Setup (No Bundler)

The simplest approach. No build tools required.

### Using cdnjs (Recommended)

```javascript
import * as pdfjsLib from 'pdfjs-dist';

pdfjsLib.GlobalWorkerOptions.workerSrc =
  'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/5.5.207/pdf.worker.min.mjs';

const loadingTask = pdfjsLib.getDocument('https://example.com/document.pdf');
const pdf = await loadingTask.promise;
console.log(`Loaded PDF with ${pdf.numPages} pages`);
```

### Using unpkg

```javascript
pdfjsLib.GlobalWorkerOptions.workerSrc =
  'https://unpkg.com/pdfjs-dist@5.5.207/build/pdf.worker.min.mjs';
```

### Using jsdelivr

```javascript
pdfjsLib.GlobalWorkerOptions.workerSrc =
  'https://cdn.jsdelivr.net/npm/pdfjs-dist@5.5.207/build/pdf.worker.min.mjs';
```

### Script Tag Setup (No ES Modules)

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/5.5.207/pdf.min.mjs" type="module"></script>
<script type="module">
  import * as pdfjsLib from 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/5.5.207/pdf.min.mjs';

  pdfjsLib.GlobalWorkerOptions.workerSrc =
    'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/5.5.207/pdf.worker.min.mjs';

  const pdf = await pdfjsLib.getDocument('document.pdf').promise;
  console.log(`Pages: ${pdf.numPages}`);
</script>
```

---

## 2. Webpack 5 Setup

Webpack 5 natively supports `new URL()` with `import.meta.url` for asset handling.

### Basic Webpack Setup

```javascript
// src/pdfSetup.ts
import * as pdfjsLib from 'pdfjs-dist';

// Webpack 5 emits pdf.worker.min.mjs as a separate chunk
pdfjsLib.GlobalWorkerOptions.workerSrc = new URL(
  'pdfjs-dist/build/pdf.worker.min.mjs',
  import.meta.url
).toString();

export { pdfjsLib };
```

### Webpack Config (webpack.config.js)

No special configuration is needed for the `new URL()` pattern. Webpack 5 handles it automatically via the `asset/resource` module type.

If the `new URL()` pattern does not work (e.g., older Webpack config), use `copy-webpack-plugin`:

```javascript
// webpack.config.js
const CopyPlugin = require('copy-webpack-plugin');

module.exports = {
  plugins: [
    new CopyPlugin({
      patterns: [
        {
          from: 'node_modules/pdfjs-dist/build/pdf.worker.min.mjs',
          to: 'pdf.worker.min.mjs',
        },
      ],
    }),
  ],
};
```

Then reference it as a static asset:

```javascript
pdfjsLib.GlobalWorkerOptions.workerSrc = '/pdf.worker.min.mjs';
```

### Webpack with TypeScript

```typescript
// src/pdfSetup.ts
import * as pdfjsLib from 'pdfjs-dist';
import type { PDFDocumentProxy } from 'pdfjs-dist';

pdfjsLib.GlobalWorkerOptions.workerSrc = new URL(
  'pdfjs-dist/build/pdf.worker.min.mjs',
  import.meta.url
).toString();

export async function loadPdf(url: string): Promise<PDFDocumentProxy> {
  const loadingTask = pdfjsLib.getDocument(url);
  return loadingTask.promise;
}
```

---

## 3. Vite Setup

Vite uses the `?url` import suffix to get asset URLs at build time.

### Basic Vite Setup

```javascript
// src/pdfSetup.ts
import * as pdfjsLib from 'pdfjs-dist';
import workerUrl from 'pdfjs-dist/build/pdf.worker.min.mjs?url';

pdfjsLib.GlobalWorkerOptions.workerSrc = workerUrl;

export { pdfjsLib };
```

### Vite with CMap and Font Assets

Copy CMap and font files to the public directory:

```bash
# Copy required assets to public/ (Vite serves these as static files)
cp -r node_modules/pdfjs-dist/cmaps/ public/cmaps/
cp -r node_modules/pdfjs-dist/standard_fonts/ public/standard_fonts/
```

```javascript
import * as pdfjsLib from 'pdfjs-dist';
import workerUrl from 'pdfjs-dist/build/pdf.worker.min.mjs?url';

pdfjsLib.GlobalWorkerOptions.workerSrc = workerUrl;

const loadingTask = pdfjsLib.getDocument({
  url: '/documents/sample.pdf',
  cMapUrl: '/cmaps/',
  cMapPacked: true,
  standardFontDataUrl: '/standard_fonts/',
});
const pdf = await loadingTask.promise;
```

### Vite Config (vite.config.ts)

No special Vite configuration is required. The `?url` suffix is a built-in Vite feature.

If you encounter build optimization issues:

```typescript
// vite.config.ts
import { defineConfig } from 'vite';

export default defineConfig({
  optimizeDeps: {
    include: ['pdfjs-dist'],
  },
});
```

---

## 4. Rollup Setup

Rollup does not natively support the `new URL()` pattern. Use a copy plugin or CDN fallback.

### Using @rollup/plugin-copy

```javascript
// rollup.config.mjs
import copy from 'rollup-plugin-copy';

export default {
  input: 'src/index.js',
  output: { dir: 'dist', format: 'es' },
  plugins: [
    copy({
      targets: [
        {
          src: 'node_modules/pdfjs-dist/build/pdf.worker.min.mjs',
          dest: 'dist',
        },
      ],
    }),
  ],
};
```

```javascript
// src/pdfSetup.js
import * as pdfjsLib from 'pdfjs-dist';

pdfjsLib.GlobalWorkerOptions.workerSrc = '/pdf.worker.min.mjs';

export { pdfjsLib };
```

### CDN Fallback for Rollup

If copying is not feasible, fall back to a CDN:

```javascript
import * as pdfjsLib from 'pdfjs-dist';
import { version } from 'pdfjs-dist/package.json';

pdfjsLib.GlobalWorkerOptions.workerSrc =
  `https://cdnjs.cloudflare.com/ajax/libs/pdf.js/${version}/pdf.worker.min.mjs`;
```

---

## 5. Next.js Setup

### App Router (Recommended)

```typescript
// src/lib/pdfSetup.ts
'use client'; // MUST be a client component — PDF.js requires browser APIs

import * as pdfjsLib from 'pdfjs-dist';

pdfjsLib.GlobalWorkerOptions.workerSrc =
  'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/5.5.207/pdf.worker.min.mjs';

export { pdfjsLib };
```

### Using public/ Directory

```bash
# Copy worker to Next.js public directory
cp node_modules/pdfjs-dist/build/pdf.worker.min.mjs public/pdf.worker.min.mjs
```

```typescript
'use client';
import * as pdfjsLib from 'pdfjs-dist';

pdfjsLib.GlobalWorkerOptions.workerSrc = '/pdf.worker.min.mjs';

export { pdfjsLib };
```

### Dynamic Import (Avoid SSR Issues)

```typescript
'use client';
import { useEffect, useState } from 'react';

export function usePdfJs() {
  const [pdfjsLib, setPdfjsLib] = useState<typeof import('pdfjs-dist') | null>(null);

  useEffect(() => {
    async function init() {
      const pdfjs = await import('pdfjs-dist');
      pdfjs.GlobalWorkerOptions.workerSrc =
        'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/5.5.207/pdf.worker.min.mjs';
      setPdfjsLib(pdfjs);
    }
    init();
  }, []);

  return pdfjsLib;
}
```

---

## 6. Fake Worker Mode (Testing / SSR / Node.js)

### Direct Import (Inline Worker)

```javascript
import * as pdfjsLib from 'pdfjs-dist';

// Import worker code directly — runs on the main thread
import 'pdfjs-dist/build/pdf.worker.min.mjs';

// PDF.js detects the worker is already loaded
const loadingTask = pdfjsLib.getDocument('document.pdf');
const pdf = await loadingTask.promise;
```

### Node.js Setup

```javascript
// Node.js does not have Web Workers — fake worker is automatic
import * as pdfjsLib from 'pdfjs-dist/legacy/build/pdf.mjs';

const loadingTask = pdfjsLib.getDocument('document.pdf');
const pdf = await loadingTask.promise;
```

---

## 7. Complete Production Setup

Full setup with worker, CMaps, fonts, error handling, and TypeScript:

```typescript
import * as pdfjsLib from 'pdfjs-dist';
import type { PDFDocumentProxy, PDFPageProxy } from 'pdfjs-dist';

// --- Worker Setup ---
pdfjsLib.GlobalWorkerOptions.workerSrc =
  'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/5.5.207/pdf.worker.min.mjs';

// --- Load Document ---
export async function loadDocument(
  source: string | ArrayBuffer,
): Promise<PDFDocumentProxy> {
  const params: Record<string, unknown> = {
    cMapUrl: '/cmaps/',
    cMapPacked: true,
    standardFontDataUrl: '/standard_fonts/',
  };

  if (typeof source === 'string') {
    params.url = source;
  } else {
    params.data = source;
  }

  const loadingTask = pdfjsLib.getDocument(params);

  loadingTask.onProgress = ({ loaded, total }: { loaded: number; total: number }) => {
    console.log(`Loading: ${Math.round((loaded / total) * 100)}%`);
  };

  return loadingTask.promise;
}

// --- Render Page ---
export async function renderPage(
  pdf: PDFDocumentProxy,
  pageNumber: number,
  canvas: HTMLCanvasElement,
  scale: number = 1.5,
): Promise<void> {
  const page: PDFPageProxy = await pdf.getPage(pageNumber);
  const viewport = page.getViewport({ scale });

  canvas.width = viewport.width;
  canvas.height = viewport.height;

  const context = canvas.getContext('2d');
  if (!context) throw new Error('Canvas 2D context not available');

  await page.render({ canvasContext: context, viewport }).promise;
}
```
