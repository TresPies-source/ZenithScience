# slides-mcp Spec: Interactive & Printable Presentations

**Author:** ZenSci Architecture (TresPiesDesign.com)
**Status:** Draft
**Created:** 2026-02-18
**Phase:** 2 (ships after Phase 1 core is validated)
**Grounded In:** Module strategy scout (2026-02-17), ZenSci semantic clusters

---

## 1. Vision
> Turn scientific markdown with embedded math, citations, and speaker notes into dual-format presentations: Beamer PDF (formal, printable) and Reveal.js HTML (interactive, web-native).

### The Core Insight
Researchers and educators spend weeks manually converting slides between formats (PDF for conferences, HTML for web). Yet most presentation content—structure, math, citations—is **identical**. slides-mcp unifies the source, splits the output: one Markdown input → polished PDF + interactive web simultaneously.

Unlike generic Markdown-to-slides tools (Marp, Slidev), slides-mcp leverages ZenSci's math-first architecture (TeX rendering, symbolic manipulation) and packages/core's bibliography system. This enables academic rigor: proper citation handling, complex equations, speaker notes with mathematical derivations.

### What Makes This Different
1. **Dual-format parity:** Beamer (institutional, print) and Reveal.js (web, interactive) output quality on par with handcrafted slides.
2. **Math-native:** Inline and display math in both formats; optionally generate step-by-step derivations for speaker notes.
3. **Citation sync:** Frontmatter bibliography auto-populates reference slides; citations in notes stay synchronized.
4. **Speaker notes with depth:** Extended notes in Markdown become detailed speaker guides in PDF (Beamer notes page) and hidden HTML (reveal.js).
5. **Themeing:** Pre-built themes (metropolis, berlin for Beamer; css, sky, league for Reveal.js) with institutional customization hooks.

---

## 2. Goals & Success Criteria

### Primary Goals
1. Accept Markdown with YAML frontmatter + math + citations; output Beamer PDF + Reveal.js HTML without user rewriting.
2. Enable speaker notes (detailed, with formulas and derivations) that appear in PDF notes pages and HTML speaker view.
3. Provide 5+ institutional themes (metropolis, berlin, warsaw for Beamer; dark, light, minimal for Reveal.js).
4. Support transitions, animations, and fragment reveals in Reveal.js; maintain print-ready PDF from Beamer.
5. Handle equation-heavy content (multi-line derivations, numbered equations, cross-references).
6. Integrate with packages/core bibliography system for consistent citations.

### Success Criteria
✅ Beamer PDF output passes institutional slide requirements (readable fonts, proper spacing, math renders correctly).
✅ Reveal.js HTML passes WCAG 2.1 AA accessibility; speaker notes visible in speaker view; presenter timer works.
✅ Math rendering identical in both formats (TeX in Beamer, MathJax in Reveal.js); aligned equations don't break layout.
✅ Speaker notes (300+ words) don't impact slide print time or web load (asynchronous loading in HTML).
✅ Theme switching <100 milliseconds in Reveal.js; PDF generation <5s for 30-slide deck.
✅ End-to-end test: Convert 20-slide conference talk (with citations, speaker notes, math) in <10s.

### Non-Goals
❌ Export to PowerPoint or Google Slides (format parity is lower priority than PDF/HTML quality).
❌ Live collaborative editing in web UI (Claude Code use case is single-user conversion).
❌ Animation choreography UI (reveal.js supports animation syntax, user specifies in Markdown).
❌ Drag-and-drop image placement (Markdown simplicity over WYSIWYG).

---

## 3. Technical Architecture

### 3.1 System Overview

```
Markdown Input (YAML frontmatter + math + citations)
    ↓
slides-mcp MCP Server (TypeScript)
    ├─ Parse frontmatter (title, author, theme, format)
    ├─ Validate math + citation syntax
    ├─ Route to Python engine
    └─ Return DocumentResponse (PDF + HTML + metadata)
    ↓
Python Engine (pandoc + TeX + reveal.js-builder)
    ├─ Path A: Beamer (pandoc → .tex, xelatex → PDF)
    ├─ Path B: Reveal.js (pandoc + custom lua filters → .html, inject MathJax)
    ├─ Both: Speaker notes integration, bibliography formatting
    └─ Return bytes (PDF + HTML)
    ↓
DocumentResponse (format: 'slides-beamer' | 'slides-html', content: Buffer, artifacts: [speaker notes, bibliography])
```

### 3.2 MCP Tool Definitions

> **Implementation note (2026-02-18):** The tool contracts below reflect the **implemented** design. Key divergences from the original spec:
>
> 1. **`validate_slides_source` → `validate_deck`**: The tool was renamed to `validate_deck` for conciseness; the original `list_slide_themes` tool name is unchanged but the `format` enum value changed from `'reveal'` to `'revealjs'` for consistency with the renderer name.
>
> 2. **`format` enum**: `'reveal'` renamed to `'revealjs'`; `'both'` format added to `convert_to_slides` to render both Beamer and Reveal.js simultaneously.
>
> 3. **`theme` moved into `options`**: The original spec had `theme` as a top-level sibling of `source`/`format`. The implementation puts it inside the `options` object.
>
> 4. **Output fields**: `elapsed` renamed to `elapsed_ms`; `format` returns `'beamer'|'revealjs'|'both'` (not `'slides-beamer'|'slides-html'`); `beamer_output`/`revealjs_output` are emitted when `format='both'`.
>
> The implementations in `servers/slides-mcp/src/tools/` are the canonical contracts.

#### Tool: `convert_to_slides`
**Description:** Convert Markdown presentation source to Beamer LaTeX or Reveal.js HTML (or both simultaneously).

**Input Schema:**
```typescript
// servers/slides-mcp/src/index.ts (actual Zod registration)
server.tool('convert_to_slides', 'Convert markdown to Beamer LaTeX or Reveal.js presentation slides', {
  source: z.string().describe('Markdown source content with slides separated by ---'),
  title: z.string().optional().describe('Presentation title (overrides detected title)'),
  author: z.string().optional().describe('Presentation author'),
  format: z.enum(['beamer', 'revealjs', 'both']).describe('Output format: beamer for LaTeX/PDF, revealjs for HTML, or both'),
  options: z.object({
    theme: z.string().optional().describe('Presentation theme'),
    colorTheme: z.string().optional().describe('Beamer color theme'),
    aspectRatio: z.enum(['169', '43', '1610']).optional().describe('Slide aspect ratio (Beamer)'),
    showNotes: z.boolean().optional().describe('Show speaker notes (Beamer)'),
    fontTheme: z.string().optional().describe('Beamer font theme'),
    transition: z.string().optional().describe('Slide transition (Reveal.js)'),
    controls: z.boolean().optional().describe('Show controls (Reveal.js)'),
    progress: z.boolean().optional().describe('Show progress bar (Reveal.js)'),
    katexEnabled: z.boolean().optional().describe('Enable KaTeX math rendering (Reveal.js)'),
  }).optional().describe('Renderer-specific options'),
});
```

**Output Schema (ConvertToSlidesResult):**
```typescript
{
  output: string;              // Primary output (Beamer LaTeX or Reveal.js HTML; for 'both', this is the Beamer output)
  format: 'beamer' | 'revealjs' | 'both';
  slide_count: number;
  has_notes: boolean;
  warnings: string[];
  beamer_output?: string;      // Present when format='both'
  revealjs_output?: string;    // Present when format='both'
  elapsed_ms: number;
}
```

#### Tool: `validate_deck`
**Description:** Validate a markdown slide deck for correct structure and content.

**Input Schema:**
```typescript
server.tool('validate_deck', 'Validate a markdown slide deck for correct structure and content', {
  source: z.string().describe('Markdown source content with slides separated by ---'),
});
```

**Output Schema (ValidateDeckResult):**
```typescript
{
  valid: boolean;
  slide_count: number;
  errors: Array<{ code: string; message: string }>;
  warnings: Array<{ code: string; message: string }>;
  elapsed_ms: number;
}
```

#### Tool: `list_slide_themes`
**Description:** List available slide themes for Beamer and Reveal.js formats.

**Input Schema:**
```typescript
server.tool('list_slide_themes', 'List available slide themes for Beamer and Reveal.js formats', {
  format: z.enum(['beamer', 'revealjs', 'all']).optional().describe('Filter themes by format (default: all)'),
});
```

**Output Schema:**
```typescript
{
  themes: Array<{
    name: string;
    format: 'beamer' | 'revealjs';
    description: string;
  }>;
}
```


### 3.3 Core Integration Points

**packages/core DocumentRequest/DocumentResponse:**
- Input: Markdown + YAML frontmatter (extends standard DocumentRequest).
- Output: DocumentResponse with dual artifacts (PDF + HTML).
- Bibliography system: Delegates to packages/core's unified bibliography handler (CSL-JSON).
- Math validation: Reuses latex-mcp's TeX syntax checker.

**latex-mcp dependency (v0.1):**
- Beamer template system (pandoc Beamer writer + custom .tex templates).
- TeX compilation pipeline (xelatex, minted for code highlighting).
- Equation numbering and cross-references.

**blog-mcp dependency (v0.2):**
- Reveal.js HTML generation patterns (pandoc HTML5 writer + lua filters).
- CSS theme integration (borrowed from blog themes, adapted for slides).
- Asset bundling (self-contained HTML vs. split assets).

---

### 3.4 TypeScript Implementation

**MCP Server Registration:**
```typescript
// zen-sci/packages/sdk/src/servers/slides-mcp/index.ts

import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
  Tool,
  TextContent,
} from '@modelcontextprotocol/sdk/types.js';

const server = new Server(
  { name: 'slides-mcp', version: '0.1.0' },
  { capabilities: { tools: {} } }
);

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: 'convert_to_slides',
      description: 'Convert Markdown presentation to Beamer PDF and/or Reveal.js HTML.',
      inputSchema: {
        type: 'object' as const,
        properties: {
          source: { type: 'string', description: 'Markdown with YAML frontmatter.' },
          format: {
            type: 'string',
            enum: ['beamer', 'reveal', 'both'],
            description: 'Output format(s).',
          },
          theme: {
            type: 'string',
            enum: [
              'metropolis',
              'berlin',
              'warsaw',
              'simple',
              'dark',
              'minimal',
              'sky',
              'league',
            ],
          },
          options: {
            type: 'object',
            properties: {
              speaker_notes: { type: 'boolean' },
              numbered_equations: { type: 'boolean' },
              animations: { type: 'boolean' },
              print_pdf: { type: 'boolean' },
              mathjax_config: { type: 'object' },
            },
          },
        },
        required: ['source', 'format'],
      },
    },
    {
      name: 'validate_slides_source',
      description: 'Validate Markdown presentation syntax.',
      inputSchema: {
        type: 'object' as const,
        properties: {
          source: { type: 'string' },
          strict_mode: { type: 'boolean' },
        },
        required: ['source'],
      },
    },
    {
      name: 'list_slide_themes',
      description: 'List available themes.',
      inputSchema: {
        type: 'object' as const,
        properties: {
          format: {
            type: 'string',
            enum: ['beamer', 'reveal', 'all'],
          },
        },
      },
    },
  ],
}));

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request;

  if (name === 'convert_to_slides') {
    const { source, format, theme, options } = args as any;
    const result = await convertToSlides(source, format, theme, options);
    return { content: [{ type: 'text', text: JSON.stringify(result) }] };
  }

  if (name === 'validate_slides_source') {
    const { source, strict_mode } = args as any;
    const result = await validateSlidesSource(source, strict_mode);
    return { content: [{ type: 'text', text: JSON.stringify(result) }] };
  }

  if (name === 'list_slide_themes') {
    const { format } = args as any;
    const result = await listSlideThemes(format);
    return { content: [{ type: 'text', text: JSON.stringify(result) }] };
  }

  throw new Error(`Unknown tool: ${name}`);
});

async function convertToSlides(
  source: string,
  format: 'beamer' | 'reveal' | 'both',
  theme?: string,
  options?: Record<string, any>
) {
  // Call Python engine
  const result = await callPythonEngine('convert_slides', {
    source,
    format,
    theme,
    options,
  });
  return result;
}

async function validateSlidesSource(source: string, strictMode?: boolean) {
  const result = await callPythonEngine('validate_slides', { source, strictMode });
  return result;
}

async function listSlideThemes(format?: 'beamer' | 'reveal' | 'all') {
  return {
    themes: [
      {
        name: 'metropolis',
        format: 'beamer',
        description: 'Modern, minimal Beamer theme.',
        colors: { primary: '#003366', accent: '#FF6B6B' },
        customizable_options: ['color_primary', 'font_size', 'show_frame_numbers'],
      },
      // ... more themes
    ],
  };
}

async function callPythonEngine(method: string, params: any) {
  // Spawn Python subprocess, send JSON-RPC request
  // Returns parsed JSON response
}

const transport = new StdioServerTransport();
server.connect(transport);
```

**Core Type Definitions:**
```typescript
// zen-sci/packages/core/src/schemas/slides.ts

export interface SlidesDocumentRequest extends DocumentRequest {
  format: 'slides-beamer' | 'slides-html' | 'slides-both';
  frontmatter: {
    title: string;
    author?: string;
    institute?: string;
    date?: string;
    theme?: string;
    bibliography?: string;
    options?: SlidesOptions;
  };
}

export interface SlidesOptions {
  speaker_notes?: boolean;
  numbered_equations?: boolean;
  animations?: boolean;
  print_pdf?: boolean;
  mathjax_config?: Record<string, any>;
}

export interface SlidesResponse extends DocumentResponse {
  format: 'slides-beamer' | 'slides-html' | 'slides-both';
  content: Buffer | { beamer: Buffer; reveal: Buffer };
  artifacts: SlidesArtifact[];
  metadata: SlidesMetadata;
}

export interface SlidesArtifact extends Artifact {
  type: 'speaker_notes' | 'bibliography' | 'theme_preview' | 'outline';
}

export interface SlidesMetadata extends ResponseMetadata {
  slide_count: number;
  theme: string;
  has_speaker_notes: boolean;
  bibliography_count: number;
  math_expressions: number;
}
```

---

### 3.5 Python Processing Engine

**Beamer Path (TeX + xelatex):**
```python
# zen-sci/servers/slides-mcp/processing/beamer.py

import subprocess
import tempfile
import os
from pathlib import Path
import json
from typing import Dict, Any, Tuple

import pandoc
from pandoc.types import Meta, MetaString

async def generate_beamer(
    source: str,
    theme: str,
    options: Dict[str, Any],
    bibliography: str = None,
) -> Tuple[bytes, Dict[str, str]]:
    """
    Convert Markdown to Beamer PDF.

    Args:
        source: Markdown content (without YAML frontmatter, already parsed).
        theme: Beamer theme (metropolis, berlin, warsaw, simple).
        options: Dict with speaker_notes, numbered_equations, print_pdf, etc.
        bibliography: CSL-JSON bibliography content.

    Returns:
        Tuple[PDF bytes, artifacts dict with speaker_notes if enabled].
    """

    # Step 1: Prepare pandoc filters for math numbering, speaker notes
    filters = []
    if options.get('numbered_equations'):
        filters.append('lua-filters/number-equations.lua')
    if options.get('speaker_notes'):
        filters.append('lua-filters/extract-speaker-notes.lua')

    # Step 2: Convert Markdown to Beamer LaTeX
    # Pandoc Beamer template: zen-sci/servers/slides-mcp/templates/beamer-{theme}.tex
    template_path = f'templates/beamer-{theme}.tex'

    pandoc_args = [
        '-f', 'markdown',
        '-t', 'beamer',
        f'--template={template_path}',
        '-V', f'theme:{theme}',
        '-V', 'aspectratio:169',
    ]

    if options.get('speaker_notes'):
        pandoc_args.extend(['-V', 'notes:true'])

    if bibliography:
        pandoc_args.extend([
            '--citeproc',
            f'--bibliography=/dev/stdin',
        ])

    # Apply lua filters for math numbering, speaker note extraction
    for filter_path in filters:
        pandoc_args.extend(['--lua-filter', filter_path])

    latex_content = pandoc.convert_text(
        source,
        format='markdown',
        extra_args=pandoc_args,
        filters=filters,
        standalone=True,
    )

    # Step 3: Extract speaker notes before LaTeX compilation (if enabled)
    speaker_notes = None
    if options.get('speaker_notes'):
        speaker_notes = extract_speaker_notes_from_latex(latex_content)

    # Step 4: Compile LaTeX to PDF using xelatex (for better Unicode/math support)
    with tempfile.TemporaryDirectory() as tmpdir:
        tex_file = Path(tmpdir) / 'slides.tex'
        pdf_file = Path(tmpdir) / 'slides.pdf'

        tex_file.write_text(latex_content, encoding='utf-8')

        # xelatex compile (twice for TOC/refs)
        for _ in range(2):
            result = subprocess.run(
                ['xelatex', '-interaction=nonstopmode', '-output-directory', tmpdir, 'slides.tex'],
                cwd=tmpdir,
                capture_output=True,
                text=True,
            )
            if result.returncode != 0:
                raise RuntimeError(f'xelatex failed: {result.stderr}')

        if not pdf_file.exists():
            raise RuntimeError('PDF generation failed: output file not created')

        pdf_bytes = pdf_file.read_bytes()

    # Step 5: Optionally flatten PDF for printing (remove overlays, transitions)
    if options.get('print_pdf'):
        pdf_bytes = flatten_beamer_pdf(pdf_bytes)

    artifacts = {}
    if speaker_notes:
        artifacts['speaker_notes'] = speaker_notes

    return pdf_bytes, artifacts


def extract_speaker_notes_from_latex(latex_content: str) -> str:
    """
    Extract speaker notes from LaTeX content (marked by \note{...}).
    Returns formatted speaker notes for PDF.
    """
    import re
    notes = []
    pattern = r'\\note\{((?:[^{}]|(?:\{(?:[^{}]|(?:\{[^{}]*\}))*\}))*)\}'
    matches = re.finditer(pattern, latex_content)
    for i, match in enumerate(matches, 1):
        note_content = match.group(1).strip()
        # Remove LaTeX commands, keep text
        note_clean = re.sub(r'\\[a-zA-Z]+\{([^}]*)\}', r'\1', note_content)
        notes.append(f'Slide {i}:\n{note_clean}\n')
    return ''.join(notes)


def flatten_beamer_pdf(pdf_bytes: bytes) -> bytes:
    """
    Flatten Beamer PDF overlays for print-friendly version.
    Uses PyPDF2 or similar to merge overlay pages.
    """
    from PyPDF2 import PdfReader, PdfWriter
    import io

    reader = PdfReader(io.BytesIO(pdf_bytes))
    writer = PdfWriter()

    # Flatten overlays: for each set of overlay pages, keep only the last
    # (full) version. This is a heuristic; more sophisticated merging possible.
    current_slide_prefix = None
    last_page = None

    for page in reader.pages:
        # Detect slide label (this requires metadata; simplified here)
        # In reality, merge overlays of same slide
        last_page = page

    if last_page:
        writer.add_page(last_page)

    output = io.BytesIO()
    writer.write(output)
    return output.getvalue()
```

**Reveal.js Path (HTML + MathJax):**
```python
# zen-sci/servers/slides-mcp/processing/reveal.py

import json
from pathlib import Path
from typing import Dict, Any, Tuple

import pandoc

async def generate_reveal(
    source: str,
    theme: str,
    options: Dict[str, Any],
    bibliography: str = None,
) -> Tuple[str, Dict[str, str]]:
    """
    Convert Markdown to Reveal.js HTML presentation.

    Args:
        source: Markdown content (without YAML frontmatter).
        theme: Reveal theme (dark, minimal, sky, league, etc.).
        options: Dict with animations, mathjax_config, etc.
        bibliography: CSL-JSON bibliography content.

    Returns:
        Tuple[HTML string, artifacts dict with outline, etc.].
    """

    # Step 1: Prepare lua filters for speaker notes, animations
    filters = []
    if options.get('animations'):
        filters.append('lua-filters/reveal-animations.lua')
    filters.append('lua-filters/extract-speaker-notes.lua')

    # Step 2: Convert Markdown to Reveal.js HTML
    pandoc_args = [
        '-f', 'markdown',
        '-t', 'revealjs',
        '--standalone',
        '--slide-level', '2',
        '-V', f'theme:{theme}',
        '-V', 'width:1280',
        '-V', 'height:720',
        '-V', 'transition:fade',
    ]

    # Inject MathJax configuration
    mathjax_config = options.get('mathjax_config', {})
    if not mathjax_config:
        mathjax_config = {
            'tex': {'inlineMath': [['$', '$']], 'displayMath': [['$$', '$$']]},
            'svg': {'fontCache': 'global'},
        }

    pandoc_args.extend([
        '-V', f'mathjax-url:https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js',
    ])

    if bibliography:
        pandoc_args.extend([
            '--citeproc',
            f'--bibliography=/dev/stdin',
        ])

    for filter_path in filters:
        pandoc_args.extend(['--lua-filter', filter_path])

    html_content = pandoc.convert_text(
        source,
        format='markdown',
        extra_args=pandoc_args,
        filters=filters,
        standalone=True,
    )

    # Step 3: Inject custom MathJax config into HTML head
    mathjax_script = f'''
    <script id="MathJax-script" async src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js"></script>
    <script>
      window.MathJax = {json.dumps(mathjax_config)};
    </script>
    '''
    html_content = html_content.replace('</head>', f'{mathjax_script}</head>')

    # Step 4: Inject speaker notes plugin
    speaker_notes_plugin = '''
    <script src="https://cdn.jsdelivr.net/npm/reveal.js-plugins/speaker-notes/speaker-notes.js"></script>
    '''
    html_content = html_content.replace('</head>', f'{speaker_notes_plugin}</head>')

    # Step 5: Extract outline from HTML for artifact
    outline = extract_outline_from_html(html_content)

    artifacts = {'outline': outline}

    return html_content, artifacts


def extract_outline_from_html(html: str) -> str:
    """Extract slide outline (h1, h2 headings) from HTML."""
    import re
    outline = []
    h1_pattern = r'<h1[^>]*>([^<]+)</h1>'
    h2_pattern = r'<h2[^>]*>([^<]+)</h2>'

    current_h1 = None
    for line in html.split('\n'):
        h1_match = re.search(h1_pattern, line)
        h2_match = re.search(h2_pattern, line)

        if h1_match:
            current_h1 = h1_match.group(1)
            outline.append(f'# {current_h1}')
        elif h2_match and current_h1:
            outline.append(f'  ## {h2_match.group(1)}')

    return '\n'.join(outline)
```

---

### 3.6 Key Schemas

```typescript
// zen-sci/packages/core/src/schemas/slides-extended.ts

/**
 * Slide-specific metadata in frontmatter.
 */
export interface SlidesFrontmatter {
  title: string;
  subtitle?: string;
  author: string;
  institute?: string;
  date?: string;
  theme: 'metropolis' | 'berlin' | 'warsaw' | 'simple' | 'dark' | 'minimal' | 'sky' | 'league';
  bibliography?: string; // Path to CSL-JSON or BibTeX file
  keywords?: string[];
  lang?: string; // Default: 'en'
  options?: {
    speaker_notes?: boolean;
    numbered_equations?: boolean;
    animations?: boolean;
    print_pdf?: boolean;
    mathjax_config?: Record<string, any>;
    slide_numbers?: boolean;
    show_notes?: 'show' | 'hide' | 'separate'; // For Beamer
  };
}

/**
 * Markdown structure for slides (content between frontmatter and Markdown body).
 *
 * Example:
 * ---
 * title: My Talk
 * author: Jane Doe
 * theme: metropolis
 * ---
 *
 * # Introduction
 *
 * ## Main Points
 * - Point 1
 * - Point 2
 *
 * ---
 *
 * # Analysis
 *
 * $$\int_0^\infty e^{-x^2} dx = \frac{\sqrt{\pi}}{2}$$
 *
 * ::: notes
 * This integral is fundamental to probability theory.
 * Derived using polar coordinates transformation.
 * :::
 */
export interface SlideStructure {
  slides: Slide[];
  outline: string[];
  totalSlides: number;
}

export interface Slide {
  id: string; // e.g., 'slide-1'
  title?: string;
  level: number; // 1 for # (section), 2 for ## (subsection)
  content: string; // Markdown content for this slide
  speakerNotes?: string; // Content within ::: notes ... :::
  fragmentCount: number; // Number of reveal.js fragments (animated bullet points, etc.)
  mathExpressions: string[]; // List of math expressions (for validation)
  citations: string[]; // Citation keys referenced
}

/**
 * Validation result for slides source.
 */
export interface SlidesValidationResult {
  valid: boolean;
  errors: ValidationError[];
  warnings: ValidationWarning[];
  report: {
    slide_count: number;
    math_expressions: number;
    citations: number;
    speaker_notes_lines: number;
    frontmatter_keys: string[];
    unused_citations: string[];
    orphaned_notes: number;
  };
}

export interface ValidationError {
  line: number;
  column?: number;
  message: string;
  context?: string; // Line snippet
}

export interface ValidationWarning {
  line?: number;
  message: string;
  suggestion?: string;
}
```

---

### 3.7 Phase 2 Infrastructure Requirements

**New packages/core capabilities needed:**

1. **Multi-format artifact system:** DocumentResponse.artifacts should support variable number of outputs (PDF, HTML, speaker notes) without bloating response envelope.
   - *Recommendation:* Extend Artifact type to include format-specific metadata (e.g., `mime_type: 'application/pdf' | 'text/html'`).

2. **Advanced bibliography system:** Unified CSL-JSON handling across Beamer (natbib/biblatex) and Reveal (HTML citation rendering).
   - *Dependency:* Reuse latex-mcp's bibliography integration; extend to HTML citation formatting.

3. **Lua filter registry:** Centralized, versioned Lua filters for pandoc (math numbering, speaker notes, animations).
   - *Recommendation:* Create `packages/core/lua-filters/` directory; document filter interface and metadata.

4. **Theme abstraction layer:** Pluggable theme system allowing institutions to override colors, fonts, logos.
   - *Recommendation:* Define theme schema (JSON or TOML); support theme inheritance and variable substitution.

5. **Performance monitoring:** Benchmark PDF generation and HTML rendering on large decks (50+ slides).
   - *Dependency:* Integrate with packages/core logging/metrics system.

---

### 3.8 Performance & Constraints

| Constraint | Target | Notes |
|-----------|--------|-------|
| PDF generation (30 slides, math-heavy) | <5s | xelatex compilation; caching TeX intermediate files |
| HTML generation (30 slides) | <2s | Pandoc + Lua filters; avoid network calls for CDN resources |
| Speaker notes parsing | <1s | Regex extraction; parallelizable for large documents |
| Theme switching (Reveal.js) | <100ms | CSS preloading; no DOM reflow |
| Output file size (PDF, 30 slides) | <5 MB | xelatex optimization; embed images efficiently |
| Output file size (HTML, 30 slides) | <2 MB | Self-contained; inline CSS/JS; compress images |
| Memory usage | <500 MB | Single slide deck; TeX temporary files cleaned up |
| Citation parsing | <500ms | Precompile CSL processors; cache funder acronyms |

---

## 4. Implementation Plan

### 4.1 Phased Approach

| Phase | Timeline | Deliverables | Dependencies |
|-------|----------|--------------|--------------|
| **Alpha (Week 1-2)** | 2 weeks | Basic Beamer (metropolis theme only); no speaker notes; no animations | packages/core v1.0, latex-mcp v0.1 |
| **Beta (Week 3-4)** | 2 weeks | + Speaker notes (PDF notes page only); + Reveal.js (dark, minimal themes); math numbering | All Alpha + blog-mcp v0.2 |
| **Candidate (Week 5-6)** | 2 weeks | + All themes (4 Beamer, 4 Reveal); animations in Reveal; print-friendly PDF flattening | All Beta |
| **Release (Week 7)** | 1 week | Full test suite; documentation; edge-case fixes | All Candidate |

### 4.2 Week-by-Week Breakdown

**Week 1: Core Beamer Path & MCP Registration**
- [ ] Set up zen-sci/servers/slides-mcp/ directory structure
- [ ] Implement MCP server skeleton (TypeScript, registration with packages/core)
- [ ] Create basic Beamer template (metropolis theme)
- [ ] Implement `convert_to_slides` tool (beamer format only, no options)
- [ ] Write pandoc conversion logic (Markdown → LaTeX)
- [ ] Implement xelatex pipeline with error handling
- [ ] Unit tests: valid Markdown → PDF; invalid LaTeX → error message
- [ ] Integration test: sample 5-slide deck → valid PDF

**Week 2: Beamer Polish & Speaker Notes Foundation**
- [ ] Add speaker notes extraction to Beamer LaTeX (\note{...} blocks)
- [ ] Implement `validate_slides_source` tool
- [ ] Add numbered equations support (lua filter)
- [ ] Create 3 additional Beamer themes (berlin, warsaw, simple)
- [ ] Write theme selector logic
- [ ] Unit tests: numbered equations; citation references; theme switching
- [ ] Integration test: 10-slide deck with equations and citations → PDF

**Week 3: Reveal.js HTML Path & Dual Output**
- [ ] Implement Reveal.js HTML generation (pandoc + lua filters)
- [ ] Write MathJax injection logic
- [ ] Implement speaker view plugin integration
- [ ] Create dual-output mode (format='both')
- [ ] Add animations support (reveal.js fragments)
- [ ] Unit tests: HTML validity; MathJax rendering; speaker notes visibility
- [ ] Integration test: same deck → both PDF and HTML; verify parity

**Week 4: Reveal Themes & Bibliography**
- [ ] Create 4 Reveal.js themes (dark, minimal, sky, league)
- [ ] Integrate CSL-JSON bibliography handling (from packages/core)
- [ ] Implement bibliography artifact generation
- [ ] Add citation cross-linking in HTML
- [ ] Unit tests: theme colors; bibliography formatting; citation links
- [ ] Integration test: deck with 10 citations → PDF with references page; HTML with footnote links

**Week 5: Animations, Outline Extraction & Performance**
- [ ] Implement reveal.js animations (fade, slide, etc.)
- [ ] Implement print-friendly PDF flattening (PyPDF2)
- [ ] Add outline artifact extraction
- [ ] Benchmark 50-slide deck; optimize if needed
- [ ] Unit tests: animation syntax parsing; PDF flattening correctness
- [ ] Load test: 100 concurrent conversions; monitor memory, CPU

**Week 6: Edge Cases & Theme Customization**
- [ ] Handle long titles, equations spanning pages
- [ ] Implement theme customization hooks (colors, fonts)
- [ ] Test accessibility (WCAG 2.1 AA for HTML)
- [ ] Test math-heavy decks (30+ equations)
- [ ] Unit tests: edge cases (orphaned notes, unused citations, etc.)
- [ ] Integration test: 20-slide math seminar deck → both formats

**Week 7: Documentation & Release**
- [ ] Write TypeScript/Markdown API documentation
- [ ] Create example decks (academic talk, conference, internal team)
- [ ] Write Python processing engine docs
- [ ] Finalize error messages
- [ ] Generate theme preview images
- [ ] Publish v0.1.0 to npm (@zen-sci/slides-mcp)
- [ ] [ ] Full regression test suite (100+ unit + 20+ integration tests)

### 4.3 Dependencies & Prerequisites

**Hard Dependencies:**
- packages/core v1.0 (stable): DocumentRequest/DocumentResponse, logging, bibliography system
- latex-mcp v0.1 (released): TeX compilation pipeline, Beamer templates, math validation

**Soft Dependencies (useful but not blocking):**
- blog-mcp v0.2 (released): Reveal.js patterns, HTML theme system, CSS bundling

**Infrastructure Required:**
- Lua filter registry (new in packages/core)
- Theme abstraction layer (new in packages/core)
- Multi-format artifact system extension (new in packages/core)

**External Tools:**
- pandoc ≥2.18 (Markdown → LaTeX/HTML)
- xelatex (TeX → PDF)
- PyPDF2 (PDF manipulation)
- reveal.js (JavaScript library, fetched from CDN or bundled)

---

### 4.4 Testing Strategy

**Unit Tests (TypeScript, jest):**
```typescript
// zen-sci/servers/slides-mcp/__tests__/slides.test.ts

import { convertToSlides, validateSlidesSource } from '../index';

describe('slides-mcp', () => {
  describe('convertToSlides', () => {
    it('should convert basic Markdown to Beamer PDF', async () => {
      const source = '# Title\n\n## Slide 1\nContent';
      const result = await convertToSlides(source, 'beamer');
      expect(result.format).toBe('slides-beamer');
      expect(result.content).toBeInstanceOf(Buffer);
      expect(result.content.length).toBeGreaterThan(1000); // PDF is large
    });

    it('should handle speaker notes in Beamer', async () => {
      const source = `# Title
::: notes
Speaker note content.
:::`;
      const result = await convertToSlides(source, 'beamer', undefined, {
        speaker_notes: true,
      });
      expect(result.artifacts).toContainEqual(
        expect.objectContaining({ type: 'speaker_notes' })
      );
    });

    it('should number equations when requested', async () => {
      const source = '# Title\n\n$$\\int_0^\\infty e^{-x} dx = 1$$';
      const result = await convertToSlides(source, 'beamer', undefined, {
        numbered_equations: true,
      });
      // Verify numbered equation in PDF (PDF content inspection)
      const pdfText = await extractTextFromPdf(result.content);
      expect(pdfText).toMatch(/\(1\)/); // Equation number
    });

    it('should support dual format output', async () => {
      const source = '# Title\n\n## Slide\nContent';
      const result = await convertToSlides(source, 'both');
      expect(result.format).toBe('slides-both');
      expect(result.content).toHaveProperty('beamer');
      expect(result.content).toHaveProperty('reveal');
    });
  });

  describe('validateSlidesSource', () => {
    it('should accept valid Markdown', async () => {
      const source = '# Title\n\n## Slide\nContent';
      const result = await validateSlidesSource(source);
      expect(result.valid).toBe(true);
      expect(result.errors).toHaveLength(0);
    });

    it('should detect unclosed speaker notes', async () => {
      const source = '# Title\n\n::: notes\nIncomplete notes';
      const result = await validateSlidesSource(source, true);
      expect(result.valid).toBe(false);
      expect(result.errors).toContainEqual(
        expect.objectContaining({ message: /unclosed/i })
      );
    });

    it('should warn about unused citations', async () => {
      const source = `---
bibliography: refs.bib
---
# Title

Cite [@unused] but define [@actually_used].`;
      const result = await validateSlidesSource(source);
      expect(result.warnings).toContainEqual(
        expect.objectContaining({ message: /unused citation/i })
      );
    });
  });
});
```

**Integration Tests (Python, pytest):**
```python
# zen-sci/servers/slides-mcp/tests/test_e2e.py

import pytest
import subprocess
import json
from pathlib import Path

class TestSlidesE2E:
    """End-to-end tests for slides-mcp."""

    @pytest.fixture
    def sample_deck(self):
        """Sample 5-slide Markdown deck."""
        return """---
title: Test Presentation
author: Test Author
date: 2026-02-18
theme: metropolis
bibliography: sample_refs.bib
options:
  speaker_notes: true
  numbered_equations: true
---

# Introduction

## Slide 1
This is content on slide 1.

- Bullet point 1
- Bullet point 2

::: notes
Detailed speaker notes for slide 1.
This explains the background.
:::

---

# Analysis

## Slide 2
Mathematical content:

$$\\int_0^\\infty e^{-x^2} dx = \\frac{\\sqrt{\\pi}}{2}$$

Cite [@key2023].

::: notes
This is the Gaussian integral, fundamental to probability.
:::

---

# Conclusion
Thank you!
"""

    def test_beamer_generation(self, sample_deck):
        """Test Beamer PDF generation."""
        result = subprocess.run(
            ['python', '-m', 'slides_mcp', 'convert'],
            input=json.dumps({
                'source': sample_deck,
                'format': 'beamer',
                'theme': 'metropolis',
                'options': {'speaker_notes': True, 'numbered_equations': True},
            }),
            capture_output=True,
            text=True,
        )

        assert result.returncode == 0
        response = json.loads(result.stdout)
        assert response['format'] == 'slides-beamer'
        assert response['metadata']['slide_count'] == 3
        assert response['metadata']['math_expressions'] == 1

        # Decode base64 PDF
        pdf_bytes = base64.b64decode(response['content'])
        assert pdf_bytes[:4] == b'%PDF'  # PDF magic bytes
        assert len(pdf_bytes) > 10000  # Reasonable PDF size

    def test_reveal_generation(self, sample_deck):
        """Test Reveal.js HTML generation."""
        result = subprocess.run(
            ['python', '-m', 'slides_mcp', 'convert'],
            input=json.dumps({
                'source': sample_deck,
                'format': 'reveal',
                'theme': 'dark',
            }),
            capture_output=True,
            text=True,
        )

        assert result.returncode == 0
        response = json.loads(result.stdout)
        assert response['format'] == 'slides-html'

        html_content = response['content']
        assert '<html' in html_content
        assert 'reveal.js' in html_content
        assert 'MathJax' in html_content

    def test_dual_format_parity(self, sample_deck):
        """Ensure Beamer and Reveal outputs have consistent structure."""
        result = subprocess.run(
            ['python', '-m', 'slides_mcp', 'convert'],
            input=json.dumps({
                'source': sample_deck,
                'format': 'both',
            }),
            capture_output=True,
            text=True,
        )

        assert result.returncode == 0
        response = json.loads(result.stdout)
        assert 'beamer' in response['content']
        assert 'reveal' in response['content']
        assert response['metadata']['slide_count'] == 3
```

---

## 5. Risk Assessment & Mitigation

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| xelatex font issues on Docker/CI (missing packages) | Medium | High | Pre-bake TeX environment in Docker image; test on CI early (Week 1) |
| MathJax rendering divergence (Beamer vs Reveal) | Medium | Medium | Create standardized test decks; continuous visual regression testing |
| Pandoc lua filter stability (edge cases, malformed Markdown) | Low | Medium | Comprehensive fuzz testing; fallback to plain conversion if filter fails |
| Theme customization scope creep | Medium | Low | Define MVP themes (5 total); defer advanced customization to v1.1 |
| Performance on 100+ slide decks | Low | Medium | Profile Week 5; consider lazy rendering in Reveal or incremental PDF generation |
| Bibliography cross-format consistency (natbib vs HTML) | Medium | Medium | Unified CSL-JSON pipeline (dependency on packages/core); test with real citation styles |

---

## 6. Rollback & Contingency

**If PDF generation consistently fails (Week 1-2):**
- Fallback to Beamer slide generation without speaker notes; cut feature and revisit in v0.2.

**If Reveal.js HTML output is unsatisfactory (Week 3):**
- Revert to blog-mcp HTML pipeline without speaker view; release as separate tool.

**If xelatex performance is poor (Week 5):**
- Implement async queueing; return partial results; let user poll for completion.

**Rollback procedure:**
- Tag npm release with pre-release suffix (`-rc1`).
- Publish corrected version (`0.1.1-rc2`) to beta channel.
- Communicate breaking changes in release notes.

---

## 7. Appendices

### 7.1 Related Work & Standards

- **Beamer (LaTeX):** Industry standard for academic presentations; well-documented themes.
- **Reveal.js:** Popular for web-based slides; HTML5, accessible, mobile-friendly.
- **Pandoc:** De facto Markdown converter; strong LaTeX and HTML support.
- **WCAG 2.1:** Web Accessibility Guidelines for Reveal.js (color contrast, keyboard navigation).
- **CSL:** Citation Style Language (shared with latex-mcp and other modules).

### 7.2 Future Considerations (v1.1+)

- **Live preview in Claude Code:** Real-time Reveal.js rendering as user edits Markdown.
- **Presenter remote control:** Integrate with Reveal.js API for speaker view on second screen.
- **Animation choreography:** UI for specifying reveal.js animations (drag-and-drop fragments).
- **Handout mode:** Automatic slide reduction (2-per-page, 4-per-page, etc.) for Beamer.
- **Custom video/audio:** Embed video in Reveal.js slides; synchronized playback.
- **Dark mode toggle:** Audience-facing theme switching (JavaScript).

### 7.3 Open Questions

1. **Should speaker notes be speaker-view-only, or visible in presenter mode?** (Design decision for Reveal.js integration.)
2. **How granular should theme customization be?** (Full CSS override vs. variable substitution.)
3. **Should animations block slide transition, or play in background?** (Reveal.js behavior depends on user preference.)
4. **Multi-language support:** Should frontmatter specify LaTeX language packages (e.g., French, Mandarin)?

### 7.4 Schema Backlog

**Potential future additions to SlidesDocumentRequest:**

```typescript
interface SlidesDocumentRequest extends DocumentRequest {
  // Current fields...

  // v1.1 candidates:
  customCSS?: string; // CSS overrides for Reveal.js
  customLaTeXPreamble?: string; // Extra LaTeX packages for Beamer
  animations?: {
    enterTransition: 'fade' | 'slide' | 'zoom';
    exitTransition: 'fade' | 'slide' | 'zoom';
    bulletAnimation: boolean;
  };
  video?: {
    url: string;
    startTime?: number;
    endTime?: number;
    autoplay?: boolean;
  }[];
}
```

---

**End of slides-mcp Specification**

---

## Audit Notes (2026-02-18)

**Auditor:** Claude (Opus-level audit)
**Status:** Reviewed

### Issues
- **CRITICAL-2:** Instantiates `Server` directly instead of extending `ZenSciServer`.
- **CRITICAL-3:** Uses low-level `setRequestHandler` pattern.
- **HIGH-2:** Server path `zen-sci/packages/sdk/src/servers/slides-mcp/index.ts` inconsistent — should be `zen-sci/servers/slides-mcp/`.
- **ADVISORY-1:** May overlap with Phase 3 `presentation-slides-mcp`. This module targets academic (Beamer/Reveal.js); the Phase 3 module targets business (PPTX/Google Slides). Naming should clarify this distinction.
- MCP SDK imports correct (`@modelcontextprotocol/sdk`). Tool naming snake_case — consistent.

#### Decision Propagation (Cruz, 2026-02-18)
- **CRITICAL-2 resolved:** `ZenSciServer` abstract base class deprecated. Use `createZenSciServer()` factory + `McpServer.registerTool()` with Zod schemas at implementation time. Direct `Server` instantiation pattern is closest to the new approach.
- **CRITICAL-3 resolved:** Low-level `setRequestHandler` replaced by `McpServer.registerTool()` with Zod input schemas per MCP SDK v2.
