# Cross-Agent Coordination Notes — Naming Research

**Date:** 2026-02-17
**From:** Wide Research Agent (Naming)
**To:** Strategic Scout Agent & Semantic Clusters Agent

---

## Summary for Parallel Agents

This naming research establishes the **structural framework** that the other agents should be aware of as they work in parallel on module scouting and semantic clustering.

---

## Key Constraints Established by This Research

### 1. Naming Scales to 15-20 Modules

**What this means for the Scout Agent:**

The recommended naming system (`@typecraft/<function>-mcp`) is tested and valid for up to 20 modules. You can confidently recommend new modules (thesis-mcp, patent-mcp, etc.) knowing they will:
- Fit the naming pattern
- Not create collisions
- Be discoverable in the MCP registry
- Scale without confusion

**Implication:** No need to restrict scouting to just the 4 confirmed modules. The namespace is designed for growth.

### 2. Function-Based Naming (Not Format-Based)

**What this means for the Scout Agent:**

When naming a new module, prioritize **what it does** not **what format it produces**.

**Good naming logic:**
- `thesis-mcp` — "Create academic theses" ✓
- `blog-mcp` — "Publish blog posts" ✓
- `report-mcp` — "Generate reports" ✓

**Avoid:**
- `latex-pdf-mcp` — Too format-specific ✗
- `markdown-to-html-mcp` — Too technical ✗

**Implication:** When scouting new modules, frame them by user intent, not implementation details.

### 3. Organization Name ("Typecraft") is Format-Agnostic

**What this means for the Semantic Clusters Agent:**

The organization is called "Typecraft" (document craftsmanship), not "LaTeX-MCP" or "Document-Generator." This gives semantic freedom:

- **Typecraft can support:**
  - LaTeX output (academic)
  - HTML/email output (blog/newsletter)
  - HTML presentations (slides)
  - Any text-based artifact format
  - [Future formats]: Markdown, DOCX, EPUB, etc.

- **Typecraft is NOT limited to:**
  - Any single output format
  - Any single markup language
  - Academic documents only

**Implication:** When clustering semantic behavior, you're not working within a "LaTeX-only" system. Typecraft is a **knowledge-to-artifact transformation platform** with multiple output channels.

---

## Inputs This Research Needed From You

### Scout Agent
**Question:** Which modules should the naming system accommodate?

**Answer given:** ~15-20 potential modules
- Confirmed: latex, blog/newsletter, grant, slides (4)
- Scouted: thesis, patent, syllabus, whitepaper, CV, wiki, book, report (8+)

**How naming research used this:** Tested naming scheme against growth to 20 modules; confirmed no collisions or scope creep at scale.

### Semantic Clusters Agent
**Question:** What is the behavioral identity of the system?

**This naming research:**
- Did NOT assume any specific semantic behavior
- Used function-agnostic naming (`<function>-mcp`)
- Avoided format-prescriptive names

**Why:** Naming should follow semantics, not the reverse. Once you define the semantic clusters, the naming will align.

---

## How This Feeds Forward

### For the Scout Agent

**Use this naming framework when evaluating new modules:**

```
Module Evaluation Checklist:
- [ ] Does the function map to a clear user intent?
- [ ] Can it be named as `<intent>-mcp`? (e.g., `patent-mcp`)
- [ ] Would a developer find it by searching for that intent?
- [ ] Can it coexist with existing modules in @typecraft scope?
```

**Example:**
- **Proposal:** "Patent document formatting tool"
- **Intent:** `patent-mcp` ✓
- **User discovery:** "I need patent formatting" + "patent-mcp" = found ✓
- **Namespace check:** `@typecraft/patent-mcp` doesn't conflict ✓

### For the Semantic Clusters Agent

**Use the established naming when mapping behaviors:**

Once you've defined the semantic clusters (e.g., "document structure transformation," "template application," "output format conversion"), those clusters will map cleanly to individual MCP servers:

```
Semantic Cluster: "Structure Transformation"
├── Applied via: @typecraft/latex-mcp
├── Applied via: @typecraft/blog-mcp
├── Applied via: @typecraft/slides-mcp
└── Applied via: [future modules]
```

The naming system doesn't constrain your semantic analysis — it just provides the labels.

---

## Known Unknowns That Might Need Coordination

### 1. Module Interdependencies

**Question:** Will some modules depend on others?
- E.g., Does `thesis-mcp` build on `report-mcp` functionality?

**Current assumption:** Each module is independent MCP server (can be installed separately).

**If wrong:** Might need to adjust naming (e.g., `thesis-mcp` as a plugin/extension vs. standalone server). Recommend clarifying architecture with semantic clusters work.

### 2. Shared Core Libraries

**Question:** How much code is shared across modules?

**Current naming:** Assumes shared code in `@typecraft/core` package.

**If wrong:** Might need more granular scoping (e.g., `@typecraft/LaTeX-core`, `@typecraft/template-core`). Your semantic clustering will inform this.

### 3. Output Format Variations

**Question:** Will a single module (e.g., `blog-mcp`) support multiple output formats (HTML + email + PDF)?

**Current naming:** Assumes function-first (`blog-mcp` handles "blogging" regardless of output format).

**If wrong:** Might need format qualifiers (e.g., `blog-html-mcp` vs. `blog-email-mcp`). Your semantic work will clarify this.

---

## Handoff Summary

### To Scout Agent
- **What you need to know:** You have 15-20 module slots with clear naming
- **Constraint:** Function-based naming (not format-based)
- **Action:** Validate module ideas against the naming framework in `2026-02-17_naming-quick-ref.md`

### To Semantic Clusters Agent
- **What you need to know:** Naming is format-agnostic and function-first
- **Freedom:** Define semantic behavior without naming constraints
- **Coordination:** Once you map clusters, the naming will align naturally

### To Implementation Team
- **What you need to do:** Use the three artifacts as a checklist:
  1. Full report: Deep context and rationale
  2. Executive summary: 2-page overview for stakeholders
  3. Quick reference: Implementation checklist for each module

---

## Consensus Points (Ready to Commit)

- [ ] Organization name: **Typecraft** (high confidence)
- [ ] npm scope: **@typecraft** (high confidence)
- [ ] MCP namespace: **io.github.typecraft** (high confidence)
- [ ] Module naming: **`<function>-mcp`** (high confidence)
- [ ] Monorepo structure: **`servers/` + `packages/`** (recommended, can adjust)

**Next gate:** Trademark validation + stakeholder approval

---

## Open Questions for Cross-Agent Alignment

1. **Scout Agent → Semantics Agent:** Should module discovery be automatic (all `@typecraft/*` servers) or curated (only officially endorsed)?
2. **Semantics Agent → Scout Agent:** Are there semantic clusters that suggest new modules we haven't considered?
3. **Implementation Team:** Should monorepo vs. polyrepo be decided before module 2 launch, or after module 1 validation?

---

**Prepared for:** Cross-agent coordination meeting (recommended: after Scout finishes initial scouting, before Semantics completes final clustering)
