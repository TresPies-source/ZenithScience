# Blog MCP v0.2: HTML Publication with SEO & Feed Generation

**Author:** ZenSci Architecture (TresPiesDesign.com)
**Status:** Draft
**Created:** 2026-02-18
**Grounded In:** ZenSci semantic clusters analysis, module strategy scout (2026-02-17)

---

## 1. Vision

> Transform markdown research articles into responsive HTML blog posts with SEO optimization, syntax highlighting, and automatic RSS feed generation.

### The Core Insight

Blog MCP validates the "format-agnostic core abstraction" thesis: if LaTeX MCP proves we can convert markdown to PDF with full validation and citation management, Blog MCP proves we can apply the same parsing, validation, and citation pipelines to a completely different output format without rewriting the core.

Blog MCP is the second module because HTML is simpler than LaTeX (no compilation step, no external dependencies for basic rendering) but rich enough to require the same architectural rigor. We validate that `packages/core` abstractions (markdown normalization, citation resolution, cross-reference checking) work for both academic (PDF) and web (HTML) publication. If we get this right, slides-mcp, newsletter-mcp, and grant-mcp will snap into place with minimal new code.

The blog module is also the first user-facing output: researchers publish articles online, we help them turn markdown into polished, SEO-friendly HTML. RSS feeds enable discoverability. Responsive design works on phones. Syntax highlighting makes code readable. This is where ZenSci becomes useful to an actual writer.

### What Makes This Different

Existing static site generators (Hugo, Jekyll, Next.js) couple blog generation with entire site infrastructure. You get themes, plugins, deployment—but you can't just convert a single markdown file to a blog post without scaffolding a full site.

Blog MCP is different because it is **standalone and composable**:

1. **Single-file conversion.** Accept a markdown file + optional frontmatter + optional .bib file. Output a self-contained HTML page (or embed in an existing site). No config files, no theme system—just conversion.

2. **Citation-aware HTML.** Reuse the citation resolution and bibliography rendering from LaTeX MCP. Generate HTML citations with link back to bibliography. Support footnotes, endnotes, and inline references.

3. **Responsive, accessible, and fast.** Output semantic HTML5 with no CSS framework bloat. Use CSS Grid for layout. Optimized for mobile. WCAG 2.1 AA compliant (alt text, semantic headings, sufficient color contrast). No JavaScript required for core reading experience.

4. **RSS feed generation from metadata.** Automatically generate RSS/Atom feeds from frontmatter (title, author, date, summary, category tags). If user is publishing multiple posts, they can generate a feed with one command.

---

## 2. Goals & Success Criteria

### Primary Goals

1. **Accept markdown + frontmatter + BibTeX and produce semantic HTML** suitable for embedding in any static site or publishing standalone.

2. **Apply the same validation pipeline as LaTeX MCP** (math validation, citation resolution, cross-reference checking) to prove core abstractions are format-agnostic.

3. **Render mathematics in HTML** using KaTeX (fast, no server dependency) with fallback to MathML for accessibility.

4. **Generate smart URLs and SEO metadata** (Open Graph, Twitter Card, structured data/JSON-LD) from frontmatter.

5. **Support syntax highlighting for code blocks** using highlight.js with automatic language detection.

6. **Enable RSS/Atom feed generation** from blog post metadata; auto-generate feeds from post directories.

7. **Output responsive, accessible HTML** with no external CSS dependencies (self-contained CSS or minimal external CDN).

### Success Criteria

- ✅ CLI and MCP tool both operational: `zen-sci-blog convert post.md --output post.html` produces valid, standalone HTML.
- ✅ HTML output is valid HTML5 (passes w3c-validator).
- ✅ Math expressions render correctly using KaTeX; fallback to MathML on KaTeX error.
- ✅ BibTeX citations are resolved; unresolved citations trigger warnings, not errors.
- ✅ Citation links in HTML are hyperlinked to bibliography section (anchor links).
- ✅ Code blocks have syntax highlighting; language auto-detected or specified in fence (```python, ```bash, etc.).
- ✅ Table of contents auto-generated from headings; TOC is sticky or collapsible on mobile.
- ✅ Output HTML includes Open Graph and Twitter Card meta tags for social sharing.
- ✅ Output HTML includes JSON-LD structured data (Article schema) for search engines.
- ✅ RSS feed generation: `zen-sci-blog generate-feed posts/ --output feed.xml` produces valid Atom 1.0 feed.
- ✅ HTML renders correctly on mobile, tablet, and desktop (tested at 320px, 768px, 1024px viewports).
- ✅ WCAG 2.1 AA compliance: proper heading hierarchy, alt text for images, sufficient color contrast (4.5:1).
- ✅ Processing latency < 2 seconds for typical blog post (< 50 pages).
- ✅ Output HTML file size < 500KB (including embedded CSS, KaTeX fonts).
- ✅ All warnings and errors logged with line numbers.
- ✅ Comprehensive test suite: unit tests for HTML generation, integration tests for markdown → HTML, accessibility tests (axe-core).

### Non-Goals (Out of Scope for this version)

- ❌ Dynamic site generation. Blog MCP converts single files; static site builders (Hugo, Jekyll) handle site scaffolding.
- ❌ Comment systems or user interaction. This is a converter, not a CMS.
- ❌ Analytics integration. We don't track users; that's the publisher's choice.
- ❌ Search functionality within the blog. Single-page output has no search. Publishers add search on their site.
- ❌ Themes or style customization beyond CSS variables. One responsive design; optional custom CSS.
- ❌ Markdown dialect extensions (e.g., Obsidian wiki links, Slack reactions). We parse CommonMark + math only.
- ❌ Dark mode toggle (JavaScript). Users follow OS preference via `prefers-color-scheme`.
- ❌ Image optimization (WebP, AVIF, srcset). Requires running heavy image processing; out of scope.

---

## 3. Technical Architecture

### 3.1 System Overview

Blog MCP mirrors LaTeX MCP's architecture but swaps the TeX compilation step for HTML rendering:

1. **Markdown Input Layer** (`packages/core`): Same as LaTeX MCP—parse frontmatter, extract metadata, normalize markdown.

2. **Validation & Transformation Layer** (`packages/core` + `servers/blog-mcp/engine`): Reuse math validation, citation resolution, cross-reference checking from core. Transform normalized markdown to HTML AST.

3. **HTML Rendering Layer** (`servers/blog-mcp/engine`): Generate semantic HTML5 from AST, inject KaTeX math, apply syntax highlighting, embed responsive CSS, generate Open Graph / JSON-LD metadata.

The **MCP Server** registers tools that route requests to the Python engine, which returns HTML content + metadata.

```
User input (markdown + .bib)
           ↓
  MCP Tool Request (JSON)
           ↓
  TypeScript Router (servers/blog-mcp/src/index.ts)
           ↓
  Python Engine subprocess (servers/blog-mcp/engine/blog_engine.py)
           ├─ Parse markdown + frontmatter (packages/core)
           ├─ Validate math (SymPy)
           ├─ Resolve citations (citeproc)
           ├─ Transform markdown → HTML AST
           ├─ Apply syntax highlighting (highlight.js)
           ├─ Inject KaTeX math rendering
           └─ Embed responsive CSS + SEO metadata
           ↓
  DocumentResponse (JSON + HTML content)
           ↓
  Output artifact (HTML file or inline content)
```

### 3.2 MCP Tool Definitions

> **Implementation note (2026-02-18):** The tool contracts below reflect the **implemented** design. Key divergences from the original spec:
>
> 1. **`convert_to_html` — `author` is a scalar string, not an array**: The implemented `author?: string` (singular) overrides frontmatter. The original spec had `author: string[]` (array). Single-author override is sufficient for blog posts; multi-author handling lives in frontmatter.
>
> 2. **`convert_to_html` — simplified options**: The original options block included `toc_position`, `math_engine`, `syntax_highlight`, `responsive`, `dark_mode_support`, `custom_css`, `include_schema`, `base_url`, `featured_image`, `summary`, `tags`, `bibliography_style`. The implementation uses a leaner options object: `toc`, `theme`, `selfContained`, `seoSiteUrl`, `seoSiteName`, `twitterHandle`.
>
> 3. **`generate_feed` — posts use `url` not `file_path`**: The original spec required `file_path` (path to `.md` file on disk). The implementation uses `url` (the published URL of the post), which is more appropriate for an MCP tool that generates feed XML from metadata — file I/O is the caller's concern. `author` in a post is `string` (not `string[]`).
>
> 4. **New `validate_post` tool**: Not in the original spec. Added to validate markdown post structure, math, citations, and accessibility before conversion.
>
> The implementations in `servers/blog-mcp/src/tools/` are the canonical contracts.

#### Tool: `convert_to_html`
**Description:** Convert markdown to a responsive HTML blog post with SEO metadata.

**Input Schema:**
```typescript
// servers/blog-mcp/src/index.ts (actual Zod registration)
server.tool('convert_to_html', 'Convert markdown to a responsive HTML blog post with SEO metadata', {
  source: z.string().describe('Markdown source content'),
  title: z.string().optional().describe('Blog post title (overrides frontmatter)'),
  author: z.string().optional().describe('Blog post author (overrides frontmatter)'),
  bibliography: z.string().optional().describe('BibTeX bibliography content'),
  options: z.object({
    toc: z.boolean().optional().describe('Generate table of contents'),
    theme: z.enum(['light', 'dark', 'system']).optional().describe('CSS theme'),
    selfContained: z.boolean().optional().describe('Produce a self-contained HTML file'),
    seoSiteUrl: z.string().optional().describe('Site URL for SEO metadata'),
    seoSiteName: z.string().optional().describe('Site name for SEO metadata'),
    twitterHandle: z.string().optional().describe('Twitter handle for card metadata'),
  }).optional().describe('Conversion options'),
});
```

**Output Schema (ConvertToHtmlResult):**
```typescript
{
  html: string;
  title: string;
  author?: string;
  slug: string;
  description?: string;
  wordCount: number;
  readingTimeMinutes: number;
  warnings: string[];
  elapsed_ms: number;
}
```

#### Tool: `generate_feed`
**Description:** Generate an Atom 1.0 RSS feed from a list of blog post metadata.

**Input Schema:**
```typescript
const PostMetadataSchema = z.object({
  title: z.string().describe('Post title'),
  url: z.string().describe('Post URL'),                 // NOTE: 'url' not 'file_path'
  date: z.string().describe('Publication date (ISO 8601)'),
  author: z.string().optional().describe('Post author'),  // scalar, not array
  summary: z.string().optional().describe('Post summary'),
  content: z.string().optional().describe('Post HTML content'),
  tags: z.array(z.string()).optional().describe('Post tags'),
});

server.tool('generate_feed', 'Generate an Atom 1.0 RSS feed from a list of blog post metadata', {
  posts: z.array(PostMetadataSchema).describe('List of post metadata'),
  feedTitle: z.string().describe('Feed title'),
  feedUrl: z.string().describe('Feed URL (self link)'),
  siteUrl: z.string().describe('Site URL'),
  description: z.string().optional().describe('Feed description'),
});
```

**Output Schema (GenerateFeedResult):**
```typescript
{
  feed: string;        // Atom XML content
  postCount: number;
  elapsed_ms: number;
}
```

#### Tool: `validate_post`
**Description:** Validate a markdown blog post for structure, accessibility, math, and citations.

> **Implementation note:** `validate_post` was not in the original spec. It was added during implementation as the standard validation tool pattern used across all MCP servers.

**Input Schema:**
```typescript
server.tool('validate_post', 'Validate a markdown blog post for structure, accessibility, math, and citations', {
  source: z.string().describe('Markdown source content'),
  bibliography: z.string().optional().describe('BibTeX bibliography content'),
});
```

**Output Schema (ValidatePostResult):**
```typescript
{
  valid: boolean;
  errors: Array<{ code: string; message: string }>;
  warnings: Array<{ code: string; message: string }>;
  wordCount: number;
  headingCount: number;
  hasTitle: boolean;
  mathCount: number;
  citationCount: number;
  elapsed_ms: number;
}
```


### 3.3 Core Integration Points

Blog MCP reuses `packages/core` abstractions identically to LaTeX MCP:

1. **`packages/core/src/parsing/`** — Same functions as LaTeX MCP:
   - `parseFrontmatter(content: string): { metadata, body }`
   - `normalizeMarkdown(content: string): NormalizedMD`
   - `extractReferences(content: string): ReferenceMap`

2. **`packages/core/src/validation/`** — Same validators:
   - `validateMathExpression(latex: string): ValidationResult` (but render via KaTeX instead of TeX)
   - `resolveCitations(normalizedContent, bibliography): CitationMap`
   - `checkCrossReferences(normalizedContent): ReferenceReport`

3. **`packages/sdk/src/mcp-server.ts`** — Same base class `MCPServer`

The **only difference** is the output layer: Blog MCP generates HTML instead of calling LaTeX compilers. This proves the abstraction is truly format-agnostic.

### 3.4 TypeScript Implementation

```typescript
// servers/blog-mcp/src/index.ts
import { MCPServer } from '@zen-sci/sdk';
import { DocumentRequest, DocumentResponse, ConversionWarning } from '@zen-sci/core';
import * as fs from 'fs';
import * as path from 'path';

interface BlogDocumentRequest extends DocumentRequest {
  format: 'html';
  slug?: string;
  date?: string;
  summary?: string;
  tags?: string[];
  bibliography?: string;
  bibliography_style: 'ieee' | 'acm' | 'chicago' | 'apa' | 'nature';
  featured_image?: string;
  options: DocumentOptions & {
    toc: boolean;
    toc_position: 'sticky' | 'inline' | 'none';
    math_engine: 'katex' | 'mathjax';
    syntax_highlight: boolean;
    responsive: boolean;
    dark_mode_support: boolean;
    custom_css?: string;
    include_schema: boolean;
  };
  base_url?: string;
}

interface BlogDocumentResponse extends DocumentResponse {
  format: 'html';
  html: string;
  excerpt: string;
  metadata: {
    title: string;
    author: string;
    slug: string;
    date: string;
    summary: string;
    tags: string[];
    word_count: number;
    read_time_minutes: number;
    featured_image?: string;
  };
  seo: {
    og_title: string;
    og_description: string;
    og_image?: string;
    twitter_card: string;
    canonical_url?: string;
  };
  citationStats: {
    total: number;
    resolved: number;
    unresolved: string[];
  };
}

export class BlogMCPServer extends MCPServer {
  constructor() {
    super('blog-mcp', {
      version: '0.2.0',
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
          name: 'convert_to_html',
          description: 'Convert markdown blog post to semantic HTML...',
          inputSchema: { /* from 3.2 */ },
          handler: this.handleConvertToHTML.bind(this)
        },
        {
          name: 'generate_feed',
          description: 'Generate RSS/Atom feed from multiple posts...',
          inputSchema: { /* from 3.2 */ },
          handler: this.handleGenerateFeed.bind(this)
        }
      ]
    }));
  }

  private async handleConvertToHTML(
    input: Record<string, any>
  ): Promise<BlogDocumentResponse> {
    const startTime = Date.now();
    const warnings: ConversionWarning[] = [];

    try {
      const request: BlogDocumentRequest = {
        id: input.id || `html-${Date.now()}`,
        source: input.source,
        format: 'html',
        frontmatter: {
          title: input.title,
          author: input.author || ['Anonymous'],
          slug: input.slug,
          date: input.date,
          summary: input.summary,
          tags: input.tags || [],
          featured_image: input.featured_image
        },
        bibliography: input.bibliography,
        bibliography_style: input.bibliography_style || 'ieee',
        options: {
          toc: input.options?.toc !== false,
          toc_position: input.options?.toc_position || 'sticky',
          math_engine: input.options?.math_engine || 'katex',
          syntax_highlight: input.options?.syntax_highlight !== false,
          responsive: input.options?.responsive !== false,
          dark_mode_support: input.options?.dark_mode_support !== false,
          custom_css: input.options?.custom_css,
          include_schema: input.options?.include_schema !== false
        },
        base_url: input.base_url
      };

      // Call Python engine
      const pythonResult = await this.callPythonEngine(
        'servers/blog-mcp/engine/blog_engine.py',
        {
          action: 'convert_to_html',
          request
        },
        { timeout: 60000 } // 1 minute
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

      const {
        html,
        excerpt,
        metadata,
        seo,
        citation_stats,
        warnings: py_warnings
      } = pythonResult;

      warnings.push(...(py_warnings || []));

      const response: BlogDocumentResponse = {
        id: request.id,
        requestId: request.id,
        format: 'html',
        content: html,
        html,
        excerpt,
        metadata,
        seo,
        citationStats: citation_stats,
        artifacts: [
          {
            type: 'html',
            path: `${metadata.slug}.html`,
            mimeType: 'text/html',
            size: html.length
          }
        ],
        warnings,
        metadata: metadata as any,
        elapsed: Date.now() - startTime
      };

      return response;
    } catch (error) {
      throw new Error(
        `HTML conversion failed: ${error instanceof Error ? error.message : String(error)}`
      );
    }
  }

  private async handleGenerateFeed(
    input: Record<string, any>
  ): Promise<Record<string, any>> {
    const pythonResult = await this.callPythonEngine(
      'servers/blog-mcp/engine/blog_engine.py',
      {
        action: 'generate_feed',
        feed_title: input.feed_title,
        feed_description: input.feed_description,
        base_url: input.base_url,
        posts: input.posts,
        format: input.format || 'atom',
        limit: input.limit || 20
      },
      { timeout: 30000 }
    );

    return pythonResult;
  }
}

// Start server
const server = new BlogMCPServer();
server.initialize().then(() => {
  console.log('Blog MCP server listening...');
});
```

### 3.5 Python Processing Engine

```python
# servers/blog-mcp/engine/blog_engine.py
import json
import sys
from pathlib import Path
from typing import Dict, Any, List, Optional, Tuple
from datetime import datetime
import logging
import html
import re
from urllib.parse import quote

# Import from packages/core
sys.path.insert(0, str(Path(__file__).parent.parent.parent.parent / 'core' / 'src'))
from parsing.markdown_parser import parse_frontmatter, normalize_markdown
from validation.math_validator import validate_latex_expression
from validation.citation_resolver import resolve_citations, parse_bibtex

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class BlogEngine:
    def __init__(self):
        pass

    def convert_to_html(self, request: Dict[str, Any]) -> Dict[str, Any]:
        """Convert markdown blog post to HTML."""
        try:
            source = request['source']
            title = request['frontmatter'].get('title', 'Untitled')
            author = request['frontmatter'].get('author', ['Anonymous'])
            slug = request['frontmatter'].get('slug') or self._slugify(title)
            date = request['frontmatter'].get('date') or datetime.now().strftime('%Y-%m-%d')
            summary = request['frontmatter'].get('summary', '')
            tags = request['frontmatter'].get('tags', [])
            featured_image = request['frontmatter'].get('featured_image')
            bibliography = request.get('bibliography')
            bib_style = request.get('bibliography_style', 'ieee')
            options = request.get('options', {})
            base_url = request.get('base_url', 'https://example.com')

            # Parse and normalize markdown
            normalized = normalize_markdown(source)

            # Resolve citations
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

            # Generate HTML
            body_html = self._markdown_to_html(
                normalized=normalized,
                citations=citation_map,
                options=options
            )

            # Extract first paragraph as excerpt
            excerpt = self._extract_excerpt(body_html)

            # Build table of contents
            toc_html = ''
            if options.get('toc'):
                toc_html = self._generate_toc(normalized.get('headings', []), options)

            # Generate complete HTML document
            html_doc = self._build_html_document(
                title=title,
                author=author,
                date=date,
                slug=slug,
                summary=summary,
                tags=tags,
                featured_image=featured_image,
                body_html=body_html,
                toc_html=toc_html,
                base_url=base_url,
                options=options
            )

            # Build SEO metadata
            seo = self._build_seo_metadata(
                title=title,
                summary=summary,
                featured_image=featured_image,
                slug=slug,
                base_url=base_url
            )

            # Calculate word count and read time
            word_count = len(normalized.get('body', '').split())
            read_time = max(1, round(word_count / 200))  # 200 words per minute

            metadata = {
                'title': title,
                'author': ' and '.join(author) if author else 'Anonymous',
                'slug': slug,
                'date': date,
                'summary': summary,
                'tags': tags,
                'word_count': word_count,
                'read_time_minutes': read_time,
                'featured_image': featured_image
            }

            return {
                'html': html_doc,
                'excerpt': excerpt,
                'metadata': metadata,
                'seo': seo,
                'citation_stats': {
                    'total': len(normalized.get('citations_used', [])),
                    'resolved': len(citation_map),
                    'unresolved': unresolved_citations
                },
                'warnings': [],
                'success': True
            }

        except Exception as e:
            logger.error(f'HTML conversion failed: {e}')
            return {
                'error': str(e),
                'success': False
            }

    def _markdown_to_html(
        self,
        normalized: Dict[str, Any],
        citations: Dict[str, Any],
        options: Dict[str, Any]
    ) -> str:
        """Convert normalized markdown to HTML."""
        body = normalized.get('body', '')

        # Simple markdown → HTML conversion (uses CommonMark-like rules)
        # For production, use a real markdown library like markdown-it

        # Handle code blocks with syntax highlighting
        if options.get('syntax_highlight'):
            body = self._highlight_code_blocks(body)

        # Handle math expressions (KaTeX or MathJax)
        if options.get('math_engine') == 'katex':
            body = self._render_math_katex(body)
        else:
            body = self._render_math_mathjax(body)

        # Handle citations
        if citations:
            body = self._link_citations(body, citations)

        # Wrap citations and bibliography
        if citations:
            bib_html = self._generate_bibliography(citations)
            body += f'\n<section id="bibliography">\n<h2>References</h2>\n{bib_html}\n</section>\n'

        return body

    def _highlight_code_blocks(self, html: str) -> str:
        """Add syntax highlighting to code blocks."""
        # Regex to find <pre><code>...</code></pre> blocks
        pattern = r'<pre><code(?:\s+class="language-(\w+)")?>(.*?)</code></pre>'

        def replace_block(match):
            lang = match.group(1) or 'text'
            code = html.unescape(match.group(2))
            # In production, call highlight.js via subprocess or use pygments
            # For now, return as-is with language class
            escaped = html.escape(code)
            return f'<pre><code class="hljs language-{lang}">{escaped}</code></pre>'

        return re.sub(pattern, replace_block, html, flags=re.DOTALL)

    def _render_math_katex(self, html: str) -> str:
        """Wrap math expressions for KaTeX rendering."""
        # Display math: $$...$$
        html = re.sub(
            r'\$\$(.*?)\$\$',
            r'<div class="math katex-display" data-latex="\1"></div>',
            html,
            flags=re.DOTALL
        )

        # Inline math: $...$
        html = re.sub(
            r'\$([^\$]+)\$',
            r'<span class="math katex-inline" data-latex="\1"></span>',
            html
        )

        return html

    def _render_math_mathjax(self, html: str) -> str:
        """Wrap math expressions for MathJax rendering."""
        html = re.sub(
            r'\$\$(.*?)\$\$',
            r'\\[\1\\]',
            html,
            flags=re.DOTALL
        )
        html = re.sub(r'\$([^\$]+)\$', r'\(\1\)', html)
        return html

    def _link_citations(self, html: str, citations: Dict[str, Any]) -> str:
        """Convert \cite{key} to hyperlinked citations."""
        pattern = r'\\cite\{([^}]+)\}'

        def replace_cite(match):
            key = match.group(1)
            if key in citations:
                return f'<a href="#cite-{key}" class="cite-ref">[{list(citations.keys()).index(key) + 1}]</a>'
            else:
                return f'<span class="cite-unresolved">[?]</span>'

        return re.sub(pattern, replace_cite, html)

    def _generate_bibliography(self, citations: Dict[str, Any]) -> str:
        """Generate HTML bibliography from resolved citations."""
        bib_html = '<ol class="bibliography">\n'
        for i, (key, entry) in enumerate(citations.items(), 1):
            authors = entry.get('authors', [])
            title = entry.get('title', 'Untitled')
            year = entry.get('year', 'n.d.')
            bib_html += f'<li id="cite-{key}"><strong>{", ".join(authors)}</strong> ({year}). "{title}".</li>\n'
        bib_html += '</ol>\n'
        return bib_html

    def _generate_toc(self, headings: List[Dict[str, Any]], options: Dict[str, Any]) -> str:
        """Generate table of contents HTML."""
        toc_html = '<nav class="toc">\n<ol>\n'
        for heading in headings:
            level = heading.get('level', 1)
            text = heading.get('text', '')
            slug = self._slugify(text)
            indent = '  ' * (level - 1)
            toc_html += f'{indent}<li><a href="#{slug}">{text}</a></li>\n'
        toc_html += '</ol>\n</nav>\n'
        return toc_html

    def _extract_excerpt(self, html: str) -> str:
        """Extract first paragraph as excerpt."""
        match = re.search(r'<p>(.*?)</p>', html)
        if match:
            text = match.group(1)
            text = re.sub(r'<[^>]+>', '', text)  # Strip HTML tags
            return text[:160] + ('...' if len(text) > 160 else '')
        return ''

    def _build_html_document(
        self,
        title: str,
        author: List[str],
        date: str,
        slug: str,
        summary: str,
        tags: List[str],
        featured_image: Optional[str],
        body_html: str,
        toc_html: str,
        base_url: str,
        options: Dict[str, Any]
    ) -> str:
        """Build complete HTML document with styling."""

        responsive_css = self._generate_responsive_css(options)
        custom_css = options.get('custom_css', '')

        head = f'''<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{html.escape(title)}</title>
  <meta name="description" content="{html.escape(summary or title)}">
  <meta name="author" content="{html.escape(', '.join(author))}">
  <link rel="canonical" href="{html.escape(base_url)}/posts/{slug}/">

  <!-- Open Graph -->
  <meta property="og:title" content="{html.escape(title)}">
  <meta property="og:description" content="{html.escape(summary or title)}">
  <meta property="og:type" content="article">
  <meta property="og:url" content="{html.escape(base_url)}/posts/{slug}/">
  {f'<meta property="og:image" content="{html.escape(featured_image)}">' if featured_image else ''}

  <!-- Twitter Card -->
  <meta name="twitter:card" content="summary_large_image">
  <meta name="twitter:title" content="{html.escape(title)}">
  <meta name="twitter:description" content="{html.escape(summary or title)}">

  <!-- JSON-LD Schema -->
  {self._generate_schema_json(title, author, date, slug, base_url) if options.get('include_schema') else ''}

  <!-- KaTeX -->
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/katex@0.16.0/dist/katex.min.css">
  <script defer src="https://cdn.jsdelivr.net/npm/katex@0.16.0/dist/katex.min.js"></script>
  <script defer src="https://cdn.jsdelivr.net/npm/katex@0.16.0/dist/contrib/auto-render.min.js"
    onload="renderMathInElement(document.body);"></script>

  <!-- Highlight.js -->
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/highlight.js@11.8.0/styles/atom-one-light.min.css">
  <script src="https://cdn.jsdelivr.net/npm/highlight.js@11.8.0/highlight.min.js"></script>
  <script>hljs.highlightAll();</script>

  <style>
{responsive_css}
{custom_css}
  </style>
</head>
<body>
  <article>
    <header>
      <h1>{html.escape(title)}</h1>
      <p class="meta">
        By {html.escape(', '.join(author))} on <time datetime="{date}">{date}</time>
      </p>
      {f'<p class="tags">{", ".join(f\'<span class="tag">{html.escape(tag)}</span>\' for tag in tags)}</p>' if tags else ''}
    </header>

    {f'<aside class="toc-container">{toc_html}</aside>' if toc_html else ''}

    <main>
{body_html}
    </main>
  </article>
</body>
</html>'''
        return head

    def _generate_responsive_css(self, options: Dict[str, Any]) -> str:
        """Generate responsive CSS."""
        css = '''
/* Base typography */
body {
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
  line-height: 1.6;
  max-width: 800px;
  margin: 0 auto;
  padding: 1rem;
  color: #333;
  background: #fff;
}

h1, h2, h3, h4, h5, h6 {
  margin-top: 1.5em;
  margin-bottom: 0.5em;
  line-height: 1.2;
}

h1 { font-size: 2.5rem; }
h2 { font-size: 2rem; }
h3 { font-size: 1.5rem; }

code {
  font-family: "Monaco", "Courier New", monospace;
  background: #f4f4f4;
  padding: 0.2em 0.4em;
  border-radius: 3px;
}

pre {
  background: #f4f4f4;
  padding: 1rem;
  border-radius: 5px;
  overflow-x: auto;
}

pre code {
  background: none;
  padding: 0;
}

/* TOC styling */
.toc {
  background: #f9f9f9;
  border-left: 4px solid #007bff;
  padding: 1rem;
  margin: 2rem 0;
  border-radius: 4px;
}

.toc ol {
  list-style: decimal;
  margin: 0;
  padding-left: 1.5rem;
}

/* Math */
.math {
  display: inline;
}

.math.katex-display {
  display: block;
  margin: 1em 0;
}

/* Responsive */
@media (max-width: 768px) {
  body { padding: 0.5rem; }
  h1 { font-size: 1.8rem; }
  h2 { font-size: 1.4rem; }
  .toc { margin: 1rem 0; }
}

/* Dark mode */
@media (prefers-color-scheme: dark) {
  body { background: #1e1e1e; color: #e0e0e0; }
  code { background: #333; }
  pre { background: #2d2d2d; }
  .toc { background: #2d2d2d; border-left-color: #0088ff; }
}
'''
        return css

    def _generate_schema_json(
        self,
        title: str,
        author: List[str],
        date: str,
        slug: str,
        base_url: str
    ) -> str:
        """Generate JSON-LD article schema."""
        schema = {
            "@context": "https://schema.org",
            "@type": "BlogPosting",
            "headline": title,
            "author": [{"@type": "Person", "name": a} for a in author],
            "datePublished": date,
            "url": f"{base_url}/posts/{slug}/"
        }
        return f'<script type="application/ld+json">{json.dumps(schema)}</script>'

    def _build_seo_metadata(
        self,
        title: str,
        summary: str,
        featured_image: Optional[str],
        slug: str,
        base_url: str
    ) -> Dict[str, str]:
        """Build SEO metadata."""
        return {
            'og_title': title,
            'og_description': summary or title,
            'og_image': featured_image or '',
            'twitter_card': 'summary_large_image',
            'canonical_url': f'{base_url}/posts/{slug}/'
        }

    def _slugify(self, text: str) -> str:
        """Convert text to URL slug."""
        slug = text.lower()
        slug = re.sub(r'[^a-z0-9\s-]', '', slug)
        slug = re.sub(r'[\s-]+', '-', slug)
        slug = slug.strip('-')
        return slug

    def generate_feed(
        self,
        feed_title: str,
        feed_description: str,
        base_url: str,
        posts: List[Dict[str, Any]],
        feed_format: str = 'atom',
        limit: int = 20
    ) -> Dict[str, Any]:
        """Generate RSS/Atom feed from post metadata."""
        try:
            posts = posts[:limit]

            if feed_format == 'atom':
                feed_xml = self._generate_atom_feed(
                    feed_title,
                    feed_description,
                    base_url,
                    posts
                )
            else:
                feed_xml = self._generate_rss_feed(
                    feed_title,
                    feed_description,
                    base_url,
                    posts
                )

            return {
                'feed': feed_xml,
                'format': feed_format,
                'posts_included': len(posts),
                'elapsed_ms': 0,
                'success': True
            }

        except Exception as e:
            logger.error(f'Feed generation failed: {e}')
            return {
                'error': str(e),
                'success': False
            }

    def _generate_atom_feed(
        self,
        title: str,
        description: str,
        base_url: str,
        posts: List[Dict[str, Any]]
    ) -> str:
        """Generate Atom 1.0 feed."""
        entries = []
        for post in posts:
            date = post.get('date', datetime.now().isoformat())
            entry = f'''  <entry>
    <title>{html.escape(post['title'])}</title>
    <link href="{html.escape(base_url)}/posts/{post['slug']}/"/>
    <id>{html.escape(base_url)}/posts/{post['slug']}/</id>
    <published>{date}T00:00:00Z</published>
    <updated>{date}T00:00:00Z</updated>
    {f'<summary>{html.escape(post.get("summary", ""))}</summary>' if post.get('summary') else ''}
    <author>
      <name>{html.escape(post['author'][0] if isinstance(post['author'], list) else post['author'])}</name>
    </author>
  </entry>'''
            entries.append(entry)

        feed = f'''<?xml version="1.0" encoding="UTF-8"?>
<feed xmlns="http://www.w3.org/2005/Atom">
  <title>{html.escape(title)}</title>
  <subtitle>{html.escape(description)}</subtitle>
  <link href="{html.escape(base_url)}/"/>
  <link href="{html.escape(base_url)}/feed.atom" rel="self"/>
  <id>{html.escape(base_url)}/</id>
  <updated>{datetime.now().isoformat()}Z</updated>
{chr(10).join(entries)}
</feed>'''
        return feed

    def _generate_rss_feed(
        self,
        title: str,
        description: str,
        base_url: str,
        posts: List[Dict[str, Any]]
    ) -> str:
        """Generate RSS 2.0 feed."""
        items = []
        for post in posts:
            item = f'''    <item>
      <title>{html.escape(post['title'])}</title>
      <link>{html.escape(base_url)}/posts/{post['slug']}/</link>
      <guid>{html.escape(base_url)}/posts/{post['slug']}/</guid>
      <pubDate>{post.get('date', datetime.now().isoformat())}</pubDate>
      {f'<description>{html.escape(post.get("summary", ""))}</description>' if post.get('summary') else ''}
      <author>{html.escape(post['author'][0] if isinstance(post['author'], list) else post['author'])}</author>
    </item>'''
            items.append(item)

        feed = f'''<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0">
  <channel>
    <title>{html.escape(title)}</title>
    <link>{html.escape(base_url)}</link>
    <description>{html.escape(description)}</description>
    <language>en-us</language>
{chr(10).join(items)}
  </channel>
</rss>'''
        return feed

def main():
    """Entry point for subprocess invocation."""
    if len(sys.argv) < 2:
        print(json.dumps({'error': 'No action specified'}))
        sys.exit(1)

    action = sys.argv[1]
    input_data = json.loads(sys.stdin.read())

    engine = BlogEngine()

    try:
        if action == 'convert_to_html':
            result = engine.convert_to_html(input_data['request'])
        elif action == 'generate_feed':
            result = engine.generate_feed(
                input_data['feed_title'],
                input_data.get('feed_description', ''),
                input_data['base_url'],
                input_data['posts'],
                input_data.get('format', 'atom'),
                input_data.get('limit', 20)
            )
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
// packages/core/src/types/blog.ts (reuses most types from core)
export interface BlogOptions extends DocumentOptions {
  toc: boolean;
  toc_position: 'sticky' | 'inline' | 'none';
  math_engine: 'katex' | 'mathjax';
  syntax_highlight: boolean;
  responsive: boolean;
  dark_mode_support: boolean;
  custom_css?: string;
  include_schema: boolean;
}

export interface BlogMetadata {
  title: string;
  author: string;
  slug: string;
  date: string;
  summary: string;
  tags: string[];
  word_count: number;
  read_time_minutes: number;
  featured_image?: string;
}

/**
 * SEOMetadata: Extends WebMetadataSchema from @zen-sci/core (packages-core-spec Section 3.4).
 * Provides blog-specific defaults on top of the shared OG/Twitter/Schema.org structure.
 */
export interface SEOMetadata extends WebMetadataSchema {
  /** Canonical URL for this blog post (overrides WebMetadataSchema.canonicalUrl) */
  canonicalUrl?: string;
}

// Usage note: The flat og_title/og_description/twitter_card fields from the original
// design are now nested under the og/twitter sub-objects inherited from WebMetadataSchema.
// Example:
//   seo.og?.title         (was: seo.og_title)
//   seo.og?.description   (was: seo.og_description)
//   seo.og?.image         (was: seo.og_image)
//   seo.twitter?.card     (was: seo.twitter_card)

export interface FeedPost {
  file_path: string;
  slug: string;
  date: string;
  title: string;
  summary: string;
  author: string[];
  tags: string[];
}
```

### 3.7 Performance & Constraints

- **Processing latency:** < 2 seconds for typical blog post (< 10,000 words). No compilation step = faster than LaTeX MCP.
- **File size limits:** Input markdown up to 100MB. Output HTML < 500KB (including embedded CSS, KaTeX fonts).
- **Concurrency:** Can handle multiple concurrent conversions (HTML rendering is stateless).
- **Memory:** Peak memory ~100MB during feed generation from 1,000+ posts.
- **Dependencies:** Python >= 3.11 with markdown, citeproc-python, PyYAML. JavaScript-free output (KaTeX, highlight.js loaded from CDN).

---

## 4. Implementation Plan

### 4.1 Phased Approach

| Phase | Duration | Focus | Deliverables |
|-------|----------|-------|--------------|
| 1 (Setup) | Week 1 | Reuse core from latex-mcp, set up blog-mcp structure | TypeScript scaffolding, MCP tool skeleton |
| 2 (HTML Generation) | Week 1-2 | Markdown → HTML conversion, CSS styling, responsive design | Working `convert_to_html` tool (basic posts) |
| 3 (Validation & SEO) | Week 2 | Reuse math/citation validators, add SEO metadata | `convert_to_html` with full validation and meta tags |
| 4 (Feed & Polish) | Week 3 | Implement `generate_feed`, comprehensive testing | Production release with RSS/Atom support |

### 4.2 Week-by-Week Breakdown

**Week 1: Project Structure & HTML Foundations**

- [ ] Initialize `servers/blog-mcp/` directory structure
- [ ] Set up TypeScript/Python scaffolding
- [ ] Confirm `packages/core` parsing and validation functions are callable
- [ ] Implement `blog_engine.py` stubs for `_markdown_to_html` and `_generate_responsive_css`
- [ ] Implement basic HTML document builder (head, body, article structure)
- [ ] Write unit tests for CSS generation

**Week 2: Markdown → HTML Pipeline**

- [ ] Implement `_markdown_to_html` with heading, paragraph, list, table parsing
- [ ] Integrate KaTeX math rendering (wrap expressions in `<span class="katex">`)
- [ ] Implement `_highlight_code_blocks` with highlight.js or Pygments
- [ ] Implement `_link_citations` to convert `\cite{key}` to HTML anchors
- [ ] Test end-to-end: markdown → HTML (no citations, basic math)
- [ ] Implement TypeScript MCP tool handler for `convert_to_html`
- [ ] Generate Open Graph and Twitter Card meta tags

**Week 3: SEO, Validation & Feed Generation**

- [ ] Reuse `validateMathExpression`, `resolveCitations`, `checkCrossReferences` from packages/core
- [ ] Implement bibliography HTML generation
- [ ] Add JSON-LD structured data (Article schema)
- [ ] Implement `generate_feed` for Atom 1.0 and RSS 2.0
- [ ] Test feed generation with multiple posts
- [ ] Extract excerpt from body HTML

**Week 4: Testing & Release**

- [ ] Write comprehensive test suite: unit tests, integration tests, accessibility tests (axe-core)
- [ ] Test responsive design at multiple viewport sizes (320px, 768px, 1024px)
- [ ] Test with real blog posts (various lengths, math complexity, citation density)
- [ ] Implement CLI tool: `zen-sci-blog convert post.md --output post.html`
- [ ] Implement CLI tool: `zen-sci-blog generate-feed posts/ --output feed.xml`
- [ ] Document user guide and API reference
- [ ] Set up CI/CD
- [ ] Release v0.2 to npm

### 4.3 Dependencies & Prerequisites

**System dependencies:**
- Node.js >= 18, TypeScript >= 5.0
- Python >= 3.11

**npm dependencies:**
```json
{
  "@modelcontextprotocol/sdk": "^2.0.0",
  "@zen-sci/core": "workspace:*",
  "@zen-sci/sdk": "workspace:*"
}
```

**Python dependencies:**
```txt
markdown>=3.4.1
citeproc-python>=0.7.0
bibtexparser>=2.0.0
pyyaml>=6.0
```

**Internal dependencies:**
- `packages/core/src/parsing/` — markdown normalization (reuse from latex-mcp)
- `packages/core/src/validation/` — math, citation, cross-reference validators (reuse from latex-mcp)
- `packages/sdk/src/mcp-server.ts` — MCP base class (reuse from latex-mcp)

### 4.4 Testing Strategy

**Unit tests:**
- Slugify: "My Blog Post" → "my-blog-post"
- Excerpt: Extract first paragraph (< 160 chars)
- SEO metadata: Correct Open Graph tags
- CSS generation: Responsive media queries present
- HTML escaping: Special chars don't break output
- Citation linking: `\cite{key}` → `<a href="#cite-key">`
- TOC generation: Correct heading nesting
- Feed generation: Valid XML, correct number of posts

**Integration tests:**
- Markdown + citations → HTML with bibliography
- Markdown + math → HTML with KaTeX rendering
- Multiple posts → valid Atom feed
- Dark mode CSS: correct `@media` queries

**Accessibility tests:**
- Run axe-core on generated HTML
- Check heading hierarchy (h1 → h2 → h3, no gaps)
- Alt text for images (if present)
- Color contrast (4.5:1 for normal text)

**Performance tests:**
- Small post (< 2KB): latency < 500ms
- Medium post (50KB): latency < 1.5s
- Large post (100KB): latency < 2s
- Feed generation (100 posts): latency < 5s

---

## 5. Risk Assessment & Mitigation

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| CSS responsiveness fails on some browsers | Medium | Medium | Test on Chrome, Firefox, Safari, Edge; use CSS Grid (well-supported). Provide fallback CSS for older browsers. |
| KaTeX CDN unavailable | Low | High | Provide offline KaTeX bundle; document fallback to MathJax. Test CDN fallback. |
| Math expressions don't render in HTML (vs. PDF) | Medium | Medium | Reuse SymPy validation from latex-mcp; test KaTeX rendering; warn user if validation fails. |
| Citation resolution fails on complex BibTeX | Medium | Medium | Use robust bibtexparser; gracefully skip unparseable entries; warn user. |
| Markdown parsing misses edge cases (nested lists, tables) | Medium | Low | Use well-tested markdown library (not regex); write comprehensive tests. |
| HTML escaping creates invalid output | Low | High | Use Python `html.escape()` consistently; test with special chars (quotes, ampersands, angle brackets). |
| Feed generation produces invalid XML | Low | High | Validate feed with feedvalidator.org; use XML library instead of string concatenation. |
| Generated HTML is not semantically correct | Medium | Medium | Use semantic HTML5 elements (`<article>`, `<header>`, `<nav>`, `<main>`); test with outliner.js. |
| Performance regression on large documents | Low | Medium | Profile with large test files (100KB+); implement streaming/chunking if needed. |

---

## 6. Rollback & Contingency

**If markdown parsing fails:**
- Fall back to plain text output (minimal formatting)
- Return error with suggestion to check syntax
- Provide raw HTML input option for advanced users

**If KaTeX CDN is unavailable:**
- Fallback to client-side MathJax rendering
- Bundle offline KaTeX as optional npm package
- Provide warning in generated HTML

**If citation resolution fails:**
- Skip unresolved citations; add to warnings
- Generate plain text citation format (Author et al., Year)
- HTML is still valid, just unformatted

**If feed generation produces invalid XML:**
- Validate with feedvalidator.org API
- Return error with suggestion to check post metadata
- Provide manual feed template for advanced users

---

## 7. Appendices

### 7.1 Related Work

- **Hugo** (gohugo.io): Fast static site generator. Full site framework; not a single-file converter.
- **Jekyll** (jekyllrb.com): GitHub Pages backed. Ruby-based; heavy theme system.
- **Next.js / Gatsby** (nextjs.org, gatsbyjs.com): React-based. Overkill for simple blog conversion.
- **pandoc** (pandoc.org): Markdown → HTML conversion. Does the job but no SEO, no styling, no feed.
- **Quarto** (quarto.org): Notebook-to-blog. Similar vision; we're markdown-first and library-based.

### 7.2 Future Considerations

- **Markdown extensions:** Support GFM tables, strikethrough, task lists
- **Image optimization:** WebP/AVIF generation; responsive srcset
- **Comment system:** Disqus integration; self-hosted comments
- **Analytics:** Optional event tracking; privacy-respecting
- **Search:** Lunr.js or Algolia integration
- **Auto-generated sitemap:** For SEO
- **OpenSearch support:** Help search engines discover posts
- **AMP version:** Lightweight mobile variant
- **EPUB export:** E-book version of blog posts

### 7.3 Open Questions

1. Should we support Markdown table syntax or require HTML tables? CommonMark GFM tables for now.
2. Do we embed CSS in HTML or link to external stylesheet? Embed for portability.
3. Should featured images be required or optional? Optional; default to placeholder or omit.
4. How do we handle cross-links between blog posts? Deferred to v0.3; only internal citations for now.
5. Should we support comments? Deferred to future version; out of scope for v0.2.

### 7.4 Schema Backlog (Schemas this module requires)

Schemas defined in `packages/core/src/types/`:

- `DocumentRequest` (shared base) — reuse from latex-mcp
- `DocumentResponse` (shared base) — reuse from latex-mcp
- `ConversionWarning` (shared) — reuse from latex-mcp
- `NormalizedMD` (shared) — reuse from latex-mcp
- `BlogOptions` (module-specific)
- `BlogMetadata` (module-specific)
- `SEOMetadata` (module-specific)
- `FeedPost` (module-specific)

All schemas are TypeScript interfaces with JSON Schema equivalents for MCP tool validation.

---

## Audit Notes (2026-02-18)

**Auditor:** Claude (Opus-level audit)
**Status:** Reviewed — fixes applied

### Fixed in This Audit
- **CRITICAL-1:** MCP SDK imports corrected from `@anthropic-ai/sdk` to `@modelcontextprotocol/sdk`. Package.json dependency also updated.
- **HIGH-3:** Python version standardized to >= 3.11.
- **MEDIUM-5:** Pandoc version standardized to >= 3.0.

### Remaining Issues
- **CRITICAL-2:** Does not extend `ZenSciServer`. Should extend it.
- **CRITICAL-3:** Uses low-level `setRequestHandler` pattern.
- Tool naming (`convert_to_html`) follows snake_case — consistent.

### Decision Propagation (Cruz, 2026-02-18)

**Cluster 4 resolved: SEOMetadata aligned.** Flat `SEOMetadata` interface (`og_title`, `og_description`, `twitter_card`) refactored to `extends WebMetadataSchema` from `@zen-sci/core`. OG and Twitter Card fields now live under nested `og` and `twitter` sub-objects, matching the shared type structure. Migration note added for implementers.

**CRITICAL-2 resolved (Cluster 1):** `ZenSciServer` abstract base class deprecated. blog-mcp should use `createZenSciServer()` factory + `McpServer.registerTool()` with Zod schemas at implementation time.

**CRITICAL-3 resolved:** Low-level `setRequestHandler` replaced by `McpServer.registerTool()` with Zod input schemas per MCP SDK v2.
