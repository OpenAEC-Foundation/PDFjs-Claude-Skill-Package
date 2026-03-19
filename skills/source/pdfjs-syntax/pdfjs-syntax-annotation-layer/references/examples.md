# Annotation Layer Examples (pdfjs-dist 5.x)

## 1. Complete Page with All Layers (Canvas + Text + Annotations)

Full pipeline for rendering a page with interactive annotations.

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";
import { TextLayer, AnnotationLayer, AnnotationStorage } from "pdfjs-dist";
import type { PDFPageProxy, PageViewport } from "pdfjs-dist";

// ALWAYS configure worker before loading documents
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

async function renderPageWithAnnotations(
  url: string,
  pageNumber: number,
  container: HTMLDivElement
): Promise<void> {
  const doc = await getDocument({ url }).promise;
  const page = await doc.getPage(pageNumber);
  const scale = 1.5;
  const viewport = page.getViewport({ scale });
  const dpr = window.devicePixelRatio || 1;

  // Container setup -- MUST be position: relative
  container.innerHTML = "";
  container.style.position = "relative";
  container.style.width = `${Math.floor(viewport.width)}px`;
  container.style.height = `${Math.floor(viewport.height)}px`;

  // --- Layer 1: Canvas ---
  const canvas = document.createElement("canvas");
  canvas.width = Math.floor(viewport.width * dpr);
  canvas.height = Math.floor(viewport.height * dpr);
  canvas.style.width = `${Math.floor(viewport.width)}px`;
  canvas.style.height = `${Math.floor(viewport.height)}px`;
  canvas.style.position = "absolute";
  canvas.style.top = "0";
  canvas.style.left = "0";
  container.appendChild(canvas);

  const ctx = canvas.getContext("2d")!;
  ctx.scale(dpr, dpr);
  await page.render({ canvasContext: ctx, viewport }).promise;

  // --- Layer 2: Text Layer ---
  const textDiv = document.createElement("div");
  textDiv.className = "textLayer";
  textDiv.style.position = "absolute";
  textDiv.style.top = "0";
  textDiv.style.left = "0";
  container.appendChild(textDiv);

  const textContent = await page.getTextContent();
  const textLayer = new TextLayer({
    container: textDiv,
    textContentSource: textContent,
    viewport: viewport,
  });
  await textLayer.render();

  // --- Layer 3: Annotation Layer (MUST be on top) ---
  const annotationDiv = document.createElement("div");
  annotationDiv.className = "annotationLayer";
  annotationDiv.style.position = "absolute";
  annotationDiv.style.top = "0";
  annotationDiv.style.left = "0";
  container.appendChild(annotationDiv);

  const annotations = await page.getAnnotations({ intent: "display" });
  const annotationStorage = new AnnotationStorage();

  const annotationLayer = new AnnotationLayer({
    div: annotationDiv,
    annotations: annotations,
    page: page,
    viewport: viewport,
    annotationStorage: annotationStorage,
    renderForms: true,
  });

  await annotationLayer.render({ viewport, annotations });
}
```

**Required CSS:**

```css
@import "pdfjs-dist/web/pdf_viewer.css";

.textLayer { z-index: 1; }
.annotationLayer { z-index: 2; }
```

---

## 2. Handling Link Annotations

Detect and handle internal (page navigation) and external (URL) links.

```typescript
import type { PDFPageProxy, PDFDocumentProxy } from "pdfjs-dist";

interface AnnotationData {
  annotationType: number;
  dest?: string | any[];
  url?: string;
  action?: string;
  newWindow?: boolean;
}

async function handleLinkAnnotations(
  page: PDFPageProxy,
  doc: PDFDocumentProxy
): Promise<void> {
  const annotations = await page.getAnnotations({ intent: "display" });

  for (const annotation of annotations) {
    // AnnotationType.LINK === 2
    if (annotation.annotationType !== 2) continue;

    if (annotation.dest) {
      // Internal link -- navigate to a destination within the PDF
      await handleInternalLink(doc, annotation.dest);
    } else if (annotation.url) {
      // External link -- open URL
      handleExternalLink(annotation.url, annotation.newWindow);
    } else if (annotation.action) {
      // Named action -- NextPage, PrevPage, FirstPage, LastPage
      handleNamedAction(annotation.action);
    }
  }
}

async function handleInternalLink(
  doc: PDFDocumentProxy,
  dest: string | any[]
): Promise<void> {
  let resolvedDest = dest;

  // Named destinations need to be resolved first
  if (typeof dest === "string") {
    resolvedDest = await doc.getDestination(dest);
    if (!resolvedDest) return;
  }

  // Explicit destination format: [pageRef, name, ...params]
  // pageRef is a reference object -- resolve to page index
  const destArray = resolvedDest as any[];
  const pageIndex = await doc.getPageIndex(destArray[0]);
  const pageNumber = pageIndex + 1; // Convert 0-based to 1-based

  console.log(`Navigate to page ${pageNumber}`);
  // Implement your page navigation here
}

function handleExternalLink(url: string, newWindow?: boolean): void {
  // ALWAYS validate external URLs before opening
  try {
    const parsed = new URL(url);
    if (parsed.protocol === "http:" || parsed.protocol === "https:") {
      window.open(url, newWindow ? "_blank" : "_self");
    }
  } catch {
    console.warn("Invalid URL in annotation:", url);
  }
}

function handleNamedAction(action: string): void {
  switch (action) {
    case "NextPage":
      console.log("Go to next page");
      break;
    case "PrevPage":
      console.log("Go to previous page");
      break;
    case "FirstPage":
      console.log("Go to first page");
      break;
    case "LastPage":
      console.log("Go to last page");
      break;
    default:
      console.warn("Unknown named action:", action);
  }
}
```

---

## 3. Interactive Form Fields with AnnotationStorage

Render PDF forms and track user input.

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";
import { AnnotationLayer, AnnotationStorage } from "pdfjs-dist";

GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

class PdfFormHandler {
  private storage = new AnnotationStorage();

  constructor() {
    // Track when form data is modified
    this.storage.onSetModified = () => {
      console.log("Form data has been modified");
    };

    this.storage.onResetModified = () => {
      console.log("Form data modifications have been reset");
    };
  }

  async renderFormPage(
    url: string,
    pageNumber: number,
    container: HTMLDivElement
  ): Promise<void> {
    const doc = await getDocument({ url }).promise;
    const page = await doc.getPage(pageNumber);
    const viewport = page.getViewport({ scale: 1.5 });

    // Setup container and canvas (abbreviated -- see Example 1 for full setup)
    container.style.position = "relative";
    // ... canvas setup ...

    const annotationDiv = document.createElement("div");
    annotationDiv.className = "annotationLayer";
    annotationDiv.style.position = "absolute";
    annotationDiv.style.top = "0";
    annotationDiv.style.left = "0";
    container.appendChild(annotationDiv);

    const annotations = await page.getAnnotations({ intent: "display" });

    // ALWAYS pass annotationStorage and renderForms: true for interactive forms
    const annotationLayer = new AnnotationLayer({
      div: annotationDiv,
      annotations: annotations,
      page: page,
      viewport: viewport,
      annotationStorage: this.storage,
      renderForms: true,
    });

    await annotationLayer.render({ viewport, annotations });
  }

  // Get all form field values
  getAllFormData(): Map<string, any> | null {
    return this.storage.getAll();
  }

  // Set a specific field value programmatically
  setFieldValue(annotationId: string, value: any): void {
    this.storage.setValue(annotationId, { value });
  }

  // Get a specific field value
  getFieldValue(annotationId: string): any {
    return this.storage.getValue(annotationId, { value: "" });
  }

  // Check if any form data has been modified
  getModifiedFields(): Set<string> {
    return this.storage.modifiedIds.ids;
  }

  // Get serializable form data for saving
  getSerializableData(): any {
    return this.storage.serializable;
  }
}
```

---

## 4. Filtering Annotations by Type

Process specific annotation types from a page.

```typescript
import { AnnotationType } from "pdfjs-dist";
import type { PDFPageProxy } from "pdfjs-dist";

async function getAnnotationsByType(
  page: PDFPageProxy
): Promise<{
  links: any[];
  forms: any[];
  markups: any[];
  attachments: any[];
  comments: any[];
}> {
  const annotations = await page.getAnnotations({ intent: "display" });

  const links = annotations.filter(
    (a) => a.annotationType === AnnotationType.LINK
  );

  const forms = annotations.filter(
    (a) => a.annotationType === AnnotationType.WIDGET
  );

  const markupTypes = [
    AnnotationType.HIGHLIGHT,
    AnnotationType.UNDERLINE,
    AnnotationType.SQUIGGLY,
    AnnotationType.STRIKEOUT,
  ];
  const markups = annotations.filter(
    (a) => markupTypes.includes(a.annotationType)
  );

  const attachments = annotations.filter(
    (a) => a.annotationType === AnnotationType.FILEATTACHMENT
  );

  const comments = annotations.filter(
    (a) => a.annotationType === AnnotationType.TEXT
  );

  return { links, forms, markups, attachments, comments };
}
```

---

## 5. Extracting Form Field Information

Read form field metadata without rendering.

```typescript
import type { PDFPageProxy } from "pdfjs-dist";
import { AnnotationType } from "pdfjs-dist";

interface FormFieldInfo {
  id: string;
  name: string;
  type: "text" | "checkbox" | "radio" | "select" | "button" | "signature";
  value: any;
  readOnly: boolean;
  options?: { display: string; export: string }[];
}

async function extractFormFields(
  page: PDFPageProxy
): Promise<FormFieldInfo[]> {
  const annotations = await page.getAnnotations({ intent: "display" });
  const fields: FormFieldInfo[] = [];

  for (const ann of annotations) {
    if (ann.annotationType !== AnnotationType.WIDGET) continue;

    let type: FormFieldInfo["type"];

    switch (ann.fieldType) {
      case "Tx":
        type = "text";
        break;
      case "Btn":
        if (ann.checkBox) type = "checkbox";
        else if (ann.radioButton) type = "radio";
        else type = "button";
        break;
      case "Ch":
        type = "select";
        break;
      case "Sig":
        type = "signature";
        break;
      default:
        continue; // Unknown field type
    }

    fields.push({
      id: ann.id,
      name: ann.fieldName || "",
      type,
      value: ann.fieldValue,
      readOnly: ann.readOnly || false,
      options: ann.options?.map((o: any) => ({
        display: o.displayValue,
        export: o.exportValue,
      })),
    });
  }

  return fields;
}
```

---

## 6. Updating Annotation Layer on Viewport Change

Re-position annotations when zoom or rotation changes.

```typescript
import { AnnotationLayer } from "pdfjs-dist";
import type { PDFPageProxy, PageViewport } from "pdfjs-dist";

class AnnotationLayerManager {
  private layer: AnnotationLayer | null = null;
  private annotations: any[] = [];

  async initialize(
    page: PDFPageProxy,
    div: HTMLDivElement,
    viewport: PageViewport
  ): Promise<void> {
    this.annotations = await page.getAnnotations({ intent: "display" });

    this.layer = new AnnotationLayer({
      div: div,
      annotations: this.annotations,
      page: page,
      viewport: viewport,
      renderForms: true,
    });

    await this.layer.render({ viewport, annotations: this.annotations });
  }

  // Call this when scale or rotation changes
  updateViewport(newViewport: PageViewport): void {
    if (!this.layer) return;

    // ALWAYS use update() for viewport changes -- it repositions
    // existing elements without re-creating them
    this.layer.update({
      viewport: newViewport,
      annotations: this.annotations,
    });
  }

  // Call this before removing the layer from DOM
  destroy(): void {
    if (this.layer) {
      this.layer.cancel();
      this.layer = null;
    }
  }
}
```

---

## 7. Print-Specific Annotations

Some annotations appear only when printing. Handle both intents correctly.

```typescript
import { AnnotationLayer, PrintAnnotationStorage } from "pdfjs-dist";
import type { PDFPageProxy, PageViewport } from "pdfjs-dist";

async function renderForPrint(
  page: PDFPageProxy,
  div: HTMLDivElement,
  viewport: PageViewport,
  annotationStorage: AnnotationStorage
): Promise<void> {
  // Use "print" intent to get print-specific annotations
  const annotations = await page.getAnnotations({ intent: "print" });

  // Create a frozen snapshot of form data for printing
  const printStorage = new PrintAnnotationStorage(annotationStorage);

  const annotationLayer = new AnnotationLayer({
    div: div,
    annotations: annotations,
    page: page,
    viewport: viewport,
    annotationStorage: printStorage,
    renderForms: false, // Forms are static when printing
  });

  await annotationLayer.render({ viewport, annotations });
}
```
