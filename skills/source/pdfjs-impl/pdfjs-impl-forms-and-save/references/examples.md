# Forms and Save Examples (pdfjs-dist 5.x)

## Complete Form Reader

Read all form fields from a PDF and display their current values.

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";
import type { PDFDocumentProxy } from "pdfjs-dist";

GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

interface FormField {
  name: string;
  type: string;
  value: string | boolean | null;
  page: number;
  readOnly: boolean;
  options?: { exportValue: string; displayValue: string }[];
}

async function readAllFormFields(pdfUrl: string): Promise<FormField[]> {
  const doc = await getDocument({ url: pdfUrl }).promise;
  const fields = await doc.getFieldObjects();
  const result: FormField[] = [];

  if (!fields) {
    console.log("This PDF has no AcroForm fields.");
    return result;
  }

  for (const [name, fieldArray] of Object.entries(fields)) {
    const f = fieldArray[0];
    result.push({
      name,
      type: f.type,
      value: f.value,
      page: f.page,
      readOnly: f.readOnly,
      options: f.options,
    });
  }

  return result;
}
```

---

## Complete Form Filler and Saver

Load a PDF, pre-fill form fields, render for user editing, then save.

```typescript
import { getDocument, GlobalWorkerOptions, AnnotationLayer } from "pdfjs-dist";
import type { PDFDocumentProxy, PDFPageProxy } from "pdfjs-dist";

GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

class PDFFormHandler {
  private doc: PDFDocumentProxy | null = null;

  async load(source: string | ArrayBuffer): Promise<void> {
    const params = typeof source === "string" ? { url: source } : { data: source };
    this.doc = await getDocument(params).promise;
  }

  // Pre-fill fields before rendering
  async prefill(values: Record<string, string | boolean>): Promise<void> {
    if (!this.doc) throw new Error("Document not loaded");

    const fields = await this.doc.getFieldObjects();
    if (!fields) throw new Error("PDF has no form fields");

    for (const [fieldName, fieldArray] of Object.entries(fields)) {
      if (!(fieldName in values)) continue;

      for (const field of fieldArray) {
        const val = values[fieldName];
        switch (field.type) {
          case "text":
            this.doc.annotationStorage.setValue(field.id, { value: String(val) });
            break;
          case "checkbox":
            this.doc.annotationStorage.setValue(field.id, { value: Boolean(val) });
            break;
          case "radiobutton":
            this.doc.annotationStorage.setValue(field.id, { value: String(val) });
            break;
          case "combobox":
          case "listbox":
            this.doc.annotationStorage.setValue(field.id, { value: String(val) });
            break;
        }
      }
    }
  }

  // Render a page with interactive form fields
  async renderPage(
    pageNum: number,
    container: HTMLDivElement,
    scale: number = 1.5
  ): Promise<void> {
    if (!this.doc) throw new Error("Document not loaded");

    const page = await this.doc.getPage(pageNum);
    const viewport = page.getViewport({ scale });
    const dpr = window.devicePixelRatio || 1;

    // Clear container
    container.innerHTML = "";
    container.style.position = "relative";
    container.style.width = `${Math.floor(viewport.width)}px`;
    container.style.height = `${Math.floor(viewport.height)}px`;

    // Canvas layer
    const canvas = document.createElement("canvas");
    canvas.width = Math.floor(viewport.width * dpr);
    canvas.height = Math.floor(viewport.height * dpr);
    canvas.style.width = `${Math.floor(viewport.width)}px`;
    canvas.style.height = `${Math.floor(viewport.height)}px`;
    canvas.style.position = "absolute";
    canvas.style.top = "0";
    canvas.style.left = "0";
    const ctx = canvas.getContext("2d")!;
    ctx.scale(dpr, dpr);
    container.appendChild(canvas);

    await page.render({ canvasContext: ctx, viewport }).promise;

    // Annotation layer with forms
    const annotationDiv = document.createElement("div");
    annotationDiv.className = "annotationLayer";
    annotationDiv.style.position = "absolute";
    annotationDiv.style.top = "0";
    annotationDiv.style.left = "0";
    annotationDiv.style.width = `${Math.floor(viewport.width)}px`;
    annotationDiv.style.height = `${Math.floor(viewport.height)}px`;
    container.appendChild(annotationDiv);

    const annotations = await page.getAnnotations({ intent: "display" });

    AnnotationLayer.render({
      viewport: viewport.clone({ dontFlip: true }),
      div: annotationDiv,
      annotations,
      page,
      annotationStorage: this.doc.annotationStorage,
      renderForms: true,
    });
  }

  // Read current form values (including user edits)
  async getCurrentValues(): Promise<Record<string, unknown>> {
    if (!this.doc) throw new Error("Document not loaded");

    const fields = await this.doc.getFieldObjects();
    if (!fields) return {};

    const result: Record<string, unknown> = {};

    for (const [name, fieldArray] of Object.entries(fields)) {
      const field = fieldArray[0];
      // Check AnnotationStorage first (user modifications)
      const stored = this.doc.annotationStorage.getRawValue(field.id);
      if (stored !== undefined) {
        result[name] = stored.value;
      } else {
        // Fall back to original PDF value
        result[name] = field.value;
      }
    }

    return result;
  }

  // Save the filled PDF
  async save(): Promise<Uint8Array> {
    if (!this.doc) throw new Error("Document not loaded");
    return await this.doc.saveDocument();
  }

  // Download the filled PDF
  async download(filename: string = "filled-form.pdf"): Promise<void> {
    const data = await this.save();
    const blob = new Blob([data], { type: "application/pdf" });
    const url = URL.createObjectURL(blob);

    const link = document.createElement("a");
    link.href = url;
    link.download = filename;
    document.body.appendChild(link);
    link.click();

    document.body.removeChild(link);
    URL.revokeObjectURL(url);
  }

  // Clean up resources
  async destroy(): Promise<void> {
    if (this.doc) {
      await this.doc.destroy();
      this.doc = null;
    }
  }
}
```

### Usage

```typescript
const handler = new PDFFormHandler();
await handler.load("/forms/application.pdf");

// Pre-fill known values
await handler.prefill({
  "applicant.name": "Jane Smith",
  "applicant.email": "jane@example.com",
  "terms.agreed": true,
  "department": "Engineering",
});

// Render all pages for user editing
const doc = handler["doc"]!;
for (let i = 1; i <= doc.numPages; i++) {
  const container = document.createElement("div");
  document.getElementById("viewer")!.appendChild(container);
  await handler.renderPage(i, container);
}

// Save button
document.getElementById("save-btn")!.addEventListener("click", async () => {
  const values = await handler.getCurrentValues();
  console.log("Form data:", values);
  await handler.download("completed-application.pdf");
});
```

---

## Form Change Detection

Track when a user modifies any form field.

```typescript
function setupFormChangeTracking(doc: PDFDocumentProxy): void {
  // Track modification state changes
  doc.annotationStorage.onSetModified = () => {
    console.log("Form has been modified");
    document.getElementById("save-btn")!.disabled = false;
    document.getElementById("save-status")!.textContent = "Unsaved changes";
  };

  doc.annotationStorage.onResetModified = () => {
    console.log("Form modifications cleared");
    document.getElementById("save-btn")!.disabled = true;
    document.getElementById("save-status")!.textContent = "All changes saved";
  };
}

// After saving, reset the modification flag
async function saveAndReset(doc: PDFDocumentProxy): Promise<void> {
  const data = await doc.saveDocument();
  // ... download or upload `data` ...
  doc.annotationStorage.resetModified();
}
```

---

## XFA Form Detection

Detect and handle XFA forms (which have limited support in PDF.js).

```typescript
async function handleFormType(doc: PDFDocumentProxy): Promise<void> {
  // Check XFA first
  const xfaHtml = await doc.allXfaHtml;
  if (xfaHtml) {
    console.warn(
      "This PDF uses XFA forms. PDF.js has limited XFA support. " +
      "Form saving may not preserve all data. " +
      "Consider server-side processing with Adobe tools for full XFA support."
    );
    // XFA forms are rendered differently -- PDF.js generates HTML from XFA data
    return;
  }

  // Check AcroForm
  const fields = await doc.getFieldObjects();
  if (fields && Object.keys(fields).length > 0) {
    console.log(`AcroForm detected with ${Object.keys(fields).length} fields`);
    // Safe to use saveDocument() for AcroForm PDFs
    return;
  }

  console.log("No interactive form fields found in this PDF");
}
```

---

## Upload Saved PDF to Server

```typescript
async function uploadFilledPDF(
  doc: PDFDocumentProxy,
  uploadUrl: string
): Promise<Response> {
  const data = await doc.saveDocument();
  const blob = new Blob([data], { type: "application/pdf" });

  const formData = new FormData();
  formData.append("file", blob, "filled-form.pdf");

  return fetch(uploadUrl, {
    method: "POST",
    body: formData,
  });
}
```

---

## Read-Only Form Rendering

Render form fields as non-interactive (display-only).

```typescript
async function renderReadOnlyForm(
  page: PDFPageProxy,
  doc: PDFDocumentProxy,
  container: HTMLDivElement,
  scale: number
): Promise<void> {
  const viewport = page.getViewport({ scale });
  const annotations = await page.getAnnotations({ intent: "display" });

  const annotationDiv = document.createElement("div");
  annotationDiv.className = "annotationLayer";
  container.appendChild(annotationDiv);

  AnnotationLayer.render({
    viewport: viewport.clone({ dontFlip: true }),
    div: annotationDiv,
    annotations,
    page,
    annotationStorage: doc.annotationStorage,
    renderForms: false, // false = read-only form display
  });
}
```
