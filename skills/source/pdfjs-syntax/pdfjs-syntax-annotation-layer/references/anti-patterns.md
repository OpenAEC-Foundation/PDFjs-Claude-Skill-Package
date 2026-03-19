# Annotation Layer Anti-Patterns (pdfjs-dist 5.x)

## 1. Wrong Z-Index: Annotations Behind Text Layer

**Symptom**: Links and form fields are visible but not clickable. Mouse events are captured by the text layer instead.

**Wrong:**
```css
.textLayer {
  z-index: 2;
}
.annotationLayer {
  z-index: 1; /* WRONG -- annotations are behind text layer */
}
```

**Correct:**
```css
.textLayer {
  z-index: 1;
}
.annotationLayer {
  z-index: 2; /* ALWAYS higher than text layer */
}
.annotationEditorLayer {
  z-index: 3; /* ALWAYS highest for editor */
}
```

**Rule**: ALWAYS stack layers in order: canvas (0), textLayer (1), annotationLayer (2), annotationEditorLayer (3). The annotation layer MUST have a higher z-index than the text layer.

---

## 2. Missing AnnotationStorage for Forms

**Symptom**: Form fields render but user input is lost when scrolling away and back, or when printing.

**Wrong:**
```typescript
// No annotationStorage -- form data is not persisted
const annotationLayer = new AnnotationLayer({
  div: annotationDiv,
  annotations: annotations,
  page: page,
  viewport: viewport,
  renderForms: true,
});
```

**Correct:**
```typescript
// ALWAYS create ONE shared AnnotationStorage for the entire document
const annotationStorage = new AnnotationStorage();

const annotationLayer = new AnnotationLayer({
  div: annotationDiv,
  annotations: annotations,
  page: page,
  viewport: viewport,
  annotationStorage: annotationStorage, // Persists form data
  renderForms: true,
});
```

**Rule**: ALWAYS pass an `AnnotationStorage` instance when `renderForms: true`. Use ONE shared instance across all pages of the same document.

---

## 3. Missing CSS Import

**Symptom**: Annotations render but appear at wrong sizes, wrong positions, or with broken styling. Link annotations may not show hover effects.

**Wrong:**
```typescript
// No CSS imported -- annotations unstyled
const annotationLayer = new AnnotationLayer({ /* ... */ });
await annotationLayer.render({ viewport, annotations });
```

**Correct:**
```css
/* ALWAYS import the official PDF.js viewer CSS */
@import "pdfjs-dist/web/pdf_viewer.css";
```

**Rule**: ALWAYS include the `pdfjs-dist/web/pdf_viewer.css` stylesheet. It contains essential positioning, sizing, and interaction styles for annotation elements.

---

## 4. Not Using position: absolute on Annotation Layer

**Symptom**: Annotation layer renders below the canvas instead of overlapping it. Annotations are displaced from their correct positions on the page.

**Wrong:**
```typescript
const annotationDiv = document.createElement("div");
annotationDiv.className = "annotationLayer";
// No positioning -- div flows normally in the document
container.appendChild(annotationDiv);
```

**Correct:**
```typescript
// Container MUST be position: relative
container.style.position = "relative";

const annotationDiv = document.createElement("div");
annotationDiv.className = "annotationLayer";
annotationDiv.style.position = "absolute";
annotationDiv.style.top = "0";
annotationDiv.style.left = "0";
container.appendChild(annotationDiv);
```

**Rule**: ALWAYS use `position: absolute` on the annotation layer div. ALWAYS use `position: relative` on its parent container. This is required for annotations to align with the rendered PDF content.

---

## 5. Wrong Intent for getAnnotations()

**Symptom**: Certain annotations are missing on screen, or print-only annotations appear in the viewer.

**Wrong:**
```typescript
// Using "print" intent for screen display -- may show print-only stamps
const annotations = await page.getAnnotations({ intent: "print" });

// Rendering these in the screen viewer...
```

**Correct:**
```typescript
// For screen display
const displayAnnotations = await page.getAnnotations({ intent: "display" });

// For printing
const printAnnotations = await page.getAnnotations({ intent: "print" });

// For all annotations regardless of visibility flags
const allAnnotations = await page.getAnnotations({ intent: "any" });
```

**Rule**: ALWAYS use `intent: "display"` for screen rendering and `intent: "print"` for print rendering. NEVER mix intents.

---

## 6. Creating Multiple AnnotationStorage Instances

**Symptom**: Form data entered on one page is not available on another page. Different pages have isolated form state.

**Wrong:**
```typescript
// Each page gets its own storage -- data is NOT shared
for (let i = 1; i <= numPages; i++) {
  const storage = new AnnotationStorage(); // WRONG -- new instance per page
  const layer = new AnnotationLayer({
    annotationStorage: storage,
    // ...
  });
}
```

**Correct:**
```typescript
// ONE storage instance for the entire document
const storage = new AnnotationStorage();

for (let i = 1; i <= numPages; i++) {
  const layer = new AnnotationLayer({
    annotationStorage: storage, // Same instance for all pages
    // ...
  });
}
```

**Rule**: ALWAYS create exactly ONE `AnnotationStorage` instance per document and share it across all pages.

---

## 7. Forgetting renderForms: true

**Symptom**: Form fields appear as static images. Users cannot type in text fields, check checkboxes, or select options.

**Wrong:**
```typescript
const annotationLayer = new AnnotationLayer({
  div: annotationDiv,
  annotations: annotations,
  page: page,
  viewport: viewport,
  // renderForms defaults to false -- forms are static
});
```

**Correct:**
```typescript
const annotationLayer = new AnnotationLayer({
  div: annotationDiv,
  annotations: annotations,
  page: page,
  viewport: viewport,
  renderForms: true, // ALWAYS set for interactive forms
  annotationStorage: storage,
});
```

**Rule**: ALWAYS set `renderForms: true` when you need interactive form fields. Without it, form widgets render as non-interactive images.

---

## 8. Not Cancelling Before Removing Annotation Layer

**Symptom**: Memory leaks, orphaned event listeners, or errors when navigating away from a page.

**Wrong:**
```typescript
// Just removing the div without cleanup
annotationDiv.remove();
```

**Correct:**
```typescript
// ALWAYS cancel before removing from DOM
annotationLayer.cancel();
annotationDiv.remove();
```

**Rule**: ALWAYS call `annotationLayer.cancel()` before removing the annotation layer div from the DOM. This cleans up pending operations and event listeners.

---

## 9. Re-rendering Instead of Updating on Viewport Change

**Symptom**: Slow zoom/rotation because the entire annotation layer is destroyed and rebuilt. Form field input is lost on zoom.

**Wrong:**
```typescript
function onZoomChange(newViewport: PageViewport): void {
  // Destroying and rebuilding loses form state and is slow
  annotationDiv.innerHTML = "";
  annotationLayer = new AnnotationLayer({ /* ... */ });
  annotationLayer.render({ viewport: newViewport, annotations });
}
```

**Correct:**
```typescript
function onZoomChange(newViewport: PageViewport): void {
  // update() repositions existing elements without re-creating them
  annotationLayer.update({ viewport: newViewport });
}
```

**Rule**: ALWAYS use `annotationLayer.update()` for viewport changes (zoom, rotation). NEVER destroy and rebuild the annotation layer just to update positioning. `update()` preserves form state and is significantly faster.

---

## 10. Not Validating External Link URLs

**Symptom**: Security vulnerability where malicious PDFs can execute JavaScript via `javascript:` URLs in link annotations.

**Wrong:**
```typescript
// Blindly opening any URL from the PDF
if (annotation.url) {
  window.open(annotation.url);
}
```

**Correct:**
```typescript
if (annotation.url) {
  try {
    const parsed = new URL(annotation.url);
    // ONLY allow safe protocols
    if (parsed.protocol === "http:" || parsed.protocol === "https:") {
      window.open(annotation.url, "_blank");
    } else {
      console.warn("Blocked unsafe URL protocol:", parsed.protocol);
    }
  } catch {
    console.warn("Invalid URL in annotation:", annotation.url);
  }
}
```

**Rule**: ALWAYS validate external link URLs from PDF annotations before opening them. NEVER allow `javascript:`, `data:`, or other potentially dangerous URL protocols. Only permit `http:` and `https:`.

---

## 11. Using AnnotationMode.ENABLE Instead of ENABLE_FORMS

**Symptom**: Annotations render on the canvas but form fields are static images, not interactive HTML elements.

**Wrong:**
```typescript
// AnnotationMode.ENABLE (1) renders annotations but not interactive forms
await page.render({
  canvasContext: ctx,
  viewport: viewport,
  annotationMode: 1, // ENABLE -- forms are read-only images
});
```

**Correct:**
```typescript
import { AnnotationMode } from "pdfjs-dist";

// Use ENABLE_FORMS (2) or ENABLE_STORAGE (3) for interactive forms
await page.render({
  canvasContext: ctx,
  viewport: viewport,
  annotationMode: AnnotationMode.ENABLE_FORMS,
});

// Then render the AnnotationLayer with renderForms: true
```

**Rule**: ALWAYS use `AnnotationMode.ENABLE_FORMS` (2) or `AnnotationMode.ENABLE_STORAGE` (3) when rendering pages with interactive forms. `AnnotationMode.ENABLE` (1) only renders annotations as static, non-interactive elements.
