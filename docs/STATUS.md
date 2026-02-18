# ZenithScience Project Status

**Author:** Health Auditor (Semantic Cluster Analysis)
**Status:** Active & Evolving
**Current Phase:** Specifications Complete / Pre-Code
**Last Updated:** 2026-02-18

---

## 1. Vision & Purpose

> Raw structured thought in → publication-ready artifact out.

ZenSci is the **Knowledge-Work Forge**: a suite of MCP servers that converts structured markdown, outlines, and AI-assisted brainstorming into five publication-ready formats. AIs are native to the target format; humans bring the thinking.

**Core Principles:**
- **Quality over speed** — Solve hard problems (LaTeX, math, proof structures) first; everything else follows
- **Shared infrastructure first** — Build core parsing, citation, validation layers once; reuse across five modules
- **AI as thinking partner** — Augment human reasoning before rendering, not replace human judgment
- **Multi-format coherence** — The same source document should render identically across LaTeX, HTML, email, slides

---

## 2. Current State

The project has completed **strategic and architectural planning**. All identity, IP, org, and module decisions are made. Directory structure is scaffolded. No production code has been written yet.

| Area | Status | Notes |
| :--- | :--- | :--- |
| **Vision & Positioning** | ✅ | Knowledge-Work Forge thesis confirmed |
| **IP & Ownership** | ✅ | TresPiesDesign.com / Cruz Morales owns all IP |
| **GitHub Org & Repo** | ✅ | github.com/TresPies-source/ZenithScience (monorepo) |
| **Product Naming** | ✅ | ZenSci (product), ZenithScience.org (domain) |
| **npm Scope** | ✅ | @zen-sci |
| **Directory Structure** | ✅ | Monorepo with zen-sci/, packages/core, packages/sdk, 5 server directories scaffolded |
| **Module Strategy** | ✅ | 5 confirmed modules; 14+ future modules identified; shipping order confirmed: latex-first |
| **Semantic Architecture** | ✅ | 9 behavioral clusters mapped; gap analysis complete; data flow charted |
| **Data Contracts Backlog** | ✅ | 35+ schemas identified; DocumentRequest schema is first-to-write |
| **Full Spec Suite (All 3 Phases)** | ✅ | 19 specs written: 4 Phase 1, 8 Phase 2, 5 Phase 3, 2 infrastructure. **Audited 2026-02-18.** |
| **Implementation Prompts (Phase 0–3)** | ✅ | Prompts for monorepo, core, sdk, latex-mcp, blog-mcp, grant-mcp, slides-mcp, newsletter-mcp, paper-mcp. **Written 2026-02-18.** |
| **packages/core spec** | ✅ | Full type definitions, 8 classes, API surface, testing strategy. **Reviewed — 4 issues noted.** |
| **packages/sdk spec** | ✅ | ZenSciServer abstract base class, PandocJob/Result, error patterns. **Reviewed — 3 fixes applied, 3 issues noted.** |
| **Core TypeScript Setup** | 🔄 | Ready to scaffold (specs define exact structure) |
| **Core Python Engine** | 🔄 | Ready to begin (pandoc wrapper, TeX engine, citation manager specced) |
| **Production Code** | ⏸️ | Awaiting v0.1 (latex-mcp) kickoff |

**Status Key:**
- ✅ **Complete** | 🔄 **In Progress** | ⏸️ **Paused** | ❌ **Blocked**

---

## 3. Active Workstreams

**Active: Pre-Code Implementation Prep.** All 19 specs complete. Ready for v0.1 kickoff.

Immediate workstream (approved by specs):
1. **TypeScript setup** — Scaffold zen-sci/packages/core and zen-sci/packages/sdk per `specs/infrastructure/` specs
2. **DocumentRequest schema** — Implement the full type definitions from packages-core-spec.md
3. **Python engine skeleton** — Implement ZenSciServer base + PandocWrapper from packages-sdk-spec.md
4. **latex-mcp v0.1 spike** — Implement per `specs/phase-1/latex-mcp-v0.1-spec.md`

---

## 4. Blockers & Dependencies

**No critical blockers.** Five decisions remain open; none block architecture work:

| Decision | Impact | Owner | Recommendation |
| :--- | :--- | :--- | :--- |
| Module shipping order | ✅ Resolved | Cruz | latex-first confirmed 2026-02-18 (flagship hardest problem first) |
| Lab notebook in Phase 1? | Medium | Cruz | Post-v0.4 (identified as strategic differentiator; v1 feature) |
| Thinking Partnership shipping | Low | Cruz | Post-v0.4 (semantic auditor consensus) |
| Trademark/domain availability | Medium | Operations | Check ZenSci, ZenithScience.org, @zen-sci before v0.1 release |
| Open-source license | ✅ Resolved | Legal | Apache 2.0 confirmed 2026-02-18 (patent grant + trademark protection) |

None of these block architecture or local development. They become critical only at release.

---

## 5. Next Steps

**Immediate (Pre-Code — implementation prompts written, ready to commission):**
1. ✅ Module shipping order confirmed: latex-first (2026-02-18)
2. Run trademark/domain availability check (ZenSci, ZenithScience.org, @zen-sci)
3. ✅ Open-source license: Apache 2.0 confirmed (2026-02-18)
4. Verify environment: Node.js LTS, Python 3.11+, pandoc >= 3.0, TeX Live, SymPy

**Implementation Prompts — Commission these sequentially to coding agents:**
- `workspace/2026-02-18_phase-0-implementation-prompt.md` — Monorepo init + packages/core (29 files, 32 requirements)
- `workspace/2026-02-18_phase-1-implementation-prompt.md` — packages/sdk + latex-mcp v0.1 (37 files, 38 requirements)
- `workspace/2026-02-18_phase-2-implementation-prompt.md` — blog-mcp v0.2 + grant-mcp v0.4 (45 files)
- `workspace/2026-02-18_phase-3-implementation-prompt.md` — slides-mcp v0.3 + newsletter-mcp v0.3 + paper-mcp (49 files)

**First Sprint (v0.1 — latex-mcp):**
1. Initialize TypeScript monorepo with shared build tooling
2. Write DocumentRequest schema (shared input contract for all modules)
3. Scaffold zen-sci/packages/core with MD parser, citation manager, validation
4. Scaffold zen-sci/packages/sdk with MCP protocol base class
5. Build latex-mcp v0.1: MCP tool registration + pandoc → LaTeX PDF pipeline
6. Integration test: raw markdown → PDF with embedded math, bibliography, TOC

**Second Sprint (v0.2 — blog-mcp):**
1. Extend core with HTML-specific rendering rules
2. Build blog-mcp: markdown → HTML blog with syntax highlighting, responsive design
3. Validate core abstraction: is blog-mcp's code simpler because core is well-designed?

---

## 6. Architecture Decisions (2026-02-17 & 2026-02-18)

1. **IP Ownership** — TresPiesDesign.com / Cruz Morales own all IP; license TBD
2. **Product Name** — ZenSci (product), ZenithScience.org (domain)
3. **GitHub Org & Repo** — TresPies-source/ZenithScience (monorepo, not "typecraft")
4. **Working Directory** — zen-sci/ (kebab-case) for code; ZenithScience/ root for decision artifacts
5. **npm Scope** — @zen-sci (not @typecraft)
6. **Monorepo Structure** — Shared core library + 5 independent MCP servers (not 5 separate repos)
7. **Full Spec Pass (2026-02-18)** — 19 specs written across all 3 phases + infrastructure in a single parallel pass
8. **SDK Pattern (2026-02-18)** — Composition over inheritance: `createZenSciServer()` factory + standalone utilities, not `ZenSciServer` abstract base class
9. **Strict Enums (2026-02-18)** — All union types (OutputFormat, BibliographyStyle, aspectRatio, SubmissionFileRole) are strict — no `| string` escape hatches
10. **Error Identity Split (2026-02-18)** — `ConversionErrorData` (wire interface) + `ConversionError` (runtime class) with `fromData()` / `toData()` bridge
11. **Module Shipping Order (2026-02-18)** — latex-first confirmed. Sequence: latex-mcp → blog-mcp → slides-mcp + newsletter-mcp (v0.3) → grant-mcp (v0.4). Cruz confirmed.
12. **Open-Source License (2026-02-18)** — Apache 2.0. Patent grant protects institutional users (NIH/NSF); trademark clause protects ZenSci/ZenithScience brand; GPL v2 incompatibility irrelevant (pandoc called as subprocess, not linked library).

---

## 7. Module Roadmap

Five confirmed modules, 14+ future modules identified across 4 phases.

| Module | Output | Phase | Users | Status | Notes |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **latex-mcp** | LaTeX + PDF | v0.1 | Mathematicians, researchers | Pre-code | Flagship; hardest problem |
| **blog-mcp** | HTML blog | v0.2 | Writers, developers | Pre-code | Validates core abstraction |
| **slides-mcp** | Beamer + Reveal.js | v0.3 | Educators, researchers | Pre-code | Validates TeX reuse |
| **newsletter-mcp** | MJML + HTML email | v0.3 | Writers, communicators | Pre-code | Parallel to slides |
| **grant-mcp** | LaTeX / Word | v0.4 | Academics | Pre-code | Phase 2: new module strategy |

**Future modules identified (Phase 2+):**
thesis-mcp, patent-mcp, whitepaper-mcp, poster-mcp, cv-mcp, lab-notebook-mcp, tutorial-mcp, data-report-mcp, and 6 others.

---

## 8. Schemas Backlog

**35+ data contracts identified.** Top priority: `DocumentRequest` (shared input contract for all modules).

Examples of other key schemas identified:
- `DocumentTree`, `ParseError`, `FrontmatterMetadata`, `ValidationResult`
- `ThinkingSession`, `ReasoningTrace`, `ProofStructure` (thinking partnership)
- `MathExpression`, `ProofStep`
- `LayoutRule`, `TOCEntry`
- `BibliographyStyle`, `CrossReference`
- `DocumentConfig`, `ModuleConfig`, `RenderConfig`

**See:** `/sessions/blissful-eloquent-gates/mnt/ZenflowProjects/ZenithScience/schemas/SCHEMAS.md`

---

## 9. Agent Workspace

Artifacts from strategic and architectural planning, organized by function:

| Directory | Contents | Purpose |
| :--- | :--- | :--- |
| **scouts/** | Strategic scout documents (product direction, module strategy) | Define what we build and in what order |
| **research/** | Naming research, GitHub org landscape analysis | Inform identity decisions |
| **architecture/** | Semantic cluster map, data flow diagrams, gap analysis | Show what the system does; identify shared infrastructure |
| **schemas/** | SCHEMAS.md backlog (35+ data contracts) | Define the data contracts between modules |
| **specs/infrastructure/** | packages/core spec, packages/sdk spec | Foundation — all modules depend on these |
| **specs/phase-1/** | latex-mcp, blog-mcp, paper-mcp, lab-notebook-mcp (4 specs) | Phase 1: research/academic core |
| **specs/phase-2/** | slides, newsletter, grant, thesis, whitepaper, patent, ebook, docs (8 specs) | Phase 2: professional ecosystem |
| **specs/phase-3/** | policy-brief, proposal, podcast, pptx, resume (5 specs) | Phase 3: niche/on-demand modules |
| **workspace/** | Implementation prompts (Phase 0–3), audit findings, onboarding prompts | Primary coding agent entry points |

---

## 10. Semantic Architecture (Behavioral Clusters)

The system comprises **9 behavioral clusters** that span the 5 confirmed modules. This reveals where shared infrastructure pays dividends and where modules must diverge.

**Tier 1 — Shared Across All Modules (core library):**
1. **Parsing** — Markdown → AST; frontmatter extraction
2. **Citation & References** — BibTeX/CSL parsing, cross-reference resolution
3. **Validation** — Schema validation, dead-link detection, math syntax checking

**Tier 2 — Format-Specific (module variants):**
4. **Structure** — Document tree (outline, sections, hierarchy)
5. **Conversion** — Target format code generation (LaTeX, HTML, email, etc.)
6. **Math Rendering** — TeX math → target format (PDF, MathML, SVG)
7. **Bibliography Rendering** — CSL styles → target format (TeX bibliography, HTML footnotes, email citations)
8. **Configuration** — Module-specific settings (paper size, CSS, email template, etc.)
9. **Rendering & Output** — Final artifact generation and validation

**Architectural Insight:** Clusters 1, 2, 3 must be shared (duplication cost is high). Clusters 4–9 have module-specific variants but inherit from core abstractions.

---

## 11. Key Metrics & Constraints

**Development Constraints:**
- No production code exists yet; all decisions are architectural
- Team size: Solo (Cruz); auditing/scouting support available
- Shipping target: v0.1 (latex-mcp) within Q1 2026 (tentative)

**Quality Targets:**
- LaTeX output must be publication-ready (journal-submission quality)
- All formats must preserve semantic meaning (math, citations, structure)
- Core abstraction must be sufficiently general for 4 more formats

---

## 12. Risk Assessment

| Risk | Probability | Impact | Mitigation |
| :--- | :--- | :--- | :--- |
| **Core abstraction is too LaTeX-specific** | Medium | High | Start with blog-mcp v0.2 to validate generality early |
| **Python engine subprocess bridge is fragile** | Medium | Medium | Test error handling and timeouts extensively; consider Docker isolation |
| **Math rendering varies too much across formats** | Low | Medium | SymPy + MathML + target-format conversion in core |
| **Trademark/domain not available** | Low | Medium | Check before v0.1 release; have backup names ready |
| **Open-source license choice alienates users** | Low | Low | Choose clearly; document IP ownership upfront |

---

## 13. Success Criteria (v0 Suite)

- ✅ **v0.1 (latex-mcp)** — Markdown with math/citations → publication-ready PDF
- ✅ **v0.2 (blog-mcp)** — Same source → HTML blog (proves core abstraction works)
- ✅ **v0.3 (slides-mcp + newsletter-mcp)** — Same source → Beamer/Reveal.js + email (proves TeX reuse)
- ✅ **v0.4 (grant-mcp)** — Same source → grant proposal (academic format edge case)
- ✅ **Core Quality** — All 5 formats from 1 markdown source; no format-specific rewrites

---

## 14. Communication & Cadence

**Decision Owner:** Cruz Morales (TresPiesDesign.com)

**Open Decisions Waiting For Input:**
1. ✅ Module shipping order — latex-first confirmed (2026-02-18)
2. Lab notebook in Phase 1? (recommend: Phase 2)
3. ✅ License and open-source policy — Apache 2.0 confirmed (2026-02-18)
4. Trademark/domain availability (pre-release check: ZenSci, ZenithScience.org, @zen-sci)

**Next Review:** Post-v0.1 completion (estimated Q2 2026)

---

## 15. Specification Audit Summary (2026-02-18)

**Auditor:** Claude (Opus-level audit, Cruz Morales session)
**Scope:** All 19 specs + STATUS.md
**Grounded in:** Context7 live docs for MCP TypeScript SDK v2, ebooklib, MJML, Zod
**Full report:** `workspace/2026-02-18_audit-findings.md`

**Result: 17 issues found (3 Critical, 5 High, 5 Medium, 4 Advisory). No spec requires rewrite.**

### Fixes Applied During Audit

| Fix | Specs Affected |
| :--- | :--- |
| MCP SDK imports corrected (`@anthropic-ai/sdk` → `@modelcontextprotocol/sdk`) | latex-mcp, blog-mcp, paper-mcp, lab-notebook-mcp |
| Python version standardized to >= 3.11 | SDK spec, latex-mcp, blog-mcp, lab-notebook-mcp |
| Pandoc version standardized to >= 3.0 | SDK spec, STATUS.md |
| Typo `convertViaP andoc` fixed | SDK spec |
| Python code bug (`append` with `end=`) fixed | patent-mcp |
| KindleGen replaced with Calibre `ebook-convert` | ebook-mcp |
| `@anthropic-ai/sdk` removed from package.json deps | paper-mcp, lab-notebook-mcp, blog-mcp |

### Open Issues Requiring Decisions (Pre-v0.1)

1. **ZenSciServer adoption:** Defined in SDK spec but no module uses it. Decide: extend everywhere, or deprecate?
2. **MCP SDK v2 migration:** SDK spec should wrap `McpServer` + Zod internally. Module specs can keep current patterns if base class handles protocol.
3. **`OutputFormat` union type:** Core spec needs complete format string enum (24 values across all modules).
4. **`ConversionError` naming conflict:** Rename interface to `ConversionErrorData`.
5. **Tool naming convention:** Phase 3 specs use camelCase; Phase 1/2 use snake_case. Standardize to snake_case (MCP convention).

### Audit Notes Added

All 19 spec files now include an "Audit Notes (2026-02-18)" section at the end documenting per-spec findings, fixes applied, and remaining issues.

### Follow-Up Audit: New Type Definitions (2026-02-18)

**Scope:** 7 new type groups in packages-core-spec.md (Section 3.4) and grant-mcp-spec.md (Section 3.6).

| Fix | Spec |
| :--- | :--- |
| Removed invalid OG type `'blog'` from `OpenGraphMetadata` | packages-core-spec |
| `SchemaOrgData.author` now supports `Organization` | packages-core-spec |
| `AccessibilityReport.validatedAt` standardized to ISO 8601 string | packages-core-spec |
| `GrantSubmissionFormat`: `'nsf-fastlane'` → `'nsf-research-gov'` (FastLane retired 2023) | grant-mcp-spec |
| `import pandoc` → `import pypandoc`; dead import removed; call signature fixed | grant-mcp-spec |
| Pandoc version `≥2.18` → `>= 3.0` | grant-mcp-spec |

**4 additional issues noted** in audit notes (FormatConstraint redundancy, ConversionRule string-typing, SEOMetadata alignment, SubmissionFile.role typing). All 4 resolved in decision propagation below.

### Key Architecture Decisions (Cruz, 2026-02-18)

All open decisions from the audit resolved. Decisions recorded at source in `workspace/2026-02-18_audit-findings.md` and propagated to all dependent specs.

| # | Decision | Cluster | Route | Specs Modified |
| :--- | :--- | :--- | :--- | :--- |
| D1 | **Drop ZenSciServer, use native McpServer** — Composition over inheritance. `createZenSciServer()` factory replaces abstract base class. | SDK Contract | B | packages-sdk-spec (rewritten), all module audit notes updated |
| D2 | **Strict enums everywhere** — No `\| string` escape hatches. All unions coordinated from authoritative source. | Naming & Shape | A | packages-core-spec (BibliographyStyle, FormatConstraint.aspectRatio), grant-mcp (SubmissionFile.role) |
| D3 | **Split ConversionError identity** — Interface → `ConversionErrorData` (wire). Class → `ConversionError` (runtime). `fromData()` / `toData()` bridge. | Error Identity | A | packages-core-spec (interface, class, all references), packages-sdk-spec (MCPErrorHandler) |
| D4 | **Align blog-mcp SEOMetadata now** — Extends `WebMetadataSchema` from core. Flat fields → nested OG/Twitter sub-objects. | Cross-Spec | A | blog-mcp-v0.2-spec (SEOMetadata rewritten) |

### Remaining Deferred Items

| Item | Status | Target |
| :--- | :--- | :--- |
| `ConversionRule.transform` string typing | Acceptable for v0.1; document registry contract at implementation | v0.1 implementation |
| `DocumentVersion` / `DocumentVersionHistory` | Post-v0.1 scope; no module should import these yet | v0.2+ |
| `OutputFormat` union completeness | Current union covers all 19 specs; verify at implementation | v0.1 implementation |
| Server file path standardization (HIGH-2) | Pick `zen-sci/servers/{module}/` pattern; update all specs | v0.1 kickoff |

---

## Notes for Future Audits

- **Re-audit after v0.1 completion:** Verify core abstraction held; identify any tight coupling that should be refactored
- **Check semantic clusters against implementation:** Do the 9 clusters map to actual code boundaries?
- **Quarterly thinking-partnership review:** Is ZenSci meeting the "augment human reasoning" thesis?
- **Forward compatibility check:** When Phase 2 modules (thesis-mcp, patent-mcp) are proposed, re-run cluster analysis to ensure they fit the architecture
- **All major open decisions resolved (2026-02-18):** 4 clusters decided and propagated. Only deferred items remain (see table above)

