---
name: pdfjs-errors-document
description: >
  Use when handling PDF document loading failures, password-protected PDFs, corrupt
  files, or missing CMap/font data. Prevents unhandled exceptions by covering all
  PDF.js error types and their recovery patterns.
  Covers InvalidPDFException, MissingPDFException, PasswordException, network/CORS
  errors, CJK font issues, and corrupt PDF recovery strategies.
  Keywords: InvalidPDFException, MissingPDFException, PasswordException, CMap, CORS, corrupt PDF.
license: MIT
compatibility: "Designed for Claude Code. Requires pdfjs-dist 5.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# pdfjs-errors-document

## Quick Reference

### Document Error Types

| Exception / Error | Likely Cause | Severity |
|-------------------|-------------|----------|
| `InvalidPDFException` | Corrupt file, not a PDF, truncated download | Critical |
| `MissingPDFException` | 404, wrong URL, file deleted | Critical |
| `PasswordException` (code `NEED_PASSWORD`) | PDF is password-protected, no password provided | High |
| `PasswordException` (code `INCORRECT_PASSWORD`) | Wrong password provided | High |
| `UnknownErrorException` | Unexpected internal error during parsing | High |
| CORS / Network error | Cross-origin fetch blocked, mixed content | Critical |
| Missing CMap data | CJK characters render as blank or tofu | Medium |
| Font loading failure | Missing `standardFontDataUrl`, broken embedded fonts | Medium |
| Memory error / OOM | Extremely large PDF exceeding browser memory | High |

### Critical Warnings

**ALWAYS** wrap `getDocument().promise` in a try/catch -- document loading can throw any of the exception types above, and unhandled rejections cause blank pages with no user feedback.

**ALWAYS** check the `name` property of caught errors to distinguish between `InvalidPDFException`, `MissingPDFException`, `PasswordException`, and `UnknownErrorException` -- each requires a different recovery strategy.

**NEVER** swallow document loading errors with an empty catch block -- the user sees a blank page with no indication of what went wrong.

**ALWAYS** configure `cMapUrl` and set `cMapPacked: true` when loading PDFs that may contain CJK (Chinese, Japanese, Korean) text -- without CMaps, CJK characters render as blank rectangles.

**ALWAYS** set `standardFontDataUrl` when using pdfjs-dist 5.x -- PDF.js needs access to standard font data files for proper text rendering of PDFs that reference standard 14 fonts.

**NEVER** retry a `PasswordException` with the same password -- it will fail again. ALWAYS prompt the user for a new password before retrying.

---

## Diagnostic Decision Tree

```
PDF document fails to load?
|
+-- InvalidPDFException
|   +-- File is 0 bytes? -> Download/upload failed, retry transfer
|   +-- File starts with "%PDF"? -> PDF is corrupt or truncated, re-obtain file
|   +-- File is actually HTML (404 page)? -> Server returned error page, fix URL
|   +-- File is a different format (DOCX, image)? -> Not a PDF, convert first
|
+-- MissingPDFException
|   +-- URL returns 404? -> Fix the URL path
|   +-- File was deleted? -> Handle gracefully, show "file not found" message
|   +-- Redirect to login page? -> Handle authentication before loading PDF
|   +-- Using relative URL? -> Use absolute URL or correct base path
|
+-- PasswordException
|   +-- code === PasswordResponses.NEED_PASSWORD? -> Prompt user for password
|   +-- code === PasswordResponses.INCORRECT_PASSWORD? -> Show "wrong password", reprompt
|   +-- Programmatic access needed? -> Pass password in getDocument({ password })
|
+-- Network / CORS error
|   +-- Mixed content (HTTP PDF on HTTPS page)? -> Serve PDF over HTTPS
|   +-- Cross-origin without CORS headers? -> Add CORS headers on PDF server
|   +-- Proxy available? -> Route PDF through same-origin proxy
|   +-- Fetch fails entirely? -> Check network, show offline message
|
+-- CJK text missing / blank characters
|   +-- cMapUrl not set? -> Set cMapUrl to cmaps directory path
|   +-- cMapPacked not true? -> Add cMapPacked: true to getDocument options
|   +-- CMap files not deployed? -> Copy cmaps/ from pdfjs-dist to public dir
|
+-- Font rendering issues
|   +-- standardFontDataUrl not set? -> Set path to standard_fonts directory
|   +-- Embedded font broken? -> PDF issue, not fixable in viewer
|   +-- Font files not deployed? -> Copy standard_fonts/ from pdfjs-dist
|
+-- Memory error / crash on large PDF
|   +-- PDF > 100 MB? -> Enable range requests (disableRange: false)
|   +-- Many high-res images? -> Reduce rendering scale
|   +-- Mobile device? -> Limit concurrent page renders
|
+-- UnknownErrorException
    +-- Check err.message for details -> May contain specific parser error
    +-- Reproducible with other PDFs? -> Likely a code issue, not PDF issue
    +-- Only this PDF? -> Likely a corrupt or unusual PDF structure
```

---

## Essential Fixes

### Fix 1: InvalidPDFException -- Corrupt or Non-PDF File

**Error**: `InvalidPDFException: Invalid PDF structure`

**Cause**: The loaded data is not a valid PDF. Common when a server returns an HTML error page instead of the PDF file, or the file is truncated.

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";

GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

async function loadPdfSafely(source: string | ArrayBuffer) {
  try {
    const doc = await getDocument(
      typeof source === "string" ? { url: source } : { data: source }
    ).promise;
    return doc;
  } catch (err: unknown) {
    if (err instanceof Error && err.name === "InvalidPDFException") {
      throw new Error(
        "The file is not a valid PDF. It may be corrupt, truncated, " +
        "or the server returned an error page instead of the PDF file."
      );
    }
    throw err;
  }
}
```

**Prevention**: ALWAYS validate that the server response has `Content-Type: application/pdf` before passing data to `getDocument()`. If loading from user upload, check that the file starts with the `%PDF` magic bytes.

### Fix 2: MissingPDFException -- File Not Found

**Error**: `MissingPDFException: Missing PDF file.`

**Cause**: The URL returned a 404 or the fetch failed entirely.

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";

GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

async function loadPdfWithNotFoundHandling(url: string) {
  try {
    return await getDocument({ url }).promise;
  } catch (err: unknown) {
    if (err instanceof Error && err.name === "MissingPDFException") {
      throw new Error(
        `PDF file not found at "${url}". ` +
        "Verify the URL is correct and the file exists on the server."
      );
    }
    throw err;
  }
}
```

### Fix 3: PasswordException -- Password-Protected PDF

**Error**: `PasswordException` with `code: PasswordResponses.NEED_PASSWORD`

**Cause**: The PDF is encrypted and requires a password to open.

```typescript
import {
  getDocument,
  GlobalWorkerOptions,
  PasswordResponses,
} from "pdfjs-dist";

GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

async function loadPasswordProtectedPdf(
  url: string,
  promptForPassword: () => Promise<string | null>
) {
  // First attempt: load without password
  try {
    return await getDocument({ url }).promise;
  } catch (err: unknown) {
    if (!(err instanceof Error) || err.name !== "PasswordException") {
      throw err;
    }

    // PDF requires a password -- prompt user
    const typedErr = err as Error & { code: number };

    if (typedErr.code === PasswordResponses.NEED_PASSWORD ||
        typedErr.code === PasswordResponses.INCORRECT_PASSWORD) {
      const password = await promptForPassword();
      if (!password) throw new Error("Password required but not provided.");

      // Retry with password
      try {
        return await getDocument({ url, password }).promise;
      } catch (retryErr: unknown) {
        if (retryErr instanceof Error &&
            retryErr.name === "PasswordException") {
          throw new Error("Incorrect password. Unable to open the PDF.");
        }
        throw retryErr;
      }
    }

    throw err;
  }
}
```

### Fix 4: CORS / Network Errors

**Error**: `Failed to fetch` or `Access to fetch blocked by CORS policy`

**Cause**: The PDF is hosted on a different origin without CORS headers, or the page uses HTTPS while the PDF is served over HTTP (mixed content).

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";

GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

// SOLUTION 1: Load through a same-origin proxy
async function loadPdfViaProxy(externalUrl: string) {
  const proxyUrl = `/api/pdf-proxy?url=${encodeURIComponent(externalUrl)}`;
  return await getDocument({ url: proxyUrl }).promise;
}

// SOLUTION 2: Fetch as ArrayBuffer with custom headers
async function loadPdfWithFetch(url: string) {
  const response = await fetch(url, {
    mode: "cors",
    credentials: "omit",
  });

  if (!response.ok) {
    throw new Error(`Failed to fetch PDF: ${response.status} ${response.statusText}`);
  }

  const data = await response.arrayBuffer();
  return await getDocument({ data }).promise;
}
```

### Fix 5: Missing CMap Data for CJK Text

**Error**: CJK characters appear as blank rectangles or tofu characters.

**Cause**: PDF.js needs CMap (Character Map) files to decode CJK-encoded text. Without them, characters in Chinese, Japanese, or Korean PDFs render incorrectly.

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";

GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

// ALWAYS configure CMap settings for CJK support
const doc = await getDocument({
  url: "/path/to/document.pdf",
  cMapUrl: new URL(
    "pdfjs-dist/cmaps/",
    import.meta.url
  ).toString(),
  cMapPacked: true,
}).promise;
```

**Prevention**: ALWAYS set `cMapUrl` and `cMapPacked: true` in `getDocument()` options. The CMap files are included in the `pdfjs-dist/cmaps/` directory. For production, copy the `cmaps/` directory to your public assets folder.

### Fix 6: Font Loading Failures

**Error**: Text renders with wrong glyphs, missing characters, or falls back to default fonts.

**Cause**: PDF.js needs standard font data files for PDFs that reference the standard 14 PDF fonts without embedding them.

```typescript
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";

GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();

// ALWAYS set standardFontDataUrl for proper font rendering
const doc = await getDocument({
  url: "/path/to/document.pdf",
  standardFontDataUrl: new URL(
    "pdfjs-dist/standard_fonts/",
    import.meta.url
  ).toString(),
  cMapUrl: new URL(
    "pdfjs-dist/cmaps/",
    import.meta.url
  ).toString(),
  cMapPacked: true,
}).promise;
```

---

## Prevention: Production-Ready Document Loading

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

interface LoadOptions {
  url: string;
  password?: string;
  onPasswordRequired?: () => Promise<string | null>;
  onError?: (type: string, message: string) => void;
}

async function loadDocument(options: LoadOptions): Promise<PDFDocumentProxy> {
  const { url, password, onPasswordRequired, onError } = options;

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
    if (!(err instanceof Error)) throw err;

    switch (err.name) {
      case "InvalidPDFException":
        onError?.("invalid", "The file is not a valid PDF.");
        throw err;

      case "MissingPDFException":
        onError?.("missing", `PDF not found at "${url}".`);
        throw err;

      case "PasswordException": {
        const typedErr = err as Error & { code: number };
        if (onPasswordRequired &&
            (typedErr.code === PasswordResponses.NEED_PASSWORD ||
             typedErr.code === PasswordResponses.INCORRECT_PASSWORD)) {
          const pw = await onPasswordRequired();
          if (pw) return loadDocument({ ...options, password: pw });
          onError?.("password", "Password is required to open this PDF.");
        }
        throw err;
      }

      default:
        onError?.("unknown", `Failed to load PDF: ${err.message}`);
        throw err;
    }
  }
}
```

---

## Reference Links

- [references/methods.md](references/methods.md) -- Exception types, error codes, PasswordResponses enum, getDocument error-related options
- [references/examples.md](references/examples.md) -- Error handling patterns, password prompt workflows, fallback strategies
- [references/anti-patterns.md](references/anti-patterns.md) -- Common mistakes: swallowing errors, not handling passwords, missing CMap config

### Official Sources

- https://mozilla.github.io/pdf.js/api/ -- PDF.js API reference (exception types)
- https://github.com/mozilla/pdf.js -- Source code (exception definitions in src/shared/util.js)
- https://github.com/mozilla/pdf.js/tree/master/examples -- Official examples with error handling
