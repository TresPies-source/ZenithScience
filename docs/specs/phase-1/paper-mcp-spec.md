# Paper MCP v0.3: Academic Paper Publication with Template Switching

**Author:** ZenSci Architecture (TresPiesDesign.com)
**Status:** Draft
**Created:** 2026-02-18
**Grounded In:** ZenSci semantic clusters analysis, module strategy scout (2026-02-17)

---

## 1. Vision

> Convert structured research notes into venue-specific academic papers using IEEE, ACM, arXiv, and custom LaTeX templates.

### The Core Insight

Paper MCP proves that the ZenSci platform can handle **template-based document generation**—a core requirement for academic publishing. Researchers write their thinking (markdown + structured sections), and Paper MCP automatically formats it for submission to specific venues (IEEE Xplore, ACM Digital Library, arXiv, etc.).

This module reuses the entire LaTeX MCP infrastructure (pandoc, TeX compilation, math/citation validation) and adds a template abstraction layer. Given the same markdown input, users can switch templates and regenerate PDFs for different conferences—IEEE conference → ACM journal → arXiv preprint—without re-editing the content.

Paper MCP is lightweight and builds entirely on top of LaTeX MCP, proving that ZenSci's "format-agnostic abstraction" extends to **style-agnostic abstractions** as well. The core insight is that academic papers are highly structured (abstract, sections, references) but stylistically variable (venue templates). Paper MCP separates content (markdown) from presentation (template).

### What Makes This Different

Existing tools force you into one template: Overleaf's templates are fixed per project; arXiv.org requires LaTeX directly; journal submission portals accept DOCX but often mangle formatting. There's no unified "write once, submit everywhere" tool for academia.

Paper MCP is different because it offers **clean template abstraction**:

1. **Template library.** Ship with IEEE, ACM, arXiv, Nature, Science templates. Users select template via `--template ieee` flag.

2. **Automatic format adaptation.** Abstract, introduction, methodology, results, discussion, conclusion, references automatically map to venue-specific section names and layouts.

3. **Submission-ready PDFs.** Output PDFs meet venue requirements: font sizes, margins, reference formatting, figure placement, page limits enforced automatically.

4. **Style override mechanism.** For custom venues, users provide their own LaTeX template with placeholders; Paper MCP injects structured content into the template.

---

## 2. Goals & Success Criteria

### Primary Goals

1. **Reuse LaTeX MCP infrastructure** (pandoc conversion, TeX compilation, math/citation validation) without duplicating code.

2. **Provide template abstraction layer** that allows users to select venue-specific templates and automatically format papers.

3. **Ship with 5 templates:** IEEE, ACM, arXiv, Nature, Science. Each template defines section names, margin requirements, font sizes, reference style.

4. **Enable custom templates** where users provide their own LaTeX template with placeholders and Paper MCP injects content.

5. **Enforce venue-specific constraints** (e.g., page limits, max figures, reference count limits) and warn users when violated.

6. **Generate submission-ready PDFs** suitable for direct upload to journal/conference submission systems.

### Success Criteria

- ✅ CLI tool operational: `zen-sci-paper convert research.md --template ieee --output paper.pdf` produces IEEE-formatted PDF.
- ✅ All 5 built-in templates work and produce correctly formatted PDFs.
- ✅ Custom template support: users provide `.tex` template with placeholders like `{ABSTRACT}`, `{SECTIONS}`, `{BIBLIOGRAPHY}`.
- ✅ Paper structure auto-detected from markdown (abstract, sections, figures); automatically mapped to template.
- ✅ BibTeX citations resolved and formatted per venue style (IEEE numeric, ACM author-date, etc.).
- ✅ Submission checklist generated for each venue (font sizes, page count, file size, image DPI).
- ✅ Processing latency < 10 seconds for typical paper (< 20 pages, < 50 references).
- ✅ Output PDFs pass submission system validation (correct fonts, metadata, embedded images).
- ✅ Warnings for constraint violations (page count exceeded, figures too low resolution, etc.).
- ✅ Unit and integration tests for all templates.

### Non-Goals (Out of Scope for this version)

- ❌ Full journal submission automation. We generate PDF; users submit manually.
- ❌ Citation style database (beyond shipped templates). Users configure via BibTeX styles.
- ❌ Figure/table auto-formatting. Users provide camera-ready images; we embed them.
- ❌ Collaboration or version control. ZenSci is a formatter; git is the source of truth.
- ❌ Proof-reading, plagiarism detection, or grammar checking. We're a formatter, not a writing assistant.

---

## 3. Technical Architecture

### 3.1 System Overview

Paper MCP is an abstraction layer on top of LaTeX MCP. Instead of converting markdown directly to LaTeX, we:

1. **Parse paper structure** (abstract, sections, figures, references) from markdown.
2. **Select template** based on user choice or venue specification.
3. **Map content to template** by substituting placeholders.
4. **Invoke LaTeX MCP pipeline** to compile to PDF.

```
User input (markdown + template choice)
           ↓
  MCP Tool Request (JSON)
           ↓
  TypeScript Router (servers/paper-mcp/src/index.ts)
           ├─ Parse paper structure
           ├─ Load template
           └─ Map content → template placeholders
           ↓
  LaTeX MCP Engine (reuse servers/latex-mcp/engine)
           ├─ Generate LaTeX from template + content
           ├─ Validate math/citations
           └─ Compile LaTeX → PDF
           ↓
  DocumentResponse (JSON + PDF path)
           ↓
  Output artifact (PDF file)
```

### 3.2 MCP Tool Definitions

```typescript
// servers/paper-mcp/src/index.ts
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: 'convert_to_paper',
      description: 'Convert markdown research article to venue-specific academic paper PDF using IEEE, ACM, arXiv, or custom LaTeX template.',
      inputSchema: {
        type: 'object',
        properties: {
          source: {
            type: 'string',
            description: 'Markdown content. Structure: # Abstract, ## Introduction, ## Methodology, etc. Support figures via ![alt](path).'
          },
          title: {
            type: 'string',
            description: 'Paper title'
          },
          author: {
            type: 'array',
            items: { type: 'string' },
            description: 'Author names'
          },
          abstract: {
            type: 'string',
            description: 'Paper abstract (can be extracted from markdown or provided separately)'
          },
          keywords: {
            type: 'array',
            items: { type: 'string' },
            description: 'Keywords for venue metadata'
          },
          template: {
            type: 'string',
            enum: ['ieee', 'acm', 'arxiv', 'nature', 'science', 'custom'],
            default: 'ieee',
            description: 'Venue template to use'
          },
          custom_template_path: {
            type: 'string',
            description: 'Path to custom LaTeX template (required if template="custom")'
          },
          bibliography: {
            type: 'string',
            description: 'BibTeX content'
          },
          options: {
            type: 'object',
            properties: {
              engine: {
                type: 'string',
                enum: ['pdflatex', 'xelatex', 'lualatex'],
                default: 'pdflatex'
              },
              max_pages: {
                type: 'integer',
                description: 'Maximum page count for venue (e.g., 8 for IEEE conference). Warning if exceeded.'
              },
              include_author_notes: {
                type: 'boolean',
                default: false,
                description: 'Include marginpar author notes (for draft mode)'
              },
              submission_mode: {
                type: 'boolean',
                default: true,
                description: 'Remove watermarks, finalize fonts, compress PDF for submission'
              }
            }
          },
          doi: {
            type: 'string',
            description: 'DOI for venue (optional, for metadata)'
          },
          conference_name: {
            type: 'string',
            description: 'Conference/Journal name (e.g., "IEEE Transactions on Software Engineering")'
          }
        },
        required: ['source', 'title', 'author', 'abstract', 'template']
      },
      outputSchema: {
        type: 'object',
        properties: {
          pdf_path: { type: 'string' },
          latex_source: { type: 'string' },
          page_count: { type: 'integer' },
          submission_checklist: {
            type: 'array',
            items: {
              type: 'object',
              properties: {
                item: { type: 'string' },
                status: { type: 'string', enum: ['pass', 'warning', 'fail'] },
                details: { type: 'string' }
              }
            },
            description: 'Pre-submission checklist for venue'
          },
          template_info: {
            type: 'object',
            properties: {
              name: { type: 'string' },
              version: { type: 'string' },
              margin: { type: 'string' },
              columns: { type: 'integer' },
              max_pages: { type: 'integer' },
              citation_style: { type: 'string' }
            }
          },
          warnings: { type: 'array', items: { type: 'object' } },
          elapsed_ms: { type: 'number' }
        }
      }
    },
    {
      name: 'list_templates',
      description: 'List available paper templates.',
      inputSchema: { type: 'object', properties: {} },
      outputSchema: {
        type: 'object',
        properties: {
          templates: {
            type: 'array',
            items: {
              type: 'object',
              properties: {
                id: { type: 'string' },
                name: { type: 'string' },
                description: { type: 'string' },
                venue_type: { type: 'string' },
                max_pages: { type: 'integer' },
                citation_style: { type: 'string' },
                columns: { type: 'integer' }
              }
            }
          }
        }
      }
    },
    {
      name: 'validate_paper_structure',
      description: 'Validate paper structure (abstract, sections, references) without generating PDF.',
      inputSchema: {
        type: 'object',
        properties: {
          source: { type: 'string' },
          template: { type: 'string' },
          max_pages: { type: 'integer' }
        },
        required: ['source', 'template']
      },
      outputSchema: {
        type: 'object',
        properties: {
          valid: { type: 'boolean' },
          structure: {
            type: 'object',
            properties: {
              has_abstract: { type: 'boolean' },
              sections: { type: 'array', items: { type: 'string' } },
              figure_count: { type: 'integer' },
              table_count: { type: 'integer' },
              citation_count: { type: 'integer' },
              word_count: { type: 'integer' },
              estimated_pages: { type: 'integer' }
            }
          },
          warnings: { type: 'array', items: { type: 'string' } }
        }
      }
    }
  ]
}));
```

### 3.3 Core Integration Points

Paper MCP extends LaTeX MCP by adding a template layer:

1. **Reuse from LaTeX MCP:** `latex_engine.py` for compilation, pandoc for markdown conversion, SymPy for math validation, citeproc for bibliography.

2. **Reuse from packages/core:** Markdown normalization, math/citation validators, cross-reference checking.

3. **New abstractions:**
   - `PaperTemplate` interface (defines venue requirements, placeholder mapping)
   - `PaperStructure` interface (parsed abstract, sections, figures)
   - Template loader that reads `.tex` files and metadata

### 3.4 TypeScript Implementation

```typescript
// servers/paper-mcp/src/index.ts
import { MCPServer } from '@zen-sci/sdk';
import { LaTeXMCPServer } from '@zen-sci/latex-mcp';
import * as fs from 'fs';
import * as path from 'path';

interface PaperTemplate {
  id: string;
  name: string;
  description: string;
  venue_type: 'conference' | 'journal' | 'preprint';
  version: string;
  latex_template: string; // .tex template content
  max_pages: number;
  columns: number;
  margin: string;
  citation_style: 'ieee' | 'acm' | 'natbib-author-date';
  placeholders: {
    abstract: string;
    title: string;
    author: string;
    keywords: string;
    sections: string;
    bibliography: string;
  };
  constraints: {
    max_figures?: number;
    max_tables?: number;
    max_references?: number;
    font_size: string;
    line_spacing: number;
  };
}

interface PaperStructure {
  title: string;
  abstract: string;
  sections: { heading: string; content: string }[];
  keywords: string[];
  figures: { caption: string; path: string }[];
  tables: string[];
  citations: string[];
  word_count: number;
}

export class PaperMCPServer extends MCPServer {
  private templates: Map<string, PaperTemplate> = new Map();
  private latexMCP: LaTeXMCPServer;

  constructor() {
    super('paper-mcp', {
      version: '0.3.0',
      homepage: 'https://github.com/TresPies-source/ZenithScience'
    });
    this.latexMCP = new LaTeXMCPServer();
  }

  async initialize() {
    await super.initialize();
    await this.latexMCP.initialize();
    this.loadTemplates();
    this.registerTools();
  }

  private loadTemplates() {
    const templatesDir = path.join(__dirname, '..', 'templates');
    const templateDirs = fs.readdirSync(templatesDir);

    for (const dir of templateDirs) {
      const metaFile = path.join(templatesDir, dir, 'meta.json');
      const texFile = path.join(templatesDir, dir, 'template.tex');

      if (fs.existsSync(metaFile) && fs.existsSync(texFile)) {
        const meta = JSON.parse(fs.readFileSync(metaFile, 'utf-8'));
        const latex = fs.readFileSync(texFile, 'utf-8');

        const template: PaperTemplate = {
          ...meta,
          latex_template: latex
        };

        this.templates.set(meta.id, template);
      }
    }
  }

  private registerTools() {
    this.server.setRequestHandler(ListToolsRequestSchema, async () => ({
      tools: [
        {
          name: 'convert_to_paper',
          description: 'Convert markdown to venue-specific academic paper...',
          inputSchema: { /* from 3.2 */ },
          handler: this.handleConvertToPaper.bind(this)
        },
        {
          name: 'list_templates',
          description: 'List available templates...',
          inputSchema: { type: 'object', properties: {} },
          handler: this.handleListTemplates.bind(this)
        },
        {
          name: 'validate_paper_structure',
          description: 'Validate paper structure...',
          inputSchema: { /* from 3.2 */ },
          handler: this.handleValidatePaperStructure.bind(this)
        }
      ]
    }));
  }

  private async handleConvertToPaper(input: Record<string, any>): Promise<any> {
    const startTime = Date.now();
    const warnings: any[] = [];

    try {
      const template = this.getTemplate(input.template);
      if (!template && input.template !== 'custom') {
        throw new Error(`Template "${input.template}" not found`);
      }

      if (input.template === 'custom' && !input.custom_template_path) {
        throw new Error('custom_template_path required for custom template');
      }

      // Parse paper structure from markdown
      const structure = this.parsePaperStructure(input.source);

      // Generate LaTeX by substituting placeholders in template
      const latex = this.generateLatexFromTemplate(
        template || { latex_template: fs.readFileSync(input.custom_template_path, 'utf-8') },
        structure,
        input.bibliography
      );

      // Use LaTeX MCP to compile to PDF
      const pdfResult = await this.latexMCP.compileToPDF(
        latex,
        input.title,
        input.author,
        input.options || {}
      );

      // Generate submission checklist
      const checklist = this.generateSubmissionChecklist(
        template,
        structure,
        pdfResult.page_count,
        input.options?.max_pages
      );

      return {
        pdf_path: pdfResult.pdf_path,
        latex_source: latex,
        page_count: pdfResult.page_count,
        submission_checklist: checklist,
        template_info: template ? this.serializeTemplate(template) : {},
        warnings,
        elapsed_ms: Date.now() - startTime,
        success: true
      };
    } catch (error) {
      return {
        error: error instanceof Error ? error.message : String(error),
        success: false
      };
    }
  }

  private handleListTemplates(input: Record<string, any>): any {
    const templates = Array.from(this.templates.values()).map(t => ({
      id: t.id,
      name: t.name,
      description: t.description,
      venue_type: t.venue_type,
      max_pages: t.max_pages,
      citation_style: t.citation_style,
      columns: t.columns
    }));

    return {
      templates,
      count: templates.length
    };
  }

  private handleValidatePaperStructure(input: Record<string, any>): any {
    try {
      const structure = this.parsePaperStructure(input.source);
      const warnings: string[] = [];

      if (!structure.abstract) {
        warnings.push('No abstract found. Add "# Abstract" section.');
      }

      if (structure.sections.length === 0) {
        warnings.push('No main sections found. Add "## Introduction", "## Methodology", etc.');
      }

      if (input.template) {
        const template = this.getTemplate(input.template);
        if (template) {
          if (structure.figures.length > (template.constraints.max_figures || 10)) {
            warnings.push(
              `Too many figures (${structure.figures.length}/${template.constraints.max_figures})`
            );
          }
        }
      }

      const estimatedPages = Math.ceil(structure.word_count / 300); // ~300 words per page
      if (input.max_pages && estimatedPages > input.max_pages) {
        warnings.push(
          `Estimated ${estimatedPages} pages exceeds max_pages limit of ${input.max_pages}`
        );
      }

      return {
        valid: warnings.length === 0,
        structure: {
          has_abstract: !!structure.abstract,
          sections: structure.sections.map(s => s.heading),
          figure_count: structure.figures.length,
          table_count: structure.tables.length,
          citation_count: structure.citations.length,
          word_count: structure.word_count,
          estimated_pages: estimatedPages
        },
        warnings
      };
    } catch (error) {
      return {
        valid: false,
        error: error instanceof Error ? error.message : String(error)
      };
    }
  }

  private parsePaperStructure(markdown: string): PaperStructure {
    // Parse markdown to extract abstract, sections, figures, tables
    const lines = markdown.split('\n');
    const structure: PaperStructure = {
      title: '',
      abstract: '',
      sections: [],
      keywords: [],
      figures: [],
      tables: [],
      citations: [],
      word_count: 0
    };

    let currentSection = '';
    let currentContent = '';
    let inAbstract = false;

    for (const line of lines) {
      if (line.startsWith('# Abstract')) {
        inAbstract = true;
      } else if (line.startsWith('# ') || line.startsWith('## ')) {
        if (currentSection && currentContent) {
          structure.sections.push({ heading: currentSection, content: currentContent });
        }
        currentSection = line.replace(/^#+\s+/, '');
        currentContent = '';
        inAbstract = false;
      } else if (line.startsWith('![')) {
        const match = line.match(/!\[(.*?)\]\((.*?)\)/);
        if (match) {
          structure.figures.push({ caption: match[1], path: match[2] });
        }
      } else {
        if (inAbstract) {
          structure.abstract += line + '\n';
        } else if (currentSection) {
          currentContent += line + '\n';
        }
      }

      // Count citations
      const citeMatches = line.match(/\\cite\{([^}]+)\}/g);
      if (citeMatches) {
        structure.citations.push(...citeMatches);
      }
    }

    if (currentSection && currentContent) {
      structure.sections.push({ heading: currentSection, content: currentContent });
    }

    structure.word_count = markdown.split(/\s+/).length;
    return structure;
  }

  private generateLatexFromTemplate(
    template: PaperTemplate,
    structure: PaperStructure,
    bibliography?: string
  ): string {
    let latex = template.latex_template;

    // Replace placeholders
    latex = latex.replace('{TITLE}', structure.title);
    latex = latex.replace('{AUTHOR}', 'TBD');
    latex = latex.replace('{ABSTRACT}', structure.abstract);

    // Build sections
    const sectionTexs = structure.sections
      .map(s => `\\section{${s.heading}}\n${s.content}`)
      .join('\n\n');
    latex = latex.replace('{SECTIONS}', sectionTexs);

    // Inject bibliography
    if (bibliography) {
      latex = latex.replace('{BIBLIOGRAPHY}', '\\bibliographystyle{ieeetr}\n\\bibliography{references}');
    }

    return latex;
  }

  private generateSubmissionChecklist(
    template: PaperTemplate | undefined,
    structure: PaperStructure,
    pageCount: number,
    maxPages?: number
  ): any[] {
    const checklist: any[] = [];

    if (template) {
      // Page count
      checklist.push({
        item: `Page count (max ${template.max_pages})`,
        status: pageCount <= template.max_pages ? 'pass' : 'fail',
        details: `${pageCount}/${template.max_pages} pages`
      });

      // Figures
      if (template.constraints.max_figures) {
        checklist.push({
          item: `Figure count (max ${template.constraints.max_figures})`,
          status: structure.figures.length <= template.constraints.max_figures ? 'pass' : 'warning',
          details: `${structure.figures.length}/${template.constraints.max_figures} figures`
        });
      }
    }

    // Abstract present
    checklist.push({
      item: 'Abstract provided',
      status: structure.abstract ? 'pass' : 'fail',
      details: structure.abstract ? 'Yes' : 'No'
    });

    // Keywords
    checklist.push({
      item: 'Keywords provided',
      status: structure.keywords.length > 0 ? 'pass' : 'warning',
      details: structure.keywords.length > 0 ? `${structure.keywords.length} keywords` : 'None'
    });

    // References
    checklist.push({
      item: 'References included',
      status: structure.citations.length > 0 ? 'pass' : 'warning',
      details: `${structure.citations.length} citations`
    });

    return checklist;
  }

  private getTemplate(id: string): PaperTemplate | undefined {
    return this.templates.get(id);
  }

  private serializeTemplate(template: PaperTemplate): any {
    return {
      name: template.name,
      version: template.version,
      margin: template.margin,
      columns: template.columns,
      max_pages: template.max_pages,
      citation_style: template.citation_style
    };
  }
}

const server = new PaperMCPServer();
server.initialize().then(() => {
  console.log('Paper MCP server listening...');
});
```

### 3.5 Python Processing Engine

```python
# servers/paper-mcp/engine/paper_engine.py
# (Minimal Python—all heavy lifting delegated to LaTeX MCP)
# This is a thin wrapper that prepares the LaTeX template for compilation.

import json
import sys
from pathlib import Path
from typing import Dict, Any

def prepare_paper_latex(template_id: str, content: Dict[str, Any]) -> str:
    """Prepare LaTeX source for paper template."""
    # Load template
    template_path = Path(__file__).parent / f'templates/{template_id}/template.tex'

    if not template_path.exists():
        raise FileNotFoundError(f'Template {template_id} not found')

    template = template_path.read_text(encoding='utf-8')

    # Substitute placeholders
    latex = template
    latex = latex.replace('{TITLE}', content.get('title', 'Untitled'))
    latex = latex.replace('{AUTHOR}', ' and '.join(content.get('author', [])))
    latex = latex.replace('{ABSTRACT}', content.get('abstract', ''))
    latex = latex.replace('{SECTIONS}', content.get('body', ''))
    latex = latex.replace('{KEYWORDS}', ', '.join(content.get('keywords', [])))

    return latex

def main():
    """Entry point for subprocess invocation."""
    if len(sys.argv) < 2:
        print(json.dumps({'error': 'No action specified'}))
        sys.exit(1)

    action = sys.argv[1]
    input_data = json.loads(sys.stdin.read())

    try:
        if action == 'prepare_template':
            latex = prepare_paper_latex(
                input_data['template_id'],
                input_data['content']
            )
            print(json.dumps({'latex': latex}))
        else:
            print(json.dumps({'error': f'Unknown action: {action}'}))
    except Exception as e:
        print(json.dumps({'error': str(e)}))
        sys.exit(1)

if __name__ == '__main__':
    main()
```

### 3.6 Key Schemas

```typescript
// servers/paper-mcp/src/types.ts
export interface PaperTemplate {
  id: string;
  name: string;
  description: string;
  venue_type: 'conference' | 'journal' | 'preprint';
  version: string;
  latex_template: string;
  max_pages: number;
  columns: number;
  margin: string;
  citation_style: 'ieee' | 'acm' | 'natbib-author-date';
  placeholders: {
    abstract: string;
    title: string;
    author: string;
    keywords: string;
    sections: string;
    bibliography: string;
  };
  constraints: {
    max_figures?: number;
    max_tables?: number;
    max_references?: number;
    font_size: string;
    line_spacing: number;
  };
}

export interface PaperStructure {
  title: string;
  abstract: string;
  sections: Array<{ heading: string; content: string }>;
  keywords: string[];
  figures: Array<{ caption: string; path: string }>;
  tables: string[];
  citations: string[];
  word_count: number;
}

export interface SubmissionChecklist {
  item: string;
  status: 'pass' | 'warning' | 'fail';
  details: string;
}
```

### 3.7 Performance & Constraints

- **Processing latency:** < 10 seconds (reuses LaTeX MCP compilation). Overhead is template loading + placeholder substitution (~100ms).
- **File size limits:** Same as LaTeX MCP (< 500MB output PDF).
- **Template management:** Ship with 5 built-in templates; users can add custom templates in `servers/paper-mcp/templates/` directory.
- **Concurrency:** Same as LaTeX MCP (single TeX compiler; multiple servers for horizontal scaling).

---

## 4. Implementation Plan

### 4.1 Phased Approach

| Phase | Duration | Focus | Deliverables |
|-------|----------|-------|--------------|
| 1 (Setup) | Week 1 | Reuse LaTeX MCP, build template abstraction | TypeScript scaffold, template interface |
| 2 (Templates) | Week 1-2 | Ship IEEE, ACM, arXiv templates | 3-5 working templates with test PDFs |
| 3 (Validation) | Week 2 | Paper structure parsing, submission checklist | `validate_paper_structure` tool functional |
| 4 (Polish & Test) | Week 3 | Edge cases, comprehensive testing | Production release |

### 4.2 Week-by-Week Breakdown

**Week 1: Template Abstraction**

- [ ] Create `PaperTemplate` interface and template loader
- [ ] Set up `servers/paper-mcp/templates/` directory structure
- [ ] Implement template metadata (`.json`) parsing
- [ ] Implement placeholder substitution logic
- [ ] Create `list_templates` MCP tool
- [ ] Write unit tests for template loading and placeholder substitution

**Week 2: Template Content**

- [ ] Obtain/create IEEE template from official source
- [ ] Obtain/create ACM template from official source
- [ ] Obtain/create arXiv template from official source
- [ ] Add Nature and Science templates
- [ ] Create `meta.json` for each template (max_pages, constraints, citation_style)
- [ ] Test each template: markdown → PDF with correct formatting
- [ ] Verify fonts, margins, column layout, reference style

**Week 3: Paper Structure & Validation**

- [ ] Implement `parsePaperStructure` to extract abstract, sections, figures, citations from markdown
- [ ] Implement `generateSubmissionChecklist` with venue-specific constraints
- [ ] Implement `validate_paper_structure` MCP tool
- [ ] Test on real papers (arXiv, IEEE, ACM)
- [ ] Generate checklist for each venue

**Week 4: Integration & Release**

- [ ] Integrate with LaTeX MCP for PDF compilation
- [ ] Implement `convert_to_paper` end-to-end
- [ ] Test all templates with sample papers
- [ ] Implement CLI tool: `zen-sci-paper convert research.md --template ieee`
- [ ] Write comprehensive test suite
- [ ] Document user guide
- [ ] Release v0.3

### 4.3 Dependencies & Prerequisites

**System dependencies:**
- Node.js >= 18, TypeScript >= 5.0
- LaTeX installation (pdflatex, xelatex, or lualatex)

**npm dependencies:**
```json
{
  "@modelcontextprotocol/sdk": "^2.0.0",
  "@zen-sci/core": "workspace:*",
  "@zen-sci/sdk": "workspace:*",
  "@zen-sci/latex-mcp": "workspace:*"
}
```

**Internal dependencies:**
- `servers/latex-mcp/` — LaTeX compilation engine
- `packages/core/src/parsing/` — markdown normalization
- `packages/core/src/validation/` — math, citation validators

### 4.4 Testing Strategy

**Unit tests:**
- Template loading and caching
- Placeholder substitution (correct {PLACEHOLDER} replacement)
- Paper structure parsing (abstract, sections, figures, citations)
- Submission checklist generation

**Integration tests:**
- Markdown → template-substituted LaTeX → PDF for each template
- Constraint validation (page count, figure count, reference count)
- Multiple templates with same markdown produce different PDFs

**Venue-specific tests:**
- IEEE template: verify IEEE citation style, 2-column layout, page limit
- ACM template: verify ACM citation style, margins, section numbering
- arXiv template: verify single-column, standard fonts, no page limit

---

## 5. Risk Assessment & Mitigation

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| Official venue templates unavailable or outdated | Medium | High | Create templates from published examples; document template version. Maintain community-contributed templates. |
| Placeholder substitution misses edge cases | Low | Medium | Comprehensive test suite with real papers. Use string replace carefully; escape special LaTeX chars. |
| Paper structure detection fails on non-standard markdown | Medium | Medium | Use robust parser; provide manual override (--abstract-file, --sections-file). |
| Constraint violations not caught (page count, figure count) | Low | High | Generate detailed submission checklist. Warn prominently; block PDF generation if --strict flag set. |
| LaTeX MCP compilation failure propagates | Medium | Medium | Graceful error handling; return detailed error from LaTeX engine. |

---

## 6. Rollback & Contingency

**If template loading fails:**
- Return error with suggestion to check template directory
- Provide fallback to basic article class (no fancy formatting)

**If placeholder substitution fails:**
- Return unmodified template + warning
- Suggest user manually edit LaTeX

**If LaTeX compilation fails:**
- Delegate to LaTeX MCP error handling
- Return detailed error message from compiler

**If constraint violations detected:**
- Return PDF + warnings; don't block generation
- Suggest user revise content (remove figures, condense text)

---

## 7. Appendices

### 7.1 Related Work

- **Official venue templates:** IEEE, ACM, Nature, Science provide LaTeX templates. Paper MCP wraps them.
- **Overleaf:** Hosts templates; we offer a programmatic interface instead of web UI.
- **Pandoc + custom templates:** Pandoc supports custom templates; Paper MCP adds template library + venue constraints.

### 7.2 Future Considerations

- **Template versioning:** Track venue template changes; notify users of updates.
- **Real-time collaboration:** Multi-author editing (deferred).
- **Author affiliation management:** Automatic formatting of author addresses per venue.
- **Figure/table auto-formatting:** Convert Markdown tables to LaTeX tables; optimize image resolution.
- **Journal submission automation:** Auto-submit to journal systems (requires credentials).
- **Preprint archival:** Auto-submit to arXiv (requires login).

### 7.3 Open Questions

1. Should we ship with all 5 templates or allow gradual expansion? Start with IEEE + arXiv; expand as demand grows.
2. How do we handle custom author data (affiliations, emails, ORCIDs)? Via YAML frontmatter for now.
3. Should we enforce page limits or just warn? Warn only; let users decide.
4. Do we support multiple author blocks (main + corresponding author)? Deferred to v0.4.

### 7.4 Schema Backlog

Schemas defined in `servers/paper-mcp/src/types.ts`:

- `PaperTemplate` (module-specific)
- `PaperStructure` (module-specific)
- `SubmissionChecklist` (module-specific)

---

## Audit Notes (2026-02-18)

**Auditor:** Claude (Opus-level audit)
**Status:** Reviewed — fixes applied

### Fixed in This Audit
- **CRITICAL-1:** Package.json dependency corrected from `@anthropic-ai/sdk` to `@modelcontextprotocol/sdk`.

### Remaining Issues
- Imports `MCPServer` from `@zen-sci/sdk` and extends it — closest to the intended `ZenSciServer` pattern but uses a different class name. Rename to `ZenSciServer` when SDK spec is finalized.
- Tool naming (`convert_to_paper`, `validate_paper_structure`) follows snake_case — consistent.

#### Decision Propagation (Cruz, 2026-02-18)
- **CRITICAL-2 resolved:** `ZenSciServer` abstract base class deprecated. paper-mcp's `MCPServer` extends pattern should be replaced with `createZenSciServer()` factory + `McpServer.registerTool()` with Zod schemas at implementation time. No rename needed — composition replaces inheritance.
- **CRITICAL-3 resolved:** Low-level `setRequestHandler` replaced by `McpServer.registerTool()` with Zod input schemas per MCP SDK v2.
