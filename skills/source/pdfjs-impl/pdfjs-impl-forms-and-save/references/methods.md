# Forms and Save API Reference (pdfjs-dist 5.x)

## AnnotationStorage

The `AnnotationStorage` class stores user modifications to form fields and annotations. It is accessed via `pdfDoc.annotationStorage` -- NEVER instantiate it directly.

### Methods

#### setValue(key: string, value: object): void

Stores a value for the given annotation ID key. Triggers the `onSetModified` callback on first modification.

```typescript
// Text field
doc.annotationStorage.setValue("24R", { value: "John Doe" });

// Checkbox
doc.annotationStorage.setValue("30R", { value: true });

// Radio button -- value is the export value of the selected option
doc.annotationStorage.setValue("35R", { value: "option2" });

// Dropdown/combobox
doc.annotationStorage.setValue("40R", { value: "Selected Item" });
```

The `key` parameter is the annotation's ID string (e.g., `"24R"`). This ID comes from `field.id` in the `getFieldObjects()` result or from `annotation.id` in the annotations array.

#### getValue(key: string, defaultValue: object): object

Returns the stored value for the key, or `defaultValue` if no value has been set.

```typescript
const val = doc.annotationStorage.getValue("24R", { value: "" });
console.log(val.value); // The stored text, or "" if unmodified
```

#### getRawValue(key: string): object | undefined

Returns the raw stored value without merging with any default. Returns `undefined` if no value has been set.

```typescript
const raw = doc.annotationStorage.getRawValue("24R");
if (raw !== undefined) {
  console.log("Field was modified:", raw.value);
}
```

#### has(key: string): boolean

Returns `true` if a value has been stored for the given key.

```typescript
if (doc.annotationStorage.has("24R")) {
  console.log("Field 24R has been modified");
}
```

#### remove(key: string): void

Deletes the stored value for the given key. If storage becomes empty after removal, the modified state is reset.

```typescript
doc.annotationStorage.remove("24R");
```

#### resetModified(): void

Clears the modification flag. After calling this, `onResetModified` is invoked.

```typescript
doc.annotationStorage.resetModified();
```

### Properties

#### size: number (getter)

Returns the number of stored entries.

```typescript
console.log(`${doc.annotationStorage.size} fields modified`);
```

#### serializable (getter)

Returns a frozen object containing the storage map, hash digest, and bitmap transfer array. Used internally by `saveDocument()`.

#### print (getter)

Returns a `PrintAnnotationStorage` instance with frozen data, used for printing with form values included.

### Callbacks

#### onSetModified: (() => void) | null

Called when the storage transitions from unmodified to modified state. Use this to enable a "Save" button.

```typescript
doc.annotationStorage.onSetModified = () => {
  document.getElementById("save-btn")!.disabled = false;
};
```

#### onResetModified: (() => void) | null

Called when the storage transitions from modified to unmodified state (after `resetModified()` or when all entries are removed).

```typescript
doc.annotationStorage.onResetModified = () => {
  document.getElementById("save-btn")!.disabled = true;
};
```

### Iteration

AnnotationStorage implements `[Symbol.iterator]()`, enabling `for...of` loops:

```typescript
for (const [key, value] of doc.annotationStorage) {
  console.log(`Annotation ${key}:`, value);
}
```

---

## PDFDocumentProxy.getFieldObjects()

### Signature

```typescript
getFieldObjects(): Promise<Record<string, FieldObject[]> | null>
```

Returns a promise that resolves to an object mapping fully qualified field names to arrays of field objects, or `null` if the PDF has no AcroForm fields.

### FieldObject Properties

| Property | Type | Description |
|----------|------|-------------|
| `id` | `string` | Annotation ID -- use as AnnotationStorage key |
| `value` | `string \| boolean \| null` | Current field value from the PDF |
| `type` | `string` | Field type: `"text"`, `"checkbox"`, `"radiobutton"`, `"combobox"`, `"listbox"`, `"signature"` |
| `name` | `string` | Fully qualified field name |
| `rect` | `number[]` | Bounding rectangle `[x1, y1, x2, y2]` |
| `page` | `number` | Zero-based page index where the field appears |
| `multiline` | `boolean` | Whether a text field supports multiple lines |
| `maxLen` | `number \| null` | Maximum character count for text fields |
| `readOnly` | `boolean` | Whether the field is read-only |
| `hidden` | `boolean` | Whether the field is hidden |
| `exportValues` | `string[]` | Available export values (radio buttons, checkboxes) |
| `editable` | `boolean` | Whether a combobox allows custom text entry |
| `options` | `{ exportValue: string; displayValue: string }[]` | Options for combobox/listbox fields |

### Example: Listing All Fields

```typescript
const fields = await doc.getFieldObjects();
if (fields) {
  for (const [name, fieldArray] of Object.entries(fields)) {
    const f = fieldArray[0];
    console.log(`${name} (${f.type}): "${f.value}" [page ${f.page}]`);
    if (f.type === "combobox" || f.type === "listbox") {
      console.log("  Options:", f.options?.map(o => o.displayValue));
    }
  }
}
```

---

## PDFDocumentProxy.saveDocument()

### Signature

```typescript
saveDocument(): Promise<Uint8Array>
```

Returns a promise that resolves to a `Uint8Array` containing the modified PDF bytes. All values stored in `annotationStorage` are serialized into the PDF structure.

### What saveDocument() Includes

- All form field values set via AnnotationStorage
- Annotation editor additions (ink, text, stamps) if AnnotationEditorLayer was used
- Original PDF content unchanged where no modifications were made

### What saveDocument() Does NOT Include

- XFA form modifications (limited/unreliable support)
- Digital signatures (PDF.js cannot sign documents)
- Structural PDF changes (adding/removing pages)

---

## PDFDocumentProxy.getData()

### Signature

```typescript
getData(): Promise<Uint8Array>
```

Returns the ORIGINAL PDF bytes as loaded. This method ignores ALL modifications made via AnnotationStorage. Use only when you need the unmodified source document.

---

## AnnotationEditorType Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `DISABLE` | `-1` | Disable all editor functionality |
| `NONE` | `0` | No active editor mode |
| `FREETEXT` | `3` | Text annotation editor |
| `HIGHLIGHT` | `9` | Text highlight editor |
| `STAMP` | `13` | Image stamp editor |
| `INK` | `15` | Freehand ink drawing editor |
| `SIGNATURE` | `101` | Signature drawing editor |

---

## AnnotationMode Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `DISABLE` | `0` | Do not render annotations |
| `ENABLE` | `1` | Render annotations (default, non-interactive) |
| `ENABLE_FORMS` | `2` | Render annotations WITH interactive form fields |
| `ENABLE_STORAGE` | `3` | Render annotations with storage for programmatic access |
