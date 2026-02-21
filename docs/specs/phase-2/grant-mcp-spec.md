# grant-mcp Spec: Funder-Compliant Research Proposals

**Author:** ZenSci Architecture (TresPiesDesign.com)
**Status:** Draft
**Created:** 2026-02-18
**Phase:** 2 (ships after Phase 1 core is validated)
**Grounded In:** Module strategy scout (2026-02-17), ZenSci semantic clusters

---

## 1. Vision
> Automatically generate NIH, NSF, and ERC grant proposals in LaTeX + Word from structured Markdown, enforcing funder-specific formatting, budget rules, and compliance gates.

### The Core Insight
Grant writers spend 40% of proposal time on formatting compliance: page limits, margin rules, font sizes, specific aims structure, budget table formats, and institutional compliance checks. Yet the **scientific content is identical** across formats. grant-mcp unifies the source (Markdown with funder-aware YAML config), auto-generates compliant PDFs and Word files, and validates against known funder rules in real-time.

Unlike generic template services, grant-mcp integrates with ZenSci's math and bibliography systems, understands research semantics (specific aims, objectives, hypotheses), and outputs both TeX (for typographic control) and .docx (for institutional review workflows).

### What Makes This Different
1. **Multi-funder architecture:** Single source → NIH, NSF, ERC outputs with funder-specific formatting, budget rules, compliance checks.
2. **Compliance gates:** Real-time validation of page limits, font sizes, margin requirements; warnings on common rejection reasons.
3. **Budget synthesis:** Auto-generate budget justifications from project activities; cross-link budget to specific aims.
4. **Structured specific aims:** Parse objectives → outcomes → impact; ensure each aim has quantified metrics and timelines.
5. **Institutional data integration:** Read institutional cost-sharing rules, overhead rates, and auto-populate budget accordingly.
6. **Dual-format parity:** PDF and Word outputs have identical scientific content; Word preserves editable structure for institutional review.

---

## 2. Goals & Success Criteria

### Primary Goals
1. Accept Markdown proposal outline with YAML funder config (NIH, NSF, ERC); output compliant PDF + Word.
2. Validate page limits, fonts, margins, and budget structure; fail early on non-compliance.
3. Generate Specific Aims page with auto-formatted objectives, outcomes, and impact statements.
4. Auto-generate budget justifications from project activities + institutional overhead rules.
5. Support structured timeline (Gantt chart or simple table) with activity-to-aim mapping.
6. Integrate with packages/core bibliography system; enforce citation style per funder.

### Success Criteria
✅ NIH R01 proposal (12 pages, specific aims + research plan + budget) converts in <10s; PDF and Word are identical except for editability.
✅ Specific Aims page is auto-generated from YAML objectives; validation catches missing timelines or unmapped activities.
✅ Budget table auto-populates based on project activities + institutional overhead; generated Word .docx is editable in MS Office.
✅ Validation catches common NIH rejection reasons: font < 10pt, margins < 0.5", budget incoherent with aims.
✅ Support for 3 major funders (NIH, NSF, ERC) with >90% rule coverage in each.
✅ End-to-end test: R01 proposal with 3 specific aims, 10 budget lines, 20 citations → PDF + Word in <10s.

### Non-Goals
❌ Integration with grants.gov or fastlane.nsf.gov upload (post-generation workflow is user responsibility).
❌ Real-time collaboration UI (Claude Code use case is single-user conversion).
❌ Automatic contact with program officers or submission tracking.
❌ Grant prediction (success probability modeling).

---

## 3. Technical Architecture

### 3.1 System Overview

```
Markdown Proposal + YAML Funder Config
    ↓
grant-mcp MCP Server (TypeScript)
    ├─ Parse YAML (funder, specific aims, budget activities, institutional info)
    ├─ Validate against funder rules (page limits, fonts, margins)
    ├─ Route to Python engine
    └─ Return DocumentResponse (PDF + Word + validation report)
    ↓
Python Engine (pandoc + TeX + python-docx)
    ├─ Path A: LaTeX (pandoc → .tex, xelatex → PDF with funder-specific template)
    ├─ Path B: Word (.docx via python-docx library)
    ├─ Both: Budget generation, Specific Aims synthesis, timeline visualization
    └─ Return bytes (PDF + .docx)
    ↓
DocumentResponse (format: 'grant-pdf' | 'grant-docx' | 'grant-both', artifacts: [budget table, specific aims, validation report])
```

### 3.2 MCP Tool Definitions

#### Tool: `generate_proposal`
**Description:** Convert structured grant sections to funder-compliant LaTeX format (NIH, NSF, ERC).

> **Implementation note (2026-02-18):** The original spec used a `source + frontmatter` input model.
> The shipped implementation uses a **sections-array model** — more ergonomic for MCP callers because
> it avoids requiring callers to assemble a YAML frontmatter blob. This is now the canonical contract.

**Input Schema:**
```json
{
  "type": "object",
  "properties": {
    "sections": {
      "type": "array",
      "description": "Ordered proposal sections. Each section has a functional role and markdown content.",
      "items": {
        "type": "object",
        "properties": {
          "role": { "type": "string", "description": "Section role (e.g., specific-aims, research-strategy, budget-justification)" },
          "content": { "type": "string", "description": "Section content in markdown" },
          "title": { "type": "string", "description": "Optional custom section title (overrides role-derived default)" }
        },
        "required": ["role", "content"]
      }
    },
    "funder": { "type": "string", "enum": ["nih", "nsf", "erc"], "description": "Funding agency" },
    "programType": { "type": "string", "description": "Program type (e.g., R01, CAREER, standard)" },
    "bibliography": { "type": "string", "description": "BibTeX bibliography content" },
    "options": {
      "type": "object",
      "properties": {
        "outputFormat": { "type": "string", "enum": ["grant-latex", "docx"] },
        "engine": { "type": "string" }
      }
    }
  },
  "required": ["sections", "funder", "programType"]
}
```

**Output Schema:**
```json
{
  "type": "object",
  "properties": {
    "latex_source": { "type": "string", "description": "Generated LaTeX source" },
    "format": { "type": "string" },
    "compliance": {
      "type": "object",
      "properties": {
        "compliant": { "type": "boolean" },
        "violations": { "type": "array", "items": { "type": "object", "properties": { "severity": { "type": "string" }, "section": { "type": "string" }, "details": { "type": "string" } } } },
        "warnings": { "type": "array", "items": { "type": "object" } },
        "score": { "type": "number" }
      }
    },
    "warnings": { "type": "array", "items": { "type": "string" } },
    "page_counts": { "type": "object", "description": "Map of section roles to estimated page counts" },
    "elapsed_ms": { "type": "number" }
  },
  "required": ["latex_source", "format", "compliance", "warnings", "page_counts", "elapsed_ms"]
}
```

#### Tool: `validate_compliance`
**Description:** Validate grant proposal sections against funder-specific compliance rules.

**Input Schema:**
```json
{
  "type": "object",
  "properties": {
    "sections": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "role": { "type": "string" },
          "content": { "type": "string" },
          "title": { "type": "string" }
        },
        "required": ["role", "content"]
      }
    },
    "funder": { "type": "string", "enum": ["nih", "nsf", "erc"] },
    "programType": { "type": "string" }
  },
  "required": ["sections", "funder", "programType"]
}
```

**Output Schema:**
```json
{
  "type": "object",
  "properties": {
    "compliant": { "type": "boolean" },
    "violations": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "severity": { "type": "string" },
          "section": { "type": "string" },
          "details": { "type": "string" }
        }
      }
    },
    "warnings": { "type": "array", "items": { "type": "object" } },
    "score": { "type": "number" }
  },
  "required": ["compliant", "violations", "warnings", "score"]
}
```

#### Tool: `check_format`
**Description:** Return submission format requirements for a given grant submission platform.

> **Implementation note (2026-02-18):** Original spec named this `list_grant_funders` with an optional
> `funder` filter returning a funder catalog. The shipped tool is named `check_format` and accepts a
> required `platform` enum, returning structured file/page-limit requirements per platform. This is now
> the canonical contract; the funder catalog use-case is a Phase 4 app-layer concern.

**Input Schema:**
```json
{
  "type": "object",
  "properties": {
    "platform": {
      "type": "string",
      "enum": ["grants-gov", "nih-era-commons", "nsf-research-gov", "erc-sep", "manual"],
      "description": "Submission platform"
    }
  },
  "required": ["platform"]
}
```

**Output Schema:**
```json
{
  "type": "object",
  "properties": {
    "platform": { "type": "string" },
    "requiredFiles": { "type": "array", "items": { "type": "string" } },
    "pageLimits": { "type": "object", "description": "Map of file role to page limit" },
    "acceptedFormats": { "type": "array", "items": { "type": "string" } },
    "notes": { "type": "string" }
  },
  "required": ["platform", "requiredFiles", "pageLimits", "acceptedFormats", "notes"]
}
```

### 3.3 Core Integration Points

**packages/core DocumentRequest/DocumentResponse:**
- Input: Markdown + structured YAML frontmatter (extends standard DocumentRequest).
- Output: DocumentResponse with dual artifacts (PDF + Word) + validation report.
- Bibliography system: Delegates to packages/core's unified CSL handler; enforces funder citation styles (e.g., NIH requires numbered citations).

**latex-mcp dependency (v0.1):**
- TeX compilation pipeline (xelatex, including natbib for citations).
- Funder-specific LaTeX templates (NIH, NSF, ERC).

---

### 3.4 TypeScript Implementation

**MCP Server Registration:**
```typescript
// zen-sci/packages/sdk/src/servers/grant-mcp/index.ts

import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from '@modelcontextprotocol/sdk/types.js';

const server = new Server(
  { name: 'grant-mcp', version: '0.1.0' },
  { capabilities: { tools: {} } }
);

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: 'convert_to_grant',
      description: 'Convert Markdown proposal to PDF and/or Word grant proposal.',
      inputSchema: {
        type: 'object' as const,
        properties: {
          source: {
            type: 'string',
            description: 'Markdown proposal content.',
          },
          frontmatter: {
            type: 'object',
            properties: {
              title: { type: 'string' },
              funder: {
                type: 'string',
                enum: ['nih', 'nsf', 'erc'],
              },
              program_type: { type: 'string' },
              principal_investigator: {
                type: 'object',
                properties: {
                  name: { type: 'string' },
                  institution: { type: 'string' },
                },
              },
              specific_aims: { type: 'array' },
              budget: { type: 'array' },
              institution: { type: 'object' },
              bibliography: { type: 'string' },
              options: { type: 'object' },
            },
            required: [
              'title',
              'funder',
              'program_type',
              'principal_investigator',
            ],
          },
          format: {
            type: 'string',
            enum: ['pdf', 'docx', 'both'],
          },
        },
        required: ['source', 'frontmatter', 'format'],
      },
    },
    {
      name: 'validate_grant_proposal',
      description: 'Validate proposal against funder rules.',
      inputSchema: {
        type: 'object' as const,
        properties: {
          source: { type: 'string' },
          frontmatter: { type: 'object' },
          strict_mode: { type: 'boolean' },
        },
        required: ['source', 'frontmatter'],
      },
    },
    {
      name: 'list_grant_funders',
      description: 'List supported funders and program types.',
      inputSchema: {
        type: 'object' as const,
        properties: {
          funder: { type: 'string' },
        },
      },
    },
  ],
}));

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request;

  if (name === 'convert_to_grant') {
    const result = await convertToGrant(args as any);
    return { content: [{ type: 'text', text: JSON.stringify(result) }] };
  }

  if (name === 'validate_grant_proposal') {
    const result = await validateGrantProposal(args as any);
    return { content: [{ type: 'text', text: JSON.stringify(result) }] };
  }

  if (name === 'list_grant_funders') {
    const result = await listGrantFunders((args as any).funder);
    return { content: [{ type: 'text', text: JSON.stringify(result) }] };
  }

  throw new Error(`Unknown tool: ${name}`);
});

async function convertToGrant(args: any) {
  // Call Python engine
  return await callPythonEngine('convert_grant', args);
}

async function validateGrantProposal(args: any) {
  return await callPythonEngine('validate_grant', args);
}

async function listGrantFunders(funder?: string) {
  const funders = [
    {
      name: 'National Institutes of Health (NIH)',
      programs: [
        {
          name: 'R01',
          page_limit: 12,
          min_font_size: 10,
          min_margin: 0.5,
          description: 'Research Project Grant',
        },
        {
          name: 'R21',
          page_limit: 6,
          min_font_size: 10,
          min_margin: 0.5,
          description: 'Exploratory/Developmental Grant',
        },
      ],
    },
    // ... more funders
  ];

  if (funder) {
    return { funders: funders.filter((f) => f.name.toLowerCase().includes(funder)) };
  }
  return { funders };
}

async function callPythonEngine(method: string, params: any) {
  // Spawn Python subprocess, send JSON-RPC request
}

const transport = new StdioServerTransport();
server.connect(transport);
```

**Core Type Definitions:**
```typescript
// zen-sci/packages/core/src/schemas/grant.ts

export interface GrantDocumentRequest extends DocumentRequest {
  format: 'grant-pdf' | 'grant-docx' | 'grant-both';
  frontmatter: GrantFrontmatter;
  source: string;
}

export interface GrantFrontmatter {
  title: string;
  funder: 'nih' | 'nsf' | 'erc';
  program_type: string; // e.g., 'nih-r01', 'nsf-standard'
  principal_investigator: PIInfo;
  project_period: {
    start_date: string; // ISO date
    end_date: string;
    duration_years: number;
  };
  specific_aims: SpecificAim[];
  budget: BudgetLineItem[];
  institution: InstitutionInfo;
  bibliography?: string;
  options?: GrantOptions;
}

export interface PIInfo {
  name: string;
  institution: string;
  department?: string;
  email?: string;
  phone?: string;
}

export interface SpecificAim {
  aim_number: number;
  title: string;
  objective: string;
  outcomes: string[];
  impact: string;
  timeline_months?: number;
  metrics?: string[]; // Quantified success criteria
}

export interface BudgetLineItem {
  category: 'personnel' | 'equipment' | 'travel' | 'other' | 'facilities';
  description: string;
  amount: number;
  justification?: string;
  linked_aim?: number;
  duration_months?: number;
}

export interface InstitutionInfo {
  name: string;
  indirect_cost_rate: number; // e.g., 0.25 for 25%
  cost_sharing?: boolean;
  duns_number?: string;
  ein?: string;
}

export interface GrantOptions {
  include_timeline?: boolean;
  include_figures?: boolean;
  validation_strict?: boolean;
  budget_auto_justify?: boolean;
}

export interface GrantResponse extends DocumentResponse {
  format: 'grant-pdf' | 'grant-docx' | 'grant-both';
  content: Buffer | { pdf: Buffer; docx: Buffer };
  artifacts: GrantArtifact[];
  metadata: GrantMetadata;
}

export interface GrantArtifact extends Artifact {
  type:
    | 'specific_aims'
    | 'budget_table'
    | 'timeline'
    | 'validation_report'
    | 'budget_justification';
}

export interface GrantMetadata extends ResponseMetadata {
  funder: string;
  program_type: string;
  page_count: number;
  budget_total: number;
  budget_overhead: number;
  compliance_status: 'compliant' | 'warning' | 'error';
  validation_messages: string[];
}
```

---

### 3.5 Python Processing Engine

**LaTeX/PDF Path:**
```python
# zen-sci/servers/grant-mcp/processing/latex_grant.py

import subprocess
import tempfile
import json
from pathlib import Path
from typing import Dict, Any, Tuple, List

import pypandoc

async def generate_grant_pdf(
    source: str,
    frontmatter: Dict[str, Any],
    bibliography: str = None,
) -> Tuple[bytes, Dict[str, str]]:
    """
    Convert Markdown proposal to funder-compliant LaTeX → PDF.

    Args:
        source: Markdown proposal body.
        frontmatter: Parsed YAML with funder, specific aims, budget, etc.
        bibliography: CSL-JSON bibliography content.

    Returns:
        Tuple[PDF bytes, artifacts dict].
    """

    funder = frontmatter['funder']
    program_type = frontmatter['program_type']

    # Step 1: Load funder-specific rules and templates
    funder_rules = load_funder_rules(funder, program_type)
    template_path = f'templates/grant-{funder}.tex'

    # Step 2: Generate Specific Aims page (auto-synthesized)
    specific_aims_content = generate_specific_aims_page(
        frontmatter['specific_aims']
    )

    # Step 3: Generate Budget section with justifications
    budget_section = generate_budget_section(
        frontmatter['budget'],
        frontmatter['institution'],
        frontmatter['specific_aims'],
        funder,
    )

    # Step 4: Combine source + generated sections
    full_proposal = f"""# Specific Aims

{specific_aims_content}

{source}

# Budget Justification

{budget_section}
"""

    # Step 5: Convert Markdown to LaTeX with pandoc + funder-specific filters
    pandoc_args = [
        '-f', 'markdown',
        '-t', 'latex',
        f'--template={template_path}',
        '-V', f'documentclass:article',
        '-V', f'geometry:margin={funder_rules["margin"]}in',
        '-V', f'fontsize:{funder_rules["min_font_size"]}pt',
        '--number-sections',
        '--lua-filter=lua-filters/grant-citations.lua',
    ]

    if bibliography:
        pandoc_args.extend([
            '--citeproc',
            '--bibliography=/dev/stdin',
            f'--csl=styles/{funder}-citations.csl',
        ])

    latex_content = pypandoc.convert_text(
        full_proposal,
        'latex',
        format='markdown',
        extra_args=pandoc_args,
    )

    # Step 6: Validate LaTeX content against funder rules
    validation_errors = validate_latex_against_rules(latex_content, funder_rules)
    if validation_errors:
        raise RuntimeError(f'LaTeX validation failed: {validation_errors}')

    # Step 7: Compile to PDF with xelatex
    with tempfile.TemporaryDirectory() as tmpdir:
        tex_file = Path(tmpdir) / 'proposal.tex'
        pdf_file = Path(tmpdir) / 'proposal.pdf'

        tex_file.write_text(latex_content, encoding='utf-8')

        # xelatex compile (twice for refs/TOC)
        for i in range(2):
            result = subprocess.run(
                ['xelatex', '-interaction=nonstopmode', '-output-directory', tmpdir, 'proposal.tex'],
                cwd=tmpdir,
                capture_output=True,
                text=True,
            )
            if result.returncode != 0 and i == 1:
                raise RuntimeError(f'xelatex failed: {result.stderr}')

        if not pdf_file.exists():
            raise RuntimeError('PDF generation failed: output file not created')

        pdf_bytes = pdf_file.read_bytes()

    # Step 8: Extract artifacts (specific aims, budget table, validation report)
    artifacts = {
        'specific_aims': specific_aims_content,
        'budget_table': budget_section,
    }

    return pdf_bytes, artifacts


def load_funder_rules(funder: str, program_type: str) -> Dict[str, Any]:
    """Load funder-specific formatting rules."""
    rules_file = Path(__file__).parent / f'rules/{funder}-{program_type}.json'
    if not rules_file.exists():
        raise ValueError(f'Unknown funder/program: {funder}/{program_type}')
    return json.loads(rules_file.read_text())


def generate_specific_aims_page(specific_aims: List[Dict[str, Any]]) -> str:
    """Auto-generate Specific Aims page from structured data."""
    lines = []
    lines.append('## Specific Aims')
    lines.append('')
    lines.append('*[Auto-generated from structured specific aims data]*')
    lines.append('')

    for aim in specific_aims:
        lines.append(f"### Aim {aim['aim_number']}: {aim['title']}")
        lines.append('')
        lines.append(f"**Objective:** {aim['objective']}")
        lines.append('')
        lines.append('**Expected Outcomes:**')
        for outcome in aim.get('outcomes', []):
            lines.append(f"- {outcome}")
        lines.append('')
        lines.append(f"**Impact:** {aim['impact']}")
        if aim.get('metrics'):
            lines.append('')
            lines.append('**Success Metrics:**')
            for metric in aim['metrics']:
                lines.append(f"- {metric}")
        lines.append('')

    return '\n'.join(lines)


def generate_budget_section(
    budget_items: List[Dict[str, Any]],
    institution: Dict[str, Any],
    specific_aims: List[Dict[str, Any]],
    funder: str,
) -> str:
    """Generate budget justification tied to specific aims."""
    lines = []
    lines.append('')
    lines.append('### Budget Summary')
    lines.append('')

    # Group budget by category and aim
    by_category = {}
    for item in budget_items:
        cat = item['category']
        if cat not in by_category:
            by_category[cat] = []
        by_category[cat].append(item)

    total_direct = 0
    for category, items in by_category.items():
        lines.append(f'#### {category.capitalize()}')
        lines.append('')
        for item in items:
            amount = item['amount']
            total_direct += amount
            aim_ref = ''
            if item.get('linked_aim'):
                aim_ref = f" (Aim {item['linked_aim']})"
            lines.append(f"- {item['description']}: ${amount:,.0f}{aim_ref}")
            if item.get('justification'):
                lines.append(f"  {item['justification']}")
        lines.append('')

    # Calculate overhead
    overhead_rate = institution.get('indirect_cost_rate', 0.25)
    overhead = total_direct * overhead_rate
    total = total_direct + overhead

    lines.append(f'**Direct Costs:** ${total_direct:,.0f}')
    lines.append(f'**Indirect Costs ({overhead_rate*100:.0f}%):** ${overhead:,.0f}')
    lines.append(f'**Total Project Cost:** ${total:,.0f}')

    return '\n'.join(lines)


def validate_latex_against_rules(latex: str, rules: Dict[str, Any]) -> List[str]:
    """Check LaTeX output for compliance with funder rules."""
    errors = []

    # Simplified checks (real implementation would parse LaTeX properly)
    if f"\\fontsize{{{rules['min_font_size']}" not in latex.lower():
        errors.append(f"Font size must be >= {rules['min_font_size']}pt")

    if f"margin={rules['margin']}in" not in latex:
        errors.append(f"Margins must be >= {rules['margin']} inches")

    page_limit = rules.get('page_limit', 12)
    # Count \newpage or estimate from content length
    # (Real implementation: compile to PDF, count pages)

    return errors
```

**Word (.docx) Path:**
```python
# zen-sci/servers/grant-mcp/processing/docx_grant.py

from docx import Document
from docx.shared import Pt, Inches, RGBColor
from docx.enum.text import WD_ALIGN_PARAGRAPH
from typing import Dict, Any, Tuple

async def generate_grant_docx(
    source: str,
    frontmatter: Dict[str, Any],
    bibliography: str = None,
) -> Tuple[bytes, Dict[str, str]]:
    """
    Generate Word .docx proposal (editable, funder-compliant).

    Args:
        source: Markdown proposal body.
        frontmatter: Parsed YAML.
        bibliography: CSL-JSON bibliography.

    Returns:
        Tuple[.docx bytes, artifacts dict].
    """

    # Create new Document
    doc = Document()

    # Set margins and font per funder rules
    funder = frontmatter['funder']
    funder_rules = load_funder_rules(funder, frontmatter['program_type'])

    # Set document-level styles
    style = doc.styles['Normal']
    font = style.font
    font.name = 'Calibri'
    font.size = Pt(funder_rules['min_font_size'])

    # Title page / cover info
    title = doc.add_paragraph(frontmatter['title'], style='Heading 1')
    title.alignment = WD_ALIGN_PARAGRAPH.CENTER

    doc.add_paragraph(f"PI: {frontmatter['principal_investigator']['name']}")
    doc.add_paragraph(f"Institution: {frontmatter['principal_investigator']['institution']}")
    doc.add_paragraph(f"Program: {frontmatter['program_type']}")
    doc.add_page_break()

    # Specific Aims section
    doc.add_heading('Specific Aims', level=1)
    for aim in frontmatter.get('specific_aims', []):
        doc.add_heading(f"Aim {aim['aim_number']}: {aim['title']}", level=2)
        doc.add_paragraph(f"Objective: {aim['objective']}")

        if aim.get('outcomes'):
            doc.add_paragraph('Expected Outcomes:', style='List Bullet')
            for outcome in aim['outcomes']:
                doc.add_paragraph(outcome, style='List Bullet 2')

        doc.add_paragraph(f"Impact: {aim['impact']}")

    doc.add_page_break()

    # Research Plan section (from Markdown source)
    doc.add_heading('Research Plan', level=1)
    # Convert Markdown to docx paragraph structure
    # (Simplified: add as plain text; real implementation parses Markdown)
    for line in source.split('\n'):
        if line.startswith('##'):
            doc.add_heading(line.replace('##', '').strip(), level=2)
        elif line.startswith('#'):
            doc.add_heading(line.replace('#', '').strip(), level=1)
        elif line.strip():
            doc.add_paragraph(line.strip())

    doc.add_page_break()

    # Budget section
    doc.add_heading('Budget Justification', level=1)
    budget_table = doc.add_table(rows=len(frontmatter.get('budget', [])) + 2, cols=4)
    budget_table.style = 'Light Grid Accent 1'

    # Header row
    header_cells = budget_table.rows[0].cells
    header_cells[0].text = 'Category'
    header_cells[1].text = 'Description'
    header_cells[2].text = 'Amount'
    header_cells[3].text = 'Justification'

    # Data rows
    total_direct = 0
    for i, item in enumerate(frontmatter.get('budget', []), start=1):
        row_cells = budget_table.rows[i].cells
        row_cells[0].text = item['category'].capitalize()
        row_cells[1].text = item['description']
        row_cells[2].text = f"${item['amount']:,.0f}"
        row_cells[3].text = item.get('justification', '')
        total_direct += item['amount']

    # Totals row
    total_row = budget_table.rows[-1].cells
    total_row[0].text = 'Total Direct'
    total_row[2].text = f"${total_direct:,.0f}"

    overhead = total_direct * frontmatter['institution'].get('indirect_cost_rate', 0.25)
    doc.add_paragraph(f"Indirect Costs: ${overhead:,.0f}")
    doc.add_paragraph(f"Total Project Cost: ${total_direct + overhead:,.0f}")

    # Serialize to bytes
    import io
    output = io.BytesIO()
    doc.save(output)
    docx_bytes = output.getvalue()

    artifacts = {}
    return docx_bytes, artifacts
```

---

### 3.6 Key Schemas

```typescript
// zen-sci/packages/core/src/schemas/grant-extended.ts

export interface GrantRules {
  funder: string;
  program_type: string;
  page_limit: number;
  min_font_size: number;
  min_margin: number; // in inches
  budget_categories: string[];
  required_sections: string[];
  citation_style: string; // e.g., 'numbered', 'author-date'
  specific_aims_page_limit?: number;
}

export interface BudgetSynthesis {
  total_direct: number;
  total_indirect: number;
  total: number;
  by_category: Record<string, number>;
  overhead_rate: number;
  cost_sharing: number;
  budget_coherence_score: number; // 0-100: how well aligned with specific aims
}

export interface ComplianceReport {
  funder: string;
  program_type: string;
  is_compliant: boolean;
  violations: ComplianceViolation[];
  warnings: ComplianceWarning[];
  estimated_page_count: number;
}

export interface ComplianceViolation {
  rule: string;
  message: string;
  severity: 'error' | 'warning';
  remediation?: string;
}

export interface ComplianceWarning {
  pattern: string; // e.g., 'common_rejection_reason'
  message: string;
  suggestion: string;
}

/**
 * GrantSubmissionFormat: Platform-specific export configuration for grant submission portals.
 * Defines how to package and structure a proposal for upload to funder systems.
 * Note: ZenSci generates the files; actual portal upload is the user's responsibility (see Non-Goals).
 */
export interface GrantSubmissionFormat {
  /** Target submission platform */
  platform: 'grants-gov' | 'nih-era-commons' | 'nsf-research-gov' | 'erc-sep' | 'manual';

  /** Funder this submission format serves */
  funder: 'nih' | 'nsf' | 'erc';

  /** Program type (e.g., 'nih-r01') */
  programType: string;

  /** Required output files for this platform */
  requiredFiles: SubmissionFile[];

  /** File naming convention required by platform (e.g., '{lastName}_{grantNumber}_{section}.pdf') */
  fileNamingPattern?: string;

  /** Maximum total submission package size in bytes */
  maxTotalSizeBytes?: number;

  /** Whether platform accepts direct PDF upload */
  acceptsPdf: boolean;

  /** Whether platform accepts Word (.docx) upload */
  acceptsDocx: boolean;

  /** Whether platform requires structured metadata (XML/JSON form fields) */
  requiresStructuredMetadata: boolean;

  /** Version of this submission format specification (tied to funder guidelines version) */
  formatVersion: string;

  /** Notes or special instructions for this platform */
  notes?: string;
}

/** Known submission file roles across major grant platforms */
export type SubmissionFileRole =
  | 'specific-aims'
  | 'research-plan'
  | 'budget-justification'
  | 'biosketch'
  | 'facilities'
  | 'data-management-plan'
  | 'references'
  | 'cover-letter'
  | 'abstract'
  | 'other';

export interface SubmissionFile {
  /** Functional role of this file (strict enum — see SubmissionFileRole) */
  role: SubmissionFileRole;

  /** Accepted MIME types (e.g., ['application/pdf', 'application/vnd.openxmlformats-officedocument.wordprocessingml.document']) */
  mimeTypes: string[];

  /** Maximum file size in bytes */
  maxSizeBytes?: number;

  /** Whether this file is required for submission */
  required: boolean;

  /** Page or section limit for this file */
  pageLimit?: number;

  /** Description of what this file should contain */
  description: string;
}
```

---

### 3.7 Phase 2 Infrastructure Requirements

**New packages/core capabilities needed:**

1. **Funder rule registry:** Centralized, versioned funder rules (NIH, NSF, ERC formats, page limits, fonts, margins).
   - *Recommendation:* Create `packages/core/funder-rules/` directory with JSON files per funder/program.

2. **Multi-format output system:** DocumentResponse.artifacts should support variable outputs (PDF + Word simultaneously).
   - *Dependency:* Extend Artifact type to distinguish between PDF and Word outputs.

3. **Advanced bibliography system:** CSL-JSON with funder-specific citation styles (NIH numbered, NSF author-date, ERC author-date).
   - *Dependency:* Integrate with latex-mcp's bibliography system; add HTML citation rendering for Word.

4. **Budget synthesis engine:** Auto-generate budget justifications from project activities + institutional overhead.
   - *Recommendation:* Create new module `packages/core/budget-synthesis/` for cost modeling and activity-to-aim mapping.

5. **Compliance validation framework:** Pluggable validators for funder-specific rules (page limits, fonts, margins, budget coherence).
   - *Recommendation:* Define validator interface in packages/core; implement per-funder validators in grant-mcp.

---

### 3.8 Performance & Constraints

| Constraint | Target | Notes |
|-----------|--------|-------|
| LaTeX → PDF compilation (12-page R01) | <8s | xelatex with cached templates |
| Markdown → Word (.docx) generation | <3s | python-docx direct serialization |
| Specific Aims synthesis | <500ms | Parse structured aims; template rendering |
| Budget generation + overhead calculation | <1s | Math-based; no external calls |
| Compliance validation | <2s | Pattern matching + heuristics |
| Memory usage | <500 MB | Single proposal; temporary files cleaned |
| Output file size (PDF, 12 pages) | <3 MB | Text-heavy; embedded fonts |
| Output file size (.docx, 12 pages) | <1 MB | XML-based format, compact |

---

## 4. Implementation Plan

### 4.1 Phased Approach

| Phase | Timeline | Deliverables | Dependencies |
|-------|----------|--------------|--------------|
| **Alpha (Week 1-2)** | 2 weeks | NIH R01 PDF only; no specific aims synthesis; basic budget table | packages/core v1.0, latex-mcp v0.1 |
| **Beta (Week 3-4)** | 2 weeks | + Specific aims synthesis; Word (.docx) output; compliance validation | All Alpha |
| **Candidate (Week 5-6)** | 2 weeks | + NSF & ERC support; budget justification synthesis; funder rule registry | All Beta + funder rules infra |
| **Release (Week 7)** | 1 week | Full test suite; documentation; template refinement | All Candidate |

### 4.2 Week-by-Week Breakdown

**Week 1: NIH R01 LaTeX Foundation & Funder Rules**
- [ ] Set up zen-sci/servers/grant-mcp/ directory structure
- [ ] Implement MCP server skeleton (TypeScript, registration)
- [ ] Create NIH R01 LaTeX template (margins, fonts, page breaks)
- [ ] Build funder rules registry (JSON schema for NIH R01)
- [ ] Implement pandoc Markdown → LaTeX pipeline
- [ ] Write xelatex compilation + error handling
- [ ] Unit tests: valid frontmatter → PDF; invalid margins → error
- [ ] Integration test: sample R01 skeleton → valid PDF

**Week 2: Budget Table & Specific Aims Page**
- [ ] Implement `convert_to_grant` tool for PDF format only
- [ ] Add budget table generation (category totals, overhead calculation)
- [ ] Implement specific aims synthesis from structured YAML
- [ ] Add comprehensive `validate_grant_proposal` tool
- [ ] Write compliance validators (page limit, font size, margins)
- [ ] Unit tests: budget calculation; specific aims formatting; compliance checks
- [ ] Integration test: full R01 proposal (specific aims + research plan + budget) → PDF

**Week 3: Word (.docx) Path & Dual Output**
- [ ] Implement Word (.docx) generation using python-docx
- [ ] Set up document structure (title page, sections, budget table)
- [ ] Implement dual-output mode (format='both')
- [ ] Add docx-specific styling (fonts, heading hierarchy)
- [ ] Unit tests: .docx validity; structure preservation; table formatting
- [ ] Integration test: same proposal → both PDF and Word; verify parity

**Week 4: NSF & ERC Support + Rules Registry**
- [ ] Add NSF-STANDARD rules (page limits, budget structure differences)
- [ ] Add ERC-STARTING rules (H2020/Horizon Europe compliance)
- [ ] Extend LaTeX templates for NSF and ERC specific formatting
- [ ] Implement funder selector logic (`list_grant_funders`)
- [ ] Unit tests: NSF proposals; ERC proposals; rule loading
- [ ] Integration test: Convert same proposal for 3 funders; verify compliance

**Week 5: Budget Justification Synthesis & Activity Mapping**
- [ ] Implement budget justification auto-generation from activities
- [ ] Add activity-to-aim linking (budget items specify which aim they support)
- [ ] Implement budget coherence score (how well aligned with specific aims)
- [ ] Add timeline visualization (Gantt chart or simple table)
- [ ] Unit tests: budget synthesis; activity mapping; coherence scoring
- [ ] Integration test: Proposal with 10 budget lines + 3 aims → coherence report

**Week 6: Citation Styles & Advanced Validation**
- [ ] Integrate funder-specific citation styles (NIH numbered, NSF author-date, ERC author-date)
- [ ] Add CSL-JSON bibliography support (from packages/core)
- [ ] Implement common rejection reason detection (warnings)
- [ ] Add institutional cost-sharing and overhead rate validation
- [ ] Unit tests: citation rendering; rejection reason detection; cost-sharing logic
- [ ] Integration test: Proposal with 20 citations → funder-specific bibliography

**Week 7: Edge Cases, Documentation & Release**
- [ ] Handle long project titles, author names, institution names
- [ ] Test multi-year budgets (year 1, 2, 3 breakdowns)
- [ ] Test cost-sharing constraints
- [ ] Write TypeScript/Markdown API documentation
- [ ] Create example proposals (R01, NSF-STANDARD, ERC-STARTING)
- [ ] Write Python processing engine docs
- [ ] Finalize error messages and compliance warnings
- [ ] Publish v0.1.0 to npm (@zen-sci/grant-mcp)
- [ ] Full regression test suite (50+ unit + 10+ integration tests)

### 4.3 Dependencies & Prerequisites

**Hard Dependencies:**
- packages/core v1.0 (stable): DocumentRequest/DocumentResponse, logging
- latex-mcp v0.1 (released): TeX compilation pipeline, LaTeX templates

**New Infrastructure (to be added to packages/core):**
- Funder rules registry (JSON files per funder/program)
- Budget synthesis module (cost modeling, activity-to-aim mapping)
- Compliance validation framework

**External Tools:**
- pandoc >= 3.0 (Markdown → LaTeX/Word)
- xelatex (TeX → PDF)
- python-docx (Word .docx generation)
- Funder documentation (NIH guidelines, NSF PAPPG, ERC guidelines)

---

### 4.4 Testing Strategy

**Unit Tests (TypeScript):**
```typescript
// zen-sci/servers/grant-mcp/__tests__/grant.test.ts

describe('grant-mcp', () => {
  describe('convertToGrant', () => {
    it('should generate compliant NIH R01 PDF', async () => {
      const frontmatter = {
        title: 'Research Project Title',
        funder: 'nih',
        program_type: 'nih-r01',
        principal_investigator: { name: 'Dr. Smith', institution: 'MIT' },
        specific_aims: [
          {
            aim_number: 1,
            title: 'Characterize protein structure',
            objective: 'Determine X-ray crystal structure',
            outcomes: ['High-resolution structure (< 2 Å)'],
            impact: 'Enables rational drug design',
          },
        ],
        budget: [
          {
            category: 'personnel',
            description: 'Postdoctoral fellow salary',
            amount: 60000,
            linked_aim: 1,
          },
        ],
        institution: { name: 'MIT', indirect_cost_rate: 0.47 },
      };
      const source = '## Research Plan\nDetailed methodology...';
      const result = await convertToGrant(source, frontmatter, 'pdf');

      expect(result.format).toBe('grant-pdf');
      expect(result.content).toBeInstanceOf(Buffer);
      expect(result.metadata.compliance_status).toBe('compliant');
      expect(result.metadata.budget_total).toBeGreaterThan(0);
    });

    it('should generate Word (.docx) with editable structure', async () => {
      // ... similar test for Word format
    });

    it('should support dual format (PDF + Word)', async () => {
      // ... test both formats simultaneously
    });
  });

  describe('validateGrantProposal', () => {
    it('should detect font size violations', async () => {
      const frontmatter = { /* ... */ };
      const result = await validateGrantProposal(source, frontmatter, true);
      expect(result.compliant).toBe(false);
      // Verify error about font size
    });

    it('should detect page limit exceeded', async () => {
      // Create very long source
      // Verify page limit error
    });

    it('should warn on weak specific aims (no metrics)', async () => {
      const frontmatter = {
        specific_aims: [
          {
            aim_number: 1,
            title: 'Study X',
            objective: 'Do Y',
            // Missing outcomes and metrics
          },
        ],
      };
      const result = await validateGrantProposal(source, frontmatter);
      expect(result.errors).toContainEqual(
        expect.objectContaining({ rule: 'missing_metrics' })
      );
    });
  });
});
```

**Integration Tests (Python):**
```python
# zen-sci/servers/grant-mcp/tests/test_grants_e2e.py

def test_nih_r01_full_lifecycle():
    """End-to-end: R01 proposal from Markdown → PDF + Word."""
    # Create test proposal with specific aims, budget, references
    # Convert to PDF
    # Verify: compliance, page count, budget total, citations
    # Convert to Word
    # Verify: docx validity, structure, editability

def test_nsf_to_erc_portability():
    """Same proposal should convert correctly to NSF and ERC formats."""
    # Create proposal for NSF
    # Convert to PDF
    # Change funder to ERC
    # Convert to PDF
    # Verify: both compliant, but different formatting

def test_budget_coherence_scoring():
    """Test budget-to-aim coherence scoring."""
    # Create proposal with 3 aims
    # Budget items linked to aims
    # Verify coherence score is high
    # Add orphaned budget item (not linked to any aim)
    # Verify warning raised
```

---

## 5. Risk Assessment & Mitigation

| Risk | Likelihood | Impact | Mitigation |
|-------|-----------|--------|-----------|
| Funder rules change frequently (new guidelines) | Medium | Medium | Version funder rules in JSON; pin rules to proposal at conversion time; update registry quarterly |
| Budget overhead calculation errors | Low | High | Comprehensive unit tests; validation against known institutional rates; manual review checklist |
| LaTeX compilation complexity (natbib, CSL) | Medium | Medium | Pre-test with real NSF/ERC citations; CI testing on funder-provided samples |
| Word (.docx) editability loss (formatting) | Low | Medium | User testing; ensure .docx is 100% editable; avoid complex styled tables |
| Citation format divergence (Beamer vs Grant) | Low | Low | Unified CSL-JSON handler in packages/core; test parity across modules |
| Specific aims synthesis oversimplification | Medium | Low | Clearly document auto-generation; allow user override; add manual editing mode |

---

## 6. Rollback & Contingency

**If NIH R01 PDF is non-compliant (Week 1-2):**
- Revert to simpler Beamer-like PDF output (no specific aims synthesis); fix in v0.2.

**If Word (.docx) generation is problematic (Week 3):**
- Ship PDF-only in v0.1; defer Word support to v0.1.1.

**If funder rules are incomplete (Week 4):**
- Ship with 1-2 funders only (NIH + NSF); add ERC in v0.1.1 after validation.

**Rollback procedure:**
- Tag npm release with `-rc` suffix if issues found.
- Publish corrected version after fixes.
- Communicate breaking changes in release notes.

---

## 7. Appendices

### 7.1 Related Work & Standards

- **NIH PAPPG:** NIH Proposal Application Guide (formatting, budget rules, allowed costs).
- **NSF PAPPG:** NSF Proposal & Award Policies and Procedures Guide.
- **ERC Guidelines:** Horizon Europe / ERC Call for Proposals (H2020 era, evolving).
- **CSL:** Citation Style Language (standardized citation formatting).
- **pandoc:** Markdown to LaTeX/Word conversion.
- **python-docx:** Programmatic Word (.docx) generation.

### 7.2 Future Considerations (v1.1+)

- **Grants.gov integration:** Auto-upload PDF + metadata to grants.gov.
- **fastLane.nsf.gov integration:** NSF submission workflow automation.
- **Biosketch generation:** Auto-generate NIH biosketch from CV.
- **Budget spreadsheet sync:** Link Excel budget file → auto-update proposal budget table.
- **Collaboration features:** Multi-user editing (with comment tracking for institutional review).
- **Funder-specific templates:** More detailed templates per program (R01 vs R21 vs R15 vs etc.).

### 7.3 Open Questions

1. **Should budget justifications be auto-generated or template-based?** (Current plan: auto-generate from activities.)
2. **How detailed should institutional cost-sharing validation be?** (MVP: simple rate lookup.)
3. **Should we support indirect cost negotiation rates that change year-by-year?** (MVP: fixed rate per proposal.)
4. **Should we generate a compliance checklist artifact for institutional review?** (Recommended: yes.)

### 7.4 Schema Backlog

**Potential future extensions:**

```typescript
export interface GrantDocumentRequest extends DocumentRequest {
  // Current fields...

  // v1.1 candidates:
  biosketch?: BiosketchData; // Auto-generate NIH biosketch
  facilities?: FacilityInfo[]; // Institutional facilities available
  collaborators?: CollaboratorInfo[]; // Co-investigators + key personnel
  institutional_review?: {
    require_cost_sharing?: boolean;
    require_overhead_rate_approval?: boolean;
    compliance_checklist?: string[];
  };
}
```

---

**End of grant-mcp Specification**

---

## Audit Notes (2026-02-18)

**Auditor:** Claude (Opus-level audit)
**Status:** Reviewed

### Issues
- **CRITICAL-2:** Instantiates `Server` directly instead of extending `ZenSciServer`.
- **CRITICAL-3:** Uses low-level `setRequestHandler` pattern.
- MCP SDK imports correct. Tool naming snake_case — consistent.
- One of the most detailed Phase 2 specs — excellent for autonomous implementation.

### Section 3.6 Audit (2026-02-18, follow-up)

**New types reviewed:** GrantSubmissionFormat, SubmissionFile.

#### Fixed
1. **`GrantSubmissionFormat.platform`:** Changed `'nsf-fastlane'` → `'nsf-research-gov'`. NSF FastLane was retired and replaced by Research.gov in 2023.
2. **Python import `import pandoc`:** Changed to `import pypandoc`. The `pandoc` Python package doesn't expose `convert_text()` with the expected signature. `pypandoc` is the correct library (consistent with other specs).
3. **Removed unused `import yaml`:** Dead import on the same line.
4. **`pypandoc.convert_text()` call signature fixed:** Added `'latex'` as second positional arg (target format). Removed `standalone=True` (not a valid pypandoc kwarg; use `extra_args=['--standalone']` if needed).
5. **Pandoc version:** Changed `≥2.18` → `>= 3.0` for cross-spec consistency.

#### Noted (require decisions)
7. **LOW: Non-Goals vs GrantSubmissionFormat tension** — No action needed (JSDoc boundary is correct).

#### Decision Propagation (Cruz, 2026-02-18)
6. **RESOLVED: `SubmissionFile.role` typed** — New `SubmissionFileRole` union type added with 10 known roles (`'specific-aims' | 'research-plan' | 'budget-justification' | 'biosketch' | 'facilities' | 'data-management-plan' | 'references' | 'cover-letter' | 'abstract' | 'other'`). Per Cluster 2 decision: strict enums everywhere, no `| string` escape hatch.
- **CRITICAL-2 resolved:** Direct `McpServer` instantiation via `createZenSciServer()` factory is now the standard pattern (per Cluster 1 decision). This spec's existing pattern of direct `Server` instantiation is closest to the new approach — update to `McpServer` + `registerTool()` with Zod at implementation time.
- **CRITICAL-3 resolved:** Low-level `setRequestHandler` replaced by `McpServer.registerTool()` with Zod input schemas per MCP SDK v2.
