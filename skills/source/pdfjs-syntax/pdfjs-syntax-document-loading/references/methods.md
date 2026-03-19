# Document Loading API — Method Reference

> Complete signatures for pdfjs-dist 5.x document loading API.

---

## getDocument()

```typescript
function getDocument(
  src: string | URL | ArrayBuffer | TypedArray | DocumentInitParameters
): PDFDocumentLoadingTask;
```

When `src` is a string, it is treated as a URL. For all other source types, pass a `DocumentInitParameters` object.

### DocumentInitParameters (complete)

```typescript
interface DocumentInitParameters {
  // Source (provide ONE of url or data)
  url?: string | URL;
  data?: ArrayBuffer | TypedArray;

  // HTTP options (url-based loading only)
  httpHeaders?: Record<string, string>;
  withCredentials?: boolean;

  // Range requests
  range?: PDFDataRangeTransport;
  rangeChunkSize?: number;           // default: 65536

  // Security
  password?: string;

  // Font & CMap resources
  cMapUrl?: string;
  cMapPacked?: boolean;              // default: true
  standardFontDataUrl?: string;

  // Worker
  worker?: PDFWorker;
  useWorkerFetch?: boolean;          // default: true (worker fetches URLs)

  // Parsing behavior
  stopAtErrors?: boolean;            // default: false
  maxImageSize?: number;             // default: -1 (unlimited)
  isEvalSupported?: boolean;         // default: true
  isOffscreenCanvasSupported?: boolean;
  canvasMaxAreaInBytes?: number;     // default: -1

  // Document context
  docBaseUrl?: string;
  enableXfa?: boolean;               // default: false

  // Advanced
  disableAutoFetch?: boolean;        // default: false
  disableFontFace?: boolean;         // default: false
  disableRange?: boolean;            // default: false
  disableStream?: boolean;           // default: false
  length?: number;                   // PDF file length hint
}
```

---

## PDFDocumentLoadingTask

Returned by `getDocument()`. Controls the loading process.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `promise` | `Promise<PDFDocumentProxy>` | Resolves when document is loaded and parsed |
| `docId` | `string` | Unique document identifier |
| `destroyed` | `boolean` | Whether `destroy()` has been called |
| `onPassword` | `(callback, reason) => void` | Password request callback |
| `onProgress` | `(progressData) => void` | Progress callback |

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `destroy()` | `(): Promise<void>` | Cancel loading and release resources. Rejects `promise` with "Loading aborted" |

### onProgress Callback

```typescript
loadingTask.onProgress = (progressData: { loaded: number; total: number }) => {
  // loaded: bytes received so far
  // total: total bytes (0 if Content-Length header missing)
};
```

### onPassword Callback

```typescript
loadingTask.onPassword = (
  updateCallback: (password: string) => void,
  reason: number  // PasswordResponses.NEED_PASSWORD (1) or INCORRECT_PASSWORD (2)
) => {
  const password = prompt('Enter PDF password:');
  if (password) {
    updateCallback(password);
  } else {
    updateCallback(''); // Will reject with PasswordException
  }
};
```

---

## PDFDocumentProxy

Represents a fully loaded PDF document.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `numPages` | `number` | Total number of pages |
| `fingerprints` | `[string, string \| null]` | Document fingerprints (MD5-based) |
| `isPureXfa` | `boolean` | `true` if document contains only XFA form data |
| `allXfaHtml` | `Object \| null` | XFA HTML representation (if enabled) |
| `annotationStorage` | `AnnotationStorage` | Storage for modified annotation values |
| `filterFactory` | `Object` | Factory for image filters |
| `loadingParams` | `Object` | Parameters used during loading |

### Methods

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `getPage` | `(pageNumber: number)` | `Promise<PDFPageProxy>` | Get page by 1-based number. Throws if `pageNumber < 1` or `> numPages` |
| `getPageIndex` | `(ref: Object)` | `Promise<number>` | Get 0-based page index from a destination reference |
| `getPageLabels` | `()` | `Promise<string[] \| null>` | Get page labels (e.g., "i", "ii", "1", "2") |
| `getPageLayout` | `()` | `Promise<string>` | Get page layout hint (e.g., "SinglePage", "TwoColumnLeft") |
| `getPageMode` | `()` | `Promise<string>` | Get page mode (e.g., "UseNone", "UseOutlines", "UseThumbs") |
| `getMetadata` | `()` | `Promise<{ info: Object, metadata: Metadata \| null, contentDispositionFilename: string \| null }>` | Get document metadata |
| `getOutline` | `()` | `Promise<Array \| null>` | Get table of contents / bookmarks tree |
| `getData` | `()` | `Promise<Uint8Array>` | Get raw PDF bytes |
| `getDownloadInfo` | `()` | `Promise<{ length: number }>` | Get file size in bytes |
| `getAttachments` | `()` | `Promise<Record<string, { filename: string, content: Uint8Array }> \| null>` | Get embedded file attachments |
| `getFieldObjects` | `()` | `Promise<Record<string, Object[]> \| null>` | Get form field definitions |
| `getPermissions` | `()` | `Promise<number[] \| null>` | Get document permissions flags |
| `getViewerPreferences` | `()` | `Promise<Object \| null>` | Get viewer preferences (print scaling, duplex, etc.) |
| `getOpenAction` | `()` | `Promise<Object \| null>` | Get document open action (e.g., go to page, run JavaScript) |
| `getMarkInfo` | `()` | `Promise<Object \| null>` | Get mark information (tagged PDF) |
| `getOptionalContentConfig` | `()` | `Promise<OptionalContentConfig>` | Get optional content (layer) configuration |
| `getJSActions` | `()` | `Promise<Object \| null>` | Get document-level JavaScript actions |
| `saveDocument` | `()` | `Promise<Uint8Array>` | Save document with modified annotations/form data |
| `cleanup` | `(keepLoadedFonts?: boolean)` | `Promise<void>` | Release cached rendering data for all pages |
| `destroy` | `()` | `Promise<void>` | Release ALL resources. Document is unusable after this call |

### getMetadata() Response Shape

```typescript
const { info, metadata, contentDispositionFilename } = await doc.getMetadata();

// info: Object with standard PDF metadata keys
interface PDFInfo {
  Title?: string;
  Author?: string;
  Subject?: string;
  Keywords?: string;
  Creator?: string;      // Application that created the document
  Producer?: string;     // PDF producer library
  CreationDate?: string; // D:YYYYMMDDHHmmSS format
  ModDate?: string;      // D:YYYYMMDDHHmmSS format
  PDFFormatVersion?: string; // e.g., "1.7"
  Language?: string;
  IsAcroFormPresent?: boolean;
  IsXFAPresent?: boolean;
  IsCollectionPresent?: boolean;
  IsSignaturesPresent?: boolean;
}

// metadata: Metadata object (XMP metadata), or null
// metadata.get('dc:title'), metadata.getAll(), etc.

// contentDispositionFilename: string | null
// From Content-Disposition header if loaded via URL
```

### getOutline() Response Shape

```typescript
interface OutlineNode {
  title: string;          // Bookmark text
  bold: boolean;          // Bold styling
  italic: boolean;        // Italic styling
  color: Uint8ClampedArray; // RGB color [r, g, b]
  dest: string | Array;   // Named destination or explicit destination
  url: string | null;     // External URL
  unsafeUrl: string;      // Original URL (may be unsafe)
  newWindow: boolean;     // Open in new window
  count: number;          // Number of visible child items
  items: OutlineNode[];   // Child bookmarks
}

// Returns OutlineNode[] | null
const outline = await doc.getOutline();
```

---

## PDFPageProxy

Represents a single page within a PDF document.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `pageNumber` | `number` | 1-based page number |
| `rotate` | `number` | Page rotation in degrees (0, 90, 180, 270) |
| `ref` | `Object` | Page reference object |
| `userUnit` | `number` | User space units (default: 1.0 = 1/72 inch) |
| `view` | `[number, number, number, number]` | Page bounding box `[x1, y1, x2, y2]` in user units |

### Methods

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `getViewport` | `(params: GetViewportParameters)` | `PageViewport` | Calculate viewport for given scale/rotation |
| `render` | `(params: RenderParameters)` | `RenderTask` | Render page to a canvas context |
| `getTextContent` | `(params?: GetTextContentParameters)` | `Promise<TextContent>` | Extract text with positions |
| `streamTextContent` | `(params?: GetTextContentParameters)` | `ReadableStream` | Stream text content incrementally |
| `getAnnotations` | `(params?: GetAnnotationsParameters)` | `Promise<Object[]>` | Get page annotations |
| `getOperatorList` | `(params?: GetOperatorListParameters)` | `Promise<PDFOperatorList>` | Get low-level rendering operators |
| `getStructTree` | `()` | `Promise<Object \| null>` | Get structure tree (tagged PDF) |
| `cleanup` | `(resetStats?: boolean)` | `boolean` | Release page rendering caches |

### getViewport Parameters

```typescript
interface GetViewportParameters {
  scale: number;           // REQUIRED — zoom level (1.0 = 100%)
  rotation?: number;       // Additional rotation in degrees (default: 0)
  offsetX?: number;        // Horizontal offset (default: 0)
  offsetY?: number;        // Vertical offset (default: 0)
  dontFlip?: boolean;      // Do not flip Y axis (default: false)
}
```

### PageViewport (returned by getViewport)

```typescript
interface PageViewport {
  viewBox: number[];     // Original page box
  scale: number;         // Applied scale
  rotation: number;      // Applied rotation
  offsetX: number;       // Applied X offset
  offsetY: number;       // Applied Y offset
  width: number;         // Calculated pixel width
  height: number;        // Calculated pixel height
  transform: number[];   // 6-element transform matrix [a, b, c, d, e, f]
  convertToViewportPoint(x: number, y: number): [number, number];
  convertToPdfPoint(x: number, y: number): [number, number];
  clone(params?: Partial<GetViewportParameters>): PageViewport;
}
```

---

## PasswordResponses Enum

```typescript
import { PasswordResponses } from 'pdfjs-dist';

PasswordResponses.NEED_PASSWORD     // = 1 — Document requires a password
PasswordResponses.INCORRECT_PASSWORD // = 2 — Provided password was wrong
```
