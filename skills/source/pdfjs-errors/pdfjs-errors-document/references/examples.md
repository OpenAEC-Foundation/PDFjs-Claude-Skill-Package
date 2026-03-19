# Document Error Handling Patterns (pdfjs-dist 5.x)

## 1. Comprehensive Error Handler

Catch and classify all document loading errors with actionable user messages.

```typescript
import {
  getDocument,
  GlobalWorkerOptions,
  PasswordResponses,
} from "pdfjs-dist";
import type { PDFDocumentProxy } from "pdfjs-dist";

GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

type ErrorType = "invalid" | "missing" | "password" | "network" | "unknown";

interface DocumentError {
  type: ErrorType;
  message: string;
  recoverable: boolean;
  originalError: Error;
}

function classifyError(err: Error, url: string): DocumentError {
  switch (err.name) {
    case "InvalidPDFException":
      return {
        type: "invalid",
        message: "This file is not a valid PDF or is corrupt.",
        recoverable: false,
        originalError: err,
      };

    case "MissingPDFException":
      return {
        type: "missing",
        message: `PDF file not found at "${url}".`,
        recoverable: false,
        originalError: err,
      };

    case "PasswordException":
      return {
        type: "password",
        message: (err as Error & { code: number }).code ===
          PasswordResponses.INCORRECT_PASSWORD
          ? "Incorrect password."
          : "This PDF is password-protected.",
        recoverable: true,
        originalError: err,
      };

    default:
      // Network errors, CORS errors, and other non-PDF.js errors
      if (err.message?.includes("Failed to fetch") ||
          err.message?.includes("NetworkError") ||
          err.message?.includes("CORS")) {
        return {
          type: "network",
          message: "Unable to download the PDF. Check your network connection.",
          recoverable: true,
          originalError: err,
        };
      }
      return {
        type: "unknown",
        message: `Failed to load PDF: ${err.message}`,
        recoverable: false,
        originalError: err,
      };
  }
}

async function loadDocument(url: string): Promise<PDFDocumentProxy> {
  try {
    return await getDocument({
      url,
      cMapUrl: new URL("pdfjs-dist/cmaps/", import.meta.url).toString(),
      cMapPacked: true,
      standardFontDataUrl: new URL(
        "pdfjs-dist/standard_fonts/",
        import.meta.url
      ).toString(),
    }).promise;
  } catch (err: unknown) {
    if (err instanceof Error) {
      const classified = classifyError(err, url);
      console.error(`[PDF] ${classified.type}: ${classified.message}`, err);
      throw classified;
    }
    throw err;
  }
}
```

---

## 2. Password Prompt Workflow (Callback-Based)

Interactive password handling using the `onPassword` callback, suitable for UI frameworks.

```typescript
import {
  getDocument,
  GlobalWorkerOptions,
  PasswordResponses,
} from "pdfjs-dist";
import type { PDFDocumentProxy } from "pdfjs-dist";

GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

/**
 * Load a PDF with interactive password prompting.
 *
 * @param url - PDF URL
 * @param promptPassword - Function that shows a password dialog and returns
 *   the entered password, or null if the user cancels.
 *   Receives a boolean indicating if the previous attempt was wrong.
 */
async function loadWithPasswordPrompt(
  url: string,
  promptPassword: (isRetry: boolean) => Promise<string | null>
): Promise<PDFDocumentProxy> {
  const loadingTask = getDocument({
    url,
    cMapUrl: new URL("pdfjs-dist/cmaps/", import.meta.url).toString(),
    cMapPacked: true,
    standardFontDataUrl: new URL(
      "pdfjs-dist/standard_fonts/",
      import.meta.url
    ).toString(),
  });

  loadingTask.onPassword = async (
    updateCallback: (password: string) => void,
    reason: number
  ) => {
    const isRetry = reason === PasswordResponses.INCORRECT_PASSWORD;
    const password = await promptPassword(isRetry);

    if (password === null) {
      // User cancelled -- destroy the loading task
      loadingTask.destroy();
      return;
    }

    updateCallback(password);
  };

  return loadingTask.promise;
}

// Usage example
const doc = await loadWithPasswordPrompt(
  "/encrypted.pdf",
  async (isRetry) => {
    const message = isRetry
      ? "Incorrect password. Please try again:"
      : "This PDF is password-protected. Enter password:";
    return window.prompt(message);
  }
);
```

---

## 3. Password Prompt Workflow (Try/Catch-Based)

Password handling using try/catch, suitable for non-interactive or server-like contexts.

```typescript
import {
  getDocument,
  GlobalWorkerOptions,
  PasswordResponses,
} from "pdfjs-dist";
import type { PDFDocumentProxy } from "pdfjs-dist";

GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

const MAX_PASSWORD_ATTEMPTS = 3;

async function loadWithPasswordRetry(
  url: string,
  getPassword: () => Promise<string | null>
): Promise<PDFDocumentProxy> {
  let password: string | undefined;
  let attempts = 0;

  while (attempts < MAX_PASSWORD_ATTEMPTS) {
    try {
      return await getDocument({
        url,
        password,
        cMapUrl: new URL("pdfjs-dist/cmaps/", import.meta.url).toString(),
        cMapPacked: true,
        standardFontDataUrl: new URL(
          "pdfjs-dist/standard_fonts/",
          import.meta.url
        ).toString(),
      }).promise;
    } catch (err: unknown) {
      if (!(err instanceof Error) || err.name !== "PasswordException") {
        throw err; // Not a password error -- rethrow
      }

      const typedErr = err as Error & { code: number };

      if (typedErr.code === PasswordResponses.NEED_PASSWORD ||
          typedErr.code === PasswordResponses.INCORRECT_PASSWORD) {
        attempts++;
        const pw = await getPassword();
        if (pw === null) throw new Error("Password entry cancelled.");
        password = pw;
        continue;
      }

      throw err;
    }
  }

  throw new Error(
    `Failed to open PDF after ${MAX_PASSWORD_ATTEMPTS} password attempts.`
  );
}
```

---

## 4. CORS Proxy Fallback

Load cross-origin PDFs by falling back to a same-origin proxy when CORS fails.

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";
import type { PDFDocumentProxy } from "pdfjs-dist";

GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

async function loadPdfWithCorsFallback(
  url: string,
  proxyBaseUrl: string = "/api/pdf-proxy"
): Promise<PDFDocumentProxy> {
  // Attempt 1: Direct load (works for same-origin or CORS-enabled servers)
  try {
    return await getDocument({
      url,
      cMapUrl: new URL("pdfjs-dist/cmaps/", import.meta.url).toString(),
      cMapPacked: true,
    }).promise;
  } catch (err: unknown) {
    if (!(err instanceof Error)) throw err;

    // Only fall back to proxy for network/CORS errors
    const isCorsError =
      err.message.includes("Failed to fetch") ||
      err.message.includes("NetworkError") ||
      err.message.includes("CORS") ||
      err.name === "MissingPDFException";

    if (!isCorsError) throw err;

    console.warn(`Direct PDF load failed, trying proxy: ${err.message}`);
  }

  // Attempt 2: Load through same-origin proxy
  const proxyUrl = `${proxyBaseUrl}?url=${encodeURIComponent(url)}`;
  return await getDocument({
    url: proxyUrl,
    cMapUrl: new URL("pdfjs-dist/cmaps/", import.meta.url).toString(),
    cMapPacked: true,
  }).promise;
}
```

---

## 5. CJK-Ready Document Loading

Complete setup for PDFs containing Chinese, Japanese, or Korean text.

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";
import type { PDFDocumentProxy } from "pdfjs-dist";

GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

/**
 * ALWAYS use this configuration when CJK PDFs may be loaded.
 * Without CMap data, CJK characters render as blank rectangles.
 *
 * Deployment checklist:
 * 1. Copy pdfjs-dist/cmaps/ to your public assets directory
 * 2. Copy pdfjs-dist/standard_fonts/ to your public assets directory
 * 3. Set cMapUrl to point to the deployed cmaps directory
 * 4. Set cMapPacked: true (pdfjs-dist ships binary-packed CMaps)
 * 5. Set standardFontDataUrl to point to the deployed fonts directory
 */
async function loadCjkDocument(url: string): Promise<PDFDocumentProxy> {
  return await getDocument({
    url,
    // CMap configuration -- REQUIRED for CJK text
    cMapUrl: new URL("pdfjs-dist/cmaps/", import.meta.url).toString(),
    cMapPacked: true,
    // Standard font data -- REQUIRED for PDFs using standard 14 fonts
    standardFontDataUrl: new URL(
      "pdfjs-dist/standard_fonts/",
      import.meta.url
    ).toString(),
  }).promise;
}
```

### Bundler Configuration for CMap Deployment

**Vite** (copies automatically with `import.meta.url` resolution):
```typescript
// No special config needed -- Vite resolves import.meta.url paths at build time
```

**Webpack** (requires copy-webpack-plugin):
```javascript
// webpack.config.js
const CopyPlugin = require("copy-webpack-plugin");

module.exports = {
  plugins: [
    new CopyPlugin({
      patterns: [
        {
          from: "node_modules/pdfjs-dist/cmaps",
          to: "cmaps/",
        },
        {
          from: "node_modules/pdfjs-dist/standard_fonts",
          to: "standard_fonts/",
        },
      ],
    }),
  ],
};

// In application code:
const doc = await getDocument({
  url: "/document.pdf",
  cMapUrl: "/cmaps/",
  cMapPacked: true,
  standardFontDataUrl: "/standard_fonts/",
}).promise;
```

---

## 6. Large PDF Loading with Range Requests

Handle extremely large PDFs without running out of memory.

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";
import type { PDFDocumentProxy } from "pdfjs-dist";

GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

/**
 * Load a large PDF using range requests.
 *
 * REQUIREMENTS for range requests to work:
 * 1. Server MUST support HTTP Range requests (Accept-Ranges: bytes)
 * 2. Server MUST return Content-Length header
 * 3. Server MUST handle Range header and return 206 Partial Content
 *
 * Without range request support, PDF.js downloads the entire file
 * into memory, which causes OOM on very large PDFs.
 */
async function loadLargePdf(url: string): Promise<PDFDocumentProxy> {
  return await getDocument({
    url,
    // NEVER set disableRange to true for large PDFs
    disableRange: false,
    // Smaller chunks reduce memory pressure
    rangeChunkSize: 65536, // 64 KB chunks (default)
    // Enable streaming for progressive loading
    disableStream: false,
    // CMap and font support
    cMapUrl: new URL("pdfjs-dist/cmaps/", import.meta.url).toString(),
    cMapPacked: true,
    standardFontDataUrl: new URL(
      "pdfjs-dist/standard_fonts/",
      import.meta.url
    ).toString(),
  }).promise;
}
```

---

## 7. Loading Task Cancellation

Properly cancel document loading when the user navigates away or loads a different PDF.

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";
import type {
  PDFDocumentProxy,
  PDFDocumentLoadingTask,
} from "pdfjs-dist";

GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

class PdfLoader {
  private currentTask: PDFDocumentLoadingTask | null = null;
  private currentDoc: PDFDocumentProxy | null = null;

  async load(url: string): Promise<PDFDocumentProxy> {
    // ALWAYS cancel any in-progress loading before starting a new one
    await this.cancel();

    this.currentTask = getDocument({
      url,
      cMapUrl: new URL("pdfjs-dist/cmaps/", import.meta.url).toString(),
      cMapPacked: true,
      standardFontDataUrl: new URL(
        "pdfjs-dist/standard_fonts/",
        import.meta.url
      ).toString(),
    });

    try {
      this.currentDoc = await this.currentTask.promise;
      return this.currentDoc;
    } finally {
      this.currentTask = null;
    }
  }

  async cancel(): Promise<void> {
    if (this.currentTask) {
      await this.currentTask.destroy();
      this.currentTask = null;
    }
    if (this.currentDoc) {
      await this.currentDoc.destroy();
      this.currentDoc = null;
    }
  }
}
```

---

## 8. File Upload Validation Before Loading

Validate user-uploaded files before passing them to PDF.js.

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";
import type { PDFDocumentProxy } from "pdfjs-dist";

GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

const PDF_MAGIC_BYTES = [0x25, 0x50, 0x44, 0x46]; // %PDF
const MAX_FILE_SIZE = 100 * 1024 * 1024; // 100 MB

interface ValidationResult {
  valid: boolean;
  error?: string;
}

function validatePdfFile(file: File): ValidationResult {
  if (file.size === 0) {
    return { valid: false, error: "File is empty." };
  }

  if (file.size > MAX_FILE_SIZE) {
    return {
      valid: false,
      error: `File is too large (${(file.size / 1024 / 1024).toFixed(1)} MB). Maximum is ${MAX_FILE_SIZE / 1024 / 1024} MB.`,
    };
  }

  if (file.type && file.type !== "application/pdf") {
    return {
      valid: false,
      error: `Expected a PDF file but got "${file.type}".`,
    };
  }

  return { valid: true };
}

async function validatePdfMagicBytes(file: File): Promise<ValidationResult> {
  const header = await file.slice(0, 4).arrayBuffer();
  const bytes = new Uint8Array(header);

  const isPdf = PDF_MAGIC_BYTES.every((byte, i) => bytes[i] === byte);
  if (!isPdf) {
    return {
      valid: false,
      error: "File does not appear to be a PDF (missing %PDF header).",
    };
  }

  return { valid: true };
}

async function loadUploadedPdf(file: File): Promise<PDFDocumentProxy> {
  // Step 1: Basic validation
  const basicCheck = validatePdfFile(file);
  if (!basicCheck.valid) throw new Error(basicCheck.error);

  // Step 2: Magic bytes check
  const magicCheck = await validatePdfMagicBytes(file);
  if (!magicCheck.valid) throw new Error(magicCheck.error);

  // Step 3: Load with PDF.js
  const data = await file.arrayBuffer();
  return await getDocument({
    data,
    cMapUrl: new URL("pdfjs-dist/cmaps/", import.meta.url).toString(),
    cMapPacked: true,
    standardFontDataUrl: new URL(
      "pdfjs-dist/standard_fonts/",
      import.meta.url
    ).toString(),
  }).promise;
}
```
