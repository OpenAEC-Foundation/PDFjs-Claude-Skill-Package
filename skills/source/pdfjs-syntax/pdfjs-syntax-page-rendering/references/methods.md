# API Signatures Reference (pdfjs-dist 5.x Page Rendering)

## PDFPageProxy.getViewport()

Creates a `PageViewport` object for the given rendering parameters.

```typescript
getViewport(params: {
  scale: number;               // REQUIRED. Zoom level (1.0 = 100%)
  rotation?: number;           // Page rotation in degrees (0, 90, 180, 270). Default: 0
  offsetX?: number;            // Horizontal offset in viewport coordinates. Default: 0
  offsetY?: number;            // Vertical offset in viewport coordinates. Default: 0
  dontFlip?: boolean;          // If true, do not flip the Y axis. Default: false
}): PageViewport
```

**Notes**:
- `scale` is the primary parameter controlling zoom level
- `rotation` is combined with the page's intrinsic rotation from the PDF metadata
- `dontFlip` is for advanced use cases where you need PDF coordinate space (origin at bottom-left) instead of canvas coordinate space (origin at top-left)

---

## PageViewport

Represents the dimensions and transform for rendering a page at a specific scale and rotation.

### Properties

```typescript
interface PageViewport {
  readonly width: number;       // Viewport width in CSS pixels (after scale + rotation)
  readonly height: number;      // Viewport height in CSS pixels (after scale + rotation)
  readonly scale: number;       // The scale factor used to create this viewport
  readonly rotation: number;    // Total rotation (page rotation + requested rotation)
  readonly viewBox: number[];   // The original page dimensions [x, y, width, height]
  readonly transform: number[]; // 6-element CSS transform matrix [a, b, c, d, e, f]
  readonly offsetX: number;     // Horizontal offset
  readonly offsetY: number;     // Vertical offset
}
```

### Methods

```typescript
// Create a new viewport with modified parameters (immutable clone)
clone(params?: {
  scale?: number;
  rotation?: number;
  offsetX?: number;
  offsetY?: number;
  dontFlip?: boolean;
}): PageViewport

// Convert PDF coordinates to viewport coordinates
convertToViewportPoint(x: number, y: number): [number, number]

// Convert viewport coordinates to PDF coordinates
convertToPdfPoint(x: number, y: number): [number, number]

// Convert a PDF rectangle to viewport rectangle
convertToViewportRectangle(rect: number[]): number[]
```

**Usage**: The `clone()` method is useful when implementing zoom -- clone the existing viewport with a new scale instead of calling `getViewport()` again from the page.

---

## PDFPageProxy.render()

Renders the page content to a canvas. Returns a `RenderTask` object.

```typescript
render(params: RenderParameters): RenderTask
```

### RenderParameters

```typescript
interface RenderParameters {
  // REQUIRED
  canvasContext: CanvasRenderingContext2D | OffscreenCanvasRenderingContext2D;
  viewport: PageViewport;

  // OPTIONAL
  transform?: number[];          // Additional transform matrix [a, b, c, d, e, f]
                                 // Applied ON TOP of the viewport transform
  background?: string;           // CSS color for canvas background. Default: "rgb(255,255,255)"
                                 // Use "rgba(0,0,0,0)" for transparent background
  annotationMode?: number;       // Controls annotation rendering:
                                 //   0 = DISABLE
                                 //   1 = ENABLE (read-only)
                                 //   2 = ENABLE_FORMS (interactive, default)
                                 //   3 = ENABLE_STORAGE (persistent)
  intent?: string;               // Rendering intent:
                                 //   "display" (default) -- screen rendering
                                 //   "print" -- print rendering (may include print-only annotations)
                                 //   "any" -- both display and print content
  canvasFactory?: object;        // Custom canvas factory (for non-browser environments)
  imageLayer?: object;           // Optional image layer for image rendering
  optionalContentConfigPromise?: Promise<object>;  // Optional content groups config
  pageColors?: {                 // Override page colors (accessibility)
    background?: string;         // Background color override
    foreground?: string;         // Text/foreground color override
  };
}
```

---

## RenderTask

Represents an in-progress page render. Returned by `page.render()`.

### Properties and Methods

```typescript
interface RenderTask {
  // Promise that resolves when rendering completes
  // Rejects if rendering is cancelled or fails
  readonly promise: Promise<void>;

  // Cancels the current rendering operation
  // Causes `promise` to reject with "Rendering cancelled" error
  cancel(): void;
}
```

### Cancel Behavior

- Calling `cancel()` causes `promise` to reject with an error where `message === "Rendering cancelled"`
- ALWAYS wrap `await renderTask.promise` in try/catch when cancellation is possible
- After cancel, the canvas content is in an indeterminate state -- ALWAYS re-render after cancel
- Cancelling an already-completed render is a no-op (safe to call)

---

## AnnotationMode Constants

```typescript
import { AnnotationMode } from "pdfjs-dist";

AnnotationMode.DISABLE;        // 0 -- Do not render annotations
AnnotationMode.ENABLE;         // 1 -- Render as static elements
AnnotationMode.ENABLE_FORMS;   // 2 -- Render with interactive form widgets
AnnotationMode.ENABLE_STORAGE; // 3 -- Render with persistent storage for form data
```

---

## Canvas DPI Setup Reference

```typescript
function setupCanvasForDPI(
  canvas: HTMLCanvasElement,
  viewport: PageViewport
): CanvasRenderingContext2D {
  const dpr = window.devicePixelRatio || 1;

  // Pixel dimensions (actual resolution)
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);

  // CSS dimensions (display size)
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;

  // Scale context so drawing operations use CSS pixel coordinates
  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);

  return ctx;
}
```

**WHY `Math.floor()`**: Canvas dimensions MUST be integers. Fractional pixel values cause sub-pixel rendering artifacts. ALWAYS floor viewport dimensions before assigning to canvas.

---

## Coordinate Conversion Methods

```typescript
// Convert a point from PDF space to canvas/screen space
const [screenX, screenY] = viewport.convertToViewportPoint(pdfX, pdfY);

// Convert a point from canvas/screen space to PDF space
const [pdfX, pdfY] = viewport.convertToPdfPoint(screenX, screenY);

// Convert a rectangle [x1, y1, x2, y2] from PDF to viewport space
const viewportRect = viewport.convertToViewportRectangle([x1, y1, x2, y2]);
// Returns [vx1, vy1, vx2, vy2] in viewport coordinates
```

**Use case**: These methods are essential when implementing click-to-coordinate features, custom annotation placement, or mapping text positions back to PDF coordinates.
