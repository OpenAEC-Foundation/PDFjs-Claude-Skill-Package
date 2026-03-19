# API Reference: Core Architecture Types

> pdfjs-dist 5.x -- All signatures verified against official type definitions.

## GlobalWorkerOptions

Static configuration object. MUST be set before calling `getDocument()`.

| Property | Type | Description |
|----------|------|-------------|
| `workerSrc` | `string \| URL` | Path or URL to `pdf.worker.mjs`. ALWAYS set this first. |
| `workerPort` | `MessagePort \| null` | Custom worker port for advanced setups (e.g., SharedWorker). |

```typescript
import { GlobalWorkerOptions } from 'pdfjs-dist';
GlobalWorkerOptions.workerSrc = '/pdf.worker.mjs';
```

---

## getDocument(source)

Entry point for loading a PDF. Returns a `PDFDocumentLoadingTask`.

### DocumentInitParameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `url` | `string \| URL` | -- | URL to fetch the PDF from |
| `data` | `TypedArray \| ArrayBuffer \| string` | -- | Binary PDF data (alternative to `url`) |
| `httpHeaders` | `Record<string, string>` | -- | HTTP headers for the request |
| `withCredentials` | `boolean` | `false` | Send cookies with cross-origin requests |
| `password` | `string` | -- | Password for encrypted PDFs |
| `range` | `PDFDataRangeTransport` | -- | Custom range request transport |
| `rangeChunkSize` | `number` | `65536` | Chunk size for range requests |
| `worker` | `PDFWorker` | -- | Pre-created worker instance |
| `cMapUrl` | `string` | -- | Path to CMap files directory (for CJK support) |
| `cMapPacked` | `boolean` | `true` | Whether CMaps are binary packed |
| `standardFontDataUrl` | `string` | -- | Path to standard font data directory |
| `useWorkerFetch` | `boolean` | `true` | Let worker fetch the PDF (vs main thread) |
| `stopAtErrors` | `boolean` | `false` | Stop parsing at first error |
| `maxImageSize` | `number` | `-1` | Max pixels for images (-1 = unlimited) |

**ALWAYS** provide either `url` or `data` -- never both.

```typescript
import { getDocument } from 'pdfjs-dist';

// From URL
const loadingTask = getDocument({ url: '/my-document.pdf' });

// From binary data
const loadingTask = getDocument({ data: arrayBuffer });

// With CJK support
const loadingTask = getDocument({
  url: '/chinese-doc.pdf',
  cMapUrl: '/cmaps/',
  cMapPacked: true,
});
```

---

## PDFDocumentLoadingTask

Returned by `getDocument()`. Tracks loading progress.

| Property/Method | Type | Description |
|-----------------|------|-------------|
| `promise` | `Promise<PDFDocumentProxy>` | Resolves when document is loaded |
| `onProgress` | `(data: {loaded: number, total: number}) => void` | Progress callback |
| `onPassword` | `(callback: Function, reason: number) => void` | Password prompt callback |
| `destroy()` | `Promise<void>` | Cancel loading and release resources |

---

## PDFDocumentProxy

Represents a loaded PDF document. Returned by resolving `PDFDocumentLoadingTask.promise`.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `numPages` | `number` | Total number of pages in the document |
| `fingerprints` | `[string, string \| null]` | Document fingerprints (original + modified) |
| `isPureXfa` | `boolean` | Whether the document is XFA-only (no standard pages) |

### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `getPage(pageNumber)` | `Promise<PDFPageProxy>` | Get a page (1-based index) |
| `getPageIndex(ref)` | `Promise<number>` | Get page index from a destination reference |
| `getDestinations()` | `Promise<Record<string, any>>` | Get all named destinations |
| `getDestination(id)` | `Promise<any>` | Get a specific named destination |
| `getPageLabels()` | `Promise<string[] \| null>` | Get page labels (e.g., "i", "ii", "1", "2") |
| `getPageLayout()` | `Promise<string>` | Get page layout hint |
| `getPageMode()` | `Promise<string>` | Get page mode hint |
| `getViewerPreferences()` | `Promise<any>` | Get viewer preference dictionary |
| `getOpenAction()` | `Promise<any>` | Get document open action |
| `getAttachments()` | `Promise<Record<string, any>>` | Get embedded file attachments |
| `getJSActions()` | `Promise<Record<string, string[]>>` | Get document-level JavaScript actions |
| `getOutline()` | `Promise<any[]>` | Get document outline (bookmarks) |
| `getOptionalContentConfig()` | `Promise<any>` | Get optional content (layer) config |
| `getPermissions()` | `Promise<number[] \| null>` | Get document permissions |
| `getMetadata()` | `Promise<{info: Object, metadata: Metadata}>` | Get document metadata |
| `getMarkInfo()` | `Promise<any>` | Get mark info dictionary |
| `getData()` | `Promise<Uint8Array>` | Get raw PDF binary data |
| `saveDocument()` | `Promise<Uint8Array>` | Save modified document (annotations, forms) |
| `getDownloadInfo()` | `Promise<{length: number}>` | Get download info |
| `cleanup()` | `Promise<void>` | Release cached page data (keeps document usable) |
| `destroy()` | `Promise<void>` | Destroy document and release ALL resources |
| `getFieldObjects()` | `Promise<Record<string, any[]> \| null>` | Get form field objects |
| `hasJSActions()` | `Promise<boolean>` | Check if document has JavaScript actions |
| `getCalculationOrderIds()` | `Promise<string[] \| null>` | Get calculation order for form fields |

**ALWAYS** call `destroy()` when finished with a document. `cleanup()` only frees page cache -- `destroy()` releases worker resources.

---

## PDFPageProxy

Represents a single PDF page. Returned by `PDFDocumentProxy.getPage()`.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `pageNumber` | `number` | 1-based page number |
| `rotate` | `number` | Page rotation in degrees (0, 90, 180, 270) |
| `ref` | `RefProxy` | Internal reference object |
| `userUnit` | `number` | User unit size (default: 1.0 = 1/72 inch) |
| `view` | `number[]` | Page bounding box `[x1, y1, x2, y2]` in PDF units |

### Methods

| Method | Parameters | Returns | Description |
|--------|-----------|---------|-------------|
| `getViewport(params)` | `{scale, rotation?, offsetX?, offsetY?, dontFlip?}` | `PageViewport` | Calculate viewport for rendering |
| `getAnnotations(params?)` | `{intent?}` | `Promise<any[]>` | Get page annotations |
| `getJSActions()` | -- | `Promise<Record<string, string[]>>` | Get page-level JavaScript actions |
| `render(params)` | `{canvasContext, viewport, intent?, transform?, background?}` | `RenderTask` | Render page to canvas |
| `getOperatorList(intent?)` | `{intent?}` | `Promise<any>` | Get raw operator list (advanced) |
| `streamTextContent(params?)` | `{includeMarkedContent?}` | `ReadableStream` | Stream text content |
| `getTextContent(params?)` | `{includeMarkedContent?}` | `Promise<TextContent>` | Get all text content at once |
| `getStructTree()` | -- | `Promise<any>` | Get structure tree (tagged PDF) |
| `cleanup(resetStats?)` | `boolean` | `boolean` | Release cached data |

### PageViewport

Returned by `getViewport()`. Contains coordinate transformation data.

| Property | Type | Description |
|----------|------|-------------|
| `width` | `number` | Viewport width in CSS pixels |
| `height` | `number` | Viewport height in CSS pixels |
| `scale` | `number` | Applied scale factor |
| `rotation` | `number` | Applied rotation |
| `viewBox` | `number[]` | Original page view box |
| `transform` | `number[]` | 6-element transformation matrix |
| `offsetX` | `number` | Horizontal offset |
| `offsetY` | `number` | Vertical offset |

| Method | Returns | Description |
|--------|---------|-------------|
| `clone(params?)` | `PageViewport` | Clone with optional overrides |
| `convertToViewportPoint(x, y)` | `[number, number]` | PDF coords to viewport coords |
| `convertToViewportRectangle(rect)` | `number[]` | PDF rect to viewport rect |
| `convertToPdfPoint(x, y)` | `[number, number]` | Viewport coords to PDF coords |

---

## RenderTask

Returned by `PDFPageProxy.render()`. Represents an in-progress render operation.

| Property/Method | Type | Description |
|-----------------|------|-------------|
| `promise` | `Promise<void>` | Resolves when rendering is complete |
| `cancel()` | `void` | Cancel the render operation |

**ALWAYS** cancel an active `RenderTask` before starting a new render on the same canvas:

```typescript
let currentRenderTask: RenderTask | null = null;

async function renderPage(page: PDFPageProxy, canvas: HTMLCanvasElement) {
  if (currentRenderTask) {
    currentRenderTask.cancel();
  }
  const viewport = page.getViewport({ scale: 1.5 });
  const context = canvas.getContext('2d')!;
  currentRenderTask = page.render({ canvasContext: context, viewport });
  try {
    await currentRenderTask.promise;
  } catch (err: any) {
    if (err.name !== 'RenderingCancelledException') {
      throw err;
    }
  }
}
```

---

## TextLayer

Class for rendering selectable text over a canvas-rendered PDF page.

### Constructor

```typescript
new TextLayer({
  textContentSource: ReadableStream | Promise<TextContent>,
  container: HTMLDivElement,
  viewport: PageViewport,
  textDivs?: HTMLElement[],
  textDivProperties?: WeakMap<HTMLElement, any>,
})
```

### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `render()` | `Promise<void>` | Render text layer spans into the container |
| `update(params)` | `void` | Update viewport (e.g., after zoom): `{viewport, onBefore?}` |
| `cancel()` | `void` | Cancel pending render |

### Static Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `TextLayer.cleanup()` | `void` | Clean up shared resources |

---

## AnnotationLayer

Class for rendering interactive annotation elements (links, form fields) over a PDF page.

### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `render(params)` | `Promise<void>` | Render annotations into the layer |
| `update(params)` | `void` | Update after viewport changes |
| `cancel()` | `void` | Cancel pending render |
| `hasEditableAnnotations()` | `boolean` | Check if layer contains editable annotations |
