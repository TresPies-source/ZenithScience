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
- 🟢 Finalized — implemented in `zen-sci/` with tests passing

---

## Core / Shared Contracts

| Schema | Description | Status | Location |
|--------|-------------|--------|----------|
| `DocumentRequest` | The shared input contract for all modules. Source content, target format, options, metadata. | 🟢 | `packages/core/src/types.ts` |
| `DocumentResponse` | Structured output from any module. Content, format, warnings, artifacts, metadata. | 🟢 | `packages/core/src/types.ts` |
| `OutputFormat` | Union type of all supported output formats (22 values). | 🟢 | `packages/core/src/types.ts` |
| `DocumentOptions` | Per-request options: title, author, date, bibliography, math, TOC, module overrides. | 🟢 | `packages/core/src/types.ts` |
| `ConversionPipelineData` | Describes processing steps a request goes through (parse → transform → render → validate). | 🟢 | `packages/core/src/types.ts` |
| `PipelineStage` | A single pipeline step with name, status, elapsed time, error. | 🟢 | `packages/core/src/types.ts` |
| `ModuleManifest` | How a module declares itself: name, version, tools, capabilities, dependencies. | 🟢 | `packages/sdk/src/factory/module-manifest.ts` |
| `MCPToolDefinition` | How individual MCP tools are registered — name, description, inputSchema, outputSchema. | 🟢 | `packages/sdk/src/factory/module-manifest.ts` |
| `ThinkingSession` | Schema for AI-assisted brainstorming sessions before conversion. | 🟢 | `packages/core/src/types.ts` |
| `CitationRecord` | Shared bibliography/citation schema used by LaTeX, Grant, and Slides modules. | 🟢 | `packages/core/src/types.ts` |
| `TemplateDefinition` | How output templates are defined per module — structure, required fields, optional sections. | 🟡 | `specs/infrastructure/packages-core-spec.md` |
| `ConversionErrorData` | Structured error data (wire format) — code, message, location. | 🟢 | `packages/core/src/types.ts` |
| `ConversionError` | Runtime error class with `fromData()` / `toData()` bridge. | 🟢 | `packages/core/src/errors.ts` |
| `Artifact` | Additional generated file from a response (PDF, .bib, EPUB, etc.) — type, buffer, mime. | 🟢 | `packages/core/src/types.ts` |
| `ResponseMetadata` | Metadata attached to DocumentResponse — format version, pandoc version, timestamps. | 🟢 | `packages/core/src/types.ts` |

---

## Infrastructure Contracts

| Schema | Description | Status | Location |
|--------|-------------|--------|----------|
| `ZenSciServerConfig` | Config for createZenSciServer() factory — name, version, manifest. | 🟢 | `packages/sdk/src/types.ts` |
| `ZenSciContext` | Runtime context returned by factory — server, logger, config, manifest. | 🟢 | `packages/sdk/src/types.ts` |
| `ConversionPipelineOptions` | Options for runConversionPipeline() — source, format, render, hooks. | 🟢 | `packages/sdk/src/types.ts` |
| `RenderConfig` | TeX engine configuration — compiler (pdflatex, xelatex, lualatex), passes, output dir. | 🟡 | `specs/infrastructure/packages-sdk-spec.md` |
| `ModuleDependency` | Declares a module's runtime dependency — name, version, type (npm, python, system). | 🟡 | `specs/infrastructure/packages-sdk-spec.md` |

---

## Semantic Cluster Schemas (from Architecture Analysis)

These schemas emerged from the 9 semantic clusters (`architecture/2026-02-17_semantic-clusters.md`). Many overlap with infrastructure contracts — cross-reference before implementing.

### Cluster 1 — Parsing

| Schema | Description | Status | Location |
|--------|-------------|--------|----------|
| `DocumentTree` | Hierarchical structure of a parsed document (sections, nodes, math, citations). | 🟢 | `packages/core/src/types.ts` |
| `DocumentNode` | Union type of all AST node types (SectionNode, ParagraphNode, MathNode, etc.). | 🟢 | `packages/core/src/types.ts` |
| `FrontmatterMetadata` | Parsed YAML/JSON metadata from document header — typed key-value pairs. | 🟢 | `packages/core/src/types.ts` |
| `ValidationResult` | Result of input validation — valid, errors[], warnings[], metadata. | 🟢 | `packages/core/src/types.ts` |

### Cluster 2 — Thinking Partnership

| Schema | Description | Status | Location |
|--------|-------------|--------|----------|
| `ReasoningStep` | A step in reasoning — alternatives considered, choice made, rationale. | 🟢 | `packages/core/src/types.ts` |
| `ProofStructure` | Theorem statement, assumptions, proof steps, QED marker. | 🟡 | `specs/phase-1/latex-mcp-v0.1-spec.md` |

### Cluster 3 — Document Structuring

| Schema | Description | Status | Location |
|--------|-------------|--------|----------|
| `SectionNode` | A section node in the AST — type, heading, children. | 🟢 | `packages/core/src/types.ts` |
| `LayoutRule` | Format-specific layout constraint (e.g., max abstract length, column count). | 🟡 | `specs/infrastructure/packages-core-spec.md` |
| `TOCEntry` | Single table-of-contents entry with level, title, anchor/page number. | 🟡 | `specs/infrastructure/packages-core-spec.md` |

### Cluster 5 — Mathematical Reasoning

| Schema | Description | Status | Location |
|--------|-------------|--------|----------|
| `MathNode` | A math AST node — latex content, mode (inline/display). | 🟢 | `packages/core/src/types.ts` |
| `ProofStep` | A single proof step — formula, justification, symbolic state. | 🟡 | `specs/phase-1/latex-mcp-v0.1-spec.md` |
| `MathEnvironment` | A named math environment (align, equation, cases, matrix) with content and constraints. | 🟡 | `specs/phase-1/latex-mcp-v0.1-spec.md` |

### Cluster 6 — Citation & Bibliography

| Schema | Description | Status | Location |
|--------|-------------|--------|----------|
| `BibliographyStyle` | Citation style union type — IEEE, APA, Chicago, etc. | 🟢 | `packages/core/src/types.ts` |
| `CitationNode` | Citation AST node with keys and prefix/suffix. | 🟢 | `packages/core/src/types.ts` |
| `CrossReference` | Internal reference to figure/theorem/footnote — key, label, display text, context. | 🟡 | `specs/infrastructure/packages-core-spec.md` |

### Clusters 7–9 — Rendering, Configuration, Validation

| Schema | Description | Status | Location |
|--------|-------------|--------|----------|
| `ConversionRule` | Mapping from a markdown/AST construct to its target format representation. | 🟢 | `packages/core/src/types.ts` |
| `FormatConstraint` | Format-specific structural and sizing limits — margins, max pages, max width, aspect ratio. | 🟢 | `packages/core/src/types.ts` |
| `ImageProcessingOptions` | Configuration for image resize, compress, format convert, and alt-text generation. | 🟢 | `packages/core/src/types.ts` |
| `ImageProcessingResult` | Output of a single image processing operation — buffer, dimensions, alt text, warnings. | 🟢 | `packages/core/src/types.ts` |
| `AccessibilityReport` | WCAG 2.1 compliance results for HTML/email/slides — level, violations, score. | 🟢 | `packages/core/src/types.ts` |
| `WebMetadataSchema` | Base OpenGraph + Twitter Card + Schema.org structured data for web output formats. | 🟢 | `packages/core/src/types.ts` |
| `DocumentVersion` | A single named version of a document with change list and workflow tag. | 🟢 | `packages/core/src/types.ts` |
| `DocumentVersionHistory` | Full version history for a document — ordered list of DocumentVersion entries. | 🟢 | `packages/core/src/types.ts` |
| `RenderError` | Rendering failure — missing font, page break error, broken image — with suggestions. | 🟡 | `specs/infrastructure/packages-sdk-spec.md` |
| `OutputArtifact` | Final output file(s) — path, size, mime type, format metadata. | 🟡 | `specs/infrastructure/packages-core-spec.md` |
| `DocumentConfig` | Document-level settings — class, margins, fonts, colors, locale. | 🟡 | `specs/infrastructure/packages-core-spec.md` |
| `ModuleConfig` | Module-specific settings — BeamerTheme, BlogTemplate, GrantFunder, etc. | 🟡 | Per-module specs (see phase-1/, phase-2/) |
| `DiagnosticReport` | Full validation output with all errors, warnings, and actionable fixes. | 🟡 | `specs/infrastructure/packages-core-spec.md` |

---

## Module-Specific Contracts

### latex-mcp (Implemented — 37 tests)

| Schema | Description | Status | Location |
|--------|-------------|--------|----------|
| `ConvertToPdfArgs` | Input args for convert_to_pdf tool. | 🟢 | `servers/latex-mcp/src/tools/convert-to-pdf.ts` |
| `ConvertToPdfResult` | Output of convert_to_pdf — latex_source, pdf_base64, page_count, warnings. | 🟢 | `servers/latex-mcp/src/tools/convert-to-pdf.ts` |
| `ValidateDocumentArgs` | Input args for validate_document tool. | 🟢 | `servers/latex-mcp/src/tools/validate-document.ts` |
| `ValidateDocumentResult` | Output of validate_document — valid, errors, warnings, citationStats, mathStats. | 🟢 | `servers/latex-mcp/src/tools/validate-document.ts` |
| `CheckCitationsArgs` | Input args for check_citations tool. | 🟢 | `servers/latex-mcp/src/tools/check-citations.ts` |
| `CheckCitationsResult` | Output of check_citations — resolved, unresolved, bibliography_entries. | 🟢 | `servers/latex-mcp/src/tools/check-citations.ts` |

### blog-mcp (Implemented — 76 tests)

| Schema | Description | Status | Location |
|--------|-------------|--------|----------|
| `ConvertToHtmlResult` | Output of convert_to_html — html, seo, rss_entry, word_count, warnings. | 🟢 | `servers/blog-mcp/src/tools/convert-to-html.ts` |
| `ValidatePostResult` | Output of validate_post — valid, errors, warnings, seo_score. | 🟢 | `servers/blog-mcp/src/tools/validate-post.ts` |
| `GenerateFeedResult` | Output of generate_feed — rss_xml, atom_xml, entry_count. | 🟢 | `servers/blog-mcp/src/tools/generate-feed.ts` |

### grant-mcp (Implemented — 50 tests)

| Schema | Description | Status | Location |
|--------|-------------|--------|----------|
| `GenerateProposalResult` | Output of generate_proposal — latex_source, sections, warnings. | 🟢 | `servers/grant-mcp/src/tools/generate-proposal.ts` |
| `ValidateComplianceResult` | Output of validate_compliance — compliant, issues, score. | 🟢 | `servers/grant-mcp/src/tools/validate-compliance.ts` |
| `CheckFormatResult` | Output of check_format — valid, errors, warnings. | 🟢 | `servers/grant-mcp/src/tools/check-format.ts` |
| `GrantSubmissionFormat` | Platform-specific packaging config (local type, not in core). | 🟢 | `servers/grant-mcp/src/formatting/compliance-checker.ts` |

### slides-mcp (Implemented — 34 tests)

| Schema | Description | Status | Location |
|--------|-------------|--------|----------|
| `ConvertToSlidesResult` | Output of convert_to_slides — output, format, slide_count, has_notes. | 🟢 | `servers/slides-mcp/src/tools/convert-to-slides.ts` |
| `ValidateDeckResult` | Output of validate_deck — valid, slide_count, errors, warnings. | 🟢 | `servers/slides-mcp/src/tools/validate-deck.ts` |

### newsletter-mcp (Implemented — 63 tests)

| Schema | Description | Status | Location |
|--------|-------------|--------|----------|
| `ConvertToEmailResult` | Output of convert_to_email — html, estimated_size_bytes, warnings. | 🟢 | `servers/newsletter-mcp/src/tools/convert-to-email.ts` |
| `ValidateEmailResult` | Output of validate_email — valid, errors, warnings, accessibility_issues. | 🟢 | `servers/newsletter-mcp/src/tools/validate-email.ts` |

### paper-mcp (Implemented — 36 tests)

| Schema | Description | Status | Location |
|--------|-------------|--------|----------|
| `ConvertToPaperResult` | Output of convert_to_paper — latex_source, format, abstract_word_count, warnings. | 🟢 | `servers/paper-mcp/src/tools/convert-to-paper.ts` |
| `ValidateSubmissionResult` | Output of validate_submission — submission_ready, errors, warnings. | 🟢 | `servers/paper-mcp/src/tools/validate-submission.ts` |
| `PaperTemplate` | Template for IEEE/ACM/arXiv — documentClass, classOptions, preambleLines, columnsMode. | 🟢 | `servers/paper-mcp/src/templates/template-registry.ts` |

### lab-notebook-mcp (Not yet implemented — Phase 2+)

| Schema | Description | Status | Spec Source |
|--------|-------------|--------|-------------|
| `LabNotebookEntry` | Single lab entry — date, hypothesis, methodology, observations, reasoning chain, tags, citations. | 🟡 | `specs/phase-1/lab-notebook-mcp-spec.md` |
| `LabNotebook` | Collection of entries with cross-referencing, reproducibility metadata, data provenance. | 🟡 | `specs/phase-1/lab-notebook-mcp-spec.md` |
| `ReproducibilityMetadata` | Environment, dependencies, external data sources for reproducible science. | 🟡 | `specs/phase-1/lab-notebook-mcp-spec.md` |

### thesis-mcp (Not yet implemented)

| Schema | Description | Status | Spec Source |
|--------|-------------|--------|-------------|
| `ThesisMeta` | Thesis-specific metadata — institution, degree level, advisor(s), defense date, committee. | 🟡 | `specs/phase-2/thesis-mcp-spec.md` |

### whitepaper-mcp (Not yet implemented)

| Schema | Description | Status | Spec Source |
|--------|-------------|--------|-------------|
| `WhitepaperStructure` | Whitepaper sections — overview, background, analysis, recommendations, conclusions. | 🟡 | `specs/phase-2/whitepaper-mcp-spec.md` |

### patent-mcp (Not yet implemented)

| Schema | Description | Status | Spec Source |
|--------|-------------|--------|-------------|
| `PatentClaim` | Single patent claim with dependent/independent relationships, formal structure, scope. | 🟡 | `specs/phase-2/patent-mcp-spec.md` |
| `PatentApplication` | Full application — title, abstract, drawings, specification, claims (as dependency tree). | 🟡 | `specs/phase-2/patent-mcp-spec.md` |

### ebook-mcp (Not yet implemented)

| Schema | Description | Status | Spec Source |
|--------|-------------|--------|-------------|
| `EpubMetadata` | EPUB-specific metadata — identifier, language, creator, date, rights, direction. | 🟡 | `specs/phase-2/ebook-mcp-spec.md` |
| `MultiFormatOutput` | Simultaneous output to LaTeX, EPUB, PDF, HTML from single source. | 🟡 | `specs/phase-2/ebook-mcp-spec.md` |
| `BookStructure` | Chapter organization, front/back matter, TOC, index, page breaks. | 🟡 | `specs/phase-2/ebook-mcp-spec.md` |

### documentation-mcp (Not yet implemented)

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
| 50 | 🟢 Finalized — implemented in code with tests passing |
| 29 | 🟡 Drafted in spec — not yet implemented (future modules) |
| 0 | 🔴 Not yet drafted |

**Implemented packages:** packages/core (54 types), packages/sdk (4 types), servers/latex-mcp, blog-mcp, grant-mcp, slides-mcp, newsletter-mcp, paper-mcp (all tool arg/result types).

**Next milestone:** Phase 4 (MCP Apps) implementation, or implement next module (lab-notebook-mcp, thesis-mcp, etc.).

---

*Last updated: 2026-02-18 — Phases 0–3 implemented; 50 schemas promoted from 🟡 to 🟢.*
*Maintained by: ZenSci architecture agents | Owner: TresPiesDesign.com / Cruz Morales*
