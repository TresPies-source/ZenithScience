# Implementation Commission: Phase 3 — slides-mcp v0.3 + newsletter-mcp v0.3 + paper-mcp

**Objective:** Complete the core ZenSci module suite by implementing `slides-mcp` (Beamer + Reveal.js presentations), `newsletter-mcp` (MJML-based HTML email), and `paper-mcp` (academic paper formats: IEEE, ACM, arXiv) — all building on the proven `packages/core` abstraction and the `@zen-sci/sdk` factory pattern.

**Commissioned by:** Cruz Morales / TresPiesDesign.com
**Date:** 2026-02-18
**License:** Apache 2.0
**Prereqs:** Phase 2 complete. The core abstraction has been validated: blog-mcp reused ≥ 70% of core parsing/validation code without modification. If this threshold was not met, do NOT start Phase 3 — extend packages/core first.
**System deps:** Node.js >= 20 LTS, Python >= 3.11, pandoc >= 3.0, pypandoc, TeX Live (full for slides-mcp Beamer), MJML npm package

---

## 1. Context & Grounding

**Primary Specifications:**
- `ZenithScience/specs/phase-2/slides-mcp-spec.md` — Beamer + Reveal.js dual output, slide structure, presenter notes, transitions. Read fully.
- `ZenithScience/specs/phase-2/newsletter-mcp-spec.md` — MJML compilation, responsive email, inline CSS, multi-block layouts. Read fully.
- `ZenithScience/specs/phase-1/paper-mcp-spec.md` — IEEE/ACM/arXiv academic paper formatting, two-column layouts, LaTeX template injection. Read fully.
- `ZenithScience/specs/infrastructure/packages-core-spec.md §3` — All shared types. `ConversionRule`, `FormatConstraint`, `ImageProcessingOptions` from §3.4 are particularly relevant.

**Pattern Files (follow these without deviating):**
- `zen-sci/servers/latex-mcp/src/index.ts` — server entry point pattern
- `zen-sci/servers/latex-mcp/src/tools/convert-to-pdf.ts` — tool handler pattern
- `zen-sci/servers/blog-mcp/src/rendering/seo-builder.ts` — utility class pattern
- `zen-sci/servers/blog-mcp/src/rendering/html-renderer.ts` — pandoc + post-processing pattern

**Phase 3 design principle:** By this phase, every module MUST be thinner than its Phase 1/2 counterparts because `packages/core` and `packages/sdk` are mature. If any Phase 3 module requires >200 lines of code for parsing, validation, or citation handling, those lines belong in core, not the module.

---

## 2. Detailed Requirements

### 2.1 — slides-mcp v0.3

**Architecture:** slides-mcp produces two output formats from a single markdown source:
1. **Beamer (LaTeX → PDF):** Academic/conference slides with TeX typography. Reuses latex-mcp's Python TeX compilation pipeline.
2. **Reveal.js (HTML):** Web-based slides with animations. Reuses blog-mcp's HTML rendering pipeline.

The module must be able to produce EITHER format from the SAME source markdown. This is the first multi-output module and validates the `OutputFormat` enum design.

**Slide markdown syntax:**
Slides are delimited by horizontal rules (`---`) in the markdown source. Each section becomes one slide. Speaker notes are marked with `> note: ...` blockquotes.

1. Scaffold `zen-sci/servers/slides-mcp/`:
   ```
   zen-sci/servers/slides-mcp/
   ├── src/
   │   ├── index.ts
   │   ├── manifest.ts
   │   ├── tools/
   │   │   ├── convert-to-slides.ts
   │   │   └── validate-deck.ts
   │   └── rendering/
   │       ├── slide-parser.ts         (markdown → slide structure)
   │       ├── beamer-renderer.ts      (slides → Beamer LaTeX)
   │       └── revealjs-renderer.ts    (slides → Reveal.js HTML)
   ├── engine/
   │   ├── slides_engine.py
   │   └── requirements.txt
   ├── tests/
   │   ├── unit/
   │   │   ├── slide-parser.test.ts
   │   │   └── renderers.test.ts
   │   └── integration/
   │       └── slide-generation.test.ts
   ├── fixtures/
   │   └── sample-deck.md
   ├── package.json
   ├── tsconfig.json
   └── README.md
   ```

2. Package.json: `@zen-sci/slides-mcp` v0.3.0; deps: `@zen-sci/core`, `@zen-sci/sdk`; devDeps: vitest, typescript.

3. Implement `src/rendering/slide-parser.ts` — `SlideParser` class:
   - `parse(source: string): SlideStructure` — splits markdown by `---` horizontally, extracts per-slide content, title, notes
   - `SlideStructure`: `{ title: string; slides: Slide[]; theme?: string; transition?: string }`
   - `Slide`: `{ index: number; title?: string; content: string; notes?: string; layout?: 'default' | 'title' | 'blank' | 'two-column' }`
   - Detect slide layout from first line: if first line is `# Title`, layout is 'title'; if `<!-- layout: two-column -->` comment present, layout is 'two-column'
   - Extract speaker notes: blockquotes starting with `> Note:` are notes, removed from slide content

4. Implement `src/rendering/beamer-renderer.ts` — `BeamerRenderer` class:
   - `render(deck: SlideStructure, options: BeamerOptions): string` — returns complete Beamer LaTeX source
   - `BeamerOptions`: `{ theme?: 'default' | 'metropolis' | 'madrid' | 'singapore'; colorTheme?: string; aspectRatio?: '169' | '43' | '1610'; showNotes?: boolean; fontTheme?: 'default' | 'professionalfonts' }`
   - Each slide becomes a `\begin{frame}...\end{frame}` block
   - Use `\frametitle{}` for slide titles
   - Speaker notes → `\note{}` blocks
   - Two-column slides → `\begin{columns}` environment
   - Inject frontmatter `title`, `author`, `date` into `\title{}`, `\author{}`, `\date{}`
   - Math content passes through directly (Beamer handles TeX math natively)

5. Implement `src/rendering/revealjs-renderer.ts` — `RevealJsRenderer` class:
   - `render(deck: SlideStructure, options: RevealJsOptions): string` — returns standalone HTML with Reveal.js embedded
   - `RevealJsOptions`: `{ theme?: 'black' | 'white' | 'league' | 'beige' | 'sky' | 'moon' | 'solarized'; transition?: 'slide' | 'fade' | 'convex' | 'concave' | 'zoom'; controls?: boolean; progress?: boolean; katexEnabled?: boolean }`
   - Each slide becomes a `<section>` in the Reveal.js HTML5 structure
   - Speaker notes → `<aside class="notes">` elements
   - Load Reveal.js from CDN (jsDelivr): `https://cdn.jsdelivr.net/npm/reveal.js@4/`
   - If `katexEnabled: true`, load KaTeX from CDN and auto-render math
   - Two-column slides → `<div class="columns"><div class="column">` layout
   - Output must be self-contained (CSS, JS loaded from CDN; no external file deps for basic operation)

6. Implement `src/tools/convert-to-slides.ts`:
   - Args: `{ source, title, author?, format: 'beamer' | 'revealjs', options?: BeamerOptions | RevealJsOptions }`
   - Parse with SlideParser, then route to BeamerRenderer or RevealJsRenderer
   - For `'beamer'`: also run TeX compilation via PythonEngine (reuse latex-mcp Python engine pattern)
   - Return: `{ output_base64, format, slide_count, has_notes: boolean, warnings }`

7. Implement `src/tools/validate-deck.ts`:
   - Args: `{ source }`
   - Parse with SlideParser; validate: slide count > 0, no empty slides, math syntax valid, images referenced exist
   - Return: `{ valid, slide_count, warnings, errors }`

8. Implement `src/index.ts` — server entry point, register: `convert_to_slides`, `validate_deck`

9. Implement Python `engine/slides_engine.py`:
   - For Beamer: run `pdflatex` on generated Beamer LaTeX source → return PDF base64
   - For Reveal.js: just return the HTML string (no compilation needed)

10. Write tests:
    - Unit: `slide-parser.test.ts` — 3-slide deck splits correctly; notes extracted; two-column detected
    - Unit: `renderers.test.ts` — BeamerRenderer returns LaTeX with `\begin{frame}` for each slide; RevealJsRenderer returns HTML with `<section>` per slide
    - Integration: markdown deck → Beamer PDF (skip if pdflatex unavailable); markdown deck → Reveal.js HTML

### 2.2 — newsletter-mcp v0.3

**Architecture:** newsletter-mcp uses MJML (an email HTML framework) to generate responsive, cross-client email HTML. MJML is the industry standard for emails that render correctly in Gmail, Outlook, Apple Mail, and mobile clients. Key insight: email HTML is NOT standard HTML — inline CSS, table layouts, VML for Outlook. MJML handles this complexity.

**Approach:** Convert markdown → MJML (structured email XML) → HTML (via MJML compiler). The pandoc markdown → HTML pipeline does NOT apply here because email HTML has fundamentally different constraints.

11. Scaffold `zen-sci/servers/newsletter-mcp/`:
    ```
    zen-sci/servers/newsletter-mcp/
    ├── src/
    │   ├── index.ts
    │   ├── manifest.ts
    │   ├── tools/
    │   │   ├── convert-to-email.ts
    │   │   └── validate-email.ts
    │   └── rendering/
    │       ├── mjml-builder.ts    (markdown → MJML XML structure)
    │       ├── block-renderer.ts  (render individual content blocks)
    │       └── mjml-compiler.ts   (MJML XML → HTML via mjml npm package)
    ├── tests/
    │   ├── unit/
    │   │   ├── mjml-builder.test.ts
    │   │   └── block-renderer.test.ts
    │   └── integration/
    │       └── email-generation.test.ts
    ├── fixtures/
    │   └── sample-newsletter.md
    ├── package.json
    ├── tsconfig.json
    └── README.md
    ```

12. Package.json: `@zen-sci/newsletter-mcp` v0.3.0; add `"mjml": "^4.14.0"` to dependencies.

13. Implement `src/rendering/block-renderer.ts` — `BlockRenderer` class:
    - `renderHeading(node: HeadingNode): string` — returns `<mj-text>` with heading styling
    - `renderParagraph(node: ParagraphNode): string` — returns `<mj-text>` with paragraph styling
    - `renderImage(node: ImageNode): string` — returns `<mj-image>` with src + alt
    - `renderButton(url: string, text: string): string` — returns `<mj-button>` CTA block
    - `renderDivider(): string` — returns `<mj-divider>`
    - `renderCode(node: CodeBlockNode): string` — returns `<mj-text>` with `font-family: monospace`
    - All methods accept nodes from `@zen-sci/core` DocumentTree types

14. Implement `src/rendering/mjml-builder.ts` — `MJMLBuilder` class:
    - `build(tree: DocumentTree, frontmatter: FrontmatterMetadata, options: EmailOptions): string` — returns complete MJML XML string
    - `EmailOptions`: `{ subject: string; previewText?: string; brandColor?: string; fontFamily?: string; logoUrl?: string; footerText?: string; unsubscribeUrl?: string }`
    - MJML structure: `<mjml><mj-head>...</mj-head><mj-body>...</mj-body></mjml>`
    - Head: `<mj-preview>` (preview text), `<mj-attributes>` (global styles), `<mj-font>`
    - Body: `<mj-section>` per document section, `<mj-column>` wrapping `<mj-text>`, `<mj-image>`, etc.
    - Header section: logo + title + preview text
    - Footer section: unsubscribe link, brand name, postal address (GDPR/CAN-SPAM compliance)

15. Implement `src/rendering/mjml-compiler.ts` — `MJMLCompiler` class:
    - `compile(mjmlSource: string): { html: string; errors: MJMLError[] }` — uses `mjml` npm package to compile
    - `MJMLError`: `{ line?: number; message: string; tagName?: string }`
    - If compilation has errors, still return HTML with errors array populated (MJML soft-fails)
    - `validate(mjmlSource: string): boolean` — returns true if no MJML errors

16. Implement `src/tools/convert-to-email.ts`:
    - Args: `{ source, subject, previewText?, brandColor?, logoUrl?, footerText?, unsubscribeUrl? }`
    - Run: SchemaValidator → MarkdownParser (get DocumentTree) → MJMLBuilder.build() → MJMLCompiler.compile()
    - Return: `{ html, subject, preview_text, char_count, estimated_size_bytes, mjml_errors, warnings }`
    - Warn if `estimated_size_bytes > 102400` (100KB — Gmail clips emails over 102KB)
    - Warn if `html` contains elements known to break in Outlook (e.g., CSS Grid, Flexbox)

17. Implement `src/tools/validate-email.ts`:
    - Args: `{ source, subject? }`
    - Run validation without compiling to final HTML
    - Return: `{ valid, errors, warnings, accessibility_issues, estimated_size_bytes }`

18. Implement `src/index.ts` — server entry point, register: `convert_to_email`, `validate_email`

19. Write tests:
    - Unit: `mjml-builder.test.ts` — markdown with one heading, one paragraph, one image → MJML with `<mj-section>`, `<mj-text>`, `<mj-image>`; footer section always present
    - Unit: `block-renderer.test.ts` — HeadingNode → `<mj-text>` with correct font size; ImageNode → `<mj-image src="...">`;
    - Integration: simple markdown newsletter → complete HTML with DOCTYPE, `<html>`, `<head>`, `<body>`; estimate size < 100KB; no MJML compilation errors for valid input

### 2.3 — paper-mcp

**Architecture:** paper-mcp is a specialization of latex-mcp for academic publication formats (IEEE, ACM, arXiv). It reuses the entire latex-mcp TeX compilation pipeline via `@zen-sci/sdk`'s `runConversionPipeline()`, but injects format-specific LaTeX templates (two-column layout, journal class files, style packages).

Key principle: **paper-mcp should contain almost no Python code**. The heavy lifting is in latex-mcp's engine. paper-mcp only adds format-specific LaTeX template injection.

20. Scaffold `zen-sci/servers/paper-mcp/`:
    ```
    zen-sci/servers/paper-mcp/
    ├── src/
    │   ├── index.ts
    │   ├── manifest.ts
    │   ├── tools/
    │   │   ├── convert-to-paper.ts
    │   │   └── validate-submission.ts
    │   └── templates/
    │       ├── template-registry.ts
    │       └── ieee-injector.ts
    ├── templates/
    │   ├── ieee/
    │   │   ├── preamble.tex
    │   │   └── IEEEtran.cls     (standard IEEE class — include a stub; real class is system-installed)
    │   ├── acm/
    │   │   └── preamble.tex
    │   └── arxiv/
    │       └── preamble.tex
    ├── tests/
    │   ├── unit/
    │   │   └── template-registry.test.ts
    │   └── integration/
    │       └── paper-generation.test.ts
    ├── fixtures/
    │   └── sample-paper.md
    ├── package.json
    ├── tsconfig.json
    └── README.md
    ```

21. Package.json: `@zen-sci/paper-mcp` v0.1.0; deps: `@zen-sci/core`, `@zen-sci/sdk`.

22. Implement `src/templates/template-registry.ts` — `TemplateRegistry` class:
    - `getTemplate(format: 'paper-ieee' | 'paper-acm' | 'paper-arxiv'): PaperTemplate`
    - `PaperTemplate`: `{ documentClass: string; classoptions: string[]; preambleLines: string[]; pandocArgs: string[]; columnsMode: 'one' | 'two' }`
    - IEEE template: `\documentclass[conference]{IEEEtran}`, two-column, `\usepackage{cite}`, `\usepackage{amsmath}`
    - ACM template: `\documentclass[sigconf]{acmart}`, two-column, `\usepackage{booktabs}`
    - arXiv template: `\documentclass[a4paper]{article}`, one-column, `\usepackage{arxiv}` (standard arXiv style)
    - Read actual preamble content from `templates/{format}/preamble.tex` files

23. Implement `src/tools/convert-to-paper.ts`:
    - Args: `{ source, title, authors: { name: string; affiliation?: string; email?: string }[], bibliography?, abstract?, format: 'paper-ieee' | 'paper-acm' | 'paper-arxiv', keywords?: string[] }`
    - Get template from TemplateRegistry
    - Build enhanced `DocumentRequest` with format + `moduleOptions: { template: PaperTemplate }`
    - Call Python engine (same `latex_engine.py` from latex-mcp via PythonEngine) with extra args for template injection
    - Return: `{ pdf_base64, latex_source, page_count, column_count, abstract_word_count, warnings }`

24. Implement `src/tools/validate-submission.ts`:
    - Args: `{ source, format, abstract?, keywords? }`
    - Check: abstract present (IEEE/ACM require it), keywords present (IEEE requires ≤6), page limits for format
    - Run SchemaValidator + MathValidator + CitationManager from `@zen-sci/core`
    - Return: `{ valid, submission_ready, errors, warnings }`

25. Implement `src/index.ts` — server entry point, register: `convert_to_paper`, `validate_submission`

26. Write LaTeX preamble stubs in `templates/`:
    - `templates/ieee/preamble.tex`: standard IEEEtran preamble (hyperref, cite, amsmath, graphicx)
    - `templates/acm/preamble.tex`: acmart preamble (booktabs, microtype, hyperref)
    - `templates/arxiv/preamble.tex`: arxiv.sty compatible preamble (amsmath, geometry, hyperref)

27. Write tests:
    - Unit: `template-registry.test.ts` — `getTemplate('paper-ieee')` returns `columnsMode: 'two'`; `getTemplate('paper-arxiv')` returns `columnsMode: 'one'`; all templates have non-empty `preambleLines`
    - Integration: markdown with abstract and keywords → paper PDF (skip if pdflatex unavailable); validate IEEE paper with 7 keywords → validation warning (IEEE max is 6)

### 2.4 — Finalize monorepo and root-level integration

28. Update `zen-sci/package.json` root build/test scripts to include all 5 servers and 2 packages.

29. Verify workspace-level `pnpm run build` compiles all 7 packages in the correct dependency order:
    - packages/core (no deps)
    - packages/sdk (depends on core)
    - servers/latex-mcp (depends on core + sdk)
    - servers/blog-mcp (depends on core + sdk)
    - servers/slides-mcp (depends on core + sdk)
    - servers/newsletter-mcp (depends on core + sdk)
    - servers/grant-mcp (depends on core + sdk)
    - servers/paper-mcp (depends on core + sdk)

30. Create `zen-sci/README.md` (update existing) with: project description, module list, local development setup (pnpm install, build, test), system dependency requirements (Node, Python, pandoc, TeX Live, MJML).

31. Update `ZenithScience/STATUS.md`: mark slides-mcp ✅, newsletter-mcp ✅, paper-mcp ✅. Update "Next Steps" section to reflect Phase 3 complete. Update "Module Roadmap" table.

32. Update `ZenithScience/SCHEMAS.md`: move `DocumentRequest`, `DocumentResponse`, `ConversionPipeline`, and key types from 🟡 to 🟢 (implemented and tested).

---

## 3. File Manifest

### Create (new files):
```
zen-sci/servers/slides-mcp/package.json
zen-sci/servers/slides-mcp/tsconfig.json
zen-sci/servers/slides-mcp/README.md
zen-sci/servers/slides-mcp/src/index.ts
zen-sci/servers/slides-mcp/src/manifest.ts
zen-sci/servers/slides-mcp/src/tools/convert-to-slides.ts
zen-sci/servers/slides-mcp/src/tools/validate-deck.ts
zen-sci/servers/slides-mcp/src/rendering/slide-parser.ts
zen-sci/servers/slides-mcp/src/rendering/beamer-renderer.ts
zen-sci/servers/slides-mcp/src/rendering/revealjs-renderer.ts
zen-sci/servers/slides-mcp/engine/slides_engine.py
zen-sci/servers/slides-mcp/engine/requirements.txt
zen-sci/servers/slides-mcp/tests/unit/slide-parser.test.ts
zen-sci/servers/slides-mcp/tests/unit/renderers.test.ts
zen-sci/servers/slides-mcp/tests/integration/slide-generation.test.ts
zen-sci/servers/slides-mcp/fixtures/sample-deck.md
zen-sci/servers/newsletter-mcp/package.json
zen-sci/servers/newsletter-mcp/tsconfig.json
zen-sci/servers/newsletter-mcp/README.md
zen-sci/servers/newsletter-mcp/src/index.ts
zen-sci/servers/newsletter-mcp/src/manifest.ts
zen-sci/servers/newsletter-mcp/src/tools/convert-to-email.ts
zen-sci/servers/newsletter-mcp/src/tools/validate-email.ts
zen-sci/servers/newsletter-mcp/src/rendering/mjml-builder.ts
zen-sci/servers/newsletter-mcp/src/rendering/block-renderer.ts
zen-sci/servers/newsletter-mcp/src/rendering/mjml-compiler.ts
zen-sci/servers/newsletter-mcp/tests/unit/mjml-builder.test.ts
zen-sci/servers/newsletter-mcp/tests/unit/block-renderer.test.ts
zen-sci/servers/newsletter-mcp/tests/integration/email-generation.test.ts
zen-sci/servers/newsletter-mcp/fixtures/sample-newsletter.md
zen-sci/servers/paper-mcp/package.json
zen-sci/servers/paper-mcp/tsconfig.json
zen-sci/servers/paper-mcp/README.md
zen-sci/servers/paper-mcp/src/index.ts
zen-sci/servers/paper-mcp/src/manifest.ts
zen-sci/servers/paper-mcp/src/tools/convert-to-paper.ts
zen-sci/servers/paper-mcp/src/tools/validate-submission.ts
zen-sci/servers/paper-mcp/src/templates/template-registry.ts
zen-sci/servers/paper-mcp/src/templates/ieee-injector.ts
zen-sci/servers/paper-mcp/templates/ieee/preamble.tex
zen-sci/servers/paper-mcp/templates/acm/preamble.tex
zen-sci/servers/paper-mcp/templates/arxiv/preamble.tex
zen-sci/servers/paper-mcp/tests/unit/template-registry.test.ts
zen-sci/servers/paper-mcp/tests/integration/paper-generation.test.ts
zen-sci/servers/paper-mcp/fixtures/sample-paper.md
```

### Modify (existing files):
```
zen-sci/README.md  (update with complete module list + dev setup)
ZenithScience/STATUS.md  (mark Phase 3 complete, update roadmap table)
ZenithScience/schemas/SCHEMAS.md  (promote implemented types to 🟢)
```

---

## 4. Success Criteria

**slides-mcp:**
- [ ] `pnpm --filter @zen-sci/slides-mcp run build` exits 0
- [ ] `pnpm --filter @zen-sci/slides-mcp run test` exits 0
- [ ] SlideParser splits 3-slide markdown deck into 3 Slide objects
- [ ] BeamerRenderer produces LaTeX with one `\begin{frame}...\end{frame}` per slide
- [ ] RevealJsRenderer produces HTML with one `<section>` per slide
- [ ] Reveal.js output loads from CDN (no bundled Reveal.js files)
- [ ] Both output formats support speaker notes from `> Note:` blockquotes
- [ ] Integration: markdown deck → Beamer PDF (skip if pdflatex unavailable)
- [ ] Integration: markdown deck → Reveal.js standalone HTML with Reveal.js CDN loaded

**newsletter-mcp:**
- [ ] `pnpm --filter @zen-sci/newsletter-mcp run build` exits 0
- [ ] `pnpm --filter @zen-sci/newsletter-mcp run test` exits 0
- [ ] MJMLBuilder produces valid MJML with `<mjml>` root and `<mj-body>` child
- [ ] MJMLCompiler.compile() returns HTML containing `<!DOCTYPE html>` and `<html>`
- [ ] Footer section always present in compiled output (CAN-SPAM compliance)
- [ ] Output estimated size < 100KB for sample newsletter
- [ ] Warning raised if estimated size > 100KB
- [ ] MJML compilation has zero errors for valid sample-newsletter.md input

**paper-mcp:**
- [ ] `pnpm --filter @zen-sci/paper-mcp run build` exits 0
- [ ] `pnpm --filter @zen-sci/paper-mcp run test` exits 0
- [ ] TemplateRegistry returns correct `columnsMode` for each format (IEEE/ACM = 'two', arXiv = 'one')
- [ ] `validate_submission` for IEEE paper with 7 keywords → warning (max is 6)
- [ ] `validate_submission` for paper missing abstract → error
- [ ] Integration: markdown paper with abstract → LaTeX source containing template preamble (skip PDF if pdflatex unavailable)

**Monorepo:**
- [ ] `pnpm run build` from `zen-sci/` root builds ALL 8 packages successfully (packages/core, packages/sdk, 6 servers)
- [ ] `pnpm run test` from `zen-sci/` root runs all test suites; only integration tests that require system deps (pandoc, pdflatex) may skip
- [ ] SCHEMAS.md has at least `DocumentRequest`, `DocumentResponse`, `ConversionPipeline` promoted to 🟢
- [ ] STATUS.md reflects Phase 3 complete

---

## 5. Constraints & Non-Goals

**slides-mcp:**
- **DO NOT** bundle Reveal.js — load from CDN only
- **DO NOT** implement animation editing or per-slide timing — this is a converter, not a slide editor
- **DO NOT** implement Keynote, PowerPoint, or Google Slides output — out of scope for v0.3
- **DO NOT** implement live reload or watch mode

**newsletter-mcp:**
- **DO NOT** implement email sending — this is a converter, not an email client
- **DO NOT** use CSS Grid or Flexbox in rendered HTML — email clients don't support them; MJML handles layout
- **DO NOT** implement subscriber list management or unsubscribe mechanics — provide the unsubscribe URL link only
- **DO NOT** embed binary images in HTML — use `src="..."` URLs

**paper-mcp:**
- **DO NOT** reimplement TeX compilation — call the same Python engine that latex-mcp uses via PythonEngine
- **DO NOT** bundle LaTeX class files (IEEEtran.cls, acmart.cls) — they must be on the system TeX installation
- **DO NOT** implement peer-review submission workflows — this is a formatter only
- **DO NOT** support journal-specific formatting beyond IEEE/ACM/arXiv in v0.1

**All modules:**
- **DO NOT** use `any` types
- **DO NOT** use `setRequestHandler` — use `server.registerTool()` MCP SDK v2 API
- **DO NOT** use `| string` escape hatches on union types
- **DO NOT** modify `packages/core` or `packages/sdk` unless a genuine gap in the abstraction is found; if so, document the gap in a comment and note it as a deferred item in STATUS.md
- All tool names use **snake_case**: `convert_to_slides`, `validate_deck`, `convert_to_email`, `validate_email`, `convert_to_paper`, `validate_submission`

---

## 6. Architecture Grounding

**slides-mcp dual-output pattern:** The same `SlideParser` is called for both Beamer and Reveal.js paths. The router in `convert-to-slides.ts` decides which renderer to call based on `format` argument. This is the first implementation of the "same source → different output" thesis. Keep it clean.

**newsletter-mcp is NOT blog-mcp for email:** Email rendering has fundamentally different constraints than HTML rendering. pandoc is NOT used for email — MJML is the source of truth. The DocumentTree from `@zen-sci/core` is used only to drive `BlockRenderer`; the final output format is MJML XML, which is then compiled to email HTML by the MJML compiler. Do not attempt to use pandoc's HTML output as a base for email.

**paper-mcp Python engine reuse:** paper-mcp's `convert_to_paper.ts` should call the same `latex_engine.py` from `servers/latex-mcp/engine/`. It does NOT maintain its own Python engine. To do this, either:
- Option A: Reference `../latex-mcp/engine/latex_engine.py` via a relative path in PythonEngine.run()
- Option B: Extract the engine to `packages/sdk/python/` as a shared Python module
Use Option A for simplicity in v0.1; note Option B as a future refactor.

**SCHEMAS.md promotion criteria:** A type qualifies for 🟢 when:
1. It has a TypeScript implementation in `packages/core/src/types.ts`
2. It has at least one test that exercises the type
3. It is used by at least one module (validated in production-path code)

**OutputFormat completeness check:** When all 3 phases are complete, verify the `OutputFormat` union in `packages-core-spec.md §3.1` covers every `supportedFormats` value declared in all 8 module manifests. If any manifest declares a format not in the union, that is a bug — either add it to the union or remove it from the manifest.

---

*Spec version: slides-mcp-spec.md, newsletter-mcp-spec.md, paper-mcp-spec.md*
*Phase: 3 of 3*
*Prereq: Phase 2 complete (blog-mcp core abstraction validated)*
*After completion: ZenSci v0.x suite is feature-complete. Next: v1.0 planning, lab-notebook-mcp, performance optimization.*
