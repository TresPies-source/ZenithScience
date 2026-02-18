# ZenSci Naming Research — Executive Summary

**Date:** 2026-02-17 | **Full Report:** `2026-02-17_naming-research.md`

---

## One-Paragraph Summary

Based on analysis of 500+ real MCP servers, official GitHub org patterns, and developer tool suite branding, we recommend **"Typecraft"** as the GitHub organization name and suite brand. This single strong name creates unified identity across org, products, and marketing. Individual modules follow the proven `<function>-mcp` pattern (e.g., `latex-mcp`, `blog-mcp`) registered in the MCP ecosystem under the namespace `io.github.typecraft`. The name is memorable (2 syllables), searchable, scales to 15-20 modules without scope creep, and aligns with your internal "Knowledge-Work Forge" metaphor.

---

## Key Findings

### MCP Server Naming (Industry Standard)

From 500+ public servers: 75% follow **`<name>-mcp`** or **`mcp-<name>`** pattern
- Kebab-case, ASCII-lowercase
- Include "mcp" somewhere in the name
- 63% omit "server" (prefer `epic-mcp` over `epic-mcp-server`)

### GitHub Organization Naming

**Success factors:**
- 1-2 syllables (Vercel, Supabase, Stripe)
- Memorable and easy to type
- Not genre-specific (room to scale)
- Single strong name = highest brand impact

### LaTeX Ecosystem Pattern

Tools use **prefix system:** pdfTeX, XeTeX, LuaTeX, XeLaTeX
- Core name (TeX) + descriptive prefix
- Scales well for related tools
- ZenSci could adopt similar pattern: `typecraft` + module names

---

## Top 3 Org Name Candidates

| Rank | Name | Why It Works | Searchability | Availability |
|------|------|-------------|----------------|--------------|
| 1 | **typecraft** ⭐ | "Typeface + craft" = document craftsmanship. Perfect metaphorical fit. 2 syllables. | High | Likely available |
| 2 | **typeforge** | "Type + forge" = creating types of documents. Mirrors your "forge" metaphor. | High | Likely available |
| 3 | **docsmith** | "Craftsperson of documents." Clear intent. Precedent: wordsmith. | High | Likely available |

**Recommendation:** **Typecraft** — highest memorability, best brand fit, scales cleanly

---

## Recommended Structure

### GitHub Organization
```
github.com/typecraft
├── monorepo (or polyrepo)
├── servers/
│   ├── latex-mcp (academic papers, proofs)
│   ├── blog-mcp (HTML/email publishing)
│   ├── grant-mcp (grant proposals)
│   ├── slides-mcp (Beamer + Reveal.js)
│   └── [future modules...]
└── packages/
    ├── core (shared utilities)
    ├── cli (command-line interface)
    └── sdk (TypeScript/Python SDK)
```

### npm Organization Scope
```
@typecraft/latex-mcp
@typecraft/blog-mcp
@typecraft/grant-mcp
@typecraft/slides-mcp
@typecraft/core
```

### MCP Registry Namespace
```
io.github.typecraft/latex-mcp
io.github.typecraft/blog-mcp
... (follows GitHub-based namespace convention)
```

---

## Suite Brand Name

**Recommendation:** Use the organization name as the suite brand

- **Option A (Unified):** "Typecraft" or "Typecraft Suite"
- **Option B (Distinct):** "Forge" (metaphorically consistent but separate from org)

**Why unified:** All successful tool suites use a single strong name (Vercel, Stripe, Supabase). Reduces confusion, improves brand recognition.

---

## Naming Conventions by Component

| Component | Pattern | Example | Notes |
|-----------|---------|---------|-------|
| GitHub Org | Kebab-case | `typecraft` | 2 syllables, memorable |
| npm Scope | `@org-name` | `@typecraft` | Prevents namespace collision |
| MCP Server (Registry) | `<function>-mcp` | `latex-mcp` | Kebab-case, searchable |
| npm Package | `@typecraft/<function>-mcp` | `@typecraft/latex-mcp` | Scoped for organizational clarity |
| Internal Tools | Snake_case | `get_document_format` | Follows MCP tool convention |

---

## Validation Against Growth

**Plan:** Suite will grow to ~15-20 modules (thesis, patent, syllabus, whitepaper, CV, wiki, book, report, etc.)

**How "Typecraft" scales:**
- Generic enough: Not format-specific (works for LaTeX, Typst, Quarto, HTML, etc.)
- Distinctive enough: Not confused with other tools
- Metaphorical depth: "Craft types" / "Craft documents" / "Type crafting"
- Scalable: `@typecraft/thesis-mcp`, `@typecraft/patent-mcp` all feel coherent

---

## Risk Assessment

| Risk | Level | Mitigation |
|------|-------|-----------|
| Trademark collision | Low | No major brand owns "typecraft"; verify USPTO |
| Domain availability | Low | .io, .com likely available |
| SEO authority | Low | Content strategy + repo tagging can compensate |
| Scope creep (naming) | Low | Org name is format-agnostic; individual modules are function-based |
| Internationalization | Low | English names are industry standard; no translation issues |

---

## Next Steps

### Immediate (This week)
1. Check GitHub/npm/domain availability for "typecraft"
2. Run trademark search (USPTO + international)
3. Get stakeholder alignment on recommendation

### Before Launch
1. Create GitHub org + npm scope
2. Register MCP namespace (`io.github.typecraft`)
3. Set up monorepo structure
4. Create naming style guide for future modules

---

## Source Quality

- **16 sources reviewed** (official docs, registries, working examples)
- **500+ real MCP servers analyzed** for patterns
- **Confidence level:** HIGH (85%)
- **Cross-referenced** across npm, GitHub, MCP registry, LaTeX ecosystem

---

**Full analysis in:** `/sessions/blissful-eloquent-gates/mnt/ZenflowProjects/ZenithScience/research/2026-02-17_naming-research.md`

**Ready for:** Stakeholder review, trademark validation, GitHub org creation
