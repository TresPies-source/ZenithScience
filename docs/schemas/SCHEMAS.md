# ZenSci — Running Schemas List

**Owner:** TresPiesDesign.com / Cruz Morales
**Product:** ZenSci
**GitHub:** github.com/TresPies-source/ZenithScience
**Domain:** ZenithScience.org
**npm scope:** @zen-sci
**Code dir:** ZenflowProjects/ZenithScience/zen-sci/
**Last updated:** 2026-02-18

---

This file tracks every data contract (schema/type/interface) that needs to be formally specified.
Updated as architecture decisions accumulate.

**Status legend:**
- 🔴 Not started — identified but no definition written anywhere
- 🟡 Drafted in spec — TypeScript interface exists in a spec file; not yet in code
- 🟠 Needs review — drafted, but may need audit for accuracy / cross-spec consistency
- 🟢 Finalized — implemented in `zen-sci/packages/core/src/types/` with tests passing

---

## Core / Shared Contracts

| Schema | Description | Status | Spec Source |
|--------|-------------|--------|-------------|
| `DocumentRequest` | The shared input contract for all modules. Source content, target format, options, metadata. | 🟡 | `specs/infrastructure/packages-core-spec.md` |
| `DocumentResponse` | Structured output from any module. Content, format, warnings, artifacts, metadata. | 🟡 | `specs/infrastructure/packages-core-spec.md` |
| `OutputFormat` | Union type of all supported output formats (`'latex' \| 'html' \| 'beamer' \| ...`). | 🟡 | `specs/infrastructure/packages-core-spec.md` |
| `DocumentOptions` | Per-request options: title, author, date, bibliography, math, TOC, module overrides. | 🟡 | `specs/infrastructure/packages-core-spec.md` |
| `ConversionPipeline` | Describes processing steps a request goes through (parse → transform → render → validate). | 🟡 | `specs/infrastructure/packages-core-spec.md` |
| `PipelineStage` | A single pipeline step with name, status, elapsed time, error. | 🟡 | `specs/infrastructure/packages-core-spec.md` |
| `ModuleManifest` | How a module declares itself: name, version, tools, capabilities, dependencies. | 🟡 | `specs/infrastructure/packages-sdk-spec.md` |
| `MCPToolDefinition` | How individual MCP tools are registered — name, description, inputSchema, outputSchema. | 🟡 | `specs/infrastructure/packages-sdk-spec.md` |
| `ThinkingSession` | Schema for AI-assisted brainstorming sessions before conversion. Captures reasoning, decisions, intent. | 🟡 | `specs/infrastructure/packages-core-spec.md` |
| `CitationRecord` | Shared bibliography/citation schema used by LaTeX, Grant, and Slides modules. | 🟡 | `specs/infrastructure/packages-core-spec.md` |
| `TemplateDefinition` | How output templates are defined per module — structure, required fields, optional sections. | 🟡 | `specs/infrastructure/packages-core-spec.md` |
| `ConversionError` | Structured error type for conversion failures — code, message, location, suggestions. | 🟡 | `specs/infrastructure/packages-core-spec.md` |
| `Artifact` | Additional generated file from a response (PDF, .bib, EPUB, etc.) — type, buffer, mime. | 🟡 | `specs/infrastructure/packages-core-spec.md` |
| `ResponseMetadata` | Metadata attached to DocumentResponse — format version, pandoc version, timestamps. | 🟡 | `specs/infrastructure/packages-core-spec.md` |

---

## Infrastructure Contracts

| Schema | Description | Status | Spec Source |
|--------|-------------|--------|-------------|
| `PandocJob` | Input to the Python processing engine — source, target format, options, filters, template. | 🟡 | `specs/infrastructure/packages-sdk-spec.md` |
| `PandocResult` | Output from the Python engine — content, warnings, exit code, elapsed time. | 🟡 | `specs/infrastructure/packages-sdk-spec.md` |
| `RenderConfig` | TeX engine configuration — compiler (pdflatex, xelatex, lualatex), passes, output dir. | 🟡 | `specs/infrastructure/packages-sdk-spec.md` |
| `ModuleDependency` | Declares a module's runtime dependency — name, version, type (npm, python, system). | 🟡 | `specs/infrastructure/packages-sdk-spec.md` |

---

## Semantic Cluster Schemas (from Architecture Analysis)

These schemas emerged from the 9 semantic clusters (`architecture/2026-02-17_semantic-clusters.md`). Many overlap with infrastructure contracts — cross-reference before implementing.

### Cluster 1 — Parsing

| Schema | Description | Status | Spec Source |
|--------|-------------|--------|-------------|
| `DocumentTree` | Hierarchical structure of a parsed document (sections, nodes, math, citations). | 🟡 | `specs/infrastructure/packages-core-spec.md` |
| `ParseError` | Structured error from parsing stage with recovery suggestions and location. | 🟡 | `specs/infrastructure/packages-core-spec.md` |
| `FrontmatterMetadata` | Parsed YAML/JSON metadata from document header — typed key-value pairs. | 🟡 | `specs/infrastructure/packages-core-spec.md` |
| `ValidationResult` | Result of input validation — is_valid, errors[], warnings[], metadata. | 🟡 | `specs/infrastructure/packages-core-spec.md` |

### Cluster 2 — Thinking Partnership

| Schema | Description | Status | Spec Source |
|--------|-------------|--------|-------------|
| `ReasoningTrace` | A decision point in reasoning — alternatives considered, choice made, rationale. | 🟡 | `specs/infrastructure/packages-core-spec.md` |
| `ProofStructure` | Theorem statement, assumptions, proof steps, QED marker. | 🟡 | `specs/phase-1/latex-mcp-v0.1-spec.md` |

### Cluster 3 — Document Structuring

| Schema | Description | Status | Spec Source |
|--------|-------------|--------|-------------|
| `DocumentSection` | A section node — type, heading, content, metadata, subsections. | 🟡 | `specs/infrastructure/packages-core-spec.md` |
| `LayoutRule` | Format-specific layout constraint (e.g., max abstract length, column count). | 🟡 | `specs/infrastructure/packages-core-spec.md` |
| `TOCEntry` | Single table-of-contents entry with level, title, anchor/page number. | 🟡 | `specs/infrastructure/packages-core-spec.md` |

### Cluster 5 — Mathematical Reasoning

| Schema | Description | Status | Spec Source |
|--------|-------------|--------|-------------|
| `MathExpression` | A parsed/validated math expression — canonical form, LaTeX representation, alternatives. | 🟡 | `specs/phase-1/latex-mcp-v0.1-spec.md` |
| `ProofStep` | A single proof step — formula, justification, symbolic state. | 🟡 | `specs/phase-1/latex-mcp-v0.1-spec.md` |
| `MathEnvironment` | A named math environment (align, equation, cases, matrix) with content and constraints. | 🟡 | `specs/phase-1/latex-mcp-v0.1-spec.md` |

### Cluster 6 — Citation & Bibliography

| Schema | Description | Status | Spec Source |
|--------|-------------|--------|-------------|
| `BibliographyStyle` | Citation format rules — style name (IEEE, APA, Chicago), format mappings. | 🟡 | `specs/infrastructure/packages-core-spec.md` |
| `CrossReference` | Internal reference to figure/theorem/footnote — key, label, display text, context. | 🟡 | `specs/infrastructure/packages-core-spec.md` |

### Clusters 7–9 — Rendering, Configuration, Validation

| Schema | Description | Status | Spec Source |
|--------|-------------|--------|-------------|
| `RenderError` | Rendering failure — missing font, page break error, broken image — with suggestions. | 🟡 | `specs/infrastructure/packages-sdk-spec.md` |
| `OutputArtifact` | Final output file(s) — path, size, mime type, format metadata. | 🟡 | `specs/infrastructure/packages-core-spec.md` |
| `DocumentConfig` | Document-level settings — class, margins, fonts, colors, locale. | 🟡 | `specs/infrastructure/packages-core-spec.md` |
| `ModuleConfig` | Module-specific settings — BeamerTheme, BlogTemplate, GrantFunder, etc. | 🟡 | Per-module specs (see phase-1/, phase-2/) |
| `DiagnosticReport` | Full validation output with all errors, warnings, and actionable fixes. | 🟡 | `specs/infrastructure/packages-core-spec.md` |
| `ConversionRule` | Mapping from a markdown/AST construct to its target format representation. | 🟡 | `specs/infrastructure/packages-core-spec.md` (§3.4) |
| `FormatConstraint` | Format-specific structural and sizing limits — margins, max pages, max width, aspect ratio. | 🟡 | `specs/infrastructure/packages-core-spec.md` (§3.4) |
| `ImageProcessingOptions` | Configuration for image resize, compress, format convert, and alt-text generation. | 🟡 | `specs/infrastructure/packages-core-spec.md` (§3.4) |
| `ImageProcessingResult` | Output of a single image processing operation — buffer, dimensions, alt text, warnings. | 🟡 | `specs/infrastructure/packages-core-spec.md` (§3.4) |
| `AccessibilityReport` | WCAG 2.1 compliance results for HTML/email/slides — level, violations, score. | 🟡 | `specs/infrastructure/packages-core-spec.md` (§3.4) |
| `WebMetadataSchema` | Base OpenGraph + Twitter Card + Schema.org structured data for web output formats. | 🟡 | `specs/infrastructure/packages-core-spec.md` (§3.4) |
| `DocumentVersion` | A single named version of a document with change list and workflow tag. | 🟡 | `specs/infrastructure/packages-core-spec.md` (§3.4) |
| `DocumentVersionHistory` | Full version history for a document — ordered list of DocumentVersion entries. | 🟡 | `specs/infrastructure/packages-core-spec.md` (§3.4) |

---

## Module-Specific Contracts

### latex-mcp (Phase 1)

| Schema | Description | Status | Spec Source |
|--------|-------------|--------|-------------|
| `LaTeXDocument` | Full LaTeX output — preamble, packages, sections, environments, bibliography. | 🟡 | `specs/phase-1/latex-mcp-v0.1-spec.md` |
| `LaTeXEnvironment` | A named LaTeX environment (theorem, proof, lemma, equation, figure, table). | 🟡 | `specs/phase-1/latex-mcp-v0.1-spec.md` |
| `PackageDeclaration` | A LaTeX package with options and version constraints. | 🟡 | `specs/phase-1/latex-mcp-v0.1-spec.md` |

### blog-mcp (Phase 1)

| Schema | Description | Status | Spec Source |
|--------|-------------|--------|-------------|
| `BlogPost` | Structured blog output — title, slug, tags, sections, frontmatter, SEO metadata. | 🟡 | `specs/phase-1/blog-mcp-v0.2-spec.md` |
| `ContentSection` | A reusable section schema shared by blog and newsletter — type, content, formatting. | 🟡 | `specs/phase-1/blog-mcp-v0.2-spec.md` |
| `SEOMetadata` | OpenGraph, Twitter Card, Schema.org metadata for blog/newsletter. | 🟡 | `specs/phase-1/blog-mcp-v0.2-spec.md` |

### paper-mcp (Phase 1)

| Schema | Description | Status | Spec Source |
|--------|-------------|--------|-------------|
| `ResearchPaper` | Full paper structure — abstract, introduction, related work, methods, results, discussion, references. | 🟡 | `specs/phase-1/paper-mcp-spec.md` |
| `AcademicTemplate` | Template abstraction — IEEE/ACM/arXiv/ProQuest/institutional LaTeX templates. | 🟡 | `specs/phase-1/paper-mcp-spec.md` |

### lab-notebook-mcp (Phase 1 — Strategic Differentiator)

| Schema | Description | Status | Spec Source |
|--------|-------------|--------|-------------|
| `LabNotebookEntry` | Single lab entry — date, hypothesis, methodology, observations, reasoning chain, tags, citations. | 🟡 | `specs/phase-1/lab-notebook-mcp-spec.md` |
| `LabNotebook` | Collection of entries with cross-referencing, reproducibility metadata, data provenance. | 🟡 | `specs/phase-1/lab-notebook-mcp-spec.md` |
| `ReproducibilityMetadata` | Environment, dependencies, external data sources for reproducible science. | 🟡 | `specs/phase-1/lab-notebook-mcp-spec.md` |

### slides-mcp (Phase 2)

| Schema | Description | Status | Spec Source |
|--------|-------------|--------|-------------|
| `SlidesDeck` | Full deck — title, theme, slides[], format (Beamer or Reveal.js), speaker notes. | 🟡 | `specs/phase-2/slides-mcp-spec.md` |
| `Slide` | Individual slide — type, title, content, layout, notes, transitions. | 🟡 | `specs/phase-2/slides-mcp-spec.md` |
| `BeamerTheme` | Beamer-specific theme and color scheme declaration. | 🟡 | `specs/phase-2/slides-mcp-spec.md` |
| `RevealConfig` | Reveal.js configuration — theme, plugins, transitions, highlight.js settings. | 🟡 | `specs/phase-2/slides-mcp-spec.md` |

### newsletter-mcp (Phase 2)

| Schema | Description | Status | Spec Source |
|--------|-------------|--------|-------------|
| `NewsletterIssue` | Email newsletter — subject, preview text, sections, CTA, platform metadata, MJML output. | 🟡 | `specs/phase-2/newsletter-mcp-spec.md` |

### grant-mcp (Phase 2)

| Schema | Description | Status | Spec Source |
|--------|-------------|--------|-------------|
| `GrantProposal` | Full grant proposal — title, funder, sections (specific aims, budget, timeline), citations. | 🟡 | `specs/phase-2/grant-mcp-spec.md` |
| `GrantTemplate` | A funder-specific template structure (NIH, NSF, ERC, etc.) with compliance rules. | 🟡 | `specs/phase-2/grant-mcp-spec.md` |
| `ResearchAim` | A specific aim or objective within a grant proposal. | 🟡 | `specs/phase-2/grant-mcp-spec.md` |
| `ComplianceMetadata` | Funder/institution/jurisdiction compliance rules and formatting requirements. | 🟡 | `specs/phase-2/grant-mcp-spec.md` |
| `GrantSubmissionFormat` | Platform-specific packaging config for submission portals (Grants.gov, NIH eRA Commons, NSF FastLane). | 🟡 | `specs/phase-2/grant-mcp-spec.md` (§3.6) |
| `SubmissionFile` | A single required file in a grant submission package — role, mime types, page limit. | 🟡 | `specs/phase-2/grant-mcp-spec.md` (§3.6) |

### thesis-mcp (Phase 2)

| Schema | Description | Status | Spec Source |
|--------|-------------|--------|-------------|
| `ThesisMeta` | Thesis-specific metadata — institution, degree level, advisor(s), defense date, committee. | 🟡 | `specs/phase-2/thesis-mcp-spec.md` |

### whitepaper-mcp (Phase 2)

| Schema | Description | Status | Spec Source |
|--------|-------------|--------|-------------|
| `WhitepaperStructure` | Whitepaper sections — overview, background, analysis, recommendations, conclusions. | 🟡 | `specs/phase-2/whitepaper-mcp-spec.md` |

### patent-mcp (Phase 2)

| Schema | Description | Status | Spec Source |
|--------|-------------|--------|-------------|
| `PatentClaim` | Single patent claim with dependent/independent relationships, formal structure, scope. | 🟡 | `specs/phase-2/patent-mcp-spec.md` |
| `PatentApplication` | Full application — title, abstract, drawings, specification, claims (as dependency tree). | 🟡 | `specs/phase-2/patent-mcp-spec.md` |

### ebook-mcp (Phase 2)

| Schema | Description | Status | Spec Source |
|--------|-------------|--------|-------------|
| `EpubMetadata` | EPUB-specific metadata — identifier, language, creator, date, rights, direction. | 🟡 | `specs/phase-2/ebook-mcp-spec.md` |
| `MultiFormatOutput` | Simultaneous output to LaTeX, EPUB, PDF, HTML from single source. | 🟡 | `specs/phase-2/ebook-mcp-spec.md` |
| `BookStructure` | Chapter organization, front/back matter, TOC, index, page breaks. | 🟡 | `specs/phase-2/ebook-mcp-spec.md` |

### documentation-mcp (Phase 2)

| Schema | Description | Status | Spec Source |
|--------|-------------|--------|-------------|
| `DocumentationConfig` | Sphinx/MkDocs integration — project structure, theme, autodoc settings, plugin list. | 🟡 | `specs/phase-2/documentation-mcp-spec.md` |
| `APIDocs` | Auto-generated API docs — function signatures, docstrings, examples, deprecation info. | 🟡 | `specs/phase-2/documentation-mcp-spec.md` |
| `APIIntegration` | Contract for auto-extracting signatures from Python/TypeScript source code. | 🟡 | `specs/phase-2/documentation-mcp-spec.md` |

### Phase 3 (Low priority — ship on demand)

| Schema | Description | Status | Spec Source |
|--------|-------------|--------|-------------|
| `PolicyBriefStructure` | Policy brief sections — exec summary, issue analysis, options, recommendation, citations. | 🟡 | `specs/phase-3/policy-brief-mcp-spec.md` |
| `ProposalStructure` | Business proposal — exec summary, problem, solution, timeline, budget, team, CTA. | 🟡 | `specs/phase-3/proposal-mcp-spec.md` |

---

## Summary

| Count | Status |
|-------|--------|
| 79 | Total schemas tracked |
| 79 | 🟡 Drafted in spec files (2026-02-18) |
| 0 | 🔴 Not yet drafted |
| 0 | 🟢 Finalized in code |

**Next milestone:** Move `DocumentRequest`, `DocumentResponse`, `ConversionPipeline` from 🟡 to 🟢 by implementing `zen-sci/packages/core/src/types/`.

---

*Last updated: 2026-02-18 — all 7 previously-undrafted schemas now drafted; 0 schemas remain 🔴.*
*Maintained by: ZenSci architecture agents | Owner: TresPiesDesign.com / Cruz Morales*
