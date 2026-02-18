# LaTeX MCP v0.1: PDF Publication from Structured Markdown

**Author:** ZenSci Architecture (TresPiesDesign.com)
**Status:** Draft
**Created:** 2026-02-18
**Grounded In:** ZenSci semantic clusters analysis, module strategy scout (2026-02-17)

---

## 1. Vision

> Transform academic and technical thinking into publication-ready PDF documents through a single markdown entry point, with full support for mathematics, citations, cross-references, and TeX typography.

### The Core Insight

LaTeX MCP is the flagship ZenSci module because it solves a fundamental problem: researchers and technical writers spend cognitive energy managing LaTeX syntax, citation formatting, and compilation workflows—energy that should go into thinking clearly. By accepting clean markdown input with optional frontmatter and BibTeX, we eliminate the need to write TeX directly while retaining complete typographic control when needed.

This module proves the "raw structured thought → publication-ready artifact" thesis. A scientist can paste their research notes (markdown) plus a .bib file and receive a production-grade PDF suitable for submission to conferences, journals, or archival. The math validation (SymPy) and citation management (citeproc) layers ensure correctness before the PDF leaves the engine—no more "rendering failed at 11pm before the deadline."

Most importantly, LaTeX MCP establishes the abstraction boundary: the shared parsing, validation, and citation pipelines in `packages/core` must handle every format converter (blog, slides, newsletter, grant). LaTeX is the most demanding—if the core abstractions work for TeX, they'll work for HTML and Markdown.

### What Makes This Different

Existing solutions fragment across three problematic approaches: (1) online LaTeX editors (Overleaf) that couple editing with compilation and require web access; (2) CLI tools (Pandoc directly) that require users to write shell commands and manage TeX installations; (3) Python-first frameworks (Sphinx, Jupyter) optimized for documentation, not formal publication.

LaTeX MCP is different because it combines three properties:

1. **Format-agnostic input abstraction.** Accept markdown, YAML frontmatter, and BibTeX as separate files. Don't couple the thinking artifact to LaTeX syntax. Users can migrate the same content to HTML, slides, or newsletters later without re-editing.

2. **Modular TeX pipeline.** Pandoc handles markdown → LaTeX conversion; SymPy validates mathematical expressions independently; citeproc generates reference lists in any CSL-JSON style (IEEE, APA, Chicago). Each component is swappable. If a better LaTeX conversion tool emerges, we plug it in without touching the validation layer.

3. **Reproducible compilation.** Deterministic output: given the same markdown, frontmatter, and bibliography, the PDF is byte-identical. This enables caching, diffing, and automated testing—essential for scientific reproducibility.

---

## 2. Goals & Success Criteria

### Primary Goals

1. **Accept markdown + frontmatter + BibTeX and produce a publication-grade PDF** with full TeX typography (font selection, microtyping, kerning, ligatures, justified text).

2. **Validate and render all mathematical content** (inline and display math) using SymPy for symbolic validation and MathJax/KaTeX fallback rendering. Catch malformed expressions at conversion time, not at PDF build.

3. **Manage citations and bibliographies** automatically: parse .bib files, validate references, generate reference lists in multiple CSL styles (IEEE, ACM, Chicago, arXiv), and embed hyperlinks in the final PDF.

4. **Support LaTeX environments** (theorem, proof, lemma, definition, equation, align, etc.) in markdown via a clean fence-based syntax, without forcing users to write raw TeX.

5. **Enable structured cross-references** (chapters, sections, figures, tables, equations) automatically, with proper numbering and hyperlinks in PDF output.

6. **Prove the format-agnostic core abstraction** by implementing the `DocumentRequest` / `DocumentResponse` contract cleanly enough that blog-mcp (v0.2) can reuse 70% of the parsing and validation code.

### Success Criteria

- ✅ CLI tool and MCP server both operational: `zen-sci-latex convert input.md --output output.pdf` and MCP tool invocation produce byte-identical PDFs.
- ✅ Support pdflatex, xelatex, and lualatex as selectable engines; default pdflatex.
- ✅ Markdown input can include fenced code blocks for LaTeX environments (e.g., `\`\`{theorem}...\`\``), and they render in the final PDF with proper numbering.
- ✅ BibTeX entries are validated and resolved; missing references trigger clear warnings, not silent failures.
- ✅ Mathematical expressions (both `$inline$` and `$$display$$`) are syntax-checked via SymPy and render correctly in PDF.
- ✅ Table of contents, section numbering, and page numbers are automatic and correct.
- ✅ PDF includes hyperlinks (internal cross-refs, external citations, URLs).
- ✅ Processing latency is < 5 seconds for typical academic papers (< 50 pages, < 100 references).
- ✅ All warnings and errors are logged with line numbers and actionable guidance.
- ✅ Unit tests cover: markdown → LaTeX conversion, citation resolution, math validation, TeX compilation, schema validation.
- ✅ Integration tests cover: end-to-end workflows (markdown + .bib → PDF); multiple TeX engines; edge cases (empty sections, orphaned citations, Unicode).

### Non-Goals (Out of Scope for this version)

- ❌ GUI editor or web interface. This is a server/CLI tool; editing happens in the user's environment.
- ❌ Real-time preview or watch mode. Focus on single-shot conversion; iteration is the user's responsibility.
- ❌ Custom LaTeX package management. Users can manually add `\usepackage{...}` in frontmatter preamble; we don't vendor or auto-install packages.
- ❌ Collaborative editing or version control integration. ZenSci is format conversion; git is the source of truth.
- ❌ Export to other formats (HTML, DOCX). That's blog-mcp, slides-mcp, etc. Each module owns its output format.
- ❌ Automatic plagiarism detection, statistical analysis, or AI-assisted writing. We're a converter, not a writing assistant.

---

## 3. Technical Architecture

### 3.1 System Overview

LaTeX MCP is organized in three layers:

1. **Markdown Input Layer** (`packages/core`): Parse YAML frontmatter, extract metadata, resolve file references (bibliography, included files), normalize markdown to intermediate representation (IR).

2. **Validation & Transformation Layer** (`packages/core` + `servers/latex-mcp/engine`): Validate math expressions (SymPy), resolve citations (citeproc), transform IR to LaTeX AST, inject cross-reference metadata.

3. **TeX Compilation Layer** (`servers/latex-mcp/engine`): Generate final LaTeX source, invoke pandoc for markdown → LaTeX, compile LaTeX → PDF via pdflatex/xelatex/lualatex, capture warnings/errors, return PDF + metadata.

The **MCP Server** (`servers/latex-mcp/src/index.ts`) registers tools in the Anthropic MCP protocol, routes requests to the Python engine via subprocess, and returns structured responses.

```
User input (markdown + .bib)
           ↓
  MCP Tool Request (JSON)
           ↓
  TypeScript Router (servers/latex-mcp/src/index.ts)
           ↓
  Python Engine subprocess (servers/latex-mcp/engine/latex_engine.py)
           ├─ Parse markdown + frontmatter (packages/core)
           ├─ Validate math (SymPy)
           ├─ Resolve citations (citeproc)
           ├─ Generate LaTeX source
           └─ Compile LaTeX → PDF (pdflatex/xelatex)
           ↓
  DocumentResponse (JSON + PDF path/base64)
           ↓
  Output artifact (PDF file)
```

### 3.2 MCP Tool Definitions

```typescript
// servers/latex-mcp/src/index.ts
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import { CallToolRequestSchema, ListToolsRequestSchema } from '@modelcontextprotocol/sdk/types.js';

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: 'convert_to_pdf',
      description: 'Convert markdown document to publication-ready PDF. Supports YAML frontmatter, BibTeX citations, LaTeX environments, and mathematical expressions.',
      inputSchema: {
        type: 'object',
        properties: {
          source: {
            type: 'string',
            description: 'Markdown content to convert. Can include fenced LaTeX environments (e.g., ```{theorem}..```)'
          },
          title: {
            type: 'string',
            description: 'Document title (required for PDF metadata)'
          },
          author: {
            type: 'array',
            items: { type: 'string' },
            description: 'Author names (for PDF metadata and document header)'
          },
          bibliography: {
            type: 'string',
            description: 'BibTeX content (.bib file contents) containing citations'
          },
          bibliography_style: {
            type: 'string',
            enum: ['ieee', 'acm', 'chicago', 'apa', 'nature', 'arxiv'],
            default: 'ieee',
            description: 'Citation style for bibliography rendering'
          },
          options: {
            type: 'object',
            properties: {
              engine: {
                type: 'string',
                enum: ['pdflatex', 'xelatex', 'lualatex'],
                default: 'pdflatex',
                description: 'LaTeX compiler to use'
              },
              toc: {
                type: 'boolean',
                default: true,
                description: 'Include table of contents'
              },
              line_numbers: {
                type: 'boolean',
                default: false,
                description: 'Enable line numbering (useful for drafts)'
              },
              page_numbers: {
                type: 'boolean',
                default: true,
                description: 'Include page numbers in footer'
              },
              geometry: {
                type: 'object',
                properties: {
                  margin: {
                    type: 'string',
                    default: '1in',
                    description: 'Page margins (e.g., "1in", "2cm")'
                  },
                  papersize: {
                    type: 'string',
                    enum: ['a4', 'letter'],
                    default: 'letter',
                    description: 'Paper size'
                  }
                },
                description: 'Page geometry settings'
              },
              font: {
                type: 'object',
                properties: {
                  main: {
                    type: 'string',
                    default: 'default',
                    description: 'Main font (e.g., "TeX Gyre Termes", "default")'
                  },
                  mono: {
                    type: 'string',
                    default: 'default',
                    description: 'Monospace font for code (e.g., "DejaVu Sans Mono")'
                  }
                },
                description: 'Font configuration'
              },
              draft_mode: {
                type: 'boolean',
                default: false,
                description: 'Enable draft mode (showkeys, no PDF compression)'
              }
            },
            description: 'Compilation options'
          },
          latex_preamble: {
            type: 'string',
            description: 'Custom LaTeX preamble (\\usepackage{...}, \\newcommand, etc.). Appended to generated preamble.'
          },
          output_dir: {
            type: 'string',
            description: 'Output directory for PDF and auxiliary files (default: temp directory)'
          }
        },
        required: ['source', 'title']
      },
      outputSchema: {
        type: 'object',
        properties: {
          pdf_path: {
            type: 'string',
            description: 'Path to generated PDF file'
          },
          pdf_base64: {
            type: 'string',
            description: 'PDF content as base64 string (for embedding in responses)'
          },
          latex_source: {
            type: 'string',
            description: 'Generated LaTeX source code'
          },
          page_count: {
            type: 'integer',
            description: 'Total number of pages in PDF'
          },
          metadata: {
            type: 'object',
            properties: {
              title: { type: 'string' },
              author: { type: 'string' },
              creation_date: { type: 'string' },
              file_size: { type: 'integer', description: 'PDF file size in bytes' }
            },
            description: 'PDF metadata'
          },
          warnings: {
            type: 'array',
            items: {
              type: 'object',
              properties: {
                level: { type: 'string', enum: ['warning', 'error'] },
                message: { type: 'string' },
                line: { type: 'integer' },
                context: { type: 'string' }
              }
            },
            description: 'Conversion warnings and errors'
          },
          citations: {
            type: 'object',
            properties: {
              total: { type: 'integer' },
              resolved: { type: 'integer' },
              unresolved: {
                type: 'array',
                items: { type: 'string' },
                description: 'Citation keys not found in bibliography'
              }
            },
            description: 'Citation statistics'
          },
          elapsed_ms: {
            type: 'number',
            description: 'Total processing time in milliseconds'
          }
        },
        required: ['pdf_path', 'latex_source', 'warnings', 'elapsed_ms']
      }
    },
    {
      name: 'validate_latex_math',
      description: 'Validate mathematical expressions without full document conversion. Useful for checking formulas before committing to PDF generation.',
      inputSchema: {
        type: 'object',
        properties: {
          expressions: {
            type: 'array',
            items: { type: 'string' },
            description: 'LaTeX math expressions to validate (e.g., ["$\\sum_{i=1}^{n} x_i$", "$$\\frac{a}{b}$$"])'
          }
        },
        required: ['expressions']
      },
      outputSchema: {
        type: 'object',
        properties: {
          results: {
            type: 'array',
            items: {
              type: 'object',
              properties: {
                expression: { type: 'string' },
                valid: { type: 'boolean' },
                sympy_form: {
                  type: 'string',
                  description: 'SymPy interpretation of the expression (if valid)'
                },
                error: {
                  type: 'string',
                  description: 'Error message (if invalid)'
                }
              }
            }
          },
          elapsed_ms: { type: 'number' }
        }
      }
    },
    {
      name: 'validate_bibliography',
      description: 'Validate BibTeX file for syntax errors and resolve citation keys.',
      inputSchema: {
        type: 'object',
        properties: {
          bibtex_content: {
            type: 'string',
            description: 'Raw BibTeX file content'
          }
        },
        required: ['bibtex_content']
      },
      outputSchema: {
        type: 'object',
        properties: {
          valid: { type: 'boolean' },
          entries: {
            type: 'array',
            items: {
              type: 'object',
              properties: {
                key: { type: 'string' },
                type: { type: 'string' },
                fields: { type: 'object' }
              }
            },
            description: 'Parsed BibTeX entries'
          },
          errors: {
            type: 'array',
            items: {
              type: 'object',
              properties: {
                line: { type: 'integer' },
                message: { type: 'string' }
              }
            }
          },
          elapsed_ms: { type: 'number' }
        }
      }
    }
  ]
}));
```

### 3.3 Core Integration Points

LaTeX MCP depends on two packages from `packages/core`:

1. **`packages/core/src/parsing/`** — Exports markdown parsing functions and frontmatter extraction:
   - `parseFrontmatter(content: string): { metadata: Record<string, any>, body: string }`
   - `normalizeMarkdown(content: string): NormalizedMD` — Converts fenced LaTeX environments to standardized AST nodes
   - `extractReferences(content: string): ReferenceMap` — Finds all `\cite{...}`, `\ref{...}`, `\label{...}` references

2. **`packages/core/src/validation/`** — Exports validation functions:
   - `validateMathExpression(latex: string): ValidationResult` — Wraps SymPy validation
   - `resolveCitations(normalizedContent: NormalizedMD, bibliography: BibtexDB): CitationMap` — Maps citation keys to resolved entries
   - `checkCrossReferences(normalizedContent: NormalizedMD): ReferenceReport` — Validates all `\ref{}` point to valid labels

3. **`packages/sdk/src/mcp-server.ts`** — Base class `MCPServer`:
   - Handles tool registration, request routing, error serialization
   - Provides `callPythonEngine(script, args, timeout)` method for subprocess invocation

### 3.4 TypeScript Implementation

```typescript
// servers/latex-mcp/src/index.ts
import { MCPServer } from '@zen-sci/sdk';
import { DocumentRequest, DocumentResponse, ConversionWarning } from '@zen-sci/core';
import * as fs from 'fs';
import * as path from 'path';

interface LaTeXDocumentRequest extends DocumentRequest {
  format: 'pdf';
  bibliography?: string;
  bibliography_style: 'ieee' | 'acm' | 'chicago' | 'apa' | 'nature' | 'arxiv';
  latex_preamble?: string;
  options: DocumentOptions & {
    engine: 'pdflatex' | 'xelatex' | 'lualatex';
    toc: boolean;
    line_numbers: boolean;
    page_numbers: boolean;
    draft_mode: boolean;
    geometry: { margin: string; papersize: 'a4' | 'letter' };
    font: { main: string; mono: string };
  };
}

interface LaTeXDocumentResponse extends DocumentResponse {
  format: 'pdf';
  pdfPath: string;
  pdfBase64?: string;
  latexSource: string;
  pageCount: number;
  citationStats: {
    total: number;
    resolved: number;
    unresolved: string[];
  };
}

export class LaTeXMCPServer extends MCPServer {
  constructor() {
    super('latex-mcp', {
      version: '0.1.0',
      homepage: 'https://github.com/TresPies-source/ZenithScience'
    });
  }

  async initialize() {
    await super.initialize();
    this.registerTools();
  }

  private registerTools() {
    this.server.setRequestHandler(ListToolsRequestSchema, async () => ({
      tools: [
        {
          name: 'convert_to_pdf',
          description: 'Convert markdown document to publication-ready PDF...',
          // ... (tool schema from 3.2 above)
          handler: this.handleConvertToPDF.bind(this)
        },
        {
          name: 'validate_latex_math',
          description: 'Validate mathematical expressions...',
          handler: this.handleValidateMath.bind(this)
        },
        {
          name: 'validate_bibliography',
          description: 'Validate BibTeX file...',
          handler: this.handleValidateBibliography.bind(this)
        }
      ]
    }));
  }

  private async handleConvertToPDF(
    input: Record<string, any>
  ): Promise<LaTeXDocumentResponse> {
    const startTime = Date.now();
    const warnings: ConversionWarning[] = [];

    try {
      // Construct DocumentRequest
      const request: LaTeXDocumentRequest = {
        id: input.id || `pdf-${Date.now()}`,
        source: input.source,
        format: 'pdf',
        frontmatter: {
          title: input.title,
          author: input.author || [],
          ...input.metadata
        },
        bibliography: input.bibliography,
        bibliography_style: input.bibliography_style || 'ieee',
        options: {
          engine: input.options?.engine || 'pdflatex',
          toc: input.options?.toc !== false,
          line_numbers: input.options?.line_numbers || false,
          page_numbers: input.options?.page_numbers !== false,
          draft_mode: input.options?.draft_mode || false,
          geometry: input.options?.geometry || {
            margin: '1in',
            papersize: 'letter'
          },
          font: input.options?.font || {
            main: 'default',
            mono: 'default'
          }
        },
        latex_preamble: input.latex_preamble
      };

      // Call Python engine
      const pythonResult = await this.callPythonEngine(
        'servers/latex-mcp/engine/latex_engine.py',
        {
          action: 'convert_to_pdf',
          request
        },
        { timeout: 180000 } // 3 minutes
      );

      if (pythonResult.error) {
        warnings.push({
          level: 'error',
          message: pythonResult.error,
          line: 0,
          context: ''
        });
        throw new Error(pythonResult.error);
      }

      // Parse Python response
      const {
        pdf_path,
        latex_source,
        page_count,
        citation_stats,
        warnings: py_warnings
      } = pythonResult;

      // Include unresolved citations as warnings
      if (citation_stats.unresolved.length > 0) {
        warnings.push({
          level: 'warning',
          message: `Unresolved citations: ${citation_stats.unresolved.join(', ')}`,
          line: 0,
          context: 'Bibliography resolution'
        });
      }

      warnings.push(...(py_warnings || []));

      // Read PDF if output_dir specified, otherwise return path
      let pdfBase64: string | undefined;
      if (input.output_dir && fs.existsSync(pdf_path)) {
        const pdfBuffer = fs.readFileSync(pdf_path);
        pdfBase64 = pdfBuffer.toString('base64');
      }

      const response: LaTeXDocumentResponse = {
        id: request.id,
        requestId: request.id,
        format: 'pdf',
        content: pdfBase64 || pdf_path,
        artifacts: [
          {
            type: 'pdf',
            path: pdf_path,
            mimeType: 'application/pdf',
            size: fs.existsSync(pdf_path) ? fs.statSync(pdf_path).size : 0
          },
          {
            type: 'source',
            path: `${pdf_path}.tex`,
            mimeType: 'text/plain',
            size: latex_source.length
          }
        ],
        warnings,
        metadata: {
          title: input.title,
          author: Array.isArray(input.author) ? input.author.join(', ') : input.author,
          pageCount,
          citationStats: citation_stats
        },
        elapsed: Date.now() - startTime
      };

      return response;
    } catch (error) {
      throw new Error(
        `LaTeX conversion failed: ${error instanceof Error ? error.message : String(error)}`
      );
    }
  }

  private async handleValidateMath(
    input: Record<string, any>
  ): Promise<Record<string, any>> {
    const { expressions } = input;

    const pythonResult = await this.callPythonEngine(
      'servers/latex-mcp/engine/latex_engine.py',
      {
        action: 'validate_math',
        expressions
      },
      { timeout: 30000 }
    );

    return pythonResult;
  }

  private async handleValidateBibliography(
    input: Record<string, any>
  ): Promise<Record<string, any>> {
    const { bibtex_content } = input;

    const pythonResult = await this.callPythonEngine(
      'servers/latex-mcp/engine/latex_engine.py',
      {
        action: 'validate_bibliography',
        bibtex_content
      },
      { timeout: 30000 }
    );

    return pythonResult;
  }
}

// Start server
const server = new LaTeXMCPServer();
server.initialize().then(() => {
  console.log('LaTeX MCP server listening...');
});
```

### 3.5 Python Processing Engine

```python
# servers/latex-mcp/engine/latex_engine.py
import json
import sys
import subprocess
import tempfile
import os
from pathlib import Path
from typing import Dict, Any, List, Optional, Tuple
import logging

# Import from packages/core
sys.path.insert(0, str(Path(__file__).parent.parent.parent.parent / 'core' / 'src'))
from parsing.markdown_parser import parse_frontmatter, normalize_markdown
from validation.math_validator import validate_latex_expression
from validation.citation_resolver import resolve_citations, parse_bibtex

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class LaTeXEngine:
    def __init__(self, compiler: str = 'pdflatex'):
        self.compiler = compiler
        self.verify_compiler()

    def verify_compiler(self):
        """Check if LaTeX compiler is available."""
        result = subprocess.run(
            [self.compiler, '--version'],
            capture_output=True,
            text=True
        )
        if result.returncode != 0:
            raise RuntimeError(f'{self.compiler} not found or not executable')

    def convert_to_pdf(self, request: Dict[str, Any]) -> Dict[str, Any]:
        """Convert markdown document to PDF."""
        try:
            source = request['source']
            title = request['frontmatter'].get('title', 'Untitled')
            author = request['frontmatter'].get('author', [])
            bibliography = request.get('bibliography')
            bib_style = request.get('bibliography_style', 'ieee')
            options = request.get('options', {})
            latex_preamble = request.get('latex_preamble', '')

            compiler = options.get('engine', 'pdflatex')
            toc = options.get('toc', True)
            line_numbers = options.get('line_numbers', False)
            draft_mode = options.get('draft_mode', False)

            with tempfile.TemporaryDirectory() as tmpdir:
                tmpdir_path = Path(tmpdir)

                # Step 1: Parse and validate markdown
                normalized = normalize_markdown(source)

                # Step 2: Validate and resolve citations
                citation_map = {}
                unresolved_citations = []
                if bibliography:
                    try:
                        bib_entries = parse_bibtex(bibliography)
                        citation_map = resolve_citations(normalized, bib_entries)
                        unresolved = normalized.get('citations_used', []) - set(citation_map.keys())
                        unresolved_citations = list(unresolved)
                    except Exception as e:
                        logger.warning(f'Bibliography resolution failed: {e}')
                        unresolved_citations = normalized.get('citations_used', [])

                # Step 3: Validate math expressions
                math_errors = []
                for math_expr in normalized.get('math_expressions', []):
                    result = validate_latex_expression(math_expr)
                    if not result['valid']:
                        math_errors.append({
                            'expression': math_expr,
                            'error': result.get('error', 'Invalid syntax')
                        })

                # Step 4: Generate LaTeX source
                latex_source = self._generate_latex(
                    normalized=normalized,
                    title=title,
                    author=author,
                    citations=citation_map,
                    bib_style=bib_style,
                    preamble=latex_preamble,
                    options=options
                )

                # Step 5: Save LaTeX source
                tex_path = tmpdir_path / 'document.tex'
                tex_path.write_text(latex_source, encoding='utf-8')

                # Step 6: Compile LaTeX to PDF
                pdf_path, compile_warnings = self._compile_latex(
                    tex_path=tex_path,
                    compiler=compiler,
                    tmpdir=tmpdir_path,
                    draft_mode=draft_mode
                )

                # Step 7: Extract PDF metadata
                page_count = self._get_pdf_page_count(pdf_path)

                # Build warnings list
                warnings = []
                if math_errors:
                    for err in math_errors:
                        warnings.append({
                            'level': 'warning',
                            'message': f'Math expression invalid: {err["error"]}',
                            'context': err['expression']
                        })
                if compile_warnings:
                    warnings.extend(compile_warnings)

                return {
                    'pdf_path': str(pdf_path),
                    'latex_source': latex_source,
                    'page_count': page_count,
                    'citation_stats': {
                        'total': len(normalized.get('citations_used', [])),
                        'resolved': len(citation_map),
                        'unresolved': unresolved_citations
                    },
                    'warnings': warnings,
                    'success': True
                }

        except Exception as e:
            logger.error(f'Conversion failed: {e}')
            return {
                'error': str(e),
                'success': False
            }

    def _generate_latex(
        self,
        normalized: Dict[str, Any],
        title: str,
        author: List[str],
        citations: Dict[str, Any],
        bib_style: str,
        preamble: str,
        options: Dict[str, Any]
    ) -> str:
        """Generate LaTeX source from normalized markdown."""

        # Build document class options
        geometry = options.get('geometry', {})
        papersize = geometry.get('papersize', 'letterpaper')
        margin = geometry.get('margin', '1in')

        # Build preamble
        doc_class = f'\\documentclass[{papersize}]{{article}}'

        packages = [
            '\\usepackage[utf8]{inputenc}',
            '\\usepackage[english]{babel}',
            '\\usepackage{geometry}',
            f'\\geometry{{margin={margin}}}',
            '\\usepackage{amsmath, amssymb, amsthm}',
            '\\usepackage{graphicx}',
            '\\usepackage{hyperref}',
            '\\usepackage{natbib}',
            '\\bibliographystyle{' + bib_style + '}',
        ]

        if options.get('line_numbers'):
            packages.append('\\usepackage{lineno}')

        if options.get('font', {}).get('main') != 'default':
            packages.append(f'\\usepackage{{{options["font"]["main"]}}}')

        if options.get('font', {}).get('mono') != 'default':
            packages.append(f'\\usepackage[scale=0.85]{{inconsolata}}')

        if options.get('draft_mode'):
            packages.append('\\usepackage[draft]{showkeys}')

        # Theorems
        preamble_cmds = [
            '\\newtheorem{theorem}{Theorem}',
            '\\newtheorem{lemma}[theorem]{Lemma}',
            '\\newtheorem{corollary}[theorem]{Corollary}',
            '\\newtheorem{definition}[theorem]{Definition}',
            '\\newtheorem{proposition}[theorem]{Proposition}',
        ]

        if preamble:
            preamble_cmds.append(preamble)

        # Build document
        title_block = f'''
\\title{{{title}}}
\\author{{{' and '.join(author) if author else 'Anonymous'}}}
\\date{{\\today}}
'''

        body = normalized.get('body', '')

        # Insert bibliography at end if citations exist
        if citations:
            body += '\n\\bibliographystyle{' + bib_style + '}\n\\bibliography{references}\n'

        latex_doc = '\n'.join([
            doc_class,
            *packages,
            *preamble_cmds,
            '\\begin{document}',
            title_block,
            '\\maketitle',
        ])

        if options.get('toc'):
            latex_doc += '\n\\tableofcontents\n\\newpage\n'

        if options.get('line_numbers'):
            latex_doc += '\\linenumbers\n'

        latex_doc += body + '\n\\end{document}'

        return latex_doc

    def _compile_latex(
        self,
        tex_path: Path,
        compiler: str,
        tmpdir: Path,
        draft_mode: bool
    ) -> Tuple[Path, List[Dict[str, Any]]]:
        """Compile LaTeX to PDF. Returns (pdf_path, warnings)."""

        warnings = []

        # Compile twice for cross-references and TOC
        for attempt in range(2):
            result = subprocess.run(
                [compiler, '-interaction=nonstopmode', '-output-directory', str(tmpdir), str(tex_path)],
                capture_output=True,
                text=True,
                timeout=120
            )

            # Parse warnings from log
            for line in result.stdout.split('\n'):
                if 'Warning' in line or 'warning' in line:
                    warnings.append({
                        'level': 'warning',
                        'message': line.strip(),
                        'context': 'LaTeX compilation'
                    })

        pdf_path = tmpdir / 'document.pdf'
        if not pdf_path.exists():
            raise RuntimeError(f'PDF compilation failed. Log output:\n{result.stdout[-2000:]}')

        return pdf_path, warnings

    def _get_pdf_page_count(self, pdf_path: Path) -> int:
        """Extract page count from PDF."""
        try:
            result = subprocess.run(
                ['pdfinfo', str(pdf_path)],
                capture_output=True,
                text=True,
                timeout=5
            )
            for line in result.stdout.split('\n'):
                if line.startswith('Pages:'):
                    return int(line.split(':')[1].strip())
        except:
            pass
        return 0

    def validate_math(self, expressions: List[str]) -> Dict[str, Any]:
        """Validate LaTeX math expressions."""
        results = []
        for expr in expressions:
            result = validate_latex_expression(expr)
            results.append(result)

        return {
            'results': results,
            'elapsed_ms': 0
        }

    def validate_bibliography(self, bibtex_content: str) -> Dict[str, Any]:
        """Validate BibTeX file."""
        try:
            entries = parse_bibtex(bibtex_content)
            return {
                'valid': True,
                'entries': entries,
                'errors': [],
                'elapsed_ms': 0
            }
        except Exception as e:
            return {
                'valid': False,
                'entries': [],
                'errors': [{'line': 0, 'message': str(e)}],
                'elapsed_ms': 0
            }

def main():
    """Entry point for subprocess invocation."""
    if len(sys.argv) < 2:
        print(json.dumps({'error': 'No action specified'}))
        sys.exit(1)

    action = sys.argv[1]
    input_data = json.loads(sys.stdin.read())

    engine = LaTeXEngine(input_data.get('options', {}).get('engine', 'pdflatex'))

    try:
        if action == 'convert_to_pdf':
            result = engine.convert_to_pdf(input_data['request'])
        elif action == 'validate_math':
            result = engine.validate_math(input_data['expressions'])
        elif action == 'validate_bibliography':
            result = engine.validate_bibliography(input_data['bibtex_content'])
        else:
            result = {'error': f'Unknown action: {action}'}

        print(json.dumps(result))
    except Exception as e:
        print(json.dumps({'error': str(e)}))
        sys.exit(1)

if __name__ == '__main__':
    main()
```

### 3.6 Key Schemas

```typescript
// packages/core/src/types/latex.ts
export interface LaTeXOptions extends DocumentOptions {
  engine: 'pdflatex' | 'xelatex' | 'lualatex';
  toc: boolean;
  line_numbers: boolean;
  page_numbers: boolean;
  draft_mode: boolean;
  geometry: {
    margin: string;
    papersize: 'a4' | 'letter';
  };
  font: {
    main: string;
    mono: string;
  };
}

export interface NormalizedMD {
  body: string;
  metadata: Record<string, unknown>;
  citations_used: Set<string>;
  references: Map<string, ReferenceLocation>;
  math_expressions: string[];
  environments: LaTeXEnvironment[];
}

export interface LaTeXEnvironment {
  type: string; // 'theorem', 'proof', 'lemma', 'equation', etc.
  label?: string;
  content: string;
  line: number;
}

export interface ReferenceLocation {
  label: string;
  type: 'section' | 'figure' | 'table' | 'equation';
  number: number;
  page?: number;
}

export interface BibtexDB {
  entries: Map<string, BibtexEntry>;
  errors: BibtexError[];
}

export interface BibtexEntry {
  key: string;
  type: string;
  fields: Record<string, string>;
}

export interface CitationMap {
  [key: string]: BibtexEntry;
}

export interface ConversionWarning {
  level: 'warning' | 'error';
  message: string;
  line: number;
  context: string;
}
```

### 3.7 Performance & Constraints

- **Processing latency:** < 5 seconds for typical academic papers (< 50 pages, < 100 references). Markdown parsing + validation: < 500ms. Pandoc conversion: < 1s. LaTeX compilation (2 passes): < 3s. Depends on system TeX installation and PDF complexity.
- **File size limits:** Input markdown up to 100MB (practical limit ~10,000 pages). Output PDF capped at 500MB. Bibliography up to 10,000 entries.
- **Concurrency:** Single LaTeX MCP server instance can handle 1 concurrent conversion (TeX compilers are not thread-safe). Multiple servers can run in parallel for horizontal scaling.
- **Memory:** Peak memory ~500MB during compilation of large documents. Temporary files cleaned up in tempdir.
- **Dependencies:** pdflatex/xelatex/lualatex installed and on PATH. Pandoc >= 3.0. Python >= 3.11 with SymPy, citeproc-python. No network calls required.

---

## 4. Implementation Plan

### 4.1 Phased Approach

| Phase | Duration | Focus | Deliverables |
|-------|----------|-------|--------------|
| 1 (Setup) | Week 1 | Project scaffolding, core type definitions, MCP server skeleton | TypeScript boilerplate, MCP tool registration |
| 2 (Markdown → LaTeX) | Week 1-2 | Pandoc integration, markdown normalization, environment fence parsing | Working `convert_to_pdf` tool (basic PDFs without citations/math) |
| 3 (Validation) | Week 2 | Math validation (SymPy), bibliography resolution (citeproc), cross-reference checking | `validate_latex_math` and `validate_bibliography` tools functional |
| 4 (Polish & Testing) | Week 3 | Edge cases, error messages, comprehensive test suite, CLI tool | Production-ready release, full test coverage |

### 4.2 Week-by-Week Breakdown

**Week 1: Project Structure & Core Infrastructure**

- [ ] Initialize `servers/latex-mcp/` directory with TypeScript/Python structure
- [ ] Set up MCP server base class inheritance from `packages/sdk`
- [ ] Define shared type schemas in `packages/core/src/types/latex.ts`
- [ ] Implement `parseFrontmatter` and `normalizeMarkdown` in `packages/core/src/parsing/`
- [ ] Create `packages/core/src/validation/math_validator.py` stub (SymPy integration)
- [ ] Create `packages/core/src/validation/citation_resolver.py` stub (citeproc integration)
- [ ] Set up GitHub repo structure and npm package scaffolding
- [ ] Write unit tests for type definitions and parsing functions

**Week 2: Markdown → LaTeX → PDF Pipeline**

- [ ] Implement `latex_engine.py` with `convert_to_pdf` method
- [ ] Integrate Pandoc for markdown → LaTeX conversion
- [ ] Test Pandoc output with multiple input formats (headings, lists, code blocks, tables)
- [ ] Implement LaTeX environment fence parsing (```{theorem}, ```{proof}, etc.)
- [ ] Implement `_generate_latex` method to build complete TeX document
- [ ] Integrate LaTeX compiler invocation (pdflatex, xelatex, lualatex)
- [ ] Implement TypeScript MCP tool handler for `convert_to_pdf`
- [ ] Write end-to-end tests: markdown → PDF (no citations, basic math)
- [ ] Implement `_get_pdf_page_count` using pdfinfo

**Week 3: Validation & Citation Management**

- [ ] Implement SymPy-based math validator in `packages/core/src/validation/math_validator.py`
- [ ] Integrate citeproc-python for bibliography rendering
- [ ] Implement BibTeX parser (`parse_bibtex` function)
- [ ] Implement citation resolver (matching \cite{} to .bib entries)
- [ ] Build cross-reference checker (`checkCrossReferences`)
- [ ] Implement `validate_latex_math` MCP tool
- [ ] Implement `validate_bibliography` MCP tool
- [ ] Test with real academic papers (ACM, IEEE, arXiv templates)
- [ ] Write tests for unresolved citations, duplicate keys, malformed math

**Week 4: Polish, Testing, Release**

- [ ] Implement CLI tool: `zen-sci-latex convert input.md --output output.pdf`
- [ ] Add comprehensive error messages with line numbers and suggestions
- [ ] Write integration tests for all three MCP tools
- [ ] Test on different TeX installations (minimal, full, MikTeX)
- [ ] Implement PDF metadata extraction (title, author, creation date, file size)
- [ ] Handle edge cases: empty documents, Unicode in titles, special LaTeX characters
- [ ] Write documentation: user guide, API reference, troubleshooting
- [ ] Set up CI/CD pipeline (GitHub Actions) for build + test
- [ ] Release v0.1 to npm: `@zen-sci/latex-mcp`

### 4.3 Dependencies & Prerequisites

**System dependencies:**
- Node.js >= 18, TypeScript >= 5.0
- Python >= 3.11
- LaTeX installation (pdflatex, xelatex, or lualatex)
- Pandoc >= 3.0
- pdfinfo utility (poppler-utils package)

**npm dependencies:**
```json
{
  "@modelcontextprotocol/sdk": "^2.0.0",
  "@zen-sci/core": "workspace:*",
  "@zen-sci/sdk": "workspace:*",
  "typescript": "^5.3.0",
  "ts-node": "^10.9.0"
}
```

**Python dependencies:**
```txt
sympy>=1.12
citeproc-python>=0.7.0
bibtexparser>=2.0.0
pyyaml>=6.0
pdfminer.six>=20221105
```

**Internal dependencies:**
- `packages/core/src/parsing/` — markdown normalization
- `packages/core/src/validation/` — math, citation, cross-reference validators
- `packages/sdk/src/mcp-server.ts` — MCP base class

### 4.4 Testing Strategy

**Unit tests** (each component in isolation):
- Parse frontmatter: valid YAML, missing fields, Unicode
- Normalize markdown: headings, lists, code blocks, LaTeX environments, math expressions
- Validate math: valid expressions, syntax errors, Unicode math characters
- Parse BibTeX: valid entries, duplicate keys, missing fields, malformed lines
- Resolve citations: matching keys, unresolved citations, circular references
- Generate LaTeX: preamble correctness, document structure, hyperlinks

**Integration tests** (end-to-end workflows):
- Markdown + no citations → PDF
- Markdown + BibTeX → PDF with bibliography
- Markdown + math expressions → PDF with correct rendering
- Markdown with cross-references → PDF with hyperlinked TOC and references
- Different TeX engines (pdflatex, xelatex, lualatex) produce valid PDFs
- Edge cases: empty document, very long title, special characters, Unicode

**Performance tests:**
- Small document (< 10 pages): latency < 2s
- Medium document (50 pages): latency < 5s
- Large document (100+ pages): latency < 10s

**Regression tests:**
- Byte-identical PDFs for identical input (deterministic output)
- Output matches reference PDFs generated by hand-written LaTeX

---

## 5. Risk Assessment & Mitigation

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| LaTeX compiler not installed on user system | High | Blocker | Provide comprehensive installation docs; fallback to online service (Overleaf API) if needed; detect missing compiler early with clear error. |
| Pandoc version incompatibility | Medium | High | Pin Pandoc >= 2.19; write version detection; maintain compatibility matrix for different versions. |
| SymPy math validation too slow for large expressions | Medium | Medium | Cache validation results; async validation in background; set timeout (5s) and warn user instead of blocking. |
| BibTeX parsing fails on non-standard formats | Medium | Medium | Use robust parser (bibtexparser); gracefully skip unparseable entries with warnings; provide manual override option. |
| PDF generation produces invisible output (encoding issues) | Low | High | Test with Unicode fonts (xelatex/lualatex); validate font availability; provide fallback fonts. |
| Circular cross-references (labels referencing non-existent chapters) | Medium | Low | Detect cycles during validation; warn user without blocking conversion. |
| Memory exhaustion on very large documents | Low | High | Stream processing where possible; warn user if document > 50MB; implement incremental compilation. |
| MCP tool invocation timeout on slow systems | Medium | Medium | Implement configurable timeout (default 180s); provide progress feedback; allow user to increase timeout in options. |
| Citation key collisions across multiple .bib files | Low | High | Namespace resolution; detect duplicates early; error message suggests renaming entries. |
| Output PDF exceeds file size limits (cloud storage, email) | Low | Medium | Pre-compress PDF; provide optimization options (reduce image resolution); warn user if > 50MB. |

---

## 6. Rollback & Contingency

**If Pandoc conversion fails:**
- Fall back to internal markdown → LaTeX converter (regex-based, handles 80% of cases)
- Return error with suggestion to simplify markdown syntax
- Provide raw LaTeX input option for users who want full control

**If LaTeX compiler not available:**
- Detect at startup; log clear error
- Suggest installation commands for common platforms (apt-get, brew, choco)
- Optionally redirect to Overleaf API (requires user API token)

**If SymPy validation is too slow:**
- Implement caching: memoize validated expressions
- Run validation async; return partial results
- User can disable math validation with `--skip-math-validation` flag

**If citeproc bibliography generation fails:**
- Fall back to simple formatted citations (Author et al., Year)
- Return warning; PDF is still valid but bibliography is unformatted
- Provide `--manual-bib` option for users to supply pre-formatted .bib

**If PDF compilation exceeds timeout:**
- Kill process after 180s
- Return partial LaTeX source for manual debugging
- Suggest user simplify document or increase `--timeout` option

---

## 7. Appendices

### 7.1 Related Work

- **Pandoc** (pandoc.org): Universal document converter. We use Pandoc for markdown → LaTeX; LaTeX MCP adds validation and PDF automation layers.
- **Sphinx** (sphinx-doc.org): Documentation generator with LaTeX backend. Designed for docs, not academic papers. Less flexible citation handling.
- **Overleaf** (overleaf.com): Web-based LaTeX editor. Coupled editing + compilation; requires internet; proprietary.
- **Quarto** (quarto.org): Scientific publishing system. Similar motivation but notebook-first; we're markdown-first and library-based.
- **Typst** (typst.app): Modern typesetting language. Simpler syntax than LaTeX; emerging ecosystem; we target LaTeX for compatibility.

### 7.2 Future Considerations

- **Incremental compilation:** Cache intermediate TeX files to speed up re-renders of unchanged sections
- **Template library:** Pre-built templates for IEEE, ACM, arXiv, Nature, etc., with automatic style-switching
- **Git integration:** Diff PDFs; track changes; version control for manuscripts
- **Real-time preview:** WebSocket-based PDF viewer with live updates as markdown changes
- **Custom fonts:** Font subsetting; embedding; advanced typography (OpenType features)
- **Multi-language support:** Babel configuration; RTL script handling; CJK font management
- **Accessibility:** Tagged PDF for screen readers; alt text for figures; semantic structure
- **Publishing workflow:** Direct submission to arXiv, Overleaf, or journal platforms
- **Collaborative editing:** Real-time sync with other writers; merge conflict resolution

### 7.3 Open Questions

1. Should we support `.md` includes (e.g., `!include other.md`)? Deferred to v0.2.
2. How do we handle custom LaTeX packages not in the base TeX distribution? User provides via preamble for now; auto-install later.
3. Should we generate `.tex` source alongside PDF, or only on request? Both; always generate for reproducibility.
4. How do we version PDFs given the same input? Add git commit hash to PDF metadata if `.git/` exists.
5. Should citations be inline (numbered) or author-date? Configurable via CSL style; default IEEE (numbered).

### 7.4 Schema Backlog (Schemas this module requires)

Schemas defined in `packages/core/src/types/`:

- `DocumentRequest` (shared base)
- `DocumentResponse` (shared base)
- `ConversionWarning` (shared)
- `Artifact` (shared)
- `ResponseMetadata` (shared)
- `LaTeXOptions` (module-specific)
- `NormalizedMD` (module-specific)
- `LaTeXEnvironment` (module-specific)
- `ReferenceLocation` (module-specific)
- `BibtexDB`, `BibtexEntry`, `BibtexError` (module-specific)
- `CitationMap` (module-specific)

All schemas are TypeScript interfaces with JSON Schema equivalents for MCP tool validation.

---

## Audit Notes (2026-02-18)

**Auditor:** Claude (Opus-level audit)
**Status:** Reviewed — fixes applied

### Fixed in This Audit
- **CRITICAL-1:** MCP SDK imports corrected from `@anthropic-ai/sdk` to `@modelcontextprotocol/sdk`.
- **HIGH-3:** Python version standardized to >= 3.11.
- **MEDIUM-5:** Pandoc version standardized to >= 3.0.

### Remaining Issues
- **CRITICAL-2:** Does not extend `ZenSciServer` from SDK spec. Should extend it for shared validation/error handling.
- **CRITICAL-3:** Uses low-level `setRequestHandler` pattern. Acceptable if `ZenSciServer` wraps the MCP protocol.
- Tool naming (`convert_to_pdf`) follows snake_case convention — consistent with Phase 2, good.

#### Decision Propagation (Cruz, 2026-02-18)
- **CRITICAL-2 resolved:** `ZenSciServer` abstract base class deprecated. Use `createZenSciServer()` factory + `McpServer.registerTool()` with Zod schemas at implementation time. Composition over inheritance.
- **CRITICAL-3 resolved:** Low-level `setRequestHandler` replaced by `McpServer.registerTool()` with Zod input schemas per MCP SDK v2.
