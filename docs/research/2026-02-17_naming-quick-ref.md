# Typecraft Naming Quick Reference

**Use this as a guide when creating new repos, packages, and registering servers.**

---

## Organization Identity

| Item | Value | Notes |
|------|-------|-------|
| GitHub Org | `typecraft` | Primary brand identity |
| Website | `typecraft.io` | Domain to secure |
| npm Organization | `@typecraft` | Scope for all packages |
| MCP Registry Namespace | `io.github.typecraft` | GitHub-based convention |

---

## Module Naming by Type

### MCP Servers (as they appear in the registry)

**Pattern:** `<function>-mcp` (kebab-case)

```
latex-mcp           # Academic papers, proofs, math docs
blog-mcp            # Blog posts, newsletters (HTML/email)
grant-mcp           # Grant proposals
slides-mcp          # Presentation decks (Beamer + Reveal.js)
thesis-mcp          # Dissertations, theses [Future]
patent-mcp          # Patent documents [Future]
syllabus-mcp        # Course syllabi [Future]
whitepaper-mcp      # Technical whitepapers [Future]
cv-mcp              # Curricula vitae [Future]
wiki-mcp            # Wiki/knowledge bases [Future]
book-mcp            # Long-form books [Future]
report-mcp          # Reports, research documents [Future]
```

### npm Packages

**Pattern:** `@typecraft/<function>-mcp` (scoped, kebab-case)

```
@typecraft/latex-mcp
@typecraft/blog-mcp
@typecraft/grant-mcp
@typecraft/slides-mcp
@typecraft/core          # Shared utilities
@typecraft/cli           # Command-line interface
@typecraft/sdk           # Language SDKs
```

### GitHub Repositories

**Single monorepo:**
```
github.com/typecraft/typecraft
  ├── servers/          # MCP server implementations
  │   ├── latex-mcp/
  │   ├── blog-mcp/
  │   ├── grant-mcp/
  │   └── slides-mcp/
  ├── packages/         # Shared libraries
  │   ├── core/
  │   ├── cli/
  │   └── sdk/
  └── docs/
```

**Or polyrepo (if preferred):**
```
github.com/typecraft/latex-mcp
github.com/typecraft/blog-mcp
github.com/typecraft/grant-mcp
... (each module in its own repo)
```

---

## Naming Rules

### Always Follow These Rules

1. **Kebab-case for everything:** `latex-mcp` not `LatexMcp` or `latex_mcp`
2. **Lowercase ASCII only:** No special characters, no spaces
3. **Include "mcp" in server names:** `latex-mcp`, not just `latex`
4. **Function-based, not format-based:** `blog-mcp` (what it does) not `html-mcp` (what it outputs)
5. **Scoped packages:** Always `@typecraft/package-name` on npm
6. **Consistent across platforms:** Same name on GitHub, npm, MCP registry

### Tools Within MCP Servers

**Pattern:** `snake_case` (MCP standard)

```
get_document_template
list_supported_formats
generate_pdf
create_from_markdown
...
```

### Prompts Within MCP Servers

**Pattern:** `snake_case` or `kebab-case`

```
draft_academic_paper
revise_grant_proposal
generate_slides_outline
...
```

---

## Checklist for New Modules

When you add a new module (e.g., `thesis-mcp`):

- [ ] **Name:** All lowercase, kebab-case, includes "mcp"
- [ ] **GitHub repo:** Under `github.com/typecraft/thesis-mcp` or in monorepo as `servers/thesis-mcp/`
- [ ] **npm package:** `@typecraft/thesis-mcp` with this exact name in `package.json`
- [ ] **MCP registry:** Register as `io.github.typecraft/thesis-mcp`
- [ ] **Tools:** All internal tools use snake_case (e.g., `generate_thesis_outline`)
- [ ] **Docs:** Clearly state function, not format (e.g., "Thesis formatting MCP" not "LaTeX MCP for theses")

---

## Searchability Keywords

When adding modules to the MCP registry, include these tags/keywords:

- `document-generation`
- `publishing`
- `knowledge-work`
- `mcp-server`
- `typecraft`
- [function-specific: `academic`, `blogging`, `grants`, `presentations`, etc.]

---

## Brand Messaging

**Official description:**

> Typecraft is an open-source suite of MCP servers for transforming structured knowledge (markdown, outlines, brainstorms) into publication-ready artifacts. Choose your output format: academic papers (LaTeX), blog posts (HTML/email), grant proposals, or presentation slides.

**Tagline options:**
- "Craft Documents from Ideas"
- "From Thought to Publication"
- "Where Knowledge Becomes Artifact"

---

## Consistency Check

**Before launching a new module, verify:**

| Check | Correct | Incorrect |
|-------|---------|-----------|
| GitHub repo name | `typecraft/thesis-mcp` | `typecraft/Thesis-MCP` or `typecraft/thesis` |
| npm package | `@typecraft/thesis-mcp` | `@typecraft-thesis-mcp` or `typecraft-thesis-mcp` |
| MCP registry ID | `io.github.typecraft/thesis-mcp` | `io.github.typecraft.thesis.mcp` |
| Tools | `get_thesis_template` | `getThesisTemplate` or `GET_THESIS_TEMPLATE` |
| Docs heading | "Thesis MCP" | "Thesis Formatter" or "LaTeX Thesis Tool" |

---

**Last Updated:** 2026-02-17
**Maintained by:** Wide Research Agent
**Reference:** Full report in `2026-02-17_naming-research.md`
