# REQUIREMENTS

## What This Skill Package Must Achieve

### Primary Goal
Enable Claude to write correct, version-aware PDF.js code for rendering and interacting with PDF documents in the browser — without hallucinating APIs.

### What Claude Should Do After Loading Skills
1. Recognize PDF.js context from user requests (PDF viewing, rendering, text extraction)
2. Select the correct skill(s) automatically based on the request
3. Write correct TypeScript/JavaScript code using pdfjs-dist 5.x APIs
4. Avoid known anti-patterns and common AI mistakes
5. Follow best practices documented in the skill references

### Quality Guarantees
| Guarantee | Description |
|-----------|-------------|
| Version-correct | Code MUST target pdfjs-dist 5.x |
| API-accurate | All method signatures verified against official docs |
| Worker-aware | Skills MUST address worker setup correctly |
| Anti-pattern-free | Known mistakes are explicitly documented and avoided |
| Deterministic | Skills use ALWAYS/NEVER language, not suggestions |
| Self-contained | Each skill works independently without requiring other skills |

---

## Per-Area Requirements

### 1. Viewer API
| Requirement | Detail |
|-------------|--------|
| Document loading | `getDocument()` with various source types (URL, ArrayBuffer, typed array) |
| Page rendering | `page.render()` with canvas context |
| Viewport | `page.getViewport()` with scale, rotation, offset |
| Loading task | `PDFDocumentLoadingTask`, progress callbacks, cancellation |
| Critical | Worker MUST be configured before any document loading |

### 2. Worker API
| Requirement | Detail |
|-------------|--------|
| Setup | `GlobalWorkerOptions.workerSrc` configuration |
| CDN usage | Correct CDN URLs for worker file matching pdfjs-dist version |
| Webpack/bundler | Worker configuration for bundlers (webpack, vite, rollup) |
| Fallback | Fake worker mode for environments without Web Worker support |
| Critical | Worker version MUST match pdfjs-dist version exactly |

### 3. Rendering Pipeline
| Requirement | Detail |
|-------------|--------|
| Canvas rendering | Canvas 2D context rendering with proper DPI handling |
| SVG rendering | SVG rendering alternative |
| Render task | `RenderTask` lifecycle, cancellation, completion |
| High DPI | `window.devicePixelRatio` handling for crisp rendering |
| Critical | ALWAYS cancel previous render task before starting new one |

### 4. Text Layer
| Requirement | Detail |
|-------------|--------|
| Text content | `page.getTextContent()` for extracting text items |
| Text layer | `TextLayer` class for selectable/searchable text overlay |
| Text extraction | Extracting plain text from PDF pages |
| Styling | CSS for text layer positioning and visibility |
| Critical | Text layer MUST be positioned absolutely over canvas |

### 5. Annotation Layer
| Requirement | Detail |
|-------------|--------|
| Annotations | `page.getAnnotations()` for reading annotations |
| Annotation layer | `AnnotationLayer` class for interactive annotations |
| Link handling | Handling internal and external link annotations |
| Form widgets | Interactive form field annotations |
| Critical | Annotation layer MUST be rendered on top of text layer |

### 6. Custom Viewer
| Requirement | Detail |
|-------------|--------|
| Page navigation | Previous/next, go to page, page count |
| Zoom controls | Zoom in/out, fit to page, fit to width, custom scale |
| Scroll handling | Scroll-based page loading, virtual scrolling |
| Thumbnails | Thumbnail generation from pages |
| Search | Text search across pages with highlighting |
| Print | Print functionality with proper page formatting |
| Critical | ALWAYS implement lazy loading — never render all pages at once |

---

## Critical Requirements (apply to ALL skills)

- All code MUST work with pdfjs-dist 5.x
- Worker setup MUST be shown in every skill that loads documents
- All TypeScript MUST include proper type imports from `pdfjs-dist`
- Canvas rendering MUST handle `devicePixelRatio` for high-DPI displays
- Code examples MUST be verified against official documentation
- Render tasks MUST be properly cancelled before re-rendering

---

## Structural Requirements

### Skill Format
- SKILL.md < 500 lines (heavy content in references/)
- YAML frontmatter with name and description (including trigger words)
- English-only content
- Deterministic language (ALWAYS/NEVER, imperative)

### Skill Categories
| Category | Purpose | Must Include |
|----------|---------|--------------|
| syntax/ | How to write it | Method signatures, code patterns, type annotations |
| impl/ | How to build it | Decision trees, workflows, step-by-step |
| errors/ | How to handle failures | Error patterns, diagnostics, recovery |
| core/ | Cross-cutting | Architecture, rendering pipeline, worker model |
| agents/ | Orchestration | Validation checklists, auto-detection |

---

## Research Requirements (before creating any skill)

1. Official documentation MUST be consulted and referenced
2. Source code MUST be checked for accuracy when docs are ambiguous
3. Anti-patterns MUST be identified from real issues (GitHub issues)
4. Code examples MUST be verified (not hallucinated)
5. Version accuracy MUST be confirmed via WebFetch (D-012)

---

## Non-Requirements (explicitly out of scope)

- No pdf-lib coverage (separate package — pdf-lib-Claude-Skill-Package)
- No PDF creation/modification (PDF.js is a viewer/renderer, not a creator)
- No server-side rendering with Node.js canvas (browser-focused)
- No comparison guides with other PDF viewers
- No React/Vue/Angular component library tutorials (skills cover the core API)
