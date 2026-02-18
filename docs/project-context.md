# ZenSci — Agent Orientation Guide

**Product:** ZenSci (ZenithScience)
**IP Owner:** TresPiesDesign.com / Cruz Morales
**Domain:** ZenithScience.org
**GitHub:** github.com/TresPies-source/ZenithScience
**npm scope:** @zen-sci
**Current phase:** Specifications Complete / Pre-Code
**Last updated:** 2026-02-18

> **If you are a new agent, read this file first. Then read CONTEXT.md. Then read STATUS.md. Then read the file(s) relevant to your task.**

---

## What ZenSci Is (3 sentences)

ZenSci is a suite of MCP (Model Context Protocol) servers that converts structured markdown and AI-assisted reasoning into publication-ready artifacts — LaTeX PDFs, HTML blog posts, grant proposals, slide decks, academic papers, ebooks, patents, and more. The core insight is that the AI acts as a **thinking partner first** (helping structure reasoning, validate math, organize arguments) and a **format converter second** (rendering to whichever format the audience expects). One input schema (`DocumentRequest`), many output formats — shared infrastructure means every quality improvement benefits all modules simultaneously.

---

## Filesystem Map

```
ZenflowProjects/ZenithScience/           ← Decision & agent workspace root
│
├── project_context.md                   ← YOU ARE HERE — read first
├── CONTEXT.md                           ← Architecture decisions, IP, tech stack, open questions
├── STATUS.md                            ← Live project health — current phase, blockers, next steps
│
├── architecture/
│   └── 2026-02-17_semantic-clusters.md  ← 9 behavioral clusters; what the system DOES
│                                            (PARSE → FORMALIZE → STRUCTURE → CONVERT →
│                                             RENDER → VALIDATE)
│
├── scouts/
│   └── 2026-02-17_module-strategy-scout.md
│                                        ← Recommended shipping order + 14-module ecosystem map
│                                            Phase 1 → Phase 2 → Phase 3 rationale
│
├── research/
│   └── 2026-02-17_naming-research.md    ← GitHub org / npm scope / branding analysis
│       (also: naming-quick-ref.md, naming-summary.md)
│
├── schemas/
│   └── SCHEMAS.md                       ← 35+ data contracts backlog; status-tracked
│                                            🔴 Not started | 🟡 Drafted in spec | 🟢 Finalized
│
├── specs/                               ← 19 draft release specifications (written 2026-02-18)
│   │
│   ├── infrastructure/                  ← READ THESE FIRST — all modules depend on them
│   │   ├── packages-core-spec.md        ← packages/core: DocumentRequest/Response types,
│   │   │                                   MarkdownParser, CitationManager, MathValidator,
│   │   │                                   ConversionPipeline, public API surface
│   │   └── packages-sdk-spec.md         ← packages/sdk: ZenSciServer abstract base class,
│   │                                       PandocJob/PandocResult, callPythonEngine pattern
│   │
│   ├── phase-1/                         ← Research & academic core (first to build)
│   │   ├── latex-mcp-v0.1-spec.md       ← Flagship. MD → LaTeX → PDF. Math (SymPy),
│   │   │                                   citations (BibTeX), theorems, figures.
│   │   ├── blog-mcp-v0.2-spec.md        ← Architectural stress-test. MD → HTML.
│   │   │                                   Proves core abstraction is format-agnostic.
│   │   ├── paper-mcp-spec.md            ← IEEE/ACM/arXiv templates. Reuses latex-mcp TeX.
│   │   └── lab-notebook-mcp-spec.md     ← Strategic differentiator. Thinking partner +
│   │                                       reproducible science. Structured entries, PDF.
│   │
│   ├── phase-2/                         ← Professional ecosystem (builds on Phase 1)
│   │   ├── slides-mcp-spec.md           ← Beamer (LaTeX PDF) + Reveal.js (HTML)
│   │   ├── newsletter-mcp-spec.md       ← MJML + HTML email
│   │   ├── grant-mcp-spec.md            ← NIH/NSF/ERC LaTeX + Word. Multi-format output.
│   │   ├── thesis-mcp-spec.md           ← University dissertations. Institutional templates.
│   │   ├── whitepaper-mcp-spec.md       ← Formal PDF + HTML. Think-tank/policy research.
│   │   ├── patent-mcp-spec.md           ← USPTO/WIPO applications. Claim tree structure.
│   │   ├── ebook-mcp-spec.md            ← EPUB 3 + MOBI + PDF. Multi-format from one source.
│   │   └── documentation-mcp-spec.md   ← Sphinx/MkDocs + auto API docs. Developer bridge.
│   │
│   └── phase-3/                         ← Niche modules — ship only when demand is clear
│       ├── policy-brief-mcp-spec.md
│       ├── proposal-mcp-spec.md
│       ├── podcast-mcp-spec.md
│       ├── presentation-slides-mcp-spec.md  ← PowerPoint; non-TeX rendering (new domain)
│       └── resume-mcp-spec.md
│
├── handoffs/
│   └── handoff_zero.md                  ← Universal agent onboarding handoff package
│
├── workspace/
│   └── 2026-02-18_opus-onboarding-prompt.md  ← Prompt for Opus audit pass
│
└── zen-sci/                             ← Product code root (scaffolded, pre-code)
    ├── README.md
    ├── packages/
    │   ├── core/                        ← Shared TypeScript library (empty — awaiting v0.0.1)
    │   └── sdk/                         ← MCP protocol base class (empty)
    └── servers/
        ├── latex-mcp/                   ← Module server stubs (empty directories)
        ├── blog-mcp/
        ├── slides-mcp/
        ├── newsletter-mcp/
        └── grant-mcp/
```

---

## Key Decisions (Immutable — Do Not Override)

| # | Decision | Value |
|---|----------|-------|
| 1 | IP Owner | TresPiesDesign.com / Cruz Morales |
| 2 | Product name | ZenSci |
| 3 | Domain | ZenithScience.org |
| 4 | GitHub org | TresPies-source (NOT typecraft) |
| 5 | Code working directory | `zen-sci/` (kebab-case throughout) |
| 5a | npm scope | `@zen-sci` |
| 5b | Naming convention | kebab-case: `latex-mcp`, `zen-sci/`, `@zen-sci` |
| 6 | Repo structure | Monorepo — `packages/core` + `packages/sdk` + `servers/[module]/` |
| 7 | Tech stack | TypeScript MCP layer + Python processing engine |
| 8 | All specs written | 2026-02-18 — 19 specs across 3 phases + 2 infrastructure |

---

## Behavioral Vocabulary (Use These Verbs Consistently)

The 9 semantic clusters define how agents should think and write about the system:

| Verb | Cluster | What it means |
|------|---------|---------------|
| **PARSE** | 1 — Parsing | Extract meaning from raw markdown, frontmatter, citations |
| **FORMALIZE** | 2 — Thinking Partnership | Help human structure fuzzy thinking into rigorous form |
| **STRUCTURE** | 3 — Document Structuring | Organize into sections/hierarchy per target format |
| **CONVERT** | 4 — Format Conversion | Translate from canonical MD to LaTeX/HTML/MJML/etc. |
| **RENDER** (math) | 5 — Mathematical | SymPy validation, equation environments, proof structure |
| **CITE** | 6 — Citation | Normalize, validate, render BibTeX/CSL bibliography |
| **RENDER** (output) | 7 — Rendering | TeX engine, pandoc, EPUB compiler — produce final artifact |
| **CONFIGURE** | 8 — Configuration | TeX engine choice, themes, templates, metadata |
| **VALIDATE** | 9 — Error Handling | Check completeness, catch incompatibilities, suggest fixes |

---

## Tech Stack Quick Reference

```
TypeScript (Node.js)
  └── @modelcontextprotocol/sdk  ← MCP protocol compliance, tool registration
  └── zod                        ← DocumentRequest schema validation
  └── packages/sdk (ZenSciServer) ← Abstract base class all servers extend

Python
  └── pandoc (CLI subprocess)    ← Core MD → LaTeX/HTML/EPUB conversion engine
  └── TeX Live                   ← pdflatex/xelatex/lualatex for PDF rendering
  └── sympy                      ← Math expression validation and LaTeX generation
  └── bibtexparser               ← BibTeX parsing for citations
  └── ebooklib                   ← EPUB 3 generation (ebook-mcp)
  └── mjml                       ← Email-safe HTML (newsletter-mcp)

Shared data contract:
  DocumentRequest → [MCP server] → [Python engine] → DocumentResponse
```

---

## What Is Complete vs. What Needs Building

| Layer | Status | Notes |
|-------|--------|-------|
| Strategic scouting | ✅ Complete | Module order, ecosystem, 14 modules mapped |
| Naming & identity | ✅ Complete | TresPies-source, @zen-sci, ZenithScience.org |
| Semantic architecture | ✅ Complete | 9 clusters, gap analysis, data flow |
| Schemas backlog | ✅ Complete | 35+ contracts identified; all 🟡 Drafted in specs |
| Specifications (all phases) | ✅ Complete | 19 specs — infrastructure through Phase 3 |
| Code scaffold | ✅ Scaffolded | zen-sci/ directory tree created; no code yet |
| packages/core | ❌ Not started | Start here — all modules depend on it |
| packages/sdk | ❌ Not started | Second — ZenSciServer base class |
| latex-mcp v0.1 | ❌ Not started | Flagship module — first spec to implement |
| All other modules | ❌ Not started | Waiting on packages/core + latex-mcp validation |

---

## Reading Order by Agent Type

### Strategic / Architectural Agent
1. This file (`project_context.md`)
2. `CONTEXT.md` — all 5 decisions + open questions
3. `STATUS.md` — current phase, blockers, active workstreams
4. `architecture/2026-02-17_semantic-clusters.md` — behavioral architecture
5. `scouts/2026-02-17_module-strategy-scout.md` — module order rationale
6. `specs/infrastructure/packages-core-spec.md` — foundation types
7. Task-specific spec files as needed

### Coding / Implementation Agent
1. This file (`project_context.md`)
2. `CONTEXT.md` — decisions (especially Decision 5: directory + kebab-case)
3. `specs/infrastructure/packages-core-spec.md` — MUST read before any code
4. `specs/infrastructure/packages-sdk-spec.md` — MUST read before any server
5. Your assigned module spec (e.g., `specs/phase-1/latex-mcp-v0.1-spec.md`)
6. `schemas/SCHEMAS.md` — cross-check data contracts
7. `zen-sci/` — inspect the empty scaffold before writing code

### Audit / Review Agent
1. This file
2. `STATUS.md`
3. All 19 spec files (systematically)
4. `schemas/SCHEMAS.md` — verify alignment
5. `workspace/2026-02-18_opus-onboarding-prompt.md` — audit criteria

---

## Open Questions (Pending Cruz Decision)

1. **Module shipping order confirmed?** Scout recommends latex-mcp → blog-mcp → slides-mcp → newsletter-mcp → grant-mcp. Awaiting confirmation.
2. **lab-notebook-mcp in Phase 1?** Currently spec'd as Phase 1 but flag as differentiator — confirm priority.
3. **Open-source license?** IP is TresPiesDesign.com/Cruz Morales. License TBD. Blocker before public shipping.
4. **Commercial vs. open-source?** Some Phase 2 modules (grant, patent) may warrant commercial tiers.
5. **MCP registry namespace?** `io.github.trespies-source` vs. alternatives.

---

*Generated: 2026-02-18 | Maintained by: ZenSci architecture agents*
*Owner: TresPiesDesign.com / Cruz Morales*
