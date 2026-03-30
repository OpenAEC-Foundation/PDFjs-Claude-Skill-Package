---
name: pdfjs-syntax-annotation-layer
description: >
  Use when rendering interactive annotations (links, forms, highlights) on PDF pages
  or implementing annotation editing. Prevents incorrect layer stacking by enforcing
  the canvas > TextLayer > AnnotationLayer order.
  Covers AnnotationLayer class, getAnnotations(), annotation types, link handling,
  form field rendering, AnnotationStorage, and AnnotationEditorLayer.
  Keywords: AnnotationLayer, getAnnotations, AnnotationStorage, Widget, Link,
  form fields, clickable links, interactive PDF, annotations not showing,
  form overlay, PDF buttons.
license: MIT
compatibility: "Designed for Claude Code. Requires pdfjs-dist 5.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# pdfjs-syntax-annotation-layer

## Quick Reference

### Annotation Rendering Pipeline

| Step | Method | Output |
|------|--------|--------|
| 1. Get annotations | `page.getAnnotations({ intent })` | `AnnotationData[]` |
| 2. Create container | `<div class="annotationLayer">` | DOM element |
| 3. Create layer | `new AnnotationLayer({ div, ... })` | `AnnotationLayer` |
| 4. Render | `annotationLayer.render({ viewport, annotations })` | Interactive HTML elements |

### Layer Stacking Order

| Layer | z-index | Purpose |
|-------|---------|---------|
| Canvas | 0 | PDF page pixels |
| TextLayer | 1 | Selectable/searchable text overlay |
| **AnnotationLayer** | **2** | Links, forms, annotations |
| AnnotationEditorLayer | 3 | Editing overlays (FreeText, Ink, etc.) |

### Annotation Types

| Type | Constant | Value | Purpose |
|------|----------|-------|---------|
| Text | `AnnotationType.TEXT` | 1 | Sticky notes / comments |
| Link | `AnnotationType.LINK` | 2 | Internal and external links |
| FreeText | `AnnotationType.FREETEXT` | 3 | Free-form text annotations |
| Highlight | `AnnotationType.HIGHLIGHT` | 9 | Text highlighting |
| Underline | `AnnotationType.UNDERLINE` | 10 | Text underline |
| Squiggly | `AnnotationType.SQUIGGLY` | 11 | Squiggly underline |
| StrikeOut | `AnnotationType.STRIKEOUT` | 12 | Strikethrough text |
| Stamp | `AnnotationType.STAMP` | 13 | Stamp images |
| Ink | `AnnotationType.INK` | 15 | Freehand drawings |
| Popup | `AnnotationType.POPUP` | 16 | Popup windows for comments |
| FileAttachment | `AnnotationType.FILEATTACHMENT` | 17 | Embedded file attachments |
| Widget | `AnnotationType.WIDGET` | 20 | Form fields (text, checkbox, radio, select, button) |

### Critical Warnings

**ALWAYS** render the annotation layer on TOP of the text layer -- annotation layer MUST have a higher z-index than the text layer, otherwise links and form fields are not clickable.

**ALWAYS** use `position: absolute` on the annotation layer div and place it inside a `position: relative` container alongside the canvas and text layer -- without absolute positioning, annotations will not align with the rendered PDF content.

**NEVER** create an AnnotationLayer without first calling `page.getAnnotations()` -- the layer requires annotation data from the PDF page to know what to render.

**ALWAYS** include the PDF.js annotation layer CSS (`pdfjs-dist/web/pdf_viewer.css`) -- without it, annotations render with wrong sizes, positions, and styling.

**NEVER** use `intent: "display"` when printing and `intent: "print"` when displaying -- some annotations are print-only or display-only. Mismatched intent causes missing or extra annotations.

---

## Essential Patterns

### Basic Annotation Layer Rendering

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";
import { AnnotationLayer } from "pdfjs-dist";
import type { PDFPageProxy, PageViewport } from "pdfjs-dist";

// ALWAYS configure worker before loading documents
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

async function renderAnnotationLayer(
  page: PDFPageProxy,
  container: HTMLDivElement,
  viewport: PageViewport
): Promise<AnnotationLayer> {
  // Step 1: Get annotation data from the page
  const annotations = await page.getAnnotations({ intent: "display" });

  // Step 2: Create the annotation layer container
  const annotationDiv = document.createElement("div");
  annotationDiv.className = "annotationLayer";
  annotationDiv.style.position = "absolute";
  annotationDiv.style.top = "0";
  annotationDiv.style.left = "0";
  annotationDiv.style.width = `${Math.floor(viewport.width)}px`;
  annotationDiv.style.height = `${Math.floor(viewport.height)}px`;
  container.appendChild(annotationDiv);

  // Step 3: Create and render the AnnotationLayer
  const annotationLayer = new AnnotationLayer({
    div: annotationDiv,
    annotations: annotations,
    page: page,
    viewport: viewport,
  });

  await annotationLayer.render({ viewport, annotations });
  return annotationLayer;
}
```

### AnnotationLayer with Link Handling

```typescript
import { AnnotationLayer } from "pdfjs-dist";
import type { PDFPageProxy, PageViewport, PDFDocumentProxy } from "pdfjs-dist";

// Link service handles navigation for internal and external links
interface SimpleLinkService {
  navigateTo(dest: string | any[]): void;
  getDestinationHash(dest: string | any[]): string;
  getAnchorUrl(hash: string): string;
  externalLinkEnabled: boolean;
  externalLinkTarget: number; // 0=NONE, 1=SELF, 2=BLANK, 3=PARENT, 4=TOP
}

function createLinkService(doc: PDFDocumentProxy): SimpleLinkService {
  return {
    externalLinkEnabled: true,
    externalLinkTarget: 2, // Open external links in new tab

    navigateTo(dest: string | any[]): void {
      if (typeof dest === "string") {
        // Named destination -- resolve and navigate
        doc.getDestination(dest).then((resolved) => {
          if (resolved) {
            doc.getPageIndex(resolved[0]).then((pageIndex) => {
              console.log("Navigate to page:", pageIndex + 1);
            });
          }
        });
      }
    },

    getDestinationHash(dest: string | any[]): string {
      return typeof dest === "string" ? `#${dest}` : "#";
    },

    getAnchorUrl(hash: string): string {
      return hash;
    },
  };
}
```

### AnnotationLayer with Form Fields (AnnotationStorage)

```typescript
import { AnnotationLayer, AnnotationStorage } from "pdfjs-dist";
import type { PDFPageProxy, PageViewport } from "pdfjs-dist";

async function renderFormAnnotations(
  page: PDFPageProxy,
  container: HTMLDivElement,
  viewport: PageViewport,
  annotationStorage: AnnotationStorage
): Promise<AnnotationLayer> {
  const annotations = await page.getAnnotations({ intent: "display" });

  const annotationDiv = document.createElement("div");
  annotationDiv.className = "annotationLayer";
  annotationDiv.style.position = "absolute";
  annotationDiv.style.top = "0";
  annotationDiv.style.left = "0";
  container.appendChild(annotationDiv);

  // ALWAYS pass annotationStorage for interactive forms
  const annotationLayer = new AnnotationLayer({
    div: annotationDiv,
    annotations: annotations,
    page: page,
    viewport: viewport,
    annotationStorage: annotationStorage,
    renderForms: true, // Enable interactive form widgets
  });

  await annotationLayer.render({ viewport, annotations });
  return annotationLayer;
}

// Using AnnotationStorage to read/write form data
const storage = new AnnotationStorage();

// Set a form field value (key is the annotation ID)
storage.setValue("field_123", { value: "John Doe" });

// Get a form field value with default
const val = storage.getValue("field_123", { value: "" });

// Get ALL stored form values
const allValues = storage.getAll();
```

---

## Decision Tree: Annotation Type Handling

```
Need to handle annotations on a PDF page?
|
+-- What kind of annotation?
|   |
|   +-- Link (AnnotationType.LINK = 2)
|   |   +-- Has `dest` property? --> Internal link (page navigation)
|   |   +-- Has `url` property? --> External link (open URL)
|   |   +-- Has `action` property? --> Named action (NextPage, PrevPage, etc.)
|   |
|   +-- Widget (AnnotationType.WIDGET = 20) -- Form field
|   |   +-- fieldType === "Tx" --> Text input field
|   |   +-- fieldType === "Btn" + checkBox --> Checkbox
|   |   +-- fieldType === "Btn" + radioButton --> Radio button
|   |   +-- fieldType === "Btn" + pushButton --> Push button
|   |   +-- fieldType === "Ch" --> Select / dropdown
|   |   +-- fieldType === "Sig" --> Signature field
|   |
|   +-- Text (AnnotationType.TEXT = 1) --> Sticky note / comment popup
|   |
|   +-- Markup (HIGHLIGHT=9, UNDERLINE=10, SQUIGGLY=11, STRIKEOUT=12)
|   |   --> Text markup overlay with optional popup
|   |
|   +-- Stamp (AnnotationType.STAMP = 13) --> Stamp image
|   |
|   +-- FileAttachment (AnnotationType.FILEATTACHMENT = 17)
|       --> Download link for embedded file
|
+-- Need to EDIT annotations? --> Use AnnotationEditorLayer
    +-- AnnotationEditorType.FREETEXT = 3
    +-- AnnotationEditorType.HIGHLIGHT = 9
    +-- AnnotationEditorType.STAMP = 13
    +-- AnnotationEditorType.INK = 15
    +-- AnnotationEditorType.SIGNATURE = 101
```

---

## AnnotationEditorLayer (Creating/Editing Annotations)

```typescript
import { AnnotationEditorLayer, AnnotationEditorType } from "pdfjs-dist";

// AnnotationEditorType constants
// DISABLE = -1   -- Editing disabled
// NONE    =  0   -- No active editor
// FREETEXT = 3   -- Free text editor
// HIGHLIGHT = 9  -- Highlight editor
// STAMP   = 13   -- Stamp editor
// INK     = 15   -- Freehand drawing editor
// SIGNATURE = 101 -- Signature editor

// The AnnotationEditorLayer is managed by AnnotationEditorUIManager
// which coordinates editing across all pages. Key methods:

// annotationEditorLayer.render({ viewport })  -- Setup the editor layer
// annotationEditorLayer.update({ viewport })  -- Update on viewport change
// annotationEditorLayer.updateMode(mode)      -- Switch editor type
// annotationEditorLayer.enable()              -- Activate editing
// annotationEditorLayer.disable()             -- Deactivate, commit changes
```

---

## Required CSS

**ALWAYS** include the PDF.js viewer stylesheet for correct annotation rendering:

```css
/* Import the official PDF.js styles */
@import "pdfjs-dist/web/pdf_viewer.css";

/* ALWAYS ensure correct stacking order */
.textLayer {
  z-index: 1;
}

.annotationLayer {
  z-index: 2;
}

.annotationEditorLayer {
  z-index: 3;
}
```

---

## AnnotationMode Values

| Value | Constant | Use in `page.render()` |
|-------|----------|------------------------|
| 0 | `AnnotationMode.DISABLE` | No annotations rendered on canvas |
| 1 | `AnnotationMode.ENABLE` | Render annotations as static (read-only) |
| 2 | `AnnotationMode.ENABLE_FORMS` | Render with interactive form widgets (default) |
| 3 | `AnnotationMode.ENABLE_STORAGE` | Render with persistent form data via AnnotationStorage |

**ALWAYS** use `AnnotationMode.ENABLE_FORMS` or `AnnotationMode.ENABLE_STORAGE` when rendering forms -- `ENABLE` renders form fields as static images that users cannot interact with.

---

## Reference Links

- [references/methods.md](references/methods.md) -- Complete API signatures for AnnotationLayer, AnnotationStorage, AnnotationEditorLayer, and annotation types
- [references/examples.md](references/examples.md) -- Working code examples for annotation rendering, link handling, and form fields
- [references/anti-patterns.md](references/anti-patterns.md) -- Common annotation layer mistakes and their fixes

### Official Sources

- https://mozilla.github.io/pdf.js/api/ -- PDF.js API reference
- https://github.com/mozilla/pdf.js/blob/master/src/display/annotation_layer.js -- AnnotationLayer source
- https://github.com/mozilla/pdf.js/blob/master/src/display/annotation_storage.js -- AnnotationStorage source
- https://github.com/mozilla/pdf.js/tree/master/examples -- Official examples
