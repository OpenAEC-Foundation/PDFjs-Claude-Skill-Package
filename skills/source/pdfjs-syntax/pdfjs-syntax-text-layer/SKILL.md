---
name: pdfjs-syntax-text-layer
description: "Creates selectable and searchable text overlays using the TextLayer class. Covers text content extraction, TextLayer rendering, CSS positioning, text selection, plain text extraction, and streaming text content. Activates when adding text selection to PDF viewer, extracting text from PDF, implementing PDF search, or fixing text layer positioning."
license: MIT
compatibility: "Designed for Claude Code. Requires pdfjs-dist 5.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# pdfjs-syntax-text-layer

## Quick Reference

### Text Layer Pipeline

| Step | Method | Output |
|------|--------|--------|
| 1. Render page to canvas | `page.render({ canvasContext, viewport })` | Rendered canvas |
| 2. Get text content | `page.getTextContent()` | `TextContent` object |
| 3. Create container div | Position absolutely over canvas | `<div>` element |
| 4. Create TextLayer | `new TextLayer({ textContentSource, container, viewport })` | `TextLayer` instance |
| 5. Render text layer | `textLayer.render()` | Promise (text spans in DOM) |

### Critical Warnings

**NEVER** use the deprecated `renderTextLayer()` function -- it was removed in pdfjs-dist 5.x. ALWAYS use the `TextLayer` class constructor and its `render()` method.

**ALWAYS** position the text layer container with `position: absolute` directly over the canvas -- without this, text selection coordinates will not align with the rendered PDF content.

**ALWAYS** set the text layer container to the same CSS dimensions as the canvas -- a size mismatch causes text spans to appear offset from their corresponding rendered glyphs.

**NEVER** forget to set `pointer-events: none` on the text layer container while keeping `pointer-events: all` on the individual text spans -- this allows click-through to the canvas for areas without text while preserving text selection.

**ALWAYS** include the PDF.js text layer CSS (`pdfjs-dist/web/pdf_viewer.css` or custom equivalent) -- without it, text spans render as visible black text on top of the canvas instead of invisible selectable overlays.

**ALWAYS** call `textLayer.cancel()` before creating a new TextLayer for the same page (e.g., on zoom/rotation) -- failing to cancel causes duplicate text spans and memory leaks.

---

## Essential Patterns

### Basic Text Layer Over Canvas

```typescript
import { getDocument, GlobalWorkerOptions, TextLayer } from "pdfjs-dist";
import type { PDFPageProxy, PageViewport } from "pdfjs-dist";

// ALWAYS configure worker before loading documents
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

async function renderPageWithTextLayer(
  url: string,
  pageNumber: number,
  container: HTMLDivElement,
  scale: number = 1.5
): Promise<void> {
  const doc = await getDocument(url).promise;
  const page = await doc.getPage(pageNumber);
  const viewport = page.getViewport({ scale });

  // Setup wrapper with relative positioning for absolute children
  container.style.position = "relative";
  container.style.width = `${Math.floor(viewport.width)}px`;
  container.style.height = `${Math.floor(viewport.height)}px`;

  // 1. Render canvas
  const canvas = document.createElement("canvas");
  const dpr = window.devicePixelRatio || 1;
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
  await page.render({ canvasContext: ctx, viewport }).promise;

  // 2. Get text content
  const textContent = await page.getTextContent();

  // 3. Create text layer container
  const textLayerDiv = document.createElement("div");
  textLayerDiv.className = "textLayer";
  textLayerDiv.style.position = "absolute";
  textLayerDiv.style.top = "0";
  textLayerDiv.style.left = "0";
  textLayerDiv.style.width = `${Math.floor(viewport.width)}px`;
  textLayerDiv.style.height = `${Math.floor(viewport.height)}px`;
  container.appendChild(textLayerDiv);

  // 4. Create and render TextLayer
  const textLayer = new TextLayer({
    textContentSource: textContent,
    container: textLayerDiv,
    viewport: viewport,
  });

  await textLayer.render();
}
```

### Required CSS for Text Layer

```css
/* Minimal CSS for invisible selectable text overlay */
.textLayer {
  position: absolute;
  text-align: initial;
  inset: 0;
  overflow: hidden;
  opacity: 1;
  line-height: 1;
  -webkit-text-size-adjust: none;
  -moz-text-size-adjust: none;
  text-size-adjust: none;
  forced-color-adjust: none;
  z-index: 1;
}

.textLayer span,
.textLayer br {
  color: transparent;
  position: absolute;
  white-space: pre;
  cursor: text;
  transform-origin: 0% 0%;
  pointer-events: all;
}

/* Text selection highlight */
.textLayer ::selection {
  background: rgba(0, 0, 255, 0.25);
}
```

> **NOTE**: For production use, ALWAYS import the full CSS from `pdfjs-dist/web/pdf_viewer.css` instead of maintaining custom CSS. The snippet above shows the minimum required properties for understanding.

### Plain Text Extraction

```typescript
async function extractPageText(page: PDFPageProxy): Promise<string> {
  const textContent = await page.getTextContent();
  const lines: string[] = [];
  let currentLine = "";

  for (const item of textContent.items) {
    // Skip marked content items (they have no str property)
    if ("str" in item) {
      currentLine += item.str;
      if (item.hasEOL) {
        lines.push(currentLine);
        currentLine = "";
      }
    }
  }

  // Push remaining text
  if (currentLine) {
    lines.push(currentLine);
  }

  return lines.join("\n");
}
```

### Streaming Text Content

```typescript
async function extractTextStreaming(page: PDFPageProxy): Promise<string> {
  // streamTextContent returns a ReadableStream -- use for large documents
  // to avoid loading entire text content into memory at once
  const stream = page.streamTextContent({
    includeMarkedContent: false,
    disableNormalization: false,
  });

  const reader = stream.getReader();
  const parts: string[] = [];

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    // Each chunk contains partial TextContent
    for (const item of value.items) {
      if ("str" in item) {
        parts.push(item.str);
        if (item.hasEOL) {
          parts.push("\n");
        }
      }
    }
  }

  return parts.join("");
}
```

### Update Text Layer on Zoom/Rotation

```typescript
import { TextLayer } from "pdfjs-dist";

let currentTextLayer: TextLayer | null = null;

async function updateTextLayer(
  page: PDFPageProxy,
  textLayerDiv: HTMLDivElement,
  newScale: number,
  rotation: number = 0
): Promise<void> {
  const newViewport = page.getViewport({ scale: newScale, rotation });

  if (currentTextLayer) {
    // Update existing text layer -- repositions spans without re-creating them
    currentTextLayer.update({
      viewport: newViewport,
      onBefore() {
        // Called before DOM updates -- use for pre-update cleanup
        textLayerDiv.style.width = `${Math.floor(newViewport.width)}px`;
        textLayerDiv.style.height = `${Math.floor(newViewport.height)}px`;
      },
    });
  } else {
    // First render -- create new TextLayer
    const textContent = await page.getTextContent();
    currentTextLayer = new TextLayer({
      textContentSource: textContent,
      container: textLayerDiv,
      viewport: newViewport,
    });
    await currentTextLayer.render();
  }
}
```

### Cancel and Cleanup

```typescript
function destroyTextLayer(textLayer: TextLayer | null): void {
  if (textLayer) {
    // Cancel any in-progress rendering
    textLayer.cancel();
  }

  // Clear global caches when no more text layers are active
  // ONLY call this when ALL text layers in the application are destroyed
  TextLayer.cleanup();
}
```

---

## Decision Tree: Text Layer Setup

```
Need text functionality on PDF page?
├── Need selectable/searchable text overlay?
│   ├── YES → Use TextLayer class with container over canvas
│   │   ├── First render? → new TextLayer() + render()
│   │   ├── Zoom/rotation changed? → textLayer.update({ viewport })
│   │   └── Page destroyed? → textLayer.cancel() + remove container
│   └── NO → Use page.getTextContent() for data only
│
├── Need plain text extraction?
│   ├── Small document → page.getTextContent() + concatenate item.str
│   └── Large document → page.streamTextContent() for streaming
│
├── Need text positions/coordinates?
│   └── Use TextItem.transform matrix from getTextContent()
│       transform[4] = x position, transform[5] = y position
│
└── Need to search within PDF text?
    └── Extract text with getTextContent() → search item.str values
        → highlight matching spans via textLayer.textDivs
```

---

## Reference Links

- [references/methods.md](references/methods.md) -- Complete API signatures for TextLayer, getTextContent, TextContent, and TextItem
- [references/examples.md](references/examples.md) -- Working code examples for text layer rendering, extraction, and streaming
- [references/anti-patterns.md](references/anti-patterns.md) -- Common text layer mistakes and their fixes

### Official Sources

- https://mozilla.github.io/pdf.js/api/ -- PDF.js API reference
- https://github.com/mozilla/pdf.js/blob/master/src/display/text_layer.js -- TextLayer source code
- https://github.com/mozilla/pdf.js/tree/master/examples -- Official examples
