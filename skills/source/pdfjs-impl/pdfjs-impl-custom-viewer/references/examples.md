# Complete Custom Viewer Examples (pdfjs-dist 5.x)

## 1. Complete PDF Viewer with All Features

A production-ready viewer with navigation, zoom, search, thumbnails, and print.

```typescript
import { getDocument, GlobalWorkerOptions, TextLayer } from "pdfjs-dist";
import type {
  PDFDocumentProxy,
  PDFPageProxy,
  RenderTask,
  TextContent,
} from "pdfjs-dist";

// ALWAYS configure worker BEFORE any document loading
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

// --- State ---

interface ViewerState {
  doc: PDFDocumentProxy;
  numPages: number;
  currentPage: number;
  scale: number;
}

const activeRenders = new Map<number, RenderTask>();
const renderedPages = new Set<number>();
let observer: IntersectionObserver | null = null;

// --- Initialization ---

async function createViewer(url: string): Promise<ViewerState> {
  const doc = await getDocument({ url }).promise;
  const numPages = doc.numPages;

  const container = document.getElementById("page-container") as HTMLDivElement;
  const firstPage = await doc.getPage(1);
  const baseViewport = firstPage.getViewport({ scale: 1.0 });
  const scale = container.clientWidth / baseViewport.width;

  const state: ViewerState = { doc, numPages, currentPage: 1, scale };

  // Create page slots
  for (let i = 1; i <= numPages; i++) {
    const page = await doc.getPage(i);
    const viewport = page.getViewport({ scale });

    const slot = document.createElement("div");
    slot.className = "page-slot";
    slot.dataset.pageNumber = String(i);
    slot.style.width = `${Math.floor(viewport.width)}px`;
    slot.style.height = `${Math.floor(viewport.height)}px`;
    slot.style.position = "relative";
    slot.style.marginBottom = "8px";
    slot.style.backgroundColor = "#e0e0e0";
    slot.style.willChange = "transform";
    container.appendChild(slot);
  }

  // Setup IntersectionObserver for lazy rendering
  observer = new IntersectionObserver(
    (entries) => {
      for (const entry of entries) {
        const pageNum = Number((entry.target as HTMLElement).dataset.pageNumber);
        if (entry.isIntersecting && !renderedPages.has(pageNum)) {
          renderPageWithLayers(state, entry.target as HTMLDivElement, pageNum);
        } else if (!entry.isIntersecting) {
          cleanupRenderedPage(pageNum, entry.target as HTMLDivElement);
        }
      }
    },
    { root: container, rootMargin: "200px 0px" }
  );

  container.querySelectorAll(".page-slot").forEach((el) => observer!.observe(el));

  // Setup UI controls
  setupNavigationControls(state, container);
  setupZoomUI(state, container);
  setupScrollDetection(state, container);

  document.getElementById("page-count")!.textContent = String(numPages);

  return state;
}

// --- Page Rendering ---

async function renderPageWithLayers(
  state: ViewerState,
  slot: HTMLDivElement,
  pageNum: number
): Promise<void> {
  if (renderedPages.has(pageNum)) return;
  renderedPages.add(pageNum);

  const page = await state.doc.getPage(pageNum);
  const viewport = page.getViewport({ scale: state.scale });
  const dpr = window.devicePixelRatio || 1;

  // Canvas layer
  const canvas = document.createElement("canvas");
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;
  canvas.style.position = "absolute";
  canvas.style.top = "0";
  canvas.style.left = "0";
  slot.appendChild(canvas);

  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);

  const renderTask = page.render({ canvasContext: ctx, viewport });
  activeRenders.set(pageNum, renderTask);

  try {
    await renderTask.promise;
  } catch (err: unknown) {
    if (err instanceof Error && err.message === "Rendering cancelled") {
      canvas.remove();
      renderedPages.delete(pageNum);
      return;
    }
    throw err;
  } finally {
    activeRenders.delete(pageNum);
  }

  // Text layer (enables selection and search highlighting)
  const textDiv = document.createElement("div");
  textDiv.className = "textLayer";
  textDiv.style.position = "absolute";
  textDiv.style.top = "0";
  textDiv.style.left = "0";
  textDiv.style.width = `${Math.floor(viewport.width)}px`;
  textDiv.style.height = `${Math.floor(viewport.height)}px`;
  slot.appendChild(textDiv);

  const textContent = await page.getTextContent();
  const textLayer = new TextLayer({
    container: textDiv,
    textContentSource: textContent,
    viewport: viewport,
  });
  await textLayer.render();
}

// --- Page Cleanup ---

function cleanupRenderedPage(pageNum: number, slot: HTMLDivElement): void {
  const task = activeRenders.get(pageNum);
  if (task) {
    task.cancel();
    activeRenders.delete(pageNum);
  }

  const canvas = slot.querySelector("canvas");
  if (canvas) {
    const ctx = canvas.getContext("2d");
    if (ctx) ctx.clearRect(0, 0, canvas.width, canvas.height);
    canvas.width = 0;
    canvas.height = 0;
    canvas.remove();
  }

  slot.querySelectorAll(".textLayer, .annotationLayer").forEach((el) => el.remove());
  renderedPages.delete(pageNum);
}

// --- Navigation ---

function setupNavigationControls(state: ViewerState, container: HTMLElement): void {
  document.getElementById("prev-page")!.addEventListener("click", () => {
    navigateToPage(state, container, state.currentPage - 1);
  });

  document.getElementById("next-page")!.addEventListener("click", () => {
    navigateToPage(state, container, state.currentPage + 1);
  });

  const pageInput = document.getElementById("page-input") as HTMLInputElement;
  pageInput.addEventListener("change", () => {
    navigateToPage(state, container, Number(pageInput.value));
  });

  // Keyboard navigation
  document.addEventListener("keydown", (e) => {
    if (e.target instanceof HTMLInputElement) return; // Skip when typing in inputs
    if (e.key === "ArrowLeft" || e.key === "PageUp") {
      navigateToPage(state, container, state.currentPage - 1);
    } else if (e.key === "ArrowRight" || e.key === "PageDown") {
      navigateToPage(state, container, state.currentPage + 1);
    } else if (e.key === "Home") {
      navigateToPage(state, container, 1);
    } else if (e.key === "End") {
      navigateToPage(state, container, state.numPages);
    }
  });
}

function navigateToPage(
  state: ViewerState,
  container: HTMLElement,
  pageNum: number
): void {
  const clamped = Math.max(1, Math.min(pageNum, state.numPages));
  state.currentPage = clamped;
  (document.getElementById("page-input") as HTMLInputElement).value = String(clamped);

  const slot = container.querySelector(`[data-page-number="${clamped}"]`);
  if (slot) {
    slot.scrollIntoView({ behavior: "smooth", block: "start" });
  }
}

// --- Scroll-based Page Detection ---

function setupScrollDetection(state: ViewerState, container: HTMLElement): void {
  let debounceTimer: ReturnType<typeof setTimeout>;

  container.addEventListener("scroll", () => {
    clearTimeout(debounceTimer);
    debounceTimer = setTimeout(() => {
      const centerY = container.getBoundingClientRect().top + container.clientHeight / 2;
      const slots = container.querySelectorAll(".page-slot");

      for (const slot of slots) {
        const rect = slot.getBoundingClientRect();
        if (rect.top <= centerY && rect.bottom >= centerY) {
          state.currentPage = Number((slot as HTMLElement).dataset.pageNumber);
          (document.getElementById("page-input") as HTMLInputElement).value =
            String(state.currentPage);
          break;
        }
      }
    }, 50);
  });
}

// --- Zoom ---

function setupZoomUI(state: ViewerState, container: HTMLElement): void {
  document.getElementById("zoom-in")!.addEventListener("click", () => {
    applyZoom(state, container, state.scale * 1.25);
  });

  document.getElementById("zoom-out")!.addEventListener("click", () => {
    applyZoom(state, container, state.scale / 1.25);
  });

  const select = document.getElementById("zoom-select") as HTMLSelectElement;
  select.addEventListener("change", () => {
    const val = select.value;
    if (val === "fit-width") {
      state.doc.getPage(1).then((p) => {
        const bv = p.getViewport({ scale: 1.0 });
        applyZoom(state, container, container.clientWidth / bv.width);
      });
    } else if (val === "fit-page") {
      state.doc.getPage(1).then((p) => {
        const bv = p.getViewport({ scale: 1.0 });
        const sw = container.clientWidth / bv.width;
        const sh = container.clientHeight / bv.height;
        applyZoom(state, container, Math.min(sw, sh));
      });
    } else {
      applyZoom(state, container, Number(val));
    }
  });

  // Ctrl+scroll for zoom
  container.addEventListener("wheel", (e) => {
    if (e.ctrlKey) {
      e.preventDefault();
      const factor = e.deltaY < 0 ? 1.1 : 0.9;
      applyZoom(state, container, state.scale * factor);
    }
  }, { passive: false });
}

function applyZoom(
  state: ViewerState,
  container: HTMLElement,
  newScale: number
): void {
  const currentPage = state.currentPage;
  state.scale = Math.max(0.25, Math.min(5.0, newScale));

  // Resize all slots and clear renders
  const slots = container.querySelectorAll(".page-slot");
  const resizePromises: Promise<void>[] = [];

  slots.forEach((slot, index) => {
    const pageNum = index + 1;
    cleanupRenderedPage(pageNum, slot as HTMLDivElement);

    resizePromises.push(
      state.doc.getPage(pageNum).then((page) => {
        const vp = page.getViewport({ scale: state.scale });
        (slot as HTMLElement).style.width = `${Math.floor(vp.width)}px`;
        (slot as HTMLElement).style.height = `${Math.floor(vp.height)}px`;
      })
    );
  });

  Promise.all(resizePromises).then(() => {
    // Scroll back to current page after resize
    navigateToPage(state, container, currentPage);
  });
}
```

---

## 2. Text Search Implementation

Search across all pages with match highlighting and navigation between results.

```typescript
interface SearchResult {
  pageNum: number;
  matchIndex: number; // Index within page's text items
  text: string;       // The matched text
}

interface SearchState {
  query: string;
  results: SearchResult[];
  currentResultIndex: number;
  textCache: Map<number, TextContent>; // Cache text content per page
}

const searchState: SearchState = {
  query: "",
  results: [],
  currentResultIndex: -1,
  textCache: new Map(),
};

async function searchAllPages(
  doc: PDFDocumentProxy,
  query: string
): Promise<SearchResult[]> {
  if (!query.trim()) return [];

  const results: SearchResult[] = [];
  const lowerQuery = query.toLowerCase();

  for (let i = 1; i <= doc.numPages; i++) {
    const page = await doc.getPage(i);

    // Cache text content -- extraction is expensive
    let textContent = searchState.textCache.get(i);
    if (!textContent) {
      textContent = await page.getTextContent();
      searchState.textCache.set(i, textContent);
    }

    // Search through text items
    textContent.items.forEach((item, idx) => {
      if ("str" in item && item.str.toLowerCase().includes(lowerQuery)) {
        results.push({
          pageNum: i,
          matchIndex: idx,
          text: item.str,
        });
      }
    });
  }

  return results;
}

function setupSearchUI(state: ViewerState, container: HTMLElement): void {
  const searchInput = document.getElementById("search-input") as HTMLInputElement;
  const searchCount = document.getElementById("search-count")!;
  let searchTimer: ReturnType<typeof setTimeout>;

  // Debounced search on input
  searchInput.addEventListener("input", () => {
    clearTimeout(searchTimer);
    searchTimer = setTimeout(async () => {
      const query = searchInput.value;
      clearHighlights(container);

      if (!query.trim()) {
        searchState.results = [];
        searchState.currentResultIndex = -1;
        searchCount.textContent = "";
        return;
      }

      searchState.query = query;
      searchState.results = await searchAllPages(state.doc, query);
      searchState.currentResultIndex = searchState.results.length > 0 ? 0 : -1;
      searchCount.textContent = `${searchState.results.length} matches`;

      if (searchState.results.length > 0) {
        highlightAndNavigate(state, container, 0);
      }
    }, 300); // 300ms debounce
  });

  // Navigate between results
  document.getElementById("search-next")!.addEventListener("click", () => {
    if (searchState.results.length === 0) return;
    searchState.currentResultIndex =
      (searchState.currentResultIndex + 1) % searchState.results.length;
    highlightAndNavigate(state, container, searchState.currentResultIndex);
    updateSearchCounter(searchCount);
  });

  document.getElementById("search-prev")!.addEventListener("click", () => {
    if (searchState.results.length === 0) return;
    searchState.currentResultIndex =
      (searchState.currentResultIndex - 1 + searchState.results.length) %
      searchState.results.length;
    highlightAndNavigate(state, container, searchState.currentResultIndex);
    updateSearchCounter(searchCount);
  });
}

function highlightAndNavigate(
  state: ViewerState,
  container: HTMLElement,
  resultIndex: number
): void {
  clearHighlights(container);
  const result = searchState.results[resultIndex];
  if (!result) return;

  // Navigate to the page containing the match
  navigateToPage(state, container, result.pageNum);

  // Highlight matches in the text layer after rendering
  requestAnimationFrame(() => {
    const slot = container.querySelector(
      `[data-page-number="${result.pageNum}"]`
    );
    if (!slot) return;

    const textLayer = slot.querySelector(".textLayer");
    if (!textLayer) return;

    const spans = textLayer.querySelectorAll("span");
    const lowerQuery = searchState.query.toLowerCase();

    spans.forEach((span) => {
      if (span.textContent?.toLowerCase().includes(lowerQuery)) {
        // Wrap matches in highlight spans
        const text = span.textContent;
        const regex = new RegExp(`(${escapeRegex(searchState.query)})`, "gi");
        span.innerHTML = text.replace(
          regex,
          '<mark class="search-highlight">$1</mark>'
        );
      }
    });

    // Scroll the current match into view
    const firstHighlight = slot.querySelector(".search-highlight");
    if (firstHighlight) {
      firstHighlight.scrollIntoView({ behavior: "smooth", block: "center" });
    }
  });
}

function clearHighlights(container: HTMLElement): void {
  container.querySelectorAll(".search-highlight").forEach((mark) => {
    const parent = mark.parentNode;
    if (parent) {
      parent.replaceChild(document.createTextNode(mark.textContent || ""), mark);
      parent.normalize(); // Merge adjacent text nodes
    }
  });
}

function updateSearchCounter(el: HTMLElement): void {
  el.textContent = `${searchState.currentResultIndex + 1} of ${searchState.results.length}`;
}

function escapeRegex(str: string): string {
  return str.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
}
```

### Search Highlight CSS

```css
.search-highlight {
  background-color: #ffff00;
  color: #000;
  border-radius: 2px;
  padding: 0 1px;
}

.search-highlight.active {
  background-color: #ff9632;
}
```

---

## 3. Thumbnail Panel

Sidebar with small page previews that navigate to pages on click.

```typescript
async function createThumbnailPanel(
  state: ViewerState,
  container: HTMLElement
): Promise<void> {
  const thumbContainer = document.getElementById("thumbnail-container")!;
  const thumbScale = 0.2; // Thumbnails at 20% of full size

  for (let i = 1; i <= state.numPages; i++) {
    const page = await state.doc.getPage(i);
    const viewport = page.getViewport({ scale: thumbScale });

    const thumbWrapper = document.createElement("div");
    thumbWrapper.className = "thumbnail-wrapper";
    thumbWrapper.dataset.pageNumber = String(i);

    const canvas = document.createElement("canvas");
    const dpr = window.devicePixelRatio || 1;
    canvas.width = Math.floor(viewport.width * dpr);
    canvas.height = Math.floor(viewport.height * dpr);
    canvas.style.width = `${Math.floor(viewport.width)}px`;
    canvas.style.height = `${Math.floor(viewport.height)}px`;

    const label = document.createElement("span");
    label.className = "thumbnail-label";
    label.textContent = String(i);

    thumbWrapper.appendChild(canvas);
    thumbWrapper.appendChild(label);
    thumbContainer.appendChild(thumbWrapper);

    // Click to navigate
    thumbWrapper.addEventListener("click", () => {
      navigateToPage(state, container, i);
      highlightActiveThumbnail(thumbContainer, i);
    });
  }

  // Lazy-render thumbnails with their own IntersectionObserver
  const thumbObserver = new IntersectionObserver(
    (entries) => {
      for (const entry of entries) {
        if (entry.isIntersecting) {
          const pageNum = Number(
            (entry.target as HTMLElement).dataset.pageNumber
          );
          renderThumbnail(state.doc, entry.target as HTMLElement, pageNum, thumbScale);
          thumbObserver.unobserve(entry.target); // Render once, keep cached
        }
      }
    },
    { root: thumbContainer, rootMargin: "100px 0px" }
  );

  thumbContainer
    .querySelectorAll(".thumbnail-wrapper")
    .forEach((el) => thumbObserver.observe(el));
}

async function renderThumbnail(
  doc: PDFDocumentProxy,
  wrapper: HTMLElement,
  pageNum: number,
  scale: number
): Promise<void> {
  const canvas = wrapper.querySelector("canvas");
  if (!canvas || canvas.dataset.rendered === "true") return;

  const page = await doc.getPage(pageNum);
  const viewport = page.getViewport({ scale });
  const dpr = window.devicePixelRatio || 1;

  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);

  await page.render({ canvasContext: ctx, viewport }).promise;
  canvas.dataset.rendered = "true";
}

function highlightActiveThumbnail(
  thumbContainer: HTMLElement,
  pageNum: number
): void {
  thumbContainer
    .querySelectorAll(".thumbnail-wrapper")
    .forEach((el) => el.classList.remove("active"));

  const active = thumbContainer.querySelector(
    `[data-page-number="${pageNum}"]`
  );
  if (active) active.classList.add("active");
}
```

### Thumbnail CSS

```css
#thumbnail-container {
  width: 150px;
  overflow-y: auto;
  height: 100vh;
  background: #f5f5f5;
  padding: 8px;
}

.thumbnail-wrapper {
  cursor: pointer;
  margin-bottom: 8px;
  border: 2px solid transparent;
  text-align: center;
  transition: border-color 0.2s;
}

.thumbnail-wrapper:hover {
  border-color: #666;
}

.thumbnail-wrapper.active {
  border-color: #0066cc;
}

.thumbnail-label {
  display: block;
  font-size: 12px;
  color: #666;
  margin-top: 2px;
}
```

---

## 4. Print Support

Render all pages at print resolution into a hidden container, then trigger `window.print()`.

```typescript
async function printDocument(state: ViewerState): Promise<void> {
  const printContainer = document.createElement("div");
  printContainer.id = "print-container";
  printContainer.style.display = "none";
  document.body.appendChild(printContainer);

  // Render ALL pages at high resolution for print
  // This is the ONE case where rendering all pages is acceptable
  // ALWAYS use intent: "print" to include print-only annotations
  const printScale = 300 / 72; // 300 DPI (PDF base is 72 DPI)

  for (let i = 1; i <= state.numPages; i++) {
    const page = await state.doc.getPage(i);
    const viewport = page.getViewport({ scale: printScale });

    const canvas = document.createElement("canvas");
    canvas.width = Math.floor(viewport.width);
    canvas.height = Math.floor(viewport.height);
    canvas.className = "print-page";
    printContainer.appendChild(canvas);

    const ctx = canvas.getContext("2d")!;
    await page.render({
      canvasContext: ctx,
      viewport,
      intent: "print", // ALWAYS use "print" intent for print rendering
    }).promise;
  }

  // Trigger browser print dialog
  window.print();

  // Cleanup after printing
  window.addEventListener(
    "afterprint",
    () => {
      // Release all print canvases
      printContainer.querySelectorAll("canvas").forEach((c) => {
        c.width = 0;
        c.height = 0;
      });
      printContainer.remove();
    },
    { once: true }
  );
}

function setupPrintButton(state: ViewerState): void {
  document.getElementById("print-btn")!.addEventListener("click", () => {
    printDocument(state);
  });
}
```

### Print CSS

```css
@media print {
  /* Hide everything except print container */
  body > *:not(#print-container) {
    display: none !important;
  }

  #print-container {
    display: block !important;
  }

  .print-page {
    width: 100%;
    height: auto;
    page-break-after: always;
    display: block;
  }

  .print-page:last-child {
    page-break-after: avoid;
  }
}

@media screen {
  #print-container {
    display: none;
  }
}
```

---

## 5. Responsive Resize Handling

Re-fit the viewer when the browser window is resized.

```typescript
function setupResizeHandler(state: ViewerState, container: HTMLElement): void {
  let resizeTimer: ReturnType<typeof setTimeout>;

  window.addEventListener("resize", () => {
    clearTimeout(resizeTimer);
    resizeTimer = setTimeout(() => {
      // Recalculate fit-to-width scale
      state.doc.getPage(1).then((page) => {
        const baseViewport = page.getViewport({ scale: 1.0 });
        const newScale = container.clientWidth / baseViewport.width;
        applyZoom(state, container, newScale);
      });
    }, 200); // 200ms debounce for resize
  });
}
```

---

## 6. Full Viewer CSS

```css
@import "pdfjs-dist/web/pdf_viewer.css";

* { box-sizing: border-box; margin: 0; padding: 0; }

#pdf-viewer {
  display: flex;
  flex-direction: column;
  height: 100vh;
  font-family: system-ui, -apple-system, sans-serif;
}

#toolbar {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 16px;
  background: #333;
  color: white;
  flex-shrink: 0;
}

#toolbar button {
  padding: 4px 12px;
  border: 1px solid #666;
  background: #444;
  color: white;
  border-radius: 4px;
  cursor: pointer;
}

#toolbar button:hover { background: #555; }

#toolbar input, #toolbar select {
  padding: 4px 8px;
  border: 1px solid #666;
  border-radius: 4px;
}

#page-input { width: 60px; text-align: center; }
#search-input { width: 200px; }

#sidebar {
  position: fixed;
  left: 0;
  top: 48px;
  width: 160px;
  height: calc(100vh - 48px);
  overflow-y: auto;
  background: #f0f0f0;
  border-right: 1px solid #ccc;
  z-index: 10;
}

#page-container {
  flex: 1;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 16px;
  background: #808080;
  margin-left: 160px; /* Account for sidebar */
}

.page-slot {
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
  background: white;
}

.page-slot canvas {
  position: absolute;
  top: 0;
  left: 0;
}

.page-slot .textLayer {
  position: absolute;
  top: 0;
  left: 0;
  z-index: 1;
}

.page-slot .annotationLayer {
  position: absolute;
  top: 0;
  left: 0;
  z-index: 2;
}

@media print {
  #toolbar, #sidebar { display: none !important; }
  #page-container {
    overflow: visible;
    height: auto;
    margin-left: 0;
    background: white;
    padding: 0;
  }
  .page-slot {
    break-after: page;
    box-shadow: none;
    margin: 0;
  }
}
```
