# API Signatures Reference (pdfjs-dist 5.x Text Layer)

## TextLayer Class

The `TextLayer` class creates a selectable and searchable text overlay positioned over a rendered PDF canvas.

### Constructor

```typescript
import { TextLayer } from "pdfjs-dist";

new TextLayer(params: {
  textContentSource: TextContent | ReadableStream;  // REQUIRED. From getTextContent() or streamTextContent()
  container: HTMLElement;                            // REQUIRED. DOM element to render text spans into
  viewport: PageViewport;                            // REQUIRED. Viewport matching the rendered canvas
  images?: TextLayerImages;                          // OPTIONAL. Handles right-click on embedded images
})
```

**Notes**:
- `textContentSource` accepts BOTH resolved `TextContent` objects and `ReadableStream` from `streamTextContent()`
- The `container` element MUST be positioned absolutely over the canvas
- The `viewport` MUST match the viewport used to render the canvas -- mismatched viewports cause text misalignment

---

### Instance Methods

#### `render(): Promise<void>`

Renders text spans into the container element. Processes text content and creates positioned `<span>` elements.

```typescript
const textLayer = new TextLayer({
  textContentSource: textContent,
  container: textLayerDiv,
  viewport: viewport,
});

await textLayer.render();
// After this resolves, textLayerDiv contains positioned <span> elements
```

**Behavior**:
- Creates one `<span>` per text item from the text content
- Positions spans using CSS transforms to match PDF text positions
- Populates `textDivs` and `textContentItemsStr` arrays
- Returns a Promise that resolves when all spans are in the DOM

---

#### `update(params: TextLayerUpdateParameters): void`

Updates an already-rendered text layer when the viewport changes (zoom or rotation). Repositions existing spans without re-creating them.

```typescript
interface TextLayerUpdateParameters {
  viewport: PageViewport;   // REQUIRED. New viewport with updated scale/rotation
  onBefore?: () => void;    // OPTIONAL. Callback invoked before DOM updates begin
}
```

```typescript
const newViewport = page.getViewport({ scale: 2.0 });
textLayer.update({
  viewport: newViewport,
  onBefore() {
    // Resize container before spans are repositioned
    textLayerDiv.style.width = `${Math.floor(newViewport.width)}px`;
    textLayerDiv.style.height = `${Math.floor(newViewport.height)}px`;
  },
});
```

**Behavior**:
- ALWAYS call this instead of destroying and re-creating the TextLayer on viewport changes
- More efficient than re-rendering because it reuses existing DOM elements
- The `onBefore` callback runs synchronously before span transforms are updated

---

#### `cancel(): void`

Cancels any in-progress text layer rendering. Throws an `AbortException` internally.

```typescript
textLayer.cancel();
```

**Behavior**:
- Safe to call even if rendering has already completed (no-op)
- ALWAYS call before creating a new TextLayer for the same container
- Does NOT remove existing DOM elements from the container

---

### Instance Properties

#### `textDivs: HTMLElement[]`

Array of `<span>` elements created during rendering. Each span corresponds to one text item from the input `TextContent`.

```typescript
await textLayer.render();

// Access individual text spans
for (const div of textLayer.textDivs) {
  console.log(div.textContent);  // Text content of this span
  console.log(div.style.transform);  // CSS transform positioning this span
}
```

**Use case**: Access these spans for text search highlighting or custom styling.

---

#### `textContentItemsStr: string[]`

Array of string values matching the `str` property of each text item. Parallel array to `textDivs`.

```typescript
await textLayer.render();

// textContentItemsStr[i] === textContent.items[i].str
for (let i = 0; i < textLayer.textContentItemsStr.length; i++) {
  console.log(textLayer.textContentItemsStr[i]);  // Raw text string
  console.log(textLayer.textDivs[i]);              // Corresponding DOM element
}
```

---

### Static Methods

#### `TextLayer.cleanup(): void`

Clears global caches (font ascent measurements, canvas contexts) shared across all TextLayer instances.

```typescript
TextLayer.cleanup();
```

**Behavior**:
- ONLY call when ALL TextLayer instances in the application have been destroyed
- Calling while active TextLayer instances exist may cause rendering issues
- Frees memory from cached font metrics and measurement canvases

---

## PDFPageProxy.getTextContent()

Extracts text content from a PDF page. Returns a structured object containing text items with positions.

```typescript
getTextContent(params?: {
  includeMarkedContent?: boolean;   // Include marked content items. Default: false
  disableNormalization?: boolean;   // Skip Unicode normalization. Default: false
}): Promise<TextContent>
```

**Notes**:
- `includeMarkedContent: true` adds `TextMarkedContent` items to the result (used for tagged PDFs and accessibility)
- `disableNormalization: true` preserves original Unicode codepoints without NFC normalization
- ALWAYS use default parameters unless you have a specific need for marked content or raw Unicode

---

## PDFPageProxy.streamTextContent()

Streaming variant of `getTextContent()`. Returns a `ReadableStream` that yields `TextContent` chunks progressively.

```typescript
streamTextContent(params?: {
  includeMarkedContent?: boolean;   // Same as getTextContent. Default: false
  disableNormalization?: boolean;   // Same as getTextContent. Default: false
}): ReadableStream<TextContent>
```

**When to use**:
- Large PDFs with many pages being processed simultaneously
- When you want to start processing text before the entire page is parsed
- Memory-constrained environments where holding full TextContent is costly

**When NOT to use**:
- Single-page text extraction (getTextContent is simpler)
- When you need all text items available at once for search/analysis

---

## TextContent Structure

The object returned by `getTextContent()` or yielded by `streamTextContent()`.

```typescript
interface TextContent {
  items: Array<TextItem | TextMarkedContent>;  // Text items with positions
  styles: Record<string, TextStyle>;           // Font style definitions keyed by fontName
}
```

---

## TextItem

Represents a single text run within the PDF page.

```typescript
interface TextItem {
  str: string;           // The actual text string
  dir: string;           // Text direction: "ltr" (left-to-right) or "rtl" (right-to-left)
  width: number;         // Width of the text run in user space units
  height: number;        // Height of the text run in user space units
  transform: number[];   // 6-element transform matrix [a, b, c, d, e, f]
                         // transform[4] = x position
                         // transform[5] = y position
  fontName: string;      // Key into TextContent.styles for font information
  hasEOL: boolean;       // True if this item ends a line
}
```

**Transform matrix interpretation**:
- `transform[0]` (a): horizontal scaling (related to font size)
- `transform[1]` (b): vertical skewing
- `transform[2]` (c): horizontal skewing
- `transform[3]` (d): vertical scaling (related to font size)
- `transform[4]` (e): x position in PDF user space
- `transform[5]` (f): y position in PDF user space

---

## TextMarkedContent

Represents marked content boundaries in tagged PDFs. Only present when `includeMarkedContent: true`.

```typescript
interface TextMarkedContent {
  type: "beginMarkedContent" | "beginMarkedContentProps" | "endMarkedContent";
  id?: string;   // Marked content identifier
  tag?: string;  // Structure tag (e.g., "P", "Span", "H1")
}
```

**Note**: TextMarkedContent items do NOT have a `str` property. ALWAYS check for `"str" in item` before accessing text properties when `includeMarkedContent` is enabled.

---

## TextStyle

Font style information referenced by `TextItem.fontName`.

```typescript
interface TextStyle {
  fontFamily: string;    // CSS font family name
  ascent: number;        // Font ascent (fraction of font size)
  descent: number;       // Font descent (fraction of font size, typically negative)
  vertical: boolean;     // True for vertical writing mode
}
```

---

## Constants

```typescript
// Maximum text divs that TextLayer will render
// Beyond this limit, text layer rendering is skipped for performance
const MAX_TEXT_DIVS_TO_RENDER = 100000;
```
