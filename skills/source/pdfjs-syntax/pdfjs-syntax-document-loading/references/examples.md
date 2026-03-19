# Document Loading — Code Examples

> All examples target pdfjs-dist 5.x with TypeScript.

---

## Worker Setup (ALWAYS do this first)

```typescript
import { GlobalWorkerOptions } from 'pdfjs-dist';

// Option 1: ES module bundler (Vite, Webpack 5, Rollup)
GlobalWorkerOptions.workerSrc = new URL(
  'pdfjs-dist/build/pdf.worker.min.mjs',
  import.meta.url
).toString();

// Option 2: CDN
GlobalWorkerOptions.workerSrc =
  'https://unpkg.com/pdfjs-dist@5.0.375/build/pdf.worker.min.mjs';

// Option 3: Local copy in public directory
GlobalWorkerOptions.workerSrc = '/pdf.worker.min.mjs';
```

---

## Loading from URL

```typescript
import { getDocument, GlobalWorkerOptions } from 'pdfjs-dist';
import type { PDFDocumentProxy } from 'pdfjs-dist';

GlobalWorkerOptions.workerSrc = new URL(
  'pdfjs-dist/build/pdf.worker.min.mjs',
  import.meta.url
).toString();

async function loadFromUrl(url: string): Promise<PDFDocumentProxy> {
  const loadingTask = getDocument(url);
  const doc = await loadingTask.promise;
  console.log(`Loaded ${doc.numPages} pages from ${url}`);
  return doc;
}
```

### With Custom HTTP Headers

```typescript
const doc = await getDocument({
  url: 'https://api.example.com/documents/report.pdf',
  httpHeaders: {
    'Authorization': 'Bearer eyJhbGciOiJIUzI1NiIs...',
    'X-Custom-Header': 'value',
  },
  withCredentials: true, // Include cookies for cross-origin
}).promise;
```

---

## Loading from File Input

```typescript
async function loadFromFileInput(file: File): Promise<PDFDocumentProxy> {
  const arrayBuffer = await file.arrayBuffer();
  const loadingTask = getDocument({ data: arrayBuffer });
  return await loadingTask.promise;
}

// Usage with <input type="file">
const fileInput = document.querySelector<HTMLInputElement>('#pdf-input');
fileInput?.addEventListener('change', async (event) => {
  const file = (event.target as HTMLInputElement).files?.[0];
  if (!file) return;

  const doc = await loadFromFileInput(file);
  console.log(`File "${file.name}" has ${doc.numPages} pages`);

  // ALWAYS destroy when done
  await doc.destroy();
});
```

---

## Loading from ArrayBuffer

```typescript
async function loadFromFetch(url: string): Promise<PDFDocumentProxy> {
  const response = await fetch(url);
  const arrayBuffer = await response.arrayBuffer();
  return await getDocument({ data: arrayBuffer }).promise;
}
```

---

## Loading from Uint8Array

```typescript
// From Node.js Buffer
function loadFromBuffer(buffer: Buffer): PDFDocumentLoadingTask {
  const uint8Array = new Uint8Array(buffer);
  return getDocument({ data: uint8Array });
}

// From raw bytes
function loadFromBytes(bytes: number[]): PDFDocumentLoadingTask {
  return getDocument({ data: new Uint8Array(bytes) });
}
```

---

## Loading from Base64

```typescript
function base64ToUint8Array(base64: string): Uint8Array {
  const binaryString = atob(base64);
  const bytes = new Uint8Array(binaryString.length);
  for (let i = 0; i < binaryString.length; i++) {
    bytes[i] = binaryString.charCodeAt(i);
  }
  return bytes;
}

async function loadFromBase64(base64Pdf: string): Promise<PDFDocumentProxy> {
  const data = base64ToUint8Array(base64Pdf);
  return await getDocument({ data }).promise;
}
```

---

## Loading with Progress Tracking

```typescript
interface LoadProgress {
  loaded: number;
  total: number;
  percent: number;
}

async function loadWithProgress(
  url: string,
  onProgress: (progress: LoadProgress) => void
): Promise<PDFDocumentProxy> {
  const loadingTask = getDocument({ url });

  loadingTask.onProgress = ({ loaded, total }) => {
    onProgress({
      loaded,
      total,
      percent: total > 0 ? Math.round((loaded / total) * 100) : 0,
    });
  };

  return await loadingTask.promise;
}

// Usage
const doc = await loadWithProgress('/large-report.pdf', (progress) => {
  progressBar.style.width = `${progress.percent}%`;
  progressBar.textContent = `${progress.percent}%`;
});
```

---

## Loading Password-Protected PDFs

### Using password parameter directly

```typescript
async function loadWithPassword(
  url: string,
  password: string
): Promise<PDFDocumentProxy> {
  return await getDocument({ url, password }).promise;
}
```

### Using onPassword callback (interactive)

```typescript
import { getDocument, PasswordResponses } from 'pdfjs-dist';

async function loadWithPasswordPrompt(url: string): Promise<PDFDocumentProxy> {
  const loadingTask = getDocument({ url });

  loadingTask.onPassword = (updateCallback, reason) => {
    const message = reason === PasswordResponses.NEED_PASSWORD
      ? 'This PDF is password-protected. Enter password:'
      : 'Incorrect password. Try again:';

    const password = prompt(message);
    if (password !== null) {
      updateCallback(password);
    } else {
      // User cancelled — this will reject the promise
      updateCallback('');
    }
  };

  return await loadingTask.promise;
}
```

### Retry pattern with error handling

```typescript
import { getDocument, PasswordResponses } from 'pdfjs-dist';
import type { PDFDocumentProxy } from 'pdfjs-dist';

async function loadWithRetry(
  url: string,
  getPassword: (isRetry: boolean) => Promise<string | null>,
  maxAttempts = 3
): Promise<PDFDocumentProxy> {
  let attempts = 0;
  let lastError: unknown;

  while (attempts < maxAttempts) {
    try {
      const params: Record<string, unknown> = { url };
      if (attempts > 0) {
        const password = await getPassword(attempts > 1);
        if (password === null) throw new Error('User cancelled');
        params.password = password;
      }
      return await getDocument(params).promise;
    } catch (error: unknown) {
      lastError = error;
      if (error instanceof Error && error.name === 'PasswordException') {
        attempts++;
        continue;
      }
      throw error; // Non-password error
    }
  }

  throw lastError;
}
```

---

## Extracting Metadata

```typescript
async function extractMetadata(doc: PDFDocumentProxy) {
  const { info, metadata, contentDispositionFilename } =
    await doc.getMetadata();

  return {
    title: info.Title || null,
    author: info.Author || null,
    subject: info.Subject || null,
    keywords: info.Keywords || null,
    creator: info.Creator || null,
    producer: info.Producer || null,
    creationDate: info.CreationDate || null,
    modDate: info.ModDate || null,
    pdfVersion: info.PDFFormatVersion || null,
    pageCount: doc.numPages,
    isAcroForm: info.IsAcroFormPresent || false,
    isXFA: info.IsXFAPresent || false,
    filename: contentDispositionFilename || null,
    // XMP metadata (if available)
    xmpTitle: metadata?.get('dc:title') || null,
  };
}
```

---

## Extracting Outline (Bookmarks)

```typescript
interface Bookmark {
  title: string;
  pageNumber: number | null;
  children: Bookmark[];
}

async function extractBookmarks(doc: PDFDocumentProxy): Promise<Bookmark[]> {
  const outline = await doc.getOutline();
  if (!outline) return [];

  async function processNode(node: any): Promise<Bookmark> {
    let pageNumber: number | null = null;

    if (node.dest) {
      try {
        // Named destination (string) or explicit destination (array)
        const dest = typeof node.dest === 'string'
          ? await doc.getDestination(node.dest)
          : node.dest;

        if (dest && dest[0]) {
          pageNumber = (await doc.getPageIndex(dest[0])) + 1; // Convert to 1-based
        }
      } catch {
        // Destination could not be resolved
      }
    }

    const children = await Promise.all(
      (node.items || []).map(processNode)
    );

    return { title: node.title, pageNumber, children };
  }

  return Promise.all(outline.map(processNode));
}
```

---

## Cancelling a Load

```typescript
class PdfLoader {
  private currentTask: PDFDocumentLoadingTask | null = null;

  async load(url: string): Promise<PDFDocumentProxy> {
    // Cancel previous load if still in progress
    if (this.currentTask) {
      await this.currentTask.destroy();
      this.currentTask = null;
    }

    this.currentTask = getDocument({ url });

    try {
      const doc = await this.currentTask.promise;
      return doc;
    } catch (error) {
      if (error instanceof Error && error.message === 'Loading aborted') {
        throw new Error('Load was cancelled by a newer request');
      }
      throw error;
    } finally {
      this.currentTask = null;
    }
  }

  cancel(): void {
    this.currentTask?.destroy();
    this.currentTask = null;
  }
}
```

---

## CJK Text Support (CMap Configuration)

```typescript
const doc = await getDocument({
  url: '/chinese-document.pdf',
  cMapUrl: '/node_modules/pdfjs-dist/cmaps/',
  cMapPacked: true,
}).promise;
```

---

## Full Example: Document Summary Extractor

```typescript
import { getDocument, GlobalWorkerOptions } from 'pdfjs-dist';
import type { PDFDocumentProxy } from 'pdfjs-dist';

GlobalWorkerOptions.workerSrc = new URL(
  'pdfjs-dist/build/pdf.worker.min.mjs',
  import.meta.url
).toString();

interface DocumentSummary {
  pageCount: number;
  title: string | null;
  author: string | null;
  hasOutline: boolean;
  hasAttachments: boolean;
  hasFormFields: boolean;
  fileSizeBytes: number;
  firstPageDimensions: { width: number; height: number };
}

async function getDocumentSummary(
  source: string | Uint8Array
): Promise<DocumentSummary> {
  const params = typeof source === 'string'
    ? { url: source }
    : { data: source };

  const doc = await getDocument(params).promise;

  try {
    const { info } = await doc.getMetadata();
    const outline = await doc.getOutline();
    const attachments = await doc.getAttachments();
    const fieldObjects = await doc.getFieldObjects();
    const { length } = await doc.getDownloadInfo();

    const firstPage = await doc.getPage(1);
    const viewport = firstPage.getViewport({ scale: 1.0 });

    return {
      pageCount: doc.numPages,
      title: info.Title || null,
      author: info.Author || null,
      hasOutline: outline !== null && outline.length > 0,
      hasAttachments: attachments !== null && Object.keys(attachments).length > 0,
      hasFormFields: fieldObjects !== null && Object.keys(fieldObjects).length > 0,
      fileSizeBytes: length,
      firstPageDimensions: {
        width: Math.round(viewport.width),
        height: Math.round(viewport.height),
      },
    };
  } finally {
    // ALWAYS destroy — even if metadata extraction fails
    await doc.destroy();
  }
}
```
