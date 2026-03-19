# GlobalWorkerOptions API & Configuration Objects

## GlobalWorkerOptions

The `GlobalWorkerOptions` object controls how PDF.js creates its Web Worker for background PDF parsing. It is a static object on the `pdfjs-dist` module — not instantiated.

### Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `workerSrc` | `string` | `""` | URL or path to the `pdf.worker.min.mjs` file. MUST be set before any `getDocument()` call. |
| `workerPort` | `Worker \| null` | `null` | Pre-created Worker instance. When set, PDF.js uses this worker instead of creating one from `workerSrc`. Used for fake worker mode or shared worker setups. |

### workerSrc

The primary configuration property. Accepts:

- **Absolute URL**: `'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/5.5.207/pdf.worker.min.mjs'`
- **Relative URL**: `'/assets/pdf.worker.min.mjs'` (resolved relative to document origin)
- **Bundler-resolved URL**: Output of `new URL(...)` or `import ... ?url`

```javascript
import * as pdfjsLib from 'pdfjs-dist';

// Set ONCE at application startup, BEFORE any getDocument() call
pdfjsLib.GlobalWorkerOptions.workerSrc =
  'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/5.5.207/pdf.worker.min.mjs';
```

**Rules:**
- ALWAYS set before calling `getDocument()`
- ALWAYS match the version to your installed `pdfjs-dist` version
- NEVER set to an empty string in production — this triggers fake worker fallback

### workerPort

Allows providing a pre-instantiated Worker object:

```javascript
import * as pdfjsLib from 'pdfjs-dist';

const worker = new Worker('/assets/pdf.worker.min.mjs', { type: 'module' });
pdfjsLib.GlobalWorkerOptions.workerPort = worker;
```

**Rules:**
- When `workerPort` is set, `workerSrc` is ignored
- ONLY use when you need explicit control over the worker lifecycle
- The worker MUST be a module worker (`type: 'module'`) for pdfjs-dist 5.x

---

## DocumentInitParameters

Passed to `getDocument()` to configure document loading. Worker-related properties:

### Core Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `url` | `string \| URL` | — | URL of the PDF file to load |
| `data` | `TypedArray \| ArrayBuffer \| string` | — | Binary PDF data (alternative to `url`) |
| `httpHeaders` | `Object` | — | HTTP headers for the fetch request |
| `withCredentials` | `boolean` | `false` | Include cookies in cross-origin requests |
| `password` | `string` | — | Password for encrypted PDFs |
| `range` | `PDFDataRangeTransport` | — | Custom range request handler |

### CMap Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `cMapUrl` | `string` | — | URL to the `cmaps/` directory. ALWAYS set for CJK text support. |
| `cMapPacked` | `boolean` | `false` | ALWAYS set to `true` — pdfjs-dist ships binary (packed) CMaps. |

```javascript
const loadingTask = pdfjsLib.getDocument({
  url: 'document.pdf',
  cMapUrl: '/cmaps/',         // Trailing slash required
  cMapPacked: true,           // ALWAYS true for pdfjs-dist
});
```

**CMap files location in pdfjs-dist:**
- npm: `node_modules/pdfjs-dist/cmaps/`
- MUST be copied to your public/static assets directory
- Contains ~200 binary CMap files for CJK character set mapping

### Standard Font Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `standardFontDataUrl` | `string` | — | URL to the `standard_fonts/` directory for the 14 standard PDF fonts. |

```javascript
const loadingTask = pdfjsLib.getDocument({
  url: 'document.pdf',
  standardFontDataUrl: '/standard_fonts/',   // Trailing slash required
});
```

**Standard fonts location in pdfjs-dist:**
- npm: `node_modules/pdfjs-dist/standard_fonts/`
- MUST be copied to your public/static assets directory
- Contains font data for the 14 standard PDF fonts (Courier, Helvetica, Times, Symbol, ZapfDingbats variants)

### Worker Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `worker` | `PDFWorker` | — | Pre-created PDFWorker instance for advanced use cases |
| `disableAutoFetch` | `boolean` | `false` | Disable automatic background fetching of PDF data |
| `disableStream` | `boolean` | `false` | Disable streaming of PDF data |

---

## PDFWorker

Low-level class for managing worker instances directly. Most users should use `GlobalWorkerOptions.workerSrc` instead.

### Constructor

```javascript
const worker = new pdfjsLib.PDFWorker({ name: 'my-worker' });
```

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `name` | `string` | Worker name (for debugging) |
| `destroyed` | `boolean` | Whether the worker has been terminated |
| `promise` | `Promise<void>` | Resolves when worker is ready |

### Methods

| Method | Return | Description |
|--------|--------|-------------|
| `destroy()` | `void` | Terminate the worker thread |

### Usage with getDocument

```javascript
const worker = new pdfjsLib.PDFWorker({ name: 'pdf-worker-1' });
await worker.promise; // Wait for worker to be ready

const loadingTask = pdfjsLib.getDocument({
  url: 'document.pdf',
  worker: worker,     // Reuse this worker across multiple documents
});
const pdf = await loadingTask.promise;

// When done with ALL documents using this worker:
worker.destroy();
```

**Rules:**
- ONLY use `PDFWorker` directly when you need to share a single worker across multiple documents
- ALWAYS call `destroy()` when the worker is no longer needed to free resources
- NEVER create a new `PDFWorker` per document — let PDF.js manage workers via `GlobalWorkerOptions.workerSrc`

---

## File Extension Reference (pdfjs-dist 5.x)

| File | Purpose | Notes |
|------|---------|-------|
| `pdf.mjs` | Main library (ES module) | Primary import for applications |
| `pdf.worker.mjs` | Worker (ES module, unminified) | Development use |
| `pdf.worker.min.mjs` | Worker (ES module, minified) | Production use — ALWAYS use this |
| `pdf.sandbox.mjs` | JavaScript sandbox for forms | Only needed for PDF forms with JS |

**IMPORTANT**: pdfjs-dist 5.x uses `.mjs` extensions exclusively for ES modules. The older `.js` extensions and UMD builds (`pdf.js`, `pdf.worker.js`) are no longer available.
