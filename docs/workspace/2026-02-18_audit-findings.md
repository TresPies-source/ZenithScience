# ZenSci Specification Audit — 2026-02-18

**Auditor:** Claude (Opus-level audit, Cruz Morales session)
**Scope:** All 19 specs across infrastructure (2), Phase 1 (4), Phase 2 (8), Phase 3 (5)
**Grounded In:** Context7 live docs for MCP TypeScript SDK, ebooklib, MJML, Zod
**Date:** 2026-02-18

---

## Executive Summary

The 19-spec suite is architecturally coherent and strategically well-positioned. The shared-core monorepo thesis, 9-cluster behavioral architecture, and phased shipping strategy are sound. However, the audit surfaced **17 actionable issues** across 4 categories, ranging from critical (wrong SDK imports that would prevent compilation) to advisory (naming inconsistencies that would create confusion during implementation).

**Severity breakdown:** 3 Critical, 5 High, 5 Medium, 4 Advisory.

No spec requires a rewrite. All issues are targeted fixes.

---

## A. Completeness Audit

### Phase 1 + Phase 2 Specs (12 specs): STRONG

All 12 specs contain: Vision, Goals/Non-Goals, MCP Tool Definitions with JSON schemas, TypeScript implementation code, Python processing engine code, week-by-week implementation plans, risk tables, and appendices. The level of detail is sufficient for autonomous implementation.

### Infrastructure Specs (2 specs): STRONG with minor issues

Both `packages-core-spec.md` and `packages-sdk-spec.md` are comprehensive. Issues noted below (naming conflict, typo, version drift).

### Phase 3 Specs (5 specs): APPROPRIATE for phase

Phase 3 specs are intentionally lighter ("simplified architecture") which is correct for trigger-based shipping. They include: Vision, Goals/Non-Goals, Strategic Fit Assessment (unique to Phase 3 — good discipline), MCP tool definitions (simplified), TypeScript schemas, implementation estimates (track-level, not week-by-week), risk tables, and open questions. They lack full TypeScript server code and Python engine code, which is appropriate since these won't be built until market triggers fire.

### Completeness Gap: `OutputFormat` Union Type

The `packages-core-spec.md` defines an `OutputFormat` type but does not enumerate all format strings used across the 19 specs. The complete set (gathered from all specs) is:

```
'latex-pdf' | 'html-blog' | 'beamer-pdf' | 'revealjs-html' |
'newsletter-html' | 'newsletter-mjml' | 'grant-pdf' | 'grant-docx' |
'thesis-pdf' | 'whitepaper-pdf' | 'patent-pdf' | 'epub' | 'mobi' | 'ebook-pdf' |
'documentation-sphinx' | 'documentation-mkdocs' |
'policy-brief' | 'proposal' | 'podcast-notes' | 'pptx' | 'google-slides' |
'resume' | 'cv' | 'paper-pdf'
```

**Action:** Add complete union type to `packages-core-spec.md`.

---

## B. Cross-Spec Consistency Audit

### CRITICAL-1: Phase 1 MCP SDK Imports Are Wrong

**Affected specs:** `latex-mcp-v0.1-spec.md`, `blog-mcp-v0.2-spec.md`

These two specs import from `@anthropic-ai/sdk`, which is Anthropic's Chat API client — not the MCP protocol SDK. This would fail at compile time.

```typescript
// WRONG (current in specs)
import { Server } from "@anthropic-ai/sdk/lib/resources/messages/tools";
import { ListToolsRequestSchema } from "@anthropic-ai/sdk/resources/messages/tools";

// CORRECT (matches Phase 2 specs)
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
```

Additionally, `paper-mcp-spec.md` and `lab-notebook-mcp-spec.md` list `@anthropic-ai/sdk` in their `package.json` dependencies. This should be `@modelcontextprotocol/sdk`.

**Action:** Fix imports in all 4 Phase 1 specs.

### CRITICAL-2: ZenSciServer Base Class Defined But Never Used

`packages-sdk-spec.md` defines a `ZenSciServer` abstract base class as the core abstraction for all module servers. However:

- `paper-mcp` and `lab-notebook-mcp` extend `MCPServer` from `@zen-sci/sdk` (close but different name)
- `latex-mcp` and `blog-mcp` don't extend any base class
- All 8 Phase 2 specs instantiate `Server` directly from the MCP SDK, bypassing the ZenSci abstraction entirely

This means the SDK spec's core design decision (shared base class) is orphaned.

**Action:** Standardize. Two options:
1. All module specs extend `ZenSciServer` from `@zen-sci/sdk` (preferred — matches the architecture)
2. Deprecate `ZenSciServer` and use raw `Server` everywhere (simpler but loses shared validation/error handling)

Recommendation: Option 1, since `ZenSciServer` provides `validateRequest()`, `handleError()`, and Python engine bridging that every module needs.

### CRITICAL-3: MCP SDK v2 API Pattern Outdated Across All Specs

All specs use the low-level pattern:
```typescript
server.setRequestHandler(ListToolsRequestSchema, async () => ({ tools: [...] }));
server.setRequestHandler(CallToolRequestSchema, async (request) => { ... });
```

The current MCP TypeScript SDK v2 (confirmed via Context7) recommends the high-level pattern:
```typescript
import { McpServer } from '@modelcontextprotocol/server';
import { z } from 'zod';

const server = new McpServer({ name: 'my-server', version: '1.0.0' });
server.registerTool('convert_to_latex', {
  description: 'Convert markdown to LaTeX PDF',
  inputSchema: z.object({ source: z.string(), metadata: z.object({...}) }),
}, async (args) => { ... });
```

**Impact:** The low-level pattern still works but is more verbose and doesn't get Zod validation for free. Since ZenSci uses `ZenSciServer` as an abstraction layer, this can be handled in the base class — module specs don't need to change their tool registration patterns if the SDK spec handles the MCP protocol internals.

**Action:** Update `packages-sdk-spec.md` to use `McpServer` internally. Module specs can continue using `this.registerTool()` via the `ZenSciServer` abstraction. Add a note in each module spec acknowledging the MCP SDK v2 pattern.

### HIGH-1: Tool Naming Convention Split

Phase 1 and Phase 2 specs use `snake_case` for tool names: `convert_to_slides`, `validate_whitepaper`, `convert_to_grant`.

Phase 3 specs use `camelCase`: `convertPolicyBrief`, `convertProposal`, `convertToSlides`.

MCP convention (per SDK docs) is `snake_case`.

**Action:** Standardize Phase 3 tool names to `snake_case`.

### HIGH-2: Server File Path Inconsistency

Two patterns exist:
- `zen-sci/packages/sdk/src/servers/{module}/index.ts` (slides-mcp, others)
- `zen-sci/servers/{module}/` (STATUS.md, some specs' Python paths)

The monorepo structure in STATUS.md says `5 server directories scaffolded` under the root, implying `zen-sci/servers/`. But spec TypeScript code paths put servers under `packages/sdk/src/servers/`.

**Action:** Pick one. Recommendation: `zen-sci/servers/{module}/` for server entry points (keeps packages/sdk clean as a library). Update all specs to match.

### HIGH-3: Python Version Inconsistency

- `packages-sdk-spec.md`: Python 3.8+
- `latex-mcp`, `blog-mcp`, `lab-notebook-mcp`: Python >= 3.9
- `STATUS.md`: Python 3.11+

**Action:** Standardize to Python 3.11+ (STATUS.md is correct — 3.8 and 3.9 are EOL or nearly so).

### HIGH-4: MCP SDK Version Outdated in packages-sdk-spec.md

The spec lists `"@modelcontextprotocol/sdk": "^0.1.0"`. The current SDK is v2.x.

**Action:** Update to `"@modelcontextprotocol/sdk": "^2.0.0"` or use the new package name `@modelcontextprotocol/server`.

### HIGH-5: `ConversionError` Naming Conflict in packages-core-spec.md

`ConversionError` is defined as both an `interface` (for error shape) and a `class` (for throwing). These should be:
- `ConversionErrorData` (interface — the serializable shape)
- `ConversionError` (class extending `Error` — the throwable)

**Action:** Rename the interface to `ConversionErrorData`.

### MEDIUM-1: Typo in packages-sdk-spec.md

Line 750: `convertViaP andoc` (space in method name). Also line 818.

**Action:** Fix to `convertViaPandoc`.

### MEDIUM-2: `BibliographyStyle` Enum Incomplete

`latex-mcp` uses styles `'nature'` and `'arxiv'` that aren't in the core spec's `BibliographyStyle` enum.

**Action:** Add all style values used across specs to the core enum.

### MEDIUM-3: KindleGen Deprecated in ebook-mcp

Amazon discontinued KindleGen in 2020. The spec references it 15+ times as the MOBI generation tool.

**Action:** Replace KindleGen references with Calibre's `ebook-convert` command: `ebook-convert input.epub output.mobi`. Update the dependency list, code examples, and risk table.

### MEDIUM-4: Python Code Bug in patent-mcp

Line 461: `lines.append(f"\\textbf{{{num}.}} ", end='')` — `list.append()` does not accept an `end` keyword argument (that's `print()`).

**Action:** Fix to `lines.append(f"\\textbf{{{num}.}} ")`.

### MEDIUM-5: Pandoc Version Inconsistency

STATUS says "2.19+", latex-mcp says ">= 2.19", but Pandoc is currently at 3.x (3.6+ as of early 2026).

**Action:** Standardize to "pandoc >= 3.0" across all specs and STATUS.md (aligns with current releases).

### ADVISORY-1: `presentation-slides-mcp` vs `slides-mcp` Overlap

`slides-mcp` (Phase 2) generates Beamer PDF + Reveal.js HTML for academic presentations.
`presentation-slides-mcp` (Phase 3) generates PPTX + Google Slides for business presentations.

The names don't communicate this distinction clearly. If both ship, users will be confused.

**Action:** Consider renaming:
- `slides-mcp` → `academic-slides-mcp` (or keep as-is since it's the flagship)
- `presentation-slides-mcp` → `pptx-mcp` (clearer about output format)

### ADVISORY-2: Phase 3 `moduleOptions` vs Phase 2 `metadata`

Phase 3 specs put module-specific options in a `moduleOptions` field on the request. Phase 2 specs put options directly in `metadata`. The core `DocumentRequest` schema should define where module options live.

**Action:** Standardize on `moduleOptions` (cleaner separation from document metadata).

### ADVISORY-3: `pypandoc` vs `pandoc` Import

Several Python code blocks use `import pandoc` with `pandoc.convert_text()`. The actual Python library is `pypandoc` with `pypandoc.convert_text()`.

**Action:** Fix Python imports to use `pypandoc` in all specs.

### ADVISORY-4: Thinking Partnership Thesis Underrepresented

The project vision positions ZenSci as a "thinking partner, not just converter." Currently, only `lab-notebook-mcp` implements thinking partnership features (hypothesis tracking, reasoning traces, proof structures). The other 18 specs are purely conversion-focused.

This is acceptable for v0.x (conversion must work first), but the thesis should be acknowledged in the architecture. The `packages-core-spec.md` should include a `ThinkingPartnership` namespace or interface that modules can optionally implement.

**Action:** Add a note to `packages-core-spec.md` about the thinking partnership extension point. No code changes needed for v0.1.

---

## C. Technical Accuracy Audit (Context7-Grounded)

### MCP TypeScript SDK (verified via Context7)

**Finding:** The current SDK exports `McpServer` from `@modelcontextprotocol/server` (not `Server` from `@modelcontextprotocol/sdk/server/index.js`). Tool registration uses `server.registerTool()` with Zod schemas (not `setRequestHandler` with raw JSON Schema). `StdioServerTransport` is imported from `@modelcontextprotocol/server` (not a separate subpath).

**Impact on specs:** The low-level `Server` class is still available and functional. Since ZenSci wraps this in `ZenSciServer`, the module specs' code patterns are acceptable IF `ZenSciServer` handles the MCP protocol internally. The SDK spec needs updating to use the current API.

### ebooklib (verified via Context7)

**Finding:** The API confirmed: `epub.EpubBook()`, `book.set_identifier()`, `book.set_title()`, `book.set_language()`, `book.add_author()`, `epub.EpubHtml(title=..., file_name=..., lang=...)`, `chapter.content = '<html>...'`, `book.add_item()`, `book.toc = (...)`, `book.spine = ['nav', c1, c2]`, `epub.write_epub('file.epub', book)`, `book.set_cover('cover.png', content, create_page=True)`.

**Impact on ebook-mcp spec:** The Python code in the spec uses a reasonable approximation of the ebooklib API. Minor adjustments needed (e.g., `book.set_cover()` for cover image handling).

### MJML (verified via Context7)

**Finding:** MJML structure confirmed: `<mjml><mj-head>...</mj-head><mj-body><mj-section><mj-column>...</mj-column></mj-section></mj-body></mjml>`. Components include `mj-text`, `mj-button`, `mj-image`, `mj-table`, `mj-section`, `mj-column`, `mj-preview`, `mj-include`, `mj-raw`. Newsletter-mcp's approach of using `mjml-cli` as a subprocess is the correct integration pattern.

**Impact on newsletter-mcp spec:** The MJML integration approach is sound. No technical corrections needed.

---

## D. Strategic Coherence Audit

### Module Phasing: COHERENT

Phase 1 (latex, blog, paper, lab-notebook) correctly targets the hardest problems first (LaTeX/math rendering) and validates the core abstraction with an orthogonal format (blog/HTML). Phase 2 extends to professional formats. Phase 3 is appropriately gated behind market triggers.

### "Thinking Partner" Thesis: PRESENT BUT THIN

The thesis is well-articulated in STATUS.md and the module strategy scout. `lab-notebook-mcp` is the only spec that operationalizes it with reasoning traces, hypothesis tracking, and proof structures. This is a strategic differentiator that should be more visible in the architecture (see ADVISORY-4).

### Phase 3 Strategic Fit Assessments: EXCELLENT

All 5 Phase 3 specs include a "Why Phase 3" section with explicit market trigger conditions and "risk of NOT shipping" analysis. This is a mature product discipline that prevents premature investment. Recommend maintaining this pattern for all future module proposals.

### Naming & Positioning: ONE CONCERN

The `presentation-slides-mcp` / `slides-mcp` overlap (see ADVISORY-1) is the only positioning issue. All other modules have clear, non-overlapping positioning.

---

## Issue Tracker Summary

| # | Severity | Issue | Affected Specs | Status |
|---|----------|-------|----------------|--------|
| CRITICAL-1 | Critical | Phase 1 MCP SDK imports wrong (`@anthropic-ai/sdk`) | latex, blog, paper, lab-notebook | Fixed |
| CRITICAL-2 | Critical | ZenSciServer defined but never used | All module specs | Noted (Audit Notes added) |
| CRITICAL-3 | Critical | MCP SDK v2 API pattern outdated | All specs | Noted (Audit Notes added) |
| HIGH-1 | High | Tool naming split (snake_case vs camelCase) | Phase 3 specs | Fixed |
| HIGH-2 | High | Server file path inconsistency | Multiple | Noted |
| HIGH-3 | High | Python version inconsistency (3.8/3.9/3.11) | SDK, Phase 1 | Fixed |
| HIGH-4 | High | MCP SDK version `^0.1.0` outdated | SDK spec | Fixed |
| HIGH-5 | High | ConversionError naming conflict | core spec | Noted |
| MEDIUM-1 | Medium | Typo `convertViaP andoc` | SDK spec | Fixed |
| MEDIUM-2 | Medium | BibliographyStyle enum incomplete | core spec | Noted |
| MEDIUM-3 | Medium | KindleGen deprecated | ebook-mcp | Fixed |
| MEDIUM-4 | Medium | Python code bug (`append` with `end=`) | patent-mcp | Fixed |
| MEDIUM-5 | Medium | Pandoc version inconsistency | Multiple | Fixed |
| ADVISORY-1 | Advisory | slides-mcp vs presentation-slides-mcp overlap | slides, presentation-slides | Noted |
| ADVISORY-2 | Advisory | moduleOptions vs metadata field location | Phase 2 vs Phase 3 | Noted |
| ADVISORY-3 | Advisory | `pypandoc` vs `pandoc` import | Multiple Python blocks | Noted |
| ADVISORY-4 | Advisory | Thinking partnership thesis underrepresented | Core architecture | Noted |

---

## Recommended Next Steps

1. **Immediate (pre-v0.1):** Fix CRITICAL-1 (already done in this audit). Resolve CRITICAL-2 by deciding on ZenSciServer usage pattern. Update SDK spec for MCP SDK v2.
2. **During v0.1 sprint:** Standardize server file paths (HIGH-2). Fix ConversionError naming (HIGH-5). Complete OutputFormat union.
3. **Before Phase 2:** Standardize `moduleOptions` field (ADVISORY-2). Fix pypandoc imports (ADVISORY-3).
4. **Before Phase 3:** Resolve slides naming overlap (ADVISORY-1). Ensure tool naming is snake_case.
5. **Ongoing:** Expand thinking partnership interface in core as lab-notebook-mcp matures.

---

## Follow-Up Audit: New Type Definitions (2026-02-18)

**Scope:** 7 new type groups added to packages-core-spec.md (Section 3.4) and grant-mcp-spec.md (Section 3.6).

### Fixes Applied

| # | File | Fix | Severity |
|---|------|-----|----------|
| F1 | packages-core-spec | Removed `'blog'` from `OpenGraphMetadata.type` — not a valid OG type | Medium |
| F2 | packages-core-spec | `SchemaOrgData.author` now supports `'Person' \| 'Organization'` | Medium |
| F3 | packages-core-spec | `AccessibilityReport.validatedAt` changed from `Date` to `string` (ISO 8601) — consistent with all other timestamp fields | Medium |
| F4 | grant-mcp-spec | `GrantSubmissionFormat.platform`: `'nsf-fastlane'` → `'nsf-research-gov'` (FastLane retired 2023) | High |
| F5 | grant-mcp-spec | `import pandoc` → `import pypandoc`; removed dead `import yaml` | Medium |
| F6 | grant-mcp-spec | `pypandoc.convert_text()` call signature corrected (added target format arg) | Medium |
| F7 | grant-mcp-spec | Pandoc version `≥2.18` → `>= 3.0` | Medium |

### Open Issues (Require Decisions)

| # | File | Issue | Severity |
|---|------|-------|----------|
| N1 | packages-core-spec | `FormatConstraint.aspectRatio` has redundant union + string | Medium |
| N2 | packages-core-spec | `ConversionRule.transform` is stringly-typed (function-by-name) | Medium |
| N3 | packages-core-spec | `blog-mcp.SEOMetadata` uses flat snake_case, doesn't structurally extend `WebMetadataSchema` | Low |
| N4 | grant-mcp-spec | `SubmissionFile.role` is bare `string`, should be union of known roles | Medium |

### Type Quality Assessment

All 7 type groups are well-structured: clear JSDoc comments, proper optional fields, and sensible defaults. The `ImageProcessingOptions.fit` values correctly match the Sharp library API. The `AccessibilityViolation.severity` levels align with axe-core. The `DocumentVersion` workflow tags cover the standard academic review lifecycle. Overall quality is high — these are ready for implementation after the noted decisions are resolved.

---

## Architectural Decisions (Cruz, 2026-02-18)

### Decision 1: SDK Contract — Drop ZenSciServer, Use Native McpServer (Cluster 1, Route B)

- **Decision:** Deprecate the `ZenSciServer` abstract base class. All module servers instantiate `McpServer` from `@modelcontextprotocol/sdk` directly and compose shared utilities (PythonEngine, PandocWrapper, MCPErrorHandler, TempFileManager, Logger) as standalone imports.
- **Reasoning:** ZenSciServer was already orphaned — no module spec used it. Composition over inheritance avoids coupling and lets modules opt into exactly the utilities they need. A `createZenSciServer()` factory function provides the shared setup pattern without enforcement.
- **Tradeoff acknowledged:** Cross-cutting concerns (logging, telemetry, error normalization) become opt-in rather than structural. Discipline replaces enforcement.
- **Implications:** packages-sdk-spec.md rewritten from abstract-class to composition pattern. All module specs already use direct instantiation, so no module-level changes needed. Subclass example updated.

### Decision 2: Naming & Shape — Strict Enums Everywhere, Coordinated from Core (Cluster 2, Route A)

- **Decision:** All union types (`OutputFormat`, `BibliographyStyle`, `FormatConstraint.aspectRatio`, `SubmissionFile.role`) are strict enums defined at their authoritative source. No `| string` escape hatches. Adding a new value requires a deliberate change to the type definition.
- **Reasoning:** With 17 modules consuming shared types, strict enums catch mismatches at compile time. The cost (core release for new values) is a feature for a monorepo with coordinated evolution.
- **Implications:** `FormatConstraint.aspectRatio` drops `| string`, keeps only known literals. `BibliographyStyle` gains `'nature'` and `'arxiv'`. `SubmissionFile.role` becomes a typed union. Phase 3 tool names already fixed to snake_case in prior audit.

### Decision 3: ConversionError Identity — Split by Purpose (Cluster 3, Route A)

- **Decision:** Rename the `ConversionError` interface to `ConversionErrorData` (serializable wire format). Keep the `ConversionError` class extending `Error` for runtime use. Add a static `ConversionError.fromData(data: ConversionErrorData)` constructor to link them.
- **Reasoning:** Clean separation of concerns. `ConversionErrorData` travels over JSON; `ConversionError` provides stack traces and catch blocks. No naming collision, no ambiguity.
- **Implications:** packages-core-spec.md interface renamed. All references to the interface shape (ConversionPipeline.result.error, PipelineStage.error, etc.) updated to `ConversionErrorData`. Class definition updated with `fromData()` factory.

### Decision 4: Cross-Spec Alignment — Align blog-mcp SEOMetadata Now (Cluster 4, Route A)

- **Decision:** Refactor blog-mcp's flat `SEOMetadata` interface to extend `WebMetadataSchema` from packages/core. The flat `og_title`/`og_description`/`twitter_card` fields are replaced with nested `og?: OpenGraphMetadata` and `twitter?: TwitterCardMetadata` sub-objects.
- **Reasoning:** Every module emitting web metadata should speak the same structural language from day one. Since no code exists yet, the cost is only spec editing time.
- **Implications:** blog-mcp-v0.2-spec.md SEOMetadata rewritten. packages-core-spec.md JSDoc on WebMetadataSchema confirmed accurate.

---

**End of Audit**
