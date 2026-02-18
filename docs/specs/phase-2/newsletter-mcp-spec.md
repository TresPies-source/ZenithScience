# newsletter-mcp Spec: Email Newsletter Publisher

**Author:** ZenSci Architecture (TresPiesDesign.com)
**Status:** Draft
**Created:** 2026-02-18
**Phase:** 2 (ships after Phase 1 core is validated)
**Grounded In:** Module strategy scout (2026-02-17), ZenSci semantic clusters

---

## 1. Vision
> Convert structured Markdown newsletters to email-safe MJML + HTML, optimized for Mailchimp, Substack, and direct SMTP delivery.

### The Core Insight
Newsletter authors manually format issues in email editors or copy-paste into platforms, losing formatting, links, and styling consistency. newsletter-mcp generates **email-safe HTML** from Markdown: MJML (responsive email markup language) for maximum email client compatibility, plus plain HTML fallback.

Unlike blog post generators (blog-mcp), newsletters are **constrained**: no JavaScript, limited CSS, strict email-client compatibility. newsletter-mcp handles this: auto-responsive layouts, image inlining, preheader text, CTA buttons, and platform-specific metadata (Mailchimp tags, Substack format).

### What Makes This Different
1. **MJML + HTML output:** Responsive email markup (MJML) + backwards-compatible HTML5.
2. **Email-safe formatting:** No JavaScript, limited CSS; tested in major email clients (Gmail, Outlook, Apple Mail).
3. **Platform support:** Auto-generate metadata for Mailchimp, Substack, and direct SMTP.
4. **CTA buttons & sections:** Predefined components for calls-to-action, featured content, footer links.
5. **Image optimization:** Compress and inline images; fallback for image-disabled clients.
6. **Preview text:** Auto-generate subject line and preheader preview text.

---

## 2. Goals & Success Criteria

### Primary Goals
1. Convert Markdown newsletter → MJML + HTML email-safe format.
2. Support multiple sections (hero, content, recommendations, CTA, footer).
3. Generate platform-specific metadata (Mailchimp, Substack, direct SMTP).
4. Optimize images for email; inline or reference with fallback.
5. Validate HTML against email client standards.
6. Support custom branding (colors, fonts, logo).

### Success Criteria
✅ Newsletter (5 sections, 10 links, 3 images, 1 CTA) generates valid MJML + HTML in <3s.
✅ Generated HTML validates as email-safe (W3C, MJML lint).
✅ Renders correctly in Gmail, Outlook, Apple Mail (via manual testing or automation).
✅ Images are optimized and embedded/referenced correctly.
✅ CTA button styling is consistent and clickable.
✅ End-to-end test: Markdown → Mailchimp HTML preview loads correctly.

### Non-Goals
❌ Newsletter subscription management (sign-up forms, list management).
❌ Analytics tracking (open rates, click tracking integration).
❌ Automated scheduling (send scheduling is user's responsibility).
❌ Template editor UI (user edits Markdown, not visual editor).

---

## 3. Technical Architecture

### 3.1 System Overview

```
Markdown Newsletter + YAML Config
    ↓
newsletter-mcp MCP Server (TypeScript)
    ├─ Parse sections and metadata
    ├─ Validate email-safe structure
    ├─ Route to Python engine
    └─ Return DocumentResponse (MJML + HTML)
    ↓
Python Engine (mjml-cli + pandoc)
    ├─ Convert Markdown to MJML sections
    ├─ Compile MJML → responsive HTML
    ├─ Generate platform metadata
    ├─ Optimize images
    └─ Return bytes (MJML + HTML + metadata)
    ↓
DocumentResponse (format: 'newsletter-html', artifacts: [html, mjml, metadata])
```

### 3.2 MCP Tool Definitions

#### Tool: `convert_to_newsletter`
**Description:** Convert Markdown newsletter to email-safe MJML + HTML.

**Input Schema:**
```json
{
  "type": "object",
  "properties": {
    "source": {
      "type": "string",
      "description": "Markdown content (sections separated by ##)"
    },
    "metadata": {
      "type": "object",
      "properties": {
        "subject": { "type": "string", "description": "Email subject line" },
        "preheader": { "type": "string", "description": "Email preview text (< 150 chars)" },
        "from_name": { "type": "string" },
        "from_email": { "type": "string" },
        "reply_to": { "type": "string" },
        "logo_url": { "type": "string" },
        "brand_color": { "type": "string", "description": "Hex color code" },
        "cta_button": {
          "type": "object",
          "properties": {
            "text": { "type": "string" },
            "url": { "type": "string" }
          }
        },
        "platform": {
          "type": "string",
          "enum": ["mailchimp", "substack", "smtp", "generic"],
          "description": "Target platform for metadata generation"
        }
      },
      "required": ["subject", "from_name", "from_email"]
    },
    "options": {
      "type": "object",
      "properties": {
        "include_footer": { "type": "boolean" },
        "include_unsubscribe": { "type": "boolean" },
        "image_optimization": { "type": "boolean" },
        "dark_mode_support": { "type": "boolean" }
      }
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
    "content": {
      "type": "string",
      "description": "Base64-encoded HTML email; includes MJML source as artifact"
    },
    "artifacts": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "type": { "type": "string", "enum": ["html", "mjml", "metadata", "validation_report"] },
          "filename": { "type": "string" },
          "content": { "type": "string" }
        }
      }
    },
    "metadata": {
      "type": "object",
      "properties": {
        "subject": { "type": "string" },
        "preheader": { "type": "string" },
        "section_count": { "type": "integer" },
        "link_count": { "type": "integer" },
        "image_count": { "type": "integer" },
        "email_safe": { "type": "boolean" },
        "file_size_bytes": { "type": "integer" }
      }
    },
    "elapsed": { "type": "number" }
  }
}
```

#### Tool: `validate_newsletter`
**Description:** Validate newsletter structure and email compatibility.

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
    "report": {
      "type": "object",
      "properties": {
        "section_count": { "type": "integer" },
        "link_count": { "type": "integer" },
        "image_count": { "type": "integer" },
        "html_safe": { "type": "boolean" },
        "subject_length": { "type": "integer" },
        "preheader_length": { "type": "integer" },
        "estimated_display_width": { "type": "string" }
      }
    }
  }
}
```

### 3.3 Core Integration Points

- **blog-mcp:** HTML styling patterns, link handling.
- **packages/core:** Logging, error handling.

---

### 3.4 TypeScript Implementation

```typescript
// zen-sci/packages/sdk/src/servers/newsletter-mcp/index.ts

import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from '@modelcontextprotocol/sdk/types.js';

const server = new Server(
  { name: 'newsletter-mcp', version: '0.1.0' },
  { capabilities: { tools: {} } }
);

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: 'convert_to_newsletter',
      description: 'Convert Markdown newsletter to email-safe MJML + HTML.',
      inputSchema: {
        type: 'object' as const,
        properties: {
          source: { type: 'string' },
          metadata: { type: 'object' },
          options: { type: 'object' },
        },
        required: ['source', 'metadata'],
      },
    },
    {
      name: 'validate_newsletter',
      description: 'Validate newsletter structure and email compatibility.',
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

  if (name === 'convert_to_newsletter') {
    const result = await convertToNewsletter(args as any);
    return { content: [{ type: 'text', text: JSON.stringify(result) }] };
  }

  if (name === 'validate_newsletter') {
    const result = await validateNewsletter(args as any);
    return { content: [{ type: 'text', text: JSON.stringify(result) }] };
  }

  throw new Error(`Unknown tool: ${name}`);
});

async function convertToNewsletter(args: any) {
  return await callPythonEngine('convert_newsletter', args);
}

async function validateNewsletter(args: any) {
  return await callPythonEngine('validate_newsletter', args);
}

async function callPythonEngine(method: string, params: any) {
  // Spawn Python subprocess
}

const transport = new StdioServerTransport();
server.connect(transport);
```

### 3.5 Python Processing Engine

```python
# zen-sci/servers/newsletter-mcp/processing/newsletter.py

import re
import tempfile
from pathlib import Path
from typing import Dict, List, Any, Tuple
import base64
import subprocess

async def generate_newsletter_html(
    source: str,
    metadata: Dict[str, Any],
    options: Dict[str, Any] = None,
) -> Tuple[str, str, Dict[str, str]]:
    """
    Generate email-safe newsletter.

    Returns:
        Tuple[HTML content, MJML source, metadata dict].
    """

    options = options or {}

    # Step 1: Parse Markdown into sections
    sections = parse_newsletter_sections(source)

    # Step 2: Build MJML structure
    mjml_content = build_mjml_newsletter(sections, metadata, options)

    # Step 3: Compile MJML to HTML using mjml-cli
    html_content = compile_mjml_to_html(mjml_content)

    # Step 4: Validate HTML for email safety
    validation = validate_email_html(html_content)

    artifacts = {
        'mjml': mjml_content,
        'validation_report': validation,
    }

    return html_content, mjml_content, artifacts


def parse_newsletter_sections(source: str) -> List[Dict[str, str]]:
    """Parse Markdown into newsletter sections."""
    sections = []

    # Split by ## (h2 headers)
    pattern = r'^## (.+)$'
    lines = source.split('\n')

    current_title = None
    current_content = []

    for line in lines:
        match = re.match(pattern, line)
        if match:
            # Save previous section
            if current_title:
                sections.append({
                    'title': current_title,
                    'content': '\n'.join(current_content).strip(),
                })
            current_title = match.group(1)
            current_content = []
        else:
            current_content.append(line)

    # Save last section
    if current_title:
        sections.append({
            'title': current_title,
            'content': '\n'.join(current_content).strip(),
        })

    return sections


def build_mjml_newsletter(
    sections: List[Dict[str, str]],
    metadata: Dict[str, Any],
    options: Dict[str, Any],
) -> str:
    """Build MJML email structure."""

    mjml_lines = ['<mjml>']
    mjml_lines.append('  <mj-head>')

    # Metadata
    mjml_lines.append(f"    <mj-title>{metadata.get('subject', '')}</mj-title>")
    mjml_lines.append(f"    <mj-preview>{metadata.get('preheader', '')}</mj-preview>")

    # Styling
    color = metadata.get('brand_color', '#0066cc')
    mjml_lines.append(f'    <mj-style>.button {{ background-color: {color}; }}</mj-style>')

    mjml_lines.append('  </mj-head>')
    mjml_lines.append('  <mj-body>')

    # Header with logo
    if metadata.get('logo_url'):
        mjml_lines.append('    <mj-section>')
        mjml_lines.append(f'      <mj-column><mj-image src="{metadata["logo_url"]}" alt="Logo" width="200px" /></mj-column>')
        mjml_lines.append('    </mj-section>')

    # Content sections
    for section in sections:
        mjml_lines.append('    <mj-section>')
        mjml_lines.append('      <mj-column>')

        # Section title
        mjml_lines.append(f'        <mj-text font-size="20px" font-weight="bold">{section["title"]}</mj-text>')

        # Section content (convert Markdown to HTML inline)
        import pandoc
        html_content = pandoc.convert_text(
            section['content'],
            format='markdown',
            to='html',
            extra_args=['--no-highlight'],
        )
        mjml_lines.append(f'        <mj-text>{html_content}</mj-text>')

        mjml_lines.append('      </mj-column>')
        mjml_lines.append('    </mj-section>')

    # CTA button
    if metadata.get('cta_button'):
        mjml_lines.append('    <mj-section>')
        mjml_lines.append('      <mj-column>')
        cta = metadata['cta_button']
        mjml_lines.append(f'        <mj-button href="{cta["url"]}" background-color="{metadata.get("brand_color", "#0066cc")}">{cta["text"]}</mj-button>')
        mjml_lines.append('      </mj-column>')
        mjml_lines.append('    </mj-section>')

    # Footer
    if options.get('include_footer', True):
        mjml_lines.append('    <mj-section>')
        mjml_lines.append('      <mj-column>')
        mjml_lines.append(f'        <mj-text font-size="12px" color="#888">')
        mjml_lines.append(f'          <p>{metadata.get("from_name", "")} &nbsp; | &nbsp; <a href="mailto:{metadata.get("reply_to", metadata.get("from_email", ""))}">Reply</a></p>')
        if options.get('include_unsubscribe', True):
            mjml_lines.append('          <p><a href="*|UNSUB|*">Unsubscribe</a></p>')
        mjml_lines.append('        </mj-text>')
        mjml_lines.append('      </mj-column>')
        mjml_lines.append('    </mj-section>')

    mjml_lines.append('  </mj-body>')
    mjml_lines.append('</mjml>')

    return '\n'.join(mjml_lines)


def compile_mjml_to_html(mjml_content: str) -> str:
    """Compile MJML to HTML using mjml-cli."""

    with tempfile.TemporaryDirectory() as tmpdir:
        mjml_file = Path(tmpdir) / 'newsletter.mjml'
        html_file = Path(tmpdir) / 'newsletter.html'

        mjml_file.write_text(mjml_content, encoding='utf-8')

        # Call mjml-cli
        result = subprocess.run(
            ['mjml', str(mjml_file), '-o', str(html_file)],
            capture_output=True,
            text=True,
        )

        if result.returncode == 0 and html_file.exists():
            return html_file.read_text(encoding='utf-8')

    raise RuntimeError('MJML compilation failed')


def validate_email_html(html: str) -> str:
    """Validate HTML for email compatibility."""

    errors = []

    # Check for JavaScript
    if '<script' in html.lower():
        errors.append('Contains <script> tags (not email-safe)')

    # Check for external stylesheets (should be inlined)
    if '<link' in html.lower():
        errors.append('Contains external <link> tags (CSS should be inlined)')

    # Check for responsive design (mobile-first)
    if 'media query' not in html.lower() and '@media' not in html:
        errors.append('Missing media queries for responsive design')

    # Check for alt text on images
    img_count = html.count('<img')
    alt_count = html.count('alt=')
    if alt_count < img_count:
        errors.append(f'Missing alt text on {img_count - alt_count} image(s)')

    report = '\n'.join(errors) if errors else 'Email-safe: No issues detected'
    return report
```

### 3.6 Key Schemas

```typescript
// zen-sci/packages/core/src/schemas/newsletter.ts

export interface NewsletterMetadata {
  subject: string;
  preheader?: string; // Preview text < 150 chars
  from_name: string;
  from_email: string;
  reply_to?: string;
  logo_url?: string;
  brand_color?: string; // Hex color
  cta_button?: {
    text: string;
    url: string;
  };
  platform: 'mailchimp' | 'substack' | 'smtp' | 'generic';
}

export interface NewsletterDocumentRequest extends DocumentRequest {
  format: 'newsletter-html';
  source: string;
  metadata: NewsletterMetadata;
  options?: NewsletterOptions;
}

export interface NewsletterOptions {
  include_footer?: boolean;
  include_unsubscribe?: boolean;
  image_optimization?: boolean;
  dark_mode_support?: boolean;
}

export interface NewsletterResponse extends DocumentResponse {
  format: 'newsletter-html';
  content: string; // HTML content
  artifacts: NewsletterArtifact[];
  metadata: NewsletterMetadata & {
    section_count: number;
    link_count: number;
    image_count: number;
    email_safe: boolean;
    file_size_bytes: number;
  };
}

export interface NewsletterArtifact extends Artifact {
  type: 'html' | 'mjml' | 'metadata' | 'validation_report';
}
```

---

### 3.7 Phase 2 Infrastructure Requirements

- MJML CLI integration (external tool).
- Email validation rules (HTML, images, links).
- Platform-specific metadata generation (Mailchimp, Substack).

---

### 3.8 Performance & Constraints

| Constraint | Target | Notes |
|-----------|--------|-------|
| Newsletter (5 sections, 10 links) | <3s | MJML compilation |
| HTML file size | <100 KB | Optimized images, inlined CSS |
| Email client compatibility | 90%+ | MJML standard compliance |

---

## 4. Implementation Plan

### 4.1 Phased Approach

| Phase | Timeline | Deliverables | Dependencies |
|-------|----------|--------------|--------------|
| **Alpha (Week 1-2)** | 2 weeks | Basic MJML → HTML generation; no platform metadata | blog-mcp v0.2 |
| **Beta (Week 3-4)** | 2 weeks | + Platform metadata (Mailchimp, Substack); image optimization | All Alpha |
| **Candidate (Week 5-6)** | 2 weeks | + Full validation; dark mode support; footer/unsubscribe | All Beta |
| **Release (Week 7)** | 1 week | Full test suite; documentation | All Candidate |

### 4.2 Week-by-Week Breakdown

**Week 1: MJML Structure & Basic Compilation**
- [ ] Set up zen-sci/servers/newsletter-mcp/ directory structure
- [ ] Implement MCP server skeleton
- [ ] Build MJML structure from Markdown sections
- [ ] Set up mjml-cli integration
- [ ] Implement MJML → HTML compilation
- [ ] Unit tests: MJML generation; HTML compilation
- [ ] Integration test: Simple newsletter → HTML

**Week 2: Metadata & Styling**
- [ ] Add subject, preheader, from/reply metadata
- [ ] Implement logo embedding
- [ ] Add brand color support
- [ ] Implement CTA button generation
- [ ] Implement `convert_to_newsletter` tool
- [ ] Unit tests: metadata handling; styling
- [ ] Integration test: Styled newsletter with metadata

**Week 3: Validation & Platform Metadata**
- [ ] Implement `validate_newsletter` tool
- [ ] Add email-safety checks (no JS, CSS inlined, alt text)
- [ ] Add Mailchimp metadata generation (merge tags, segments)
- [ ] Add Substack metadata generation
- [ ] Unit tests: validation logic; platform metadata
- [ ] Integration test: Newsletter → Mailchimp preview

**Week 4: Image Optimization & Footer**
- [ ] Implement image optimization (compression, sizing)
- [ ] Add footer section support
- [ ] Add unsubscribe link generation
- [ ] Add contact info formatting
- [ ] Unit tests: image optimization; footer styling
- [ ] Integration test: Newsletter with images and footer

**Week 5: Dark Mode & Advanced Features**
- [ ] Implement dark mode CSS support (prefers-color-scheme)
- [ ] Add media query optimization (mobile-first)
- [ ] Add link tracking metadata (platform-specific)
- [ ] Unit tests: dark mode rendering; responsive design
- [ ] Integration test: Newsletter renders correctly on mobile

**Week 6: Email Client Testing**
- [ ] Manual testing in Gmail, Outlook, Apple Mail
- [ ] Fix rendering issues (fallback styling)
- [ ] Test with email preview tools (Litmus, similar)
- [ ] Validate HTML structure completeness
- [ ] Integration test: Cross-email-client validation

**Week 7: Documentation & Release**
- [ ] Write API documentation
- [ ] Create example newsletters (product update, digest)
- [ ] Write Python processing engine docs
- [ ] Finalize error messages and validation warnings
- [ ] Publish v0.1.0 to npm (@zen-sci/newsletter-mcp)
- [ ] Full regression test suite (25+ unit + 8+ integration tests)

### 4.3 Dependencies & Prerequisites

**Hard Dependencies:**
- blog-mcp v0.2 (HTML styling patterns)
- packages/core v1.0

**External Tools:**
- mjml-cli (MJML compilation)
- Image optimization libraries (imagemagick or Pillow)

---

### 4.4 Testing Strategy

**Unit Tests:** MJML generation, HTML compilation, metadata handling, validation logic.

**Integration Tests:** End-to-end newsletter generation; platform metadata correctness; email client rendering (automated or manual sampling).

---

## 5. Risk Assessment & Mitigation

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Email client compatibility issues (Outlook, Gmail peculiarities) | Medium | Medium | Use MJML standard; manual testing in major clients; document known issues |
| Image handling across clients (some disable images) | Low | Low | Always include alt text and fallback text |
| mjml-cli not available in deployment | Low | Medium | Provide error message; document requirement; fallback to basic HTML |

---

## 6. Rollback & Contingency

**If dark mode support is problematic (Week 5):**
- Ship without dark mode in v0.1; add in v0.1.1.

**If email client compatibility is poor (Week 6):**
- Focus on Gmail/Outlook/Apple Mail (95% of users); defer others to v0.1.1.

---

## 7. Appendices

### 7.1 Related Work & Standards

- **MJML (Mailjet Markup Language):** Responsive email markup language (open-source).
- **MJML Documentation:** Full MJML specification and best practices.
- **Email Standards:** HTML email best practices (CSS inlining, alt text, responsive design).
- **MIME RFC 2045:** Email message format standard.

### 7.2 Future Considerations (v1.1+)

- **Drag-and-drop template editor:** Visual UI for non-technical users.
- **A/B testing support:** Generate subject line variants.
- **Subscriber segmentation:** Conditional sections based on subscriber attributes.
- **Analytics integration:** Click tracking, open tracking (platform-specific).
- **Automation workflows:** Scheduled sends, triggered sends.

### 7.3 Open Questions

1. **Should we support plain-text email version?** (MVP: HTML only; plain-text fallback in v0.1.1.)
2. **How deep should image optimization go?** (MVP: basic compression; advanced optimization in v0.1.1.)

---

**End of newsletter-mcp Specification**

---

## Audit Notes (2026-02-18)

**Auditor:** Claude (Opus-level audit)
**Status:** Reviewed

### Issues
- **CRITICAL-2:** Instantiates `Server` directly instead of extending `ZenSciServer`.
- **CRITICAL-3:** Uses low-level `setRequestHandler` pattern.
- MJML integration approach (using `mjml-cli` subprocess) confirmed correct via Context7.
- MCP SDK imports correct. Tool naming snake_case — consistent.
- Spec is shorter/less detailed than other Phase 2 specs. Adequate for implementation but could benefit from more detailed Python engine code.

#### Decision Propagation (Cruz, 2026-02-18)
- **CRITICAL-2 resolved:** `ZenSciServer` abstract base class deprecated. Use `createZenSciServer()` factory + `McpServer.registerTool()` with Zod schemas at implementation time.
- **CRITICAL-3 resolved:** Low-level `setRequestHandler` replaced by `McpServer.registerTool()` with Zod input schemas per MCP SDK v2.
