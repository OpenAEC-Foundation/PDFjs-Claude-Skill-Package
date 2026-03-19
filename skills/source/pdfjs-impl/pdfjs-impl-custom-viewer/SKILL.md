---
name: pdfjs-impl-custom-viewer
description: "Builds a complete custom PDF viewer with page navigation, zoom controls, text search, print support, and thumbnails. Covers lazy page loading with IntersectionObserver, virtual scrolling, scroll-based page detection, and memory management. Activates when building a PDF viewer, adding PDF viewing to a web app, implementing page navigation, zoom, search, print, or thumbnail generation."
license: MIT
compatibility: "Designed for Claude Code. Requires pdfjs-dist 5.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# pdfjs-impl-custom-viewer

## Quick Reference

### Viewer Architecture

| Component | Purpose | Key Pattern |
|-----------|---------|-------------|
| Page container | Scrollable wrapper holding all page slots | Single `<div>` with `overflow-y: auto` |
| Page slots | Placeholder divs sized to page dimensions | Created for ALL pages, rendered lazily |
| IntersectionObserver | Triggers render/cleanup as pages enter/leave viewport | `rootMargin: "200px"` for pre-rendering |
| Page renderer | Renders canvas + text + annotation layers | Cancel-then-render with DPI handling |
| Navigation bar | Previous/next, go-to-page, page count | Updates on scroll position change |
| Zoom controls | Scale manipulation with re-render | Cancel all visible renders, re-render at new scale |
| Search engine | Text extraction + match highlighting | `page.getTextContent()` across all pages |
| Thumbnail panel | Small-scale page previews | Render at scale 0.2-0.3, cache as ImageBitmap |
| Print handler | High-resolution render for printing | `intent: "print"`, CSS `@media print` |

### Critical Warnings

**NEVER** render all pages at once -- ALWAYS use IntersectionObserver to render only visible pages plus a buffer. A 500-page PDF at scale 1.5 on a Retina display would consume 18 GB of memory.

**ALWAYS** clean up pages that scroll out of view -- remove canvas elements and nullify references to allow garbage collection. Without cleanup, memory grows linearly as the user scrolls.

**ALWAYS** cancel in-progress RenderTasks before starting new renders -- zoom changes, page navigation, and cleanup all require cancellation first.

**NEVER** call `getPage()` for all pages during initialization -- get page dimensions from the first page or use `doc.getPage()` lazily. Calling `getPage()` for 500 pages blocks the UI.

**ALWAYS** debounce scroll-based current page detection -- the scroll event fires at 60fps; reading `getBoundingClientRect()` for every page on every frame causes layout thrashing.

**ALWAYS** set `will-change: transform` on page containers to promote them to compositor layers and prevent full-page repaints during scrolling.

---

## Essential Patterns

### Viewer HTML Structure

```html
<div id="pdf-viewer">
  <div id="toolbar">
    <button id="prev-page">Previous</button>
    <input id="page-input" type="number" min="1" /> / <span id="page-count"></span>
    <button id="next-page">Next</button>
    <button id="zoom-in">+</button>
    <button id="zoom-out">-</button>
    <select id="zoom-select">
      <option value="fit-width">Fit Width</option>
      <option value="fit-page">Fit Page</option>
      <option value="0.5">50%</option>
      <option value="1.0">100%</option>
      <option value="1.5">150%</option>
      <option value="2.0">200%</option>
    </select>
    <input id="search-input" type="text" placeholder="Search..." />
    <button id="search-prev">Prev Match</button>
    <button id="search-next">Next Match</button>
    <span id="search-count"></span>
    <button id="print-btn">Print</button>
  </div>
  <div id="sidebar">
    <div id="thumbnail-container"></div>
  </div>
  <div id="page-container"></div>
</div>
```

### Viewer Initialization

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";
import type { PDFDocumentProxy } from "pdfjs-dist";

// ALWAYS configure worker BEFORE loading documents
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

interface ViewerState {
  doc: PDFDocumentProxy;
  currentPage: number;
  scale: number;
  numPages: number;
  pageHeights: number[];  // Cache page heights for scroll calculations
}

async function initViewer(url: string): Promise<ViewerState> {
  const doc = await getDocument({ url }).promise;
  const numPages = doc.numPages;

  // Get first page dimensions for initial scale calculation
  const firstPage = await doc.getPage(1);
  const container = document.getElementById("page-container")!;
  const baseViewport = firstPage.getViewport({ scale: 1.0 });
  const scale = container.clientWidth / baseViewport.width; // Fit-to-width

  // Create page slots for ALL pages (lightweight placeholders)
  const pageHeights: number[] = [];
  for (let i = 1; i <= numPages; i++) {
    const page = await doc.getPage(i);
    const viewport = page.getViewport({ scale });
    pageHeights.push(viewport.height);

    const slot = document.createElement("div");
    slot.className = "page-slot";
    slot.dataset.pageNumber = String(i);
    slot.style.width = `${Math.floor(viewport.width)}px`;
    slot.style.height = `${Math.floor(viewport.height)}px`;
    slot.style.position = "relative";
    slot.style.marginBottom = "8px";
    slot.style.backgroundColor = "#e0e0e0";
    slot.style.willChange = "transform"; // GPU layer promotion
    container.appendChild(slot);
  }

  const state: ViewerState = { doc, currentPage: 1, scale, numPages, pageHeights };

  setupLazyRendering(state, container);
  setupNavigation(state, container);
  setupZoomControls(state, container);

  document.getElementById("page-count")!.textContent = String(numPages);
  return state;
}
```

### Lazy Rendering with IntersectionObserver

```typescript
import type { PDFDocumentProxy, RenderTask } from "pdfjs-dist";

const activeRenders = new Map<number, RenderTask>();
const renderedPages = new Set<number>();

function setupLazyRendering(state: ViewerState, container: HTMLElement): void {
  const observer = new IntersectionObserver(
    (entries) => {
      for (const entry of entries) {
        const pageNum = Number((entry.target as HTMLElement).dataset.pageNumber);

        if (entry.isIntersecting) {
          if (!renderedPages.has(pageNum)) {
            renderPage(state, entry.target as HTMLDivElement, pageNum);
          }
        } else {
          // ALWAYS clean up pages that leave the viewport
          cleanupPage(pageNum, entry.target as HTMLDivElement);
        }
      }
    },
    {
      root: container,
      rootMargin: "200px 0px", // Pre-render 200px above and below viewport
    }
  );

  container.querySelectorAll(".page-slot").forEach((el) => observer.observe(el));
}
```

### Current Page Detection from Scroll

```typescript
function setupScrollPageDetection(
  state: ViewerState,
  container: HTMLElement
): void {
  let debounceTimer: ReturnType<typeof setTimeout>;

  container.addEventListener("scroll", () => {
    clearTimeout(debounceTimer);
    debounceTimer = setTimeout(() => {
      const containerRect = container.getBoundingClientRect();
      const centerY = containerRect.top + containerRect.height / 2;
      const slots = container.querySelectorAll(".page-slot");

      for (const slot of slots) {
        const rect = slot.getBoundingClientRect();
        if (rect.top <= centerY && rect.bottom >= centerY) {
          state.currentPage = Number((slot as HTMLElement).dataset.pageNumber);
          updatePageIndicator(state);
          break;
        }
      }
    }, 50); // 50ms debounce
  });
}

function updatePageIndicator(state: ViewerState): void {
  const input = document.getElementById("page-input") as HTMLInputElement;
  input.value = String(state.currentPage);
}
```

---

## Common Operations

### Page Navigation

```typescript
function goToPage(state: ViewerState, container: HTMLElement, pageNum: number): void {
  const clamped = Math.max(1, Math.min(pageNum, state.numPages));
  const slot = container.querySelector(`[data-page-number="${clamped}"]`);
  if (slot) {
    slot.scrollIntoView({ behavior: "smooth", block: "start" });
    state.currentPage = clamped;
    updatePageIndicator(state);
  }
}

function setupNavigation(state: ViewerState, container: HTMLElement): void {
  document.getElementById("prev-page")!.addEventListener("click", () => {
    goToPage(state, container, state.currentPage - 1);
  });
  document.getElementById("next-page")!.addEventListener("click", () => {
    goToPage(state, container, state.currentPage + 1);
  });
  (document.getElementById("page-input") as HTMLInputElement)
    .addEventListener("change", (e) => {
      goToPage(state, container, Number((e.target as HTMLInputElement).value));
    });

  setupScrollPageDetection(state, container);
}
```

### Zoom Controls

```typescript
function setupZoomControls(state: ViewerState, container: HTMLElement): void {
  document.getElementById("zoom-in")!.addEventListener("click", () => {
    setScale(state, container, state.scale * 1.25);
  });
  document.getElementById("zoom-out")!.addEventListener("click", () => {
    setScale(state, container, state.scale / 1.25);
  });
  document.getElementById("zoom-select")!.addEventListener("change", (e) => {
    const value = (e.target as HTMLSelectElement).value;
    if (value === "fit-width") {
      fitToWidth(state, container);
    } else if (value === "fit-page") {
      fitToPage(state, container);
    } else {
      setScale(state, container, Number(value));
    }
  });
}

function setScale(state: ViewerState, container: HTMLElement, newScale: number): void {
  state.scale = Math.max(0.25, Math.min(5.0, newScale));

  // Resize ALL page slots, then re-render only visible ones
  const slots = container.querySelectorAll(".page-slot");
  slots.forEach(async (slot, index) => {
    const page = await state.doc.getPage(index + 1);
    const viewport = page.getViewport({ scale: state.scale });
    (slot as HTMLElement).style.width = `${Math.floor(viewport.width)}px`;
    (slot as HTMLElement).style.height = `${Math.floor(viewport.height)}px`;

    // Clear rendered content -- IntersectionObserver will re-render visible pages
    cleanupPage(index + 1, slot as HTMLDivElement);
    renderedPages.delete(index + 1);
  });
}

function fitToWidth(state: ViewerState, container: HTMLElement): void {
  state.doc.getPage(1).then((page) => {
    const baseViewport = page.getViewport({ scale: 1.0 });
    const newScale = container.clientWidth / baseViewport.width;
    setScale(state, container, newScale);
  });
}

function fitToPage(state: ViewerState, container: HTMLElement): void {
  state.doc.getPage(1).then((page) => {
    const baseViewport = page.getViewport({ scale: 1.0 });
    const scaleW = container.clientWidth / baseViewport.width;
    const scaleH = container.clientHeight / baseViewport.height;
    setScale(state, container, Math.min(scaleW, scaleH));
  });
}
```

### Memory Cleanup

```typescript
function cleanupPage(pageNum: number, slot: HTMLDivElement): void {
  // Cancel any in-progress render
  const task = activeRenders.get(pageNum);
  if (task) {
    task.cancel();
    activeRenders.delete(pageNum);
  }

  // Remove canvas and layer elements to free memory
  const canvas = slot.querySelector("canvas");
  if (canvas) {
    const ctx = canvas.getContext("2d");
    if (ctx) {
      ctx.clearRect(0, 0, canvas.width, canvas.height);
    }
    canvas.width = 0;  // Release GPU memory
    canvas.height = 0;
    canvas.remove();
  }

  // Remove text and annotation layers
  slot.querySelectorAll(".textLayer, .annotationLayer").forEach((el) => el.remove());
  renderedPages.delete(pageNum);
}
```

---

## Decision Tree: Viewer Feature Selection

```
Building a custom PDF viewer?
├── Need scrollable multi-page view?
│   ├── YES → Create page slots for ALL pages, use IntersectionObserver
│   │         NEVER render all pages at once
│   └── NO (single page) → Simple prev/next with single canvas
│
├── Need zoom?
│   ├── YES → Resize all slots, clear rendered pages, let observer re-render
│   └── NO → Use fit-to-width as fixed scale
│
├── Need search?
│   ├── YES → Extract text from ALL pages (lazy), highlight matches
│   └── NO → Skip text layer if not needed for selection either
│
├── Need thumbnails?
│   ├── YES → Render at scale 0.2-0.3, cache results, lazy-load
│   └── NO → Skip sidebar
│
├── Need print?
│   ├── YES → Hidden print container, render at 300 DPI, CSS @media print
│   └── NO → Skip print button
│
└── Memory-constrained environment?
    ├── YES → Aggressive cleanup (remove pages 1 screen away from viewport)
    └── NO → Keep buffer of 2-3 pages above/below (rootMargin: "600px")
```

---

## Required CSS

```css
@import "pdfjs-dist/web/pdf_viewer.css";

#page-container {
  overflow-y: auto;
  height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  background: #808080;
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
  #toolbar, #sidebar { display: none; }
  #page-container { overflow: visible; height: auto; }
  .page-slot { break-after: page; box-shadow: none; margin: 0; }
}
```

---

## Reference Links

- [references/methods.md](references/methods.md) -- Viewer patterns, IntersectionObserver API usage, page render/cleanup lifecycle
- [references/examples.md](references/examples.md) -- Complete viewer with navigation, zoom, search, print, and thumbnails
- [references/anti-patterns.md](references/anti-patterns.md) -- Render all pages, no lazy loading, no cleanup, and other mistakes

### Official Sources

- https://mozilla.github.io/pdf.js/api/ -- PDF.js API reference
- https://github.com/mozilla/pdf.js/tree/master/web -- Reference viewer implementation
- https://github.com/mozilla/pdf.js/tree/master/examples -- Official examples
- https://github.com/mozilla/pdf.js -- Source code and types
