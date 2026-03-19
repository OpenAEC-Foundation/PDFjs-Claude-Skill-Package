# API Signatures Reference (pdfjs-dist 5.x Annotation Layer)

## PDFPageProxy.getAnnotations()

Retrieves all annotation data for a page.

```typescript
getAnnotations(params?: {
  intent?: string;  // "display" | "print" | "any"
                    // "display" -- annotations visible on screen (default)
                    // "print" -- annotations visible when printing
                    // "any" -- all annotations regardless of visibility
}): Promise<AnnotationData[]>
```

**Notes**:
- ALWAYS specify `intent` explicitly -- the default may vary between versions
- Some annotations have `annotationFlags` that control visibility per intent (e.g., PRINT flag, HIDDEN flag)
- The returned array contains raw annotation data objects, NOT rendered elements

---

## AnnotationData

The raw data structure returned by `getAnnotations()`. Key properties:

```typescript
interface AnnotationData {
  id: string;                    // Unique annotation identifier
  annotationType: number;        // AnnotationType constant (1=TEXT, 2=LINK, 20=WIDGET, etc.)
  rect: [number, number, number, number]; // Bounding rectangle [x1, y1, x2, y2] in PDF coordinates
  subtype: string;               // Annotation subtype string ("Link", "Text", "Widget", etc.)
  annotationFlags: number;       // Bitfield from AnnotationFlag constants
  color: Uint8ClampedArray;      // RGB color array
  hasAppearance: boolean;        // Whether the annotation has a custom appearance stream

  // Link-specific
  url?: string;                  // External URL (link annotations)
  dest?: string | any[];         // Named destination or explicit destination (link annotations)
  action?: string;               // Named action ("GoTo", "NextPage", "PrevPage", etc.)
  newWindow?: boolean;           // Open link in new window

  // Widget/form-specific
  fieldName?: string;            // Form field name
  fieldType?: string;            // "Tx" (text), "Btn" (button), "Ch" (choice), "Sig" (signature)
  fieldValue?: string | string[];// Current field value
  alternativeText?: string;      // Alt text for accessibility
  defaultAppearance?: string;    // Default appearance string for text fields
  readOnly?: boolean;            // Whether field is read-only
  checkBox?: boolean;            // Whether button is a checkbox
  radioButton?: boolean;         // Whether button is a radio button
  pushButton?: boolean;          // Whether button is a push button
  multiLine?: boolean;           // Whether text field is multiline
  maxLen?: number;               // Maximum text length
  options?: { displayValue: string; exportValue: string }[]; // Choice field options

  // Markup annotation-specific
  contents?: string;             // Text content / comment body
  title?: string;                // Author / title
  creationDate?: string;         // ISO date string
  modificationDate?: string;     // ISO date string
  hasPopup?: boolean;            // Whether annotation has an associated popup
  popupRef?: string;             // Reference to popup annotation ID

  // FileAttachment-specific
  file?: { filename: string; content: Uint8Array }; // Attached file data
}
```

---

## AnnotationLayer

Main class for rendering annotation HTML elements over a PDF page.

### Constructor

```typescript
new AnnotationLayer(params: {
  div: HTMLDivElement;                    // Container element for annotations
  annotations: AnnotationData[];         // From page.getAnnotations()
  page: PDFPageProxy;                    // The PDF page object
  viewport: PageViewport;                // Current viewport for positioning
  annotationStorage?: AnnotationStorage; // For form data persistence
  linkService?: object;                  // For handling link navigation
  downloadManager?: object;             // For file attachment downloads
  renderForms?: boolean;                 // true = interactive forms (default: false)
  imageResourcesPath?: string;           // Path to annotation icon images
  pageColors?: {                         // Accessibility color overrides
    background?: string;
    foreground?: string;
  };
})
```

### Methods

```typescript
// Render all annotations as HTML elements inside the div
render(params: {
  viewport: PageViewport;
  annotations: AnnotationData[];
}): Promise<void>

// Update annotations when viewport changes (zoom, rotation)
update(params: {
  viewport: PageViewport;
  annotations?: AnnotationData[];
}): void

// Cancel any pending annotation rendering
cancel(): void
```

**Notes**:
- `render()` creates HTML elements (links, form inputs, etc.) positioned over the canvas
- `update()` repositions existing elements without re-creating them -- use on zoom/rotation
- ALWAYS call `cancel()` before removing the annotation layer from the DOM

---

## AnnotationStorage

Stores form field values and editor state for the current document session.

### Constructor

```typescript
new AnnotationStorage()
```

### Core Methods

```typescript
// Get value for an annotation, with a fallback default
getValue(key: string, defaultValue: any): any

// Get raw value without default fallback
getRawValue(key: string): any | undefined

// Set a value for an annotation (triggers modification tracking)
setValue(key: string, value: any): void

// Check if a value exists
has(key: string): boolean

// Remove a stored value
remove(key: string): void

// Get all stored values as a Map
getAll(): Map<string, any> | null
```

### State Management

```typescript
// Reset the modification state (marks storage as unmodified)
resetModified(): void

// Get serializable snapshot for saving/printing
get serializable: { map: Map<string, any>; hash: string; transfer: any[] }

// Get IDs of modified annotations
get modifiedIds: { ids: Set<string>; hash: string }

// Get editor statistics (telemetry)
get editorStats: Map<string, any> | null
```

### Callbacks

```typescript
// Set callback for when storage is modified
set onSetModified(callback: () => void)

// Set callback for when modification state is reset
set onResetModified(callback: () => void)

// Set callback for editor type changes
set onAnnotationEditor(callback: (type: number) => void)
```

---

## PrintAnnotationStorage

Extends `AnnotationStorage` for synchronized printing. Creates a frozen snapshot of the parent storage.

```typescript
new PrintAnnotationStorage(parent: AnnotationStorage)

// Returns frozen serializable data (prevents modification during print)
get serializable: { map: Map<string, any>; hash: string; transfer: any[] }

// ALWAYS returns empty set (printing does not track modifications)
get modifiedIds: { ids: Set<string>; hash: string }
```

---

## AnnotationEditorLayer

Layer for creating and editing annotations. Managed by `AnnotationEditorUIManager`.

### Constructor

```typescript
new AnnotationEditorLayer(params: {
  uiManager: AnnotationEditorUIManager;  // Central editor manager
  pageIndex: number;                      // Zero-based page index
  div: HTMLDivElement;                    // Container element
  viewport: PageViewport;                 // Current viewport
  accessibilityManager?: object;         // Accessibility support
  annotationLayer?: AnnotationLayer;     // Existing annotation layer
  drawLayer?: object;                    // Drawing support layer
  textLayer?: object;                    // Text layer reference
  l10n?: object;                         // Localization
})
```

### Key Methods

```typescript
// Setup the layer with viewport dimensions
render(params: { viewport: PageViewport }): void

// Update viewport (zoom, rotation changes)
update(params: { viewport: PageViewport }): void

// Switch editor mode (FREETEXT, INK, HIGHLIGHT, STAMP, SIGNATURE)
updateMode(mode: number): void

// Activate editing on this layer
enable(): void

// Deactivate editing, commit pending changes
disable(): void

// Add an editor to the layer
add(editor: AnnotationEditor): void

// Remove an editor from the layer
remove(editor: AnnotationEditor): void
```

---

## AnnotationEditorType Constants

```typescript
import { AnnotationEditorType } from "pdfjs-dist";

AnnotationEditorType.DISABLE;    // -1 -- Editing disabled
AnnotationEditorType.NONE;       //  0 -- No active editor mode
AnnotationEditorType.FREETEXT;   //  3 -- Free text editor
AnnotationEditorType.HIGHLIGHT;  //  9 -- Highlight editor
AnnotationEditorType.STAMP;      // 13 -- Stamp editor
AnnotationEditorType.INK;        // 15 -- Freehand drawing editor
AnnotationEditorType.SIGNATURE;  // 101 -- Signature editor
```

---

## AnnotationType Constants

```typescript
import { AnnotationType } from "pdfjs-dist";

AnnotationType.TEXT;            //  1
AnnotationType.LINK;            //  2
AnnotationType.FREETEXT;        //  3
AnnotationType.LINE;            //  4
AnnotationType.SQUARE;          //  5
AnnotationType.CIRCLE;          //  6
AnnotationType.POLYGON;         //  7
AnnotationType.POLYLINE;        //  8
AnnotationType.HIGHLIGHT;       //  9
AnnotationType.UNDERLINE;       // 10
AnnotationType.SQUIGGLY;        // 11
AnnotationType.STRIKEOUT;       // 12
AnnotationType.STAMP;           // 13
AnnotationType.CARET;           // 14
AnnotationType.INK;             // 15
AnnotationType.POPUP;           // 16
AnnotationType.FILEATTACHMENT;  // 17
AnnotationType.WIDGET;          // 20
```

---

## AnnotationFlag Constants

```typescript
import { AnnotationFlag } from "pdfjs-dist";

AnnotationFlag.INVISIBLE;       // 0x01 -- Do not display if no handler
AnnotationFlag.HIDDEN;          // 0x02 -- Do not display or print
AnnotationFlag.PRINT;           // 0x04 -- Print when printing
AnnotationFlag.NOZOOM;          // 0x08 -- Do not scale with page
AnnotationFlag.NOROTATE;        // 0x10 -- Do not rotate with page
AnnotationFlag.NOVIEW;          // 0x20 -- Do not display on screen
AnnotationFlag.READONLY;        // 0x40 -- Do not allow interaction
AnnotationFlag.LOCKED;          // 0x80 -- Do not allow deletion/modification
AnnotationFlag.TOGGLENOVIEW;    // 0x100 -- Invert NOVIEW for export
AnnotationFlag.LOCKEDCONTENTS;  // 0x200 -- Lock annotation contents
```

Usage: Check flags with bitwise AND: `if (annotation.annotationFlags & AnnotationFlag.HIDDEN) { /* skip */ }`

---

## AnnotationMode Constants

```typescript
import { AnnotationMode } from "pdfjs-dist";

AnnotationMode.DISABLE;         // 0 -- No annotations rendered on canvas
AnnotationMode.ENABLE;          // 1 -- Static annotations (read-only)
AnnotationMode.ENABLE_FORMS;    // 2 -- Interactive form widgets (default)
AnnotationMode.ENABLE_STORAGE;  // 3 -- Forms with persistent AnnotationStorage
```
