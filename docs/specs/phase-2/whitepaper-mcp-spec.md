# whitepaper-mcp Spec: Policy & Research Whitepapers

**Author:** ZenSci Architecture (TresPiesDesign.com)
**Status:** Draft
**Created:** 2026-02-18
**Phase:** 2 (ships after Phase 1 core is validated)
**Grounded In:** Module strategy scout (2026-02-17), ZenSci semantic clusters

---

## 1. Vision
> Convert research Markdown into publication-ready PDF whitepapers (formal, structured, policy-focused) for think tanks, policy organizations, and academic research centers.

### The Core Insight
Policy researchers publish whitepapers with strict structure: **overview, background, analysis, recommendations, bibliography**—but these are formatted manually to institutional standards. whitepaper-mcp auto-generates compliant PDFs with proper section styling, executive summaries, and policy-specific formatting (sidebars, callout boxes, recommendations tables).

Unlike blog-mcp (narrative-friendly HTML), whitepaper-mcp targets PDF output for institutional credibility, printability, and archival. Whitepapers are **data-heavy**: tables, citations, policy recommendations—all must be formatted consistently.

### What Makes This Different
1. **Structured sections:** Overview, background, analysis, recommendations, appendices—auto-formatted with consistent styling.
2. **Policy-specific features:** Recommendation callout boxes, policy tables, citation-heavy footnotes.
3. **Executive summary synthesis:** Auto-generate executive summary from structured metadata.
4. **Institutional branding:** Logo and header integration; institutional styling.
5. **Citation depth:** Full bibliography with cross-references; footnote-style citations in policy context.

---

## 2. Goals & Success Criteria

### Primary Goals
1. Convert Markdown whitepaper → PDF with proper section styling.
2. Auto-generate executive summary from frontmatter metadata.
3. Support policy tables, recommendations callout boxes, and structured sidebars.
4. Integrate comprehensive bibliography with footnote citations.
5. Support institutional branding (logos, colors, header/footer).
6. Validate structure (all sections present, page limits respected).

### Success Criteria
✅ 30-page whitepaper with 50+ citations converts in <8s.
✅ Executive summary is auto-generated and coherent.
✅ Policy recommendations are clearly highlighted and formatted.
✅ PDF is print-ready (no layout issues, proper margins).
✅ Bibliography is comprehensive and properly formatted.
✅ End-to-end test: Whitepaper (5 sections, 50 citations, 10 recommendations) → PDF in <8s.

### Non-Goals
❌ HTML whitepaper output (PDF only).
❌ Interactive policy comparison tools.
❌ Real-time policy database integration.

---

## 3. Technical Architecture

### 3.1 System Overview

```
Markdown + YAML Metadata
    ↓
whitepaper-mcp MCP Server (TypeScript)
    ├─ Parse sections and metadata
    ├─ Auto-generate executive summary
    ├─ Route to Python engine
    └─ Return DocumentResponse (PDF)
    ↓
Python Engine (pandoc + TeX)
    ├─ Assemble sections with styling
    ├─ Generate policy callout boxes
    ├─ Compile LaTeX → PDF (xelatex)
    └─ Return bytes (PDF)
    ↓
DocumentResponse (format: 'whitepaper-pdf', artifacts: [executive summary, bibliography])
```

### 3.2 MCP Tool Definitions

#### Tool: `convert_to_whitepaper`
**Description:** Convert Markdown whitepaper to PDF.

**Input Schema:**
```json
{
  "type": "object",
  "properties": {
    "source": {
      "type": "string",
      "description": "Markdown content (## Overview, ## Background, ## Analysis, ## Recommendations)"
    },
    "metadata": {
      "type": "object",
      "properties": {
        "title": { "type": "string" },
        "authors": { "type": "array", "items": { "type": "string" } },
        "institution": { "type": "string" },
        "date": { "type": "string", "format": "date" },
        "abstract": { "type": "string" },
        "key_findings": { "type": "array", "items": { "type": "string" }, "description": "3-5 bullet points" },
        "recommendations": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "number": { "type": "integer" },
              "title": { "type": "string" },
              "description": { "type": "string" }
            }
          }
        },
        "bibliography": { "type": "string", "description": "CSL-JSON or BibTeX file path" },
        "branding": {
          "type": "object",
          "properties": {
            "logo_path": { "type": "string" },
            "color_primary": { "type": "string", "description": "Hex color code" },
            "color_accent": { "type": "string" }
          }
        }
      },
      "required": ["title", "authors", "institution", "date"]
    }
  },
  "required": ["source", "metadata"]
}
```

**Output Schema:**
```json
{
  "type": "object",
  "properties": {
    "id": { "type": "string" },
    "format": { "type": "string" },
    "content": { "type": "string", "description": "Base64-encoded PDF" },
    "artifacts": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "type": { "type": "string", "enum": ["executive_summary", "bibliography", "recommendations_table"] },
          "filename": { "type": "string" },
          "content": { "type": "string" }
        }
      }
    },
    "metadata": {
      "type": "object",
      "properties": {
        "page_count": { "type": "integer" },
        "section_count": { "type": "integer" },
        "citation_count": { "type": "integer" },
        "recommendation_count": { "type": "integer" }
      }
    },
    "elapsed": { "type": "number" }
  }
}
```

#### Tool: `validate_whitepaper`
**Description:** Validate whitepaper structure and content.

**Input Schema:**
```json
{
  "type": "object",
  "properties": {
    "source": { "type": "string" },
    "metadata": { "type": "object" },
    "strict_mode": { "type": "boolean" }
  },
  "required": ["source", "metadata"]
}
```

**Output Schema:**
```json
{
  "type": "object",
  "properties": {
    "valid": { "type": "boolean" },
    "errors": { "type": "array", "items": { "type": "object" } },
    "warnings": { "type": "array" },
    "structure_report": {
      "type": "object",
      "properties": {
        "has_overview": { "type": "boolean" },
        "has_background": { "type": "boolean" },
        "has_analysis": { "type": "boolean" },
        "has_recommendations": { "type": "boolean" },
        "citation_count": { "type": "integer" },
        "recommendation_count": { "type": "integer" }
      }
    }
  }
}
```

### 3.3 Core Integration Points

- **latex-mcp:** TeX compilation pipeline; PDF generation.
- **packages/core:** Bibliography system (CSL-JSON).

---

### 3.4 TypeScript Implementation

```typescript
// zen-sci/packages/sdk/src/servers/whitepaper-mcp/index.ts

import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from '@modelcontextprotocol/sdk/types.js';

const server = new Server(
  { name: 'whitepaper-mcp', version: '0.1.0' },
  { capabilities: { tools: {} } }
);

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: 'convert_to_whitepaper',
      description: 'Convert Markdown whitepaper to PDF.',
      inputSchema: {
        type: 'object' as const,
        properties: {
          source: { type: 'string' },
          metadata: { type: 'object' },
        },
        required: ['source', 'metadata'],
      },
    },
    {
      name: 'validate_whitepaper',
      description: 'Validate whitepaper structure.',
      inputSchema: {
        type: 'object' as const,
        properties: {
          source: { type: 'string' },
          metadata: { type: 'object' },
          strict_mode: { type: 'boolean' },
        },
        required: ['source', 'metadata'],
      },
    },
  ],
}));

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request;

  if (name === 'convert_to_whitepaper') {
    const result = await convertToWhitepaper(args as any);
    return { content: [{ type: 'text', text: JSON.stringify(result) }] };
  }

  if (name === 'validate_whitepaper') {
    const result = await validateWhitepaper(args as any);
    return { content: [{ type: 'text', text: JSON.stringify(result) }] };
  }

  throw new Error(`Unknown tool: ${name}`);
});

async function convertToWhitepaper(args: any) {
  return await callPythonEngine('convert_whitepaper', args);
}

async function validateWhitepaper(args: any) {
  return await callPythonEngine('validate_whitepaper', args);
}

async function callPythonEngine(method: string, params: any) {
  // Spawn Python subprocess
}

const transport = new StdioServerTransport();
server.connect(transport);
```

### 3.5 Python Processing Engine

```python
# zen-sci/servers/whitepaper-mcp/processing/whitepaper.py

import subprocess
import tempfile
from pathlib import Path
from typing import Dict, List, Any, Tuple

import pandoc

async def generate_whitepaper_pdf(
    source: str,
    metadata: Dict[str, Any],
    bibliography: str = None,
) -> Tuple[bytes, Dict[str, str]]:
    """Generate whitepaper PDF."""

    # Step 1: Auto-generate executive summary
    exec_summary = generate_executive_summary(metadata)

    # Step 2: Build LaTeX with executive summary + sections + recommendations callout
    full_content = f"""# Executive Summary

{exec_summary}

{source}

# Recommendations

"""
    for rec in metadata.get('recommendations', []):
        full_content += f"\\\\fcolorbox{{lightgray}}{{white}}{{{rec['title']}}}\n{rec['description']}\n\n"

    # Step 3: Convert to LaTeX with pandoc
    pandoc_args = [
        '-f', 'markdown',
        '-t', 'latex',
        '--template=templates/whitepaper.tex',
        '--number-sections',
        '-V', 'documentclass:article',
        '-V', 'geometry:margin=1in',
    ]

    if bibliography:
        pandoc_args.extend([
            '--citeproc',
            '--bibliography=/dev/stdin',
            '--csl=styles/chicago-notes.csl',
        ])

    latex_content = pandoc.convert_text(
        full_content,
        format='markdown',
        extra_args=pandoc_args,
        standalone=True,
    )

    # Step 4: Compile to PDF
    with tempfile.TemporaryDirectory() as tmpdir:
        tex_file = Path(tmpdir) / 'whitepaper.tex'
        pdf_file = Path(tmpdir) / 'whitepaper.pdf'

        tex_file.write_text(latex_content, encoding='utf-8')

        result = subprocess.run(
            ['xelatex', '-interaction=nonstopmode', '-output-directory', tmpdir, 'whitepaper.tex'],
            cwd=tmpdir,
            capture_output=True,
            text=True,
        )

        if not pdf_file.exists():
            raise RuntimeError('PDF generation failed')

        pdf_bytes = pdf_file.read_bytes()

    artifacts = {
        'executive_summary': exec_summary,
    }

    return pdf_bytes, artifacts


def generate_executive_summary(metadata: Dict[str, Any]) -> str:
    """Auto-generate executive summary from metadata."""
    lines = []

    if metadata.get('abstract'):
        lines.append(metadata['abstract'])
        lines.append('')

    if metadata.get('key_findings'):
        lines.append('**Key Findings:**')
        lines.append('')
        for finding in metadata['key_findings']:
            lines.append(f"- {finding}")
        lines.append('')

    return '\n'.join(lines)
```

### 3.6 Key Schemas

```typescript
// zen-sci/packages/core/src/schemas/whitepaper.ts

export interface WhitepaperMetadata {
  title: string;
  authors: string[];
  institution: string;
  date: string; // ISO date
  abstract: string;
  key_findings: string[];
  recommendations: PolicyRecommendation[];
  bibliography?: string;
  branding?: BrandingInfo;
}

export interface PolicyRecommendation {
  number: number;
  title: string;
  description: string;
}

export interface BrandingInfo {
  logo_path?: string;
  color_primary?: string;
  color_accent?: string;
}

export interface WhitepaperDocumentRequest extends DocumentRequest {
  format: 'whitepaper-pdf';
  source: string;
  metadata: WhitepaperMetadata;
}

export interface WhitepaperResponse extends DocumentResponse {
  format: 'whitepaper-pdf';
  content: Buffer;
  artifacts: WhitepaperArtifact[];
}

export interface WhitepaperArtifact extends Artifact {
  type: 'executive_summary' | 'bibliography' | 'recommendations_table';
}
```

---

### 3.7 Phase 2 Infrastructure Requirements

- Policy table/callout box LaTeX templates.
- Whitepaper-specific CSS styling (if future HTML support added).

---

### 3.8 Performance & Constraints

| Constraint | Target | Notes |
|-----------|--------|-------|
| 30-page whitepaper (50+ citations) | <8s | xelatex compilation |
| Executive summary generation | <500ms | Template rendering |
| Recommendations callout generation | <1s | Markdown → LaTeX |

---

## 4. Implementation Plan

### 4.1 Phased Approach

| Phase | Timeline | Deliverables | Dependencies |
|-------|----------|--------------|--------------|
| **Alpha (Week 1-2)** | 2 weeks | Basic PDF generation; no executive summary | packages/core v1.0, latex-mcp v0.1 |
| **Beta (Week 3-4)** | 2 weeks | + Executive summary synthesis; recommendations callout boxes | All Alpha |
| **Candidate (Week 5-6)** | 2 weeks | + Full validation; institutional branding | All Beta |
| **Release (Week 7)** | 1 week | Full test suite; documentation | All Candidate |

### 4.2 Week-by-Week Breakdown

**Week 1: Basic PDF Generation**
- [ ] Set up zen-sci/servers/whitepaper-mcp/ directory structure
- [ ] Implement MCP server skeleton
- [ ] Create whitepaper LaTeX template
- [ ] Implement pandoc Markdown → LaTeX conversion
- [ ] Implement xelatex compilation pipeline
- [ ] Unit tests: LaTeX generation; PDF compilation
- [ ] Integration test: Basic whitepaper → PDF

**Week 2: Executive Summary & Metadata Handling**
- [ ] Implement executive summary auto-generation from metadata
- [ ] Add key findings section
- [ ] Implement author and institution metadata handling
- [ ] Implement `convert_to_whitepaper` tool
- [ ] Unit tests: executive summary generation; metadata parsing
- [ ] Integration test: Whitepaper with metadata → PDF

**Week 3: Recommendations & Callout Boxes**
- [ ] Implement policy recommendation callout box generation
- [ ] Add styled recommendation tables
- [ ] Create recommendation section template
- [ ] Unit tests: callout box generation; table styling
- [ ] Integration test: Whitepaper with recommendations → PDF

**Week 4: Validation & Bibliography**
- [ ] Implement `validate_whitepaper` tool
- [ ] Add structure validation (required sections present)
- [ ] Integrate CSL-JSON bibliography
- [ ] Unit tests: validation logic; bibliography formatting
- [ ] Integration test: Full whitepaper (50 citations) → PDF

**Week 5: Institutional Branding**
- [ ] Add logo support (header/footer)
- [ ] Add color customization
- [ ] Implement branding metadata handling
- [ ] Unit tests: logo embedding; color parsing
- [ ] Integration test: Branded whitepaper → PDF

**Week 6: Edge Cases & Polish**
- [ ] Handle long titles, author lists
- [ ] Test citation-heavy whitepapers (80+ citations)
- [ ] Test large recommendation lists (20+ recommendations)
- [ ] Unit tests: edge cases
- [ ] Integration test: Large-scale whitepaper → PDF

**Week 7: Documentation & Release**
- [ ] Write API documentation
- [ ] Create example whitepapers (policy, research)
- [ ] Write Python processing engine docs
- [ ] Finalize error messages
- [ ] Publish v0.1.0 to npm (@zen-sci/whitepaper-mcp)
- [ ] Full regression test suite (25+ unit + 8+ integration tests)

### 4.3 Dependencies & Prerequisites

**Hard Dependencies:**
- packages/core v1.0
- latex-mcp v0.1

---

### 4.4 Testing Strategy

**Unit Tests:** Executive summary generation, recommendation callout creation, bibliography formatting, validation logic.

**Integration Tests:** Full whitepapers with citations, recommendations, branding; structure validation.

---

## 5. Risk Assessment & Mitigation

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| LaTeX callout box rendering issues | Low | Medium | Pre-test templates; use proven LaTeX packages |
| Executive summary oversimplification | Medium | Low | Template-driven; allow manual override |

---

## 6. Rollback & Contingency

**If executive summary auto-generation is unsatisfactory (Week 2):**
- Make it optional; allow manual-only mode; fix in v0.1.1.

---

## 7. Appendices

### 7.1 Related Work & Standards

- **Think tank white papers:** Common format for policy organizations.
- **Chicago Manual of Style:** Citation format for policy documents.
- **LaTeX fcolorbox:** Callout box package.

### 7.2 Future Considerations (v1.1+)

- **HTML whitepaper output:** Web-friendly version.
- **Interactive recommendations:** Policy comparison UI.
- **Institutional template library:** Pre-built templates for common think tanks.

---

**End of whitepaper-mcp Specification**

---

## Audit Notes (2026-02-18)

**Auditor:** Claude (Opus-level audit)
**Status:** Reviewed

### Issues
- **CRITICAL-2:** Instantiates `Server` directly instead of extending `ZenSciServer`.
- **CRITICAL-3:** Uses low-level `setRequestHandler` pattern.
- MCP SDK imports correct. Tool naming snake_case — consistent.
- Shorter spec but adequate. Executive summary auto-generation is a good differentiator vs policy-brief-mcp.

#### Decision Propagation (Cruz, 2026-02-18)
- **CRITICAL-2 resolved:** `ZenSciServer` abstract base class deprecated. Use `createZenSciServer()` factory + `McpServer.registerTool()` with Zod schemas at implementation time.
- **CRITICAL-3 resolved:** Low-level `setRequestHandler` replaced by `McpServer.registerTool()` with Zod input schemas per MCP SDK v2.
