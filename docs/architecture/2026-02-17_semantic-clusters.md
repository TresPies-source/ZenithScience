# ZenSci Semantic Cluster Map

**Date:** 2026-02-17
**System:** Knowledge-Work Forge — MCP Suite for Publication-Ready Artifacts
**Scope:** LaTeX, Blog, Newsletter, Grant, Slides modules (v0.1–v0.4)
**Methodology:** Behavioral clustering — grouped by action verbs, not file structure

---

## Executive Summary

ZenithScience (ZenSci) is an **input-transformation-output pipeline suite** that converts raw structured thinking (markdown, outlines, brainstorming notes) into publication-ready artifacts across five formats. The system enables AI to speak natively in each target format while helping humans formalize and refine reasoning *before* rendering.

**Core behavioral promise:** Structured thought in → publication-ready artifact out.

The system is organized into **9 semantic clusters**, with a shared `core` library handling cross-cutting concerns. The strongest seams are between LaTeX, Slides, and Grant modules (all use the TeX ecosystem); Blog and Newsletter are lighter-weight HTML-based variants.

---

## Cluster Map

### Cluster 1: Parsing & Understanding

**What it does:** Extract structured meaning from unstructured or semi-structured input. Recognize document intent, extract components, validate input format.

**Capabilities:**
- PARSE markdown files into document tree (headings, sections, lists, code blocks)
- PARSE mathematical expressions (Unicode, LaTeX, SymPy syntax) and normalize to canonical form
- PARSE metadata/frontmatter (YAML, JSON) from document headers
- PARSE bibliography/references (BibTeX, CSL-JSON, plain text citations)
- PARSE outline structures (numbered/bulleted hierarchies) into document skeleton
- PARSE table data (markdown, CSV, structured cells) into semantic grid
- PARSE code blocks with language annotation and syntax preservation
- PARSE URLs and resolve references (internal, external, DOI)
- VALIDATE input document completeness (required sections present, no orphaned references)

**Modules that own this:**
- `core` (all parsing primitives — MD tree, math, bibliography)
- `latex-mcp` (math expression validation, LaTeX-specific syntax)

**Lives in:** `core` (shared) + module-specific extensions

**Why it matters:** Garbage in = garbage out. Parsing errors cascade. Strong parsing creates confidence in downstream transformations.

**Schemas needed:**
- `DocumentTree` — hierarchical structure of parsed document
- `MathExpression` — canonical form + alternative representations
- `Frontmatter` — metadata dict with type safety
- `ParseError` — structured errors with recovery suggestions

---

### Cluster 2: Thinking Partnership & Reasoning Formalization

**What it does:** Capture AI-assisted brainstorming, reasoning chains, and refinement decisions. Help humans structure fuzzy thinking into rigorous output.

**Capabilities:**
- CAPTURE brainstorming session context (user intent, raw notes, research direction)
- FORMALIZE mathematical reasoning into proof structure (assumptions → steps → conclusion)
- STRUCTURE narrative arguments into logical flow
- IDENTIFY gaps in reasoning and suggest refinements
- SCAFFOLD theorem/proof/lemma environments with reasoning chain
- TRACE decision points where AI helped refine thinking
- EMBED reasoning metadata in output (optional: show "thinking" in rendered document)
- VALIDATE internal consistency of argument or proof

**Modules that own this:**
- `core` (reasoning schema + validation)
- `latex-mcp` (theorem/proof structuring; mathematical reasoning)

**Lives in:** `core` (shared schema) — primarily used by LaTeX and Grant modules

**Why it matters:** This is the **differentiator**. Conversion tools are commodities; thinking partners are defensible. The system doesn't just output LaTeX — it helped you think better first.

**Schemas needed:**
- `ThinkingSession` — context, questions, reasoning steps, refinements
- `ReasoningTrace` — decision points, alternatives considered, choice rationale
- `ProofStructure` — theorem statement, assumptions, proof steps, QED marker

---

### Cluster 3: Document Structuring & Layout

**What it does:** Organize content into the semantic and visual structure of the target format. Apply templates, manage hierarchy, enforce format-specific rules.

**Capabilities:**
- STRUCTURE document into sections, subsections, chapters (module-appropriate)
- APPLY templates (academic paper, grant proposal, blog post, slide deck) with required/optional fields
- HANDLE page breaks, column layouts, two-column vs. single-column rendering
- MANAGE slide layouts (title, content, 2-col, code, image) in Reveal.js or Beamer
- ENFORCE academic document rules (abstract placement, bibliography at end, footnote format)
- STRUCTURE grant sections in funder-specific order (NSF, NIH, ERC templates differ)
- STRUCTURE email newsletters into sections, CTAs, social links (MJML constraints)
- HANDLE figure/table placement and caption formatting
- MANAGE outline-to-document translation (convert brainstorm outline into full-featured document)
- GENERATE table of contents, indices, cross-references

**Modules that own this:**
- `latex-mcp` (academic document structure, theorem environments, figure/table management)
- `grant-mcp` (funder-specific section ordering, structured templates)
- `slides-mcp` (slide layouts, speaker notes, transitions)
- `blog-mcp` (blog post sections, metadata frontmatter, permalink structure)
- `newsletter-mcp` (email sections, MJML constraints, responsive layout)

**Lives in:** Shared `core` (template engine, section schema) + module-specific implementations

**Why it matters:** Structure is what makes output look intentional, not random. Users trust formatted output more than plain text.

**Schemas needed:**
- `TemplateDefinition` — required/optional fields, examples, constraints
- `DocumentSection` — type, heading, content, metadata
- `LayoutRule` — module-specific rules (e.g., "abstract must be <250 words")
- `TOCEntry` — hierarchical table-of-contents entry with page/slide numbers

---

### Cluster 4: Format Conversion & Translation

**What it does:** Translate from the canonical input format (markdown + metadata) to each target format (LaTeX, HTML, MJML, etc.). Handle format-specific idioms and constraints.

**Capabilities:**
- CONVERT markdown to LaTeX (via pandoc + custom filters)
- CONVERT markdown to HTML (blog post structure)
- CONVERT markdown to MJML (email-safe, responsive HTML)
- CONVERT markdown to Beamer (LaTeX slides)
- CONVERT markdown to Reveal.js JSON/HTML (web slides)
- CONVERT markdown tables to format-specific table syntax (LaTeX tabular, HTML `<table>`, MJML rows)
- CONVERT markdown code blocks to format-specific code rendering (LaTeX minted, HTML `<pre>`, email-safe escaped blocks)
- CONVERT markdown emphasis to format-specific styling (LaTeX `\textit{}` vs. HTML `<em>` vs. email plain text)
- CONVERT citations (BibTeX keys → formatted references in each output format)
- CONVERT mathematical expressions to native format (LaTeX for PDF, MathJax for HTML, Unicode for email)
- HANDLE image references, alt text, captions per format
- TRANSLATE headings hierarchy to format-appropriate levels

**Modules that own this:**
- `latex-mcp` (MD → LaTeX, primary converter)
- `blog-mcp` (MD → HTML)
- `newsletter-mcp` (MD → MJML)
- `slides-mcp` (MD → Beamer or Reveal.js)
- `grant-mcp` (MD → LaTeX with grant sections)

**Lives in:** `core` (pandoc wrapper, format mappings) + module-specific conversion rules

**Why it matters:** Conversion is not just string replacement. Each format has idioms, constraints, and affordances. Bad conversion makes output look amateur.

**Schemas needed:**
- `ConversionRule` — mapping from markdown construct to target format
- `ConversionPipeline` — sequence of transformations (parse → validate → convert → render)
- `FormatConstraint` — format-specific limits (email width, slide size, PDF margins)

---

### Cluster 5: Mathematical Reasoning & Symbolic Manipulation

**What it does:** Parse, validate, and render mathematical content. Help users express mathematics precisely in LaTeX and other formats.

**Capabilities:**
- PARSE mathematical expressions (Unicode, LaTeX, SymPy/Python syntax)
- VALIDATE mathematical notation (balanced parentheses, matching delimiters, correct operator precedence)
- CONVERT between math representations (e.g., SymPy → LaTeX)
- RENDER equations in LaTeX (display, inline, numbered, aligned)
- STRUCTURE proofs (theorem → proof environment with step numbering)
- GENERATE symbolic derivatives, integrals, simplifications (SymPy)
- HANDLE special mathematical environments (align*, gather*, cases, matrix)
- MANAGE definition order (lemmas before theorems, hypotheses before conclusion)
- VALIDATE mathematical consistency (variable definitions, dimension matching)
- SUGGEST improvements to mathematical presentation (spacing, operator choice, formatting)

**Modules that own this:**
- `latex-mcp` (primary mathematical expression handler)

**Lives in:** `core` (math parsing, validation) + `latex-mcp` (rendering)

**Why it matters:** Math is the hardest part of LaTeX conversion. Precision here differentiates ZenSci from dumb converters.

**Schemas needed:**
- `MathExpression` — canonical form, LaTeX output, alternative representations
- `ProofStep` — formula, justification, symbolic state
- `MathEnvironment` — type (align, equation, etc.), content, constraints

---

### Cluster 6: Citation & Bibliography Management

**What it does:** Normalize, validate, and render bibliographic data across formats. Ensure citations are consistent and complete.

**Capabilities:**
- PARSE bibliography sources (BibTeX, CSL-JSON, RIS, plain text)
- VALIDATE citations (check for missing fields, malformed keys)
- NORMALIZE citation format (standardize author names, publication titles, URLs)
- RENDER bibliographies in format-specific styles (LaTeX `\cite`, HTML footnotes, email plain text)
- MANAGE citation keys (generate, resolve, check for conflicts)
- EXTRACT metadata from citations (DOI, arXiv ID, ORCID)
- GENERATE cross-references (Figure 1, Theorem 2.3, Ref [5])
- HANDLE multiple bibliography styles (IEEE, APA, Chicago, custom)
- VALIDATE URLs and DOI links (optional: check link viability)
- RESOLVE missing citations and suggest sources

**Modules that own this:**
- `core` (shared bibliography schema, parsing, validation)
- `latex-mcp` (LaTeX bibliography rendering via BibTeX)
- `grant-mcp` (academic citations required for grants)
- `slides-mcp` (optional bibliography for research presentations)

**Lives in:** `core` (shared) — used by LaTeX, Grant, Slides

**Why it matters:** Academic credibility depends on citations. Wrong bibliography = rejected paper/grant. This is non-negotiable.

**Schemas needed:**
- `CitationRecord` — author, title, year, DOI, arXiv, URL, etc.
- `BibliographyStyle` — citation format rules
- `CrossReference` — reference to figure/theorem/footnote with context

---

### Cluster 7: Rendering & Output Generation

**What it does:** Produce the final output artifact. Compile, validate, and optimize for distribution.

**Capabilities:**
- RENDER LaTeX to PDF (via pdflatex, xelatex, or lualatex)
- RENDER HTML (blog post, newsletter, web content)
- RENDER MJML to email-safe HTML (responsive, preview-friendly)
- RENDER Beamer to PDF (slides, handouts)
- RENDER Reveal.js to HTML+JS (interactive web slides)
- HANDLE fonts and encoding (UTF-8, special characters, math fonts)
- OPTIMIZE output size (compress figures, minimize CSS, strip unused packages)
- GENERATE multiple output formats from single source (e.g., both PDF and MJML email)
- VALIDATE rendering (no missing fonts, correct page breaks, all images embedded)
- HANDLE rendering errors and provide actionable suggestions

**Modules that own this:**
- `latex-mcp` (LaTeX → PDF)
- `slides-mcp` (Beamer → PDF, Reveal.js → HTML)
- `blog-mcp` (Markdown → HTML)
- `newsletter-mcp` (MJML → HTML email)
- `grant-mcp` (LaTeX grant → PDF or Word)

**Lives in:** `core` (rendering orchestration) + module-specific engines (TeX, Node.js, pandoc)

**Why it matters:** Users only see the rendered output. A missing font or broken link kills credibility.

**Schemas needed:**
- `RenderConfig` — compiler choice, optimization flags, output format
- `RenderError` — font missing, page break failed, image not found, with recovery suggestions
- `OutputArtifact` — final file(s), size, mime type, metadata

---

### Cluster 8: Configuration & Customization

**What it does:** Let users control behavior, choose rendering options, set preferences, and customize output appearance.

**Capabilities:**
- CONFIGURE TeX engine (pdflatex, xelatex, lualatex; tradeoff: compatibility vs. Unicode support)
- CONFIGURE pandoc options (filters, reader/writer options, template)
- CHOOSE document class (article, report, book for LaTeX; theme for Reveal.js)
- SET page geometry (margins, column width, font size, line spacing)
- CHOOSE color schemes and themes (Beamer themes, blog CSS, email template)
- CONTROL output detail (show thinking/reasoning or hide; include/exclude TOC)
- MANAGE package dependencies (load only needed LaTeX packages to minimize bloat)
- CONFIGURE citation style (APA, Chicago, IEEE, custom)
- SET metadata (author, date, institution, subject tags)
- CHOOSE language/locale (affects typography, quotation marks, date formatting)

**Modules that own this:**
- `core` (shared configuration schema, validation)
- Each module (module-specific options: LaTeX class vs. slide theme, etc.)

**Lives in:** Shared config + module overrides

**Why it matters:** One size doesn't fit all. Researchers need control over packages and fonts; bloggers need theme choice; grant writers need funder-specific templates.

**Schemas needed:**
- `RenderConfig` — engine, compiler flags, output options
- `DocumentConfig` — class, margins, fonts, colors, metadata
- `ModuleConfig` — module-specific settings (e.g., BeamerTheme, BlogTemplate)

---

### Cluster 9: Error Handling, Validation & Diagnostics

**What it does:** Detect problems early, provide actionable error messages, suggest fixes. Ensure output quality before delivery.

**Capabilities:**
- VALIDATE input document (required sections present, no orphaned references, math syntax correct)
- VALIDATE bibliography (all citations resolvable, no duplicate keys, complete metadata)
- VALIDATE rendering (no missing fonts, correct image formats, page breaks reasonable)
- DETECT incompatibilities (e.g., UTF-8 math symbols in pdflatex without unicode support)
- CHECK link viability (optional: verify URLs aren't dead, DOI resolves)
- ENFORCE style rules (line length limits, code block formatting, heading capitalization)
- GENERATE helpful error messages with line/section numbers and suggestions
- PROVIDE warnings for non-fatal issues (unused citations, missing alt text, accessibility issues)
- SUGGEST fixes (e.g., "Missing abstract — templates require one" → provide template)
- PROFILE rendering (show timings, package load time, identify slow operations)

**Modules that own this:**
- `core` (shared validation framework, error schema)
- Each module (module-specific validators)

**Lives in:** `core` (validation framework) + module-specific validators

**Why it matters:** Early error detection prevents failed builds and frustrated users. Good diagnostics make debugging 10x faster.

**Schemas needed:**
- `ValidationError` — error code, message, location (file, line, section), severity, suggestion
- `ValidationResult` — is_valid, errors[], warnings[], metadata
- `DiagnosticReport` — full validation output with actionable fixes

---

### Cross-Cutting Concerns (Every Module Needs These)

| Concern | What it does | Owner | Status |
|---------|-------------|-------|--------|
| **Protocol Compliance** | Register MCP tools correctly, handle tool calls, return responses per spec | `core` (TypeScript MCP SDK wrapper) | 🔴 Critical dependency |
| **Process Orchestration** | Coordinate Python engine calls, manage async/subprocess, handle timeouts | `core` (Node.js subprocess manager) | 🔴 |
| **Logging & Observability** | Debug logs, performance metrics, error tracing | `core` (structured logging) | 🔴 |
| **Dependency Management** | Python environment, TeX packages, fonts, pandoc filters | `core` (initialization + per-module manifest) | 🔴 |
| **Testing & Validation** | Unit tests, integration tests, snapshot tests for output | Per module | 🔴 |
| **Security & Sandboxing** | Input validation, prevent code injection, safe subprocess execution | `core` (input sanitizer) | 🟡 Needs threat modeling |
| **Performance Caching** | Cache parsed documents, compiled PDFs, pandoc results | `core` (optional LRU cache) | 🔴 |
| **Documentation Generation** | Inline help, tool descriptions, example usage | Per module | 🔴 |

---

## Gap Analysis: Capabilities Missing (v0.x Backlog)

### High-Impact Gaps

**1. Semantic HTML Output (Blog/Newsletter)**
- **Gap:** No HTML-specific semantic elements (microdata, structured data, rich snippets)
- **Impact:** Blog posts and newsletters won't integrate with SEO tools, social sharing, or accessibility tools
- **What's needed:** `MetadataSchema` for OpenGraph, Twitter Card, Schema.org tags
- **Module affected:** `blog-mcp`, `newsletter-mcp`

**2. Image Handling & Optimization**
- **Gap:** No image processing (resize, compress, convert formats, generate alt text)
- **Impact:** Users must optimize images manually; PDFs may be bloated
- **What's needed:** `ImageProcessing` cluster (compress JPEG/PNG, detect faces for alt text, extract thumbnails)
- **Module affected:** All modules

**3. Accessibility Compliance**
- **Gap:** No checking for WCAG 2.1 compliance (alt text, color contrast, heading hierarchy)
- **Impact:** Rendered documents may not be accessible to screen readers
- **What's needed:** `AccessibilityValidator` (check alt text, heading order, color ratios)
- **Module affected:** Blog, Newsletter, Slides

**4. Interactive Content (Slides)**
- **Gap:** Reveal.js can do animations/interactions, but no abstraction to add them easily
- **Impact:** Web slides are static; no animations, polls, or interactive elements
- **What's needed:** `InteractiveSlide` schema, animation builder
- **Module affected:** `slides-mcp` (Reveal.js variant only)

**5. Collaborative Editing & Versioning**
- **Gap:** No multi-user editing, version control, or change tracking
- **Impact:** Team collaboration requires external tools (Git, Google Docs)
- **What's needed:** `DocumentVersion`, `ChangeTrack`, `MergeStrategy`
- **Module affected:** All modules

**6. Export Flexibility (Grant Proposals)**
- **Gap:** Grant proposals currently output LaTeX → PDF or Word; no support for platform-specific formats (NIH eRA Commons, Grants.gov)
- **Impact:** Users must reformat for submission platforms
- **What's needed:** `GrantSubmissionFormat` schema, platform-specific exporters
- **Module affected:** `grant-mcp`

### Medium-Impact Gaps

**7. Live Preview (Developer Experience)**
- **Gap:** No live preview while typing; users must render to see output
- **Impact:** Slower feedback loop; harder to debug formatting
- **What's needed:** Streaming renderer, partial document compilation, hot-reload mechanism

**8. Template Library & Discovery**
- **Gap:** Users don't know what templates exist or how to use them
- **Impact:** Low template adoption; people create from scratch
- **What's needed:** `TemplateRegistry`, template browsing tool, example gallery

**9. Math-to-Markdown Fallback**
- **Gap:** If LaTeX math can't be rendered in HTML/email, no graceful fallback
- **Impact:** Email newsletters and blog posts lose equations; readers see `[Math not rendered]`
- **What's needed:** `MathFallback` — convert LaTeX to Unicode, MathML, or PNG images

**10. Multi-Language Support**
- **Gap:** No support for non-English typography (CJK, Arabic, Indic scripts) or localization
- **Impact:** Users with non-Latin scripts can't use the system well
- **What's needed:** `LocaleConfig`, language-specific typography rules, font fallbacks

### Low-Priority Gaps (Future Modules)

**11. Presentation Mode (Slides)**
- **Gap:** No presenter view, speaker notes display, timing tools
- **What's needed:** Real-time presenter controller, notes on secondary screen

**12. Data Visualization from Structured Data**
- **Gap:** No automatic chart generation from CSV/JSON data
- **What's needed:** `DataVisualization` cluster (tables → charts, timeseries → graphs)

**13. Citation Retrieval & Enrichment**
- **Gap:** Users must manually find and enter citations; no automatic metadata fetching
- **What's needed:** API integration (CrossRef, arXiv, Semantic Scholar) to auto-populate citations

**14. Collaborative LaTeX Comments** (Future)
- **Gap:** No way for multiple reviewers to leave comments in rendered LaTeX
- **What's needed:** PDF annotation, change tracking, comment synthesis

---

## Overlap Map: Shared Capabilities (These Belong in `core`)

### Tier 1: Must Be Shared (Used by 3+ modules)

| Capability | Used by | Shared in `core` | Notes |
|-----------|---------|-----------------|-------|
| Markdown parsing & tokenization | All 5 | YES — `MDParser` | Core of everything |
| Frontmatter/metadata extraction | Blog, Newsletter, LaTeX, Slides, Grant | YES — `FrontmatterParser` | Every module needs metadata |
| Citation parsing & validation | LaTeX, Grant, Slides | YES — `CitationManager` | BibTeX is a shared dependency |
| Bibliography rendering (different formats) | LaTeX (BibTeX), Grant (LaTeX), Slides (footnotes) | PARTIAL — `CitationRenderer` in core; format-specific in modules | Core provides the data contract |
| Template engine | Blog, Newsletter, Grant, Slides | YES — `TemplateEngine` | Generic template application |
| Error/warning schema | All 5 | YES — `ValidationError` | Consistent error reporting |
| MCP protocol compliance | All 5 | YES — `MCPServer` base class | DRY principle |

### Tier 2: Should Be Shared (Used by 2+ modules)

| Capability | Used by | Current Location | Should be | Notes |
|-----------|---------|------------------|-----------|-------|
| LaTeX to PDF rendering | LaTeX, Grant | Module-specific | `core` or shared service | Both use same TeX pipeline; extract to avoid duplication |
| Table conversion (MD table → format) | LaTeX, Blog, Newsletter, Slides | Module-specific | `core` with format plugins | Generic table parsing in core; format-specific rendering in modules |
| Image reference handling | All 5 | Module-specific | `core` | Same logic: parse ref, validate path, embed or link |
| Code block formatting | LaTeX (minted), Blog (fenced code), Newsletter (escaped), Slides (highlight.js) | Module-specific | `core` with format plugins | Parse once, render per format |
| Heading hierarchy validation | All 5 | Module-specific | `core` | Ensure no orphaned levels (h1 → h3 jump is invalid) |

### Tier 3: Consider Sharing (Currently duplicated, may not be worth the effort)

| Capability | Used by | Trade-off | Recommendation |
|-----------|---------|-----------|-----------------|
| Style/color handling | Blog, Newsletter, Slides | Small duplication vs. added complexity | Keep separate for now; revisit in v0.3 |
| Font management | LaTeX, Slides (Reveal.js uses web fonts) | Very different (TeX vs. CSS) | Keep separate; not worth abstracting |
| Section ordering rules | Grant (NSF vs. NIH vs. ERC differ), LaTeX (standard order) | Grant has complex rules; others simple | Shared base with grant-specific overrides |

---

## Behavioral Identity Statement

**ZenithScience is a human-AI collaboration engine that converts unstructured thinking into publication-ready artifacts.**

More precisely: You bring the thinking (brainstorming notes, outlines, ideas, mathematical reasoning). ZenSci helps you formalize and structure that thinking, then renders it into whichever format your audience expects — an academic paper in LaTeX, a blog post in HTML, a grant proposal in PDF, a presentation in web slides. The AI speaks natively in each format; you focus on the ideas.

Unlike converters, ZenSci embeds a **thinking partnership mode**: before rendering, you and Claude can reason through proofs, arguments, and structure together. The thinking is captured and optionally embedded in output. This transforms the system from "tool that changes syntax" to "tool that helped you think better, then proved it by rendering beautifully."

ZenSci optimizes for **academic and research workflows** (the hardest use case — LaTeX, bibliography, mathematical rigor). It then extends to content and strategic writing (blog, newsletter, grant proposals, slide decks). One input format (markdown + reasoning), five publication formats. Shared infrastructure means fixes to parsing or bibliography management benefit all modules simultaneously.

---

## Naming Implications for Scout Research Agent

**From the semantic clusters, these are the core action verbs:**
- PARSE, FORMALIZE, STRUCTURE, CONVERT, RENDER, VALIDATE

**Note (Decision Applied 2026-02-17):** GitHub organization is **`TresPies-source`** (decided by Cruz Morales). The research below on organizational naming philosophy remains relevant for brand positioning and future suite identity, but does not override the GitHub org decision.

**Recommended vocabulary for public naming (GitHub org, repos):**

### Strong Candidates (Reflect Core Actions)

1. **Knowledge-Forge** (action: forge = shape raw material into refined artifact)
   - Org: ~~`knowledge-forge/`~~ → **`trespies-source`** (DECIDED)
   - Repos: `trespies-source/latex-mcp`, `trespies-source/blog-mcp`, etc.
   - Tone: Artisanal, intentional, craft-focused
   - Fits: All three user personas (developers, researchers, writers)

2. **StructureFlow** or **Zenflow** (action: structure + flow = organize and move through pipeline)
   - Org: ~~`structureflow/` or `zenflow/`~~ → **`trespies-source`** (DECIDED)
   - Repos: `trespies-source/latex-mcp`, `trespies-source/blog-mcp`, etc.
   - Tone: Clean, systematic, workflow-focused
   - Fits: Developer-first, but accessible to all

3. **PublishForge** or **DocForge** (action: forge documents for publishing)
   - Org: ~~`publishforge/` or `docforge/`~~ → **`trespies-source`** (DECIDED)
   - Repos: More module-focused
   - Tone: Publishing-centric, less "thinking partner"
   - Fits: Academic/researcher use case primarily

### Secondary Candidates

4. **MindToTeX** / **MindRender** (too narrow, focuses only on LaTeX)
5. **ThoughtLab** / **ResearchForge** (good for academic positioning, but alienates developers)
6. **StructureSync** / **PipelineForge** (too technical for general positioning)

**Recommendation:** GitHub org is **`TresPies-source`** (DECIDED 2026-02-17). Semantic research on "forge," "knowledge," "structure" remains relevant for internal positioning and suite branding if distinct from GitHub org.

---

## Cross-Cluster Dependencies & Architecture Implications

### Critical Dependencies (v0.1 blocker)

```
┌─────────────────────────────────────────────────┐
│ Core: Parsing & Validation Framework            │
│ (MD parser, Math expression, Frontmatter)       │
└──────────────────┬──────────────────────────────┘
                   │
      ┌────────────┼────────────┐
      ▼            ▼            ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ latex-mcp    │ │ blog-mcp     │ │ slides-mcp   │
│ (TeX engine) │ │ (HTML)       │ │ (Beamer/RJS) │
└──────────────┘ └──────────────┘ └──────────────┘
```

**Decision:** Build `core` first (v0.0.1). Then `latex-mcp` (v0.1) — it's the hardest and creates the architecture for everything else. Blog and Slides can follow in parallel; Grant needs LaTeX pipeline, so it comes after.

### Data Flow Through Clusters

```
USER INPUT (Markdown + Brainstorming Context)
    │
    ├─→ Parsing & Understanding
    │   (Extract structure, citations, math, metadata)
    │
    ├─→ [Optional] Thinking Partnership
    │   (AI-assisted reasoning, refinement)
    │
    ├─→ Document Structuring
    │   (Apply template, validate hierarchy)
    │
    ├─→ Configuration & Customization
    │   (Choose rendering options, theme)
    │
    ├─→ Format Conversion
    │   (Markdown → LaTeX/HTML/MJML/etc.)
    │
    ├─→ Error Handling & Validation
    │   (Check compatibility, suggest fixes)
    │
    ├─→ Rendering & Output
    │   (TeX engine, pandoc, JavaScript compile)
    │
    └─→ PUBLISHED ARTIFACT (PDF/HTML/Email)
```

---

## Schema Implications (Feed to `/schemas/SCHEMAS.md`)

### New Schemas Identified by This Audit

**To add to `SCHEMAS.md`:**

1. **Core / Shared Contracts**
   - `DocumentTree` — result of parsing markdown (hierarchical structure)
   - `ParseError` — structured error from parsing with recovery suggestions
   - `FrontmatterMetadata` — parsed YAML/JSON metadata from document header
   - `ValidationResult` — result of validation (is_valid, errors[], warnings[])

2. **Thinking Partnership (New Cluster)**
   - `ThinkingSession` — captures brainstorming context, decisions, reasoning
   - `ReasoningTrace` — decision point in reasoning (considered options, chose X)
   - `ProofStructure` — theorem statement, assumptions, proof steps, QED

3. **Mathematical (New Cluster)**
   - `MathExpression` — canonical form, LaTeX representation, alternatives
   - `ProofStep` — single step in proof (formula, justification, state)

4. **Document Structure (New Cluster)**
   - `LayoutRule` — format-specific constraints (e.g., max abstract length)
   - `TOCEntry` — single table-of-contents entry with hierarchy level

5. **Citation / Bibliography (New Cluster)**
   - `BibliographyStyle` — rules for rendering citations (IEEE, APA, Chicago)
   - `CrossReference` — reference to figure/theorem/footnote with context

6. **Configuration (New Cluster)**
   - `DocumentConfig` — document-level settings (class, margins, fonts, metadata)
   - `ModuleConfig` — module-specific settings (BeamerTheme, BlogTemplate, etc.)

7. **Rendering & Output (New Cluster)**
   - `RenderConfig` — TeX compiler choice, flags, output options
   - `RenderError` — rendering failure with suggestions (missing font, page break error)
   - `OutputArtifact` — final output file(s), size, mime type, metadata

8. **Validation & Diagnostics (New Cluster)**
   - `DiagnosticReport` — full validation output with helpful fixes
   - Already have `ValidationError` from core

---

## Recommended v0.1 → v0.4 Roadmap (Based on Clusters)

### v0.1: LaTeX Foundation (Week 1–3)

**Clusters to implement:**
1. Parsing & Understanding (core)
2. Mathematical Reasoning & Symbolic (core + latex-mcp)
3. Document Structuring (core + latex-mcp)
4. Configuration & Customization (core + latex-mcp)
5. Format Conversion (latex-mcp specific)
6. Rendering & Output (latex-mcp specific)
7. Error Handling & Validation (core + latex-mcp)

**Skip for v0.1:**
- Thinking Partnership (defer; LaTeX alone is hard enough)
- Citation management (use basic BibTeX refs; enhance in v0.2)

**Success criteria:** MCP server accepts markdown, outputs LaTeX/PDF with correct math, figures, basic citations.

### v0.2: Blog & Newsletter (Week 4–5)

**New clusters:**
- Format Conversion (HTML/MJML variants)
- Rendering & Output (HTML, MJML engines)

**Reuse from v0.1:**
- Parsing & Understanding (core)
- Document Structuring (core + blog/newsletter-specific layouts)
- Configuration & Customization
- Error Handling

**New gap:** SEO/metadata (add `MetadataSchema`).

### v0.3: Slides (Week 6–7)

**New clusters:**
- Configuration: Beamer themes vs. Reveal.js themes
- Rendering: TeX slides vs. web slides (different engines)

**Reuse:** Everything from v0.1 (TeX rendering) + v0.2 (HTML rendering).

### v0.4: Grant Proposals (Week 8–10)

**New clusters:**
- Document Structuring: Grant-specific section ordering (NSF, NIH, ERC templates)
- Citation Management: Grant proposals have strict bibliography rules

**Reuse:** LaTeX from v0.1, citation management from core.

### Post-v0.4: Thinking Partnership & Image Handling

Once all formats ship, add the differentiated clusters:
- Thinking Partnership & Reasoning (Claude + LaTeX brainstorming)
- Image handling & optimization (all modules)
- Accessibility compliance (Blog, Newsletter, Slides)

---

## Checkpoints for Future Audits

This semantic cluster map should be revisited:

1. **After v0.1 ships** — Verify actual implementation matches cluster structure. Adjust if needed.
2. **When adding modules** — Do new modules map to existing clusters, or do they reveal new clusters?
3. **When user feedback accumulates** — Does actual usage align with the thinking-partner positioning? Or is conversion the primary value?
4. **Quarterly** — Check if gap analysis is being addressed, and reprioritize.

---

## Related Documents

- **CONTEXT.md** — Project vision and user personas
- **scouts/2026-02-17_module-strategy-scout.md** — Module strategy and routing options
- **schemas/SCHEMAS.md** — Running data contracts backlog (update with new schemas from this audit)
- **research/2026-02-17_naming-research.md** — Naming research (informs vocabulary section here)

---

## Decisions Applied (Cruz Morales, 2026-02-17)

### GitHub Organization — DECIDED
- **GitHub org:** `TresPies-source` (Decision 4)
- **npm scope:** `@zen-sci` (Decision 4, supersedes `@typecraft` from naming research)
- **Domain:** ZenithScience.org (Decision 3)
- **IP Owner:** TresPiesDesign.com / Cruz Morales (Decision 1)

### Working Directory
- **Code dir:** `ZenflowProjects/ZenithScience/zen-sci/` (kebab-case, Decision 5)

### What Remains Open
- Whether "forge" and "knowledge" semantic research informs suite brand positioning
- Exact npm scope finalization (`@zen-sci` vs. `@zensci`)
- MCP registry namespace finalization (`io.github.trespies-source` vs. alternatives)

---

**End of Semantic Cluster Map**
Generated by System Health Auditor (2026-02-17)
Updated with Decisions (Cruz Morales, 2026-02-17)
