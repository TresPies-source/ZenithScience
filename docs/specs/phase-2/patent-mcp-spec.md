# patent-mcp Spec: USPTO/WIPO Patent Application Generator

**Author:** ZenSci Architecture (TresPiesDesign.com)
**Status:** Draft
**Created:** 2026-02-18
**Phase:** 2 (ships after Phase 1 core is validated)
**Grounded In:** Module strategy scout (2026-02-17), ZenSci semantic clusters

---

## 1. Vision
> Auto-generate USPTO and WIPO patent applications from technical descriptions, building dependent/independent claim trees, formatting specifications, and managing drawing references.

### The Core Insight
Patent drafting is **highly specialized**: claims must follow strict logical dependency trees (independent claims → dependent claims with specific limitations), specifications require cross-references to drawing numbers and abstract, and USPTO/WIPO have different formatting rules. patent-mcp bridges this gap: structured Markdown input (claims structure + specification) → compliant LaTeX/Word patent documents.

Unlike generic patent templates, patent-mcp **validates claim logic** (dependent claims must narrow parent claims) and auto-formats claims trees. It enforces USPTO/WIPO rules: claim numbering, term usage, specification formatting.

### What Makes This Different
1. **Claim tree validation:** Detect invalid claim dependencies; validate claim narrowing logic.
2. **Multi-jurisdiction support:** Generate USPTO and WIPO (PCT) formats simultaneously.
3. **Drawing reference management:** Auto-link specification text to drawing numbers; validate all drawings referenced.
4. **Claim formatting:** Auto-format independent → dependent chain; highlight term definitions.
5. **Specification structure:** Enforce IEEE/USPTO field/background/summary/detailed description structure.

---

## 2. Goals & Success Criteria

### Primary Goals
1. Parse Markdown patent specification + claims structure; output USPTO/WIPO-compliant documents.
2. Validate claim trees (dependency hierarchy, claim narrowing).
3. Auto-format independent and dependent claims with proper numbering.
4. Link specification text to drawing references; validate all drawings cited.
5. Generate abstract, title, and filing metadata.
6. Support multiple figures and technical drawings.

### Success Criteria
✅ Patent application (15 independent claims, 40 dependent claims, 5 figures) validates correctly.
✅ Claim tree validation detects invalid dependencies (e.g., dependent claim broader than parent).
✅ Generated PDF meets USPTO formatting requirements (font, margins, claim numbering).
✅ All drawing references in specification are valid and linked.
✅ Both USPTO and WIPO formats generated without manual intervention.
✅ End-to-end test: Technical description + claims → USPTO PDF + WIPO PDF in <10s.

### Non-Goals
❌ Integration with USPTO or WIPO submission systems.
❌ Prior art search or patentability analysis.
❌ Patent term/expiration tracking.
❌ Automated responses to office actions.

---

## 3. Technical Architecture

### 3.1 System Overview

```
Markdown Specification + Claims Structure
    ↓
patent-mcp MCP Server (TypeScript)
    ├─ Parse claims tree (independent → dependent)
    ├─ Validate claim logic and drawing references
    ├─ Route to Python engine
    └─ Return DocumentResponse (PDF + metadata)
    ↓
Python Engine (pandoc + TeX)
    ├─ Assemble specification + claims + abstract
    ├─ Format claims with dependency tree visualization
    ├─ Compile LaTeX → PDF
    └─ Return bytes (PDF)
    ↓
DocumentResponse (format: 'patent-pdf', artifacts: [claims tree, abstract, validation report])
```

### 3.2 MCP Tool Definitions

#### Tool: `convert_to_patent`
**Description:** Convert patent specification to USPTO/WIPO-compliant application.

**Input Schema:**
```json
{
  "type": "object",
  "properties": {
    "source": {
      "type": "string",
      "description": "Markdown specification (Field, Background, Summary, Detailed Description)"
    },
    "claims": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "claim_number": { "type": "integer" },
          "type": { "type": "string", "enum": ["independent", "dependent"] },
          "preamble": { "type": "string", "description": "e.g., 'A method comprising:'" },
          "limitations": { "type": "array", "items": { "type": "string" } },
          "depends_on": { "type": "integer", "description": "For dependent claims, parent claim #" }
        },
        "required": ["claim_number", "type", "preamble"]
      }
    },
    "metadata": {
      "type": "object",
      "properties": {
        "title": { "type": "string" },
        "inventors": { "type": "array", "items": { "type": "string" } },
        "applicant": { "type": "string" },
        "abstract": { "type": "string" },
        "filing_date": { "type": "string", "format": "date" },
        "jurisdiction": { "type": "string", "enum": ["uspto", "wipo", "both"] },
        "drawings": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "figure_number": { "type": "integer" },
              "description": { "type": "string" },
              "filename": { "type": "string" }
            }
          }
        }
      },
      "required": ["title", "inventors", "applicant", "abstract", "jurisdiction"]
    }
  },
  "required": ["source", "claims", "metadata"]
}
```

**Output Schema:**
```json
{
  "type": "object",
  "properties": {
    "id": { "type": "string" },
    "format": { "type": "string" },
    "content": { "type": "string", "description": "Base64-encoded PDF or {uspto: PDF, wipo: PDF}" },
    "artifacts": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "type": { "type": "string", "enum": ["claims_tree", "abstract", "validation_report", "drawing_manifest"] },
          "filename": { "type": "string" },
          "content": { "type": "string" }
        }
      }
    },
    "metadata": {
      "type": "object",
      "properties": {
        "jurisdiction": { "type": "string" },
        "claim_count": { "type": "integer" },
        "independent_claims": { "type": "integer" },
        "dependent_claims": { "type": "integer" },
        "drawing_count": { "type": "integer" },
        "compliance_status": { "type": "string" }
      }
    },
    "elapsed": { "type": "number" }
  }
}
```

#### Tool: `validate_patent`
**Description:** Validate patent claims and specification against USPTO/WIPO rules.

**Input Schema:**
```json
{
  "type": "object",
  "properties": {
    "source": { "type": "string" },
    "claims": { "type": "array" },
    "metadata": { "type": "object" },
    "strict_mode": { "type": "boolean" }
  },
  "required": ["source", "claims", "metadata"]
}
```

**Output Schema:**
```json
{
  "type": "object",
  "properties": {
    "valid": { "type": "boolean" },
    "jurisdiction": { "type": "string" },
    "errors": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "type": { "type": "string", "enum": ["invalid_dependency", "claim_breadth_violation", "missing_antecedent", "drawing_mismatch"] },
          "claim_number": { "type": "integer" },
          "message": { "type": "string" }
        }
      }
    },
    "warnings": { "type": "array", "items": { "type": "object" } },
    "report": {
      "type": "object",
      "properties": {
        "total_claims": { "type": "integer" },
        "independent_claims": { "type": "integer" },
        "dependency_chain_lengths": { "type": "array", "items": { "type": "integer" } },
        "undefined_terms": { "type": "array", "items": { "type": "string" } },
        "uncited_drawings": { "type": "array", "items": { "type": "integer" } }
      }
    }
  }
}
```

### 3.3 Core Integration Points

- **latex-mcp:** TeX compilation pipeline.
- **packages/core:** Logging and error handling.

---

### 3.4 TypeScript Implementation

```typescript
// zen-sci/packages/sdk/src/servers/patent-mcp/index.ts

import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from '@modelcontextprotocol/sdk/types.js';

const server = new Server(
  { name: 'patent-mcp', version: '0.1.0' },
  { capabilities: { tools: {} } }
);

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: 'convert_to_patent',
      description: 'Convert patent specification to USPTO/WIPO-compliant application.',
      inputSchema: {
        type: 'object' as const,
        properties: {
          source: { type: 'string' },
          claims: { type: 'array' },
          metadata: { type: 'object' },
        },
        required: ['source', 'claims', 'metadata'],
      },
    },
    {
      name: 'validate_patent',
      description: 'Validate patent claims and specification.',
      inputSchema: {
        type: 'object' as const,
        properties: {
          source: { type: 'string' },
          claims: { type: 'array' },
          metadata: { type: 'object' },
          strict_mode: { type: 'boolean' },
        },
        required: ['source', 'claims', 'metadata'],
      },
    },
  ],
}));

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request;

  if (name === 'convert_to_patent') {
    const result = await convertToPatent(args as any);
    return { content: [{ type: 'text', text: JSON.stringify(result) }] };
  }

  if (name === 'validate_patent') {
    const result = await validatePatent(args as any);
    return { content: [{ type: 'text', text: JSON.stringify(result) }] };
  }

  throw new Error(`Unknown tool: ${name}`);
});

async function convertToPatent(args: any) {
  return await callPythonEngine('convert_patent', args);
}

async function validatePatent(args: any) {
  return await callPythonEngine('validate_patent', args);
}

async function callPythonEngine(method: string, params: any) {
  // Spawn Python subprocess
}

const transport = new StdioServerTransport();
server.connect(transport);
```

### 3.5 Python Processing Engine

```python
# zen-sci/servers/patent-mcp/processing/patent.py

import subprocess
import tempfile
from pathlib import Path
from typing import Dict, List, Any, Tuple, Set

import pandoc

async def generate_patent_pdf(
    source: str,
    claims: List[Dict[str, Any]],
    metadata: Dict[str, Any],
) -> Tuple[bytes | Dict[str, bytes], Dict[str, str]]:
    """Generate USPTO/WIPO patent PDF(s)."""

    # Step 1: Validate claims tree
    validation = validate_claims_tree(claims)
    if not validation['valid']:
        raise RuntimeError(f"Invalid claims tree: {validation['errors']}")

    # Step 2: Validate drawing references in specification
    drawing_refs = extract_drawing_references(source)
    drawing_numbers = {d['figure_number'] for d in metadata.get('drawings', [])}
    missing_drawings = drawing_refs - drawing_numbers
    if missing_drawings:
        raise RuntimeError(f"Missing drawings: {missing_drawings}")

    # Step 3: Build LaTeX patent specification
    patent_latex = build_patent_latex(source, claims, metadata)

    # Step 4: Compile to PDF (once for both jurisdictions; style varies in output)
    pdfs = {}
    for jurisdiction in metadata.get('jurisdictions', ['uspto']):
        template = f'templates/patent-{jurisdiction}.tex'
        latex_content = patent_latex.replace('%%JURISDICTION%%', jurisdiction.upper())

        with tempfile.TemporaryDirectory() as tmpdir:
            tex_file = Path(tmpdir) / 'patent.tex'
            pdf_file = Path(tmpdir) / 'patent.pdf'

            tex_file.write_text(latex_content, encoding='utf-8')

            result = subprocess.run(
                ['xelatex', '-interaction=nonstopmode', '-output-directory', tmpdir, 'patent.tex'],
                cwd=tmpdir,
                capture_output=True,
                text=True,
            )

            if not pdf_file.exists():
                raise RuntimeError(f'PDF generation failed for {jurisdiction}')

            pdfs[jurisdiction] = pdf_file.read_bytes()

    # Return single PDF if only one jurisdiction; dict otherwise
    if len(pdfs) == 1:
        return list(pdfs.values())[0], {}
    return pdfs, {}


def validate_claims_tree(claims: List[Dict[str, Any]]) -> Dict[str, Any]:
    """
    Validate claim tree structure.
    - All independent claims must be at root (no depends_on)
    - All dependent claims must refer to valid parent claims
    - No circular dependencies
    - Dependent claims must be narrower than parent
    """
    errors = []
    claim_numbers = {c['claim_number'] for c in claims}

    # Check for invalid dependencies
    for claim in claims:
        if claim['type'] == 'dependent':
            parent_num = claim.get('depends_on')
            if not parent_num:
                errors.append(f"Dependent claim {claim['claim_number']} missing depends_on")
            elif parent_num not in claim_numbers:
                errors.append(
                    f"Dependent claim {claim['claim_number']} refers to non-existent claim {parent_num}"
                )

    # Check for breadth violations (simplified: dependent must add limitations)
    for claim in claims:
        if claim['type'] == 'dependent':
            parent_claim = next((c for c in claims if c['claim_number'] == claim['depends_on']), None)
            if parent_claim:
                parent_limitations = set(parent_claim.get('limitations', []))
                claim_limitations = set(claim.get('limitations', []))
                if not claim_limitations:
                    errors.append(
                        f"Dependent claim {claim['claim_number']} adds no new limitations"
                    )
                elif claim_limitations.issubset(parent_limitations):
                    errors.append(
                        f"Dependent claim {claim['claim_number']} may be broader than parent"
                    )

    return {'valid': len(errors) == 0, 'errors': errors}


def extract_drawing_references(source: str) -> Set[int]:
    """Extract all drawing figure numbers referenced in specification."""
    import re
    matches = re.findall(r'FIG\.?\s*(\d+)', source, re.IGNORECASE)
    return set(int(m) for m in matches)


def build_patent_latex(
    source: str,
    claims: List[Dict[str, Any]],
    metadata: Dict[str, Any],
) -> str:
    """Build complete patent LaTeX document."""
    lines = []

    # Title page
    lines.append(f"\\title{{{metadata['title']}}}")
    lines.append(f"\\author{{{', '.join(metadata['inventors'])}}}")
    lines.append(f"\\date{{{metadata.get('filing_date', '')}}}")
    lines.append('')

    # Abstract
    lines.append('\\section*{Abstract}')
    lines.append(metadata.get('abstract', ''))
    lines.append('')

    # Specification
    lines.append('\\section{Specification}')
    lines.append(source)
    lines.append('')

    # Claims
    lines.append('\\section{Claims}')
    lines.append('')
    claim_latex = format_claims(claims)
    lines.append(claim_latex)

    return '\n'.join(lines)


def format_claims(claims: List[Dict[str, Any]]) -> str:
    """Format claims with proper numbering and dependencies."""
    lines = []

    for claim in sorted(claims, key=lambda c: c['claim_number']):
        num = claim['claim_number']
        preamble = claim['preamble']
        limitations = claim.get('limitations', [])
        depends_on = claim.get('depends_on')

        # Format claim number
        lines.append(f"\\textbf{{{num}.}} ")

        # Format preamble and dependencies
        if depends_on:
            lines.append(f"The claim of {depends_on}, wherein {preamble.lower()}")
        else:
            lines.append(preamble)

        # Format limitations as enumerated list or numbered steps
        for i, limitation in enumerate(limitations, 1):
            lines.append(f"  {i}. {limitation}")

        lines.append('')

    return '\n'.join(lines)
```

### 3.6 Key Schemas

```typescript
// zen-sci/packages/core/src/schemas/patent.ts

export interface PatentClaim {
  claim_number: number;
  type: 'independent' | 'dependent';
  preamble: string; // "A method comprising:", "An apparatus comprising:", etc.
  limitations: string[]; // Ordered list of claim elements
  depends_on?: number; // For dependent claims
}

export interface PatentMetadata {
  title: string;
  inventors: string[];
  applicant: string;
  abstract: string;
  filing_date: string; // ISO date
  jurisdiction: 'uspto' | 'wipo' | 'both';
  drawings: PatentDrawing[];
}

export interface PatentDrawing {
  figure_number: number;
  description: string;
  filename?: string; // Path to image file
}

export interface PatentDocumentRequest extends DocumentRequest {
  format: 'patent-pdf' | 'patent-docx';
  source: string;
  claims: PatentClaim[];
  metadata: PatentMetadata;
}

export interface PatentResponse extends DocumentResponse {
  format: 'patent-pdf' | 'patent-docx';
  content: Buffer | { uspto: Buffer; wipo: Buffer };
  artifacts: PatentArtifact[];
  metadata: PatentMetadata & { claim_count: number };
}

export interface PatentArtifact extends Artifact {
  type: 'claims_tree' | 'abstract' | 'validation_report' | 'drawing_manifest';
}

export interface ClaimTreeValidation {
  valid: boolean;
  errors: ClaimError[];
  warnings: string[];
}

export interface ClaimError {
  type:
    | 'invalid_dependency'
    | 'claim_breadth_violation'
    | 'missing_antecedent'
    | 'circular_dependency';
  claim_number: number;
  message: string;
}
```

---

### 3.7 Phase 2 Infrastructure Requirements

- Patent claim validation rules (USPTO, WIPO).
- Patent LaTeX templates (USPTO, WIPO formats).
- Claim tree visualization (optional: Mermaid diagrams).

---

### 3.8 Performance & Constraints

| Constraint | Target | Notes |
|-----------|--------|-------|
| Patent with 15 independent + 40 dependent claims | <10s | Claim validation + LaTeX compilation |
| Claim tree validation | <1s | Dependency graph analysis |
| Drawing reference validation | <500ms | Regex extraction + set comparison |

---

## 4. Implementation Plan

### 4.1 Phased Approach

| Phase | Timeline | Deliverables | Dependencies |
|-------|----------|--------------|--------------|
| **Alpha (Week 1-2)** | 2 weeks | Basic USPTO PDF generation; no validation | packages/core v1.0, latex-mcp v0.1 |
| **Beta (Week 3-4)** | 2 weeks | + Claims tree validation; drawing reference checking | All Alpha |
| **Candidate (Week 5-6)** | 2 weeks | + WIPO support; dual-jurisdiction output | All Beta |
| **Release (Week 7)** | 1 week | Full test suite; documentation | All Candidate |

### 4.2 Week-by-Week Breakdown

**Week 1: USPTO PDF Generation & Claims Formatting**
- [ ] Set up zen-sci/servers/patent-mcp/ directory structure
- [ ] Implement MCP server skeleton
- [ ] Create USPTO LaTeX patent template
- [ ] Implement claims formatting (preamble + limitations)
- [ ] Implement pandoc Markdown → LaTeX conversion
- [ ] Unit tests: claims formatting; LaTeX generation
- [ ] Integration test: Basic patent → PDF

**Week 2: Specification Formatting & Abstract**
- [ ] Implement specification structure (Field, Background, Summary, Detailed Description)
- [ ] Add abstract page generation
- [ ] Add title page with inventor metadata
- [ ] Implement drawing reference list
- [ ] Implement `convert_to_patent` tool
- [ ] Unit tests: specification structure; abstract generation
- [ ] Integration test: Full patent with specification → PDF

**Week 3: Claims Tree Validation**
- [ ] Implement claim dependency graph validation
- [ ] Detect circular dependencies
- [ ] Detect breadth violations (simplified)
- [ ] Implement `validate_patent` tool
- [ ] Unit tests: dependency validation; breadth checking
- [ ] Integration test: Invalid claims tree → errors; valid claims → validation passes

**Week 4: Drawing Reference Validation**
- [ ] Extract drawing references from specification text
- [ ] Validate all referenced drawings exist
- [ ] Detect uncited drawings
- [ ] Add drawing manifest artifact
- [ ] Unit tests: reference extraction; validation
- [ ] Integration test: Mismatched drawings → error; valid references → pass

**Week 5: WIPO Support & Multi-Jurisdiction**
- [ ] Create WIPO (PCT) LaTeX template
- [ ] Add jurisdiction selector logic
- [ ] Implement dual-format output (USPTO + WIPO)
- [ ] Handle jurisdiction-specific formatting differences
- [ ] Unit tests: WIPO template; dual output
- [ ] Integration test: Patent → both USPTO and WIPO PDFs

**Week 6: Claims Tree Visualization & Detailed Validation**
- [ ] Implement claims tree visualization (Mermaid or text-based)
- [ ] Add antecedent basis checking (simplified: word matching)
- [ ] Add term definition detection
- [ ] Unit tests: tree visualization; antecedent checking
- [ ] Integration test: Complex claims tree → visualization

**Week 7: Documentation & Release**
- [ ] Write API documentation
- [ ] Create example patents (software, biotechnology)
- [ ] Write Python processing engine docs
- [ ] Finalize error messages
- [ ] Publish v0.1.0 to npm (@zen-sci/patent-mcp)
- [ ] Full regression test suite (30+ unit + 10+ integration tests)

### 4.3 Dependencies & Prerequisites

**Hard Dependencies:**
- packages/core v1.0
- latex-mcp v0.1

**External Resources:**
- USPTO patent application guidelines
- WIPO PCT application guidelines
- Patent claim writing best practices

---

### 4.4 Testing Strategy

**Unit Tests:** Claims formatting, dependency validation, drawing reference extraction, specification structure validation.

**Integration Tests:** End-to-end patent generation (USPTO + WIPO); claim tree validation; complex specifications with drawings.

---

## 5. Risk Assessment & Mitigation

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Claim breadth validation is too simplistic | Medium | Medium | Start with basic validation; refine iteratively; document limitations |
| Antecedent basis checking is challenging | Medium | Medium | Implement word-matching fallback; manual review checklist |
| WIPO formatting differs significantly from USPTO | Low | Medium | Reference actual PCT guidelines; test with real examples |

---

## 6. Rollback & Contingency

**If claim validation is too complex (Week 3):**
- Defer advanced validation to v0.1.1; ship with basic checks only.

**If WIPO support is problematic (Week 5):**
- Ship USPTO-only in v0.1; add WIPO in v0.1.1.

---

## 7. Appendices

### 7.1 Related Work & Standards

- **USPTO Utility Patent Application Guide:** Official patent application requirements.
- **WIPO PCT Guidelines:** Patent Cooperation Treaty application rules.
- **Patent claim writing standards:** WIPO/USPTO best practices.

### 7.2 Future Considerations (v1.1+)

- **Office action response automation:** Auto-generate responses to USPTO/WIPO office actions.
- **Prior art search integration:** Link to patent databases.
- **Patent term/maintenance fee tracking:** Expiration date calculation.
- **Continued examination applications:** CIP, RCE support.

### 7.3 Open Questions

1. **How sophisticated should claim breadth validation be?** (MVP: basic; warn on potential issues.)
2. **Should we support design patents?** (MVP: utility patents only; design patents later.)

---

**End of patent-mcp Specification**

---

## Audit Notes (2026-02-18)

**Auditor:** Claude (Opus-level audit)
**Status:** Reviewed — fix applied

### Fixed in This Audit
- **MEDIUM-4:** Python code bug on line 461 — `lines.append(..., end='')` corrected to `lines.append(...)`. `list.append()` does not accept `end` keyword argument.

### Remaining Issues
- **CRITICAL-2:** Instantiates `Server` directly instead of extending `ZenSciServer`.
- **CRITICAL-3:** Uses low-level `setRequestHandler` pattern.
- MCP SDK imports correct. Tool naming snake_case — consistent.

#### Decision Propagation (Cruz, 2026-02-18)
- **CRITICAL-2 resolved:** `ZenSciServer` abstract base class deprecated. Use `createZenSciServer()` factory + `McpServer.registerTool()` with Zod schemas at implementation time.
- **CRITICAL-3 resolved:** Low-level `setRequestHandler` replaced by `McpServer.registerTool()` with Zod input schemas per MCP SDK v2.
