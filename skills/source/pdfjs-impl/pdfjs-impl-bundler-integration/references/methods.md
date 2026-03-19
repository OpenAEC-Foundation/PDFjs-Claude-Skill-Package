# Bundler Configuration Reference (pdfjs-dist 5.x)

## GlobalWorkerOptions

Configuration object for the PDF.js worker. MUST be set before calling `getDocument()`.

```typescript
import { GlobalWorkerOptions } from "pdfjs-dist";

// Option 1: Set worker source URL (worker created internally)
GlobalWorkerOptions.workerSrc: string;

// Option 2: Set worker port directly (preferred -- you control the Worker instance)
GlobalWorkerOptions.workerPort: Worker;
```

### workerSrc

| Property | Detail |
|----------|--------|
| Type | `string` |
| Default | `"./pdf.worker.mjs"` (fallback, rarely resolves correctly) |
| Purpose | URL path to the worker script file |
| When to use | When the bundler does not support `new Worker(new URL(...))` |

### workerPort

| Property | Detail |
|----------|--------|
| Type | `Worker` |
| Default | `undefined` |
| Purpose | Pre-created Worker instance |
| When to use | ALWAYS prefer this when your bundler supports `new URL(..., import.meta.url)` |

**Note**: Setting `workerPort` takes precedence over `workerSrc`. When `workerPort` is set, `workerSrc` is ignored.

---

## getDocument() -- Bundler-Relevant Parameters

```typescript
import { getDocument } from "pdfjs-dist";

const loadingTask = getDocument({
  url: string;                    // URL to PDF file
  cMapUrl?: string;               // Base URL for CMap files (e.g., "/cmaps/")
  cMapPacked?: boolean;           // ALWAYS true for pdfjs-dist (ships .bcmap files)
  standardFontDataUrl?: string;   // Base URL for standard fonts (e.g., "/standard_fonts/")
  useWorkerFetch?: boolean;       // Let worker fetch CMaps/fonts directly (default: auto)
  isEvalSupported?: boolean;      // Allow eval() for font rendering (default: true)
});
```

### cMapUrl

| Property | Detail |
|----------|--------|
| Type | `string` |
| Required | Only for CJK PDFs |
| Value | URL path ending with `/` pointing to copied `.bcmap` files |
| Example | `"/cmaps/"`, `"/assets/cmaps/"` |
| Source | `node_modules/pdfjs-dist/cmaps/` (169 `.bcmap` files) |

### cMapPacked

| Property | Detail |
|----------|--------|
| Type | `boolean` |
| Default | `true` |
| Value | ALWAYS `true` when using pdfjs-dist -- the package ships binary-packed `.bcmap` files |

### standardFontDataUrl

| Property | Detail |
|----------|--------|
| Type | `string` |
| Required | Only for PDFs that reference standard 14 fonts without embedding them |
| Value | URL path ending with `/` pointing to copied font files |
| Example | `"/standard_fonts/"`, `"/assets/fonts/"` |
| Source | `node_modules/pdfjs-dist/standard_fonts/` (16 files) |

### useWorkerFetch

| Property | Detail |
|----------|--------|
| Type | `boolean` |
| Default | Auto-detected based on whether cMapUrl and standardFontDataUrl are valid fetch URLs |
| Purpose | When `true`, the worker fetches CMap and font files directly (more efficient). When `false`, the main thread fetches them. |
| Note | ALWAYS leave as default unless you have specific CORS or CSP constraints |

### isEvalSupported

| Property | Detail |
|----------|--------|
| Type | `boolean` |
| Default | `true` |
| Purpose | Controls whether `eval()` can be used for font rendering optimizations |
| Note | Set to `false` when Content Security Policy (CSP) blocks `eval()`. This slightly reduces font rendering performance but is required for strict CSP environments. |

---

## webpack.mjs Entry Point

The `pdfjs-dist/webpack.mjs` file is a convenience wrapper that auto-configures the worker.

```typescript
// What pdfjs-dist/webpack.mjs contains:
import { GlobalWorkerOptions } from "./build/pdf.mjs";

if (typeof window !== "undefined" && "Worker" in window) {
  GlobalWorkerOptions.workerPort = new Worker(
    new URL("./build/pdf.worker.mjs", import.meta.url),
    { type: "module" }
  );
}

export * from "./build/pdf.mjs";
```

### How It Works

1. Creates a `Worker` using `new URL("./build/pdf.worker.mjs", import.meta.url)`
2. Webpack 5+ and Vite recognize this pattern and emit the worker as a separate asset
3. Sets `GlobalWorkerOptions.workerPort` to the created Worker instance
4. Re-exports everything from `build/pdf.mjs`

### Compatibility

| Bundler | Supported | Notes |
|---------|-----------|-------|
| Webpack 5+ | Yes | Native `new URL()` + `new Worker()` support |
| Vite | Yes | Native ESM support with `import.meta.url` |
| Rollup | Partial | Requires `@rollup/plugin-url` or manual handling |
| esbuild | No | Does not support `new URL(..., import.meta.url)` for workers |
| Webpack 4 | No | Does not support the `new URL()` worker pattern |

---

## Standard 14 PDF Fonts

These fonts are referenced by name in PDFs but may not be embedded. When a PDF references these fonts without embedding them, pdfjs-dist needs the font files.

| Font Name | File in standard_fonts/ |
|-----------|------------------------|
| Courier | FoxitFixed.pfb |
| Courier-Bold | FoxitFixedBold.pfb |
| Courier-BoldOblique | FoxitFixedBoldItalic.pfb |
| Courier-Oblique | FoxitFixedItalic.pfb |
| Helvetica | LiberationSans-Regular.ttf |
| Helvetica-Bold | LiberationSans-Bold.ttf |
| Helvetica-BoldOblique | LiberationSans-BoldItalic.ttf |
| Helvetica-Oblique | LiberationSans-Italic.ttf |
| Times-Roman | FoxitSerif.pfb |
| Times-Bold | FoxitSerifBold.pfb |
| Times-BoldItalic | FoxitSerifBoldItalic.pfb |
| Times-Italic | FoxitSerifItalic.pfb |
| Symbol | FoxitSymbol.pfb |
| ZapfDingbats | FoxitDingbats.pfb |
