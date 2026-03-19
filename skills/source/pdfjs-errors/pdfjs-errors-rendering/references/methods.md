# Rendering API Reference (pdfjs-dist 5.x)

## PDFPageProxy.render()

The primary rendering method. Returns a `RenderTask` that controls the render operation.

```typescript
import type { PDFPageProxy } from "pdfjs-dist";

// Signature
page.render(params: RenderParameters): RenderTask;
```

### RenderParameters

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `canvasContext` | `CanvasRenderingContext2D` | Yes | The 2D canvas context to render into |
| `viewport` | `PageViewport` | Yes | The viewport defining transform and dimensions |
| `intent` | `"display" \| "print"` | No | Render intent. Default: `"display"` |
| `annotationMode` | `number` | No | Controls annotation rendering (0=disable, 1=enable, 2=forms, 3=storage) |
| `transform` | `number[]` | No | Additional transform matrix applied during rendering |
| `background` | `string` | No | Background color for the canvas (default: transparent) |
| `optionalContentConfigPromise` | `Promise` | No | Promise for optional content group configuration |
| `pageColors` | `object` | No | Override page foreground/background colors |

### Example Usage

```typescript
import type { PDFPageProxy } from "pdfjs-dist";

const viewport = page.getViewport({ scale: 1.5 });
const ctx = canvas.getContext("2d")!;

const renderTask = page.render({
  canvasContext: ctx,
  viewport: viewport,
  intent: "display",
});

await renderTask.promise;
```

---

## RenderTask

Returned by `page.render()`. Controls and monitors the render operation.

```typescript
interface RenderTask {
  promise: Promise<void>;
  cancel(): void;
  onContinue: ((cont: () => void) => void) | null;
}
```

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `promise` | `Promise<void>` | Resolves when rendering completes. Rejects with `RenderingCancelledException` if cancelled. |
| `onContinue` | `function \| null` | Callback for pausing/resuming rendering (rarely used). |

### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `cancel()` | `void` | Cancels the render operation. The `promise` will reject with `RenderingCancelledException`. |

### RenderingCancelledException

Thrown when `renderTask.cancel()` is called while rendering is in progress.

```typescript
try {
  await renderTask.promise;
} catch (err: unknown) {
  if (err instanceof Error && err.name === "RenderingCancelledException") {
    // Render was intentionally cancelled -- not an error
    return;
  }
  throw err; // Unexpected rendering error
}
```

**ALWAYS** check for `RenderingCancelledException` by name, not by message string. The error name is stable across versions; the message may change.

---

## PDFPageProxy.getViewport()

Creates a `PageViewport` that defines the rendering transform.

```typescript
page.getViewport(params: GetViewportParameters): PageViewport;
```

### GetViewportParameters

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `scale` | `number` | Yes | - | Scale factor for rendering |
| `rotation` | `number` | No | 0 | Rotation in degrees (0, 90, 180, 270) |
| `offsetX` | `number` | No | 0 | Horizontal offset |
| `offsetY` | `number` | No | 0 | Vertical offset |
| `dontFlip` | `boolean` | No | false | If true, do not flip the Y axis |

### PageViewport Properties

| Property | Type | Description |
|----------|------|-------------|
| `width` | `number` | Width in CSS pixels at the given scale |
| `height` | `number` | Height in CSS pixels at the given scale |
| `scale` | `number` | The scale factor |
| `rotation` | `number` | The rotation angle |
| `viewBox` | `number[]` | The original page dimensions `[x, y, width, height]` |
| `transform` | `number[]` | The 6-element transform matrix |

### Scale Calculation

```typescript
// Calculate scale to fit a specific width
const desiredWidth = 800; // pixels
const unscaledViewport = page.getViewport({ scale: 1.0 });
const fitScale = desiredWidth / unscaledViewport.width;
const fittedViewport = page.getViewport({ scale: fitScale });

// Calculate scale to fit a container
function scaleToFit(page: PDFPageProxy, container: HTMLElement): number {
  const unscaled = page.getViewport({ scale: 1.0 });
  const scaleX = container.clientWidth / unscaled.width;
  const scaleY = container.clientHeight / unscaled.height;
  return Math.min(scaleX, scaleY); // Fit within container
}
```

---

## Canvas Context Methods (Relevant to PDF.js)

### getContext("2d")

```typescript
const ctx = canvas.getContext("2d");
```

**ALWAYS** check the return value -- `getContext()` returns `null` if:
- The canvas already has a different context type (e.g., WebGL)
- The canvas has been detached from the DOM in some browsers
- The browser cannot allocate a new context (context limit reached)

### Canvas Dimension Limits

Browsers enforce maximum canvas dimensions:

| Browser | Max Area (pixels) | Max Dimension (px) |
|---------|------------------|--------------------|
| Chrome | ~268 million | 65,535 |
| Firefox | ~500 million | 32,767 |
| Safari | ~67 million | 16,384 |

```typescript
// ALWAYS check canvas dimensions against limits before rendering
function isCanvasSizeValid(width: number, height: number): boolean {
  const MAX_AREA = 16_777_216; // Conservative: 16M pixels (safe for all browsers)
  const MAX_DIM = 16_384;       // Conservative max dimension
  return (
    width > 0 &&
    height > 0 &&
    width <= MAX_DIM &&
    height <= MAX_DIM &&
    width * height <= MAX_AREA
  );
}
```

**NEVER** create canvases that exceed these limits -- the browser will silently fail or produce a blank/corrupted canvas with no error.

---

## CanvasFactory (Advanced)

PDF.js uses a `CanvasFactory` internally to create and manage canvases. You can override it for custom canvas management.

```typescript
import { getDocument } from "pdfjs-dist";

class CustomCanvasFactory {
  create(width: number, height: number): { canvas: HTMLCanvasElement; context: CanvasRenderingContext2D } {
    const canvas = document.createElement("canvas");
    canvas.width = width;
    canvas.height = height;
    const context = canvas.getContext("2d");
    if (!context) {
      throw new Error("Cannot create 2D canvas context");
    }
    return { canvas, context };
  }

  reset(
    canvasAndContext: { canvas: HTMLCanvasElement; context: CanvasRenderingContext2D },
    width: number,
    height: number
  ): void {
    canvasAndContext.canvas.width = width;
    canvasAndContext.canvas.height = height;
  }

  destroy(canvasAndContext: { canvas: HTMLCanvasElement; context: CanvasRenderingContext2D }): void {
    canvasAndContext.canvas.width = 0;
    canvasAndContext.canvas.height = 0;
  }
}
```

**When to use**: Custom canvas pooling, OffscreenCanvas for web workers, or canvas memory management in large document viewers.

---

## Text Layer API

### renderTextLayer()

Renders selectable text over the canvas rendering.

```typescript
import { renderTextLayer } from "pdfjs-dist";
import type { TextContent } from "pdfjs-dist";

const textContent: TextContent = await page.getTextContent();

const textLayerDiv = document.createElement("div");
textLayerDiv.className = "textLayer";

const task = renderTextLayer({
  textContentSource: textContent,
  container: textLayerDiv,
  viewport: viewport, // MUST match the canvas viewport exactly
});

await task.promise;
```

### renderTextLayer Parameters

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `textContentSource` | `TextContent \| ReadableStream` | Yes | Text content from `page.getTextContent()` |
| `container` | `HTMLElement` | Yes | DOM element to render text spans into |
| `viewport` | `PageViewport` | Yes | MUST be the same viewport used for canvas render |

**ALWAYS** use the same `viewport` instance for both `page.render()` and `renderTextLayer()`. Creating a new viewport with the same parameters may produce floating-point differences that cause misalignment.

---

## Annotation Layer API

### AnnotationLayer.render()

Renders interactive annotations (links, form fields) over the canvas.

```typescript
import { AnnotationLayer } from "pdfjs-dist";
import type { PDFPageProxy } from "pdfjs-dist";

const annotations = await page.getAnnotations();

const annotationLayerDiv = document.createElement("div");
annotationLayerDiv.className = "annotationLayer";

AnnotationLayer.render({
  annotations: annotations,
  div: annotationLayerDiv,
  viewport: viewport, // MUST match canvas viewport
  page: page,
});
```

### AnnotationLayer.render Parameters

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `annotations` | `AnnotationData[]` | Yes | Annotations from `page.getAnnotations()` |
| `div` | `HTMLElement` | Yes | Container element for annotation elements |
| `viewport` | `PageViewport` | Yes | MUST match the canvas viewport |
| `page` | `PDFPageProxy` | Yes | The page reference |
| `linkService` | `IPDFLinkService` | No | Custom link handler for internal PDF links |
| `downloadManager` | `IDownloadManager` | No | Custom download handler for file attachments |

**ALWAYS** position the annotation layer div above the canvas and text layer using CSS `z-index`. **NEVER** set `pointer-events: none` on the annotation layer container.
