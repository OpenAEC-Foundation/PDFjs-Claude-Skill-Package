# Approved Sources Registry

All skills in this package MUST be verified against these approved sources. No other sources are authoritative.

## Official Documentation

| Source | URL | Purpose |
|--------|-----|---------|
| PDF.js Website | https://mozilla.github.io/pdf.js/ | Primary reference for PDF.js |
| PDF.js API Docs | https://mozilla.github.io/pdf.js/api/ | Complete API reference |
| PDF.js Getting Started | https://mozilla.github.io/pdf.js/getting_started/ | Setup and usage guides |

## Source Code

| Source | URL | Purpose |
|--------|-----|---------|
| PDF.js GitHub | https://github.com/mozilla/pdf.js | Main repository, source code, issues |
| PDF.js Examples | https://github.com/mozilla/pdf.js/tree/master/examples | Official code examples |
| PDF.js Web Viewer | https://github.com/mozilla/pdf.js/tree/master/web | Reference viewer implementation |
| PDF.js Wiki | https://github.com/mozilla/pdf.js/wiki | Architecture docs, developer guides |

## Package Documentation

| Source | URL | Purpose |
|--------|-----|---------|
| pdfjs-dist npm | https://www.npmjs.com/package/pdfjs-dist | npm package, version info |
| pdfjs-dist types | https://github.com/nicolo-ribaudo/pdfjs-dist-types | TypeScript type definitions |

## PDF Specification

| Source | URL | Purpose |
|--------|-----|---------|
| PDF Reference 1.7 | https://opensource.adobe.com/dc-acrobat-sdk-docs/pdfstandards/PDF32000_2008.pdf | PDF format specification |

## Community (for Anti-Pattern Research)

| Source | URL | Purpose |
|--------|-----|---------|
| GitHub Issues | https://github.com/mozilla/pdf.js/issues | Real-world error patterns |
| Stack Overflow | https://stackoverflow.com/questions/tagged/pdf.js | Common questions and pitfalls |

## Claude / Anthropic (Skill Development Platform)

| Source | URL | Purpose |
|--------|-----|---------|
| Agent Skills Standard | https://agentskills.io | Open standard |
| Agent Skills Spec | https://github.com/agentskills/agentskills | Specification |
| Agent Skills Best Practices | https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices | Authoring guide |

## OpenAEC Foundation

| Source | URL | Purpose |
|--------|-----|---------|
| ERPNext Skill Package | https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package | Methodology template |
| Tauri 2 Skill Package | https://github.com/OpenAEC-Foundation/Tauri-2-Claude-Skill-Package | Methodology template |
| pdf-lib Skill Package | https://github.com/OpenAEC-Foundation/pdf-lib-Claude-Skill-Package | Companion package (PDF creation) |

## Source Verification Rules

1. **Primary sources ONLY** — Official docs, official repo, npm package docs.
2. **NEVER trust random blog posts** — Even popular ones may be outdated or wrong for v4.
3. **Verify code against official docs** — Every code snippet in a skill MUST match current API.
4. **Note when source was last verified** — Track in the table below.
5. **Cross-reference if docs are sparse** — When official docs lack detail, verify against source code and examples.

## Last Verified

| Technology | Date | Action | Notes |
|------------|------|--------|-------|
| PDF.js Website | 2026-03-19 | Verified | Getting started guide confirmed |
| PDF.js API Docs | 2026-03-19 | Verified | API reference for v5.x confirmed |
| PDF.js GitHub | 2026-03-19 | Verified | Source code cross-referenced for v5 API |
| pdfjs-dist npm | 2026-03-19 | Verified | v5.5.207 confirmed as latest |
| PDF.js Wiki | 2026-03-19 | Verified | Architecture docs reviewed |
| PDF.js Examples | 2026-03-19 | Verified | Official examples checked |
