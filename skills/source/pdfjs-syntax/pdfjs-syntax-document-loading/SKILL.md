---
name: pdfjs-syntax-document-loading
description: "Loads PDF documents using getDocument() with all source types. Covers PDFDocumentLoadingTask, PDFDocumentProxy, PDFPageProxy, progress tracking, cancellation, metadata extraction, outline/bookmarks, and document cleanup. Activates when loading PDFs, extracting PDF metadata, getting page count, or accessing PDF document properties."
license: MIT
compatibility: "Designed for Claude Code. Requires pdfjs-dist 5.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# pdfjs-syntax-document-loading

## Quick Reference

### Core Loading API

| Function / Class | Purpose | Import |
|-----------------|---------|--------|
| `getDocument(source)` | Create a loading task from any source type | `import { getDocument } from 'pdfjs-dist'` |
| `PDFDocumentLoadingTask` | Returned by `getDocument()` — tracks loading progress, provides `promise` | Returned type |
| `PDFDocumentProxy` | Resolved from loading task — represents a loaded PDF document | Resolved type |
| `PDFPageProxy` | Resolved from `doc.getPage(n)` — represents a single page | Resolved type |

### Source Types

| Source | Parameter | When to Use |
|--------|-----------|-------------|
| Remote URL | `{ url: 'https://...' }` | PDF hosted on a server |
| ArrayBuffer | `{ data: arrayBuffer }` | File from `<input type="file">` via `FileReader` |
| Uint8Array | `{ data: uint8Array }` | Raw binary data, Node.js buffers |
| Base64 string | `{ data: atob(b64) }` | PDF received as base64 from an API |

### Critical Warnings

**NEVER** call `getDocument()` before setting `GlobalWorkerOptions.workerSrc` — the worker is required for parsing. Without it, PDF.js falls back to a fake worker on the main thread, causing UI freezes on large documents.

**ALWAYS** call `doc.destroy()` when finished with a document to prevent memory leaks. Each loaded document holds parsed page data, fonts, and image caches in memory.

**NEVER** pass a raw string of binary data to `getDocument()` — ALWAYS convert to `Uint8Array` first. Strings undergo encoding transformations that corrupt binary PDF data.

**ALWAYS** handle `PasswordException` for user-uploaded PDFs — a missing password causes an unrecoverable rejection without a meaningful error message if not caught.

**ALWAYS** use 1-based page numbers with `getPage()` — page 1 is `getPage(1)`, NOT `getPage(0)`. Using 0 throws a "Page number out of range" error.

**NEVER** load all pages synchronously in a tight loop — ALWAYS use async iteration or lazy loading to avoid blocking the main thread and consuming excessive memory.

---

## Decision Tree: Choosing a Source Type

```
How do you have the PDF data?
│
├─ URL to a remote/local file
│  └─ Use { url: '...' }
│     ├─ Need auth? → Add httpHeaders / withCredentials
│     └─ Large file? → Set rangeChunkSize for streaming
│
├─ File from <input type="file"> or drag-and-drop
│  └─ Use FileReader.readAsArrayBuffer() → { data: arrayBuffer }
│
├─ Raw bytes (fetch response, Node.js Buffer)
│  └─ Use { data: new Uint8Array(buffer) }
│
└─ Base64-encoded string (from API response)
   └─ Decode first:
      const raw = atob(base64String);
      const bytes = new Uint8Array(raw.length);
      for (let i = 0; i < raw.length; i++) bytes[i] = raw.charCodeAt(i);
      → { data: bytes }
```

---

## Loading Lifecycle

```
getDocument(source)
    │
    ├─ Returns: PDFDocumentLoadingTask
    │   ├─ .promise          → Promise<PDFDocumentProxy>
    │   ├─ .onProgress       → callback({loaded, total})
    │   └─ .destroy()        → cancel loading
    │
    ▼
PDFDocumentProxy (loaded document)
    │
    ├─ .numPages             → total page count
    ├─ .fingerprints         → [string, string | null]
    ├─ .isPureXfa            → boolean (XFA-only form)
    │
    ├─ .getPage(num)         → Promise<PDFPageProxy>
    ├─ .getMetadata()        → Promise<{info, metadata, ...}>
    ├─ .getOutline()         → Promise<Array | null>
    ├─ .getAttachments()     → Promise<Object | null>
    ├─ .getFieldObjects()    → Promise<Object | null>
    ├─ .getPermissions()     → Promise<Array | null>
    │
    ├─ .cleanup()            → release rendering caches (keep document)
    └─ .destroy()            → release ALL resources (document gone)
        │
        ▼
    PDFPageProxy (single page)
        │
        ├─ .pageNumber       → 1-based page number
        ├─ .rotate           → rotation in degrees (0, 90, 180, 270)
        ├─ .userUnit         → user space unit (default 1.0 = 1/72 inch)
        ├─ .view             → [x1, y1, x2, y2] bounding box
        │
        ├─ .getViewport(params)    → PageViewport
        ├─ .render(params)         → RenderTask
        ├─ .getTextContent(params) → Promise<TextContent>
        ├─ .getAnnotations(params) → Promise<Array>
        └─ .cleanup()              → release page rendering resources
```

---

## Essential Patterns

### Minimal Document Load

```typescript
import { getDocument, GlobalWorkerOptions } from 'pdfjs-dist';

// ALWAYS set workerSrc BEFORE calling getDocument
GlobalWorkerOptions.workerSrc = new URL(
  'pdfjs-dist/build/pdf.worker.min.mjs',
  import.meta.url
).toString();

const loadingTask = getDocument('https://example.com/document.pdf');
const doc = await loadingTask.promise;

console.log(`Pages: ${doc.numPages}`);

// ALWAYS destroy when done
await doc.destroy();
```

### Loading with Progress Tracking

```typescript
const loadingTask = getDocument({ url: '/large-document.pdf' });

loadingTask.onProgress = ({ loaded, total }) => {
  if (total > 0) {
    const percent = Math.round((loaded / total) * 100);
    console.log(`Loading: ${percent}%`);
  }
};

const doc = await loadingTask.promise;
```

### Loading Password-Protected PDFs

```typescript
import { getDocument, PasswordResponses } from 'pdfjs-dist';

async function loadProtectedPdf(
  source: string | Uint8Array,
  passwordProvider: () => Promise<string>
): Promise<PDFDocumentProxy> {
  try {
    const loadingTask = getDocument({ url: source });
    return await loadingTask.promise;
  } catch (error) {
    if (error.name === 'PasswordException') {
      if (error.code === PasswordResponses.NEED_PASSWORD ||
          error.code === PasswordResponses.INCORRECT_PASSWORD) {
        const password = await passwordProvider();
        const retryTask = getDocument({ url: source, password });
        return await retryTask.promise;
      }
    }
    throw error;
  }
}
```

### Cancelling a Load

```typescript
const loadingTask = getDocument({ url: '/document.pdf' });

// Cancel after timeout
setTimeout(() => {
  loadingTask.destroy(); // Rejects the promise with 'Loading aborted'
}, 5000);

try {
  const doc = await loadingTask.promise;
} catch (error) {
  if (error.message === 'Loading aborted') {
    console.log('Load was cancelled');
  }
}
```

---

## Memory Management

### cleanup() vs destroy()

| Method | Scope | Effect | Use When |
|--------|-------|--------|----------|
| `page.cleanup()` | Single page | Releases cached rendering data (images, fonts) for that page | Scrolling away from a page in a viewer |
| `doc.cleanup()` | All pages | Calls `cleanup()` on every cached page | Reducing memory in long-running viewer |
| `doc.destroy()` | Entire document | Releases ALL resources, terminates worker connection | Done with the document entirely |

### Rules

- **ALWAYS** call `doc.destroy()` before loading a new document in a single-document viewer
- **ALWAYS** call `doc.destroy()` in component unmount/cleanup handlers (React `useEffect` return, Angular `ngOnDestroy`)
- **NEVER** use a `PDFDocumentProxy` or its pages after calling `destroy()` — all methods will reject

```typescript
// React cleanup pattern
useEffect(() => {
  const loadingTask = getDocument({ url: pdfUrl });
  let doc: PDFDocumentProxy | null = null;

  loadingTask.promise.then((loadedDoc) => {
    doc = loadedDoc;
    // ... render pages
  });

  return () => {
    loadingTask.destroy(); // Cancel if still loading
    doc?.destroy();        // Release if loaded
  };
}, [pdfUrl]);
```

---

## DocumentInitParameters Reference

ALWAYS pass an object to `getDocument()` for anything beyond a simple URL string. Key parameters:

| Parameter | Type | Default | Purpose |
|-----------|------|---------|---------|
| `url` | `string` | -- | URL to PDF |
| `data` | `ArrayBuffer \| TypedArray` | -- | Raw PDF binary data |
| `httpHeaders` | `Record<string, string>` | -- | Custom HTTP headers for URL loading |
| `withCredentials` | `boolean` | `false` | Include cookies in cross-origin requests |
| `password` | `string` | -- | Password for encrypted PDFs |
| `cMapUrl` | `string` | -- | Path to CMap files (required for CJK text) |
| `cMapPacked` | `boolean` | `true` | Whether CMap files are binary packed |
| `standardFontDataUrl` | `string` | -- | Path to standard font files |
| `rangeChunkSize` | `number` | `65536` | Bytes per range request chunk |
| `docBaseUrl` | `string` | -- | Base URL for resolving relative links in the PDF |
| `stopAtErrors` | `boolean` | `false` | Stop parsing on recoverable errors |
| `maxImageSize` | `number` | `-1` | Max pixels for images (-1 = unlimited) |

For complete parameter list and method signatures, see [references/methods.md](references/methods.md).

---

## Reference Links

- [references/methods.md](references/methods.md) -- Complete API signatures for getDocument, PDFDocumentLoadingTask, PDFDocumentProxy, PDFPageProxy
- [references/examples.md](references/examples.md) -- Working code examples for all source types, progress tracking, password handling
- [references/anti-patterns.md](references/anti-patterns.md) -- Common mistakes and their fixes

### Official Sources

- https://mozilla.github.io/pdf.js/api/
- https://github.com/nicolo-ribaudo/pdfjs-dist/tree/master (npm package source)
- https://github.com/nicolo-ribaudo/pdfjs-dist/blob/master/types/src/display/api.d.ts (TypeScript definitions)
- https://mozilla.github.io/pdf.js/getting_started/
