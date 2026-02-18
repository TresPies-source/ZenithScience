# Strategic Scout: Knowledge-Work Forge — MD to LaTeX MCP Suite

**Date:** 2026-02-17
**Owner:** Cruz / Tres Pies Design
**Tension:** Between building a *narrow conversion utility* (MD → LaTeX, developer tool) and building a *broad thinking environment* (AI-assisted mathematical and strategic reasoning that produces publication-ready artifacts across multiple formats).

---

## Context

The core idea: an MCP server that helps AIs speak in LaTeX and converts Markdown to LaTeX. Further, help strategic thinkers and mathematicians brainstorm with AI and then take their work and turn it into LaTeX. The MCP application will display LaTeX in Claude Code.

**Suite scope confirmed:** Four modules — LaTeX, Blog/Newsletter, Grant Proposal, Slide Decks (Beamer, Reveal.js).
**Runtime:** Hybrid TypeScript (MCP protocol layer) + Python (conversion engine — pandoc, SymPy, LaTeX toolchain).
**Timeline pressure:** None — quality and architecture over speed.
**Primary user:** All three — developers, mathematicians/researchers, strategic thinkers/writers. Design for the suite from day one.

---

## Routes Explored: Core Product Direction

| Name | Thesis | Risk | Impact | Duration | Optimizes For | Sacrifices |
|------|--------|------|--------|----------|---------------|------------|
| The Precise Converter | Build a tight, reliable MD→LaTeX pipe. Symbols, tables, equations — all handled correctly. | Becomes a commodity quickly; no moat | Developers use it everywhere; high adoption ceiling | 4–6 weeks | Correctness, reliability | Depth of thinking-partner use case |
| The Mathematical Thinking Partner | Claude helps you reason through math, then renders it in LaTeX. Conversion is the *last* step, not the *only* step. | Scope creep; harder to ship v1 | Unique positioning; genuinely useful to researchers and mathematicians | 8–12 weeks | Depth of use, differentiation | Fast time-to-market |
| The Suite Architect | Design MD→LaTeX as infrastructure for a family of MCPs (blog, newsletter, academic paper, grant proposal). Ship the suite together or in sequence. | Coordination overhead; harder to focus | Network effects between tools; shared infrastructure pays dividends | 3–6 months | Long-term platform play | Speed of any single tool |
| The Claude Code Native | Build specifically for Claude Code — LaTeX renders in the IDE, math is first-class in the dev environment. | Narrow audience; dependent on Claude Code ecosystem | Deep integration, no competition in this exact space | 3–5 weeks | Developer workflow integration | Mathematician/strategist use case |
| The Academic Publisher | Target researchers moving from AI-assisted ideation to submission-ready LaTeX (papers, proofs, slide decks). Full pipeline. | Niche market; longer sales cycle | High willingness to pay; academic publishing is painful | 10–16 weeks | Workflow completeness, publication quality | General-purpose appeal |

---

## Routes Explored: Repository Structure

| Name | Thesis | Risk | Impact | Duration to First Module | Optimizes For | Sacrifices |
|------|--------|------|--------|--------------------------|---------------|------------|
| The Unified Forge (Monorepo) | All four modules in one repo with shared `core/` — types, conversion pipeline, pandoc bindings, tests | All-or-nothing releases; repo grows heavy | Atomic refactors; shared LaTeX→HTML pipeline maintained once | 2–3 weeks | Developer experience, code reuse | Independent versioning per module |
| The Constellation (Polyrepo + Org) | Each module is its own repo under one org; shared code extracted to `core` library | Shared library versioning creates coordination overhead | Each module discoverable/forkable independently | 3–4 weeks | Modularity, independent release cadence | Atomic cross-module refactors |
| The Omnibus (One MCP, Many Tools) | One MCP server, one install; all modules are registered tools inside a single server | Installs all dependencies regardless of user need | Simplest user experience; protocol consistency | 2 weeks | User simplicity, installation ease | Modularity, selective installation |
| The Extensible Engine (Core + Plugins) | A shared MCP framework is the core; each module is a plugin; community can extend | Most complex to architect; over-engineering risk at v0.1 | Most extensible long-term; community can contribute new formats | 5–8 weeks | Long-term extensibility, community | Speed to v1, simplicity |

---

## What We Heard

**On product direction:** User confirmed the Suite Architect route is the core bet, with the Mathematical Thinking Partner embedded inside it. Designing for all three users (developer, mathematician, strategist) from day one. Full suite: LaTeX, Blog/Newsletter, Grant Proposal, Slide Decks.

**On repo structure:** User asked to scout the tradeoffs — no preference stated yet.

**Key insight:** The four modules share more infrastructure than typical MCP suites. LaTeX, grant proposals, and slide decks all use the same underlying toolchain (pandoc, TeX engine, bibliography management). The monorepo approach is natural, not forced.

---

## Synthesized Direction: The Knowledge-Work Forge

A suite of MCP servers built on a shared core: **raw structured thought in (markdown, outlines, notes, AI-assisted brainstorming) → publication-ready artifacts out**. LaTeX is the flagship module and the hardest problem; solving it well creates the architecture for everything else.

**What makes this different from a converter:** The AI doesn't just translate syntax — it helps structure, formalize, and refine the thinking *before* rendering. The LaTeX output is evidence that the thinking was done well.

**Recommended architecture:**
- **Monorepo** with `packages/core` (shared conversion pipeline, pandoc bindings, TeX toolchain), then `packages/latex-mcp`, `packages/blog-mcp`, `packages/grant-mcp`, `packages/slides-mcp`
- **TypeScript** MCP protocol layer (Anthropic SDK compliance, clean tool registration)
- **Python** processing engine for conversion (pandoc, SymPy, LaTeX generation, bibliography)
- MCP server calls Python engine via subprocess or local service
- v0.1: LaTeX module only, monorepo scaffolded, core shared library established
- v0.2: Blog/Newsletter module (leverages shared MD parsing from core)
- v0.3: Slides module (Beamer + Reveal.js — same LaTeX engine, different output target)
- v0.4: Grant Proposal module (structured templates, citation management)

---

## Repository Name Candidates

**Functional:**
- `md-to-latex-mcp` — clear, searchable
- `latexify-mcp` — action-oriented
- `texbridge-mcp` — bridging metaphor

**Brandable:**
- `texcraft` — crafting structured thought into LaTeX
- `mathscribe` — the scribe for mathematical thinking
- `penrose-mcp` — nod to Roger Penrose (mathematical elegance)
- `formulaic` — formula + systematic
- `inkwell-mcp` — writing metaphor, outputs publishable work

**Suite-oriented:**
- `typeset-mcp` — broad enough to cover all output formats
- `scholarflow` — academic workflow suite framing
- `docforge` — forging documents from raw thought
- `knowledge-forge` — covers all knowledge-work outputs

---

## Next Steps

1. **Choose a repo name and GitHub org name** — the org name matters most; it frames the suite (e.g., `texforge/`, `docforge/`, `knowledgeforge/`)
2. **Scaffold the monorepo** — `packages/core`, `packages/latex-mcp`, shared `tsconfig`, Python engine stub
3. **Define the shared data model** — what does a "document request" look like in the core? (source: string, format: enum, options: object) — this contract is the foundation of all modules
4. **Build the LaTeX module v0.1** — MD → LaTeX conversion with math symbol handling, tables, figures; test with Claude Code
5. **Validate the thinking-partner use case** — before shipping v0.2, test whether AI-assisted brainstorming → LaTeX is actually the differentiator users value, or whether clean conversion is sufficient

---

## Open Questions

- Which module ships after LaTeX? Blog/Newsletter is fastest to build; Slides is most architecturally similar to LaTeX (shares the TeX engine); Grant Proposal has highest value to a specific audience
- Does the Blog/Newsletter MCP output HTML/MJML or a structured format that a separate publishing tool handles?
- Is Beamer (LaTeX slides) or Reveal.js (web slides) the primary slide target? They require different rendering pipelines
- What is the distribution model — npm package, Docker container, or both?
- Is there a commercial layer (API, hosted service) or is this fully open-source?
