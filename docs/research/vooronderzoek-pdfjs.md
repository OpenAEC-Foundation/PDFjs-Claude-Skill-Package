# Vooronderzoek PDF.js — Complete API Surface Research

**Date**: 2026-03-19
**Scope**: pdfjs-dist 4.x (with notes on 5.x changes)
**Sources**: Official PDF.js documentation, GitHub repository, npm package, source code
**Status**: Phase 2 — Deep Research

---

## §1: PDF.js Architecture Overview

### 1.1 What PDF.js Is

PDF.js is Mozilla's open-source, web-standards-based platform for parsing and rendering PDF documents. It runs entirely in the browser using HTML5 Canvas and JavaScript — no plugins or native code required. Licensed under Apache 2.0.

### 1.2 Three-Layer Architecture

PDF.js is organized into three distinct layers:

| Layer | Purpose | Typical User |
|-------|---------|-------------|
| **Core** | Binary PDF parsing, stream decoding, font processing | Advanced/internal use only |
| **Display** | High-level API for rendering pages, extracting text, handling annotations | Application developers |
| **Viewer** | Complete UI with search, thumbnails, navigation, print | End-user embedding |

The **Display layer** (`pdfjs-dist`) is the primary target for this skill package. It exposes the public API that developers interact with: `getDocument()`, `PDFDocumentProxy`, `PDFPageProxy`, rendering, text extraction, and annotations.

### 1.3 Worker Thread Model

PDF.js uses a **Web Worker** architecture to keep the main thread responsive:

- **Main thread**: Holds the Display layer API (`PDFDocumentProxy`, `PDFPageProxy`). Handles rendering to canvas, DOM manipulation for text/annotation layers.
- **Worker thread**: Runs the Core layer. Performs all CPU-intensive work: PDF parsing, stream decoding, font processing, operator list generation.
- **Communication**: Main thread and worker communicate via `MessageHandler` using structured cloning. The API objects on the main thread are *proxies* — they forward calls to the worker and return Promises.

This architecture means:
- PDF parsing NEVER blocks the UI thread
- The worker MUST be loaded separately (see §7)
- Worker and library versions MUST match exactly

### 1.4 Package Structure (pdfjs-dist)

The `pdfjs-dist` npm package contains:

```
pdfjs-dist/
├── build/
│   ├── pdf.mjs              # Main library (ES module)
│   ├── pdf.worker.mjs        # Worker script (ES module)
│   ├── pdf.min.mjs           # Minified main library
│   ├── pdf.worker.min.mjs    # Minified worker
│   ├── pdf.sandbox.mjs       # Sandboxed JS execution
│   └── *.map                 # Source maps
├── cmaps/                    # CMap files for CJK font support
├── standard_fonts/           # Standard 14 PDF fonts
├── types/                    # TypeScript declarations
│   └── src/
│       └── display/
│           ├── api.d.ts
│           ├── text_layer.d.ts
│           └── annotation_layer.d.ts
├── legacy/                   # Legacy browser build
│   └── build/
│       ├── pdf.mjs
│       └── pdf.worker.mjs
└── web/                      # Viewer components (optional)
```

### 1.5 Version History and Current State

| Version | Key Changes |
|---------|------------|
| **2.x** | Initial npm distribution, function-based APIs |
| **3.x** | TypeScript definitions added, improved annotations |
| **4.x** | Class-based TextLayer/AnnotationLayer APIs, private class fields, deprecated function-based `renderTextLayer()` |
| **5.x** | OpenJPEG moved to WASM (`wasmUrl` required), ICC profile support (`iccUrl`), CSS variable dependencies for layers, signature editor, `MissingPDFException` and `UnexpectedResponseException` merged |

**Current latest**: pdfjs-dist 5.5.207 (as of March 2026)
**This package targets**: pdfjs-dist 4.x (still widely deployed; concepts apply to 5.x with noted changes)

**IMPORTANT v4→v5 breaking changes**:
- `wasmUrl` API option REQUIRED for JPEG 2000 support (OpenJPEG decoder moved to separate .wasm file)
- `iccUrl` API option added for ICC profile color conversion
- `MissingPDFException` and `UnexpectedResponseException` consolidated into a single exception
- Viewer component `render()` methods changed to accept parameter objects instead of positional arguments
- `userUnit` now applied via CSS (affects text/annotation layer positioning)
- New CSS variables required by text and annotation layers

---

## §2: Core API — Document Loading

### 2.1 getDocument()

The primary entry point for loading a PDF document.

```typescript
function getDocument(
  src: string | URL | TypedArray | ArrayBuffer | DocumentInitParameters
): PDFDocumentLoadingTask
```

**Source types** (the `src` parameter):
- **string**: URL to PDF file (same-origin or CORS-enabled)
- **URL**: URL object pointing to PDF
- **TypedArray / ArrayBuffer**: Raw binary PDF data already in memory
- **DocumentInitParameters**: Object with fine-grained options (see below)

**DocumentInitParameters** (key properties):

| Parameter | Type | Description |
|-----------|------|-------------|
| `url` | string \| URL | PDF file location |
| `data` | TypedArray \| ArrayBuffer \| Array\<number\> | Binary PDF content |
| `password` | string | Password for encrypted PDFs |
| `worker` | PDFWorker | Custom worker instance |
| `cMapUrl` | string | URL to CMap files directory (for CJK fonts) |
| `cMapPacked` | boolean | Whether CMaps are binary packed (default: true) |
| `standardFontDataUrl` | string | URL to standard fonts directory |
| `verbosity` | number | Logging level |
| `stopAtErrors` | boolean | Halt on errors instead of attempting recovery |
| `useSystemFonts` | boolean | Fall back to system fonts |
| `useWorkerFetch` | boolean | Let worker fetch data (default: true for URLs) |
| `wasmUrl` | string | **(v5+)** Path to OpenJPEG WASM file |
| `iccUrl` | string | **(v5+)** Path to ICC profile WASM file |
| `useWasm` | boolean | Enable WebAssembly optimization |
| `isEvalSupported` | boolean | Whether eval() is allowed (for font processing) |
| `disableAutoFetch` | boolean | Disable automatic data fetching beyond first page |
| `disableStream` | boolean | Disable streaming of PDF data |
| `disableRange` | boolean | Disable range requests |
| `maxImageSize` | number | Maximum image size in pixels (-1 = unlimited) |
| `rangeChunkSize` | number | Size of range request chunks |
| `length` | number | PDF file length for range request optimization |

### 2.2 PDFDocumentLoadingTask

Returned by `getDocument()`. Provides progress monitoring and completion promise.

**Properties**:

| Property | Type | Description |
|----------|------|-------------|
| `promise` | Promise\<PDFDocumentProxy\> | Resolves when document is loaded |
| `docId` | string | Unique identifier for the loading task |
| `destroyed` | boolean | Whether the task has been destroyed |
| `onProgress` | function | Callback receiving `{loaded, total}` for progress bars |
| `onPassword` | function | Callback for password-protected PDFs: `(callback, reason)` |

**Methods**:

| Method | Returns | Description |
|--------|---------|-------------|
| `destroy()` | Promise | Aborts network requests and destroys worker |
| `getData()` | Promise\<Uint8Array\> | Gets raw PDF data (even during initialization) |

**Usage pattern**:

```typescript
const loadingTask = pdfjsLib.getDocument(url);

loadingTask.onProgress = (progress) => {
  const percent = (progress.loaded / progress.total) * 100;
  console.log(`Loading: ${percent.toFixed(1)}%`);
};

loadingTask.onPassword = (callback, reason) => {
  const password = prompt('Enter PDF password:');
  callback(password);
};

const pdf = await loadingTask.promise;
```

### 2.3 PDFDocumentProxy

Proxy to the PDF document in the worker thread. Obtained from `loadingTask.promise`.

**Properties**:

| Property | Type | Description |
|----------|------|-------------|
| `numPages` | number | Total page count |
| `fingerprints` | Array\<string \| null\> | Document identifiers (2 elements; second for modified docs) |
| `isPureXfa` | boolean | True if document is XFA-only |
| `annotationStorage` | AnnotationStorage | Storage for form field / annotation data |
| `loadingTask` | PDFDocumentLoadingTask | Reference to the loading task |
| `loadingParams` | DocumentInitParameters | Subset of init params needed by viewer |

**Methods — Page Access**:

| Method | Parameters | Returns | Description |
|--------|-----------|---------|-------------|
| `getPage(pageNumber)` | pageNumber: number (1-based) | Promise\<PDFPageProxy\> | Get a specific page |
| `getPageIndex(ref)` | ref: RefProxy | Promise\<number\> | Get page number from internal reference |
| `cachedPageNumber(ref)` | ref: RefProxy | number \| null | Get cached page number (no worker round-trip) |

**Methods — Document Metadata**:

| Method | Returns | Description |
|--------|---------|-------------|
| `getMetadata()` | Promise\<{info, metadata}\> | Document info and XMP metadata |
| `getMarkInfo()` | Promise\<MarkInfo \| null\> | Accessibility mark info |
| `getPermissions()` | Promise\<Array\<number\> \| null\> | Document permissions flags |
| `getViewerPreferences()` | Promise\<Object \| null\> | Viewer display preferences |

**Methods — Navigation & Structure**:

| Method | Returns | Description |
|--------|---------|-------------|
| `getOutline()` | Promise\<Array\<OutlineNode\>\> | Document outline / bookmarks |
| `getDestination(id)` | Promise\<Array \| null\> | Named destination |
| `getDestinations()` | Promise\<Object\> | All named destinations |
| `getPageLabels()` | Promise\<Array\<string\> \| null\> | Custom page labels (e.g., "i", "ii", "1", "2") |
| `getPageLayout()` | Promise\<string\> | Page layout preference |
| `getPageMode()` | Promise\<string\> | Page mode (e.g., UseOutlines, UseNone) |
| `getOpenAction()` | Promise\<any \| null\> | Action to perform on document open |

**Methods — Annotations & Forms**:

| Method | Returns | Description |
|--------|---------|-------------|
| `getFieldObjects()` | Promise\<Object \| null\> | Form field objects |
| `hasJSActions()` | Promise\<boolean\> | Whether document has JavaScript actions |
| `getJSActions()` | Promise\<Object \| null\> | JavaScript actions |
| `getCalculationOrderIds()` | Promise\<Array\<string\> \| null\> | Calculation order for form fields |
| `getAnnotationsByType(types, pageIndexesToSkip)` | Promise\<Array\<Object\>\> | Annotations filtered by type |

**Methods — Attachments & Content**:

| Method | Returns | Description |
|--------|---------|-------------|
| `getAttachments()` | Promise\<Object \| null\> | Embedded file attachments |
| `getOptionalContentConfig(params)` | Promise\<OptionalContentConfig\> | Optional content (layers) |

**Methods — Document Data**:

| Method | Returns | Description |
|--------|---------|-------------|
| `getData()` | Promise\<Uint8Array\> | Complete raw PDF binary data |
| `getDownloadInfo()` | Promise\<{length}\> | Download info with content length |
| `saveDocument()` | Promise\<Uint8Array\> | Save with current annotation/form changes |
| `extractPages(pageInfos)` | Promise\<Uint8Array\> | Extract specific pages into new PDF |

**Methods — Lifecycle**:

| Method | Returns | Description |
|--------|---------|-------------|
| `cleanup(keepLoadedFonts?)` | Promise | Clean up resources (default: also unloads fonts) |
| `destroy()` | void | Destroy document proxy and terminate worker |

### 2.4 PDFPageProxy

Proxy to a single PDF page in the worker thread. Obtained from `pdf.getPage(pageNumber)`.

**Properties**:

| Property | Type | Description |
|----------|------|-------------|
| `pageNumber` | number | 1-based page number |
| `rotate` | number | Page rotation in degrees (clockwise) |
| `ref` | RefProxy \| null | Internal page reference |
| `userUnit` | number | User space unit size (1/72 inch default) |
| `view` | Array\<number\> | Visible page area: [x1, y1, x2, y2] in user space units |
| `stats` | StatTimer \| null | Performance statistics (when enabled) |
| `isPureXfa` | boolean | Whether page is XFA-only |

**Methods**:

| Method | Parameters | Returns | Description |
|--------|-----------|---------|-------------|
| `getViewport(params)` | `{scale, rotation?, offsetX?, offsetY?, dontFlip?}` | PageViewport | Calculate viewport dimensions |
| `render(params)` | RenderParameters | RenderTask | Start rendering to canvas |
| `getTextContent(params?)` | `{includeMarkedContent?, disableNormalization?}` | Promise\<TextContent\> | Extract text with positions |
| `streamTextContent(params?)` | Same as getTextContent | ReadableStream | Stream text content in chunks |
| `getAnnotations(params?)` | `{intent?}` | Promise\<Array\> | Get annotation data |
| `getJSActions()` | — | Promise\<Object \| null\> | Page-level JavaScript actions |
| `getOperatorList(params)` | GetOperatorListParameters | Promise\<PDFOperatorList\> | Get raw operator list (advanced) |
| `getStructTree()` | — | Promise\<StructTreeNode\> | Structure tree for accessibility |
| `getXfa()` | — | Promise\<Object \| null\> | XFA DOM tree |
| `cleanup(resetStats?)` | boolean (default: false) | boolean | Release page resources |

---

## §3: Rendering Pipeline

### 3.1 PageViewport

Created via `page.getViewport()`. Defines the coordinate space for rendering.

```typescript
const viewport = page.getViewport({
  scale: 1.5,          // Required: zoom level
  rotation: 0,         // Optional: additional rotation in degrees
  offsetX: 0,          // Optional: horizontal offset
  offsetY: 0,          // Optional: vertical offset
  dontFlip: false      // Optional: don't flip Y-axis (advanced)
});
```

**PageViewport properties**:

| Property | Type | Description |
|----------|------|-------------|
| `width` | number | Viewport width in CSS pixels |
| `height` | number | Viewport height in CSS pixels |
| `scale` | number | Scale factor |
| `rotation` | number | Rotation in degrees |
| `offsetX` | number | Horizontal offset |
| `offsetY` | number | Vertical offset |
| `transform` | Array\<number\> | 6-element transformation matrix |
| `viewBox` | Array\<number\> | Original page dimensions |

**PageViewport methods**:

| Method | Parameters | Returns | Description |
|--------|-----------|---------|-------------|
| `clone(params?)` | Same as getViewport | PageViewport | Create modified copy |
| `convertToViewportPoint(x, y)` | PDF coordinates | [x, y] | PDF→screen coordinate conversion |
| `convertToPdfPoint(x, y)` | Screen coordinates | [x, y] | Screen→PDF coordinate conversion |
| `convertToViewportRectangle(rect)` | [x1,y1,x2,y2] | [x1,y1,x2,y2] | Convert PDF rectangle to screen |

### 3.2 Canvas Rendering

The fundamental rendering flow:

```typescript
// 1. Get viewport
const scale = 1.5;
const viewport = page.getViewport({ scale });

// 2. Handle HiDPI displays
const outputScale = window.devicePixelRatio || 1;

// 3. Set canvas dimensions
const canvas = document.getElementById('pdf-canvas');
const context = canvas.getContext('2d');

// Physical pixel dimensions (actual rendering resolution)
canvas.width = Math.floor(viewport.width * outputScale);
canvas.height = Math.floor(viewport.height * outputScale);

// CSS dimensions (display size)
canvas.style.width = Math.floor(viewport.width) + 'px';
canvas.style.height = Math.floor(viewport.height) + 'px';

// 4. Create transform for HiDPI
const transform = outputScale !== 1
  ? [outputScale, 0, 0, outputScale, 0, 0]
  : null;

// 5. Render
const renderTask = page.render({
  canvasContext: context,
  viewport: viewport,
  transform: transform,
});

await renderTask.promise;
```

**RenderParameters** (passed to `page.render()`):

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `canvasContext` | CanvasRenderingContext2D | Yes | 2D canvas context |
| `viewport` | PageViewport | Yes | Target viewport |
| `transform` | Array\<number\> | No | Additional transform (for HiDPI) |
| `background` | string \| CanvasGradient \| CanvasPattern | No | Background fill |
| `annotationMode` | number | No | 0=disabled, 1=enabled, 2=forms, 3=storage |
| `optionalContentConfigPromise` | Promise | No | Optional content layers |
| `pageColors` | Object | No | Custom foreground/background colors |

### 3.3 RenderTask

Returned by `page.render()`. Controls the rendering lifecycle.

**Properties**:

| Property | Type | Description |
|----------|------|-------------|
| `promise` | Promise\<void\> | Resolves on completion, rejects on cancel/error |
| `onContinue` | function | Callback for incremental rendering pauses |
| `onError` | function | Synchronous error callback |
| `separateAnnots` | boolean | Whether form fields render separately |

**Methods**:

| Method | Parameters | Description |
|--------|-----------|-------------|
| `cancel(extraDelay?)` | number (default: 0) | Cancel rendering. Rejects the promise |

**CRITICAL pattern — Cancel before re-render**:

```typescript
let currentRenderTask: RenderTask | null = null;

async function renderPage(pageNum: number, scale: number) {
  // ALWAYS cancel previous render before starting new one
  if (currentRenderTask) {
    currentRenderTask.cancel();
    currentRenderTask = null;
  }

  const page = await pdf.getPage(pageNum);
  const viewport = page.getViewport({ scale });
  // ... set canvas dimensions ...

  currentRenderTask = page.render({ canvasContext: ctx, viewport });

  try {
    await currentRenderTask.promise;
  } catch (err) {
    if (err.name === 'RenderingCancelledException') {
      // Expected when cancel() was called — ignore
      return;
    }
    throw err; // Re-throw real errors
  }
}
```

### 3.4 HiDPI (devicePixelRatio) Handling

**WHY**: On HiDPI/Retina displays, `devicePixelRatio` is typically 2 or 3. Without handling this, PDFs appear blurry because the canvas pixel dimensions match CSS pixels, not physical pixels.

**HOW**: The canvas physical size must be multiplied by `devicePixelRatio`, while the CSS display size stays at the viewport dimensions. A transform matrix scales the rendering to fill the larger canvas.

```typescript
const outputScale = window.devicePixelRatio || 1;

// Physical pixels = CSS pixels × devicePixelRatio
canvas.width = Math.floor(viewport.width * outputScale);
canvas.height = Math.floor(viewport.height * outputScale);

// CSS display size = viewport dimensions
canvas.style.width = Math.floor(viewport.width) + 'px';
canvas.style.height = Math.floor(viewport.height) + 'px';

// Transform scales rendering to match physical resolution
const transform = outputScale !== 1
  ? [outputScale, 0, 0, outputScale, 0, 0]
  : null;
```

### 3.5 OffscreenCanvas

PDF.js supports OffscreenCanvas for rendering in workers or for better performance:

```typescript
const offscreen = new OffscreenCanvas(width, height);
const ctx = offscreen.getContext('2d');
const renderTask = page.render({ canvasContext: ctx, viewport });
await renderTask.promise;
// Transfer to main canvas or use as ImageBitmap
```

### 3.6 SVG Rendering

SVG rendering has been **deprecated and removed** in PDF.js v4+. ALWAYS use Canvas rendering. The `SVGGraphics` class is no longer part of the public API.

---

## §4: Text Layer

### 4.1 Text Content Extraction

```typescript
const textContent = await page.getTextContent({
  includeMarkedContent: false,  // Include marked content items
  disableNormalization: false,  // Disable Unicode normalization
});
```

**TextContent structure**:

```typescript
interface TextContent {
  items: Array<TextItem | TextMarkedContent>;
  styles: Record<string, TextStyle>;
}

interface TextItem {
  str: string;                  // The text string
  dir: string;                  // Text direction ('ltr', 'rtl', 'ttb')
  width: number;                // Width in user space
  height: number;               // Height in user space
  transform: number[];          // 6-element transform matrix [a, b, c, d, e, f]
  fontName: string;             // Reference to styles object key
  hasEOL: boolean;              // Whether item ends with end-of-line
}

interface TextStyle {
  fontFamily: string;           // Font family name
  ascent: number;               // Font ascent
  descent: number;              // Font descent
  vertical: boolean;            // Vertical writing mode
}
```

### 4.2 TextLayer Class (v4+ API)

**IMPORTANT**: The old `renderTextLayer()` function is **deprecated** since v4. ALWAYS use the `TextLayer` class.

```typescript
import { TextLayer } from 'pdfjs-dist';

const textLayer = new TextLayer({
  textContentSource: textContent,   // From page.getTextContent() or page.streamTextContent()
  container: textLayerDiv,          // DOM element to receive text spans
  viewport: viewport,              // PageViewport for positioning
});

await textLayer.render();
```

**Constructor parameters** (TextLayerParameters):

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `textContentSource` | ReadableStream \| TextContent | Yes | Text data from page |
| `container` | HTMLElement | Yes | DOM container for text spans |
| `viewport` | PageViewport | Yes | Viewport for layout calculations |

**Methods**:

| Method | Returns | Description |
|--------|---------|-------------|
| `render()` | Promise | Render text spans into container |
| `update(params)` | void | Update layout for new viewport (e.g., after zoom) |
| `cancel()` | void | Cancel ongoing rendering |

**Update parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `viewport` | PageViewport | New viewport |
| `onBefore` | function | Callback before DOM updates |

**Properties (getters)**:

| Property | Type | Description |
|----------|------|-------------|
| `textDivs` | HTMLElement[] | Array of span elements |
| `textContentItemsStr` | string[] | Array of text strings |

**Constants**:

| Constant | Value | Description |
|----------|-------|-------------|
| `MAX_TEXT_DIVS_TO_RENDER` | 100,000 | Safety limit for very large pages |
| `DEFAULT_FONT_SIZE` | 30 | Default font size in pixels |

### 4.3 CSS Requirements for Text Layer

The text layer MUST have proper CSS to overlay the canvas exactly. ALWAYS include these styles:

```css
.textLayer {
  position: absolute;
  left: 0;
  top: 0;
  right: 0;
  bottom: 0;
  overflow: hidden;
  opacity: 0.25;           /* Semi-transparent to allow canvas to show through */
  line-height: 1.0;
}

.textLayer span {
  color: transparent;       /* Text is invisible — only used for selection */
  position: absolute;
  white-space: pre;
  pointer-events: all;      /* Enable text selection */
}

.textLayer ::selection {
  background: rgba(0, 0, 255, 0.25);  /* Selection highlight */
}
```

**NOTE**: PDF.js ships its own CSS for the text layer. In v4+, import it from `pdfjs-dist/web/pdf_viewer.css` or use the specific text layer styles. In v5+, the text layer depends on CSS variables that must be set.

### 4.4 Extracting Plain Text

To extract plain text without rendering a layer:

```typescript
async function extractPageText(pdf: PDFDocumentProxy, pageNum: number): Promise<string> {
  const page = await pdf.getPage(pageNum);
  const textContent = await page.getTextContent();
  return textContent.items
    .filter((item): item is TextItem => 'str' in item)
    .map(item => item.str)
    .join('');
}

async function extractAllText(pdf: PDFDocumentProxy): Promise<string> {
  const texts: string[] = [];
  for (let i = 1; i <= pdf.numPages; i++) {
    texts.push(await extractPageText(pdf, i));
  }
  return texts.join('\n\n');
}
```

---

## §5: Annotation Layer

### 5.1 Getting Annotation Data

```typescript
const annotations = await page.getAnnotations({
  intent: 'display'  // 'display' | 'print' | 'any'
});
```

Each annotation object contains (AnnotationData):

| Property | Type | Description |
|----------|------|-------------|
| `id` | string | Unique annotation ID |
| `subtype` | string | Annotation type (see table below) |
| `rect` | number[] | Bounding rectangle [x1, y1, x2, y2] |
| `contents` | string | Text contents |
| `color` | Uint8ClampedArray | Color in RGB |
| `borderStyle` | Object | Border width, style, dash |
| `hasAppearance` | boolean | Whether annotation has custom appearance |

### 5.2 Annotation Types

| Subtype | Category | Description |
|---------|----------|-------------|
| `Text` | Markup | Sticky note icon |
| `Link` | Navigation | Hyperlink (internal or external) |
| `FreeText` | Markup | Text box annotation |
| `Line` | Drawing | Straight line with endpoints |
| `Square` | Drawing | Rectangle shape |
| `Circle` | Drawing | Ellipse shape |
| `Polygon` | Drawing | Multi-point closed shape |
| `PolyLine` | Drawing | Multi-point open shape |
| `Highlight` | Text Markup | Text highlight overlay |
| `Underline` | Text Markup | Text underline |
| `StrikeOut` | Text Markup | Text strikethrough |
| `Squiggly` | Text Markup | Squiggly underline |
| `Stamp` | Visual | Predefined stamp image |
| `Caret` | Text Markup | Text insertion mark |
| `Ink` | Drawing | Freehand drawing |
| `Popup` | Interactive | Pop-up comment window |
| `FileAttachment` | Embedded | Embedded file icon |
| `Widget` | Form | Form field (sub-types below) |

**Widget sub-types** (form fields):

| Field Type | Description |
|-----------|-------------|
| `Tx` (Text) | Text input field |
| `Btn` (Button) | Checkbox, radio button, push button |
| `Ch` (Choice) | Dropdown, list box |
| `Sig` (Signature) | Digital signature field |

### 5.3 AnnotationLayer Class

```typescript
import { AnnotationLayer } from 'pdfjs-dist';

const annotationLayer = new AnnotationLayer({
  div: annotationLayerDiv,           // Container element
  annotationStorage: pdf.annotationStorage,  // For form data persistence
  page: page,                        // PDFPageProxy
  viewport: viewport,                // PageViewport
  linkService: linkService,          // PDFLinkService instance (for link navigation)
  renderForms: true,                 // Enable form field rendering
  imageResourcesPath: '',            // Path to annotation images
  downloadManager: null,             // For file attachment downloads
});
```

**Constructor parameters**:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `div` | HTMLElement | Yes | Container element |
| `annotationStorage` | AnnotationStorage | No | Form data storage |
| `page` | PDFPageProxy | Yes | Page proxy |
| `viewport` | PageViewport | Yes | Display viewport |
| `linkService` | PDFLinkService | No | Link navigation handler |
| `renderForms` | boolean | No | Enable interactive form fields |
| `imageResourcesPath` | string | No | Path to annotation icon images |
| `downloadManager` | DownloadManager | No | File download handler |
| `eval` | Function | No | Custom eval for JavaScript actions |
| `printing` | boolean | No | Render in print mode |

**Methods**:

| Method | Returns | Description |
|--------|---------|-------------|
| `render(params)` | Promise | Render annotations into container. In v5+, takes parameter object |
| `update(params)` | void | Update layer for new viewport |
| `hasEditableAnnotations()` | boolean | Whether layer has editable content |

### 5.4 AnnotationStorage

Manages persistent storage of annotation modifications (form field values, editor annotations).

```typescript
// Access via document proxy
const storage = pdf.annotationStorage;

// Set a form field value
storage.setValue(annotationId, { value: 'new text value' });

// Get all stored values
const allValues = storage.getAll();

// Save document with changes
const modifiedPdf = await pdf.saveDocument();
```

### 5.5 AnnotationEditorLayer

For creating new annotations (available since v3.8, expanded in v4+). Supports four annotation types:

| Editor Type | Description |
|-------------|-------------|
| FreeText | Add text annotations |
| Ink | Freehand drawing |
| Stamp | Add image stamps |
| Highlight | Highlight text selections |
| Signature | **(v5+)** Handwritten signatures |

**IMPORTANT limitation**: Annotations created by the editor are rendered as HTML overlays. They may not persist correctly in other PDF readers. The editor layer is primarily designed for use within the PDF.js viewer UI and does not have a fully documented standalone API for pdfjs-dist consumers.

---

## §6: Custom Viewer Implementation

### 6.1 Minimal Custom Viewer Architecture

A custom PDF viewer built with pdfjs-dist requires:

1. **Canvas layer**: Renders the visual content
2. **Text layer**: Enables text selection and search (positioned above canvas)
3. **Annotation layer**: Handles links, forms, interactive elements (positioned above text layer)

```html
<div id="viewer-container" style="position: relative;">
  <canvas id="pdf-canvas"></canvas>
  <div id="text-layer" class="textLayer"></div>
  <div id="annotation-layer" class="annotationLayer"></div>
</div>
```

All three layers must be positioned absolutely within the same container, sized identically to the viewport.

### 6.2 Page Navigation

The official Previous/Next example demonstrates the key pattern:

```typescript
let pdfDoc: PDFDocumentProxy;
let pageNum = 1;
let pageRendering = false;
let pageNumPending: number | null = null;

async function renderPage(num: number) {
  pageRendering = true;
  const page = await pdfDoc.getPage(num);
  const viewport = page.getViewport({ scale });
  // ... canvas setup and render ...
  const renderTask = page.render({ canvasContext: ctx, viewport });
  await renderTask.promise;
  pageRendering = false;

  // If a page was queued during rendering, render it now
  if (pageNumPending !== null) {
    renderPage(pageNumPending);
    pageNumPending = null;
  }
}

function queueRenderPage(num: number) {
  if (pageRendering) {
    pageNumPending = num;  // Queue — don't render concurrently
  } else {
    renderPage(num);
  }
}

function onPrevPage() {
  if (pageNum <= 1) return;
  pageNum--;
  queueRenderPage(pageNum);
}

function onNextPage() {
  if (pageNum >= pdfDoc.numPages) return;
  pageNum++;
  queueRenderPage(pageNum);
}
```

### 6.3 Zoom Controls

```typescript
let currentScale = 1.0;

function zoomIn() {
  currentScale *= 4 / 3;  // ~33% increase
  renderPage(pageNum);
}

function zoomOut() {
  currentScale *= 3 / 4;  // ~25% decrease
  renderPage(pageNum);
}

function fitToWidth(containerWidth: number) {
  const page = await pdfDoc.getPage(pageNum);
  const unscaledViewport = page.getViewport({ scale: 1.0 });
  currentScale = containerWidth / unscaledViewport.width;
  renderPage(pageNum);
}

function fitToPage(containerWidth: number, containerHeight: number) {
  const page = await pdfDoc.getPage(pageNum);
  const unscaledViewport = page.getViewport({ scale: 1.0 });
  const scaleX = containerWidth / unscaledViewport.width;
  const scaleY = containerHeight / unscaledViewport.height;
  currentScale = Math.min(scaleX, scaleY);
  renderPage(pageNum);
}
```

### 6.4 Scroll-Based Page Loading (Virtual Scrolling)

**NEVER render all pages at once** — this causes memory exhaustion. A letter-size page at 96 DPI requires 816×1056 pixels (~3.5MB). At HiDPI (2×), that is ~14MB per page. 100 pages = 1.4GB of canvas memory.

**Pattern**: Render only visible pages, destroy off-screen pages.

```typescript
function getVisiblePages(container: HTMLElement, pageHeights: number[]): number[] {
  const scrollTop = container.scrollTop;
  const viewportHeight = container.clientHeight;
  const scrollBottom = scrollTop + viewportHeight;

  let cumulativeHeight = 0;
  const visiblePages: number[] = [];

  for (let i = 0; i < pageHeights.length; i++) {
    const pageTop = cumulativeHeight;
    const pageBottom = cumulativeHeight + pageHeights[i];

    if (pageBottom > scrollTop && pageTop < scrollBottom) {
      visiblePages.push(i + 1); // 1-based page numbers
    }

    cumulativeHeight = pageBottom;
    if (pageTop > scrollBottom) break; // No more visible pages
  }

  return visiblePages;
}
```

### 6.5 Thumbnail Generation

```typescript
async function renderThumbnail(
  pdf: PDFDocumentProxy,
  pageNum: number,
  thumbCanvas: HTMLCanvasElement,
  maxWidth: number
) {
  const page = await pdf.getPage(pageNum);
  const unscaledViewport = page.getViewport({ scale: 1.0 });
  const thumbScale = maxWidth / unscaledViewport.width;
  const viewport = page.getViewport({ scale: thumbScale });

  thumbCanvas.width = viewport.width;
  thumbCanvas.height = viewport.height;
  const ctx = thumbCanvas.getContext('2d');

  await page.render({ canvasContext: ctx, viewport }).promise;
}
```

### 6.6 Text Search

```typescript
async function searchInDocument(
  pdf: PDFDocumentProxy,
  query: string
): Promise<Array<{ page: number; matches: string[] }>> {
  const results = [];

  for (let i = 1; i <= pdf.numPages; i++) {
    const page = await pdf.getPage(i);
    const textContent = await page.getTextContent();
    const pageText = textContent.items
      .filter((item): item is TextItem => 'str' in item)
      .map(item => item.str)
      .join(' ');

    if (pageText.toLowerCase().includes(query.toLowerCase())) {
      // Find match positions for highlighting
      const regex = new RegExp(query, 'gi');
      const matches = pageText.match(regex) || [];
      results.push({ page: i, matches });
    }
  }

  return results;
}
```

### 6.7 Print Functionality

The standard pattern for printing uses a hidden iframe with high-resolution page renders:

```typescript
async function printPdf(pdf: PDFDocumentProxy) {
  const printContainer = document.createElement('div');
  printContainer.style.display = 'none';
  document.body.appendChild(printContainer);

  for (let i = 1; i <= pdf.numPages; i++) {
    const page = await pdf.getPage(i);
    const viewport = page.getViewport({ scale: 150 / 72 }); // 150 DPI

    const canvas = document.createElement('canvas');
    canvas.width = viewport.width;
    canvas.height = viewport.height;
    const ctx = canvas.getContext('2d');

    await page.render({ canvasContext: ctx, viewport }).promise;

    const img = document.createElement('img');
    img.src = canvas.toDataURL();
    img.style.pageBreakAfter = 'always';
    printContainer.appendChild(img);
  }

  window.print();
  document.body.removeChild(printContainer);
}
```

### 6.8 HTTP Range Requests

PDF.js automatically uses HTTP Range Requests when the server supports them. PDF structure places vital data (cross-reference table) at the end of the file, so PDF.js:

1. Fetches the end of the file first (to get the xref table)
2. Fetches only the data needed for the current page
3. Requests additional data on demand as pages are viewed

This means a 100MB PDF does NOT need to be fully downloaded before the first page renders. Server must support `Accept-Ranges: bytes` header.

---

## §7: Worker Configuration

### 7.1 Setting the Worker Source

**CRITICAL**: The worker MUST be configured before calling `getDocument()`.

```typescript
import * as pdfjsLib from 'pdfjs-dist';

// Option 1: Relative path
pdfjsLib.GlobalWorkerOptions.workerSrc = './pdf.worker.mjs';

// Option 2: CDN
pdfjsLib.GlobalWorkerOptions.workerSrc =
  'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/4.6.82/pdf.worker.min.mjs';

// Option 3: npm package path
pdfjsLib.GlobalWorkerOptions.workerSrc =
  'node_modules/pdfjs-dist/build/pdf.worker.mjs';
```

### 7.2 CDN URLs

Three CDNs host pdfjs-dist:

| CDN | URL Pattern |
|-----|-------------|
| **cdnjs** | `https://cdnjs.cloudflare.com/ajax/libs/pdf.js/{VERSION}/pdf.worker.min.mjs` |
| **unpkg** | `https://unpkg.com/pdfjs-dist@{VERSION}/build/pdf.worker.min.mjs` |
| **jsDelivr** | `https://cdn.jsdelivr.net/npm/pdfjs-dist@{VERSION}/build/pdf.worker.min.mjs` |

**CRITICAL**: The CDN version MUST match the installed pdfjs-dist version exactly. A mismatch causes the error: *"The API version a.b.c does not match Worker version x.y.z"*.

### 7.3 Webpack Configuration

**Method 1**: Use the autoconfiguration module:

```javascript
// This automatically configures the worker
import 'pdfjs-dist/webpack';
import * as pdfjsLib from 'pdfjs-dist';
```

**Method 2**: Manual worker entry point:

```javascript
import * as pdfjsLib from 'pdfjs-dist';

// Webpack 5: use import.meta.url for worker resolution
pdfjsLib.GlobalWorkerOptions.workerSrc = new URL(
  'pdfjs-dist/build/pdf.worker.mjs',
  import.meta.url
).toString();
```

**Method 3**: Dedicated worker entry:

```javascript
// webpack.config.js
module.exports = {
  entry: {
    main: './src/index.js',
    'pdf.worker': 'pdfjs-dist/build/pdf.worker.entry.js',
  },
};
```

### 7.4 Vite Configuration

Vite has known issues with pdfjs-dist worker resolution. Recommended approaches:

**Method 1**: URL import (recommended):

```typescript
import * as pdfjsLib from 'pdfjs-dist';
import pdfjsWorker from 'pdfjs-dist/build/pdf.worker.min.mjs?url';

pdfjsLib.GlobalWorkerOptions.workerSrc = pdfjsWorker;
```

**Method 2**: import.meta.url resolution:

```typescript
import * as pdfjsLib from 'pdfjs-dist';

pdfjsLib.GlobalWorkerOptions.workerSrc = new URL(
  'pdfjs-dist/build/pdf.worker.min.mjs',
  import.meta.url
).toString();
```

**Method 3**: Legacy build (if module resolution fails):

```typescript
import * as pdfjsLib from 'pdfjs-dist/legacy/build/pdf.mjs';

pdfjsLib.GlobalWorkerOptions.workerSrc = new URL(
  'pdfjs-dist/legacy/build/pdf.worker.min.mjs',
  import.meta.url
).toString();
```

### 7.5 Fake Worker Mode

When a Web Worker cannot be used (e.g., some Node.js environments, testing):

```typescript
// Option 1: Use workerPort with an inline worker
const worker = new Worker(
  new URL('pdfjs-dist/build/pdf.worker.mjs', import.meta.url),
  { type: 'module' }
);
pdfjsLib.GlobalWorkerOptions.workerPort = worker;

// Option 2: Disable worker entirely (runs on main thread — blocks UI)
// This happens automatically when workerSrc is not set and pdfjs falls back
// to "fake worker" mode. NEVER do this in production.
```

When running without a worker, a console warning appears: *"Setting up fake worker."* This is acceptable for testing but NEVER for production — it blocks the main thread during PDF parsing.

### 7.6 Version Matching

The library (`pdf.mjs`) and worker (`pdf.worker.mjs`) MUST be the same version. To verify:

```typescript
// Both files contain a pdfjsVersion constant
// When they don't match, you get:
// Error: "The API version X.Y.Z does not match the Worker version A.B.C"
```

Common causes of version mismatch:
- Browser cache retaining old worker file after update
- Mixing local library with CDN worker (or vice versa)
- Multiple copies of pdfjs-dist in node_modules (dependency conflicts)

---

## §8: TypeScript Support

### 8.1 Type Imports

```typescript
import {
  getDocument,
  GlobalWorkerOptions,
  version,
} from 'pdfjs-dist';

import type {
  PDFDocumentProxy,
  PDFDocumentLoadingTask,
  PDFPageProxy,
  PageViewport,
  RenderTask,
  TextContent,
  TextItem,
  TextMarkedContent,
  TextStyle,
  DocumentInitParameters,
  OnProgressParameters,
  RenderParameters,
  TypedArray,
} from 'pdfjs-dist';

import { TextLayer } from 'pdfjs-dist';
import { AnnotationLayer } from 'pdfjs-dist';
```

Alternative import path for types:

```typescript
import type { PDFDocumentProxy } from 'pdfjs-dist/types/src/display/api';
```

### 8.2 Key Type Definitions

```typescript
// OnProgressParameters — used with loadingTask.onProgress
interface OnProgressParameters {
  loaded: number;
  total: number;
}

// TextContent — returned by page.getTextContent()
interface TextContent {
  items: Array<TextItem | TextMarkedContent>;
  styles: Record<string, TextStyle>;
}

// TextItem — individual text item in TextContent
interface TextItem {
  str: string;
  dir: string;
  width: number;
  height: number;
  transform: number[];
  fontName: string;
  hasEOL: boolean;
}

// GetViewportParameters — passed to page.getViewport()
interface GetViewportParameters {
  scale: number;
  rotation?: number;
  offsetX?: number;
  offsetY?: number;
  dontFlip?: boolean;
}

// RenderParameters — passed to page.render()
interface RenderParameters {
  canvasContext: CanvasRenderingContext2D;
  viewport: PageViewport;
  transform?: number[];
  background?: string | CanvasGradient | CanvasPattern;
  annotationMode?: number;
  optionalContentConfigPromise?: Promise<any>;
  pageColors?: { background?: string; foreground?: string };
}
```

### 8.3 TypeScript Project Setup

```typescript
// tsconfig.json — key settings
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",  // or "node16"
    "esModuleInterop": true,
    "strict": true
  }
}
```

---

## §9: Common Patterns and Best Practices

### 9.1 Proper Initialization Sequence

ALWAYS follow this order:

```typescript
// 1. Import library
import * as pdfjsLib from 'pdfjs-dist';

// 2. Set worker source BEFORE loading any document
pdfjsLib.GlobalWorkerOptions.workerSrc = '/path/to/pdf.worker.mjs';

// 3. Optional: configure CMap and font paths
const loadingTask = pdfjsLib.getDocument({
  url: pdfUrl,
  cMapUrl: '/cmaps/',
  cMapPacked: true,
  standardFontDataUrl: '/standard_fonts/',
});

// 4. Await document
const pdf = await loadingTask.promise;

// 5. Render pages
const page = await pdf.getPage(1);
```

### 9.2 Memory Management

```typescript
// ALWAYS destroy when done with a document
async function cleanup(pdf: PDFDocumentProxy, loadingTask: PDFDocumentLoadingTask) {
  // 1. Cancel any active render tasks
  if (currentRenderTask) {
    currentRenderTask.cancel();
  }

  // 2. Clean up page resources
  // page.cleanup() is called automatically by destroy()

  // 3. Destroy the document (terminates worker)
  await pdf.destroy();

  // Alternative: destroy via loading task
  // await loadingTask.destroy();
}
```

### 9.3 Error Handling Patterns

```typescript
import * as pdfjsLib from 'pdfjs-dist';

try {
  const loadingTask = pdfjsLib.getDocument(url);
  const pdf = await loadingTask.promise;
} catch (error) {
  if (error.name === 'PasswordException') {
    // PDF is password protected
    // Use loadingTask.onPassword callback instead
  } else if (error.name === 'InvalidPDFException') {
    // File is not a valid PDF
  } else if (error.name === 'MissingPDFException') {
    // File not found (v4; merged into single exception in v5)
  } else if (error.name === 'UnexpectedResponseException') {
    // Server returned unexpected response (v4; merged in v5)
  } else if (error.name === 'UnknownErrorException') {
    // Unknown error from worker
  }
}

// Render error handling
try {
  await renderTask.promise;
} catch (error) {
  if (error.name === 'RenderingCancelledException') {
    // Render was cancelled — expected, not an error
    return;
  }
  console.error('Render failed:', error);
}
```

### 9.4 Performance Optimization

1. **Lazy rendering**: Only render visible pages (see §6.4)
2. **Cancel before re-render**: ALWAYS cancel active RenderTask before starting a new one (see §3.3)
3. **Reuse page objects**: Cache `PDFPageProxy` objects; don't call `getPage()` repeatedly for the same page
4. **Use streaming text content**: `page.streamTextContent()` instead of `page.getTextContent()` for large pages
5. **Optimize canvas size**: Don't render at higher resolution than needed
6. **HTTP Range Requests**: Ensure server supports them for large PDFs
7. **Web-optimized PDFs**: Use linearized PDFs for faster first-page rendering

### 9.5 CMap and Standard Font Configuration

CMap files are needed for CJK (Chinese, Japanese, Korean) font support. Standard fonts are the 14 base PDF fonts.

```typescript
const loadingTask = pdfjsLib.getDocument({
  url: pdfUrl,
  cMapUrl: 'https://cdn.jsdelivr.net/npm/pdfjs-dist@4.6.82/cmaps/',
  cMapPacked: true,
  standardFontDataUrl: 'https://cdn.jsdelivr.net/npm/pdfjs-dist@4.6.82/standard_fonts/',
});
```

Without CMap configuration, CJK characters may render as blank boxes or missing glyphs. ALWAYS configure `cMapUrl` if your application may encounter non-Latin PDFs.

### 9.6 Password-Protected PDFs

```typescript
const loadingTask = pdfjsLib.getDocument(url);

loadingTask.onPassword = (callback, reason) => {
  // reason === 1: first request for password
  // reason === 2: incorrect password, try again
  if (reason === 2) {
    alert('Incorrect password. Try again.');
  }
  const password = prompt('Enter PDF password:');
  if (password) {
    callback(password);
  } else {
    // User cancelled — destroy the loading task
    loadingTask.destroy();
  }
};

try {
  const pdf = await loadingTask.promise;
  // PDF loaded successfully with correct password
} catch (error) {
  // Handle final failure
}
```

---

## §10: Anti-Patterns and Common Mistakes

### 10.1 NOT Setting Worker Before getDocument()

**WRONG**:
```typescript
const loadingTask = pdfjsLib.getDocument(url);
pdfjsLib.GlobalWorkerOptions.workerSrc = '/pdf.worker.mjs'; // TOO LATE
```

**RIGHT**:
```typescript
pdfjsLib.GlobalWorkerOptions.workerSrc = '/pdf.worker.mjs'; // BEFORE getDocument
const loadingTask = pdfjsLib.getDocument(url);
```

Setting the worker source after `getDocument()` may cause a "fake worker" fallback, running PDF parsing on the main thread and blocking the UI.

### 10.2 NOT Cancelling Render Tasks Before Re-Rendering

**WRONG**:
```typescript
function onZoom() {
  page.render({ canvasContext: ctx, viewport: newViewport }); // Previous render still running!
}
```

**RIGHT**:
```typescript
let currentTask: RenderTask | null = null;

function onZoom() {
  if (currentTask) currentTask.cancel();
  currentTask = page.render({ canvasContext: ctx, viewport: newViewport });
}
```

Not cancelling causes rendering conflicts, visual glitches, and wasted CPU/memory. Multiple concurrent renders to the same canvas produce corrupted output.

### 10.3 Rendering All Pages at Once

**WRONG**:
```typescript
// Memory explosion — NEVER do this
for (let i = 1; i <= pdf.numPages; i++) {
  const canvas = document.createElement('canvas');
  // ... renders all 500 pages into memory
}
```

A letter-size page at 96 DPI with devicePixelRatio=2 uses ~14MB of canvas memory. 100 pages = 1.4GB. Most browsers will crash or refuse allocation.

**RIGHT**: Render only visible pages. Destroy off-screen page canvases. Use placeholder divs with correct dimensions for scroll positioning.

### 10.4 Worker Version Mismatch

**WRONG**:
```typescript
// Library is v4.6.82 (from npm), worker is v4.5.123 (from CDN cache)
pdfjsLib.GlobalWorkerOptions.workerSrc =
  'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/4.5.123/pdf.worker.min.mjs';
```

**RIGHT**: ALWAYS use the same version for library and worker. Use the `pdfjsLib.version` property to construct the CDN URL dynamically:

```typescript
pdfjsLib.GlobalWorkerOptions.workerSrc =
  `https://cdnjs.cloudflare.com/ajax/libs/pdf.js/${pdfjsLib.version}/pdf.worker.min.mjs`;
```

### 10.5 Missing devicePixelRatio Handling

**WRONG**:
```typescript
canvas.width = viewport.width;
canvas.height = viewport.height;
// Renders at CSS pixel resolution — blurry on Retina/HiDPI displays
```

**RIGHT**: See §3.4 for the correct HiDPI pattern. ALWAYS multiply canvas dimensions by `window.devicePixelRatio` and use the transform matrix.

### 10.6 Not Cleaning Up PDFDocumentProxy

**WRONG**:
```typescript
function loadNewPdf(url: string) {
  const pdf = await pdfjsLib.getDocument(url).promise;
  // Old PDF document is never destroyed — worker thread leaks
}
```

**RIGHT**: ALWAYS call `pdf.destroy()` before loading a new document. Each `getDocument()` call creates a new worker connection. Without cleanup, workers accumulate and consume memory.

### 10.7 Using Deprecated APIs from v2/v3

| Deprecated (v2/v3) | Current (v4+) |
|---------------------|--------------|
| `renderTextLayer(params)` | `new TextLayer(params); textLayer.render()` |
| `PDFJS.workerSrc = ...` | `pdfjsLib.GlobalWorkerOptions.workerSrc = ...` |
| `PDFJS.getDocument(...)` | `pdfjsLib.getDocument(...)` |
| `page.getAnnotations()` without intent | `page.getAnnotations({ intent: 'display' })` |

### 10.8 Incorrect Text Layer Positioning

**WRONG**: Text layer div is not position:absolute or has different dimensions than the canvas. Text selection is misaligned with the rendered content.

**RIGHT**: The text layer MUST:
- Be `position: absolute` within the same container as the canvas
- Have the same width/height as the canvas CSS dimensions
- Use the same viewport for rendering
- Include the correct CSS for transparent text and selection highlighting

### 10.9 Missing CMap Data for CJK Fonts

**WRONG**: Not configuring `cMapUrl` when loading documents that may contain Chinese, Japanese, or Korean text. Characters render as blank rectangles.

**RIGHT**: ALWAYS set `cMapUrl` and `cMapPacked: true` in production applications. Point to the `cmaps/` directory from the pdfjs-dist package or a CDN.

### 10.10 Ignoring CORS for Cross-Origin PDFs

Loading a PDF from a different origin without CORS headers causes a network error. The same-origin policy applies. Solutions:
- Configure CORS on the PDF server (`Access-Control-Allow-Origin`)
- Use a server-side proxy
- Download the PDF server-side and serve from same origin

---

## §11: API Surface Summary

### 11.1 Core Classes

| Class | Location | Purpose |
|-------|----------|---------|
| `PDFDocumentLoadingTask` | pdfjsLib | Document loading lifecycle |
| `PDFDocumentProxy` | pdfjsLib | Loaded document interface |
| `PDFPageProxy` | pdfjsLib | Single page interface |
| `PageViewport` | pdfjsLib | Coordinate system and transforms |
| `RenderTask` | pdfjsLib | Canvas rendering lifecycle |
| `TextLayer` | pdfjs-dist | Text overlay for selection/search |
| `AnnotationLayer` | pdfjs-dist | Annotation and form rendering |
| `AnnotationEditorLayer` | pdfjs-dist/web | Annotation creation/editing |
| `AnnotationStorage` | pdfjsLib | Form data persistence |
| `PDFWorker` | pdfjsLib | Worker thread management |
| `PDFDataRangeTransport` | pdfjsLib | Custom data transport |

### 11.2 Key Functions

| Function | Parameters | Returns | Purpose |
|----------|-----------|---------|---------|
| `getDocument(src)` | string \| URL \| TypedArray \| DocumentInitParameters | PDFDocumentLoadingTask | Load a PDF document |

### 11.3 Configuration Objects

| Object | Property | Type | Purpose |
|--------|----------|------|---------|
| `GlobalWorkerOptions` | `workerSrc` | string | Path to worker script |
| `GlobalWorkerOptions` | `workerPort` | Worker | Custom worker instance |

### 11.4 Constants

| Constant | Type | Purpose |
|----------|------|---------|
| `version` | string | Library version string |
| `build` | string | Build identifier |

### 11.5 Method Quick Reference — PDFDocumentProxy

| Method | Returns | Category |
|--------|---------|----------|
| `getPage(n)` | Promise\<PDFPageProxy\> | Access |
| `getPageIndex(ref)` | Promise\<number\> | Access |
| `getMetadata()` | Promise\<{info, metadata}\> | Metadata |
| `getOutline()` | Promise\<OutlineNode[]\> | Navigation |
| `getDestination(id)` | Promise\<Array\> | Navigation |
| `getPageLabels()` | Promise\<string[]\> | Navigation |
| `getAttachments()` | Promise\<Object\> | Content |
| `getData()` | Promise\<Uint8Array\> | Data |
| `saveDocument()` | Promise\<Uint8Array\> | Data |
| `extractPages(pages)` | Promise\<Uint8Array\> | Data |
| `getFieldObjects()` | Promise\<Object\> | Forms |
| `hasJSActions()` | Promise\<boolean\> | Actions |
| `destroy()` | void | Lifecycle |
| `cleanup()` | Promise | Lifecycle |

### 11.6 Method Quick Reference — PDFPageProxy

| Method | Returns | Category |
|--------|---------|----------|
| `getViewport(params)` | PageViewport | Rendering |
| `render(params)` | RenderTask | Rendering |
| `getTextContent(params?)` | Promise\<TextContent\> | Text |
| `streamTextContent(params?)` | ReadableStream | Text |
| `getAnnotations(params?)` | Promise\<Array\> | Annotations |
| `getOperatorList(params)` | Promise\<PDFOperatorList\> | Advanced |
| `getStructTree()` | Promise\<StructTreeNode\> | Accessibility |
| `getJSActions()` | Promise\<Object\> | Actions |
| `cleanup()` | boolean | Lifecycle |

### 11.7 Event/Callback Patterns

| Object | Callback | Signature | Purpose |
|--------|----------|-----------|---------|
| PDFDocumentLoadingTask | `onProgress` | `({loaded, total}) => void` | Loading progress |
| PDFDocumentLoadingTask | `onPassword` | `(callback, reason) => void` | Password prompt |
| RenderTask | `onContinue` | `(resume) => void` | Incremental rendering |
| RenderTask | `onError` | `(error) => void` | Render error |

---

## §12: Skill Package Mapping

Based on this research, the following skill areas are identified for the PDF.js skill package:

### Syntax Skills
| Skill | Section Reference | Coverage |
|-------|------------------|----------|
| pdfjs-syntax-document-loading | §2 | getDocument, DocumentInitParameters, PDFDocumentLoadingTask |
| pdfjs-syntax-page-rendering | §3 | getViewport, render, RenderTask, HiDPI |
| pdfjs-syntax-text-layer | §4 | TextLayer class, TextContent, getTextContent |
| pdfjs-syntax-annotation-layer | §5 | AnnotationLayer, annotation types, AnnotationStorage |
| pdfjs-syntax-document-proxy | §2.3 | PDFDocumentProxy methods and properties |
| pdfjs-syntax-page-proxy | §2.4 | PDFPageProxy methods and properties |
| pdfjs-syntax-viewport | §3.1 | PageViewport, coordinate conversion |

### Implementation Skills
| Skill | Section Reference | Coverage |
|-------|------------------|----------|
| pdfjs-impl-custom-viewer | §6 | Building a viewer from scratch |
| pdfjs-impl-page-navigation | §6.2 | Previous/next, go-to-page, queued rendering |
| pdfjs-impl-zoom-controls | §6.3 | Zoom in/out, fit-to-width, fit-to-page |
| pdfjs-impl-virtual-scrolling | §6.4 | Scroll-based page loading, memory management |
| pdfjs-impl-text-search | §6.6 | Cross-page search with highlighting |
| pdfjs-impl-thumbnails | §6.5 | Thumbnail generation |
| pdfjs-impl-print | §6.7 | Print functionality |
| pdfjs-impl-text-extraction | §4.4 | Plain text extraction from pages |
| pdfjs-impl-form-filling | §5.4 | AnnotationStorage, form interaction, saveDocument |

### Error Skills
| Skill | Section Reference | Coverage |
|-------|------------------|----------|
| pdfjs-errors-worker | §7, §10.1, §10.4 | Worker setup errors, version mismatch |
| pdfjs-errors-rendering | §10.2, §10.3, §10.5 | Render conflicts, memory, HiDPI |
| pdfjs-errors-loading | §9.3, §10.6, §10.10 | Loading errors, CORS, cleanup |

### Core Skills
| Skill | Section Reference | Coverage |
|-------|------------------|----------|
| pdfjs-core-architecture | §1 | Three-layer model, worker thread, package structure |
| pdfjs-core-worker-config | §7 | Worker setup for all bundlers |
| pdfjs-core-typescript | §8 | Type imports, project setup |
| pdfjs-core-memory-management | §9.2, §10.3, §10.6 | Cleanup, destroy, lazy rendering |
| pdfjs-core-best-practices | §9 | Initialization, performance, CMap, passwords |

### Agent Skills
| Skill | Section Reference | Coverage |
|-------|------------------|----------|
| pdfjs-validation-agent | All sections | Validate PDF.js code against best practices |
| pdfjs-viewer-generator | §6 | Generate custom viewer code |

---

## Sources Consulted

| Source | URL | Date Verified |
|--------|-----|---------------|
| PDF.js Website | https://mozilla.github.io/pdf.js/ | 2026-03-19 |
| PDF.js API (Draft) | https://mozilla.github.io/pdf.js/api/draft/ | 2026-03-19 |
| PDF.js Getting Started | https://mozilla.github.io/pdf.js/getting_started/ | 2026-03-19 |
| PDF.js Wiki | https://github.com/mozilla/pdf.js/wiki | 2026-03-19 |
| PDF.js Examples | https://mozilla.github.io/pdf.js/examples/ | 2026-03-19 |
| PDF.js GitHub (api.js) | https://github.com/mozilla/pdf.js/blob/master/src/display/api.js | 2026-03-19 |
| PDF.js GitHub (text_layer.js) | https://github.com/mozilla/pdf.js/blob/master/src/display/text_layer.js | 2026-03-19 |
| PDF.js GitHub (annotation_layer.js) | https://github.com/mozilla/pdf.js/blob/master/src/display/annotation_layer.js | 2026-03-19 |
| PDF.js v5.0.375 Release | https://github.com/mozilla/pdf.js/releases/tag/v5.0.375 | 2026-03-19 |
| PDF.js Wiki Setup Guide | https://github.com/mozilla/pdf.js/wiki/Setup-PDF.js-in-a-website | 2026-03-19 |
| PDF.js Wiki FAQ | https://github.com/mozilla/pdf.js/wiki/Frequently-Asked-Questions | 2026-03-19 |
| PDF.js Wiki Viewer Options | https://github.com/mozilla/pdf.js/wiki/Viewer-options | 2026-03-19 |
| Nutrient Blog (Complete Guide) | https://www.nutrient.io/blog/complete-guide-to-pdfjs/ | 2026-03-19 |
| PDFDocumentProxy API Docs | https://mozilla.github.io/pdf.js/api/draft/module-pdfjsLib-PDFDocumentProxy.html | 2026-03-19 |
| PDFPageProxy API Docs | https://mozilla.github.io/pdf.js/api/draft/module-pdfjsLib-PDFPageProxy.html | 2026-03-19 |
| RenderTask API Docs | https://mozilla.github.io/pdf.js/api/draft/module-pdfjsLib-RenderTask.html | 2026-03-19 |
| PDFDocumentLoadingTask API Docs | https://mozilla.github.io/pdf.js/api/draft/module-pdfjsLib-PDFDocumentLoadingTask.html | 2026-03-19 |
| pdfjsLib Module Exports | https://mozilla.github.io/pdf.js/api/draft/module-pdfjsLib.html | 2026-03-19 |
| GitHub Issue #18206 | https://github.com/mozilla/pdf.js/issues/18206 | 2026-03-19 |
| GitHub Discussion #17989 | https://github.com/mozilla/pdf.js/discussions/17989 | 2026-03-19 |
