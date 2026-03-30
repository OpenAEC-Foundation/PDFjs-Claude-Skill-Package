---
name: pdfjs-syntax-worker-setup
description: >
  Use when configuring the PDF.js Web Worker for document parsing or fixing worker
  loading errors. Prevents the #1 PDF.js mistake: version mismatch between pdfjs-dist
  and the worker file. Covers GlobalWorkerOptions.workerSrc, CDN URLs, webpack/vite/rollup
  bundler configuration, fake worker mode, CMap and standard font setup.
  Keywords: workerSrc, GlobalWorkerOptions, pdf.worker.mjs, CDN, webpack, vite, CMap.
license: MIT
compatibility: "Designed for Claude Code. Requires pdfjs-dist 5.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# pdfjs-syntax-worker-setup

## Quick Reference

### Worker Configuration Properties

| Property | Type | Purpose |
|----------|------|---------|
| `GlobalWorkerOptions.workerSrc` | `string` | URL or path to `pdf.worker.min.mjs` — MUST be set before any `getDocument()` call |
| `GlobalWorkerOptions.workerPort` | `Worker` | Pre-created Worker instance — use for fake worker or custom worker |
| `DocumentInitParameters.cMapUrl` | `string` | Path to `cmaps/` directory — required for CJK text rendering |
| `DocumentInitParameters.cMapPacked` | `boolean` | ALWAYS set to `true` when using cMapUrl (binary CMap format) |
| `DocumentInitParameters.standardFontDataUrl` | `string` | Path to `standard_fonts/` directory — required for standard 14 PDF fonts |

### CDN URL Patterns (pdfjs-dist 5.x)

| CDN | URL Pattern |
|-----|-------------|
| cdnjs | `https://cdnjs.cloudflare.com/ajax/libs/pdf.js/{VERSION}/pdf.worker.min.mjs` |
| unpkg | `https://unpkg.com/pdfjs-dist@{VERSION}/build/pdf.worker.min.mjs` |
| jsdelivr | `https://cdn.jsdelivr.net/npm/pdfjs-dist@{VERSION}/build/pdf.worker.min.mjs` |

### Bundler Support

| Bundler | Approach | Key Pattern |
|---------|----------|-------------|
| Webpack 5 | `new URL()` + asset/resource | `new URL('pdfjs-dist/build/pdf.worker.min.mjs', import.meta.url)` |
| Vite | `?url` import suffix | `import workerUrl from 'pdfjs-dist/build/pdf.worker.min.mjs?url'` |
| Rollup | Copy plugin or CDN fallback | Copy `pdf.worker.min.mjs` to output directory |
| Next.js | CDN or copy to `public/` | CDN recommended; avoid SSR worker instantiation |

### Critical Warnings

**NEVER** call `getDocument()` before setting `GlobalWorkerOptions.workerSrc` — causes silent failures or an automatic fallback to fake worker mode, which runs parsing on the main thread and blocks the UI.

**NEVER** mix pdfjs-dist versions between `pdf.mjs` and `pdf.worker.mjs` — causes the error `"API version does not match Worker version"`. The worker file version MUST match the npm package version EXACTLY.

**ALWAYS** pin the CDN version to match your installed `pdfjs-dist` npm version EXACTLY. Using `latest` or a mismatched version causes version mismatch errors at runtime.

**ALWAYS** configure `cMapUrl` when rendering PDFs containing CJK (Chinese/Japanese/Korean) text — without it, CJK characters render as blank rectangles or tofu.

**NEVER** use a relative path for `workerSrc` without understanding your bundler's base URL — the worker file is loaded relative to the document origin, not the script file.

**ALWAYS** set `cMapPacked: true` when using `cMapUrl` — pdfjs-dist ships binary CMaps, and omitting this flag causes CMap parsing failures.

---

## Decision Tree: Choose Your Setup Method

```
Need PDF.js worker setup?
│
├── No bundler (plain HTML / script tags)?
│   └── USE CDN setup → simplest, zero config
│       └── Pick: cdnjs (most popular) | unpkg | jsdelivr
│
├── Using a bundler?
│   ├── Webpack 5?
│   │   └── USE new URL() + import.meta.url pattern
│   │       └── Webpack handles it as asset/resource automatically
│   │
│   ├── Vite?
│   │   └── USE ?url import suffix
│   │       └── Vite resolves the URL at build time
│   │
│   ├── Rollup?
│   │   └── USE copy plugin OR CDN fallback
│   │       └── Rollup does not handle new URL() natively
│   │
│   └── Next.js?
│       └── USE CDN for simplicity OR copy to public/
│           └── Avoid worker instantiation during SSR
│
└── Testing or SSR environment?
    └── USE fake worker mode
        └── Import pdf.worker.mjs directly (no Web Worker thread)
```

---

## Complete Setup Patterns

### 1. CDN Setup (Simplest: No Bundler)

```javascript
import * as pdfjsLib from 'pdfjs-dist';

// ALWAYS set workerSrc BEFORE any getDocument() call
pdfjsLib.GlobalWorkerOptions.workerSrc =
  'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/5.5.207/pdf.worker.min.mjs';

// Now safe to load documents
const loadingTask = pdfjsLib.getDocument('document.pdf');
const pdf = await loadingTask.promise;
```

### 2. Webpack 5 Setup

```javascript
import * as pdfjsLib from 'pdfjs-dist';

// Webpack 5 resolves new URL() as asset/resource
pdfjsLib.GlobalWorkerOptions.workerSrc = new URL(
  'pdfjs-dist/build/pdf.worker.min.mjs',
  import.meta.url
).toString();

const loadingTask = pdfjsLib.getDocument('document.pdf');
const pdf = await loadingTask.promise;
```

### 3. Vite Setup

```javascript
import * as pdfjsLib from 'pdfjs-dist';
import workerUrl from 'pdfjs-dist/build/pdf.worker.min.mjs?url';

// Vite resolves ?url imports to the asset URL at build time
pdfjsLib.GlobalWorkerOptions.workerSrc = workerUrl;

const loadingTask = pdfjsLib.getDocument('document.pdf');
const pdf = await loadingTask.promise;
```

### 4. Full Setup with CMap and Standard Fonts

```javascript
import * as pdfjsLib from 'pdfjs-dist';

// Set worker source first
pdfjsLib.GlobalWorkerOptions.workerSrc =
  'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/5.5.207/pdf.worker.min.mjs';

// Load document with CMap and font support
const loadingTask = pdfjsLib.getDocument({
  url: 'document.pdf',
  cMapUrl: '/cmaps/',
  cMapPacked: true,
  standardFontDataUrl: '/standard_fonts/',
});
const pdf = await loadingTask.promise;
```

### 5. Fake Worker Mode (Testing / SSR)

```javascript
import * as pdfjsLib from 'pdfjs-dist';

// Import worker code directly — runs on main thread, no Web Worker
import 'pdfjs-dist/build/pdf.worker.min.mjs';

// No workerSrc needed — PDF.js detects the inline worker
const loadingTask = pdfjsLib.getDocument('document.pdf');
const pdf = await loadingTask.promise;
```

**WARNING**: Fake worker mode blocks the main thread during PDF parsing. ONLY use for testing, SSR pre-rendering, or Node.js environments where Web Workers are unavailable.

---

## Version Matching

ALWAYS ensure version alignment between the npm package and the worker file:

```bash
# Check your installed version
npm list pdfjs-dist
# Output: pdfjs-dist@5.5.207

# Your workerSrc MUST use the same version
# CORRECT:
# 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/5.5.207/pdf.worker.min.mjs'
#
# WRONG (version mismatch):
# 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/4.0.379/pdf.worker.min.mjs'
```

To automate version matching:

```javascript
import * as pdfjsLib from 'pdfjs-dist';
import { version } from 'pdfjs-dist/package.json';

pdfjsLib.GlobalWorkerOptions.workerSrc =
  `https://cdnjs.cloudflare.com/ajax/libs/pdf.js/${version}/pdf.worker.min.mjs`;
```

---

## Reference Links

- [references/methods.md](references/methods.md) -- GlobalWorkerOptions API, all properties and configuration objects
- [references/examples.md](references/examples.md) -- Worker setup for CDN, webpack, vite, rollup, next.js
- [references/anti-patterns.md](references/anti-patterns.md) -- Common worker setup mistakes

### Official Sources

- https://mozilla.github.io/pdf.js/api/
- https://github.com/nicolo-ribaudo/pdfjs-dist
- https://github.com/nicolo-ribaudo/pdfjs-dist/blob/master/types/src/display/api.d.ts
- https://mozilla.github.io/pdf.js/getting_started/
