# Scaffolding Decision Tree and Configuration Options (pdfjs-dist 5.x)

## Setup Selection Decision Tree

```
What is your project context?
│
├── No build tools, just HTML files?
│   └── VANILLA CDN
│       Pros: Zero config, instant start, no node_modules
│       Cons: No tree-shaking, no TypeScript, CDN dependency
│       When: Prototypes, demos, learning, CodePen/JSFiddle
│
├── Existing webpack project / need fine-grained control?
│   └── WEBPACK
│       Pros: Full control, established ecosystem, code splitting
│       Cons: More config, copy-webpack-plugin required for worker
│       When: Enterprise apps, legacy codebases, complex build needs
│
├── New project / modern tooling?
│   └── VITE
│       Pros: Fastest dev server, simplest worker config, native ESM
│       Cons: Less mature plugin ecosystem than webpack
│       When: New projects, SPAs, anything without legacy constraints
│
└── Server-side rendering / React framework?
    └── NEXT.JS
        Pros: SSR, React ecosystem, file-based routing
        Cons: Worker must be loaded client-side only, dynamic imports needed
        When: React apps with SSR, full-stack applications
```

---

## package.json Configuration

### Minimal (All Setups)

```json
{
  "name": "pdfjs-viewer",
  "private": true,
  "version": "1.0.0",
  "type": "module",
  "dependencies": {
    "pdfjs-dist": "^5.0.0"
  }
}
```

### Webpack Additions

```json
{
  "devDependencies": {
    "webpack": "^5.90.0",
    "webpack-cli": "^5.1.0",
    "webpack-dev-server": "^5.0.0",
    "html-webpack-plugin": "^5.6.0",
    "copy-webpack-plugin": "^12.0.0",
    "ts-loader": "^9.5.0",
    "css-loader": "^7.1.0",
    "style-loader": "^4.0.0",
    "typescript": "^5.5.0"
  },
  "scripts": {
    "dev": "webpack serve --mode development",
    "build": "webpack --mode production"
  }
}
```

### Vite Additions

```json
{
  "devDependencies": {
    "vite": "^6.0.0",
    "typescript": "^5.5.0"
  },
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  }
}
```

### Next.js Additions

```json
{
  "dependencies": {
    "next": "^15.0.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "pdfjs-dist": "^5.0.0"
  },
  "devDependencies": {
    "@types/react": "^19.0.0",
    "@types/node": "^22.0.0",
    "typescript": "^5.5.0"
  },
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start"
  }
}
```

---

## Worker Configuration Details

### How the Worker Works

PDF.js offloads PDF parsing to a Web Worker to keep the UI thread responsive. The worker file (`pdf.worker.min.mjs`) MUST be loaded as a separate file, NOT bundled into your main JavaScript.

### Worker Path by Setup

| Setup | Method | Code |
|-------|--------|------|
| Vanilla CDN | CDN URL string | `GlobalWorkerOptions.workerSrc = "https://cdnjs.cloudflare.com/ajax/libs/pdf.js/5.x.x/pdf.worker.min.mjs"` |
| Webpack | copy-webpack-plugin + import.meta.url | `GlobalWorkerOptions.workerSrc = new URL("pdfjs-dist/build/pdf.worker.min.mjs", import.meta.url).toString()` |
| Vite | Native ESM import.meta.url | `GlobalWorkerOptions.workerSrc = new URL("pdfjs-dist/build/pdf.worker.min.mjs", import.meta.url).toString()` |
| Next.js | Public folder or CDN | `GlobalWorkerOptions.workerSrc = "/pdf.worker.min.mjs"` |

### Worker Version Matching Rule

The worker file version MUST match the pdfjs-dist package version exactly. NEVER load a worker from a different version than the installed pdfjs-dist package. A mismatch causes:
- Deserialization errors
- Silent failures during PDF parsing
- Cryptic "Invalid PDF structure" errors

---

## TypeScript Configuration

### tsconfig.json (Webpack and Vite)

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "types": ["pdfjs-dist"]
  },
  "include": ["src"]
}
```

### Key Settings Explained

| Setting | Value | Why |
|---------|-------|-----|
| `target` | ES2020 | Required for `import.meta.url` support |
| `module` | ESNext | Required for dynamic imports and ESM |
| `moduleResolution` | bundler | Matches webpack 5 / Vite resolution |
| `lib` includes DOM | Required | PDF.js operates on DOM elements (canvas, div) |

---

## Bundler Configuration Details

### Webpack: Worker File Handling

Webpack does NOT automatically copy the worker file. You MUST use `copy-webpack-plugin`:

```javascript
// In webpack.config.js plugins array
new CopyWebpackPlugin({
  patterns: [
    {
      from: "node_modules/pdfjs-dist/build/pdf.worker.min.mjs",
      to: "pdf.worker.min.mjs",
    },
  ],
}),
```

**Alternative**: Use `import.meta.url` with webpack 5's built-in asset modules:

```typescript
// This works in webpack 5 with experiments.outputModule enabled
GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url
).toString();
```

For this to work, add to webpack.config.js:
```javascript
experiments: {
  outputModule: true,
},
```

### Vite: Zero-Config Worker

Vite handles `import.meta.url` natively. No plugins or special configuration needed for the worker file. Vite automatically resolves and serves the worker file during development and bundles it correctly for production.

### Next.js: Server Component Restrictions

PDF.js uses browser APIs (canvas, window). ALWAYS:
1. Mark the viewer component with `"use client"` directive
2. Use dynamic import with `ssr: false`
3. Load the worker from `/public` or a CDN

---

## CSS Layer Stacking Reference

### Required Layer Order

| Layer | CSS Property | z-index | Purpose |
|-------|-------------|---------|---------|
| Canvas | `position: absolute` | 0 (base) | Rendered PDF pixels |
| TextLayer | `position: absolute` | 1 | Transparent selectable text over canvas |
| AnnotationLayer | `position: absolute` | 2 | Clickable links, form fields |

### Container Requirements

The page container MUST have `position: relative` so that absolutely positioned child layers stack correctly within it.

### Official Stylesheet

ALWAYS import the official PDF.js viewer CSS:

```css
@import "pdfjs-dist/web/pdf_viewer.css";
```

This stylesheet contains critical styles for:
- Text layer span positioning and opacity
- Annotation layer element positioning
- Form widget styling
- Highlight and selection styling

Without this stylesheet, text selection will not work and annotations will be mispositioned.

---

## Navigation Implementation Reference

### Page Navigation API

```typescript
// Get total pages
const totalPages: number = pdfDoc.numPages;

// Get specific page (1-indexed, NEVER 0-indexed)
const page = await pdfDoc.getPage(pageNumber);

// Page number validation
function isValidPage(num: number, total: number): boolean {
  return Number.isInteger(num) && num >= 1 && num <= total;
}
```

### Zoom Levels

| Action | Scale Change | Common Implementation |
|--------|-------------|----------------------|
| Zoom in | Multiply by 1.25 | `scale *= 1.25` |
| Zoom out | Divide by 1.25 | `scale /= 1.25` |
| Fit to width | Calculate from container | `containerWidth / baseViewport.width` |
| Fit to page | Calculate from container | `Math.min(containerWidth / w, containerHeight / h)` |
| Actual size | Set to 1.0 | `scale = 1.0` |

### Keyboard Shortcuts (Common)

| Key | Action |
|-----|--------|
| ArrowLeft / PageUp | Previous page |
| ArrowRight / PageDown | Next page |
| Home | First page |
| End | Last page |
| + / = | Zoom in |
| - | Zoom out |
| 0 | Reset zoom to 100% |
