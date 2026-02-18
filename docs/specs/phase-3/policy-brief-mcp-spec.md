# policy-brief-mcp Spec: Think Tank & Policy Brief Publishing Engine

**Author:** ZenSci Architecture (TresPiesDesign.com)
**Status:** Draft — Ship on Demand
**Created:** 2026-02-18
**Phase:** 3 (lower priority; trigger-based shipping)
**Grounded In:** Module strategy scout (2026-02-17)

---

## 1. Vision

Enable policy researchers and think tanks to author evidence-based policy briefs with structured analysis, formal citations, and executive summary extraction—then publish as publication-grade PDF, HTML, or markdown.

### The Core Insight

Policy briefs occupy a distinct middle ground between research papers and whitepapers: they demand rigorous citation and data grounding but must communicate to non-expert stakeholders. The target audience (think tanks, government research bureaus, NGOs) already uses markdown or Word for drafting, but conversion to polished PDF or web-published HTML is manual and error-prone. A ZenSci module here captures analysis rigor while eliminating layout friction.

### What Makes This Different

Unlike the **whitepaper-mcp** (which emphasizes narrative flow and business framing), policy-brief-mcp enforces structured sections (executive summary → issue analysis → policy options → recommendation → citations). It integrates cost-benefit analysis tables, policy trade-off matrices, and cross-reference citation numbering. The output template supports multi-jurisdiction briefs and acknowledges "dissenting view" sections for transparency.

---

## 2. Goals & Success Criteria

### Primary Goals

1. **Enable rapid policy brief authoring** — markdown input → structured sections → publication-ready PDF/HTML in <30 seconds
2. **Enforce evidence grounding** — automatic citation validation, missing reference detection, and CSL style application
3. **Support policy-specific layouts** — executive summary extraction, policy option matrices, cost-benefit tables with proper formatting

### Success Criteria

✅ Can convert 80% of real think-tank briefs (samples from Brookings, Center for Strategic Studies) without manual layout intervention
✅ Policy option matrices render identically in PDF and HTML
✅ Citation validation catches 95% of orphaned references and malformed BibTeX entries
✅ Executive summary auto-extraction achieves 85%+ accuracy on unseen briefs

### Non-Goals

❌ Generate policy recommendations algorithmically (human analyst writes these)
❌ Support legislative drafting or statute formatting (separate domain)
❌ Integrate with government legislative databases (out of scope)
❌ Real-time multi-user collaborative editing (use Google Docs for drafting)

---

## 3. Strategic Fit Assessment

**Why Phase 3 (not Phase 1 or 2):**
Policy briefs are a specialized domain with less general-purpose demand than research papers, lab notebooks, or email. They require partnership validation with think tanks to confirm that the structured template approach (executive summary → analysis → policy options → recommendation) actually reduces author friction. Until we have 3+ think tank pilots expressing demand, this is speculative infrastructure.

**Market Trigger Conditions:**

- **Trigger 1:** A established think tank (Brookings, RAND, CSIS, or equivalent) explicitly requests policy-brief-mcp as part of a partnership engagement
- **Trigger 2:** Unsolicited user requests (via GitHub issues or email) for policy brief templating exceed 10+ over a 3-month period
- **Trigger 3:** Partner organizations send sample briefs for format validation and report >80% automation potential

**Risk of NOT shipping:**
**Medium.** Policy research is a significant downstream market (thousands of analysts globally), but without partner validation, we risk building for a use case that doesn't exist. If we do ship on demand and miss these triggers, the module will languish unmaintained, damaging trust.

---

## 4. Technical Architecture (simplified)

### 4.1 MCP Tools

**`convertPolicyBrief`**
- **Input:** `DocumentRequest` (source markdown, output format: 'policy-brief')
- **Output:** `DocumentResponse` (PDF or HTML with structured sections)
- **Processing:** Parse frontmatter (title, authors, agency, date) → validate sections → extract executive summary → apply CSL citations → render

**`validatePolicyBriefStructure`**
- **Input:** markdown source
- **Output:** `ValidationResult` with required sections checklist and warnings (e.g., missing recommendation section, orphaned citations)

**`extractExecutiveSummary`**
- **Input:** full brief markdown
- **Output:** auto-generated 150–200 word summary + metadata (key findings, policy options identified)

### 4.2 Infrastructure Reuse

- **MarkdownParser** — parse frontmatter and section hierarchy
- **CitationManager** — resolve `[@key]` references; apply CSL styles (especially Chicago/APA for policy)
- **SchemaValidator** — validate `DocumentRequest` against policy-brief schema
- **Pandoc Python engine** — LaTeX → PDF compilation; Beamer-style headers for PDF bookmarks

### 4.3 Novel Infrastructure Required

**Policy Brief Schema (`PolicyBriefRequest`)**
```typescript
interface PolicyBriefRequest extends DocumentRequest {
  format: 'policy-brief';
  moduleOptions: {
    briefType: 'domestic' | 'international' | 'multi-jurisdiction';
    includeDissent: boolean;
    includeCostBenefit: boolean;
    targetAudience?: string;
    executiveSummaryLength?: number; // words, default 150
  };
}
```

**Policy Option Matrix Renderer** — parse `| policy-option | pros | cons | estimated-cost |` markdown tables and render as formatted sidebars in PDF/HTML

**Executive Summary Extractor** — LLM-free heuristic (extracts first N paragraphs of analysis + one-liner per policy option) to scaffold author review

### 4.4 Key Schemas

```typescript
// Extends DocumentNode union
type PolicyBriefNode = DocumentNode | {
  type: 'policy-option-matrix';
  options: { name: string; pros: string[]; cons: string[]; cost?: string }[];
} | {
  type: 'cost-benefit-table';
  rows: { program: string; benefit: string; cost: string; roi?: string }[];
};

interface ExecutiveSummaryMetadata {
  keyFindings: string[];
  policyOptionsIdentified: string[];
  recommendation: string;
  confidence: 'high' | 'medium' | 'low';
}
```

---

## 5. Implementation Estimate

| Track | Duration (weeks) | Dependencies |
|-------|---------|-------------|
| Schema + types | 1 | Phase 1 core (DocumentRequest, ValidationResult) |
| Policy option matrix renderer | 2 | Pandoc TeX templates, whitepaper-mcp layout base |
| Executive summary extractor | 1.5 | MarkdownParser, section detection |
| MCP tool registration | 1 | packages/sdk (`createZenSciServer()` factory) |
| Testing + think tank validation | 2 | Sample briefs from partner (if available) |
| **Total** | **7.5** | whitepaper-mcp (reusable PDF templates) |

---

## 6. Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| Think tank partners reject structured format (prefer freeform) | Medium | High | Start with partner interviews (1 week) before implementation; offer both flexible + structured modes |
| Executive summary extraction quality too low for publication | Medium | Medium | Use heuristic approach initially; mark as "author-generated scaffold" requiring editorial review |
| CSL citation formatting breaks on edge cases (uncommon types) | Low | Medium | Comprehensive BibTeX test suite from policy domain; fallback to raw bibliography |
| Policy option matrix rendering misaligns in PDF vs HTML | Medium | Low | Use CSS grid + LaTeX table abstraction; test on 20+ real matrices from partner briefs |

---

## 7. Open Questions & Future Considerations

1. **Co-branding & attribution:** Should policy briefs generated by ZenSci include a footer attribution, or is that a distraction for policy organizations wanting to publish under institutional letterhead?

2. **Dissenting view integration:** Should we auto-detect and segregate "however, alternative view:" language into a formal dissent section, or leave this to author discretion?

3. **Legislative & cost analysis APIs:** Down the road, could integrate with USAspending.gov or equivalent to auto-populate cost estimates for federal programs?

4. **Variant: International policy briefs:** Should we support multi-language briefs (e.g., English + French for Canadian policy context)?

5. **Accessibility in PDF:** Do policy briefs have specific WCAG compliance requirements when distributed as institutional PDFs?

---

## Audit Notes (2026-02-18)

**Auditor:** Claude (Opus-level audit)
**Status:** Reviewed

### Issues
- **HIGH-1:** Tool names use camelCase (`convertPolicyBrief`, `validatePolicyBriefStructure`, `extractExecutiveSummary`). Should be snake_case for MCP convention consistency: `convert_policy_brief`, `validate_policy_brief_structure`, `extract_executive_summary`.
- **ADVISORY-2:** Uses `moduleOptions` field for module-specific options. Phase 2 specs put options in `metadata`. Standardize on `moduleOptions` (cleaner separation).
- Strategic Fit Assessment is excellent — clear trigger conditions and "risk of NOT shipping" analysis.
- Appropriately lighter spec for Phase 3.
