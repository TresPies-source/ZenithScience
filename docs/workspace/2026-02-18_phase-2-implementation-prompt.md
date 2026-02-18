# Implementation Commission: Phase 2 — blog-mcp v0.2 + grant-mcp v0.4

**Objective:** Implement `blog-mcp` (markdown to responsive HTML with SEO and RSS) and `grant-mcp` (academic grant proposals in LaTeX/Word format), proving that `packages/core` abstractions are format-agnostic and that the ZenSci pipeline works across document types beyond PDF.

**Commissioned by:** Cruz Morales / TresPiesDesign.com
**Date:** 2026-02-18
**License:** Apache 2.0
**Prereqs:** Phase 1 complete. `pnpm --filter @zen-sci/latex-mcp run test` passes (including at least one integration test producing a real PDF). This prerequisite is non-negotiable: blog-mcp exists to *prove* that packages/core works for a second format. If packages/core isn't validated by latex-mcp, blog-mcp can't confirm anything.
**System deps:** Node.js >= 20 LTS, Python >= 3.11, pandoc >= 3.0, pypandoc, katex (npm), highlight.js (npm)

---

## 1. Context & Grounding

**Primary Specifications (read fully before writing):**
- `ZenithScience/specs/phase-1/blog-mcp-v0.2-spec.md` — blog-mcp architecture, MCP tool definitions, HTML rendering engine, SEO metadata, RSS feed, accessibility targets. Read in full.
- `ZenithScience/specs/phase-2/grant-mcp-spec.md` — grant-mcp architecture, NIH/NSF/ERC format support, GrantSubmissionFormat types (§3.6), Python engine, compliance validation. Read in full.
- `ZenithScience/specs/infrastructure/packages-core-spec.md §3.4` — `WebMetadataSchema`, `OpenGraphMetadata`, `TwitterCardMetadata`, `SchemaOrgData` (these are the canonical types blog-mcp builds on).

**Pattern Files:**
- `zen-sci/servers/latex-mcp/src/index.ts` — Follow the exact same `createZenSciServer()` + `server.registerTool()` pattern for both blog-mcp and grant-mcp server entry points.
- `zen-sci/servers/latex-mcp/src/tools/convert-to-pdf.ts` — Follow the same tool handler structure: validate args → build DocumentRequest → run pipeline → return structured response.
- `zen-sci/servers/latex-mcp/engine/latex_engine.py` — Follow the same stdin/stdout JSON pattern for Python engines.

**Critical Validation Goal for blog-mcp:**
`blog-mcp-v0.2-spec.md §1` states that blog-mcp validates the "format-agnostic core abstraction" thesis. Specifically: the blog module MUST reuse `MarkdownParser`, `CitationManager`, `SchemaValidator`, and `MathValidator` from `@zen-sci/core` without modification. If you find yourself reimplementing any of these in blog-mcp, stop and fix the core abstraction instead.

---

## 2. Detailed Requirements

### 2.1 — Scaffold servers/blog-mcp

1. Create `zen-sci/servers/blog-mcp/` directory structure:
   ```
   zen-sci/servers/blog-mcp/
   ├── src/
   │   ├── index.ts
   │   ├── manifest.ts
   │   ├── tools/
   │   │   ├── convert-to-html.ts
   │   │   ├── generate-feed.ts
   │   │   └── validate-post.ts
   │   └── rendering/
   │       ├── html-renderer.ts       (Markdown AST → HTML)
   │       ├── seo-builder.ts         (OpenGraph, Twitter Card, JSON-LD)
   │       ├── syntax-highlighter.ts  (highlight.js integration)
   │       ├── katex-renderer.ts      (math → HTML via KaTeX)
   │       └── rss-generator.ts       (RSS/Atom feed XML)
   ├── engine/
   │   ├── html_engine.py             (pandoc-based HTML generation)
   │   └── requirements.txt
   ├── tests/
   │   ├── unit/
   │   │   ├── html-renderer.test.ts
   │   │   ├── seo-builder.test.ts
   │   │   └── rss-generator.test.ts
   │   └── integration/
   │       └── end-to-end.test.ts
   ├── fixtures/
   │   ├── sample-post.md
   │   ├── sample-post.bib
   │   └── expected-meta.json
   ├── package.json
   ├── tsconfig.json
   └── README.md
   ```

2. Create `zen-sci/servers/blog-mcp/package.json`:
   ```json
   {
     "name": "@zen-sci/blog-mcp",
     "version": "0.2.0",
     "description": "ZenSci Blog MCP: markdown to responsive HTML with SEO and RSS",
     "type": "module",
     "main": "./dist/index.js",
     "bin": { "zen-sci-blog": "./dist/cli.js" },
     "scripts": {
       "build": "tsc --build",
       "start": "node dist/index.js",
       "test": "vitest run",
       "typecheck": "tsc --noEmit"
     },
     "dependencies": {
       "@modelcontextprotocol/sdk": "^1.5.0",
       "@zen-sci/core": "workspace:*",
       "@zen-sci/sdk": "workspace:*",
       "katex": "^0.16.9",
       "highlight.js": "^11.9.0"
     },
     "devDependencies": {
       "@types/node": "^20.0.0",
       "typescript": "^5.3.0",
       "vitest": "^1.2.0"
     }
   }
   ```

3. Create `zen-sci/servers/blog-mcp/engine/requirements.txt`: `pypandoc>=1.12`

### 2.2 — Implement blog-mcp manifest and SEO types

4. Implement `src/manifest.ts` for blog-mcp:
   - `name: 'blog-mcp'`, `version: '0.2.0'`
   - `supportedFormats: ['html', 'email']`
   - `capabilities: ['seo-metadata', 'rss-feed', 'syntax-highlighting', 'math-rendering', 'accessibility-validation']`

5. Implement `src/rendering/seo-builder.ts`:
   - `buildSEOMetadata(frontmatter: FrontmatterMetadata, options: SEOOptions): WebMetadataSchema` — builds the full `WebMetadataSchema` from `@zen-sci/core` types
   - `SEOOptions` interface: `{ siteUrl?: string; siteName?: string; twitterHandle?: string; defaultImage?: string }`
   - **Use `WebMetadataSchema`, `OpenGraphMetadata`, `TwitterCardMetadata`, `SchemaOrgData` from `@zen-sci/core/types` directly** — do NOT redefine these types
   - `buildOpenGraph(frontmatter, options): OpenGraphMetadata` — maps frontmatter fields; `type` is always `'article'` for blog posts
   - `buildTwitterCard(frontmatter, options): TwitterCardMetadata` — `card: 'summary_large_image'` if image present, else `'summary'`
   - `buildSchemaOrg(frontmatter, options): SchemaOrgData` — produces `@type: 'BlogPosting'` with all spec-required fields
   - `renderToHtmlHead(meta: WebMetadataSchema): string` — returns HTML `<meta>` and `<script type="application/ld+json">` tags as string

6. Implement `src/rendering/katex-renderer.ts`:
   - `renderInlineMath(expression: string): string` — calls `katex.renderToString(expression, { throwOnError: false, displayMode: false })`
   - `renderDisplayMath(expression: string): string` — same but `displayMode: true`
   - `renderAllMath(html: string): string` — post-processes HTML to replace `$...$` and `$$...$$` delimiters with KaTeX output
   - On KaTeX error: return `<span class="math-error" title="{error}">{originalExpression}</span>`

7. Implement `src/rendering/syntax-highlighter.ts`:
   - `highlight(code: string, language?: string): string` — uses highlight.js `hljs.highlight()` for known languages, `hljs.highlightAuto()` for unknown
   - Returns highlighted HTML string with `<code class="hljs language-{lang}">` wrapping
   - `getEmbeddedCSS(): string` — returns the highlight.js CSS for inline embedding (use a minimal theme like 'github')

8. Implement `src/rendering/html-renderer.ts`:
   - `HTMLRenderer` class that converts a `DocumentTree` (from `@zen-sci/core`) to HTML string
   - Use pandoc via `PandocWrapper` (from `@zen-sci/sdk`) for the core markdown → HTML conversion: `pandoc.convert(source, 'markdown', 'html5', ['--no-highlight', '--mathjax='])`
   - Post-process pandoc HTML output:
     1. Apply KaTeX rendering to all math elements (replace MathJax placeholders)
     2. Apply syntax highlighting to all `<code>` blocks
     3. Add `id` attributes to all headings for anchor links
     4. Add `class="zen-sci-figure"` to all `<img>` wrappers
     5. Add `target="_blank" rel="noopener noreferrer"` to external links
   - `render(source: string, frontmatter: FrontmatterMetadata, meta: WebMetadataSchema, options: HTMLRenderOptions): string` — returns complete, standalone HTML page
   - `HTMLRenderOptions`: `{ toc?: boolean; selfContained?: boolean; cssTheme?: 'light' | 'dark' | 'system'; embedCSS?: boolean }`
   - Standalone HTML template: include `<html lang="{lang}">`, proper `<head>` with charset/viewport/meta tags, SEO meta tags, optional TOC, KaTeX CSS, highlight.js CSS, main content `<article>`, bibliography section

9. Implement `src/rendering/rss-generator.ts`:
   - `generateFeed(posts: PostMetadata[], options: FeedOptions): string` — generates Atom 1.0 XML
   - `PostMetadata`: `{ title: string; url: string; date: string; author?: string; summary?: string; content?: string; tags?: string[] }`
   - `FeedOptions`: `{ feedTitle: string; feedUrl: string; siteUrl: string; description?: string; author?: string }`
   - Output Atom 1.0 per RFC 4287; include `<updated>`, `<id>`, `<link>`, `<entry>` elements
   - Validate: `date` must be valid ISO 8601; `url` must be non-empty; `feedTitle` must be non-empty

### 2.3 — Implement blog-mcp tools and server

10. Implement `src/tools/convert-to-html.ts`:
    - Args: `{ source, title, author?, bibliography?, options?: { toc, theme, selfContained, seoSiteUrl, seoSiteName, twitterHandle } }`
    - Build `DocumentRequest` from args
    - Run: SchemaValidator → MarkdownParser → CitationManager (if bibliography) → MathValidator → HTMLRenderer + SEOBuilder
    - Return: `{ html, page_url?, metadata: { title, author, word_count, reading_time_minutes }, seo: WebMetadataSchema, warnings, citations }`
    - `reading_time_minutes`: word_count / 200 (average reading speed), rounded up

11. Implement `src/tools/generate-feed.ts`:
    - Args: `{ posts: PostMetadata[], feedTitle, feedUrl, siteUrl, description? }`
    - Call `RSSGenerator.generateFeed()`
    - Return: `{ feed_xml, post_count, generated_at }`

12. Implement `src/tools/validate-post.ts`:
    - Args: `{ source, bibliography? }`
    - Run full validation suite: SchemaValidator, MarkdownParser, MathValidator, CitationManager, accessibility check (WCAG heading hierarchy)
    - Return: `{ valid, errors, warnings, accessibility_issues, citation_stats, math_stats }`

13. Implement `src/index.ts` — blog-mcp MCP server entry point:
    - Follow exact same pattern as `servers/latex-mcp/src/index.ts`
    - Register tools: `convert_to_html`, `generate_feed`, `validate_post` (snake_case)
    - Tool input schemas per `blog-mcp-v0.2-spec.md §3.2`

### 2.4 — Write blog-mcp tests

14. Implement `tests/unit/seo-builder.test.ts`:
    - Frontmatter with title, author, date → OpenGraph metadata with all required fields
    - `og:type` is always `'article'` for blog posts (not `'website'`)
    - Twitter card is `'summary_large_image'` when image is present
    - SchemaOrg `@type` is `'BlogPosting'`
    - `renderToHtmlHead()` produces valid meta tags string

15. Implement `tests/unit/rss-generator.test.ts`:
    - 3 post metadata entries → valid Atom XML with 3 `<entry>` elements
    - Feed has correct `<updated>` timestamp
    - Each entry has `<id>`, `<title>`, `<link>`, `<updated>`
    - Missing required field (empty `feedTitle`) throws validation error

16. Implement `tests/unit/html-renderer.test.ts`:
    - Simple markdown → HTML contains `<article>` wrapper
    - Code block with language → `<code class="hljs language-python">` (or similar)
    - Math expression `$x^2$` → KaTeX-rendered `<span>` (not raw `$x^2$`)
    - External link → has `target="_blank" rel="noopener noreferrer"`

17. Implement `tests/integration/end-to-end.test.ts`:
    - `convert_to_html` tool: markdown with frontmatter → complete HTML page with `<html>`, `<head>`, `<body>`
    - HTML contains correct `<title>` from frontmatter
    - HTML contains Open Graph meta tags with correct values
    - HTML passes basic W3C structure check (has html, head, body)
    - HTML file size < 500KB
    - Skip if pandoc unavailable

### 2.5 — Scaffold servers/grant-mcp

18. Create `zen-sci/servers/grant-mcp/` directory structure:
    ```
    zen-sci/servers/grant-mcp/
    ├── src/
    │   ├── index.ts
    │   ├── manifest.ts
    │   ├── tools/
    │   │   ├── generate-proposal.ts
    │   │   ├── validate-compliance.ts
    │   │   └── check-format.ts
    │   └── formatting/
    │       ├── nih-formatter.ts
    │       ├── nsf-formatter.ts
    │       └── compliance-checker.ts
    ├── engine/
    │   ├── grant_engine.py
    │   ├── compliance_validator.py
    │   └── requirements.txt
    ├── templates/
    │   ├── nih/
    │   │   ├── specific-aims.tex
    │   │   └── research-strategy.tex
    │   └── nsf/
    │       ├── project-description.tex
    │       └── data-management-plan.tex
    ├── tests/
    │   ├── unit/
    │   │   ├── compliance-checker.test.ts
    │   │   └── formatters.test.ts
    │   └── integration/
    │       └── proposal-generation.test.ts
    ├── fixtures/
    │   ├── nih-r01-sample.md
    │   └── nsf-career-sample.md
    ├── package.json
    ├── tsconfig.json
    └── README.md
    ```

19. Create `zen-sci/servers/grant-mcp/package.json`:
    ```json
    {
      "name": "@zen-sci/grant-mcp",
      "version": "0.4.0",
      "description": "ZenSci Grant MCP: academic grant proposals in LaTeX and Word",
      "type": "module",
      "main": "./dist/index.js",
      "scripts": {
        "build": "tsc --build",
        "start": "node dist/index.js",
        "test": "vitest run",
        "typecheck": "tsc --noEmit"
      },
      "dependencies": {
        "@modelcontextprotocol/sdk": "^1.5.0",
        "@zen-sci/core": "workspace:*",
        "@zen-sci/sdk": "workspace:*"
      },
      "devDependencies": {
        "@types/node": "^20.0.0",
        "typescript": "^5.3.0",
        "vitest": "^1.2.0"
      }
    }
    ```

20. Create `engine/requirements.txt`: `pypandoc>=1.12`, `python-docx>=1.1.0`, `bibtexparser>=1.4.0`

### 2.6 — Implement grant-mcp formatting and compliance

21. Implement `src/manifest.ts` for grant-mcp:
    - `name: 'grant-mcp'`, `version: '0.4.0'`
    - `supportedFormats: ['grant-latex', 'docx']`
    - `capabilities: ['nih-formatting', 'nsf-formatting', 'erc-formatting', 'compliance-validation', 'submission-packaging']`
    - `systemDependencies: ['pandoc>=3.0', 'pdflatex (for grant-latex output)']`

22. Implement `src/formatting/compliance-checker.ts`:
    - Uses `GrantSubmissionFormat` and `SubmissionFile` types from `@zen-sci/core` (§3.4)
    - `checkCompliance(sections: GrantSection[], format: GrantSubmissionFormat): ComplianceResult`
    - `ComplianceResult`: `{ compliant: boolean; violations: ComplianceViolation[]; warnings: ComplianceWarning[]; score: number }`
    - Check: page limits per section (Specific Aims ≤ 1 page NIH, ≤ 2 pages NSF), required sections present, file size constraints
    - `ComplianceViolation`: `{ section: string; rule: string; severity: 'error' | 'warning'; details: string }`

23. Implement `src/formatting/nih-formatter.ts`:
    - `NIHFormatter` class
    - `format(sections: GrantSection[], options: NIHFormatOptions): string` — returns LaTeX source following NIH R01/R21/R03 formatting rules
    - `NIHFormatOptions`: `{ grantType: 'R01' | 'R21' | 'R03' | 'K99' | 'F31'; pageNumbering: 'section' | 'continuous'; fontFamily?: string }`
    - Apply NIH formatting rules: 11pt Arial or Georgia, 0.5" margins all sides, Times New Roman acceptable
    - Inject NIH header template from `templates/nih/` LaTeX files
    - Export `GrantSection` interface: `{ role: SubmissionFileRole; content: string; title?: string; pageLimit?: number }`

24. Implement `src/formatting/nsf-formatter.ts`:
    - `NSFFormatter` class following same pattern as NIHFormatter
    - NSF rules: 11pt or larger, 1" margins, no smaller than 10pt for references
    - Apply NSF-specific section structure from `templates/nsf/` templates
    - `NSFFormatOptions`: `{ programType: 'CAREER' | 'EAGER' | 'standard'; includeDataManagementPlan: boolean }`

### 2.7 — Implement grant-mcp tools and server

25. Implement `src/tools/generate-proposal.ts`:
    - Args: `{ sections: { role: SubmissionFileRole; content: string; pageLimit?: number }[], platform: GrantSubmissionFormat['platform'], funder: 'nih' | 'nsf' | 'erc', programType: string, bibliography?, options?: { outputFormat: 'grant-latex' | 'docx'; engine?: 'pdflatex' | 'xelatex' } }`
    - Route to NIHFormatter or NSFFormatter based on `funder`
    - Call Python engine for LaTeX → PDF or markdown → DOCX compilation
    - Return: `{ output_path?, output_base64, format: 'grant-latex' | 'docx', compliance: ComplianceResult, warnings, page_counts: Record<SubmissionFileRole, number> }`

26. Implement `src/tools/validate-compliance.ts`:
    - Args: `{ sections: GrantSection[], platform, funder, programType }`
    - Run ComplianceChecker without generating output
    - Return full `ComplianceResult`

27. Implement `src/tools/check-format.ts`:
    - Args: `{ platform: GrantSubmissionFormat['platform'] }`
    - Return the `GrantSubmissionFormat` definition for the given platform (required files, page limits, accepted formats)
    - Hard-coded format definitions per `grant-mcp-spec.md §3.6` type definitions

28. Implement `src/index.ts` — grant-mcp server entry point:
    - Register tools: `generate_proposal`, `validate_compliance`, `check_format`
    - Follow same pattern as latex-mcp index.ts

### 2.8 — Implement Python engines

29. Implement `engine/grant_engine.py` for grant-mcp:
    - Reads JSON from stdin: `{ request_id, sections, platform, funder, output_format, bibliography?, options }`
    - For `'grant-latex'` output: generate LaTeX using template injection + pandoc conversion; compile to PDF
    - For `'docx'` output: use python-docx to generate Word document with proper styles
    - Apply NIH/NSF page margins and fonts in template
    - Write JSON to stdout: `{ output_base64, page_counts, warnings }`

30. Implement `engine/compliance_validator.py`:
    - Reads JSON: `{ sections, format_config }`
    - Checks character/word/page counts per section
    - Returns JSON: `{ violations, warnings, score }`

### 2.9 — Write grant-mcp tests

31. Implement `tests/unit/compliance-checker.test.ts`:
    - NIH Specific Aims section over 1 page → compliance violation of severity 'error'
    - Missing required section (`specific-aims`) for NIH R01 → violation
    - NSF section within limits → no violations

32. Implement `tests/unit/formatters.test.ts`:
    - NIHFormatter.format() with valid sections → returns string containing `\documentclass`
    - NSFFormatter.format() with valid sections → returns string containing `\documentclass`
    - Format-specific page limit applied to output

33. Implement `tests/integration/proposal-generation.test.ts`:
    - Simple NIH-style sections → LaTeX output with correct structure
    - DOCX output: file is valid DOCX (check buffer starts with PK magic bytes)
    - Compliance check: over-limit section → violation reported
    - Skip if pandoc or pdflatex unavailable

---

## 3. File Manifest

### Create (new files):
```
zen-sci/servers/blog-mcp/package.json
zen-sci/servers/blog-mcp/tsconfig.json
zen-sci/servers/blog-mcp/README.md
zen-sci/servers/blog-mcp/src/index.ts
zen-sci/servers/blog-mcp/src/manifest.ts
zen-sci/servers/blog-mcp/src/tools/convert-to-html.ts
zen-sci/servers/blog-mcp/src/tools/generate-feed.ts
zen-sci/servers/blog-mcp/src/tools/validate-post.ts
zen-sci/servers/blog-mcp/src/rendering/html-renderer.ts
zen-sci/servers/blog-mcp/src/rendering/seo-builder.ts
zen-sci/servers/blog-mcp/src/rendering/syntax-highlighter.ts
zen-sci/servers/blog-mcp/src/rendering/katex-renderer.ts
zen-sci/servers/blog-mcp/src/rendering/rss-generator.ts
zen-sci/servers/blog-mcp/engine/html_engine.py
zen-sci/servers/blog-mcp/engine/requirements.txt
zen-sci/servers/blog-mcp/tests/unit/html-renderer.test.ts
zen-sci/servers/blog-mcp/tests/unit/seo-builder.test.ts
zen-sci/servers/blog-mcp/tests/unit/rss-generator.test.ts
zen-sci/servers/blog-mcp/tests/integration/end-to-end.test.ts
zen-sci/servers/blog-mcp/fixtures/sample-post.md
zen-sci/servers/blog-mcp/fixtures/sample-post.bib
zen-sci/servers/blog-mcp/fixtures/expected-meta.json
zen-sci/servers/grant-mcp/package.json
zen-sci/servers/grant-mcp/tsconfig.json
zen-sci/servers/grant-mcp/README.md
zen-sci/servers/grant-mcp/src/index.ts
zen-sci/servers/grant-mcp/src/manifest.ts
zen-sci/servers/grant-mcp/src/tools/generate-proposal.ts
zen-sci/servers/grant-mcp/src/tools/validate-compliance.ts
zen-sci/servers/grant-mcp/src/tools/check-format.ts
zen-sci/servers/grant-mcp/src/formatting/nih-formatter.ts
zen-sci/servers/grant-mcp/src/formatting/nsf-formatter.ts
zen-sci/servers/grant-mcp/src/formatting/compliance-checker.ts
zen-sci/servers/grant-mcp/engine/grant_engine.py
zen-sci/servers/grant-mcp/engine/compliance_validator.py
zen-sci/servers/grant-mcp/engine/requirements.txt
zen-sci/servers/grant-mcp/templates/nih/specific-aims.tex
zen-sci/servers/grant-mcp/templates/nih/research-strategy.tex
zen-sci/servers/grant-mcp/templates/nsf/project-description.tex
zen-sci/servers/grant-mcp/templates/nsf/data-management-plan.tex
zen-sci/servers/grant-mcp/tests/unit/compliance-checker.test.ts
zen-sci/servers/grant-mcp/tests/unit/formatters.test.ts
zen-sci/servers/grant-mcp/tests/integration/proposal-generation.test.ts
zen-sci/servers/grant-mcp/fixtures/nih-r01-sample.md
zen-sci/servers/grant-mcp/fixtures/nsf-career-sample.md
```

---

## 4. Success Criteria

**blog-mcp:**
- [ ] `pnpm --filter @zen-sci/blog-mcp run build` exits 0
- [ ] `pnpm --filter @zen-sci/blog-mcp run test` exits 0 (all unit tests pass)
- [ ] Integration test: simple markdown → valid HTML page with `<html>`, `<head>`, `<body>`
- [ ] HTML contains correct Open Graph meta tags (og:title, og:type=article, og:description)
- [ ] HTML contains JSON-LD `<script type="application/ld+json">` with `@type: 'BlogPosting'`
- [ ] Math expression `$E=mc^2$` rendered by KaTeX (not raw dollar signs in output)
- [ ] Code block with language fence → syntax-highlighted HTML
- [ ] `generate_feed` returns valid Atom XML with `<feed>` root element
- [ ] HTML file size < 500KB for sample-post.md
- [ ] SEOBuilder uses `WebMetadataSchema`, `OpenGraphMetadata`, `TwitterCardMetadata`, `SchemaOrgData` directly from `@zen-sci/core` — NO redefinition of these types
- [ ] `og:type` is `'article'` for blog posts (NEVER `'blog'` — that was removed from the OG spec)

**grant-mcp:**
- [ ] `pnpm --filter @zen-sci/grant-mcp run build` exits 0
- [ ] `pnpm --filter @zen-sci/grant-mcp run test` exits 0
- [ ] `generate_proposal` with NIH sections → returns LaTeX source containing `\documentclass`
- [ ] `generate_proposal` with NSF sections + `'docx'` output → returns valid DOCX buffer (PK magic bytes)
- [ ] `validate_compliance` with over-limit Specific Aims → returns `compliant: false` with violation
- [ ] `check_format` for `'grants-gov'` → returns correct `GrantSubmissionFormat` definition
- [ ] `GrantSubmissionFormat.platform` uses `'nsf-research-gov'` not `'nsf-fastlane'` (NSF FastLane retired 2023)
- [ ] `SubmissionFile.role` uses `SubmissionFileRole` strict union type (not `string`)

**Abstraction validation (the whole point of blog-mcp):**
- [ ] `blog-mcp` imports and uses `MarkdownParser` from `@zen-sci/core` — it does NOT re-implement markdown parsing
- [ ] `blog-mcp` imports and uses `CitationManager` from `@zen-sci/core` — it does NOT re-implement citation resolution
- [ ] `blog-mcp` imports and uses `SchemaValidator` from `@zen-sci/core` — it does NOT re-implement validation

---

## 5. Constraints & Non-Goals

**blog-mcp:**
- **DO NOT** implement a static site generator, CMS, or site scaffolding — single-file conversion only
- **DO NOT** add JavaScript to HTML output for core reading — JS may be used for TOC interaction only
- **DO NOT** implement comment systems, analytics, or search — scope is conversion only
- **DO NOT** redefine `WebMetadataSchema`, `OpenGraphMetadata`, `TwitterCardMetadata`, or `SchemaOrgData` — import from `@zen-sci/core`
- **DO NOT** use `'blog'` as OG type — it is not a valid og:type (use `'article'`)
- **DO NOT** add CSS frameworks (Bootstrap, Tailwind) — output must be self-contained with minimal CSS

**grant-mcp:**
- **DO NOT** implement submission to NIH/NSF portals — this is a formatter, not a submission client
- **DO NOT** implement budget calculation — format the budget justification section only
- **DO NOT** support agencies outside NIH/NSF/ERC in v0.4 — stub with `'manual'` platform for others
- **DO NOT** implement `'nsf-fastlane'` as a platform — FastLane is retired; use `'nsf-research-gov'`
- **DO NOT** use `SubmissionFile.role: string` — must use `SubmissionFileRole` union type

**Both modules:**
- **DO NOT** modify `@zen-sci/core` or `@zen-sci/sdk` during this phase — if core doesn't support what you need, note it as a deferred item
- **DO NOT** use `any` types
- **DO NOT** use `setRequestHandler` pattern — use `server.registerTool()` MCP SDK v2 API
- **DO NOT** modify files in `ZenithScience/specs/`, `ZenithScience/CONTEXT.md`, `ZenithScience/architecture/`
- All tool names use **snake_case**: `convert_to_html`, `generate_feed`, `validate_post`, `generate_proposal`, `validate_compliance`, `check_format`

---

## 6. Architecture Grounding

**Core abstraction validation metric:** After implementing blog-mcp, count the number of lines in blog-mcp's `src/tools/` and `src/rendering/` that deal with parsing, validation, or citation resolution versus the lines in `@zen-sci/core`. If blog-mcp has MORE parsing/validation code than it imports from core, the abstraction has failed and packages/core needs to be extended before Phase 3.

**`GrantSubmissionFormat` platform values** (from packages-core-spec.md §3.4 and grant-mcp-spec.md §3.6):
- `'grants-gov'` — HHS/NIH submissions
- `'nih-era-commons'` — NIH eRA Commons direct submission
- `'nsf-research-gov'` — NSF Research.gov (replaced FastLane in 2023)
- `'erc-sep'` — European Research Council Submission and Evaluation Platform
- `'manual'` — Manual/postal submission

**DOCX generation:** Use `python-docx` for Word output. Do NOT use pandoc for DOCX in grant-mcp — pandoc produces generic DOCX; `python-docx` allows precise style control required for grant formatting (fonts, margins, section headers per agency specs).

**HTML template approach:** blog-mcp does NOT call `packages/core`'s MarkdownParser to get a DocumentTree and then convert to HTML (that would duplicate pandoc). Instead: pass raw markdown to pandoc (via PandocWrapper) for HTML generation, then post-process the pandoc HTML output (KaTeX, highlight.js, SEO injection). `MarkdownParser` is used for validation and citation extraction BEFORE the pandoc call.

---

*Spec version: blog-mcp-v0.2-spec.md, grant-mcp-spec.md*
*Phase: 2 of 3*
*Prereq: Phase 1 complete (latex-mcp integration test passes)*
*Next: Phase 3 — slides-mcp v0.3 + newsletter-mcp v0.3 + paper-mcp*
