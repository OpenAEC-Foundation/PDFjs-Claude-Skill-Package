# Code Examples: Core Architecture

> pdfjs-dist 5.x -- All examples verified against official API.

## Minimal Setup (Vanilla JavaScript)

The absolute minimum to render a PDF page:

```typescript
import { GlobalWorkerOptions, getDocument } from 'pdfjs-dist';

// Step 1: ALWAYS configure worker first
GlobalWorkerOptions.workerSrc = new URL(
  'pdfjs-dist/build/pdf.worker.mjs',
  import.meta.url
).toString();

// Step 2: Load document
const loadingTask = getDocument({ url: '/document.pdf' });
const pdfDoc = await loadingTask.promise;

// Step 3: Get page (1-based)
const page = await pdfDoc.getPage(1);

// Step 4: Create viewport
const scale = 1.5;
const viewport = page.getViewport({ scale });

// Step 5: Setup canvas with HiDPI support
const canvas = document.getElementById('pdf-canvas') as HTMLCanvasElement;
const context = canvas.getContext('2d')!;
const dpr = window.devicePixelRatio || 1;

canvas.width = Math.floor(viewport.width * dpr);
canvas.height = Math.floor(viewport.height * dpr);
canvas.style.width = `${Math.floor(viewport.width)}px`;
canvas.style.height = `${Math.floor(viewport.height)}px`;
context.scale(dpr, dpr);

// Step 6: Render
const renderTask = page.render({ canvasContext: context, viewport });
await renderTask.promise;

// Step 7: ALWAYS cleanup when done
pdfDoc.destroy();
```

---

## Full Page with TextLayer and AnnotationLayer

Complete setup for a page with selectable text and clickable links:

```typescript
import {
  GlobalWorkerOptions,
  getDocument,
  TextLayer,
  AnnotationLayer,
} from 'pdfjs-dist';

// Import the required CSS for text and annotation layers
import 'pdfjs-dist/web/pdf_viewer.css';

GlobalWorkerOptions.workerSrc = new URL(
  'pdfjs-dist/build/pdf.worker.mjs',
  import.meta.url
).toString();

async function renderFullPage(
  url: string,
  container: HTMLDivElement,
  pageNum: number,
  scale: number = 1.5
) {
  const pdfDoc = await getDocument({ url }).promise;
  const page = await pdfDoc.getPage(pageNum);
  const viewport = page.getViewport({ scale });
  const dpr = window.devicePixelRatio || 1;

  // Create page container
  const pageDiv = document.createElement('div');
  pageDiv.style.position = 'relative';
  pageDiv.style.width = `${Math.floor(viewport.width)}px`;
  pageDiv.style.height = `${Math.floor(viewport.height)}px`;
  container.appendChild(pageDiv);

  // Canvas layer (z-index: 0)
  const canvas = document.createElement('canvas');
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.width = '100%';
  canvas.style.height = '100%';
  canvas.style.position = 'absolute';
  canvas.style.top = '0';
  canvas.style.left = '0';
  pageDiv.appendChild(canvas);

  const context = canvas.getContext('2d')!;
  context.scale(dpr, dpr);

  // Render canvas
  await page.render({ canvasContext: context, viewport }).promise;

  // TextLayer (z-index: 1)
  const textLayerDiv = document.createElement('div');
  textLayerDiv.className = 'textLayer';
  textLayerDiv.style.position = 'absolute';
  textLayerDiv.style.top = '0';
  textLayerDiv.style.left = '0';
  textLayerDiv.style.width = `${Math.floor(viewport.width)}px`;
  textLayerDiv.style.height = `${Math.floor(viewport.height)}px`;
  pageDiv.appendChild(textLayerDiv);

  const textLayer = new TextLayer({
    textContentSource: page.streamTextContent(),
    container: textLayerDiv,
    viewport,
  });
  await textLayer.render();

  // AnnotationLayer (z-index: 2)
  const annotationLayerDiv = document.createElement('div');
  annotationLayerDiv.className = 'annotationLayer';
  annotationLayerDiv.style.position = 'absolute';
  annotationLayerDiv.style.top = '0';
  annotationLayerDiv.style.left = '0';
  pageDiv.appendChild(annotationLayerDiv);

  const annotations = await page.getAnnotations();
  if (annotations.length > 0) {
    const annotationLayer = new AnnotationLayer({
      div: annotationLayerDiv,
      page,
      viewport: viewport.clone({ dontFlip: true }),
      annotations,
      linkService: {
        // Minimal link service for external links
        getDestinationHash: (dest: any) => '#',
        getAnchorUrl: (hash: string) => hash,
        navigateTo: (dest: any) => {},
        goToDestination: (dest: any) => {},
        addLinkAttributes: (link: HTMLAnchorElement, url: string) => {
          link.href = url;
          link.target = '_blank';
          link.rel = 'noopener noreferrer';
        },
        goToPage: (pageNum: number) => {},
        isPageVisible: (pageNum: number) => true,
        isPageCached: (pageNum: number) => true,
        externalLinkEnabled: true,
      },
    });
    await annotationLayer.render();
  }

  return pdfDoc; // Caller is responsible for calling pdfDoc.destroy()
}
```

---

## Worker Configuration for Vite

```typescript
import { GlobalWorkerOptions } from 'pdfjs-dist';

// Vite handles the URL resolution with import.meta.url
GlobalWorkerOptions.workerSrc = new URL(
  'pdfjs-dist/build/pdf.worker.mjs',
  import.meta.url
).toString();
```

## Worker Configuration for Webpack 5

```typescript
import { GlobalWorkerOptions } from 'pdfjs-dist';
import PdfWorker from 'pdfjs-dist/build/pdf.worker.mjs?worker';

// Webpack 5 with module workers
const worker = new PdfWorker();
GlobalWorkerOptions.workerPort = worker;
```

Alternative using URL:

```typescript
import { GlobalWorkerOptions } from 'pdfjs-dist';

GlobalWorkerOptions.workerSrc = new URL(
  'pdfjs-dist/build/pdf.worker.mjs',
  import.meta.url
).toString();
```

## Worker Configuration for CDN

```html
<script type="module">
  import { GlobalWorkerOptions, getDocument } from
    'https://cdn.jsdelivr.net/npm/pdfjs-dist@5.5.207/build/pdf.mjs';

  GlobalWorkerOptions.workerSrc =
    'https://cdn.jsdelivr.net/npm/pdfjs-dist@5.5.207/build/pdf.worker.mjs';

  const pdfDoc = await getDocument('/my-doc.pdf').promise;
  console.log(`Pages: ${pdfDoc.numPages}`);
</script>
```

---

## Loading with Progress

```typescript
const loadingTask = getDocument({ url: '/large-document.pdf' });

loadingTask.onProgress = ({ loaded, total }) => {
  if (total > 0) {
    const percent = Math.round((loaded / total) * 100);
    progressBar.style.width = `${percent}%`;
    progressBar.textContent = `${percent}%`;
  }
};

const pdfDoc = await loadingTask.promise;
```

---

## Loading from Binary Data

```typescript
// From fetch response
const response = await fetch('/document.pdf');
const arrayBuffer = await response.arrayBuffer();
const pdfDoc = await getDocument({ data: arrayBuffer }).promise;

// From Uint8Array (e.g., from file input)
const fileInput = document.getElementById('file') as HTMLInputElement;
fileInput.addEventListener('change', async (e) => {
  const file = fileInput.files?.[0];
  if (!file) return;

  const arrayBuffer = await file.arrayBuffer();
  const pdfDoc = await getDocument({ data: new Uint8Array(arrayBuffer) }).promise;
  console.log(`Loaded ${pdfDoc.numPages} pages`);
});
```

---

## Password-Protected PDF

```typescript
const loadingTask = getDocument({ url: '/encrypted.pdf' });

loadingTask.onPassword = (callback, reason) => {
  // reason: 1 = need password, 2 = incorrect password
  const password = prompt(
    reason === 1
      ? 'Enter password for this PDF:'
      : 'Incorrect password. Try again:'
  );
  if (password) {
    callback(password);
  } else {
    loadingTask.destroy();
  }
};

try {
  const pdfDoc = await loadingTask.promise;
  console.log(`Loaded ${pdfDoc.numPages} pages`);
} catch (err) {
  console.error('Failed to load PDF:', err);
}
```

---

## CJK Font Support

```typescript
const pdfDoc = await getDocument({
  url: '/chinese-document.pdf',
  cMapUrl: '/node_modules/pdfjs-dist/cmaps/',
  cMapPacked: true,
  standardFontDataUrl: '/node_modules/pdfjs-dist/standard_fonts/',
}).promise;
```

ALWAYS provide `cMapUrl` and `cMapPacked` when rendering documents that may contain CJK (Chinese, Japanese, Korean) characters. Without CMaps, CJK text renders as blank or garbled.

---

## HiDPI / Retina Canvas Rendering

**ALWAYS** apply this pattern to avoid blurry rendering:

```typescript
function setupHiDPICanvas(
  canvas: HTMLCanvasElement,
  viewport: PageViewport
): CanvasRenderingContext2D {
  const dpr = window.devicePixelRatio || 1;

  // Set actual size in memory (scaled up)
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);

  // Set display size (CSS pixels)
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;

  // Scale context so drawing operations are in CSS pixel space
  const context = canvas.getContext('2d')!;
  context.scale(dpr, dpr);

  return context;
}
```

---

## Lazy Page Rendering (Intersection Observer)

NEVER render all pages upfront. ALWAYS use lazy loading:

```typescript
function createLazyPageRenderer(
  pdfDoc: PDFDocumentProxy,
  container: HTMLDivElement,
  scale: number = 1.5
) {
  const observer = new IntersectionObserver(
    (entries) => {
      for (const entry of entries) {
        if (entry.isIntersecting) {
          const pageNum = Number(entry.target.getAttribute('data-page'));
          renderPage(pdfDoc, entry.target as HTMLDivElement, pageNum, scale);
          observer.unobserve(entry.target);
        }
      }
    },
    { rootMargin: '200px' } // Pre-load pages 200px before they scroll into view
  );

  // Create placeholder divs for all pages
  for (let i = 1; i <= pdfDoc.numPages; i++) {
    const placeholder = document.createElement('div');
    placeholder.setAttribute('data-page', String(i));
    placeholder.style.height = '800px'; // Approximate height
    placeholder.style.marginBottom = '8px';
    container.appendChild(placeholder);
    observer.observe(placeholder);
  }
}

async function renderPage(
  pdfDoc: PDFDocumentProxy,
  container: HTMLDivElement,
  pageNum: number,
  scale: number
) {
  const page = await pdfDoc.getPage(pageNum);
  const viewport = page.getViewport({ scale });

  // Update container to actual page size
  container.style.height = `${Math.floor(viewport.height)}px`;
  container.style.width = `${Math.floor(viewport.width)}px`;

  const canvas = document.createElement('canvas');
  const context = canvas.getContext('2d')!;
  const dpr = window.devicePixelRatio || 1;

  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.width = '100%';
  canvas.style.height = '100%';
  context.scale(dpr, dpr);

  container.innerHTML = '';
  container.appendChild(canvas);

  await page.render({ canvasContext: context, viewport }).promise;
}
```

---

## Document Metadata

```typescript
const { info, metadata } = await pdfDoc.getMetadata();

console.log('Title:', info.Title);
console.log('Author:', info.Author);
console.log('Subject:', info.Subject);
console.log('Creator:', info.Creator);
console.log('Producer:', info.Producer);
console.log('Creation Date:', info.CreationDate);
console.log('Modification Date:', info.ModDate);
console.log('PDF Version:', info.PDFFormatVersion);
console.log('Page Count:', pdfDoc.numPages);

// XMP metadata (if available)
if (metadata) {
  console.log('XMP Metadata:', metadata.getAll());
}
```

---

## Document Outline (Bookmarks)

```typescript
const outline = await pdfDoc.getOutline();

if (outline) {
  function renderOutline(items: any[], level: number = 0) {
    for (const item of items) {
      console.log(`${'  '.repeat(level)}${item.title}`);
      if (item.items && item.items.length > 0) {
        renderOutline(item.items, level + 1);
      }
    }
  }
  renderOutline(outline);
}
```
