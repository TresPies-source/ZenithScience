# Opus Onboarding & Spec Audit Prompt — ZenSci

**For:** Opus-level agent
**Task:** Onboard → Audit 19 specs → Improve
**Created:** 2026-02-18
**Author:** Cruz Morales / ZenSci Architecture

---

## Prompt

You are onboarding to the **ZenSci (ZenithScience)** project — a suite of MCP servers that converts structured markdown thought into publication-ready artifacts (LaTeX, HTML, email, slides, grants, patents, ebooks, and more). Your job is to audit all 19 draft specifications and make targeted improvements.

---

### Step 1: Onboard (read these files in order)

1. `ZenflowProjects/ZenithScience/CONTEXT.md` — project identity, IP owner (TresPiesDesign.com / Cruz Morales), key decisions
2. `ZenflowProjects/ZenithScience/STATUS.md` — current phase, architecture decisions, module roadmap
3. `ZenflowProjects/ZenithScience/architecture/2026-02-17_semantic-clusters.md` — the 9 behavioral clusters that define what the system does
4. `ZenflowProjects/ZenithScience/scouts/2026-02-17_module-strategy-scout.md` — the recommended shipping order and rationale for 14 modules
5. `ZenflowProjects/ZenithScience/schemas/SCHEMAS.md` — the 35+ data contract backlog

Then scan all 19 specs:
- `specs/infrastructure/` — packages-core-spec.md, packages-sdk-spec.md
- `specs/phase-1/` — latex-mcp, blog-mcp, paper-mcp, lab-notebook-mcp
- `specs/phase-2/` — slides, newsletter, grant, thesis, whitepaper, patent, ebook, documentation (8 files)
- `specs/phase-3/` — policy-brief, proposal, podcast, presentation-slides, resume (5 files)

---

### Step 2: Audit against these criteria

**A. Completeness** — every spec must have: Vision, Goals + Success Criteria, Non-Goals, MCP Tool Definitions (with real inputSchema/outputSchema JSON), TypeScript types, Python processing pattern, week-by-week implementation plan, risk table, appendices. Flag any section that is vague, placeholder-ish, or missing.

**B. Cross-spec consistency** — confirm:
- All specs reference `packages/core` correctly and agree on the `DocumentRequest`/`DocumentResponse` interface shape from `packages-core-spec.md`
- Dependency chains are accurate (Phase 2 specs say they depend on Phase 1; Phase 1 specs don't reference Phase 2)
- `OutputFormat` enum in packages-core-spec.md covers every format each module needs
- All kebab-case naming is consistent (`latex-mcp`, `@zen-sci`, `zen-sci/`, etc.)

**C. Technical accuracy** — use **Context7** to verify:
- `@modelcontextprotocol/sdk` — current tool registration API shapes (`ListToolsRequestSchema`, `CallToolRequestSchema`, transport patterns)
- `pandoc` — current CLI flags, supported `--to` format identifiers, available filters
- `ebooklib` — correct EPUB 3 generation API for `ebook-mcp-spec.md`
- `MJML` — current API for `newsletter-mcp-spec.md`
- Any TypeScript SDK patterns that may have changed

**D. Strategic coherence** — each spec's vision and positioning must align with ZenSci's core thesis: *"thinking partner, not just converter."* Check that `lab-notebook-mcp` is positioned as the differentiator, not just another format converter.

---

### Step 3: Make targeted improvements

For each issue found, edit the spec file directly. Improvements may include:
- Filling in vague sections with concrete content
- Correcting API shapes to match current library versions (ground in Context7 output)
- Adding missing cross-references between related specs (e.g., grant-mcp ↔ thesis-mcp shared bibliography system)
- Strengthening the `packages-core-spec.md` type definitions if any spec reveals a gap
- Writing a brief **Audit Notes** section at the top of any spec that received significant changes, documenting what was updated and why

After completing improvements, update `ZenflowProjects/ZenithScience/STATUS.md`:
- Change spec status from "Draft" to "Reviewed" for specs that pass
- Add a new entry under Architecture Decisions: `Spec Audit (2026-02-18) — [summary of key findings]`

---

### Suggested Tools & Connectors

| Tool | Purpose |
|------|---------|
| **Context7** | Fetch live docs: `@modelcontextprotocol/sdk`, `pandoc`, `ebooklib`, `mjml`, `sympy`, `zod` — ground all TypeScript/Python patterns in current API reality |
| **GitHub connector** | Check `github.com/modelcontextprotocol` for reference server implementations; scan for any existing art at `TresPies-source` |
| **Web Search** | Verify current MCP specification version, pandoc 3.x release notes, USPTO patent format requirements for `patent-mcp-spec.md`, EPUB 3.3 spec for `ebook-mcp-spec.md` |
| **Read + Edit tools** | Navigate and edit spec files directly — no need for bash unless checking directory structure |

**Priority Context7 lookups:**
```
@modelcontextprotocol/sdk  → tool registration, transport, schema types
pandoc                     → markdown → latex/html/docx/epub conversion flags
ebooklib                   → epub.EpubBook API
mjml                       → render() API and template syntax
sympy                      → latex() output and expression validation
zod                        → z.object() schema patterns for DocumentRequest validation
```

---

### Deliverables

1. All 19 spec files updated in place (or annotated with Audit Notes if major changes)
2. STATUS.md updated with audit summary and reviewed spec count
3. A brief `workspace/2026-02-18_audit-findings.md` summarizing: total specs reviewed, issues found per category (completeness / consistency / technical accuracy / strategic coherence), key improvements made, and any unresolved open questions for Cruz

---

**Working directory:** `/sessions/blissful-eloquent-gates/mnt/ZenflowProjects/ZenithScience/`
**IP Owner:** TresPiesDesign.com / Cruz Morales
**Naming convention:** kebab-case throughout (`zen-sci/`, `@zen-sci`, `latex-mcp`, etc.)
