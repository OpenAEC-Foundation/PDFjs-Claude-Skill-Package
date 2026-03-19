---
name: pdfjs-core-architecture
description: >
  Use when starting a PDF.js project, understanding PDF.js internals, or reasoning
  about the rendering pipeline. Prevents the common mistake of misunderstanding the
  three-layer model (Core/Display/Viewer) and worker thread architecture.
  Covers component hierarchy, pdfjs-dist 5.x package structure, and rendering pipeline overview.
  Keywords: PDF.js, pdfjs-dist, architecture, worker, PDFDocumentProxy, PDFPageProxy, layers.
license: MIT
compatibility: "Designed for Claude Code. Requires pdfjs-dist 5.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# pdfjs-core-architecture

## Quick Reference

### Three-Layer Architecture (pdfjs-dist 5.x)

| Layer | File | Thread | Role |
|-------|------|--------|------|
| Core | `pdf.worker.mjs` | Web Worker | Binary PDF parsing, stream decoding, font processing. No public API -- NEVER import directly. |
| Display | `pdf.mjs` | Main thread | Public API surface: `getDocument()`, proxy objects, layer classes. This is what you import. |
| Viewer | `viewer.mjs` | Main thread | Mozilla's reference UI application. NEVER use as a library -- build your own viewer with the Display API. |

### Package Structure (pdfjs-dist 5.x)

| Path | Purpose |
|------|---------|
| `build/pdf.mjs` | Display layer -- main entry point for all applications |
| `build/pdf.worker.mjs` | Core layer -- set path via `GlobalWorkerOptions.workerSrc` |
| `build/pdf.sandbox.mjs` | Sandboxed JavaScript evaluation for PDF forms |
| `cmaps/` | Character maps for CJK fonts -- set path via `cMapUrl` option |
| `standard_fonts/` | Standard 14 PDF fonts -- set path via `standardFontDataUrl` option |
| `types/` | TypeScript type definitions |

### Key Types Overview

| Type | Import | Purpose |
|------|--------|---------|
| `GlobalWorkerOptions` | `pdfjs-dist` | Configure worker script path before any PDF loading |
| `getDocument()` | `pdfjs-dist` | Entry point -- returns `PDFDocumentLoadingTask` |
| `PDFDocumentLoadingTask` | returned by `getDocument()` | Loading handle with `.promise` property |
| `PDFDocumentProxy` | resolved from loading task | Document-level operations: page access, metadata, outline |
| `PDFPageProxy` | resolved from `getPage()` | Page-level operations: render, text content, annotations |
| `PageViewport` | returned by `getViewport()` | Coordinate system for rendering at a given scale/rotation |
| `RenderTask` | returned by `render()` | Cancellable render operation with `.promise` |
| `TextLayer` | `pdfjs-dist` | Overlay for selectable/searchable text |
| `AnnotationLayer` | `pdfjs-dist` | Overlay for links, form fields, annotations |

### Layer Stacking Order

```
┌─────────────────────────────────┐
│  AnnotationLayer (z-index: 2)   │  ← Links, forms, interactive elements
├─────────────────────────────────┤
│  TextLayer (z-index: 1)         │  ← Selectable, searchable text overlay
├─────────────────────────────────┤
│  Canvas (z-index: 0)            │  ← Visual pixel rendering of PDF page
└─────────────────────────────────┘
```

ALWAYS stack layers in this order. The canvas provides the visual rendering, the TextLayer enables text selection, and the AnnotationLayer handles interactive elements on top.

### Critical Warnings

**NEVER** call `getDocument()` before setting `GlobalWorkerOptions.workerSrc` -- the worker cannot be found and loading silently fails or throws.

**NEVER** render all pages at once -- ALWAYS use lazy/virtualized loading. A 500-page PDF rendered simultaneously will exhaust memory and crash the tab.

**NEVER** skip `devicePixelRatio` handling -- rendering without scaling for HiDPI displays produces blurry output on every modern screen.

**NEVER** start a new render without cancelling the previous `RenderTask` -- concurrent renders on the same canvas corrupt the graphics state and throw `RenderingCancelledException`.

**NEVER** use the deprecated `renderTextLayer()` function -- ALWAYS use the `TextLayer` class (pdfjs-dist 5.x).

**ALWAYS** ensure the worker version matches the pdfjs-dist version exactly -- version mismatches cause silent failures or explicit errors.

**ALWAYS** call `PDFDocumentProxy.destroy()` when done to release worker resources and prevent memory leaks.

---

## Worker Thread Model

PDF.js uses a **two-thread architecture** for performance:

### Main Thread (Display Layer)

- Runs your application code and the Display API (`pdf.mjs`)
- Creates proxy objects (`PDFDocumentProxy`, `PDFPageProxy`) that represent worker-side data
- Manages canvas rendering, TextLayer, and AnnotationLayer
- Handles user interaction and DOM manipulation

### Worker Thread (Core Layer)

- Runs `pdf.worker.mjs` in a Web Worker
- Performs CPU-intensive work: PDF binary parsing, stream decompression, font decoding
- Communicates with the main thread via structured cloning (MessageHandler)
- Has NO DOM access -- all rendering decisions are relayed to the main thread

### Communication Flow

```
Main Thread (Display)              Worker Thread (Core)
       |                                   |
       |--- getDocument(source) --------->|
       |                                   |--- parse PDF binary
       |                                   |--- extract xref table
       |<-- PDFDocumentProxy -------------|
       |                                   |
       |--- getPage(pageNum) ------------>|
       |                                   |--- decode page streams
       |<-- PDFPageProxy -----------------|
       |                                   |
       |--- render() -------------------->|
       |                                   |--- build operator list
       |<-- operator list ----------------|
       |--- execute on canvas             |
       |                                   |
       |--- getTextContent() ------------>|
       |                                   |--- extract text items
       |<-- text content -----------------|
```

### Worker Configuration

```typescript
import { GlobalWorkerOptions } from 'pdfjs-dist';

// ALWAYS set workerSrc before calling getDocument()
GlobalWorkerOptions.workerSrc = new URL(
  'pdfjs-dist/build/pdf.worker.mjs',
  import.meta.url
).toString();
```

**Bundler-specific patterns**: Vite, Webpack, and other bundlers each require different worker configuration. See [references/examples.md](references/examples.md) for bundler-specific setups.

---

## Component Hierarchy

The complete object hierarchy from initialization to rendering:

```
GlobalWorkerOptions.workerSrc = '...'
        │
        ▼
getDocument(source)
        │
        ▼
PDFDocumentLoadingTask
  ├── .promise → PDFDocumentProxy
  ├── .onProgress(callback)
  └── .destroy()
               │
               ▼
        PDFDocumentProxy
          ├── .numPages
          ├── .getPage(num) → PDFPageProxy
          ├── .getMetadata()
          ├── .getOutline()
          ├── .getAttachments()
          ├── .getDestinations()
          ├── .destroy()
          │
          ▼
        PDFPageProxy
          ├── .pageNumber
          ├── .getViewport({scale, rotation}) → PageViewport
          ├── .render({canvasContext, viewport}) → RenderTask
          ├── .getTextContent() → TextContent
          ├── .getAnnotations() → Annotation[]
          ├── .cleanup()
          │
          ├──▶ RenderTask
          │     ├── .promise → void
          │     └── .cancel()
          │
          ├──▶ TextLayer
          │     ├── new TextLayer({textContentSource, container, viewport})
          │     ├── .render()
          │     └── .cancel()
          │
          └──▶ AnnotationLayer
                ├── .render()
                ├── .update()
                └── .cancel()
```

---

## Rendering Pipeline

The complete pipeline from configuration to a fully interactive PDF page:

### Step 1: Configure Worker

```typescript
import { GlobalWorkerOptions, getDocument } from 'pdfjs-dist';

GlobalWorkerOptions.workerSrc = '/pdf.worker.mjs';
```

### Step 2: Load Document

```typescript
const loadingTask = getDocument({ url: '/document.pdf' });
loadingTask.onProgress = ({ loaded, total }) => {
  console.log(`${Math.round(loaded / total * 100)}%`);
};
const pdfDoc = await loadingTask.promise;
```

### Step 3: Get Page and Viewport

```typescript
const page = await pdfDoc.getPage(1); // 1-based page numbers
const scale = 1.5;
const viewport = page.getViewport({ scale });
```

### Step 4: Render to Canvas

```typescript
const canvas = document.createElement('canvas');
const context = canvas.getContext('2d');
const dpr = window.devicePixelRatio || 1;

canvas.width = Math.floor(viewport.width * dpr);
canvas.height = Math.floor(viewport.height * dpr);
canvas.style.width = `${Math.floor(viewport.width)}px`;
canvas.style.height = `${Math.floor(viewport.height)}px`;
context.scale(dpr, dpr);

const renderTask = page.render({ canvasContext: context, viewport });
await renderTask.promise;
```

### Step 5: Add TextLayer

```typescript
import { TextLayer } from 'pdfjs-dist';

const textLayerDiv = document.createElement('div');
textLayerDiv.className = 'textLayer';

const textLayer = new TextLayer({
  textContentSource: page.streamTextContent(),
  container: textLayerDiv,
  viewport,
});
await textLayer.render();
```

### Step 6: Add AnnotationLayer

```typescript
import { AnnotationLayer } from 'pdfjs-dist';

const annotationLayerDiv = document.createElement('div');
annotationLayerDiv.className = 'annotationLayer';

// AnnotationLayer requires AnnotationStorage and link service
// See references/examples.md for complete setup
```

### Step 7: Cleanup

```typescript
// ALWAYS destroy when done
pdfDoc.destroy();
```

---

## Reference Links

- [references/methods.md](references/methods.md) -- API signatures for PDFDocumentProxy, PDFPageProxy, GlobalWorkerOptions, TextLayer, AnnotationLayer
- [references/examples.md](references/examples.md) -- Working code examples for common setups (Vite, Webpack, vanilla)
- [references/anti-patterns.md](references/anti-patterns.md) -- What NOT to do, with explanations

### Official Sources

- https://mozilla.github.io/pdf.js/api/
- https://github.com/nicolo-ribaudo/pdfjs-dist/blob/master/types/src/display/api.d.ts
- https://github.com/nicolo-ribaudo/pdfjs-dist
- https://mozilla.github.io/pdf.js/getting_started/
