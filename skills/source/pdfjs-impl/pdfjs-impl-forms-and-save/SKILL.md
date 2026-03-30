---
name: pdfjs-impl-forms-and-save
description: >
  Use when implementing interactive PDF form filling, reading form field values,
  or saving modified PDFs with user input. Prevents data loss from not persisting
  AnnotationStorage changes and incorrect form field type handling.
  Covers AnnotationStorage API, form field types (text, checkbox, radio, dropdown,
  signature), getFieldObjects(), saveDocument(), and download patterns.
  Keywords: PDF forms, AnnotationStorage, getFieldObjects, saveDocument,
  form filling, AcroForm, fill PDF form, save filled PDF, interactive form,
  read form values, download modified PDF.
license: MIT
compatibility: "Designed for Claude Code. Requires pdfjs-dist 5.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# pdfjs-impl-forms-and-save

## Quick Reference

### Form Architecture

| Component | Purpose | Key API |
|-----------|---------|---------|
| AnnotationLayer | Renders interactive form fields on the page | `AnnotationLayer.render()` |
| AnnotationStorage | Stores user-modified form values in memory | `pdfDoc.annotationStorage` |
| getFieldObjects() | Reads all form field metadata from the PDF | `pdfDoc.getFieldObjects()` |
| saveDocument() | Exports PDF bytes WITH form data embedded | `pdfDoc.saveDocument()` |
| getData() | Exports ORIGINAL PDF bytes WITHOUT modifications | `pdfDoc.getData()` |
| AnnotationEditorLayer | Enables ink, text, stamp, signature editing | `AnnotationEditorLayer.render()` |

### Critical Warnings

**NEVER** use `getData()` to export a filled form -- `getData()` returns the original unmodified PDF bytes. ALWAYS use `saveDocument()` to include AnnotationStorage changes.

**ALWAYS** render the AnnotationLayer with `annotationStorage` linked to the document -- without this, form field changes are never captured.

**NEVER** assume all PDFs use AcroForm -- check for XFA forms first. PDF.js has limited XFA support and silently drops some field types.

**ALWAYS** call `saveDocument()` AFTER the user has finished editing -- AnnotationStorage updates are synchronous, but the save is async.

**NEVER** modify `annotationStorage` directly via internal maps -- ALWAYS use `annotationStorage.setValue(key, value)` to ensure modification tracking works.

**ALWAYS** set `annotationMode` to `AnnotationMode.ENABLE_FORMS` (value `2`) when rendering the AnnotationLayer for interactive forms. The default mode (`1`) renders annotations but disables form interaction.

---

## Essential Patterns

### Rendering Interactive Form Fields

```typescript
import { getDocument, GlobalWorkerOptions, AnnotationLayer } from "pdfjs-dist";
import type { PDFDocumentProxy, PDFPageProxy } from "pdfjs-dist";

// ALWAYS configure worker BEFORE loading documents
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

async function renderFormPage(
  doc: PDFDocumentProxy,
  pageNum: number,
  container: HTMLDivElement
): Promise<void> {
  const page = await doc.getPage(pageNum);
  const scale = 1.5;
  const viewport = page.getViewport({ scale });
  const dpr = window.devicePixelRatio || 1;

  // 1. Render canvas
  const canvas = document.createElement("canvas");
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;
  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);
  container.appendChild(canvas);
  await page.render({ canvasContext: ctx, viewport }).promise;

  // 2. Render annotation layer with forms ENABLED
  const annotationDiv = document.createElement("div");
  annotationDiv.className = "annotationLayer";
  annotationDiv.style.position = "absolute";
  annotationDiv.style.top = "0";
  annotationDiv.style.left = "0";
  annotationDiv.style.width = `${Math.floor(viewport.width)}px`;
  annotationDiv.style.height = `${Math.floor(viewport.height)}px`;
  container.appendChild(annotationDiv);

  const annotations = await page.getAnnotations({ intent: "display" });

  const annotationLayerParams = {
    viewport: viewport.clone({ dontFlip: true }),
    div: annotationDiv,
    annotations,
    page,
    // ALWAYS link to the document's annotationStorage
    annotationStorage: doc.annotationStorage,
    renderForms: true,  // ALWAYS true for interactive forms
  };

  AnnotationLayer.render(annotationLayerParams);
}
```

### Reading All Form Fields

```typescript
async function readFormFields(
  doc: PDFDocumentProxy
): Promise<Map<string, { type: string; value: unknown }>> {
  // getFieldObjects() returns ALL form fields grouped by fully qualified name
  const fieldObjects = await doc.getFieldObjects();
  const result = new Map<string, { type: string; value: unknown }>();

  if (!fieldObjects) {
    // PDF has no AcroForm fields
    return result;
  }

  for (const [fieldName, fields] of Object.entries(fieldObjects)) {
    // Each field name maps to an array (one entry per widget annotation)
    const field = fields[0]; // Primary field object
    result.set(fieldName, {
      type: field.type,   // "text", "checkbox", "radiobutton", "combobox", "listbox", "signature"
      value: field.value, // Current value from the PDF
    });
  }

  return result;
}
```

### Pre-filling Form Fields Programmatically

```typescript
async function prefillForm(
  doc: PDFDocumentProxy,
  values: Record<string, string | boolean>
): Promise<void> {
  const fieldObjects = await doc.getFieldObjects();
  if (!fieldObjects) return;

  for (const [fieldName, fields] of Object.entries(fieldObjects)) {
    if (!(fieldName in values)) continue;

    for (const field of fields) {
      // field.id is the annotation ID used as AnnotationStorage key
      const storageKey = field.id;
      const newValue = values[fieldName];

      // ALWAYS use setValue() -- never write to internal storage directly
      switch (field.type) {
        case "text":
          doc.annotationStorage.setValue(storageKey, { value: String(newValue) });
          break;
        case "checkbox":
          doc.annotationStorage.setValue(storageKey, {
            value: Boolean(newValue),
          });
          break;
        case "radiobutton":
          doc.annotationStorage.setValue(storageKey, { value: String(newValue) });
          break;
        case "combobox":
        case "listbox":
          doc.annotationStorage.setValue(storageKey, { value: String(newValue) });
          break;
      }
    }
  }
}
```

---

## Saving and Downloading

### saveDocument() -- Export with Form Data

```typescript
async function saveFilledPDF(doc: PDFDocumentProxy): Promise<Uint8Array> {
  // saveDocument() serializes AnnotationStorage changes INTO the PDF bytes
  const data = await doc.saveDocument();
  return data; // Uint8Array of the modified PDF
}
```

### Download Pattern

```typescript
async function downloadFilledPDF(
  doc: PDFDocumentProxy,
  filename: string = "filled-form.pdf"
): Promise<void> {
  const data = await doc.saveDocument();
  const blob = new Blob([data], { type: "application/pdf" });
  const url = URL.createObjectURL(blob);

  const link = document.createElement("a");
  link.href = url;
  link.download = filename;
  document.body.appendChild(link);
  link.click();

  // ALWAYS clean up the object URL to prevent memory leaks
  document.body.removeChild(link);
  URL.revokeObjectURL(url);
}
```

### getData() vs saveDocument()

```
Need to export the PDF?
├── User has filled form fields or added annotations?
│   ├── YES → ALWAYS use saveDocument()
│   │         Returns modified PDF with AnnotationStorage changes embedded
│   └── NO  → Use getData()
│             Returns the original unmodified PDF bytes
│
└── Need the raw original bytes regardless of edits?
    └── Use getData() -- ignores all AnnotationStorage modifications
```

---

## Decision Tree: Form Type Handling

```
PDF loaded -- does it have forms?
├── Call getFieldObjects()
│   ├── Returns null → No AcroForm fields. Check for XFA (see below).
│   └── Returns object → AcroForm fields present
│       │
│       ├── field.type === "text"
│       │   → Renders as <input type="text"> or <textarea>
│       │   → Storage value: { value: "string" }
│       │
│       ├── field.type === "checkbox"
│       │   → Renders as <input type="checkbox">
│       │   → Storage value: { value: true/false }
│       │
│       ├── field.type === "radiobutton"
│       │   → Renders as <input type="radio"> (grouped by field name)
│       │   → Storage value: { value: "exportValue" }
│       │
│       ├── field.type === "combobox"
│       │   → Renders as <select> (dropdown)
│       │   → Storage value: { value: "selectedOption" }
│       │
│       ├── field.type === "listbox"
│       │   → Renders as <select multiple>
│       │   → Storage value: { value: "selectedOption" }
│       │
│       └── field.type === "signature"
│           → Renders as placeholder element
│           → PDF.js does NOT support digital signing
│           → Use AnnotationEditorLayer for ink/image signatures
│
├── Check for XFA forms
│   ├── doc.allXfaHtml is non-null → XFA form detected
│   │   → PDF.js renders XFA with LIMITED support
│   │   → NEVER rely on saveDocument() for XFA -- data may be lost
│   │   → Recommend server-side processing for XFA forms
│   └── doc.allXfaHtml is null → Not an XFA form
│
└── Need annotation editing (ink, text, stamps)?
    → Use AnnotationEditorLayer (see below)
```

---

## Annotation Editor Layer

```typescript
import { AnnotationEditorLayer } from "pdfjs-dist";

// AnnotationEditorType values (from pdfjs-dist)
const AnnotationEditorType = {
  DISABLE: -1,
  NONE: 0,
  FREETEXT: 3,
  HIGHLIGHT: 9,
  STAMP: 13,
  INK: 15,
  SIGNATURE: 101,
};

// Enable ink drawing mode on a page
function enableInkEditor(
  page: PDFPageProxy,
  doc: PDFDocumentProxy,
  container: HTMLDivElement,
  scale: number
): void {
  const viewport = page.getViewport({ scale });
  const editorDiv = document.createElement("div");
  editorDiv.className = "annotationEditorLayer";
  editorDiv.style.position = "absolute";
  editorDiv.style.top = "0";
  editorDiv.style.left = "0";
  container.appendChild(editorDiv);

  // The editor layer handles user interaction for adding annotations
  // Switch modes by setting the editor type on the viewer's event bus
  // ALWAYS render AFTER the annotation layer
}
```

---

## Detecting Form Presence

```typescript
async function detectFormType(
  doc: PDFDocumentProxy
): Promise<"acroform" | "xfa" | "none"> {
  // Check XFA first -- XFA takes priority
  const xfaHtml = await doc.allXfaHtml;
  if (xfaHtml) return "xfa";

  // Check AcroForm
  const fields = await doc.getFieldObjects();
  if (fields && Object.keys(fields).length > 0) return "acroform";

  return "none";
}
```

---

## Required CSS

```css
@import "pdfjs-dist/web/pdf_viewer.css";

.annotationLayer {
  position: absolute;
  top: 0;
  left: 0;
  z-index: 2;
}

/* Form field styling */
.annotationLayer input[type="text"],
.annotationLayer textarea {
  border: 1px solid transparent;
  background: rgba(0, 84, 255, 0.13);
}

.annotationLayer input[type="text"]:focus,
.annotationLayer textarea:focus {
  border-color: #0054ff;
  outline: none;
}

.annotationLayer input[type="checkbox"],
.annotationLayer input[type="radio"] {
  cursor: pointer;
}

.annotationLayer select {
  background: rgba(0, 84, 255, 0.13);
  border: 1px solid transparent;
}
```

---

## Reference Links

- [references/methods.md](references/methods.md) -- AnnotationStorage API, getFieldObjects() return types, saveDocument() details
- [references/examples.md](references/examples.md) -- Complete form reading, filling, saving, and download workflows
- [references/anti-patterns.md](references/anti-patterns.md) -- Common form handling mistakes with WRONG/CORRECT code

### Official Sources

- https://mozilla.github.io/pdf.js/api/ -- PDF.js API reference
- https://github.com/mozilla/pdf.js/blob/master/src/display/annotation_storage.js -- AnnotationStorage source
- https://github.com/mozilla/pdf.js/blob/master/src/display/api.js -- saveDocument, getFieldObjects
- https://github.com/mozilla/pdf.js/blob/master/src/display/annotation_layer.js -- Form field rendering
