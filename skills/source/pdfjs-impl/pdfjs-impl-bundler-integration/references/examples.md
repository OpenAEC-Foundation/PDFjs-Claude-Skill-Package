# Bundler Integration Examples (pdfjs-dist 5.x)

Complete, copy-paste-ready configurations for each bundler.

---

## 1. Webpack 5+ (Recommended: Zero-Config)

### Application Code

```typescript
// src/pdf-viewer.ts
import * as pdfjsLib from "pdfjs-dist/webpack.mjs";

// Worker is ALREADY configured -- no manual setup needed

async function loadPdf(url: string): Promise<void> {
  const doc = await pdfjsLib.getDocument({
    url,
    cMapUrl: "/cmaps/",
    cMapPacked: true,
    standardFontDataUrl: "/standard_fonts/",
  }).promise;

  const page = await doc.getPage(1);
  const viewport = page.getViewport({ scale: 1.5 });

  const canvas = document.getElementById("pdf-canvas") as HTMLCanvasElement;
  const dpr = window.devicePixelRatio || 1;
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;

  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);

  await page.render({ canvasContext: ctx, viewport }).promise;
}
```

### Webpack Config with CMap/Font Copying

```javascript
// webpack.config.js
const path = require("path");
const CopyPlugin = require("copy-webpack-plugin");

module.exports = {
  entry: "./src/pdf-viewer.ts",
  output: {
    path: path.resolve(__dirname, "dist"),
    filename: "bundle.js",
    clean: true,
  },
  resolve: {
    extensions: [".ts", ".js", ".mjs"],
  },
  module: {
    rules: [
      {
        test: /\.ts$/,
        use: "ts-loader",
        exclude: /node_modules/,
      },
    ],
  },
  plugins: [
    new CopyPlugin({
      patterns: [
        // Copy CMap files for CJK PDF support
        {
          from: "node_modules/pdfjs-dist/cmaps/",
          to: "cmaps/",
        },
        // Copy standard font files
        {
          from: "node_modules/pdfjs-dist/standard_fonts/",
          to: "standard_fonts/",
        },
      ],
    }),
  ],
};
```

### Package Dependencies

```json
{
  "dependencies": {
    "pdfjs-dist": "^5.5.0"
  },
  "devDependencies": {
    "copy-webpack-plugin": "^12.0.0",
    "ts-loader": "^9.5.0",
    "typescript": "^5.4.0",
    "webpack": "^5.90.0",
    "webpack-cli": "^5.1.0"
  }
}
```

---

## 2. Webpack 5+ (Manual Worker Entry Point)

Use this approach when you need explicit control over the worker bundle name.

### Webpack Config

```javascript
// webpack.config.js
const path = require("path");
const CopyPlugin = require("copy-webpack-plugin");

module.exports = {
  entry: {
    main: "./src/pdf-viewer.ts",
    // ALWAYS create a separate entry for the worker
    "pdf.worker": "pdfjs-dist/build/pdf.worker.min.mjs",
  },
  output: {
    path: path.resolve(__dirname, "dist"),
    filename: "[name].bundle.js",
    clean: true,
  },
  resolve: {
    extensions: [".ts", ".js", ".mjs"],
  },
  module: {
    rules: [
      {
        test: /\.ts$/,
        use: "ts-loader",
        exclude: /node_modules/,
      },
    ],
  },
  plugins: [
    new CopyPlugin({
      patterns: [
        { from: "node_modules/pdfjs-dist/cmaps/", to: "cmaps/" },
        { from: "node_modules/pdfjs-dist/standard_fonts/", to: "standard_fonts/" },
      ],
    }),
  ],
};
```

### Application Code (Manual Worker Path)

```typescript
// src/pdf-viewer.ts
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";

// Point to the worker bundle created by the separate entry point
GlobalWorkerOptions.workerSrc = "/pdf.worker.bundle.js";

const doc = await getDocument("/document.pdf").promise;
```

---

## 3. Vite

### Application Code

```typescript
// src/pdf-viewer.ts
import * as pdfjsLib from "pdfjs-dist/webpack.mjs";

// Worker is auto-configured via webpack.mjs entry point
// Vite supports the same new URL() + new Worker() pattern as webpack 5

async function loadPdf(url: string): Promise<void> {
  const doc = await pdfjsLib.getDocument({
    url,
    cMapUrl: "/cmaps/",
    cMapPacked: true,
    standardFontDataUrl: "/standard_fonts/",
  }).promise;

  const page = await doc.getPage(1);
  const viewport = page.getViewport({ scale: 1.5 });

  const canvas = document.getElementById("pdf-canvas") as HTMLCanvasElement;
  const dpr = window.devicePixelRatio || 1;
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;

  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);

  await page.render({ canvasContext: ctx, viewport }).promise;
}
```

### Vite Config

```typescript
// vite.config.ts
import { defineConfig } from "vite";
import { viteStaticCopy } from "vite-plugin-static-copy";

export default defineConfig({
  optimizeDeps: {
    // ALWAYS exclude pdfjs-dist from pre-bundling
    // Pre-bundling breaks the import.meta.url worker resolution
    exclude: ["pdfjs-dist"],
  },
  plugins: [
    viteStaticCopy({
      targets: [
        {
          src: "node_modules/pdfjs-dist/cmaps/*",
          dest: "cmaps",
        },
        {
          src: "node_modules/pdfjs-dist/standard_fonts/*",
          dest: "standard_fonts",
        },
      ],
    }),
  ],
});
```

### Vite Alternative: Manual Worker Setup

```typescript
// src/pdf-viewer.ts
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";

// Vite resolves new URL() + import.meta.url at build time
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();
```

### Package Dependencies

```json
{
  "dependencies": {
    "pdfjs-dist": "^5.5.0"
  },
  "devDependencies": {
    "vite": "^6.0.0",
    "vite-plugin-static-copy": "^2.2.0",
    "typescript": "^5.4.0"
  }
}
```

---

## 4. Rollup

### Application Code

```typescript
// src/pdf-viewer.ts
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";

// Rollup: use new URL() pattern with @rollup/plugin-url
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

const doc = await getDocument({
  url: "/document.pdf",
  cMapUrl: "/cmaps/",
  cMapPacked: true,
  standardFontDataUrl: "/standard_fonts/",
}).promise;
```

### Rollup Config

```javascript
// rollup.config.mjs
import resolve from "@rollup/plugin-node-resolve";
import commonjs from "@rollup/plugin-commonjs";
import typescript from "@rollup/plugin-typescript";
import copy from "rollup-plugin-copy";

export default {
  input: "src/pdf-viewer.ts",
  output: {
    dir: "dist",
    format: "es",
    sourcemap: true,
  },
  plugins: [
    resolve({ browser: true }),
    commonjs(),
    typescript(),
    copy({
      targets: [
        // Copy worker file to output directory
        {
          src: "node_modules/pdfjs-dist/build/pdf.worker.min.mjs",
          dest: "dist",
        },
        // Copy CMap files
        {
          src: "node_modules/pdfjs-dist/cmaps/*",
          dest: "dist/cmaps",
        },
        // Copy standard font files
        {
          src: "node_modules/pdfjs-dist/standard_fonts/*",
          dest: "dist/standard_fonts",
        },
      ],
    }),
  ],
};
```

### Package Dependencies

```json
{
  "dependencies": {
    "pdfjs-dist": "^5.5.0"
  },
  "devDependencies": {
    "@rollup/plugin-commonjs": "^28.0.0",
    "@rollup/plugin-node-resolve": "^16.0.0",
    "@rollup/plugin-typescript": "^12.1.0",
    "rollup": "^4.28.0",
    "rollup-plugin-copy": "^3.5.0",
    "typescript": "^5.4.0"
  }
}
```

---

## 5. Next.js (App Router)

### Component Code

```typescript
// app/components/PdfViewer.tsx
"use client";

import { useEffect, useRef } from "react";
import type { PDFDocumentProxy } from "pdfjs-dist";

export default function PdfViewer({ url }: { url: string }) {
  const canvasRef = useRef<HTMLCanvasElement>(null);

  useEffect(() => {
    let doc: PDFDocumentProxy | null = null;

    async function render() {
      // ALWAYS use dynamic import for pdfjs-dist in Next.js
      // This prevents SSR from trying to load the browser-only library
      const pdfjsLib = await import("pdfjs-dist");

      // Worker MUST be in public/ directory for Next.js
      pdfjsLib.GlobalWorkerOptions.workerSrc = "/pdf.worker.min.mjs";

      doc = await pdfjsLib.getDocument({
        url,
        cMapUrl: "/cmaps/",
        cMapPacked: true,
        standardFontDataUrl: "/standard_fonts/",
      }).promise;

      const page = await doc.getPage(1);
      const viewport = page.getViewport({ scale: 1.5 });
      const canvas = canvasRef.current!;
      const dpr = window.devicePixelRatio || 1;

      canvas.width = Math.floor(viewport.width * dpr);
      canvas.height = Math.floor(viewport.height * dpr);
      canvas.style.width = `${Math.floor(viewport.width)}px`;
      canvas.style.height = `${Math.floor(viewport.height)}px`;

      const ctx = canvas.getContext("2d")!;
      ctx.scale(dpr, dpr);
      await page.render({ canvasContext: ctx, viewport }).promise;
    }

    render();

    return () => {
      doc?.destroy();
    };
  }, [url]);

  return <canvas ref={canvasRef} />;
}
```

### File Setup

Copy these files to your Next.js `public/` directory:

```bash
# Copy worker file
cp node_modules/pdfjs-dist/build/pdf.worker.min.mjs public/

# Copy CMap files (for CJK support)
cp -r node_modules/pdfjs-dist/cmaps public/cmaps

# Copy standard fonts
cp -r node_modules/pdfjs-dist/standard_fonts public/standard_fonts
```

### next.config.mjs

```javascript
// next.config.mjs
/** @type {import('next').NextConfig} */
const nextConfig = {
  // ALWAYS configure webpack to handle .mjs files in Next.js
  webpack: (config) => {
    config.resolve.alias.canvas = false;
    return config;
  },
};

export default nextConfig;
```

### Automate with postinstall Script

```json
{
  "scripts": {
    "postinstall": "cp node_modules/pdfjs-dist/build/pdf.worker.min.mjs public/ && cp -r node_modules/pdfjs-dist/cmaps public/cmaps && cp -r node_modules/pdfjs-dist/standard_fonts public/standard_fonts"
  }
}
```

---

## 6. Nuxt.js 3

### Component Code

```vue
<!-- components/PdfViewer.vue -->
<template>
  <!-- ALWAYS wrap in ClientOnly -- pdfjs-dist is browser-only -->
  <ClientOnly>
    <canvas ref="canvasRef" />
  </ClientOnly>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from "vue";
import type { PDFDocumentProxy } from "pdfjs-dist";

const props = defineProps<{ url: string }>();
const canvasRef = ref<HTMLCanvasElement | null>(null);
let doc: PDFDocumentProxy | null = null;

onMounted(async () => {
  // ALWAYS use dynamic import in Nuxt to avoid SSR issues
  const pdfjsLib = await import("pdfjs-dist");

  // Worker MUST be in public/ directory
  pdfjsLib.GlobalWorkerOptions.workerSrc = "/pdf.worker.min.mjs";

  doc = await pdfjsLib.getDocument({
    url: props.url,
    cMapUrl: "/cmaps/",
    cMapPacked: true,
    standardFontDataUrl: "/standard_fonts/",
  }).promise;

  const page = await doc.getPage(1);
  const viewport = page.getViewport({ scale: 1.5 });
  const canvas = canvasRef.value!;
  const dpr = window.devicePixelRatio || 1;

  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;

  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);
  await page.render({ canvasContext: ctx, viewport }).promise;
});

onUnmounted(() => {
  doc?.destroy();
});
</script>
```

### File Setup

```bash
# Copy worker to Nuxt 3 public/ directory
cp node_modules/pdfjs-dist/build/pdf.worker.min.mjs public/

# Copy CMap files
cp -r node_modules/pdfjs-dist/cmaps public/cmaps

# Copy standard fonts
cp -r node_modules/pdfjs-dist/standard_fonts public/standard_fonts
```

---

## 7. CDN (No Bundler)

```html
<!DOCTYPE html>
<html>
<body>
  <canvas id="pdf-canvas"></canvas>

  <script type="module">
    import * as pdfjsLib from "https://cdn.jsdelivr.net/npm/pdfjs-dist@5.5.207/build/pdf.min.mjs";

    // ALWAYS match the worker CDN version to the library version
    pdfjsLib.GlobalWorkerOptions.workerSrc =
      "https://cdn.jsdelivr.net/npm/pdfjs-dist@5.5.207/build/pdf.worker.min.mjs";

    const doc = await pdfjsLib.getDocument("/document.pdf").promise;
    const page = await doc.getPage(1);
    const viewport = page.getViewport({ scale: 1.5 });

    const canvas = document.getElementById("pdf-canvas");
    const dpr = window.devicePixelRatio || 1;
    canvas.width = Math.floor(viewport.width * dpr);
    canvas.height = Math.floor(viewport.height * dpr);
    canvas.style.width = `${Math.floor(viewport.width)}px`;
    canvas.style.height = `${Math.floor(viewport.height)}px`;

    const ctx = canvas.getContext("2d");
    ctx.scale(dpr, dpr);
    await page.render({ canvasContext: ctx, viewport }).promise;
  </script>
</body>
</html>
```
