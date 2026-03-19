# Code Examples (pdfjs-dist 5.x Text Layer)

## Complete Text Layer with Canvas

Full working example showing canvas rendering with a selectable text overlay.

```typescript
import { getDocument, GlobalWorkerOptions, TextLayer } from "pdfjs-dist";
import type { PDFDocumentProxy, PDFPageProxy, PageViewport } from "pdfjs-dist";

// ALWAYS configure worker before loading documents
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

interface PageView {
  canvas: HTMLCanvasElement;
  textLayerDiv: HTMLDivElement;
  textLayer: TextLayer | null;
}

async function createPageView(
  page: PDFPageProxy,
  container: HTMLDivElement,
  scale: number = 1.5
): Promise<PageView> {
  const viewport = page.getViewport({ scale });

  // Wrapper holds canvas + text layer in stacking order
  container.style.position = "relative";
  container.style.width = `${Math.floor(viewport.width)}px`;
  container.style.height = `${Math.floor(viewport.height)}px`;

  // Canvas layer (z-index: 0)
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

  // Text layer (z-index: 1) -- MUST be above canvas
  const textLayerDiv = document.createElement("div");
  textLayerDiv.className = "textLayer";
  textLayerDiv.style.position = "absolute";
  textLayerDiv.style.top = "0";
  textLayerDiv.style.left = "0";
  textLayerDiv.style.width = `${Math.floor(viewport.width)}px`;
  textLayerDiv.style.height = `${Math.floor(viewport.height)}px`;
  textLayerDiv.style.zIndex = "1";
  container.appendChild(textLayerDiv);

  // Get text content and render text layer
  const textContent = await page.getTextContent();
  const textLayer = new TextLayer({
    textContentSource: textContent,
    container: textLayerDiv,
    viewport: viewport,
  });
  await textLayer.render();

  return { canvas, textLayerDiv, textLayer };
}
```

---

## Text Layer with Streaming Content

Uses `streamTextContent()` to start rendering text spans before the full content is available.

```typescript
import { TextLayer } from "pdfjs-dist";
import type { PDFPageProxy, PageViewport } from "pdfjs-dist";

async function renderTextLayerStreaming(
  page: PDFPageProxy,
  textLayerDiv: HTMLDivElement,
  viewport: PageViewport
): Promise<TextLayer> {
  // streamTextContent returns a ReadableStream
  // TextLayer accepts this directly as textContentSource
  const stream = page.streamTextContent({
    includeMarkedContent: false,
  });

  const textLayer = new TextLayer({
    textContentSource: stream,  // Pass stream directly -- TextLayer handles it
    container: textLayerDiv,
    viewport: viewport,
  });

  await textLayer.render();
  return textLayer;
}
```

---

## Extract All Text from Document

Extracts plain text from every page in a PDF document.

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";

GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

async function extractDocumentText(url: string): Promise<string> {
  const doc = await getDocument(url).promise;
  const pageTexts: string[] = [];

  for (let i = 1; i <= doc.numPages; i++) {
    const page = await doc.getPage(i);
    const textContent = await page.getTextContent();
    const pageText: string[] = [];

    for (const item of textContent.items) {
      if ("str" in item) {
        pageText.push(item.str);
        if (item.hasEOL) {
          pageText.push("\n");
        }
      }
    }

    pageTexts.push(pageText.join(""));
  }

  return pageTexts.join("\n\n");  // Double newline between pages
}
```

---

## Text Search with Highlighting

Searches text content and highlights matching spans in the text layer.

```typescript
import type { TextLayer } from "pdfjs-dist";

interface SearchMatch {
  itemIndex: number;
  textDiv: HTMLElement;
  text: string;
}

function searchInTextLayer(
  textLayer: TextLayer,
  query: string
): SearchMatch[] {
  const matches: SearchMatch[] = [];
  const lowerQuery = query.toLowerCase();

  for (let i = 0; i < textLayer.textContentItemsStr.length; i++) {
    const text = textLayer.textContentItemsStr[i];
    if (text.toLowerCase().includes(lowerQuery)) {
      matches.push({
        itemIndex: i,
        textDiv: textLayer.textDivs[i],
        text: text,
      });
    }
  }

  return matches;
}

function highlightMatches(matches: SearchMatch[]): void {
  for (const match of matches) {
    match.textDiv.style.backgroundColor = "rgba(255, 255, 0, 0.4)";
    // Make the highlight visible even though text is transparent
    match.textDiv.style.borderRadius = "2px";
  }
}

function clearHighlights(matches: SearchMatch[]): void {
  for (const match of matches) {
    match.textDiv.style.backgroundColor = "";
    match.textDiv.style.borderRadius = "";
  }
}
```

---

## Update Text Layer on Viewport Change

Efficiently updates text layer positioning when zoom or rotation changes.

```typescript
import { TextLayer } from "pdfjs-dist";
import type { PDFPageProxy, PageViewport } from "pdfjs-dist";

class PageTextLayerManager {
  private textLayer: TextLayer | null = null;
  private textLayerDiv: HTMLDivElement;
  private page: PDFPageProxy;

  constructor(page: PDFPageProxy, textLayerDiv: HTMLDivElement) {
    this.page = page;
    this.textLayerDiv = textLayerDiv;
  }

  async initialize(viewport: PageViewport): Promise<void> {
    const textContent = await this.page.getTextContent();
    this.textLayer = new TextLayer({
      textContentSource: textContent,
      container: this.textLayerDiv,
      viewport: viewport,
    });
    await this.textLayer.render();
  }

  updateViewport(newViewport: PageViewport): void {
    if (!this.textLayer) return;

    // update() repositions existing spans -- much faster than re-rendering
    this.textLayer.update({
      viewport: newViewport,
      onBefore: () => {
        this.textLayerDiv.style.width = `${Math.floor(newViewport.width)}px`;
        this.textLayerDiv.style.height = `${Math.floor(newViewport.height)}px`;
      },
    });
  }

  destroy(): void {
    if (this.textLayer) {
      this.textLayer.cancel();
      this.textLayer = null;
    }
    // Remove all child spans
    this.textLayerDiv.innerHTML = "";
  }
}
```

---

## Extract Text with Position Data

Extracts text along with position coordinates for layout analysis.

```typescript
import type { PDFPageProxy } from "pdfjs-dist";

interface PositionedText {
  text: string;
  x: number;
  y: number;
  width: number;
  height: number;
  fontName: string;
  direction: string;
}

async function extractTextWithPositions(
  page: PDFPageProxy
): Promise<PositionedText[]> {
  const textContent = await page.getTextContent();
  const results: PositionedText[] = [];

  for (const item of textContent.items) {
    if (!("str" in item) || item.str.trim() === "") continue;

    results.push({
      text: item.str,
      x: item.transform[4],       // X position in PDF user space
      y: item.transform[5],       // Y position in PDF user space
      width: item.width,
      height: item.height,
      fontName: item.fontName,
      direction: item.dir,
    });
  }

  return results;
}
```
