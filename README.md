# PDF.js Claude Skill Package

![Claude Code Ready](https://img.shields.io/badge/Claude_Code-Ready-blue?style=flat-square&logo=data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCAyNCAyNCI+PHBhdGggZD0iTTEyIDJDNi40OCAyIDIgNi40OCAyIDEyczQuNDggMTAgMTAgMTAgMTAtNC40OCAxMC0xMFMxNy41MiAyIDEyIDJ6IiBmaWxsPSIjZmZmIi8+PC9zdmc+)
![pdfjs-dist 4.x](https://img.shields.io/badge/pdfjs--dist-4.x-FF7139?style=flat-square)
![TypeScript](https://img.shields.io/badge/TypeScript-Ready-3178C6?style=flat-square&logo=typescript&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)

**Deterministic Claude AI skills for PDF.js (Mozilla) PDF rendering and viewer development — TypeScript/JavaScript coverage.**

Built on the [Agent Skills](https://agentskills.org) open standard.

---

## Why This Exists

Without skills, Claude generates incorrect PDF.js code:

```javascript
// Wrong — missing worker setup, no DPI handling, deprecated API
const pdf = await pdfjsLib.getDocument('document.pdf');
const page = await pdf.getPage(1);
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');
page.render({ canvasContext: ctx, viewport: page.getViewport(1.0) });
```

With this skill package, Claude produces correct PDF.js code:

```typescript
// Correct — worker configured, proper viewport, DPI-aware rendering
import * as pdfjsLib from 'pdfjs-dist';

pdfjsLib.GlobalWorkerOptions.workerSrc = `//unpkg.com/pdfjs-dist@${pdfjsLib.version}/build/pdf.worker.min.mjs`;

const loadingTask = pdfjsLib.getDocument('document.pdf');
const pdf = await loadingTask.promise;
const page = await pdf.getPage(1);
const scale = 1.5;
const viewport = page.getViewport({ scale });
const canvas = document.getElementById('canvas') as HTMLCanvasElement;
const ctx = canvas.getContext('2d')!;
const dpr = window.devicePixelRatio || 1;
canvas.width = Math.floor(viewport.width * dpr);
canvas.height = Math.floor(viewport.height * dpr);
canvas.style.width = `${Math.floor(viewport.width)}px`;
canvas.style.height = `${Math.floor(viewport.height)}px`;
ctx.scale(dpr, dpr);
await page.render({ canvasContext: ctx, viewport }).promise;
```

---

## Current Progress

**Phase 1: Setup + Raw Masterplan** — 50% complete

| Phase | Status |
|-------|--------|
| 1. Setup + Raw Masterplan | IN PROGRESS |
| 2. Deep Research | NOT STARTED |
| 3. Masterplan Refinement | NOT STARTED |
| 4. Topic-Specific Research | NOT STARTED |
| 5. Skill Creation | NOT STARTED |
| 6. Validation | NOT STARTED |
| 7. Publication | NOT STARTED |

## Skill Categories

| Category | Description |
|----------|-------------|
| `syntax/` | API syntax, method signatures, type patterns |
| `impl/` | Step-by-step development workflows and integration guides |
| `errors/` | Error diagnosis, debugging patterns, anti-patterns |
| `core/` | Cross-cutting architecture, rendering pipeline, worker model |
| `agents/` | Intelligent orchestration for viewer generation and review |

## Installation

### Claude Code

```bash
# Option 1: Clone the full package
git clone https://github.com/OpenAEC-Foundation/PDFjs-Claude-Skill-Package.git
cp -r PDFjs-Claude-Skill-Package/skills/source/ ~/.claude/skills/pdfjs/

# Option 2: Add as git submodule
git submodule add https://github.com/OpenAEC-Foundation/PDFjs-Claude-Skill-Package.git .claude/skills/pdfjs
```

### Claude.ai (Web)

Upload individual SKILL.md files as project knowledge.

## Version Compatibility

| Technology | Versions | Notes |
|------------|----------|-------|
| pdfjs-dist | **4.x** | Primary target |
| TypeScript | 4.x / 5.x | Type safety |
| Node.js | 18+ | Build tooling |
| Browsers | Modern (Chrome, Firefox, Safari, Edge) | Full support |

## Methodology

This package is developed using the **7-phase research-first methodology**, proven across multiple skill packages:

1. **Setup + Raw Masterplan** — Project structure and governance files
2. **Deep Research** — Comprehensive source analysis of PDF.js documentation, source code, and community resources
3. **Masterplan Refinement** — Skill inventory refinement based on research findings
4. **Topic-Specific Research** — Deep-dive per skill topic
5. **Skill Creation** — Deterministic skill files following Agent Skills standard
6. **Validation** — Correctness, completeness, and consistency checks
7. **Publication** — GitHub release and documentation

## Documentation

| Document | Purpose |
|----------|---------|
| [ROADMAP.md](ROADMAP.md) | Project status (single source of truth) |
| [REQUIREMENTS.md](REQUIREMENTS.md) | Quality guarantees and per-area requirements |
| [DECISIONS.md](DECISIONS.md) | Architectural decisions with rationale |
| [SOURCES.md](SOURCES.md) | Official reference URLs and verification rules |
| [WAY_OF_WORK.md](WAY_OF_WORK.md) | 7-phase development methodology |
| [LESSONS.md](LESSONS.md) | Lessons learned during development |
| [CHANGELOG.md](CHANGELOG.md) | Version history |

## Related Projects

| Project | Description |
|---------|-------------|
| [pdf-lib Skill Package](https://github.com/OpenAEC-Foundation/pdf-lib-Claude-Skill-Package) | Skills for pdf-lib PDF creation/modification |
| [ERPNext Skill Package](https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package) | 28 skills for ERPNext/Frappe development |
| [Tauri 2 Skill Package](https://github.com/OpenAEC-Foundation/Tauri-2-Claude-Skill-Package) | 27 skills for Tauri 2 desktop applications |
| [Blender-Bonsai Skill Package](https://github.com/OpenAEC-Foundation/Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package) | 73 skills for Blender, Bonsai, IfcOpenShell & Sverchok |
| [OpenAEC Foundation](https://github.com/OpenAEC-Foundation) | Parent organization |

## License

[MIT](LICENSE)

---

Part of the [OpenAEC Foundation](https://github.com/OpenAEC-Foundation) ecosystem.
