# Forms and Save Anti-Patterns (pdfjs-dist 5.x)

Common mistakes when implementing PDF form filling and saving, and how to fix them.

---

## 1. Using getData() Instead of saveDocument()

**Severity**: Critical -- causes complete loss of all form data entered by the user.

### Wrong

```typescript
// BROKEN: getData() returns the ORIGINAL PDF without any modifications
async function savePDF(doc: PDFDocumentProxy): Promise<void> {
  const data = await doc.getData(); // Returns original bytes!
  const blob = new Blob([data], { type: "application/pdf" });
  downloadBlob(blob, "form.pdf");
  // User opens the downloaded file and ALL form entries are gone
}
```

### Correct

```typescript
// ALWAYS use saveDocument() to include AnnotationStorage modifications
async function savePDF(doc: PDFDocumentProxy): Promise<void> {
  const data = await doc.saveDocument(); // Includes form field values
  const blob = new Blob([data], { type: "application/pdf" });
  downloadBlob(blob, "form.pdf");
}
```

**Why**: `getData()` returns the raw PDF bytes as originally loaded. It has no knowledge of AnnotationStorage. Only `saveDocument()` serializes the stored form values back into the PDF structure.

---

## 2. Rendering Forms Without annotationStorage

**Severity**: Critical -- form fields render but user edits are never captured.

### Wrong

```typescript
// BROKEN: No annotationStorage linked -- edits are lost
AnnotationLayer.render({
  viewport: viewport.clone({ dontFlip: true }),
  div: annotationDiv,
  annotations,
  page,
  // Missing: annotationStorage
  renderForms: true,
});

// Later: saveDocument() returns empty form because nothing was stored
const data = await doc.saveDocument(); // No form data!
```

### Correct

```typescript
// ALWAYS link annotationStorage from the document
AnnotationLayer.render({
  viewport: viewport.clone({ dontFlip: true }),
  div: annotationDiv,
  annotations,
  page,
  annotationStorage: doc.annotationStorage, // ALWAYS include this
  renderForms: true,
});

// Now saveDocument() includes all user edits
const data = await doc.saveDocument(); // Form data preserved
```

**Why**: The AnnotationLayer needs a reference to the document's AnnotationStorage to write form field changes. Without it, the rendered HTML inputs work visually but their values are never persisted to storage.

---

## 3. Using Wrong Key Format for AnnotationStorage

**Severity**: High -- form values are stored but not associated with the correct field, causing silent data loss on save.

### Wrong

```typescript
// BROKEN: Using field NAME as the storage key
const fields = await doc.getFieldObjects();
for (const [name, fieldArray] of Object.entries(fields!)) {
  doc.annotationStorage.setValue(name, { value: "test" });
  // AnnotationStorage uses annotation IDs like "24R", not field names!
}
```

### Correct

```typescript
// ALWAYS use field.id (the annotation ID) as the storage key
const fields = await doc.getFieldObjects();
for (const [name, fieldArray] of Object.entries(fields!)) {
  for (const field of fieldArray) {
    doc.annotationStorage.setValue(field.id, { value: "test" });
    // field.id = "24R" (the annotation reference ID)
  }
}
```

**Why**: AnnotationStorage maps annotation IDs (e.g., `"24R"`) to values. Field names (e.g., `"applicant.name"`) are human-readable identifiers but are NOT used as storage keys. Using the wrong key means `saveDocument()` cannot match the stored value to any annotation in the PDF.

---

## 4. Not Checking for XFA Before Saving

**Severity**: High -- `saveDocument()` on XFA forms produces corrupted or incomplete output.

### Wrong

```typescript
// BROKEN: Blindly calling saveDocument() on any PDF with forms
async function saveAnyForm(doc: PDFDocumentProxy): Promise<Uint8Array> {
  return await doc.saveDocument();
  // If this is an XFA form, the saved PDF may be missing fields,
  // have incorrect values, or fail to open in other readers
}
```

### Correct

```typescript
// ALWAYS check for XFA before saving
async function saveFormSafely(doc: PDFDocumentProxy): Promise<Uint8Array> {
  const xfaHtml = await doc.allXfaHtml;
  if (xfaHtml) {
    throw new Error(
      "XFA forms are not fully supported by PDF.js. " +
      "Use server-side processing (e.g., Adobe PDF Services) for reliable XFA saving."
    );
  }

  return await doc.saveDocument();
}
```

**Why**: PDF.js has limited XFA form support. While it can render many XFA forms for display, the `saveDocument()` method does not reliably serialize XFA data modifications back into the PDF structure. Users may lose data without any error or warning.

---

## 5. Setting Checkbox Values as Strings

**Severity**: Medium -- checkbox appears checked in the viewer but is not saved correctly.

### Wrong

```typescript
// BROKEN: Using string "true" instead of boolean true
doc.annotationStorage.setValue(field.id, { value: "true" });
// The checkbox may render incorrectly or save with the wrong state

// Also wrong: using 1/0 instead of boolean
doc.annotationStorage.setValue(field.id, { value: 1 });
```

### Correct

```typescript
// ALWAYS use boolean true/false for checkbox fields
doc.annotationStorage.setValue(field.id, { value: true });

// For radio buttons, use the export value STRING
doc.annotationStorage.setValue(radioField.id, { value: "Yes" });
```

**Why**: Checkboxes in PDF.js expect a boolean value in AnnotationStorage. Using a string `"true"` or a number `1` may cause type mismatches during serialization. Radio buttons, conversely, expect a string matching one of the defined export values.

---

## 6. Not Cleaning Up Object URLs After Download

**Severity**: Medium -- causes memory leaks, especially when users save multiple times.

### Wrong

```typescript
// BROKEN: Object URL is never revoked
async function downloadPDF(doc: PDFDocumentProxy): Promise<void> {
  const data = await doc.saveDocument();
  const blob = new Blob([data], { type: "application/pdf" });
  const url = URL.createObjectURL(blob);

  const link = document.createElement("a");
  link.href = url;
  link.download = "form.pdf";
  link.click();
  // url is never revoked -- blob stays in memory forever
  // Clicking "Save" 10 times = 10 PDF-sized blobs leaked
}
```

### Correct

```typescript
// ALWAYS revoke the object URL after the download starts
async function downloadPDF(doc: PDFDocumentProxy): Promise<void> {
  const data = await doc.saveDocument();
  const blob = new Blob([data], { type: "application/pdf" });
  const url = URL.createObjectURL(blob);

  const link = document.createElement("a");
  link.href = url;
  link.download = "form.pdf";
  document.body.appendChild(link);
  link.click();

  document.body.removeChild(link);
  URL.revokeObjectURL(url); // Free the blob memory
}
```

**Why**: `URL.createObjectURL()` creates a persistent reference to the blob in memory. Without `URL.revokeObjectURL()`, the blob is never garbage collected. For a 10 MB PDF saved 10 times, this leaks 100 MB of memory.

---

## 7. Assuming getFieldObjects() Always Returns Data

**Severity**: Medium -- causes runtime crashes on PDFs without forms.

### Wrong

```typescript
// BROKEN: No null check -- crashes on PDFs without forms
async function listFields(doc: PDFDocumentProxy): Promise<void> {
  const fields = await doc.getFieldObjects();
  // TypeError: Cannot convert undefined or null to object
  for (const [name, fieldArray] of Object.entries(fields)) {
    console.log(name, fieldArray[0].value);
  }
}
```

### Correct

```typescript
// ALWAYS check for null before iterating
async function listFields(doc: PDFDocumentProxy): Promise<void> {
  const fields = await doc.getFieldObjects();
  if (!fields) {
    console.log("No form fields found in this PDF");
    return;
  }

  for (const [name, fieldArray] of Object.entries(fields)) {
    if (fieldArray.length > 0) {
      console.log(name, fieldArray[0].value);
    }
  }
}
```

**Why**: `getFieldObjects()` returns `null` (not an empty object) when the PDF has no AcroForm dictionary. Calling `Object.entries(null)` throws a TypeError.

---

## 8. Rendering Forms with renderForms: false

**Severity**: Medium -- form fields are visible but completely non-interactive.

### Wrong

```typescript
// BROKEN: Forms render as static elements -- user cannot type or click
AnnotationLayer.render({
  viewport: viewport.clone({ dontFlip: true }),
  div: annotationDiv,
  annotations,
  page,
  annotationStorage: doc.annotationStorage,
  renderForms: false, // Renders non-interactive form widgets
});
// User sees form fields but cannot interact with them
// No error is thrown -- it just silently doesn't work
```

### Correct

```typescript
// Set renderForms: true for interactive form fields
AnnotationLayer.render({
  viewport: viewport.clone({ dontFlip: true }),
  div: annotationDiv,
  annotations,
  page,
  annotationStorage: doc.annotationStorage,
  renderForms: true, // ALWAYS true when users need to fill forms
});
```

**Why**: With `renderForms: false`, PDF.js renders form widget annotations as static visual elements (like stamps). The fields look correct but do not accept user input. This is useful for print previews or read-only displays, but NEVER for form filling workflows.
