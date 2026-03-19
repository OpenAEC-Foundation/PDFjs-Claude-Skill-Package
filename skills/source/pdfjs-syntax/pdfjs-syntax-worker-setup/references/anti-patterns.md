# Anti-Patterns (PDF.js Worker Setup)

## 1. Calling getDocument() Before Setting workerSrc

```javascript
// WRONG: workerSrc not set — falls back to fake worker (main thread parsing)
import * as pdfjsLib from 'pdfjs-dist';

const pdf = await pdfjsLib.getDocument('document.pdf').promise; // No worker!

// CORRECT: ALWAYS set workerSrc first
import * as pdfjsLib from 'pdfjs-dist';

pdfjsLib.GlobalWorkerOptions.workerSrc =
  'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/5.5.207/pdf.worker.min.mjs';
const pdf = await pdfjsLib.getDocument('document.pdf').promise;
```

**WHY**: Without `workerSrc`, PDF.js silently falls back to running the worker code on the main thread. This blocks the UI during PDF parsing, causing the page to freeze for large documents. There is no error — it just degrades silently.

---

## 2. Version Mismatch Between pdf.mjs and pdf.worker.mjs

```javascript
// WRONG: npm has pdfjs-dist@5.5.207 but worker is from a different version
pdfjsLib.GlobalWorkerOptions.workerSrc =
  'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/4.0.379/pdf.worker.min.mjs';

// CORRECT: Versions MUST match exactly
pdfjsLib.GlobalWorkerOptions.workerSrc =
  'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/5.5.207/pdf.worker.min.mjs';

// BEST: Automate version matching
import { version } from 'pdfjs-dist/package.json';
pdfjsLib.GlobalWorkerOptions.workerSrc =
  `https://cdnjs.cloudflare.com/ajax/libs/pdf.js/${version}/pdf.worker.min.mjs`;
```

**WHY**: PDF.js performs a strict version check between the main library and the worker. A mismatch throws: `"API version 'X.Y.Z' does not match Worker version 'A.B.C'"`. This error is not recoverable — the document cannot be loaded.

---

## 3. Using Relative Path Without Understanding Base URL

```javascript
// WRONG: Relative path resolved against document origin, not script location
pdfjsLib.GlobalWorkerOptions.workerSrc = './pdf.worker.min.mjs';
// If page is at /app/viewer, this resolves to /app/viewer/pdf.worker.min.mjs
// but the file is likely at /assets/pdf.worker.min.mjs

// CORRECT: Use absolute path from root
pdfjsLib.GlobalWorkerOptions.workerSrc = '/assets/pdf.worker.min.mjs';

// OR: Use a full URL
pdfjsLib.GlobalWorkerOptions.workerSrc =
  'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/5.5.207/pdf.worker.min.mjs';
```

**WHY**: The worker URL is resolved by the browser's `new Worker(url)` constructor, which uses the page's base URL as the resolution root. A relative path like `./file.mjs` resolves relative to the current page URL, not the JavaScript file that set it.

---

## 4. Missing cMapUrl for CJK Documents

```javascript
// WRONG: No cMapUrl — CJK characters render as blank rectangles
const pdf = await pdfjsLib.getDocument('chinese-document.pdf').promise;

// CORRECT: ALWAYS include cMapUrl for CJK text support
const pdf = await pdfjsLib.getDocument({
  url: 'chinese-document.pdf',
  cMapUrl: '/cmaps/',
  cMapPacked: true,
}).promise;
```

**WHY**: CJK (Chinese, Japanese, Korean) PDFs use character maps (CMaps) to decode text. Without `cMapUrl`, PDF.js cannot decode these characters, resulting in blank rectangles, tofu characters, or missing text.

---

## 5. Forgetting cMapPacked: true

```javascript
// WRONG: cMapUrl set but cMapPacked omitted
const pdf = await pdfjsLib.getDocument({
  url: 'document.pdf',
  cMapUrl: '/cmaps/',
  // cMapPacked not set — defaults to false
}).promise;

// CORRECT: ALWAYS pair cMapUrl with cMapPacked: true
const pdf = await pdfjsLib.getDocument({
  url: 'document.pdf',
  cMapUrl: '/cmaps/',
  cMapPacked: true,
}).promise;
```

**WHY**: pdfjs-dist ships CMaps in binary (packed) format. Without `cMapPacked: true`, PDF.js tries to parse them as plain text, causing CMap loading failures and broken CJK text rendering.

---

## 6. Setting workerSrc Multiple Times

```javascript
// WRONG: Setting workerSrc in every component
function ComponentA() {
  pdfjsLib.GlobalWorkerOptions.workerSrc = '/pdf.worker.min.mjs'; // set here
  // ...
}
function ComponentB() {
  pdfjsLib.GlobalWorkerOptions.workerSrc = '/pdf.worker.min.mjs'; // and here
  // ...
}

// CORRECT: Set workerSrc ONCE at application startup
// src/pdfSetup.ts
import * as pdfjsLib from 'pdfjs-dist';
pdfjsLib.GlobalWorkerOptions.workerSrc =
  'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/5.5.207/pdf.worker.min.mjs';
export { pdfjsLib };

// In components: import from the setup module
import { pdfjsLib } from './pdfSetup';
```

**WHY**: `GlobalWorkerOptions.workerSrc` is a global setting. Setting it multiple times is wasteful and risks race conditions if different parts of the application set different values. Set it once in a central initialization module.

---

## 7. Creating a New PDFWorker Per Document

```javascript
// WRONG: Creates and leaks worker instances
async function loadPdf(url: string) {
  const worker = new pdfjsLib.PDFWorker({ name: 'worker' });
  const pdf = await pdfjsLib.getDocument({ url, worker }).promise;
  // worker is never destroyed — memory leak!
  return pdf;
}

// CORRECT: Let PDF.js manage workers via GlobalWorkerOptions
pdfjsLib.GlobalWorkerOptions.workerSrc = '/pdf.worker.min.mjs';

async function loadPdf(url: string) {
  return pdfjsLib.getDocument(url).promise;
}
```

**WHY**: Each `PDFWorker` creates a new Web Worker thread. Creating one per document wastes resources and causes memory leaks if `destroy()` is not called. PDF.js internally pools and reuses workers when configured via `GlobalWorkerOptions.workerSrc`.

---

## 8. Using .js Extensions Instead of .mjs (pdfjs-dist 5.x)

```javascript
// WRONG: pdfjs-dist 5.x no longer ships .js files
pdfjsLib.GlobalWorkerOptions.workerSrc = '/pdf.worker.min.js';

// CORRECT: Use .mjs extension for pdfjs-dist 5.x
pdfjsLib.GlobalWorkerOptions.workerSrc = '/pdf.worker.min.mjs';
```

**WHY**: pdfjs-dist 5.x ships only ES modules with `.mjs` extensions. The older UMD builds (`.js`) are no longer available. Using `.js` extensions results in a 404 error.

---

## 9. Loading Worker Over HTTP on an HTTPS Page

```javascript
// WRONG: Mixed content — browser blocks HTTP worker on HTTPS page
pdfjsLib.GlobalWorkerOptions.workerSrc =
  'http://cdn.example.com/pdf.worker.min.mjs';

// CORRECT: ALWAYS use HTTPS for CDN URLs
pdfjsLib.GlobalWorkerOptions.workerSrc =
  'https://cdn.example.com/pdf.worker.min.mjs';
```

**WHY**: Browsers block mixed content (loading HTTP resources from an HTTPS page). The worker fails to load silently, and PDF.js falls back to fake worker mode on the main thread.

---

## 10. Using Fake Worker in Production

```javascript
// WRONG: Importing worker directly in production
import 'pdfjs-dist/build/pdf.worker.min.mjs';
// All PDF parsing now happens on the main thread!

// CORRECT: Use real worker in production, fake worker only for testing/SSR
pdfjsLib.GlobalWorkerOptions.workerSrc = '/pdf.worker.min.mjs';
```

**WHY**: Fake worker mode runs PDF parsing on the main thread, which blocks the UI. For large PDFs, this causes the page to freeze for several seconds. ONLY use fake worker mode in testing environments, SSR pre-rendering, or Node.js.
