# Document Error Types and API Reference (pdfjs-dist 5.x)

## Exception Classes

PDF.js defines specific exception types in `src/shared/util.js`. All exceptions extend a base `BaseException` class and are thrown during document loading via `getDocument().promise`.

### InvalidPDFException

Thrown when the loaded data is not a valid PDF file.

```typescript
// Exception structure
interface InvalidPDFException extends Error {
  name: "InvalidPDFException";
  message: string; // Description of what is invalid
}
```

**Common messages**:
- `"Invalid PDF structure"` -- file does not start with `%PDF` header
- `"No PDF header found"` -- first bytes are not a PDF magic number
- `"Invalid PDF stream"` -- PDF cross-reference table or stream is corrupt
- `"XRef entry is not free"` -- cross-reference table has structural errors

**When thrown**: During PDF parsing, after the data is fetched but before any pages are available.

**Recovery**: NEVER retry with the same data -- the file itself is invalid. Inform the user the file is corrupt or not a PDF.

---

### MissingPDFException

Thrown when the PDF file cannot be retrieved from the specified source.

```typescript
// Exception structure
interface MissingPDFException extends Error {
  name: "MissingPDFException";
  message: string; // Description of why the file is missing
}
```

**Common messages**:
- `"Missing PDF file."` -- generic file-not-found
- `"Missing PDF \"<url>\"."` -- URL-specific not-found

**When thrown**: During the fetch phase, before any PDF parsing begins.

**Recovery**: Verify the URL is correct. Check server logs for 404 responses. If the file was recently uploaded, it may not have propagated yet -- retry after a short delay.

---

### PasswordException

Thrown when a PDF is encrypted and requires a password, or when the provided password is incorrect.

```typescript
// Exception structure
interface PasswordException extends Error {
  name: "PasswordException";
  message: string;
  code: number; // PasswordResponses enum value
}
```

**The `code` property** distinguishes between two states:

| Code | Constant | Meaning |
|------|----------|---------|
| `1` | `PasswordResponses.NEED_PASSWORD` | PDF is encrypted, no password was provided |
| `2` | `PasswordResponses.INCORRECT_PASSWORD` | A password was provided but it is wrong |

**When thrown**: After the PDF header is parsed and encryption is detected.

**Recovery**: Prompt the user for a password and retry with `getDocument({ url, password })`.

---

### UnknownErrorException

Thrown for unexpected internal errors during PDF parsing.

```typescript
// Exception structure
interface UnknownErrorException extends Error {
  name: "UnknownErrorException";
  message: string;
  details: string; // Additional error details from the parser
}
```

**When thrown**: During any phase of PDF processing when an unexpected condition occurs.

**Recovery**: Log the full error including `details` for debugging. This is usually a PDF.js bug or an extremely unusual PDF structure. Report to the PDF.js issue tracker with a sample PDF if possible.

---

## PasswordResponses Enum

```typescript
import { PasswordResponses } from "pdfjs-dist";

// Values:
PasswordResponses.NEED_PASSWORD       // 1
PasswordResponses.INCORRECT_PASSWORD  // 2
```

**ALWAYS** import `PasswordResponses` from `pdfjs-dist` rather than hardcoding numeric values. The enum ensures forward compatibility if values change in future versions.

---

## getDocument() Error-Related Options

The `getDocument()` function accepts a `DocumentInitParameters` object. These options affect error handling and data loading:

### Source Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `url` | `string \| URL` | -- | PDF URL. Throws `MissingPDFException` on 404. |
| `data` | `ArrayBuffer \| TypedArray` | -- | Raw PDF data. Throws `InvalidPDFException` if not valid PDF. |
| `httpHeaders` | `Record<string, string>` | `{}` | Custom HTTP headers for the fetch request. |
| `withCredentials` | `boolean` | `false` | Send cookies with cross-origin requests. |
| `password` | `string` | `""` | Password for encrypted PDFs. |

### CMap Options (CJK Support)

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `cMapUrl` | `string` | `""` | URL to the directory containing CMap files. |
| `cMapPacked` | `boolean` | `false` | Whether CMap files are binary-packed (`.bcmap`). ALWAYS set to `true` when using pdfjs-dist bundled CMaps. |

**ALWAYS** set both `cMapUrl` and `cMapPacked: true` when CJK support is needed. The CMap files are located at `node_modules/pdfjs-dist/cmaps/`.

**CMap directory contents**:
```
pdfjs-dist/cmaps/
  ├── Adobe-CNS1-UCS2.bcmap    # Traditional Chinese
  ├── Adobe-GB1-UCS2.bcmap     # Simplified Chinese
  ├── Adobe-Japan1-UCS2.bcmap  # Japanese
  ├── Adobe-Korea1-UCS2.bcmap  # Korean
  └── ... (78 CMap files total)
```

### Font Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `standardFontDataUrl` | `string` | `""` | URL to the directory containing standard font data files. |
| `useSystemFonts` | `boolean` | `true` | Whether to use system fonts as fallback. |

**ALWAYS** set `standardFontDataUrl` when deploying pdfjs-dist 5.x. PDF.js needs these files for PDFs that reference the 14 standard PDF fonts (Helvetica, Times, Courier, etc.) without embedding them.

**Standard font data directory**:
```
pdfjs-dist/standard_fonts/
  ├── FoxitFixed.pfb
  ├── FoxitFixedBold.pfb
  ├── FoxitFixedBoldItalic.pfb
  ├── FoxitFixedItalic.pfb
  ├── FoxitSans.pfb
  ├── FoxitSansBold.pfb
  ├── FoxitSansItalic.pfb
  ├── FoxitSerif.pfb
  ├── FoxitSerifBold.pfb
  ├── FoxitSerifBoldItalic.pfb
  ├── FoxitSerifItalic.pfb
  └── ... (additional font metrics files)
```

### Range Request Options (Large PDF Handling)

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `disableRange` | `boolean` | `false` | Disable HTTP range requests. NEVER set to `true` for large PDFs. |
| `disableStream` | `boolean` | `false` | Disable streaming of PDF data. |
| `rangeChunkSize` | `number` | `65536` | Size of each range request chunk in bytes. |
| `length` | `number` | -- | Total file size hint for range requests. |

**ALWAYS** keep `disableRange: false` (the default) for PDFs larger than 10 MB. Range requests allow PDF.js to load only the parts of the PDF that are needed, preventing out-of-memory errors on large files.

---

## Error Detection by Exception Name

```typescript
import { getDocument } from "pdfjs-dist";

try {
  const doc = await getDocument({ url }).promise;
} catch (err: unknown) {
  if (!(err instanceof Error)) throw err;

  // ALWAYS use err.name to identify exception type
  switch (err.name) {
    case "InvalidPDFException":
      // File is not a valid PDF
      break;
    case "MissingPDFException":
      // File not found (404)
      break;
    case "PasswordException":
      // Password required or incorrect
      // Access code via: (err as any).code
      break;
    case "UnknownErrorException":
      // Unexpected parser error
      // Access details via: (err as any).details
      break;
    default:
      // Network error, CORS error, or other non-PDF.js error
      break;
  }
}
```

**NEVER** use `instanceof` checks for PDF.js exception types -- they are not exported as classes in pdfjs-dist 5.x. ALWAYS use the `name` property for identification.

---

## PDFDocumentLoadingTask Error Events

The `PDFDocumentLoadingTask` object returned by `getDocument()` provides an `onPassword` callback for handling password-protected PDFs without try/catch.

```typescript
import { getDocument, PasswordResponses } from "pdfjs-dist";
import type { OnPasswordCallback } from "pdfjs-dist";

const loadingTask = getDocument({ url: "/encrypted.pdf" });

// Callback-based password handling
loadingTask.onPassword = (
  updateCallback: (password: string) => void,
  reason: number
) => {
  if (reason === PasswordResponses.NEED_PASSWORD) {
    // First attempt -- prompt user for password
    const password = prompt("Enter PDF password:");
    if (password) updateCallback(password);
  } else if (reason === PasswordResponses.INCORRECT_PASSWORD) {
    // Retry -- previous password was wrong
    const password = prompt("Wrong password. Try again:");
    if (password) updateCallback(password);
  }
};

const doc = await loadingTask.promise;
```

**ALWAYS** prefer the `onPassword` callback over try/catch for password handling when building interactive viewers -- it integrates cleanly with UI frameworks and avoids re-creating the loading task.
