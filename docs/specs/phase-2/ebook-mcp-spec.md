# ebook-mcp Spec: Multi-Format Ebook Publisher

**Author:** ZenSci Architecture (TresPiesDesign.com)
**Status:** Draft
**Created:** 2026-02-18
**Phase:** 2 (ships after Phase 1 core is validated)
**Grounded In:** Module strategy scout (2026-02-17), ZenSci semantic clusters

---

## 1. Vision
> Convert Markdown book content to EPUB 3, MOBI, and PDF simultaneously—supporting authors, researchers, and self-publishers with single-source multi-format output.

### The Core Insight
Self-publishing authors manually convert books between formats: EPUB for Amazon/Smashwords, MOBI for Kindle, PDF for archives. Yet **the content is identical**. ebook-mcp unifies the source (Markdown chapters + YAML metadata) and auto-generates all three formats with proper metadata, CSS styling, and navigation.

Unlike generic ebook tools (Calibre, Pandoc alone), ebook-mcp integrates with ZenSci's math rendering (TeX in EPUB via MathJax), bibliography system, and generates production-ready packages ready for distribution.

### What Makes This Different
1. **True multi-format:** EPUB 3 + MOBI + PDF from single source; consistent styling across formats.
2. **Math support:** LaTeX equations in EPUB (via MathJax in HTML5), MOBI (via alt text), PDF (native TeX).
3. **Navigation:** Auto-generate NCX (MOBI) and EPUB nav.xhtml; table of contents.
4. **Metadata:** ISBN, author, publisher info auto-embedded in EPUB/MOBI metadata.
5. **CSS styling:** Professional typography; print-optimized PDF via LaTeX.
6. **Cover art:** Auto-generate or embed cover images (EPUB, MOBI, PDF).

---

## 2. Goals & Success Criteria

### Primary Goals
1. Convert Markdown book → EPUB 3, MOBI, PDF simultaneously.
2. Support 20+ chapter books with cross-references.
3. Embed mathematics (LaTeX) correctly in all three formats.
4. Auto-generate table of contents and navigation files.
5. Support custom CSS styling (colors, fonts, layout).
6. Validate ebook structure (valid EPUB/MOBI packaging).

### Success Criteria
✅ 200-page ebook (20 chapters, 50 citations, 15 math equations) generates all 3 formats in <15s.
✅ EPUB3 validates with epubcheck; MOBI validates with Calibre ebook-convert; PDF is print-ready.
✅ Table of contents auto-generated; navigation works in Kindle, Apple Books, Calibre.
✅ Math equations render correctly in all three formats (MathJax in EPUB/MOBI, TeX in PDF).
✅ ISBN metadata embedded in EPUB/MOBI headers.
✅ End-to-end test: Book source → EPUB + MOBI + PDF; all three valid and readable.

### Non-Goals
❌ Print-on-demand integration (ISBN assignment, distribution).
❌ DRM encryption (digital rights management).
❌ Audio narration or multimedia (ebooks are text + images only).

---

## 3. Technical Architecture

### 3.1 System Overview

```
Markdown Chapters + YAML Metadata
    ↓
ebook-mcp MCP Server (TypeScript)
    ├─ Parse chapters and metadata
    ├─ Validate ebook structure
    ├─ Route to Python engine
    └─ Return DocumentResponse (EPUB + MOBI + PDF)
    ↓
Python Engine (ebooklib + pandoc + TeX)
    ├─ Path A: EPUB 3 (ebooklib + HTML5 + CSS)
    ├─ Path B: MOBI (Calibre ebook-convert or ebooklib)
    ├─ Path C: PDF (pandoc → LaTeX → xelatex)
    └─ Return bytes (3 ebook files)
    ↓
DocumentResponse (format: 'ebook-multi', content: {epub, mobi, pdf})
```

### 3.2 MCP Tool Definitions

#### Tool: `convert_to_ebook`
**Description:** Convert Markdown book to EPUB 3, MOBI, and PDF.

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
          "content": { "type": "string", "description": "Markdown content" }
        },
        "required": ["title", "content"]
      }
    },
    "metadata": {
      "type": "object",
      "properties": {
        "title": { "type": "string" },
        "author": { "type": "string" },
        "publisher": { "type": "string" },
        "isbn": { "type": "string" },
        "description": { "type": "string" },
        "language": { "type": "string", "default": "en" },
        "copyright_year": { "type": "integer" },
        "cover_image_path": { "type": "string" },
        "bibliography": { "type": "string" }
      },
      "required": ["title", "author", "publisher"]
    },
    "format": {
      "type": "string",
      "enum": ["epub", "mobi", "pdf", "all"],
      "description": "Output format(s)."
    },
    "options": {
      "type": "object",
      "properties": {
        "include_toc": { "type": "boolean" },
        "custom_css": { "type": "string", "description": "Custom CSS for styling" },
        "font_name": { "type": "string", "default": "sans-serif" },
        "include_cover": { "type": "boolean" }
      }
    }
  },
  "required": ["chapters", "metadata", "format"]
}
```

**Output Schema:**
```json
{
  "type": "object",
  "properties": {
    "id": { "type": "string" },
    "format": { "type": "string" },
    "content": {
      "type": "string",
      "description": "Base64-encoded ebook file(s); if multi-format, JSON object with {epub, mobi, pdf}"
    },
    "artifacts": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "type": { "type": "string", "enum": ["table_of_contents", "metadata", "validation_report"] },
          "filename": { "type": "string" },
          "content": { "type": "string" }
        }
      }
    },
    "metadata": {
      "type": "object",
      "properties": {
        "chapter_count": { "type": "integer" },
        "word_count": { "type": "integer" },
        "file_sizes": {
          "type": "object",
          "properties": {
            "epub": { "type": "integer" },
            "mobi": { "type": "integer" },
            "pdf": { "type": "integer" }
          }
        },
        "validation_status": { "type": "object" }
      }
    },
    "elapsed": { "type": "number" }
  }
}
```

#### Tool: `validate_ebook`
**Description:** Validate ebook structure and metadata.

**Input Schema:**
```json
{
  "type": "object",
  "properties": {
    "chapters": { "type": "array" },
    "metadata": { "type": "object" },
    "formats": {
      "type": "array",
      "items": { "type": "string", "enum": ["epub", "mobi", "pdf"] }
    },
    "strict_mode": { "type": "boolean" }
  },
  "required": ["chapters", "metadata", "formats"]
}
```

**Output Schema:**
```json
{
  "type": "object",
  "properties": {
    "valid": { "type": "boolean" },
    "errors": { "type": "array", "items": { "type": "object" } },
    "warnings": { "type": "array", "items": { "type": "object" } },
    "report": {
      "type": "object",
      "properties": {
        "chapter_count": { "type": "integer" },
        "word_count": { "type": "integer" },
        "metadata_complete": { "type": "boolean" },
        "isbn_valid": { "type": "boolean" }
      }
    }
  }
}
```

### 3.3 Core Integration Points

- **packages/core:** Bibliography system (CSL-JSON), logging.
- **latex-mcp:** PDF generation pipeline (pandoc → LaTeX → xelatex).
- **blog-mcp:** HTML styling patterns (CSS, responsive design).

---

### 3.4 TypeScript Implementation

```typescript
// zen-sci/packages/sdk/src/servers/ebook-mcp/index.ts

import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from '@modelcontextprotocol/sdk/types.js';

const server = new Server(
  { name: 'ebook-mcp', version: '0.1.0' },
  { capabilities: { tools: {} } }
);

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: 'convert_to_ebook',
      description: 'Convert Markdown book to EPUB 3, MOBI, and/or PDF.',
      inputSchema: {
        type: 'object' as const,
        properties: {
          chapters: { type: 'array' },
          metadata: { type: 'object' },
          format: { type: 'string', enum: ['epub', 'mobi', 'pdf', 'all'] },
          options: { type: 'object' },
        },
        required: ['chapters', 'metadata', 'format'],
      },
    },
    {
      name: 'validate_ebook',
      description: 'Validate ebook structure and metadata.',
      inputSchema: {
        type: 'object' as const,
        properties: {
          chapters: { type: 'array' },
          metadata: { type: 'object' },
          formats: { type: 'array' },
          strict_mode: { type: 'boolean' },
        },
        required: ['chapters', 'metadata', 'formats'],
      },
    },
  ],
}));

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request;

  if (name === 'convert_to_ebook') {
    const result = await convertToEbook(args as any);
    return { content: [{ type: 'text', text: JSON.stringify(result) }] };
  }

  if (name === 'validate_ebook') {
    const result = await validateEbook(args as any);
    return { content: [{ type: 'text', text: JSON.stringify(result) }] };
  }

  throw new Error(`Unknown tool: ${name}`);
});

async function convertToEbook(args: any) {
  return await callPythonEngine('convert_ebook', args);
}

async function validateEbook(args: any) {
  return await callPythonEngine('validate_ebook', args);
}

async function callPythonEngine(method: string, params: any) {
  // Spawn Python subprocess
}

const transport = new StdioServerTransport();
server.connect(transport);
```

### 3.5 Python Processing Engine

```python
# zen-sci/servers/ebook-mcp/processing/ebook_generator.py

import io
import tempfile
from pathlib import Path
from typing import Dict, List, Any, Tuple, Union

from ebooklib import epub
import pandoc

async def generate_ebook_all_formats(
    chapters: List[Dict[str, Any]],
    metadata: Dict[str, Any],
    custom_css: str = None,
    cover_image_path: str = None,
) -> Tuple[Dict[str, bytes], Dict[str, str]]:
    """
    Generate EPUB 3, MOBI, and PDF from book chapters.

    Returns:
        Tuple[{epub: bytes, mobi: bytes, pdf: bytes}, artifacts dict].
    """

    results = {}

    # EPUB 3
    epub_bytes = await generate_epub3(chapters, metadata, custom_css, cover_image_path)
    results['epub'] = epub_bytes

    # MOBI (using Calibre ebook-convert or ebooklib)
    mobi_bytes = await generate_mobi(chapters, metadata, custom_css, cover_image_path)
    results['mobi'] = mobi_bytes

    # PDF (pandoc + xelatex)
    pdf_bytes = await generate_pdf(chapters, metadata)
    results['pdf'] = pdf_bytes

    artifacts = {'table_of_contents': generate_toc(chapters)}

    return results, artifacts


async def generate_epub3(
    chapters: List[Dict[str, Any]],
    metadata: Dict[str, Any],
    custom_css: str = None,
    cover_image_path: str = None,
) -> bytes:
    """Generate EPUB 3 (HTML5 + OPF metadata)."""

    book = epub.EpubBook()

    # Set metadata
    book.set_identifier(metadata.get('isbn', f"uuid-{metadata.get('title', 'ebook')}"))
    book.set_title(metadata['title'])
    book.set_language(metadata.get('language', 'en'))
    book.add_author(metadata['author'])

    if metadata.get('publisher'):
        book.metadata.append(('metadata', 'publisher', metadata['publisher']))

    if metadata.get('copyright_year'):
        book.metadata.append(('metadata', 'date', str(metadata['copyright_year'])))

    # Add cover image if provided
    if cover_image_path:
        cover_path = Path(cover_image_path)
        if cover_path.exists():
            with open(cover_path, 'rb') as f:
                book.set_cover('cover.jpg', f.read())

    # Create chapters as EPUB chapter objects
    chapters_list = []
    for chapter in chapters:
        c = epub.EpubHtml(
            title=chapter['title'],
            file_name=f"chap_{chapter.get('chapter_number', len(chapters_list) + 1):02d}.xhtml",
            lang='en',
        )

        # Convert Markdown to HTML
        html_content = pandoc.convert_text(
            chapter['content'],
            format='markdown',
            to='html',
            extra_args=[
                '--mathjax',  # Enable MathJax in HTML
                '--toc',
            ],
        )

        c.content = html_content
        book.add_item(c)
        chapters_list.append(c)

    # Add custom CSS if provided
    if custom_css:
        c = epub.EpubItem()
        c.set_filename('style/custom.css')
        c.set_content(custom_css)
        book.add_item(c)

    # Add table of contents
    book.toc = chapters_list

    # Create spine (reading order)
    book.spine = ['nav'] + chapters_list

    # Add navigation files
    book.add_item(epub.EpubNcx())  # NCX (MOBI-style navigation)
    book.add_item(epub.EpubNav())   # EPUB3 nav.xhtml

    # Serialize to bytes
    output = io.BytesIO()
    epub.write_epub(output, book, {})
    output.seek(0)
    return output.getvalue()


async def generate_mobi(
    chapters: List[Dict[str, Any]],
    metadata: Dict[str, Any],
    custom_css: str = None,
    cover_image_path: str = None,
) -> bytes:
    """
    Generate MOBI (via Calibre ebook-convert or ebook conversion).
    Note: ebooklib has limited MOBI support; alternative is to generate EPUB then convert.
    """

    # Option 1: Generate EPUB then convert to MOBI using Calibre ebook-convert (external tool)
    epub_bytes = await generate_epub3(chapters, metadata, custom_css, cover_image_path)

    with tempfile.TemporaryDirectory() as tmpdir:
        epub_file = Path(tmpdir) / 'book.epub'
        mobi_file = Path(tmpdir) / 'book.mobi'

        epub_file.write_bytes(epub_bytes)

        # Call Calibre ebook-convert (requires external installation)
        import subprocess
        result = subprocess.run(
            ['ebook-convert', str(epub_file), str(mobi_file)],
            capture_output=True,
            text=True,
        )

        if result.returncode == 0 and mobi_file.exists():
            return mobi_file.read_bytes()

    # Fallback: Return EPUB if Calibre ebook-convert not available
    return epub_bytes


async def generate_pdf(
    chapters: List[Dict[str, Any]],
    metadata: Dict[str, Any],
) -> bytes:
    """Generate PDF (pandoc + xelatex)."""

    # Assemble Markdown from chapters
    markdown_content = f"""---
title: {metadata['title']}
author: {metadata['author']}
date: {metadata.get('copyright_year', '')}
---

"""

    for chapter in chapters:
        markdown_content += f"\n# {chapter['title']}\n\n{chapter['content']}\n\n"

    # Convert to LaTeX
    latex_content = pandoc.convert_text(
        markdown_content,
        format='markdown',
        to='latex',
        extra_args=[
            '--number-sections',
            '--toc',
            '-V', 'documentclass:book',
            '-V', 'geometry:margin=1in',
        ],
        standalone=True,
    )

    # Compile with xelatex
    with tempfile.TemporaryDirectory() as tmpdir:
        tex_file = Path(tmpdir) / 'book.tex'
        pdf_file = Path(tmpdir) / 'book.pdf'

        tex_file.write_text(latex_content, encoding='utf-8')

        # Compile (twice for TOC)
        for _ in range(2):
            result = subprocess.run(
                ['xelatex', '-interaction=nonstopmode', '-output-directory', tmpdir, 'book.tex'],
                cwd=tmpdir,
                capture_output=True,
                text=True,
            )

        if pdf_file.exists():
            return pdf_file.read_bytes()

    raise RuntimeError('PDF generation failed')


def generate_toc(chapters: List[Dict[str, Any]]) -> str:
    """Generate table of contents markdown."""
    lines = ['# Table of Contents\n']

    for chapter in chapters:
        num = chapter.get('chapter_number', len(lines))
        lines.append(f"{num}. {chapter['title']}")

    return '\n'.join(lines)
```

### 3.6 Key Schemas

```typescript
// zen-sci/packages/core/src/schemas/ebook.ts

export interface EbookMetadata {
  title: string;
  author: string;
  publisher: string;
  isbn?: string;
  description?: string;
  language: string; // ISO 639-1 (default: 'en')
  copyright_year?: number;
  cover_image_path?: string;
  bibliography?: string; // CSL-JSON or BibTeX file path
}

export interface EbookChapter {
  chapter_number?: number;
  title: string;
  content: string; // Markdown content
}

export interface EbookDocumentRequest extends DocumentRequest {
  format: 'ebook-epub' | 'ebook-mobi' | 'ebook-pdf' | 'ebook-all';
  chapters: EbookChapter[];
  metadata: EbookMetadata;
  options?: EbookOptions;
}

export interface EbookOptions {
  include_toc?: boolean;
  custom_css?: string;
  font_name?: string;
  include_cover?: boolean;
}

export interface EbookResponse extends DocumentResponse {
  format: 'ebook-epub' | 'ebook-mobi' | 'ebook-pdf' | 'ebook-all';
  content: Buffer | { epub: Buffer; mobi: Buffer; pdf: Buffer };
  artifacts: EbookArtifact[];
  metadata: EbookMetadata & { chapter_count: number; word_count: number };
}

export interface EbookArtifact extends Artifact {
  type: 'table_of_contents' | 'metadata' | 'validation_report';
}
```

---

### 3.7 Phase 2 Infrastructure Requirements

- EPUB 3 validation (epubcheck integration).
- MOBI validation (Calibre ebook-convert or alternative).
- CSS styling framework (responsive, print-optimized).

---

### 3.8 Performance & Constraints

| Constraint | Target | Notes |
|-----------|--------|-------|
| 200-page ebook (20 chapters, 50 citations) | <15s | All 3 formats; parallel generation if possible |
| EPUB generation | <5s | ebooklib + HTML conversion |
| MOBI generation | <5s | Calibre ebook-convert external tool |
| PDF generation | <8s | pandoc + xelatex |
| File sizes | ~2-5 MB each | Compressed; images included |

---

## 4. Implementation Plan

### 4.1 Phased Approach

| Phase | Timeline | Deliverables | Dependencies |
|-------|----------|--------------|--------------|
| **Alpha (Week 1-2)** | 2 weeks | EPUB 3 generation only; basic metadata | packages/core v1.0, blog-mcp v0.2 |
| **Beta (Week 3-4)** | 2 weeks | + MOBI (via external Calibre ebook-convert); TOC generation | All Alpha |
| **Candidate (Week 5-6)** | 2 weeks | + PDF generation; cover art support; multi-format validation | All Beta + latex-mcp v0.1 |
| **Release (Week 7)** | 1 week | Full test suite; documentation | All Candidate |

### 4.2 Week-by-Week Breakdown

**Week 1: EPUB 3 Generation**
- [ ] Set up zen-sci/servers/ebook-mcp/ directory structure
- [ ] Implement MCP server skeleton
- [ ] Build EPUB 3 structure (OPF manifest, NCX, nav.xhtml)
- [ ] Convert Markdown chapters to HTML with MathJax
- [ ] Implement metadata handling (ISBN, author, title)
- [ ] Unit tests: EPUB structure generation; HTML conversion
- [ ] Integration test: 5-chapter book → valid EPUB

**Week 2: CSS Styling & Metadata**
- [ ] Add CSS styling support (custom colors, fonts)
- [ ] Implement cover image embedding
- [ ] Add publisher/copyright metadata
- [ ] Implement `convert_to_ebook` tool (EPUB only)
- [ ] Unit tests: CSS injection; metadata handling; cover art
- [ ] Integration test: Styled EPUB with cover image

**Week 3: MOBI Generation & Table of Contents**
- [ ] Set up Calibre ebook-convert integration (external tool call)
- [ ] Implement MOBI generation (via EPUB → Calibre ebook-convert)
- [ ] Implement table of contents generation
- [ ] Add navigation file generation
- [ ] Unit tests: Calibre ebook-convert integration; TOC generation
- [ ] Integration test: EPUB → MOBI; TOC navigable in Kindle simulator

**Week 4: Validation & Multi-Format**
- [ ] Implement `validate_ebook` tool
- [ ] Add EPUB validation (epubcheck integration)
- [ ] Add MOBI validation
- [ ] Support multi-format output (EPUB + MOBI)
- [ ] Unit tests: validation logic; multi-format output
- [ ] Integration test: Book → both EPUB and MOBI; validate both

**Week 5: PDF Generation**
- [ ] Implement PDF generation (pandoc → LaTeX → xelatex)
- [ ] Create ebook-specific LaTeX template (book class)
- [ ] Add print-optimized PDF styling
- [ ] Implement tri-format generation (EPUB + MOBI + PDF)
- [ ] Unit tests: PDF generation; LaTeX compilation
- [ ] Integration test: Book → all 3 formats

**Week 6: Bibliography & Math Support**
- [ ] Integrate CSL-JSON bibliography support
- [ ] Verify MathJax rendering in EPUB
- [ ] Verify LaTeX math in PDF
- [ ] Test MOBI math rendering (fallback to alt text)
- [ ] Unit tests: bibliography integration; math rendering
- [ ] Integration test: Math-heavy ebook → all 3 formats

**Week 7: Documentation & Release**
- [ ] Write API documentation
- [ ] Create example ebook (academic, fiction)
- [ ] Write Python processing engine docs
- [ ] Test with real-world books
- [ ] Finalize error messages
- [ ] Publish v0.1.0 to npm (@zen-sci/ebook-mcp)
- [ ] Full regression test suite (35+ unit + 10+ integration tests)

### 4.3 Dependencies & Prerequisites

**Hard Dependencies:**
- packages/core v1.0
- blog-mcp v0.2 (HTML styling patterns)
- latex-mcp v0.1 (PDF generation)

**External Tools:**
- ebooklib (EPUB generation)
- Calibre ebook-convert (MOBI generation; requires external installation)
- pandoc ≥2.18
- xelatex

**Optional Tools:**
- epubcheck (EPUB validation)

---

### 4.4 Testing Strategy

**Unit Tests:** EPUB structure generation, MOBI conversion, PDF compilation, metadata handling, CSS injection.

**Integration Tests:** End-to-end ebook generation (all 3 formats); validation of generated files; complex books with math and citations.

---

## 5. Risk Assessment & Mitigation

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Calibre ebook-convert not available in deployment environment | Medium | Medium | Provide fallback (ship EPUB only if Calibre ebook-convert missing); document requirement |
| MOBI math support is limited | Medium | Low | Use alt text for equations in MOBI; document limitation |
| File size/compression issues | Low | Low | Use standard ebook compression; test with large books |

---

## 6. Rollback & Contingency

**If Calibre ebook-convert integration is problematic (Week 3):**
- Ship EPUB + PDF only in v0.1; add MOBI in v0.1.1.

**If PDF generation is complex (Week 5):**
- Ship EPUB + MOBI in v0.1; defer PDF to v0.1.1.

---

## 7. Appendices

### 7.1 Related Work & Standards

- **EPUB 3 Specification:** Official EPUB 3 standard (W3C).
- **MOBI/KF8:** Amazon Kindle formats and Calibre ebook-convert tool.
- **ebooklib:** Python library for EPUB generation.
- **LaTeX book class:** Professional book formatting in LaTeX.

### 7.2 Future Considerations (v1.1+)

- **Print-on-demand integration:** Direct Amazon/IngramSpark formatting.
- **Audio narration:** Audiobook generation (DAISY format).
- **Interactive ebooks:** Embedded JavaScript/interactivity (EPUB 3 Fixed Layout).
- **Annotation support:** Reader highlights and notes (EPUB, MOBI).

### 7.3 Open Questions

1. **Should we auto-generate cover art?** (MVP: accept image file; defer auto-generation to v1.1.)
2. **How detailed should math fallback be in MOBI?** (MVP: alt text; defer better rendering to v1.1.)

---

**End of ebook-mcp Specification**

---

## Audit Notes (2026-02-18)

**Auditor:** Claude (Opus-level audit)
**Status:** Reviewed — fix applied

### Fixed in This Audit
- **MEDIUM-3:** KindleGen references replaced with Calibre `ebook-convert`. Amazon discontinued KindleGen in 2020. Calibre's `ebook-convert` is the maintained alternative for EPUB → MOBI conversion.

### Remaining Issues
- **CRITICAL-2:** Instantiates `Server` directly instead of extending `ZenSciServer`.
- **CRITICAL-3:** Uses low-level `setRequestHandler` pattern.
- ebooklib API usage confirmed correct via Context7 (set_identifier, set_title, EpubHtml, add_item, write_epub pattern).
- MCP SDK imports correct. Tool naming snake_case — consistent.

#### Decision Propagation (Cruz, 2026-02-18)
- **CRITICAL-2 resolved:** `ZenSciServer` abstract base class deprecated. Use `createZenSciServer()` factory + `McpServer.registerTool()` with Zod schemas at implementation time.
- **CRITICAL-3 resolved:** Low-level `setRequestHandler` replaced by `McpServer.registerTool()` with Zod input schemas per MCP SDK v2.
