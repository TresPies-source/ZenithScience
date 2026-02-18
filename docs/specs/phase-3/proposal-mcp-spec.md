# proposal-mcp Spec: Business & Product Proposal Generator

**Author:** ZenSci Architecture (TresPiesDesign.com)
**Status:** Draft — Ship on Demand
**Created:** 2026-02-18
**Phase:** 3 (lower priority; trigger-based shipping)
**Grounded In:** Module strategy scout (2026-02-17)

---

## 1. Vision

Transform business proposal outlines into polished PDF or DOCX documents with consistent branding, timeline visualization, budget breakdowns, and call-to-action formatting—enabling sales teams and product managers to move from markdown sketch to investable proposal in minutes.

### The Core Insight

Proposals are a universal artifact across startups, consulting firms, and corporate innovation teams. Yet teams still spend disproportionate effort on formatting: timeline Gantt charts, budget tables, signature blocks. Unlike technical papers (which demand citation and mathematical rigor), proposals prioritize *persuasion and clarity*. A ZenSci module captures the data structure (problem → solution → timeline → budget → team → CTA) and renders it with subtle persuasive design (color, hierarchy, call-to-action prominence).

### What Makes This Different

Unlike whitepaper-mcp (which is research-focused) or policy-brief-mcp (which is analysis-focused), proposal-mcp optimizes for *salesmanship* while maintaining data integrity. Timeline visualization (Gantt chart or milestone list), budget breakdown (with confidence levels), and explicit CTA placement are first-class features. The module treats proposals as persuasive data documents, not essays.

---

## 2. Goals & Success Criteria

### Primary Goals

1. **Eliminate proposal formatting overhead** — markdown outline → PDF/DOCX with timelines, budgets, and CTA in <60 seconds
2. **Enable non-designers to author investable proposals** — consistent typography, color hierarchy, and visual emphasis without manual tweaking
3. **Support multiple proposal types** — product pitch, service proposal, grant proposal, partnership pitch

### Success Criteria

✅ Can convert 75% of real sales proposals (from SaaS, consulting, and service firms) without manual design intervention
✅ Timeline visualization (Gantt or milestone list) renders correctly in PDF and DOCX
✅ Budget breakdowns with confidence/discount levels update correctly across all sections
✅ CTA section prominence and styling pass design review from 3 external proposal experts

### Non-Goals

❌ Generate proposal content algorithmically (human writes problem statement and solution)
❌ Integrate with CRM systems for automatic client data population (out of scope)
❌ Support video or interactive elements (PDF/DOCX static format limitation)
❌ Multi-signature workflows (external signature systems own that problem)

---

## 3. Strategic Fit Assessment

**Why Phase 3 (not Phase 1 or 2):**
Proposals are a *commoditized* domain—many tools exist (Proposify, PandaDoc, Qwilr). ZenSci's differentiation is markdown + citation + PDF precision, which matters for technical proposals but less so for business ones. Until we have SaaS or consulting firms explicitly requesting our module for proposal+research-paper integration (e.g., "proposal + technical appendix"), this is speculative. Additionally, Word/DOCX output requires a different rendering pipeline than LaTeX, adding complexity without clear ROI.

**Market Trigger Conditions:**

- **Trigger 1:** A SaaS company or consulting firm requests proposal-mcp as part of a partnership, explicitly citing ZenSci's markdown + research integration
- **Trigger 2:** GitHub issues/email requests for proposal templating exceed 8+ over 3 months
- **Trigger 3:** A pilot shows that integrating technical proposal + research paper (e.g., AI startup pitch + academic whitepaper appendix) is genuinely valuable vs. using two separate tools

**Risk of NOT shipping:**
**Low.** The proposal market is well-served by competitors. If we don't ship, no credible risk to ZenSci's core mission (research & technical documentation). However, if shipping, the feature could attract SaaS/startup users who value the markdown + research integration story.

---

## 4. Technical Architecture (simplified)

### 4.1 MCP Tools

**`convertProposal`**
- **Input:** `DocumentRequest` (source markdown, output format: 'proposal')
- **Output:** `DocumentResponse` (PDF or DOCX with formatted sections, timelines, budget)
- **Processing:** Parse frontmatter → validate proposal structure → extract timeline → render budget breakdown → compile to PDF/DOCX

**`validateProposalStructure`**
- **Input:** markdown source
- **Output:** `ValidationResult` with required sections checklist (executive summary, problem, solution, timeline, budget, team, CTA)

**`renderTimeline`**
- **Input:** proposal markdown with `[milestone]` blocks
- **Output:** Gantt chart or milestone list (SVG or table) ready for embedding in PDF/DOCX

**`budgetBreakdown`**
- **Input:** proposal markdown with `[budget]` blocks containing line items + confidence/discount
- **Output:** formatted budget table with subtotals, discounts, final total; supports CSV export

### 4.2 Infrastructure Reuse

- **MarkdownParser** — parse frontmatter, section hierarchy, and proposal-specific blocks (`[timeline]`, `[budget]`)
- **SchemaValidator** — validate `DocumentRequest` against proposal schema
- **CitationManager** — optional: if proposal includes research citations, apply CSL formatting
- **Pandoc Python engine** — LaTeX → PDF; python-docx or similar for DOCX output

### 4.3 Novel Infrastructure Required

**Proposal Schema (`ProposalRequest`)**
```typescript
interface ProposalRequest extends DocumentRequest {
  format: 'proposal';
  moduleOptions: {
    proposalType: 'product-pitch' | 'service-proposal' | 'grant-proposal' | 'partnership-pitch';
    timelineStyle: 'gantt' | 'milestone-list';
    includeBudgetBreakdown: boolean;
    currencyCode?: string; // default 'USD'
    colorScheme?: 'default' | 'dark' | 'vibrant'; // basic theming
  };
}
```

**Timeline Block Parser & Renderer**
```typescript
interface TimelineBlock {
  type: 'timeline';
  milestones: {
    name: string;
    date: string; // YYYY-MM-DD
    dependencies?: string[];
    owner?: string;
  }[];
}
```

**Budget Block Parser & Renderer**
```typescript
interface BudgetBlock {
  type: 'budget';
  rows: {
    category: string;
    amount: number;
    confidence?: number; // 0-1, for sensitivity
    notes?: string;
  }[];
  discountPercent?: number;
}
```

**DOCX Renderer** — adapter for python-docx (simpler than LaTeX but less control) or use Pandoc's native DOCX support

### 4.4 Key Schemas

```typescript
// Extends DocumentNode union
type ProposalNode = DocumentNode | {
  type: 'timeline';
  milestones: TimelineMilestone[];
} | {
  type: 'budget';
  rows: BudgetRow[];
  total: number;
  discountPercent?: number;
} | {
  type: 'team';
  members: TeamMember[];
} | {
  type: 'cta';
  text: string;
  deadline?: string;
  contactEmail?: string;
};

interface TimelineMilestone {
  name: string;
  date: string;
  dependencies?: string[];
  owner?: string;
}

interface BudgetRow {
  category: string;
  amount: number;
  confidence: number; // 0-1
  notes?: string;
}

interface TeamMember {
  name: string;
  role: string;
  bio?: string;
  image?: string;
}
```

---

## 5. Implementation Estimate

| Track | Duration (weeks) | Dependencies |
|-------|---------|-------------|
| Schema + types | 1 | Phase 1 core (DocumentRequest) |
| Timeline parser & renderer (SVG/Gantt) | 1.5 | MarkdownParser; D3.js or canvas library |
| Budget block parser & renderer | 1 | MarkdownParser; table layout |
| DOCX output adapter | 1.5 | Pandoc DOCX or python-docx library |
| Proposal layout templates (PDF & DOCX) | 1.5 | Whitepaper layout base; design review |
| MCP tool registration & testing | 1 | packages/sdk |
| **Total** | **8** | whitepaper-mcp (layout templates); Pandoc DOCX support |

---

## 6. Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| DOCX output quality significantly lower than PDF | Medium | High | Test Pandoc DOCX render early (week 1); if poor, defer DOCX to Phase 4 and ship PDF-only |
| Timeline Gantt rendering breaks on complex dependencies | Low | Medium | Start with simple milestone list; Gantt as future enhancement; test with 10+ real timelines |
| Budget sensitivity analysis (confidence levels) adds complexity without user demand | Medium | Low | Defer to Phase 4; ship simple budget breakdown in Phase 3 |
| Design review feedback forces template overhaul mid-implementation | Medium | Medium | Get design buy-in early (week 1); establish design review checklist before coding |
| Proposal domain too commoditized; users prefer dedicated tools | Medium | Low | This is acceptable; we're not betting the company on proposals. Ship if triggers align; otherwise deprioritize |

---

## 7. Open Questions & Future Considerations

1. **Branding & white-label:** Should proposal-mcp support custom logos, color schemes, and fonts, or is basic theming sufficient for Phase 3?

2. **Multi-currency & localization:** Should budgets support multiple currencies or locale-specific number formatting (e.g., 1.000,00 for European format)?

3. **Approval workflows:** Could we integrate with simple approval tracking (e.g., status badges like "Draft → Reviewed → Final")?

4. **Template library:** Should we pre-ship 5–10 proposal templates (SaaS pitch, consulting engagement, grant proposal, partnership, vendor RFP response), or let users build from markdown structure?

5. **Variant: RFP response generator:** Should proposal-mcp also support RFP (Request for Proposal) response formatting, or is that a separate module?

6. **Integration with payment/contract systems:** Future: could proposal PDFs embed QR codes linking to contract signing or payment pages?

---

## Audit Notes (2026-02-18)

**Auditor:** Claude (Opus-level audit)
**Status:** Reviewed

### Issues
- **HIGH-1:** Tool names use camelCase (`convertProposal`, `validateProposalStructure`, `renderTimeline`, `budgetBreakdown`). Should be snake_case: `convert_proposal`, `validate_proposal_structure`, `render_timeline`, `budget_breakdown`.
- **ADVISORY-2:** Uses `moduleOptions` field — consistent with other Phase 3 specs but differs from Phase 2 pattern.
- Strategic Fit Assessment correctly identifies this as commoditized market.
