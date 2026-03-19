# Document Loading Anti-Patterns (pdfjs-dist 5.x)

Common mistakes that cause document loading failures and how to fix them.

---

## 1. Swallowing Document Loading Errors

**Severity**: Critical -- users see a blank page with no feedback.

### Wrong

```typescript
import { getDocument } from "pdfjs-dist";

// BROKEN: Empty catch block silently hides all errors
async function loadPdf(url: string) {
  try {
    return await getDocument({ url }).promise;
  } catch (err) {
    // "It's fine, just ignore it"
    console.log(err);
    return null;
  }
}
```

### Correct

```typescript
import { getDocument, PasswordResponses } from "pdfjs-dist";
import type { PDFDocumentProxy } from "pdfjs-dist";

async function loadPdf(
  url: string,
  onError: (type: string, message: string) => void
): Promise<PDFDocumentProxy | null> {
  try {
    return await getDocument({ url }).promise;
  } catch (err: unknown) {
    if (!(err instanceof Error)) {
      onError("unknown", "An unexpected error occurred.");
      return null;
    }

    // ALWAYS classify errors and show specific messages
    switch (err.name) {
      case "InvalidPDFException":
        onError("invalid", "This file is not a valid PDF.");
        break;
      case "MissingPDFException":
        onError("missing", "The PDF file was not found.");
        break;
      case "PasswordException":
        onError("password", "This PDF requires a password.");
        break;
      default:
        onError("error", `Failed to load PDF: ${err.message}`);
    }
    return null;
  }
}
```

**Why**: Swallowing errors means users see a blank viewer with no explanation. They cannot diagnose whether the URL is wrong, the file is corrupt, or a password is needed. ALWAYS show specific, actionable error messages based on the exception type.

---

## 2. Not Handling Password-Protected PDFs

**Severity**: High -- password-protected PDFs cause an unhandled promise rejection.

### Wrong

```typescript
import { getDocument } from "pdfjs-dist";

// BROKEN: No password handling -- PasswordException becomes unhandled rejection
async function loadPdf(url: string) {
  const doc = await getDocument({ url }).promise;
  return doc;
}
```

### Correct

```typescript
import { getDocument, PasswordResponses } from "pdfjs-dist";
import type { PDFDocumentProxy } from "pdfjs-dist";

async function loadPdf(
  url: string,
  onPasswordNeeded: () => Promise<string | null>
): Promise<PDFDocumentProxy> {
  const loadingTask = getDocument({ url });

  loadingTask.onPassword = async (
    updateCallback: (password: string) => void,
    reason: number
  ) => {
    const password = await onPasswordNeeded();
    if (password) {
      updateCallback(password);
    } else {
      loadingTask.destroy();
    }
  };

  return loadingTask.promise;
}
```

**Why**: Approximately 5-10% of PDFs in the wild are password-protected. If your application does not handle `PasswordException`, these PDFs cause unhandled promise rejections that crash the loading flow. ALWAYS implement either the `onPassword` callback or a try/catch with retry logic.

---

## 3. Missing CMap Configuration for CJK PDFs

**Severity**: Medium -- CJK text appears as blank rectangles (tofu).

### Wrong

```typescript
import { getDocument } from "pdfjs-dist";

// BROKEN: No CMap configuration -- CJK text will not render
const doc = await getDocument({ url: "/chinese-document.pdf" }).promise;
```

### Correct

```typescript
import { getDocument } from "pdfjs-dist";

// ALWAYS include CMap configuration
const doc = await getDocument({
  url: "/chinese-document.pdf",
  cMapUrl: new URL("pdfjs-dist/cmaps/", import.meta.url).toString(),
  cMapPacked: true,
}).promise;
```

**Why**: PDF.js cannot decode CJK character encodings without CMap data. Without `cMapUrl`, Chinese, Japanese, and Korean characters render as empty boxes. This affects all CJK PDFs, not just specific ones. ALWAYS set `cMapUrl` and `cMapPacked: true` as a default, even if you do not expect CJK content -- it is a zero-cost safety net when no CJK is present.

---

## 4. Missing standardFontDataUrl

**Severity**: Medium -- text in PDFs using standard 14 fonts renders with wrong glyphs or spacing.

### Wrong

```typescript
import { getDocument } from "pdfjs-dist";

// BROKEN: No standard font data -- PDFs referencing standard fonts render poorly
const doc = await getDocument({ url: "/report.pdf" }).promise;
```

### Correct

```typescript
import { getDocument } from "pdfjs-dist";

const doc = await getDocument({
  url: "/report.pdf",
  standardFontDataUrl: new URL(
    "pdfjs-dist/standard_fonts/",
    import.meta.url
  ).toString(),
}).promise;
```

**Why**: Many PDFs reference the standard 14 PDF fonts (Helvetica, Times-Roman, Courier, etc.) without embedding them, expecting the viewer to provide them. Without `standardFontDataUrl`, PDF.js uses imprecise fallback metrics, causing text to render with incorrect spacing, wrong glyphs, or misaligned layouts.

---

## 5. Retrying PasswordException with the Same (Wrong) Password

**Severity**: Medium -- creates an infinite retry loop.

### Wrong

```typescript
import { getDocument } from "pdfjs-dist";

// BROKEN: Retries with the same password forever
async function loadPdf(url: string, password: string) {
  while (true) {
    try {
      return await getDocument({ url, password }).promise;
    } catch (err: unknown) {
      if (err instanceof Error && err.name === "PasswordException") {
        continue; // Retry with same wrong password -- infinite loop!
      }
      throw err;
    }
  }
}
```

### Correct

```typescript
import { getDocument, PasswordResponses } from "pdfjs-dist";
import type { PDFDocumentProxy } from "pdfjs-dist";

const MAX_ATTEMPTS = 3;

async function loadPdf(
  url: string,
  getNewPassword: () => Promise<string | null>
): Promise<PDFDocumentProxy> {
  let password: string | undefined;

  for (let attempt = 0; attempt < MAX_ATTEMPTS; attempt++) {
    try {
      return await getDocument({ url, password }).promise;
    } catch (err: unknown) {
      if (!(err instanceof Error) || err.name !== "PasswordException") {
        throw err;
      }
      // ALWAYS get a NEW password from the user before retrying
      const newPassword = await getNewPassword();
      if (!newPassword) throw new Error("Password entry cancelled.");
      password = newPassword;
    }
  }

  throw new Error(`Failed after ${MAX_ATTEMPTS} password attempts.`);
}
```

**Why**: A wrong password will always be wrong. Retrying without getting a new password from the user creates an infinite loop that freezes the application. ALWAYS prompt for a new password and ALWAYS cap the number of retry attempts.

---

## 6. Loading Cross-Origin PDFs Without CORS Handling

**Severity**: High -- PDFs from external servers fail silently.

### Wrong

```typescript
import { getDocument } from "pdfjs-dist";

// BROKEN: No CORS handling for cross-origin PDF
const doc = await getDocument({
  url: "https://external-server.com/document.pdf",
}).promise;
// Throws: "Failed to fetch" or CORS error
```

### Correct

```typescript
import { getDocument } from "pdfjs-dist";

// SOLUTION 1: Pre-fetch with explicit CORS mode and pass ArrayBuffer
async function loadCrossOriginPdf(url: string) {
  const response = await fetch(url, { mode: "cors" });
  if (!response.ok) {
    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
  }
  const data = await response.arrayBuffer();
  return await getDocument({ data }).promise;
}

// SOLUTION 2: Use a same-origin proxy
async function loadViaProxy(externalUrl: string) {
  const proxyUrl = `/api/pdf-proxy?url=${encodeURIComponent(externalUrl)}`;
  return await getDocument({ url: proxyUrl }).promise;
}
```

**Why**: Browsers enforce same-origin policy on fetch requests. If the external server does not send `Access-Control-Allow-Origin` headers, the PDF download fails. ALWAYS either pre-fetch with proper CORS handling, use a same-origin proxy, or ensure the PDF server sends CORS headers.

---

## 7. Disabling Range Requests for Large PDFs

**Severity**: High -- causes out-of-memory crashes on large PDFs.

### Wrong

```typescript
import { getDocument } from "pdfjs-dist";

// BROKEN: Forces entire PDF into memory at once
const doc = await getDocument({
  url: "/massive-200mb-document.pdf",
  disableRange: true,   // Downloads ALL 200 MB into memory
  disableStream: true,  // No streaming either
}).promise;
```

### Correct

```typescript
import { getDocument } from "pdfjs-dist";

// ALWAYS keep range requests enabled for large PDFs
const doc = await getDocument({
  url: "/massive-200mb-document.pdf",
  disableRange: false,   // Default -- PDF.js fetches only needed parts
  disableStream: false,  // Default -- enables progressive loading
  rangeChunkSize: 65536, // Default 64 KB chunks
}).promise;
```

**Why**: With `disableRange: true`, PDF.js downloads the entire file into an ArrayBuffer before parsing. For a 200 MB PDF, this requires 200 MB of contiguous memory, which causes OOM crashes on mobile devices and lower-end machines. Range requests allow PDF.js to fetch only the parts it needs (header, cross-reference table, requested pages), keeping memory usage manageable. NEVER disable range requests unless the server genuinely does not support them.

---

## 8. Not Validating Uploaded Files Before Loading

**Severity**: Medium -- PDF.js throws cryptic errors on non-PDF files.

### Wrong

```typescript
import { getDocument } from "pdfjs-dist";

// BROKEN: Passes any file directly to PDF.js without validation
async function handleUpload(file: File) {
  const data = await file.arrayBuffer();
  // If user uploads a JPEG, they get "InvalidPDFException" with no context
  const doc = await getDocument({ data }).promise;
  return doc;
}
```

### Correct

```typescript
import { getDocument } from "pdfjs-dist";

async function handleUpload(file: File) {
  // Step 1: Check MIME type
  if (file.type && file.type !== "application/pdf") {
    throw new Error(
      `Please upload a PDF file. The selected file is "${file.type}".`
    );
  }

  // Step 2: Check magic bytes
  const header = new Uint8Array(await file.slice(0, 5).arrayBuffer());
  const magic = String.fromCharCode(...header);
  if (!magic.startsWith("%PDF")) {
    throw new Error(
      "The selected file does not appear to be a PDF. " +
      "Please select a valid PDF file."
    );
  }

  // Step 3: Load with PDF.js
  const data = await file.arrayBuffer();
  return await getDocument({ data }).promise;
}
```

**Why**: Users can upload any file type through a file input, even with `accept=".pdf"` (which is just a hint, not enforced). Without validation, non-PDF files produce confusing `InvalidPDFException` errors. ALWAYS validate the file before passing it to PDF.js to provide clear, user-friendly error messages.

---

## 9. Using instanceof to Check PDF.js Exception Types

**Severity**: Medium -- exception type checks fail silently, skipping error-specific handling.

### Wrong

```typescript
import { getDocument } from "pdfjs-dist";

try {
  const doc = await getDocument({ url }).promise;
} catch (err) {
  // BROKEN: PDF.js exceptions are NOT exported as classes in pdfjs-dist 5.x
  // This check will NEVER match
  if (err instanceof InvalidPDFException) {
    // This code never executes
  }
}
```

### Correct

```typescript
import { getDocument } from "pdfjs-dist";

try {
  const doc = await getDocument({ url }).promise;
} catch (err: unknown) {
  if (!(err instanceof Error)) throw err;

  // ALWAYS use the name property to identify PDF.js exception types
  if (err.name === "InvalidPDFException") {
    // This correctly matches the exception
  }
}
```

**Why**: PDF.js exception classes (`InvalidPDFException`, `MissingPDFException`, `PasswordException`, `UnknownErrorException`) are not exported as constructors from pdfjs-dist 5.x. Attempting `instanceof` checks always evaluates to `false`, causing all error-specific handling to be silently skipped. ALWAYS use `err.name` string comparison instead.
