# Examples: Memory Management

> pdfjs-dist 5.x -- Complete working code examples for memory-safe PDF viewers.

## Example 1: Document Swap with Full Cleanup

A single-page viewer that correctly handles switching between PDFs.

```typescript
import { GlobalWorkerOptions, getDocument } from 'pdfjs-dist';
import type { PDFDocumentProxy, RenderTask, PageViewport } from 'pdfjs-dist';

GlobalWorkerOptions.workerSrc = new URL(
  'pdfjs-dist/build/pdf.worker.mjs',
  import.meta.url
).toString();

class SinglePageViewer {
  private doc: PDFDocumentProxy | null = null;
  private renderTask: RenderTask | null = null;
  private blobUrl: string | null = null;
  private canvas: HTMLCanvasElement;
  private ctx: CanvasRenderingContext2D;

  constructor(canvas: HTMLCanvasElement) {
    this.canvas = canvas;
    this.ctx = canvas.getContext('2d')!;
  }

  async loadFromUrl(url: string): Promise<void> {
    await this.cleanup();
    this.doc = await getDocument(url).promise;
    await this.renderPage(1);
  }

  async loadFromFile(file: File): Promise<void> {
    await this.cleanup();

    this.blobUrl = URL.createObjectURL(file);
    try {
      this.doc = await getDocument(this.blobUrl).promise;
    } catch (err) {
      URL.revokeObjectURL(this.blobUrl);
      this.blobUrl = null;
      throw err;
    }
    // Revoke immediately -- PDF.js has read the data
    URL.revokeObjectURL(this.blobUrl);
    this.blobUrl = null;

    await this.renderPage(1);
  }

  private async renderPage(pageNum: number): Promise<void> {
    if (!this.doc) return;

    // ALWAYS cancel previous render
    if (this.renderTask) {
      this.renderTask.cancel();
      this.renderTask = null;
    }

    const page = await this.doc.getPage(pageNum);
    const scale = 1.5;
    const viewport = page.getViewport({ scale });
    const dpr = window.devicePixelRatio || 1;

    this.canvas.width = Math.floor(viewport.width * dpr);
    this.canvas.height = Math.floor(viewport.height * dpr);
    this.canvas.style.width = `${Math.floor(viewport.width)}px`;
    this.canvas.style.height = `${Math.floor(viewport.height)}px`;
    this.ctx.scale(dpr, dpr);

    this.renderTask = page.render({
      canvasContext: this.ctx,
      viewport,
    });

    try {
      await this.renderTask.promise;
    } catch (err: any) {
      if (err.name === 'RenderingCancelledException') return;
      throw err;
    } finally {
      this.renderTask = null;
    }
  }

  async cleanup(): Promise<void> {
    // Step 1: Cancel active render
    if (this.renderTask) {
      this.renderTask.cancel();
      this.renderTask = null;
    }

    // Step 2: Clear canvas
    this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
    this.canvas.width = 0;
    this.canvas.height = 0;

    // Step 3: Destroy document (releases worker)
    if (this.doc) {
      await this.doc.destroy();
      this.doc = null;
    }

    // Step 4: Revoke blob URL if still held
    if (this.blobUrl) {
      URL.revokeObjectURL(this.blobUrl);
      this.blobUrl = null;
    }
  }

  async destroy(): Promise<void> {
    await this.cleanup();
    this.canvas.remove();
  }
}
```

---

## Example 2: Multi-Page Viewer with Page Pool

A scrollable viewer that renders only visible pages and evicts off-screen pages.

```typescript
import { GlobalWorkerOptions, getDocument } from 'pdfjs-dist';
import type { PDFDocumentProxy, PDFPageProxy, RenderTask } from 'pdfjs-dist';

const MAX_RENDERED_PAGES = 7;

interface PageEntry {
  pageNum: number;
  page: PDFPageProxy;
  renderTask: RenderTask | null;
  canvas: HTMLCanvasElement;
}

class PooledPdfViewer {
  private doc: PDFDocumentProxy | null = null;
  private renderedPages = new Map<number, PageEntry>();
  private observer: IntersectionObserver;
  private container: HTMLElement;
  private scale = 1.5;

  constructor(container: HTMLElement) {
    this.container = container;

    this.observer = new IntersectionObserver(
      (entries) => this.handleIntersection(entries),
      { root: container, rootMargin: '300px' }
    );
  }

  async load(url: string): Promise<void> {
    await this.destroy();

    this.doc = await getDocument(url).promise;

    // Create placeholder divs for all pages
    for (let i = 1; i <= this.doc.numPages; i++) {
      const placeholder = document.createElement('div');
      placeholder.className = 'page-placeholder';
      placeholder.dataset.pageNum = String(i);
      // Set approximate height so scrollbar is accurate
      placeholder.style.height = '1100px';
      placeholder.style.width = '850px';
      this.container.appendChild(placeholder);
      this.observer.observe(placeholder);
    }
  }

  private handleIntersection(entries: IntersectionObserverEntry[]): void {
    for (const entry of entries) {
      const pageNum = Number(entry.target.dataset.pageNum);
      if (entry.isIntersecting) {
        this.renderPageInView(pageNum, entry.target as HTMLElement);
      } else {
        this.evictPage(pageNum);
      }
    }
  }

  private async renderPageInView(pageNum: number, placeholder: HTMLElement): Promise<void> {
    if (!this.doc || this.renderedPages.has(pageNum)) return;

    // Enforce pool limit before rendering new page
    this.enforcePoolLimit(pageNum);

    const page = await this.doc.getPage(pageNum);
    const viewport = page.getViewport({ scale: this.scale });
    const dpr = window.devicePixelRatio || 1;

    const canvas = document.createElement('canvas');
    canvas.width = Math.floor(viewport.width * dpr);
    canvas.height = Math.floor(viewport.height * dpr);
    canvas.style.width = `${Math.floor(viewport.width)}px`;
    canvas.style.height = `${Math.floor(viewport.height)}px`;

    const ctx = canvas.getContext('2d')!;
    ctx.scale(dpr, dpr);

    const renderTask = page.render({ canvasContext: ctx, viewport });

    const pageEntry: PageEntry = { pageNum, page, renderTask, canvas };
    this.renderedPages.set(pageNum, pageEntry);

    try {
      await renderTask.promise;
      placeholder.innerHTML = '';
      placeholder.style.height = `${Math.floor(viewport.height)}px`;
      placeholder.style.width = `${Math.floor(viewport.width)}px`;
      placeholder.appendChild(canvas);
    } catch (err: any) {
      if (err.name !== 'RenderingCancelledException') {
        console.error(`Failed to render page ${pageNum}:`, err);
      }
      this.renderedPages.delete(pageNum);
    }
  }

  private evictPage(pageNum: number): void {
    const entry = this.renderedPages.get(pageNum);
    if (!entry) return;

    // Cancel active render
    if (entry.renderTask) {
      entry.renderTask.cancel();
    }

    // Release page cache
    entry.page.cleanup();

    // Release canvas GPU memory
    const ctx = entry.canvas.getContext('2d');
    if (ctx) ctx.clearRect(0, 0, entry.canvas.width, entry.canvas.height);
    entry.canvas.width = 0;
    entry.canvas.height = 0;
    entry.canvas.remove();

    this.renderedPages.delete(pageNum);
  }

  private enforcePoolLimit(currentPageNum: number): void {
    if (this.renderedPages.size < MAX_RENDERED_PAGES) return;

    // Evict the page farthest from current view
    let farthestPage = -1;
    let farthestDistance = -1;

    for (const [pageNum] of this.renderedPages) {
      const distance = Math.abs(pageNum - currentPageNum);
      if (distance > farthestDistance) {
        farthestDistance = distance;
        farthestPage = pageNum;
      }
    }

    if (farthestPage !== -1) {
      this.evictPage(farthestPage);
    }
  }

  async destroy(): Promise<void> {
    this.observer.disconnect();

    // Cancel all renders and clean all pages
    for (const [pageNum] of this.renderedPages) {
      this.evictPage(pageNum);
    }
    this.renderedPages.clear();

    if (this.doc) {
      await this.doc.destroy();
      this.doc = null;
    }

    this.container.innerHTML = '';
  }
}
```

---

## Example 3: React Hook with Proper Cleanup

```typescript
import { useEffect, useRef, useState } from 'react';
import { GlobalWorkerOptions, getDocument } from 'pdfjs-dist';
import type { PDFDocumentProxy, RenderTask } from 'pdfjs-dist';

GlobalWorkerOptions.workerSrc = new URL(
  'pdfjs-dist/build/pdf.worker.mjs',
  import.meta.url
).toString();

function usePdfPage(url: string, pageNum: number, scale: number) {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    let doc: PDFDocumentProxy | null = null;
    let renderTask: RenderTask | null = null;
    let cancelled = false;

    async function render() {
      try {
        setLoading(true);
        setError(null);

        doc = await getDocument(url).promise;
        if (cancelled) { await doc.destroy(); return; }

        const page = await doc.getPage(pageNum);
        if (cancelled) { await doc.destroy(); return; }

        const viewport = page.getViewport({ scale });
        const canvas = canvasRef.current;
        if (!canvas || cancelled) { await doc.destroy(); return; }

        const dpr = window.devicePixelRatio || 1;
        canvas.width = Math.floor(viewport.width * dpr);
        canvas.height = Math.floor(viewport.height * dpr);
        canvas.style.width = `${Math.floor(viewport.width)}px`;
        canvas.style.height = `${Math.floor(viewport.height)}px`;

        const ctx = canvas.getContext('2d')!;
        ctx.scale(dpr, dpr);

        renderTask = page.render({ canvasContext: ctx, viewport });
        await renderTask.promise;
        setLoading(false);
      } catch (err: any) {
        if (err.name === 'RenderingCancelledException' || cancelled) return;
        setError(err);
        setLoading(false);
      }
    }

    render();

    // Cleanup on dependency change or unmount
    return () => {
      cancelled = true;
      if (renderTask) renderTask.cancel();
      if (doc) doc.destroy();
    };
  }, [url, pageNum, scale]);

  return { canvasRef, loading, error };
}
```

---

## Example 4: Event Listener Cleanup Pattern

```typescript
class PdfViewerWithEvents {
  private doc: PDFDocumentProxy | null = null;
  private resizeHandler: (() => void) | null = null;
  private keyHandler: ((e: KeyboardEvent) => void) | null = null;

  async init(url: string): Promise<void> {
    this.doc = await getDocument(url).promise;

    // Store bound handlers so they can be removed
    this.resizeHandler = () => this.handleResize();
    this.keyHandler = (e) => this.handleKeydown(e);

    window.addEventListener('resize', this.resizeHandler);
    document.addEventListener('keydown', this.keyHandler);
  }

  private handleResize(): void {
    // Re-render at new size
  }

  private handleKeydown(e: KeyboardEvent): void {
    // Page navigation
  }

  async destroy(): Promise<void> {
    // ALWAYS remove event listeners on destroy
    if (this.resizeHandler) {
      window.removeEventListener('resize', this.resizeHandler);
      this.resizeHandler = null;
    }
    if (this.keyHandler) {
      document.removeEventListener('keydown', this.keyHandler);
      this.keyHandler = null;
    }

    if (this.doc) {
      await this.doc.destroy();
      this.doc = null;
    }
  }
}
```

---

## Example 5: AbortController Pattern for Loading

Use `AbortController` to cancel long-running document loads:

```typescript
class CancellableLoader {
  private currentController: AbortController | null = null;
  private doc: PDFDocumentProxy | null = null;

  async load(url: string): Promise<PDFDocumentProxy> {
    // Cancel previous load if still in progress
    if (this.currentController) {
      this.currentController.abort();
    }
    if (this.doc) {
      await this.doc.destroy();
      this.doc = null;
    }

    this.currentController = new AbortController();
    const loadingTask = getDocument(url);

    // If aborted, destroy the loading task
    this.currentController.signal.addEventListener('abort', () => {
      loadingTask.destroy();
    });

    this.doc = await loadingTask.promise;
    this.currentController = null;
    return this.doc;
  }

  async destroy(): Promise<void> {
    if (this.currentController) {
      this.currentController.abort();
      this.currentController = null;
    }
    if (this.doc) {
      await this.doc.destroy();
      this.doc = null;
    }
  }
}
```
