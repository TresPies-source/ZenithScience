# ZenithScience (ZenSci) — Shared Project Context

**Owner:** TresPiesDesign.com / Cruz Morales
**Domain:** ZenithScience.org
**GitHub:** github.com/TresPies-source/ZenithScience
**Product name:** ZenSci
**Working directory:** ZenflowProjects/ZenithScience/zen-sci/
**Last updated:** 2026-02-17

---

## Vision

The **Knowledge-Work Forge** — a suite of MCP servers that takes raw structured thought (markdown, outlines, AI-assisted brainstorming) and forges it into publication-ready artifacts. AIs speak in the target format natively; humans bring the thinking.

The LaTeX MCP is the flagship module — the hardest problem. Solving it well creates the shared architecture for everything else.

---

## Synthesized Direction

> "Raw structured thought in → publication-ready artifact out."
> The format differs per module. The pipeline logic is shared.

---

## Confirmed Suite Modules (v0)

| Module | Output Format | Primary User | Shared Infrastructure |
|--------|--------------|--------------|----------------------|
| `latex-mcp` | LaTeX (.tex, .pdf) | Mathematicians, researchers, academics | pandoc, TeX engine, bibliography |
| `blog-mcp` | HTML, markdown | Writers, developers | MD parsing, frontmatter |
| `newsletter-mcp` | MJML, HTML email | Marketers, writers | Email templates, responsive layout |
| `grant-mcp` | LaTeX / Word (.docx) | Academics, researchers | Structure templates, citations |
| `slides-mcp` | Beamer (LaTeX) + Reveal.js | Educators, researchers, presenters | TeX engine OR web renderer |

---

## Target Users (All Three, Designed Together)

1. **Developers** — want LaTeX/formatted output as native tool output in Claude Code
2. **Mathematicians / Researchers** — brainstorm with AI, formalize reasoning, publish
3. **Strategic Thinkers / Writers** — high-level reasoning → publication-quality documents

---

## Tech Stack Decisions (Preliminary)

- **MCP Protocol Layer:** TypeScript / Node.js (Anthropic MCP SDK compliance)
- **Processing Engine:** Python (pandoc, SymPy, LaTeX toolchain, bibliography management)
- **Architecture pattern:** MCP server calls Python engine via subprocess or local service
- **Repo structure:** Monorepo (`packages/core` + per-module packages) — decision under active scouting

---

## Strategic Position

- Not just a converter — the AI helps structure and formalize thinking *before* rendering
- LaTeX output is evidence that thinking was done rigorously
- Suite creates network effects: same input format, different publication targets

---

## Architecture Decisions (Cruz Morales, 2026-02-17)

### Decision 1 — IP Ownership

**Statement:** "We're constrained by being the intellectual property of TresPiesDesign.com and Cruz Morales"

**Decision:** All intellectual property in the ZenSci / ZenithScience project is owned by TresPiesDesign.com and Cruz Morales.

**Implications:** All documents reflect this ownership in headers. No open-source licensing decision has been made yet — IP is attributed but license is TBD.

---

### Decision 2 — Product Name Confirmed

**Statement:** "the product name will be ZenSci"

**Decision:** The product is officially named ZenSci. This is both the public-facing name and the internal nickname.

**Implications:** All documents use "ZenSci" consistently.

---

### Decision 3 — Project Website Domain

**Statement:** "ZenithScience.org is our domain name and project website"

**Decision:** The official domain is ZenithScience.org.

**Implications:** All references to website or domain use ZenithScience.org.

---

### Decision 4 — GitHub Organization

**Statement:** "TresPies-source is our GitHub"

**Decision:** The GitHub organization is `TresPies-source` (github.com/TresPies-source). The monorepo is github.com/TresPies-source/ZenithScience.

**Implications:** This SUPERSEDES the naming research agent's recommendation of "typecraft" as the GitHub org. The typecraft recommendation may still apply as a suite brand concept, but the GitHub org is TresPies-source. npm scope becomes `@zen-sci` or `@zensci` (NOT `@typecraft`). MCP registry namespace becomes `io.github.trespies-source`.

---

### Decision 5 — Working Directory Structure + Kebab-Case

**Statement:** "ZenflowProjects/ZenithScience/zensci/ will be the working directory for the product itself so that the actual code is one layer deeper but still in the same repo as the decisions and agent workspace" + "use kebab-case"

**Decision:** Product code lives at `ZenflowProjects/ZenithScience/zen-sci/` (kebab-case). Project management artifacts (CONTEXT.md, scouts/, research/, architecture/, schemas/, specs/) live at `ZenflowProjects/ZenithScience/`. Kebab-case applies to all directories and package names throughout the project.

**Implications:** All future specs and implementation prompts target `ZenflowProjects/ZenithScience/zen-sci/` for code. Decision documents stay at the parent level. All directory names use kebab-case.

---

## Open Strategic Questions

1. What order should modules ship? (LaTeX first — then what?) → **DECIDED (Decision 5 partially): Blog validates architecture first; then slides; then newsletter**
2. What other modules belong in the ecosystem beyond the 4 confirmed? → **Open — Decision 5 established working directory, but module expansion decisions remain open**
3. What should the GitHub org be named? → **DECIDED: TresPies-source (Decision 4)**
4. Should the suite be one MCP with many tools, or separate MCPs per module? → **Open**
5. Is there a commercial layer, or fully open-source? → **Open**

---

## Parallel Agent Work (2026-02-17)

Three agents are working simultaneously. Their outputs live in:
- `scouts/` — strategic scout on module strategy
- `research/` — wide research on naming conventions
- `architecture/` — semantic clusters map

Agents should write findings that inform each other. The naming research should know what modules the scout proposes. The scout should be informed by the semantic clusters.

---

## Running Schemas List

See `schemas/SCHEMAS.md` for the full list of data contracts to specify.
