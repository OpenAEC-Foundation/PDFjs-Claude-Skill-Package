# Anti-Patterns (pdfjs-dist 5.x Text Layer)

## AP-01: Using Deprecated renderTextLayer() Function

### Wrong

```typescript
// NEVER use the deprecated renderTextLayer function -- removed in pdfjs-dist 5.x
import { renderTextLayer } from "pdfjs-dist";

renderTextLayer({
  textContent: textContent,
  container: textLayerDiv,
  viewport: viewport,
  textDivs: [],
});
```

### Correct

```typescript
// ALWAYS use the TextLayer class
import { TextLayer } from "pdfjs-dist";

const textLayer = new TextLayer({
  textContentSource: textContent,
  container: textLayerDiv,
  viewport: viewport,
});
await textLayer.render();
```

### Why
The `renderTextLayer()` function was the legacy API and has been removed in pdfjs-dist 5.x. The `TextLayer` class provides a cleaner API with `render()`, `update()`, and `cancel()` lifecycle methods.

---

## AP-02: Missing Absolute Positioning on Text Layer Container

### Wrong

```typescript
const textLayerDiv = document.createElement("div");
textLayerDiv.className = "textLayer";
// No positioning -- text spans appear below the canvas
container.appendChild(textLayerDiv);
```

### Correct

```typescript
const textLayerDiv = document.createElement("div");
textLayerDiv.className = "textLayer";
textLayerDiv.style.position = "absolute";
textLayerDiv.style.top = "0";
textLayerDiv.style.left = "0";
textLayerDiv.style.width = `${Math.floor(viewport.width)}px`;
textLayerDiv.style.height = `${Math.floor(viewport.height)}px`;
container.appendChild(textLayerDiv);
```

### Why
TextLayer spans are positioned with CSS transforms relative to their container. Without `position: absolute` and matching dimensions, the text layer renders in normal document flow, completely misaligned from the canvas.

---

## AP-03: Missing Text Layer CSS (Visible Black Text)

### Wrong

```typescript
// No CSS imported or defined for .textLayer
const textLayer = new TextLayer({
  textContentSource: textContent,
  container: textLayerDiv,
  viewport: viewport,
});
await textLayer.render();
// Result: visible black text overlaid on the PDF canvas
```

### Correct

```css
/* Import PDF.js viewer CSS */
@import "pdfjs-dist/web/pdf_viewer.css";

/* OR apply minimal required styles */
.textLayer span {
  color: transparent;
  position: absolute;
  white-space: pre;
  cursor: text;
  transform-origin: 0% 0%;
}
```

### Why
Without `color: transparent`, the text spans render as visible black text on top of the canvas, creating double-rendered text. The text layer MUST be invisible -- its purpose is selection and searchability, not display.

---

## AP-04: Viewport Mismatch Between Canvas and Text Layer

### Wrong

```typescript
const canvasViewport = page.getViewport({ scale: 1.5 });
// ... render canvas with canvasViewport ...

// Using different scale for text layer
const textViewport = page.getViewport({ scale: 1.0 });
const textLayer = new TextLayer({
  textContentSource: textContent,
  container: textLayerDiv,
  viewport: textViewport,  // WRONG -- does not match canvas viewport
});
```

### Correct

```typescript
const viewport = page.getViewport({ scale: 1.5 });
// ... render canvas with viewport ...

// ALWAYS use the SAME viewport for text layer
const textLayer = new TextLayer({
  textContentSource: textContent,
  container: textLayerDiv,
  viewport: viewport,  // Same viewport as canvas
});
```

### Why
The text layer positions spans based on the viewport transform. If the text layer viewport does not match the canvas viewport, text selection areas will be shifted, scaled incorrectly, or rotated differently from the visible text.

---

## AP-05: Not Cancelling Text Layer Before Re-creating

### Wrong

```typescript
async function onZoom(newScale: number): Promise<void> {
  const viewport = page.getViewport({ scale: newScale });
  textLayerDiv.innerHTML = "";

  // Creating new TextLayer without cancelling the previous one
  const textLayer = new TextLayer({
    textContentSource: await page.getTextContent(),
    container: textLayerDiv,
    viewport: viewport,
  });
  await textLayer.render();
}
```

### Correct

```typescript
let currentTextLayer: TextLayer | null = null;

async function onZoom(newScale: number): Promise<void> {
  // ALWAYS cancel previous text layer first
  if (currentTextLayer) {
    currentTextLayer.cancel();
    currentTextLayer = null;
  }

  const viewport = page.getViewport({ scale: newScale });
  textLayerDiv.innerHTML = "";

  currentTextLayer = new TextLayer({
    textContentSource: await page.getTextContent(),
    container: textLayerDiv,
    viewport: viewport,
  });
  await currentTextLayer.render();
}
```

### Better (use update instead of re-creating)

```typescript
function onZoom(newScale: number): void {
  if (!currentTextLayer) return;

  const viewport = page.getViewport({ scale: newScale });
  // update() is more efficient -- reuses existing DOM elements
  currentTextLayer.update({
    viewport: viewport,
    onBefore() {
      textLayerDiv.style.width = `${Math.floor(viewport.width)}px`;
      textLayerDiv.style.height = `${Math.floor(viewport.height)}px`;
    },
  });
}
```

### Why
Without cancelling, the previous `render()` may still be processing asynchronously. This causes duplicate text spans, race conditions, and memory leaks. For simple viewport changes, `update()` is preferred over destroying and re-creating the text layer.

---

## AP-06: Accessing TextItem.str Without Type Guard

### Wrong

```typescript
const textContent = await page.getTextContent({
  includeMarkedContent: true,
});

for (const item of textContent.items) {
  // CRASHES when item is TextMarkedContent (no str property)
  console.log(item.str);
}
```

### Correct

```typescript
const textContent = await page.getTextContent({
  includeMarkedContent: true,
});

for (const item of textContent.items) {
  // ALWAYS check for str property when includeMarkedContent is true
  if ("str" in item) {
    console.log(item.str);
  }
}
```

### Why
When `includeMarkedContent: true`, the items array contains both `TextItem` and `TextMarkedContent` objects. `TextMarkedContent` has `type` and optionally `id`/`tag` properties but does NOT have `str`, `width`, `height`, or `transform`. Accessing `.str` on a `TextMarkedContent` item returns `undefined` and causes silent bugs.

---

## AP-07: Missing Parent Container Relative Positioning

### Wrong

```html
<!-- Parent container has no positioning context -->
<div id="page-container">
  <canvas></canvas>
  <div class="textLayer" style="position: absolute; top: 0; left: 0;"></div>
</div>
```

### Correct

```html
<!-- Parent MUST have position: relative to anchor absolute children -->
<div id="page-container" style="position: relative;">
  <canvas style="position: absolute; top: 0; left: 0;"></canvas>
  <div class="textLayer" style="position: absolute; top: 0; left: 0;"></div>
</div>
```

### Why
When the parent container lacks `position: relative`, the absolutely-positioned text layer and canvas find their nearest positioned ancestor (which may be `<body>`). This causes them to render at completely different locations in the page, making text selection impossible.

---

## AP-08: Calling TextLayer.cleanup() While Instances Are Active

### Wrong

```typescript
// Page 1 text layer is still active
const textLayer1 = new TextLayer({ /* ... */ });
await textLayer1.render();

// Cleanup while textLayer1 is still in use
TextLayer.cleanup();  // Destroys shared caches that textLayer1 needs
```

### Correct

```typescript
// Destroy all text layers first
textLayer1.cancel();
textLayer2.cancel();

// ONLY then clean up global caches
TextLayer.cleanup();
```

### Why
`TextLayer.cleanup()` clears global font metric caches and canvas contexts shared across ALL TextLayer instances. Calling it while active instances exist causes those instances to lose their cached measurements, leading to incorrect text positioning on subsequent updates.
