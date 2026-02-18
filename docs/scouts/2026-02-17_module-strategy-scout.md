# Module Strategy Scout — ZenSci

**Date:** 2026-02-17
**Tensions scouted:** Module shipping order + Ecosystem expansion
**Agent:** Strategic Scout

---

## Tension 1: Module Shipping Order

**The Tension:**
After `latex-mcp` ships as v0.1, what ships next? The choice determines whether we stress-test the architecture with a similar problem (slides, which shares the TeX engine) or validate the core abstraction with a different format (blog, which uses HTML). This tension reflects a deeper architectural question: Is our abstraction robust enough to handle any format, or only TeX-like formats?

---

### Routes

#### Route A: Adjacent-to-LaTeX First — `slides-mcp` (Validate TeX Reuse)

**Thesis:** Slides share the core TeX infrastructure (Beamer engine, math rendering, bibliography). Shipping slides second validates the assumption that "different output format = reuse core engine."

| Dimension | Assessment |
|-----------|-----------|
| **Risk** | Low-to-medium. Heavy code reuse from latex-mcp, but Beamer-specific styling is new. Fast feedback on TeX pipeline reuse. If it fails, you debug the core while it's fresh. |
| **Impact** | Deep confidence in the TeX pathway. Sets the pattern for all future LaTeX-adjacent modules (papers, theses). Proves `packages/core` TeX pipeline is production-ready. |
| **Duration** | 8-10 weeks |
| **Optimizes For** | Architecture confidence, rapid feedback on shared infrastructure, proof-of-concept for "one pipeline, many TeX formats" |
| **Sacrifices** | Market diversity (no HTML/non-TeX output yet). Blog/newsletter users still waiting. Psychological: feels like "more of the same" rather than expansion. |

---

#### Route B: Orthogonal-to-LaTeX First — `blog-mcp` (Stress-Test the Core Abstraction)

**Thesis:** Blog uses HTML, Markdown, frontmatter — zero TeX. Shipping blog second proves the core abstraction (`DocumentRequest → DocumentResponse` pipeline) is truly format-agnostic, not TeX-specific. This validates the entire architectural philosophy.

| Dimension | Assessment |
|-----------|-----------|
| **Risk** | Medium. Different infrastructure (no TeX engine), completely new rendering path. If the shared core abstraction is weak, it breaks here. But early discovery is valuable. |
| **Impact** | Validates architectural robustness. If blog works, you know ANY future format will work. Forces early investment in true abstraction. Captures first non-academic user cohort. |
| **Duration** | 10-12 weeks (new rendering path, no TeX code reuse, but simpler output than academic LaTeX) |
| **Optimizes For** | Architectural robustness, proving the "knowledge-work" positioning works across domains, user diversity, early validation of core assumptions |
| **Sacrifices** | Slower time to shipping (more from scratch). No fast wins from TeX code reuse. Psychological: feels riskier because it's orthogonal. |

---

#### Route C: Hybrid Parallel Tracks — Both `blog-mcp` and `slides-mcp` (Two-Track Execution)

**Thesis:** Ship both in parallel, staggered: Blog in weeks 8-10, slides in weeks 12-14. Two teams work independently; shared core is tested simultaneously by both. Fastest path to format diversity.

| Dimension | Assessment |
|-----------|-----------|
| **Risk** | Medium-to-high coordination overhead. Two modules competing for attention and code review bandwidth. Core abstraction tested under time pressure. Root-cause debugging harder if both have issues. Requires clear API contracts upfront. |
| **Impact** | Fastest time-to-market diversity (both formats shipping within 4 weeks). Both test core assumptions simultaneously — real stress test. Market appeal expanded fast. Early signal on whether abstraction is sound. |
| **Duration** | 12-14 weeks total (parallel execution compresses timeline vs. sequential, but not as fast as doing one alone) |
| **Optimizes For** | Speed, parallelization, simultaneous architecture validation, market diversity |
| **Sacrifices** | Focused debugging. Requires larger team or split focus. Cannot pivot mid-stream. |

---

#### Route D: Newsletter-First — `newsletter-mcp` (Fast Win / High-ROI Format)

**Thesis:** Newsletter is simpler than blog or slides (constrained format, MJML template-driven). Ship before anything else besides LaTeX. Email is proven high-ROI for writers/marketers. Fastest time to working product outside academic.

| Dimension | Assessment |
|-----------|-----------|
| **Risk** | Low. Most constrained format (MJML templates). Fewest degrees of freedom. But doesn't test core assumptions — similar-enough to blog that you learn nothing new. |
| **Impact** | Ships fastest (6-8 weeks). Captures writers/marketers early. Proves market fit outside academia. But doesn't stress-test architecture or validate abstraction. Quick wins, but shallow learning. |
| **Duration** | 6-8 weeks |
| **Optimizes For** | Speed, quick wins, non-academic user capture, template reuse |
| **Sacrifices** | Doesn't validate core abstraction or TeX reuse. Email rendering quirks distract from core work. Feels like side quest, not strategic. |

---

#### Route E: Grant-First — `grant-mcp` (Revenue Anchor + Template Complexity)

**Thesis:** Grants are LaTeX + Word (.docx) + structured templates + funder-specific logic. Close to LaTeX (shares TeX pipeline) but adds multi-format output and compliance rules. Shipping grants second proves the architecture handles format families and domain-specific templates.

| Dimension | Assessment |
|-----------|-----------|
| **Risk** | Medium-to-high. Complex domain (funder rules, compliance, institutional templates). Long feedback loops with actual grant writers. Multi-format output (TeX + .docx) adds complexity. |
| **Impact** | Direct revenue opportunity. Anchors academic use case firmly. Forces early investment in template abstraction (reusable across paper/thesis/grant). High-value users (researchers with budgets). |
| **Duration** | 12-14 weeks (TeX reuse + significant domain logic + multi-format) |
| **Optimizes For** | Revenue, domain anchor, template abstraction maturity, institutional partnerships |
| **Sacrifices** | Slower to ship than blog/slides/newsletter. Domain complexity delays feedback. Doesn't as strongly validate core abstraction as blog would. |

---

## What Resonates / What to Conclude

**Recommended shipping order: Route B → Route A → Route C (reformulated)**

Specifically: `latex-mcp` v0.1 → `blog-mcp` (weeks 8-12) → `slides-mcp` (weeks 12-16) → `newsletter-mcp` (weeks 16-20).

**Rationale:**

1. **Blog first validates the architecture.** If the core abstraction (`DocumentRequest → DocumentResponse` pipeline) is weak, blog will fail early. Better to discover this when you have room to pivot. Route B is the riskier, more informative choice.

2. **Slides second proves TeX reuse when you're confident.** Once blog ships, you have proof the core is sound. Slides validates the second pillar: TeX pipeline reuse. Route A becomes low-risk validation.

3. **Newsletter is the easy follow-up.** HTML + templates from blog can power newsletter. Route D's speed advantage disappears if you ship blog first anyway; newsletter becomes the natural third module using existing infrastructure.

4. **Why not Route C (parallel)?** Parallel execution is valuable, but only AFTER you have core patterns proven. Running blog + slides simultaneously is expensive before you know the abstraction works. Serialize first, parallelize later.

5. **Why not Route E (grant-first)?** Grants are valuable long-term, but heavy on domain logic and complex multi-format output. Ship after you have confidence in core abstraction (blog + slides). Grants become Phase 2.

**The strategic insight:** The order should be determined by **architectural risk**, not by user value or time-to-market. Blog-first is the scout's move: it reveals whether the entire philosophy is sound.

---

## Tension 2: Ecosystem Expansion

**The Tension:**
The current four modules (LaTeX, Blog/Newsletter, Grant, Slides) are confirmed. But the vision is a full knowledge-work suite. What OTHER output formats or document types should the ecosystem eventually support? This tension reflects the question: Is ZenSci an academic tool, a general publishing forge, or something else entirely?

---

### New Module Candidates

| Module | Output Format | Primary User / Use Case | Strategic Fit (1-5) | Risk Level | Phase | Key Reuse |
|--------|---------------|------------------------|---------------------|-----------|-------|-----------|
| `paper-mcp` | Academic paper (arXiv, IEEE, ACM LaTeX templates) | Researchers submitting to peer-reviewed venues | 5 | Low | 1 | latex-mcp TeX pipeline, citation engine |
| `lab-notebook-mcp` | Markdown + PDF + structured metadata (reproducible science) | Researchers, scientists, experimental practitioners | 5 | Low-Medium | 1 | Core abstraction, thinking-partner positioning |
| `thesis-mcp` | University thesis/dissertation (ProQuest, institutional templates) | PhD students, researchers | 4 | Low-Medium | 1 | latex-mcp TeX pipeline, institutional templates |
| `book-mcp` | LaTeX + PDF (print-on-demand) + EPUB + Markdown | Authors, academic authors, publishers | 4 | Medium | 1 | latex-mcp TeX, EPUB generation, multi-format output |
| `whitepaper-mcp` | LaTeX + PDF + HTML (formal whitepaper) | Researchers, think tanks, policy advocates | 4 | Low-Medium | 1 | latex-mcp TeX pipeline, formal structure |
| `patent-mcp` | Patent application (USPTO/WIPO templates) + claims structure | Engineers, researchers, IP lawyers | 4 | Medium-High | 2 | Template abstraction (from grant-mcp), citation patterns, compliance logic |
| `documentation-mcp` | Markdown + PDF + auto-generated API docs (Sphinx/MkDocs) | Technical writers, open-source maintainers, developer teams | 3 | Low-Medium | 2 | blog-mcp markdown pipeline, developer ecosystem integration (Claude Code) |
| `ebook-mcp` | EPUB 3 + MOBI + PDF (self-publishing) | Authors, indie publishers, academic authors | 3 | Medium | 2 | latex-mcp TeX (for math-heavy books), multi-format output, EPUB generation |
| `policy-brief-mcp` | PDF + HTML + plain-text summary | Policy researchers, think tanks, advocacy organizations | 3 | Low | 2 | whitepaper-mcp structure, formal template system |
| `proposal-mcp` | Business/product proposal (narrative + budget + timeline) | Entrepreneurs, product managers, consultants | 3 | Low-Medium | 2 | Template abstraction, structured sections, budget logic |
| `podcast-mcp` | Transcript + show notes + RSS feed + episode metadata | Podcasters, audio creators | 2 | Low-Medium | 3 | blog-mcp HTML/markdown, minimal novel infrastructure |
| `presentation-slides-mcp` (PowerPoint) | PowerPoint (.pptx) + Google Slides JSON | Business presenters, consultants, corporate teams | 2 | Medium | 3 | Non-TeX rendering engine, visual layout system (new domain) |
| `resume-mcp` | Markdown + LaTeX + ATS-friendly plain text | Job seekers, students, professionals | 2 | Low | 3 | Trivial format; commoditized; minimal strategic differentiation |
| `marketing-one-pager-mcp` | HTML + PDF + social media cards | Marketers, product teams | 1 | Low | 3+ | Off-brand; commoditized; low strategic fit for knowledge work |

---

## Recommended Expansion Sequence

### Phase 1: The Knowledge-Work Core (Months 0-6 after latex-mcp ships)

**Goal:** Establish ZenSci as the ultimate suite for research, thinking, and formal knowledge capture.

**Modules:**
1. `blog-mcp` — (part of shipping order)
2. `slides-mcp` — (part of shipping order)
3. `paper-mcp` — Peer-reviewed academic papers (IEEE, ACM, arXiv templates)
4. `lab-notebook-mcp` — The differentiator; proves "thinking partner" positioning

**Why this grouping:**
- Completes the academic/research core: think (lab notebook) → formalize (paper) → present (slides).
- Lab notebooks are ZenSci's unique ace. No competitor owns this positioning. It validates the thesis: "raw structured thought → publication-ready artifact."
- High code reuse (3 modules reuse latex-mcp TeX pipeline).
- Captures research-intensive users early. Establishes brand in academia.

**Strategic positioning:** "The forge for formal thinking. Researchers brainstorm with AI, then forge their reasoning into papers, slides, and lab records."

---

### Phase 2: Expand Academic + Professional Ecosystems (Months 7-12)

**Goal:** Broaden to adjacent professional/academic audiences while deepening template and domain logic.

**Modules:**
1. `thesis-mcp` — PhD dissertations, institutional theses
2. `whitepaper-mcp` — Think-tank and policy research
3. `grant-mcp` — (already confirmed; revenue anchor)
4. `patent-mcp` — Patent applications for engineers/IP lawyers
5. `ebook-mcp` — Self-publishing, long-form works
6. `documentation-mcp` — Technical documentation; bridges to developer ecosystem

**Why this grouping:**
- Thesis expands academic reach (every PhD candidate vs. just researchers with papers).
- Whitepaper differentiates from blog (serious, formal, policy-focused vs. casual, blog-style).
- Grant is revenue-generating (already confirmed in CONTEXT.md).
- Patent proves template abstraction handles domain-specific compliance logic.
- Ebook captures self-publishing market (growing, high-value users).
- Documentation ties to broader Claude ecosystem and developer positioning (Claude Code integration).

**Strategic positioning:** "From research to revenue. Theses, grants, patents, books, and formal documentation all flow from the same thinking-to-artifact forge."

**New infrastructure needs:**
- Multi-format output system (e.g., LaTeX + .docx, EPUB + PDF).
- Template abstraction (institutionally aware, funder-aware, compliance-aware).
- Advanced citation/bibliography system (unified across paper, thesis, grant, patent).

---

### Phase 3: Niche Modules & Lower Priority (Months 13+)

**Goal:** Ship modules only if demand is strong or strategic fit becomes clear.

**Modules (lower priority):**
1. `policy-brief-mcp` — Variant of whitepaper; lower priority unless think-tank partnerships emerge.
2. `proposal-mcp` — Business proposals; moderate fit; commoditized domain.
3. `podcast-mcp` — Audio transcripts + show notes; off-brand; minimal TeX reuse.
4. `presentation-slides-mcp` (PowerPoint) — Alternative to Beamer; lower TeX fit; new rendering engine.
5. `resume-mcp` — Commoditized format; low strategic differentiation.
6. `marketing-one-pager-mcp` — Off-brand; explicitly OUT OF SCOPE for knowledge work.

**Why these are lower priority:**
- Either lower strategic fit (podcast, resume) or explicit off-brand positioning (marketing).
- Commoditized domains where ZenSci has no unique advantage.
- Minimal reuse from existing modules.
- Ship only if customer demand is clear or competitive pressure emerges.

---

## New Schemas Identified

These new modules require the following data contracts. Append to `/sessions/blissful-eloquent-gates/mnt/ZenflowProjects/ZenithScience/schemas/SCHEMAS.md`:

### Lab Notebook & Reproducible Science

| Schema | Description | Related Module |
|--------|-------------|-----------------|
| `LabNotebookEntry` | Single lab entry: date, hypothesis, methodology, observations, reasoning chain, tags, citations | lab-notebook-mcp |
| `LabNotebook` | Collection of entries with cross-referencing, reproducibility metadata, data provenance | lab-notebook-mcp |
| `ReproducibilityMetadata` | Captures environment, dependencies, external data sources for reproducible science | lab-notebook-mcp |

### Academic Papers & Formal Research

| Schema | Description | Related Module |
|--------|-------------|-----------------|
| `ResearchPaper` | Full paper structure: abstract, introduction, related work, methods, results, discussion, references | paper-mcp |
| `ThesisMeta` | Thesis-specific metadata: institution, degree level, advisor(s), defense date, keywords, committee | thesis-mcp |
| `AcademicTemplate` | Abstraction for template systems: IEEE/ACM/arXiv/ProQuest/institutional LaTeX templates | paper-mcp, thesis-mcp |

### Books & Multi-Format Output

| Schema | Description | Related Module |
|--------|-------------|-----------------|
| `BookStructure` | Chapter organization, front/back matter, TOC generation, index metadata, page breaks | book-mcp |
| `EpubMetadata` | EPUB-specific metadata: identifier, language, creator, date, rights, language, direction | ebook-mcp |
| `MultiFormatOutput` | Abstraction for simultaneous output to LaTeX, EPUB, PDF, HTML from single source | book-mcp, ebook-mcp |

### Patents & Legal Documents

| Schema | Description | Related Module |
|--------|-------------|-----------------|
| `PatentClaim` | Single patent claim with dependent/independent relationships, formal structure, scope | patent-mcp |
| `PatentApplication` | Full application: title, abstract, drawings, specification, claims (organized as dependency tree) | patent-mcp |
| `ComplianceMetadata` | Funder/institution/jurisdiction-specific compliance rules, formatting requirements | patent-mcp, grant-mcp |

### Technical Documentation

| Schema | Description | Related Module |
|--------|-------------|-----------------|
| `DocumentationConfig` | Sphinx/MkDocs integration: project structure, theme, autodoc settings, plugin list | documentation-mcp |
| `APIDocs` | Auto-generated API documentation: function signatures, docstrings, examples, deprecation info | documentation-mcp |
| `APIIntegration` | Contract for auto-extracting signatures from code (Python, TypeScript, etc.) | documentation-mcp |

### Policy & Whitepapers

| Schema | Description | Related Module |
|--------|-------------|-----------------|
| `PolicyBriefStructure` | Policy brief sections: executive summary, issue analysis, policy options, recommendation, citations | policy-brief-mcp |
| `WhitepaperStructure` | Whitepaper sections: overview, background, analysis, recommendations, formal conclusions | whitepaper-mcp |

### Lower-Priority Extensions

| Schema | Description | Related Module |
|--------|-------------|-----------------|
| `ProposalStructure` | Business proposal: executive summary, problem, solution, timeline, budget, team, call-to-action | proposal-mcp |

---

## Open Questions for Cruz

1. **Lab Notebook — AI Integration:** How much should lab-notebook-mcp integrate with Claude's memory/reflection system? Should entries link to thinking sessions? Or stay as pure document capture?

2. **Paper vs. Thesis vs. Grant Boundary:** These three are all academic, LaTeX-based. Should they share a single template system, or be separate module-level templates?

3. **Multi-Format Output Architecture:** Several Phase 2 modules need simultaneous output (book → PDF + EPUB + Markdown; grant → TeX + .docx). Should this be a core `packages/core` capability, or module-specific?

4. **Documentation + Claude Code:** Should documentation-mcp integrate with Claude Code for auto-generating docs from code + inline AI reasoning? Or stay purely format-conversion?

5. **Patent Compliance:** Patent writing is heavily regulated. Should patent-mcp embed funder compliance logic (like grant-mcp), or stay format/structure only?

6. **Commercial vs. Open-Source:** Are any Phase 2 modules (patent, grant, ebook) expected to have commercial tiers or enterprise licensing? Or fully open-source across the suite?

7. **Resume-mcp Scope:** Resume is in the backlog (Phase 3). Is it out-of-scope entirely, or low-priority pending demand?

8. **Podcast Positioning:** Podcast is low-fit (Phase 3). Should it be explicitly deprecated from the vision, or kept as future expansion?

---

## Summary: The Two Tensions Resolved

**Tension 1 — Shipping Order:**
- `latex-mcp` (v0.1) → `blog-mcp` (8-12w) → `slides-mcp` (12-16w) → `newsletter-mcp` (16-20w).
- Blog first validates architectural robustness; slides proves TeX reuse; newsletter is easy follow-up.
- This order prioritizes architectural confidence over fast wins.

**Tension 2 — Ecosystem Expansion:**
- **Phase 1:** Establish academic/research core with paper-mcp, lab-notebook-mcp (differentiator), thesis-mcp, whitepaper-mcp.
- **Phase 2:** Expand professional reach with grant-mcp, patent-mcp, ebook-mcp, documentation-mcp (developer bridge).
- **Phase 3:** Niche modules only if demand is clear (podcast, proposal, policy-brief, presentation-slides, resume are lower priority).
- Lab notebooks are the strategic ace — they prove ZenSci's unique "thinking partner" positioning.

**Next steps:**
1. Confirm module order with Cruz and product leadership.
2. Detailed schemas specification for Phase 1 modules.
3. Naming research agent coordinates with this list for consistent repo naming (`[descriptor]-mcp`).
4. Semantic clusters agent maps the core capabilities these modules require.

---

**Document saved:** 2026-02-17 at 02:17 UTC

---

## Decisions Applied (Cruz Morales, 2026-02-17)

### IP & Ownership
- **Owner:** TresPiesDesign.com / Cruz Morales (Decision 1)

### GitHub Organization
- **GitHub org:** `TresPies-source` (Decision 4, supersedes "typecraft" naming research)
- **Domain:** ZenithScience.org (Decision 3)

### npm Scope
- **npm scope:** `@zen-sci` (from Decision 4, supersedes `@typecraft`)

### Working Directory for Code
- **Code directory:** `ZenflowProjects/ZenithScience/zen-sci/` (kebab-case, Decision 5)
- **Project artifacts:** Stay at `ZenflowProjects/ZenithScience/` (CONTEXT.md, scouts/, research/, etc.)

### Module Order & Lab-Notebook Phase 1 Inclusion
- **Recommended order remains open:** Scout recommends blog-first → slides → newsletter, but Phase 1 module list (paper-mcp, lab-notebook-mcp) needs Cruz's confirmation
- **Lab notebook as differentiator:** Remains high-priority strategic recommendation pending approval
- **Open for Cruz:** Should lab-notebook-mcp be Phase 1 or Phase 2? Does it include AI memory/reflection integration?
