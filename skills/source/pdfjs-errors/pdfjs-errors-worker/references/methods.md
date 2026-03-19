# Worker Error Types and API Reference (pdfjs-dist 5.x)

## GlobalWorkerOptions

Static configuration object that controls how PDF.js loads its Web Worker.

```typescript
import { GlobalWorkerOptions } from "pdfjs-dist";
```

### Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `workerSrc` | `string` | `""` | Path or URL to the worker script file. MUST be set before calling `getDocument()`. |
| `workerPort` | `Worker \| null` | `null` | Pre-created Worker instance. Overrides `workerSrc` when set. |

### workerSrc

```typescript
// ALWAYS set before any getDocument() call
GlobalWorkerOptions.workerSrc: string;
```

**Validation**: Accepts only string values. Throws an error for invalid types.

**File options for pdfjs-dist 5.x**:
| File | Format | Use Case |
|------|--------|----------|
| `pdf.worker.min.mjs` | ES module (minified) | Modern bundlers (Vite, Webpack 5) |
| `pdf.worker.mjs` | ES module | Development / debugging |
| `pdf.worker.min.js` | IIFE (minified) | Legacy script tags, importScripts() |
| `pdf.worker.js` | IIFE | Legacy development |

**ALWAYS** use `.mjs` files when your application uses ES modules. **NEVER** mix `.mjs` API with `.js` worker or vice versa.

### workerPort

```typescript
// Use a pre-created Worker instance instead of workerSrc
GlobalWorkerOptions.workerPort: Worker | null;
```

**Validation**: Accepts only Worker instances or null. Throws an error for invalid types.

**When to use**: When you need full control over Worker creation (e.g., custom headers, specific Worker options, SharedWorker).

```typescript
// Example: Custom Worker with specific options
const worker = new Worker("/pdf.worker.min.mjs", { type: "module" });
GlobalWorkerOptions.workerPort = worker;
```

**NEVER** set both `workerSrc` and `workerPort` -- `workerPort` takes precedence and `workerSrc` is ignored when `workerPort` is set.

---

## PDFWorker Class

Internal class that manages the worker lifecycle. Normally created automatically by `getDocument()`, but can be created manually for advanced use cases.

```typescript
import { PDFWorker } from "pdfjs-dist";
```

### Constructor Parameters

```typescript
interface PDFWorkerParameters {
  name?: string;       // Worker name for debugging (appears in browser DevTools)
  port?: Worker;       // Pre-created Worker/MessagePort instance
  verbosity?: number;  // Logging level (0=errors, 1=warnings, 5=info)
}
```

### Static Methods

| Method | Description |
|--------|-------------|
| `PDFWorker.fromPort(params)` | Creates PDFWorker from existing Worker port |

### Instance Properties

| Property | Type | Description |
|----------|------|-------------|
| `promise` | `Promise<void>` | Resolves when worker is ready |
| `port` | `Worker \| null` | The underlying Worker instance |
| `destroyed` | `boolean` | Whether the worker has been destroyed |
| `name` | `string` | Worker name (for debugging) |

### Instance Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `destroy()` | `void` | Terminates the worker and cleans up resources |

---

## Worker Error Messages Reference

### Critical Errors (Application Cannot Proceed)

#### Version Mismatch

```
The API version "5.0.0" does not match the Worker version "4.9.155".
```

- **Source**: `src/core/worker.js` -- thrown during worker handshake
- **Cause**: The API code (`pdf.min.mjs`) and worker code (`pdf.worker.min.mjs`) are from different pdfjs-dist versions
- **Severity**: Critical -- no PDF operations will succeed
- **Fix**: Ensure both files come from the same pdfjs-dist version

#### Missing Worker Source

```
No "GlobalWorkerOptions.workerSrc" specified.
```

- **Source**: `src/display/api.js` -- thrown during worker initialization
- **Cause**: `getDocument()` was called without configuring `GlobalWorkerOptions.workerSrc` or `GlobalWorkerOptions.workerPort`
- **Severity**: Critical -- triggers fake worker fallback (if available) or fails entirely
- **Fix**: Set `GlobalWorkerOptions.workerSrc` before calling `getDocument()`

### High-Severity Errors

#### Fake Worker Setup Failure

```
Setting up fake worker failed: "{reason.message}".
```

- **Source**: `src/display/api.js` -- thrown when fake worker initialization fails
- **Cause**: Dynamic import of the worker module failed (CSP restriction, missing module, path error)
- **Severity**: High -- no fallback available, PDF loading fails completely
- **Fix**: Ensure pdfjs-dist is properly installed and importable

#### Worker Construction Failure (Browser)

```
Failed to construct 'Worker': Script at '{url}' cannot be accessed from origin '{origin}'.
```

- **Source**: Browser -- thrown by the Worker constructor
- **Cause**: CORS policy blocks loading the worker script from a different origin
- **Severity**: High -- PDF.js may fall back to fake worker with degraded performance
- **Fix**: Self-host the worker file or use a CORS-compatible CDN

### Medium-Severity Errors

#### Worker Destroyed Prematurely

```
Worker was destroyed
```

- **Source**: `src/display/api.js` and `src/core/worker.js`
- **Cause**: `destroy()` was called on the PDFDocumentProxy or PDFWorker while operations were still pending
- **Severity**: Medium -- current operation fails, but app can recover
- **Fix**: Await all pending operations before calling `destroy()`

#### Port Reuse Conflict

```
Cannot use more than one PDFWorker per port.
```

- **Source**: `src/display/api.js` -- thrown in PDFWorker constructor
- **Cause**: Two PDFWorker instances were created with the same Worker port
- **Severity**: Medium -- second worker creation fails
- **Fix**: Use separate Worker instances or share a single PDFWorker

#### Worker Destroyed During Creation

```
PDFWorker.create - the worker is being destroyed.
Please remember to await `PDFDocumentLoadingTask.destroy()`-calls.
```

- **Source**: `src/display/api.js` -- thrown during worker initialization
- **Cause**: `destroy()` was called while the worker was still being created
- **Severity**: Medium -- race condition between creation and destruction
- **Fix**: Await the loading task's promise before destroying, or cancel the loading task first

---

## Worker File Locations in pdfjs-dist 5.x

```
node_modules/pdfjs-dist/
├── build/
│   ├── pdf.min.mjs              # API (ES module, minified)
│   ├── pdf.mjs                  # API (ES module)
│   ├── pdf.worker.min.mjs       # Worker (ES module, minified)
│   ├── pdf.worker.mjs           # Worker (ES module)
│   ├── pdf.min.js               # API (IIFE, minified)
│   ├── pdf.js                   # API (IIFE)
│   ├── pdf.worker.min.js        # Worker (IIFE, minified)
│   └── pdf.worker.js            # Worker (IIFE)
└── types/
    └── src/
        └── display/
            ├── api.d.ts          # TypeScript definitions
            └── worker_options.d.ts
```

**ALWAYS** match the format: if you use `pdf.min.mjs` (ESM), use `pdf.worker.min.mjs` (ESM). NEVER mix `.mjs` with `.js` builds.

---

## CDN URL Templates

### cdnjs (Cloudflare)

```
https://cdnjs.cloudflare.com/ajax/libs/pdf.js/{version}/pdf.worker.min.mjs
```

### unpkg

```
https://unpkg.com/pdfjs-dist@{version}/build/pdf.worker.min.mjs
```

### jsdelivr

```
https://cdn.jsdelivr.net/npm/pdfjs-dist@{version}/build/pdf.worker.min.mjs
```

**ALWAYS** replace `{version}` with the exact installed pdfjs-dist version (e.g., `5.0.0`). **NEVER** use `@latest` or floating version ranges in production CDN URLs.
