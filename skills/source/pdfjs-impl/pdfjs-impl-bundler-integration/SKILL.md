---
name: pdfjs-impl-bundler-integration
description: >
  Use when integrating pdfjs-dist with webpack, Vite, Rollup, Next.js, or Nuxt.js
  build systems. Prevents worker loading failures caused by incorrect bundler
  configuration for the pdf.worker.mjs file.
  Covers webpack worker-loader/asset module config, Vite import.meta.url pattern,
  Rollup config, CMap/font file copying, and tree-shaking considerations.
  Keywords: webpack, vite, rollup, Next.js, bundler, worker-loader, import.meta.url.
license: MIT
compatibility: "Designed for Claude Code. Requires pdfjs-dist 5.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# pdfjs-impl-bundler-integration

## Quick Reference

### pdfjs-dist 5.x Package Structure

| Path | Purpose | Must Be Served |
|------|---------|----------------|
| `build/pdf.mjs` | Main library (ESM) | Bundled into app |
| `build/pdf.min.mjs` | Minified main library | Bundled into app |
| `build/pdf.worker.mjs` | Worker script (ESM) | Separate file or worker port |
| `build/pdf.worker.min.mjs` | Minified worker | Separate file or worker port |
| `cmaps/*.bcmap` | CJK character maps (169 files) | Static assets |
| `standard_fonts/*.pfb/.ttf` | Standard PDF fonts (16 files) | Static assets |
| `webpack.mjs` | Zero-config webpack/Vite entry | Import instead of `build/pdf.mjs` |

### Critical Warnings

**ALWAYS** use `pdfjs-dist/webpack.mjs` as the import path when using webpack 5+ or Vite -- this file auto-configures the worker using `import.meta.url` and `new Worker()`, eliminating all manual worker path setup.

**NEVER** set `GlobalWorkerOptions.workerSrc` to a hardcoded path like `"/pdf.worker.js"` -- bundlers change output paths. Use the `webpack.mjs` entry point or the `new URL(..., import.meta.url)` pattern instead.

**ALWAYS** copy `cmaps/` and `standard_fonts/` to your public/static directory when your application handles CJK PDFs or PDFs with embedded standard fonts -- these files are NOT bundled automatically.

**NEVER** import `pdf.worker.mjs` directly into your main bundle -- the worker MUST run in a separate thread. Importing it into the main bundle defeats the purpose and blocks the UI thread.

**ALWAYS** use `.mjs` file extensions when referencing pdfjs-dist 5.x files -- the package ships ESM-only (no `.js` or `.cjs` files in the modern build).

**NEVER** mix pdfjs-dist versions between the main library and worker -- the worker version MUST match exactly or you get silent failures and corrupted renders.

---

## Decision Tree: Which Setup to Use

```
Which bundler are you using?
|
+-- Webpack 5+
|   +-- Use `import * as pdfjsLib from "pdfjs-dist/webpack.mjs"`
|   +-- DONE. Worker is auto-configured via workerPort.
|   +-- Need CJK/fonts? -> Add copy-webpack-plugin (see references/examples.md)
|
+-- Vite
|   +-- Use `import * as pdfjsLib from "pdfjs-dist/webpack.mjs"`
|   +-- Vite supports the same new URL() + new Worker() pattern
|   +-- Add pdfjs-dist to optimizeDeps.exclude in vite.config.ts
|   +-- Need CJK/fonts? -> Use vite-plugin-static-copy
|
+-- Rollup
|   +-- Manual worker setup with new URL() pattern
|   +-- Use @rollup/plugin-copy for static assets
|   +-- See references/examples.md for full config
|
+-- Next.js
|   +-- Use dynamic import with { ssr: false }
|   +-- Copy worker to public/ directory
|   +-- Set workerSrc to "/pdf.worker.min.mjs"
|   +-- See references/examples.md for full config
|
+-- Nuxt.js
|   +-- Wrap in <ClientOnly> component
|   +-- Copy worker to static/ (Nuxt 2) or public/ (Nuxt 3)
|   +-- See references/examples.md for full config
|
+-- No bundler (CDN / script tag)
|   +-- Use unpkg or jsdelivr CDN URLs
|   +-- Set GlobalWorkerOptions.workerSrc to CDN URL
```

---

## Essential Patterns

### Webpack 5+ (Zero-Config)

```typescript
// ALWAYS use webpack.mjs -- it auto-configures the worker
import * as pdfjsLib from "pdfjs-dist/webpack.mjs";

// Worker is ALREADY configured via workerPort -- no manual setup needed
const doc = await pdfjsLib.getDocument("/document.pdf").promise;
```

The `webpack.mjs` entry point does this internally:

```typescript
// What webpack.mjs does under the hood:
GlobalWorkerOptions.workerPort = new Worker(
  new URL("./build/pdf.worker.mjs", import.meta.url),
  { type: "module" }
);
```

Webpack 5+ recognizes the `new URL(..., import.meta.url)` pattern and emits the worker as a separate asset automatically.

### Vite

```typescript
// Vite also supports the webpack.mjs entry point
import * as pdfjsLib from "pdfjs-dist/webpack.mjs";

// Worker is auto-configured -- no manual setup needed
const doc = await pdfjsLib.getDocument("/document.pdf").promise;
```

```typescript
// vite.config.ts
import { defineConfig } from "vite";

export default defineConfig({
  optimizeDeps: {
    // ALWAYS exclude pdfjs-dist from dependency pre-bundling
    // Pre-bundling breaks the worker URL resolution
    exclude: ["pdfjs-dist"],
  },
});
```

### Manual Worker Setup (when webpack.mjs is not suitable)

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";

// Option A: Using import.meta.url (works in webpack 5+, Vite, Rollup)
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

// Option B: Using workerPort directly (preferred -- avoids URL resolution)
GlobalWorkerOptions.workerPort = new Worker(
  new URL("pdfjs-dist/build/pdf.worker.min.mjs", import.meta.url),
  { type: "module" }
);
```

### CMap and Font Configuration

```typescript
import * as pdfjsLib from "pdfjs-dist/webpack.mjs";

const doc = await pdfjsLib.getDocument({
  url: "/document.pdf",

  // ALWAYS set cMapUrl when handling CJK (Chinese, Japanese, Korean) PDFs
  cMapUrl: "/cmaps/",
  cMapPacked: true, // ALWAYS true -- pdfjs-dist ships .bcmap (binary packed) files

  // ALWAYS set standardFontDataUrl when PDFs use standard 14 fonts
  // without embedding them
  standardFontDataUrl: "/standard_fonts/",
}).promise;
```

---

## CMap and Font File Copying

### Source Locations in node_modules

| Asset | Source Path | File Count |
|-------|------------|------------|
| CMap files | `node_modules/pdfjs-dist/cmaps/` | 169 `.bcmap` files |
| Standard fonts | `node_modules/pdfjs-dist/standard_fonts/` | 16 files (`.pfb`, `.ttf`) |

### When to Copy

| Scenario | CMap Files Needed | Standard Fonts Needed |
|----------|-------------------|-----------------------|
| English-only PDFs with embedded fonts | No | No |
| CJK PDFs (Chinese, Japanese, Korean) | Yes | No |
| PDFs referencing standard 14 fonts | No | Yes |
| General-purpose PDF viewer | Yes | Yes |

---

## Tree-Shaking Considerations

### What CAN be tree-shaken

- `AnnotationLayer` -- omit if not rendering annotations
- `AnnotationEditorLayer` -- omit if not enabling annotation editing
- `TextLayer` -- omit if not rendering selectable text
- `XfaLayer` -- omit if not rendering XFA forms
- `SignatureExtractor` -- omit if not extracting signatures

### What CANNOT be tree-shaken

- `getDocument` -- core functionality, always needed
- `GlobalWorkerOptions` -- worker setup, always needed
- `PDFWorker` -- worker management, always needed
- The worker file itself -- MUST be loaded as a separate file

### Import Only What You Need

```typescript
// GOOD: Import specific exports for tree-shaking
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";

// AVOID: Namespace import prevents tree-shaking in some bundlers
import * as pdfjsLib from "pdfjs-dist";
```

**Note**: When using `pdfjs-dist/webpack.mjs`, you MUST use `import *` because it re-exports everything and sets up the worker as a side effect. Tree-shaking individual exports requires importing from `pdfjs-dist` directly with manual worker setup.

---

## Reference Links

- [references/methods.md](references/methods.md) -- Bundler-specific configuration options and API parameters
- [references/examples.md](references/examples.md) -- Complete copy-paste configs for webpack, Vite, Rollup, Next.js, and Nuxt.js
- [references/anti-patterns.md](references/anti-patterns.md) -- Common bundler integration mistakes and fixes

### Official Sources

- https://github.com/nicolo-ribaudo/pdf.js/tree/master/examples/webpack -- Official webpack example
- https://github.com/nicolo-ribaudo/pdf.js -- Source code and types
- https://mozilla.github.io/pdf.js/api/ -- PDF.js API reference
