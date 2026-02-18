# thesis-mcp Spec: University Dissertation Compiler

**Author:** ZenSci Architecture (TresPiesDesign.com)
**Status:** Draft
**Created:** 2026-02-18
**Phase:** 2 (ships after Phase 1 core is validated)
**Grounded In:** Module strategy scout (2026-02-17), ZenSci semantic clusters

---

## 1. Vision
> Convert structured Markdown dissertation (chapters, bibliography, frontmatter) into institution-compliant LaTeX PDF, supporting ProQuest submission and custom institutional templates.

### The Core Insight
PhD students manually format 200+ page dissertations to match institutional requirements: ProQuest standards, committee metadata, abstract format, title pages. Yet the **scientific content is identical** across versions. thesis-mcp unifies the source (Markdown chapters + YAML metadata) and auto-generates compliant PDFs without manual reformatting.

Unlike generic LaTeX templates, thesis-mcp understands dissertation structure: abstract page, committee page, acknowledgments, chapters with cross-referencing, appendices, bibliography—and enforces institution-specific rules (margins, pagination, font sizes).

### What Makes This Different
1. **Institution-aware templates:** Built-in templates for 20+ major universities (MIT, Stanford, Berkeley, etc.); custom template support.
2. **Multi-chapter management:** Automatic chapter numbering, cross-references, and TOC generation.
3. **ProQuest compliance:** Auto-generate ProQuest metadata; validate against submission requirements.
4. **Committee metadata:** Embed advisor/committee names, signatures page, approval dates.
5. **Mathematical dissertation support:** Full TeX/SymPy integration; equation numbering and cross-refs.
6. **Unified bibliography:** CSL-JSON bibliography shared across chapters; auto-formatting per institution.

---

## 2. Goals & Success Criteria

### Primary Goals
1. Accept Markdown chapters + YAML dissertation metadata; auto-generate institution-compliant PDF.
2. Support 5+ institutional templates (MIT, Stanford, Berkeley, Harvard, UCLA).
3. Generate frontmatter (title page, abstract, acknowledgments, committee page) automatically.
4. Manage chapter cross-references, equation numbering, and appendices.
5. Validate against ProQuest and institutional submission requirements.
6. Support math-heavy dissertations (complex equations, derivations, proofs).

### Success Criteria
✅ Full dissertation (200 pages, 5 chapters, 100+ citations, 20+ equations) generates in <15s.
✅ PDF meets institutional formatting requirements (margins, fonts, pagination) without manual tweaks.
✅ ProQuest metadata page is auto-generated and submission-ready.
✅ Cross-references within chapters and across chapters work correctly.
✅ Bibliography is consistent across all chapters; CSL formatting applied correctly.
✅ End-to-end test: MIT dissertation template + 5 chapters → compliant PDF in <15s.

### Non-Goals
❌ Interactive defense scheduling or committee management.
❌ Automatic signature collection (committee signatures are outside scope).
❌ Integration with university registrar systems.

---

## 3. Technical Architecture

### 3.1 System Overview

```
Markdown Chapters + YAML Metadata
    ↓
thesis-mcp MCP Server (TypeScript)
    ├─ Parse chapters and metadata
    ├─ Validate against institution rules
    ├─ Route to Python engine
    └─ Return DocumentResponse (PDF + metadata)
    ↓
Python Engine (pandoc + TeX + SymPy)
    ├─ Assemble chapters (numbering, cross-refs)
    ├─ Generate frontmatter pages
    ├─ Compile LaTeX → PDF (xelatex)
    └─ Return bytes (PDF)
    ↓
DocumentResponse (format: 'thesis-pdf', artifacts: [frontmatter, metadata, ProQuest info])
```

### 3.2 MCP Tool Definitions

#### Tool: `convert_to_thesis`
**Description:** Convert Markdown chapters to institution-compliant dissertation PDF.

**Input Schema:**
```json
{
  "type": "object",
  "properties": {
    "chapters": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "chapter_number": { "type": "integer" },
          "title": { "type": "string" },
          "content": { "type": "string", "description": "Markdown content" },
          "frontmatter": { "type": "boolean", "description": "If true, no chapter number" }
        },
        "required": ["title", "content"]
      }
    },
    "metadata": {
      "type": "object",
      "properties": {
        "title": { "type": "string" },
        "author": { "type": "string" },
        "institution": { "type": "string", "enum": ["mit", "stanford", "berkeley", "harvard", "ucla", "custom"] },
        "degree": { "type": "string", "enum": ["phd", "masters"] },
        "degree_field": { "type": "string" },
        "graduation_date": { "type": "string", "format": "date" },
        "advisors": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "name": { "type": "string" },
              "title": { "type": "string" },
              "role": { "type": "string", "enum": ["primary", "secondary", "committee"] }
            }
          }
        },
        "committee": {
          "type": "array",
          "items": { "type": "string", "description": "Committee member names" }
        },
        "abstract": { "type": "string" },
        "acknowledgments": { "type": "string" },
        "bibliography": { "type": "string", "description": "CSL-JSON or BibTeX file path" },
        "appendices": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "title": { "type": "string" },
              "content": { "type": "string" }
            }
          }
        }
      },
      "required": ["title", "author", "institution", "degree", "degree_field", "abstract"]
    }
  },
  "required": ["chapters", "metadata"]
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
          "type": { "type": "string", "enum": ["frontmatter", "proquests_metadata", "validation_report"] },
          "filename": { "type": "string" },
          "content": { "type": "string" }
        }
      }
    },
    "metadata": {
      "type": "object",
      "properties": {
        "institution": { "type": "string" },
        "page_count": { "type": "integer" },
        "chapter_count": { "type": "integer" },
        "compliance_status": { "type": "string" }
      }
    },
    "elapsed": { "type": "number" }
  }
}
```

#### Tool: `validate_dissertation`
**Description:** Validate dissertation against institution and ProQuest rules.

**Input Schema:**
```json
{
  "type": "object",
  "properties": {
    "chapters": { "type": "array" },
    "metadata": { "type": "object" },
    "strict_mode": { "type": "boolean" }
  },
  "required": ["chapters", "metadata"]
}
```

**Output Schema:**
```json
{
  "type": "object",
  "properties": {
    "compliant": { "type": "boolean" },
    "institution": { "type": "string" },
    "errors": { "type": "array", "items": { "type": "object" } },
    "warnings": { "type": "array", "items": { "type": "object" } }
  }
}
```

#### Tool: `list_thesis_templates`
**Description:** List available institution templates.

**Output Schema:**
```json
{
  "type": "object",
  "properties": {
    "templates": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "institution": { "type": "string" },
          "degree_types": { "type": "array", "items": { "type": "string" } },
          "description": { "type": "string" }
        }
      }
    }
  }
}
```

### 3.3 Core Integration Points

**packages/core DocumentRequest/DocumentResponse:**
- Input: Markdown chapters + metadata (extends DocumentRequest).
- Output: DocumentResponse with PDF artifact + ProQuest metadata.
- Bibliography system: Delegates to packages/core CSL handler.

**latex-mcp dependency (v0.1):**
- TeX compilation pipeline; institution-specific LaTeX templates.

---

### 3.4 TypeScript Implementation

```typescript
// zen-sci/packages/sdk/src/servers/thesis-mcp/index.ts

import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from '@modelcontextprotocol/sdk/types.js';

const server = new Server(
  { name: 'thesis-mcp', version: '0.1.0' },
  { capabilities: { tools: {} } }
);

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: 'convert_to_thesis',
      description: 'Convert Markdown chapters to institution-compliant dissertation PDF.',
      inputSchema: {
        type: 'object' as const,
        properties: {
          chapters: { type: 'array' },
          metadata: { type: 'object' },
        },
        required: ['chapters', 'metadata'],
      },
    },
    {
      name: 'validate_dissertation',
      description: 'Validate dissertation against institution and ProQuest rules.',
      inputSchema: {
        type: 'object' as const,
        properties: {
          chapters: { type: 'array' },
          metadata: { type: 'object' },
          strict_mode: { type: 'boolean' },
        },
        required: ['chapters', 'metadata'],
      },
    },
    {
      name: 'list_thesis_templates',
      description: 'List available institution templates.',
      inputSchema: { type: 'object' as const, properties: {} },
    },
  ],
}));

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request;

  if (name === 'convert_to_thesis') {
    const result = await convertToThesis(args as any);
    return { content: [{ type: 'text', text: JSON.stringify(result) }] };
  }

  if (name === 'validate_dissertation') {
    const result = await validateDissertation(args as any);
    return { content: [{ type: 'text', text: JSON.stringify(result) }] };
  }

  if (name === 'list_thesis_templates') {
    const result = await listThesisTemplates();
    return { content: [{ type: 'text', text: JSON.stringify(result) }] };
  }

  throw new Error(`Unknown tool: ${name}`);
});

async function convertToThesis(args: any) {
  return await callPythonEngine('convert_thesis', args);
}

async function validateDissertation(args: any) {
  return await callPythonEngine('validate_thesis', args);
}

async function listThesisTemplates() {
  return {
    templates: [
      {
        institution: 'MIT',
        degree_types: ['phd', 'masters'],
        description: 'MIT thesis template (from dspace.mit.edu)',
      },
      {
        institution: 'Stanford',
        degree_types: ['phd', 'masters'],
        description: 'Stanford dissertation template',
      },
      // ... more institutions
    ],
  };
}

async function callPythonEngine(method: string, params: any) {
  // Spawn Python subprocess
}

const transport = new StdioServerTransport();
server.connect(transport);
```

### 3.5 Python Processing Engine

```python
# zen-sci/servers/thesis-mcp/processing/thesis_generator.py

import subprocess
import tempfile
from pathlib import Path
from typing import Dict, List, Any, Tuple

import pandoc

async def generate_thesis_pdf(
    chapters: List[Dict[str, Any]],
    metadata: Dict[str, Any],
    bibliography: str = None,
) -> Tuple[bytes, Dict[str, str]]:
    """
    Convert Markdown chapters to institution-compliant dissertation PDF.

    Args:
        chapters: List of chapter dicts (title, content, etc.).
        metadata: Dissertation metadata (author, advisor, institution, etc.).
        bibliography: CSL-JSON bibliography.

    Returns:
        Tuple[PDF bytes, artifacts dict].
    """

    institution = metadata['institution']
    template_path = f'templates/thesis-{institution}.tex'

    # Step 1: Generate frontmatter pages (title, abstract, acknowledgments, committee)
    frontmatter_content = generate_frontmatter(metadata)

    # Step 2: Assemble chapters with numbering and cross-ref setup
    chapters_content = assemble_chapters(chapters)

    # Step 3: Combine frontmatter + chapters + appendices
    full_thesis = f"""{frontmatter_content}

{chapters_content}
"""

    if metadata.get('appendices'):
        appendix_content = generate_appendices(metadata['appendices'])
        full_thesis += f"\n{appendix_content}"

    # Step 4: Convert to LaTeX with pandoc
    pandoc_args = [
        '-f', 'markdown',
        '-t', 'latex',
        f'--template={template_path}',
        '--number-sections',
        '--toc',
        '-V', 'documentclass:book',
        '-V', 'geometry:margin=1in',
        '-V', f'author:{metadata["author"]}',
        '-V', f'title:{metadata["title"]}',
        '--lua-filter=lua-filters/thesis-citations.lua',
    ]

    if bibliography:
        pandoc_args.extend([
            '--citeproc',
            '--bibliography=/dev/stdin',
            '--csl=styles/ieee.csl',  # Institution-specific citation style
        ])

    latex_content = pandoc.convert_text(
        full_thesis,
        format='markdown',
        extra_args=pandoc_args,
        standalone=True,
    )

    # Step 5: Compile LaTeX to PDF
    with tempfile.TemporaryDirectory() as tmpdir:
        tex_file = Path(tmpdir) / 'thesis.tex'
        pdf_file = Path(tmpdir) / 'thesis.pdf'

        tex_file.write_text(latex_content, encoding='utf-8')

        # xelatex compile (3x for TOC, refs, cross-refs)
        for i in range(3):
            result = subprocess.run(
                ['xelatex', '-interaction=nonstopmode', '-output-directory', tmpdir, 'thesis.tex'],
                cwd=tmpdir,
                capture_output=True,
                text=True,
            )
            if result.returncode != 0 and i == 2:
                raise RuntimeError(f'xelatex failed: {result.stderr}')

        if not pdf_file.exists():
            raise RuntimeError('PDF generation failed')

        pdf_bytes = pdf_file.read_bytes()

    # Step 6: Extract ProQuest metadata
    proquests_metadata = generate_proquests_metadata(metadata)

    artifacts = {
        'frontmatter': frontmatter_content,
        'proquests_metadata': proquests_metadata,
    }

    return pdf_bytes, artifacts


def generate_frontmatter(metadata: Dict[str, Any]) -> str:
    """Generate title page, abstract, acknowledgments, committee page."""
    lines = []

    # Title page
    lines.append(f"# {metadata['title']}")
    lines.append('')
    lines.append(f"**{metadata['author']}**")
    lines.append('')
    lines.append(f"{metadata['degree'].upper()} in {metadata['degree_field']}")
    lines.append('')
    lines.append(f"{metadata['institution']}")
    lines.append('')
    lines.append(f"Graduation date: {metadata['graduation_date']}")
    lines.append('')

    # Abstract
    lines.append('## Abstract')
    lines.append('')
    lines.append(metadata.get('abstract', ''))
    lines.append('')

    # Acknowledgments
    if metadata.get('acknowledgments'):
        lines.append('## Acknowledgments')
        lines.append('')
        lines.append(metadata['acknowledgments'])
        lines.append('')

    # Committee page
    if metadata.get('advisors') or metadata.get('committee'):
        lines.append('## Committee')
        lines.append('')
        for advisor in metadata.get('advisors', []):
            lines.append(f"- {advisor['name']} ({advisor.get('title', '')})")
        for member in metadata.get('committee', []):
            lines.append(f"- {member}")
        lines.append('')

    return '\n'.join(lines)


def assemble_chapters(chapters: List[Dict[str, Any]]) -> str:
    """Assemble chapters with numbering and cross-ref setup."""
    lines = []

    for chapter in chapters:
        if chapter.get('frontmatter'):
            # No chapter number for frontmatter chapters
            lines.append(f"## {chapter['title']}")
        else:
            num = chapter.get('chapter_number', 1)
            lines.append(f"# {chapter['title']}")

        lines.append('')
        lines.append(chapter['content'])
        lines.append('')

    return '\n'.join(lines)


def generate_appendices(appendices: List[Dict[str, Any]]) -> str:
    """Generate appendix chapters."""
    lines = ['# Appendices', '']

    for i, appendix in enumerate(appendices, 1):
        lines.append(f"## Appendix {chr(64 + i)}: {appendix['title']}")
        lines.append('')
        lines.append(appendix['content'])
        lines.append('')

    return '\n'.join(lines)


def generate_proquests_metadata(metadata: Dict[str, Any]) -> str:
    """Generate ProQuest submission metadata."""
    lines = [
        'ProQuest Metadata:',
        f"Title: {metadata['title']}",
        f"Author: {metadata['author']}",
        f"Degree: {metadata['degree'].upper()}",
        f"Field: {metadata['degree_field']}",
        f"Institution: {metadata['institution']}",
        f"Graduation Date: {metadata['graduation_date']}",
    ]

    if metadata.get('advisors'):
        lines.append('Advisors:')
        for advisor in metadata['advisors']:
            lines.append(f"  - {advisor['name']}")

    return '\n'.join(lines)
```

### 3.6 Key Schemas

```typescript
// zen-sci/packages/core/src/schemas/thesis.ts

export interface ThesisMetadata {
  title: string;
  author: string;
  institution: 'mit' | 'stanford' | 'berkeley' | 'harvard' | 'ucla' | 'custom';
  degree: 'phd' | 'masters';
  degree_field: string;
  graduation_date: string; // ISO date
  abstract: string;
  acknowledgments?: string;
  advisors: AdvisorInfo[];
  committee: string[];
  bibliography?: string;
  appendices?: AppendixSection[];
}

export interface AdvisorInfo {
  name: string;
  title?: string;
  role: 'primary' | 'secondary' | 'committee';
}

export interface AppendixSection {
  title: string;
  content: string;
}

export interface ThesisDocumentRequest extends DocumentRequest {
  format: 'thesis-pdf';
  chapters: ChapterContent[];
  metadata: ThesisMetadata;
}

export interface ChapterContent {
  chapter_number?: number;
  title: string;
  content: string;
  frontmatter?: boolean;
}

export interface ThesisResponse extends DocumentResponse {
  format: 'thesis-pdf';
  content: Buffer;
  artifacts: ThesisArtifact[];
  metadata: ThesisMetadata;
}

export interface ThesisArtifact extends Artifact {
  type: 'frontmatter' | 'proquests_metadata' | 'validation_report';
}
```

---

### 3.7 Phase 2 Infrastructure Requirements

- Institution template registry (JSON + LaTeX templates per institution).
- ProQuest metadata validation rules.
- Chapter cross-reference system (shared with doc-mcp).

---

### 3.8 Performance & Constraints

| Constraint | Target | Notes |
|-----------|--------|-------|
| Full dissertation (200 pages, 5 chapters) | <15s | xelatex 3x compile for TOC/refs |
| Frontmatter generation | <1s | Template rendering |
| Chapter assembly + cross-refs | <2s | Markdown processing |
| Memory usage | <500 MB | Single dissertation |

---

## 4. Implementation Plan

### 4.1 Phased Approach

| Phase | Timeline | Deliverables | Dependencies |
|-------|----------|--------------|--------------|
| **Alpha (Week 1-2)** | 2 weeks | Basic MIT PDF; no frontmatter synthesis | packages/core v1.0, latex-mcp v0.1 |
| **Beta (Week 3-4)** | 2 weeks | + Frontmatter pages; committee metadata; appendices | All Alpha |
| **Candidate (Week 5-6)** | 2 weeks | + 4 more institutions (Stanford, Berkeley, Harvard, UCLA); ProQuest metadata | All Beta |
| **Release (Week 7)** | 1 week | Full test suite; documentation | All Candidate |

### 4.2 Week-by-Week Breakdown

**Week 1: MIT Template & Chapter Assembly**
- [ ] Set up zen-sci/servers/thesis-mcp/ directory structure
- [ ] Implement MCP server skeleton (TypeScript)
- [ ] Create MIT LaTeX thesis template
- [ ] Build chapter assembly logic (numbering, cross-refs)
- [ ] Implement xelatex compilation pipeline
- [ ] Unit tests: chapter assembly; PDF generation
- [ ] Integration test: 3-chapter dissertation → PDF

**Week 2: Frontmatter & Metadata**
- [ ] Auto-generate title page, abstract page
- [ ] Implement committee page generation
- [ ] Implement acknowledgments section
- [ ] Add advisor/committee metadata handling
- [ ] Implement `convert_to_thesis` tool
- [ ] Unit tests: frontmatter generation; metadata handling
- [ ] Integration test: Full dissertation with frontmatter

**Week 3: Appendices & Cross-References**
- [ ] Implement appendix generation (Appendix A, B, C, ...)
- [ ] Add cross-reference system (chapter-to-chapter, equation cross-refs)
- [ ] Implement table of contents generation
- [ ] Add `validate_dissertation` tool
- [ ] Unit tests: appendix formatting; cross-refs; TOC
- [ ] Integration test: Dissertation with appendices + cross-refs

**Week 4: Stanford, Berkeley, Harvard Templates**
- [ ] Create Stanford dissertation template
- [ ] Create Berkeley dissertation template
- [ ] Create Harvard dissertation template
- [ ] Implement template selector logic
- [ ] Unit tests: all templates compile correctly
- [ ] Integration test: Same dissertation across 4 institutions

**Week 5: UCLA + ProQuest Metadata**
- [ ] Create UCLA dissertation template
- [ ] Implement ProQuest metadata generation
- [ ] Validate against ProQuest submission requirements
- [ ] Unit tests: ProQuest metadata format; validation
- [ ] Integration test: Full dissertation + ProQuest metadata

**Week 6: Bibliography & Edge Cases**
- [ ] Integrate CSL-JSON bibliography per institution style
- [ ] Handle math-heavy chapters (equations, proofs)
- [ ] Test multi-year dissertations with year-specific chapters
- [ ] Unit tests: bibliography formatting; math rendering
- [ ] Integration test: Math-heavy dissertation + citations

**Week 7: Polish & Release**
- [ ] Handle long titles, author names
- [ ] Test with real MIT/Stanford dissertations
- [ ] Write TypeScript/Python API documentation
- [ ] Create example dissertation (all 5 institutions)
- [ ] Finalize error messages
- [ ] Publish v0.1.0 to npm (@zen-sci/thesis-mcp)
- [ ] Full regression test suite (30+ unit + 10+ integration tests)

### 4.3 Dependencies & Prerequisites

**Hard Dependencies:**
- packages/core v1.0 (stable)
- latex-mcp v0.1 (released)

**New Infrastructure:**
- Institution template registry
- ProQuest rules validation

**External Tools:**
- pandoc ≥2.18
- xelatex
- Institution thesis guidelines (MIT, Stanford, Berkeley, Harvard, UCLA)

---

### 4.4 Testing Strategy

**Unit Tests:** Chapter assembly, frontmatter generation, template selection, PDF compilation.

**Integration Tests:** Full dissertations across 5 institutions; ProQuest metadata validation; cross-reference correctness.

---

## 5. Risk Assessment & Mitigation

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Institution rules diverge significantly | Medium | Medium | Template modularization; custom template support |
| xelatex compilation fails on edge cases | Low | Medium | Comprehensive unit tests; fallback to basic template |
| Cross-reference resolution fails | Low | High | Thorough testing; clear error messages |

---

## 6. Rollback & Contingency

**If multi-institution support is complex (Week 4):**
- Ship MIT-only in v0.1; add other institutions in v0.1.1.

---

## 7. Appendices

### 7.1 Related Work & Standards

- **ProQuest:** Largest dissertation database (dissertations.com); submission requirements.
- **MIT DSpace:** MIT thesis format and templates.
- **Stanford Dissertation Guide:** Stanford dissertation formatting requirements.
- **Institutional thesis guides:** Berkeley, Harvard, UCLA theses office guidelines.

### 7.2 Future Considerations (v1.1+)

- **Automated submission to ProQuest:** Direct upload capability.
- **Defense scheduling:** Integrate with committee calendar.
- **Signature collection:** Digital signature integration.

### 7.3 Open Questions

1. **Should we support institutional signing pages (scanned signatures)?** (MVP: text-based committee list.)
2. **How deeply should we customize per institution?** (MVP: margins, fonts, title page layout.)

---

**End of thesis-mcp Specification**

---

## Audit Notes (2026-02-18)

**Auditor:** Claude (Opus-level audit)
**Status:** Reviewed

### Issues
- **CRITICAL-2:** Instantiates `Server` directly instead of extending `ZenSciServer`.
- **CRITICAL-3:** Uses low-level `setRequestHandler` pattern.
- MCP SDK imports correct. Tool naming snake_case — consistent.

#### Decision Propagation (Cruz, 2026-02-18)
- **CRITICAL-2 resolved:** `ZenSciServer` abstract base class deprecated. Use `createZenSciServer()` factory + `McpServer.registerTool()` with Zod schemas at implementation time.
- **CRITICAL-3 resolved:** Low-level `setRequestHandler` replaced by `McpServer.registerTool()` with Zod input schemas per MCP SDK v2.
