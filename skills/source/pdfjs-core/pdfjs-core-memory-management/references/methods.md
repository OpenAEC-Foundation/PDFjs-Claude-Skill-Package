# Methods: Memory Management APIs

> pdfjs-dist 5.x -- Cleanup and destroy method signatures.

## PDFDocumentProxy.destroy()

```typescript
destroy(): Promise<void>
```

Releases all resources held by the document: worker thread communication channel, cached page data, font data, and internal buffers. After calling `destroy()`, the `PDFDocumentProxy` instance and ALL associated `PDFPageProxy` instances become invalid.

**Returns:** `Promise<void>` -- resolves when all resources are released.

**Preconditions:**
- ALWAYS cancel all active `RenderTask` instances before calling `destroy()`
- ALWAYS cancel active `TextLayer` and `AnnotationLayer` operations first

**Post-conditions:**
- The worker thread associated with this document is terminated
- All `PDFPageProxy` objects from this document are invalidated
- Any subsequent method call on this proxy or its pages throws an error

```typescript
// ALWAYS await destroy when loading a replacement document
await pdfDoc.destroy();
```

---

## PDFDocumentLoadingTask.destroy()

```typescript
destroy(): Promise<void>
```

Aborts the loading process and releases resources. Use this to cancel a document load that is still in progress (e.g., user navigates away before loading completes).

```typescript
const loadingTask = getDocument(url);

// User cancelled -- abort loading
await loadingTask.destroy();
```

---

## PDFPageProxy.cleanup()

```typescript
cleanup(resetStats?: boolean): boolean
```

Releases cached rendering data (operator list, image data) for the page. The `PDFPageProxy` object remains valid after cleanup -- calling `render()` again will re-fetch the data from the worker.

**Parameters:**
- `resetStats` (optional, default `false`) -- If `true`, resets the page's rendering statistics.

**Returns:** `boolean` -- `true` if cleanup was performed, `false` if a render was active and cleanup was skipped.

**Key behavior:**
- Does NOT invalidate the `PDFPageProxy` -- the page can be re-rendered later
- Returns `false` if a `RenderTask` is currently active on this page -- ALWAYS cancel the render task first
- Use for page pool eviction, NOT for final teardown

```typescript
// Evict page from cache (page can be re-rendered later)
const wasCleanedUp = page.cleanup();
if (!wasCleanedUp) {
  // A render task is still active -- cancel it first
  renderTask.cancel();
  page.cleanup();
}
```

---

## RenderTask.cancel()

```typescript
cancel(extraDelay?: number): void
```

Cancels the ongoing render operation. The `RenderTask.promise` will reject with a `RenderingCancelledException`.

**Parameters:**
- `extraDelay` (optional) -- Additional delay in milliseconds before the cancel takes effect. Rarely used.

**Behavior:**
- Synchronous call -- cancellation is initiated immediately
- The promise rejects with `{ name: 'RenderingCancelledException' }`
- ALWAYS handle the rejection to avoid unhandled promise errors

```typescript
renderTask.cancel();

try {
  await renderTask.promise;
} catch (err: any) {
  if (err.name === 'RenderingCancelledException') {
    // Expected -- not an error
    return;
  }
  throw err; // Re-throw unexpected errors
}
```

---

## TextLayer.cancel()

```typescript
cancel(): void
```

Cancels an in-progress text layer rendering operation. ALWAYS call before removing the text layer container from the DOM or before destroying the parent page.

```typescript
textLayer.cancel();
textLayerDiv.remove();
```

---

## AnnotationLayer.cancel()

```typescript
cancel(): void
```

Cancels an in-progress annotation layer rendering operation. ALWAYS call before removing the annotation layer container from the DOM.

```typescript
annotationLayer.cancel();
annotationLayerDiv.remove();
```

---

## URL.revokeObjectURL()

```typescript
URL.revokeObjectURL(url: string): void
```

Not a PDF.js method, but CRITICAL for memory management when loading PDFs from `Blob` or `File` objects.

**When to call:** Immediately after `getDocument(blobUrl).promise` resolves or rejects. PDF.js reads the full binary data during loading -- the blob URL is not needed afterward.

```typescript
const blobUrl = URL.createObjectURL(pdfBlob);
try {
  const doc = await getDocument(blobUrl).promise;
} finally {
  URL.revokeObjectURL(blobUrl);
}
```

---

## Canvas Context Reset Methods

### CanvasRenderingContext2D.clearRect()

```typescript
ctx.clearRect(0, 0, canvas.width, canvas.height);
```

Clears all pixels in the canvas. Use before re-rendering a page at a different scale or with different content.

### Canvas Dimension Reset

```typescript
canvas.width = 0;
canvas.height = 0;
```

Setting both dimensions to 0 forces the browser to release the GPU-backed bitmap. This is more aggressive than `clearRect()` and ALWAYS releases memory. Use when a canvas will not be immediately reused.

### Full Canvas Cleanup Sequence

```typescript
function destroyCanvas(canvas: HTMLCanvasElement): void {
  const ctx = canvas.getContext('2d');
  if (ctx) {
    ctx.clearRect(0, 0, canvas.width, canvas.height);
  }
  canvas.width = 0;
  canvas.height = 0;
  canvas.remove();
}
```

---

## OffscreenCanvas.transferToImageBitmap()

```typescript
const bitmap = offscreenCanvas.transferToImageBitmap();
```

When using `OffscreenCanvas` for rendering in a worker, `transferToImageBitmap()` transfers ownership of the pixel data. After transfer, the `OffscreenCanvas` can be reused for the next render without allocating new memory.

ALWAYS call `bitmap.close()` when the `ImageBitmap` is no longer needed:

```typescript
const bitmap = offscreenCanvas.transferToImageBitmap();
mainCtx.drawImage(bitmap, 0, 0);
bitmap.close(); // Release transferred pixel data
```
