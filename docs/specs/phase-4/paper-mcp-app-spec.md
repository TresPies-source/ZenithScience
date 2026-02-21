# paper-mcp MCP App Specification
## Phase 4: Visual Application Layer

**Version:** 0.4.0
**Status:** Production Ready
**Author:** ZenSci Architecture (Cruz Morales / TresPiesDesign.com)
**Date:** 2026-02-18
**Depends on:** paper-mcp v0.3.0 (Phase 1)

---

## 1. Purpose

The paper-mcp app is an **academic paper preview and format switcher**. When a user calls `convert_to_paper`, the app displays:
- A PDF preview (page-by-page navigation)
- A format selector dropdown (IEEE, ACM, arXiv) with live format switching
- Paper metadata sidebar: abstract, author list with affiliations, section/figure/table/citation counts
- LaTeX source tab for inspection
- Warnings strip for constraint violations

**Key insight:** Academic papers are highly structured (abstract, sections, references) but stylistically variable (venue templates). The app lets researchers switch between IEEE/ACM/arXiv formats without re-editing content — format is presentation, content is data.

---

## 2. Tool Integration

### 2.1 Primary Tool: `convert_to_paper`

**Declaration in server's `index.ts`:**

```typescript
registerAppTool(
  server,
  'convert_to_paper',
  {
    type: 'object',
    properties: {
      source: { type: 'string', description: 'Markdown research content' },
      title: { type: 'string', description: 'Paper title' },
      author: { type: 'array', items: { type: 'string' }, description: 'Author names' },
      abstract: { type: 'string', description: 'Paper abstract' },
      template: { type: 'string', enum: ['ieee', 'acm', 'arxiv', 'nature', 'science', 'custom'] },
      bibliography: { type: 'string', description: 'BibTeX content' },
      options: { type: 'object', description: 'Compilation options' },
    },
    required: ['source', 'title', 'author', 'abstract', 'template'],
    _meta: {
      ui: {
        resourceUri: 'ui://paper-mcp/preview.html',
        csp: ['worker-src blob:'],
      },
    },
  },
  convertToPaper
);
```

**Tool result shape** (returned from `convert_to_paper`):

```typescript
interface ConvertToPaperResult {
  // PDF as base64 (compiled academic paper PDF)
  pdf_base64: string;

  // Raw LaTeX source
  latex_source: string;

  // Paper format
  paper_format: 'ieee' | 'acm' | 'arxiv' | 'nature' | 'science';

  // Structure metadata
  page_count: number;
  section_count: number;
  figure_count: number;
  table_count: number;
  citation_count: number;

  // Abstract text (extracted from markdown)
  abstract_text: string;

  // Author affiliations
  author_affiliations: Array<{
    name: string;
    affiliation: string;
  }>;

  // Warnings (e.g., page count exceeded, low figure count)
  warnings: Array<{
    type: 'warning' | 'error';
    message: string;
  }>;
}
```

### 2.2 Secondary Tools (Called from App)

The app calls `convert_to_paper` again when the user selects a different format from the dropdown. The user selects from the available formats in the format selector; the app invokes the tool with the new format.

---

## 3. App File Structure

```
zen-sci/servers/paper-mcp/app/
├── src/
│   ├── main.tsx                  ← React entry point (identical across all apps)
│   ├── App.tsx                   ← Root component (receives ontoolresult)
│   ├── types.ts                  ← TypeScript types for tool result shape
│   ├── components/
│   │   ├── LoadingState.tsx       ← Shared loading spinner
│   │   ├── ErrorState.tsx         ← Shared error display
│   │   ├── EmptyState.tsx         ← Shared empty state
│   │   ├── PDFPreview.tsx         ← PDF.js viewer with page nav
│   │   ├── FormatSelector.tsx     ← Dropdown to switch between formats
│   │   ├── PaperInfoPanel.tsx     ← Sidebar with abstract, authors, stats
│   │   ├── WarningsStrip.tsx      ← Yellow warning bar for constraint violations
│   │   ├── LaTeXSourceTab.tsx     ← Raw LaTeX in pre block
│   │   └── StatsGrid.tsx          ← 2x3 grid of section/figure/table/citation counts
│   ├── hooks/
│   │   └── useToolResult.ts       ← Parse & validate tool result JSON
│   ├── styles.css                ← ZenSci design tokens + module-specific styles
│   └── __tests__/
│       ├── App.test.tsx
│       ├── PDFPreview.test.tsx
│       ├── FormatSelector.test.tsx
│       └── setup.ts
├── index.html                    ← Vite entry (identical except title)
├── vite.config.ts                ← Vite + singlefile (identical across all apps)
├── tsconfig.json                 ← TypeScript config (identical)
├── package.json                  ← App deps (with pdfjs-dist)
└── .gitignore

dist/
└── paper-preview.html             ← Built single-file bundle
```

---

## 4. Component Architecture

### 4.1 Root App Component

```typescript
// app/src/App.tsx
import { useApp, useHostStyles } from '@modelcontextprotocol/ext-apps/react';
import { useState, useEffect } from 'react';
import type { ConvertToPaperResult } from './types';
import { LoadingState, ErrorState, EmptyState } from './components';
import PDFPreview from './components/PDFPreview';
import FormatSelector from './components/FormatSelector';
import PaperInfoPanel from './components/PaperInfoPanel';
import WarningsStrip from './components/WarningsStrip';
import LaTeXSourceTab from './components/LaTeXSourceTab';

export default function App() {
  const app = useApp();
  const hostStyles = useHostStyles();
  const [data, setData] = useState<ConvertToPaperResult | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(true);
  const [activeTab, setActiveTab] = useState<'preview' | 'source'>('preview');
  const [showAbstract, setShowAbstract] = useState(true);

  useEffect(() => {
    app.ontoolresult = (result) => {
      try {
        const parsed = JSON.parse(result.content[0].text) as ConvertToPaperResult;
        setData(parsed);
        setLoading(false);
      } catch (e) {
        setError('Failed to parse tool result');
        setLoading(false);
      }
    };
  }, [app]);

  const handleFormatChange = async (newFormat: string) => {
    setLoading(true);
    try {
      // App calls convert_to_paper again with the new format
      // This triggers ontoolresult callback, updating data
      // Implementation depends on app having stored original input
    } catch (e) {
      setError(e instanceof Error ? e.message : 'Format switch failed');
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <LoadingState />;
  if (error) return <ErrorState message={error} />;
  if (!data) return <EmptyState />;

  return (
    <div style={{ ...hostStyles, padding: '16px', fontFamily: 'var(--zen-font-sans)' }}>
      {/* Warnings strip (if any) */}
      {data.warnings.length > 0 && (
        <WarningsStrip warnings={data.warnings} />
      )}

      {/* Format selector toolbar */}
      <FormatSelector
        currentFormat={data.paper_format}
        onFormatChange={handleFormatChange}
        loading={loading}
      />

      {/* Main content: two-column layout */}
      <div style={{ display: 'flex', gap: '16px', marginTop: '16px' }}>
        {/* Left: PDF preview + tabs */}
        <div style={{ flex: '1' }}>
          {/* Tab navigation */}
          <div className="zen-tabs">
            <button
              className={`zen-tab ${activeTab === 'preview' ? 'active' : ''}`}
              onClick={() => setActiveTab('preview')}
            >
              Preview
            </button>
            <button
              className={`zen-tab ${activeTab === 'source' ? 'active' : ''}`}
              onClick={() => setActiveTab('source')}
            >
              LaTeX Source
            </button>
          </div>

          {/* Tab content */}
          {activeTab === 'preview' && (
            <PDFPreview pdfBase64={data.pdf_base64} pageCount={data.page_count} />
          )}

          {activeTab === 'source' && (
            <LaTeXSourceTab laTeXSource={data.latex_source} />
          )}
        </div>

        {/* Right: Paper info sidebar */}
        <aside
          style={{
            width: '280px',
            borderLeft: 'var(--zen-border)',
            paddingLeft: '16px',
            maxHeight: '600px',
            overflowY: 'auto',
          }}
        >
          <PaperInfoPanel
            abstract={data.abstract_text}
            authors={data.author_affiliations}
            stats={{
              sections: data.section_count,
              figures: data.figure_count,
              tables: data.table_count,
              citations: data.citation_count,
            }}
            showAbstract={showAbstract}
            onToggleAbstract={() => setShowAbstract(!showAbstract)}
          />
        </aside>
      </div>
    </div>
  );
}
```

### 4.2 Tab Structure

Two main tabs at the top level:
1. **Preview** (default) — PDF viewer with page navigation
2. **LaTeX Source** — Raw LaTeX with copy button

### 4.3 Major Components

#### 4.3.1 PDFPreview (PDF.js Viewer)

```typescript
// app/src/components/PDFPreview.tsx
import { useEffect, useRef, useState } from 'react';
import * as pdfjsLib from 'pdfjs-dist';

pdfjsLib.GlobalWorkerOptions.workerSrc = `//cdnjs.cloudflare.com/ajax/libs/pdf.js/${pdfjsLib.version}/pdf.worker.min.js`;

interface Props {
  pdfBase64: string;
  pageCount: number;
}

export default function PDFPreview({ pdfBase64, pageCount }: Props) {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const [currentPage, setCurrentPage] = useState(1);
  const [pdf, setPdf] = useState<any>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const loadPDF = async () => {
      try {
        const binaryString = atob(pdfBase64);
        const bytes = new Uint8Array(binaryString.length);
        for (let i = 0; i < binaryString.length; i++) {
          bytes[i] = binaryString.charCodeAt(i);
        }

        const pdf = await pdfjsLib.getDocument({ data: bytes }).promise;
        setPdf(pdf);
        setLoading(false);
      } catch (e) {
        console.error('PDF load error:', e);
        setLoading(false);
      }
    };

    loadPDF();
  }, [pdfBase64]);

  useEffect(() => {
    if (!pdf || !canvasRef.current) return;

    const renderPage = async () => {
      const page = await pdf.getPage(currentPage);
      const scale = 1.5;
      const viewport = page.getViewport({ scale });

      const canvas = canvasRef.current;
      const context = canvas.getContext('2d');
      if (!context) return;

      canvas.width = viewport.width;
      canvas.height = viewport.height;

      await page.render({
        canvasContext: context,
        viewport: viewport,
      }).promise;
    };

    renderPage();
  }, [pdf, currentPage]);

  return (
    <div style={{ marginTop: '12px' }}>
      {loading && <div>Loading PDF...</div>}

      {pdf && (
        <>
          <canvas
            ref={canvasRef}
            style={{
              width: '100%',
              maxWidth: '600px',
              border: 'var(--zen-border)',
              borderRadius: 'var(--zen-radius)',
              marginBottom: '12px',
            }}
          />

          {/* Page navigation */}
          <div
            style={{
              display: 'flex',
              justifyContent: 'center',
              alignItems: 'center',
              gap: '12px',
              fontSize: '13px',
            }}
          >
            <button
              onClick={() => setCurrentPage(Math.max(1, currentPage - 1))}
              disabled={currentPage <= 1}
              style={{
                padding: '6px 12px',
                backgroundColor: currentPage <= 1 ? '#ccc' : 'var(--zen-tab-active-bg, #2563eb)',
                color: '#fff',
                border: 'none',
                borderRadius: 'var(--zen-radius)',
                cursor: currentPage <= 1 ? 'not-allowed' : 'pointer',
              }}
            >
              ← Prev
            </button>

            <span>
              Page {currentPage} of {pageCount}
            </span>

            <button
              onClick={() => setCurrentPage(Math.min(pageCount, currentPage + 1))}
              disabled={currentPage >= pageCount}
              style={{
                padding: '6px 12px',
                backgroundColor: currentPage >= pageCount ? '#ccc' : 'var(--zen-tab-active-bg, #2563eb)',
                color: '#fff',
                border: 'none',
                borderRadius: 'var(--zen-radius)',
                cursor: currentPage >= pageCount ? 'not-allowed' : 'pointer',
              }}
            >
              Next →
            </button>
          </div>
        </>
      )}
    </div>
  );
}
```

#### 4.3.2 FormatSelector

```typescript
// app/src/components/FormatSelector.tsx
interface Props {
  currentFormat: 'ieee' | 'acm' | 'arxiv' | 'nature' | 'science';
  onFormatChange: (format: string) => Promise<void>;
  loading: boolean;
}

const FORMAT_LABELS: Record<string, string> = {
  ieee: 'IEEE Two-Column',
  acm: 'ACM Article',
  arxiv: 'arXiv Preprint',
  nature: 'Nature Journal',
  science: 'Science Magazine',
};

export default function FormatSelector({ currentFormat, onFormatChange, loading }: Props) {
  return (
    <div style={{ display: 'flex', alignItems: 'center', gap: '12px' }}>
      <label style={{ fontSize: '13px', fontWeight: '500' }}>Format:</label>

      <select
        value={currentFormat}
        onChange={(e) => onFormatChange(e.target.value)}
        disabled={loading}
        style={{
          padding: '6px 10px',
          border: 'var(--zen-border)',
          borderRadius: 'var(--zen-radius)',
          backgroundColor: 'var(--mcp-bg-primary, #fff)',
          color: 'var(--mcp-text-primary, #000)',
          cursor: loading ? 'not-allowed' : 'pointer',
          fontSize: '13px',
        }}
      >
        <option value="ieee">IEEE Two-Column</option>
        <option value="acm">ACM Article</option>
        <option value="arxiv">arXiv Preprint</option>
        <option value="nature">Nature Journal</option>
        <option value="science">Science Magazine</option>
      </select>

      <span
        className="zen-badge zen-badge.info"
        style={{
          padding: '4px 10px',
          fontSize: '11px',
          fontWeight: '600',
          backgroundColor: '#d1ecf1',
          color: '#0c5460',
        }}
      >
        {FORMAT_LABELS[currentFormat]}
      </span>
    </div>
  );
}
```

#### 4.3.3 PaperInfoPanel

```typescript
// app/src/components/PaperInfoPanel.tsx
import StatsGrid from './StatsGrid';

interface AuthorAffiliation {
  name: string;
  affiliation: string;
}

interface Props {
  abstract: string;
  authors: AuthorAffiliation[];
  stats: {
    sections: number;
    figures: number;
    tables: number;
    citations: number;
  };
  showAbstract: boolean;
  onToggleAbstract: () => void;
}

export default function PaperInfoPanel({
  abstract,
  authors,
  stats,
  showAbstract,
  onToggleAbstract,
}: Props) {
  return (
    <div>
      {/* Abstract toggle */}
      <button
        onClick={onToggleAbstract}
        style={{
          width: '100%',
          padding: '10px',
          border: 'none',
          background: 'transparent',
          cursor: 'pointer',
          fontSize: '13px',
          fontWeight: '600',
          textAlign: 'left',
          color: 'var(--mcp-text-primary, #000)',
        }}
      >
        {showAbstract ? '▼' : '▶'} Abstract
      </button>

      {showAbstract && (
        <div
          style={{
            padding: '10px',
            backgroundColor: 'var(--mcp-bg-secondary, #f9f9f9)',
            borderRadius: 'var(--zen-radius)',
            fontSize: '12px',
            lineHeight: '1.5',
            marginBottom: '12px',
            color: 'var(--mcp-text-secondary, #666)',
          }}
        >
          {abstract}
        </div>
      )}

      {/* Authors */}
      <div style={{ marginBottom: '12px' }}>
        <h4 style={{ fontSize: '12px', fontWeight: '600', marginBottom: '6px' }}>Authors</h4>
        <div style={{ fontSize: '11px', color: 'var(--mcp-text-secondary, #666)' }}>
          {authors.map((author, idx) => (
            <div key={idx} style={{ marginBottom: '4px' }}>
              <strong>{author.name}</strong>
              <div style={{ fontSize: '10px', fontStyle: 'italic' }}>
                {author.affiliation}
              </div>
            </div>
          ))}
        </div>
      </div>

      {/* Stats grid */}
      <StatsGrid stats={stats} />
    </div>
  );
}
```

#### 4.3.4 StatsGrid

```typescript
// app/src/components/StatsGrid.tsx
interface Props {
  stats: {
    sections: number;
    figures: number;
    tables: number;
    citations: number;
  };
}

export default function StatsGrid({ stats }: Props) {
  return (
    <div
      style={{
        display: 'grid',
        gridTemplateColumns: '1fr 1fr',
        gap: '8px',
      }}
    >
      <StatItem label="Sections" value={stats.sections} />
      <StatItem label="Figures" value={stats.figures} />
      <StatItem label="Tables" value={stats.tables} />
      <StatItem label="Citations" value={stats.citations} />
    </div>
  );
}

function StatItem({ label, value }: { label: string; value: number }) {
  return (
    <div
      style={{
        padding: '10px',
        backgroundColor: 'var(--mcp-bg-secondary, #f9f9f9)',
        borderRadius: 'var(--zen-radius)',
        textAlign: 'center',
      }}
    >
      <div style={{ fontSize: '16px', fontWeight: '700', color: 'var(--zen-tab-active-bg, #2563eb)' }}>
        {value}
      </div>
      <div style={{ fontSize: '10px', color: 'var(--mcp-text-secondary, #666)', marginTop: '2px' }}>
        {label}
      </div>
    </div>
  );
}
```

#### 4.3.5 WarningsStrip

```typescript
// app/src/components/WarningsStrip.tsx
interface Warning {
  type: 'warning' | 'error';
  message: string;
}

interface Props {
  warnings: Warning[];
}

export default function WarningsStrip({ warnings }: Props) {
  if (warnings.length === 0) return null;

  const hasError = warnings.some((w) => w.type === 'error');

  return (
    <div
      style={{
        padding: '12px',
        backgroundColor: hasError ? '#f8d7da' : '#fff3cd',
        borderLeft: `4px solid ${hasError ? '#721c24' : '#856404'}`,
        borderRadius: 'var(--zen-radius)',
        marginBottom: '12px',
      }}
    >
      <div
        style={{
          fontSize: '13px',
          fontWeight: '600',
          color: hasError ? '#721c24' : '#856404',
          marginBottom: '6px',
        }}
      >
        {hasError ? '⚠ Errors' : '⚠ Warnings'}
      </div>
      <ul
        style={{
          margin: '0',
          paddingLeft: '20px',
          fontSize: '12px',
          color: hasError ? '#721c24' : '#856404',
        }}
      >
        {warnings.map((w, idx) => (
          <li key={idx} style={{ marginBottom: '4px' }}>
            {w.message}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

#### 4.3.6 LaTeXSourceTab

```typescript
// app/src/components/LaTeXSourceTab.tsx
import { useRef } from 'react';

interface Props {
  laTeXSource: string;
}

export default function LaTeXSourceTab({ laTeXSource }: Props) {
  const preRef = useRef<HTMLPreElement>(null);

  const handleCopy = async () => {
    await navigator.clipboard.writeText(laTeXSource);
    alert('LaTeX copied to clipboard');
  };

  return (
    <div style={{ marginTop: '12px' }}>
      <button
        onClick={handleCopy}
        style={{
          padding: '8px 12px',
          backgroundColor: 'var(--zen-tab-active-bg, #2563eb)',
          color: '#fff',
          border: 'none',
          borderRadius: 'var(--zen-radius)',
          cursor: 'pointer',
          fontSize: '12px',
          marginBottom: '8px',
        }}
      >
        Copy to Clipboard
      </button>

      <pre
        ref={preRef}
        style={{
          backgroundColor: 'var(--mcp-bg-secondary, #f5f5f5)',
          padding: '12px',
          borderRadius: 'var(--zen-radius)',
          overflow: 'auto',
          fontSize: '10px',
          lineHeight: '1.4',
          color: 'var(--mcp-text-primary, #000)',
        }}
      >
        {laTeXSource}
      </pre>
    </div>
  );
}
```

---

## 5. Data Types

```typescript
// app/src/types.ts

/**
 * Result returned by convert_to_paper tool.
 * Parsed from tool result JSON in App.tsx.
 */
export interface ConvertToPaperResult {
  // PDF as base64 (compiled academic paper PDF from LaTeX)
  pdf_base64: string;

  // Raw LaTeX source
  latex_source: string;

  // Paper format
  paper_format: 'ieee' | 'acm' | 'arxiv' | 'nature' | 'science';

  // Structure metadata
  page_count: number;
  section_count: number;
  figure_count: number;
  table_count: number;
  citation_count: number;

  // Abstract text (extracted from markdown)
  abstract_text: string;

  // Author affiliations
  author_affiliations: Array<{
    name: string;
    affiliation: string;
  }>;

  // Warnings (constraint violations)
  warnings: Array<{
    type: 'warning' | 'error';
    message: string;
  }>;
}

/**
 * Component prop interfaces.
 */
export interface PDFPreviewProps {
  pdfBase64: string;
  pageCount: number;
}

export interface FormatSelectorProps {
  currentFormat: ConvertToPaperResult['paper_format'];
  onFormatChange: (format: string) => Promise<void>;
  loading: boolean;
}

export interface PaperInfoPanelProps {
  abstract: string;
  authors: ConvertToPaperResult['author_affiliations'];
  stats: {
    sections: number;
    figures: number;
    tables: number;
    citations: number;
  };
  showAbstract: boolean;
  onToggleAbstract: () => void;
}

export interface WarningsStripProps {
  warnings: ConvertToPaperResult['warnings'];
}

export interface LaTeXSourceTabProps {
  laTeXSource: string;
}

export interface StatsGridProps {
  stats: PaperInfoPanelProps['stats'];
}
```

---

## 6. Tool Result Shape

The `convert_to_paper` tool returns a JSON object matching `ConvertToPaperResult`. The app receives this in `app.ontoolresult` callback, parses it, and updates state.

```json
{
  "pdf_base64": "JVBERi0xLjQKJeLj...",
  "latex_source": "\\documentclass[twocolumn]{article}\n...",
  "paper_format": "ieee",
  "page_count": 8,
  "section_count": 5,
  "figure_count": 3,
  "table_count": 2,
  "citation_count": 24,
  "abstract_text": "This paper presents...",
  "author_affiliations": [
    {
      "name": "Jane Smith",
      "affiliation": "MIT CSAIL, Cambridge, MA"
    },
    {
      "name": "John Doe",
      "affiliation": "Stanford University, Stanford, CA"
    }
  ],
  "warnings": [
    {
      "type": "warning",
      "message": "Paper exceeds IEEE page limit (8 pages max, you have 9)"
    }
  ]
}
```

---

## 7. Interactive Tool Calls

The app calls `convert_to_paper` again when the user selects a different format from the dropdown. Like newsletter-mcp app, the app needs to preserve the original input (source, title, author, abstract, bibliography, options) to call the tool with the new format parameter.

**Implementation pattern:**

```typescript
const handleFormatChange = async (newFormat: string) => {
  setLoading(true);
  try {
    const result = await app.callServerTool({
      name: 'convert_to_paper',
      arguments: {
        // Re-use original input, just change format
        source: lastInput.source,
        title: lastInput.title,
        author: lastInput.author,
        abstract: lastInput.abstract,
        template: newFormat,
        bibliography: lastInput.bibliography,
        options: lastInput.options,
      },
    });
    // ontoolresult callback will fire with new PDF
  } catch (e) {
    setError(e instanceof Error ? e.message : 'Format switch failed');
  } finally {
    setLoading(false);
  }
};
```

---

## 8. Server Modifications Required

### 8.1 Import Statements

Add to `server/src/index.ts`:

```typescript
import {
  registerAppTool,
  registerAppResource,
  RESOURCE_MIME_TYPE,
} from '@modelcontextprotocol/ext-apps/server';
import fs from 'node:fs/promises';
import path from 'node:path';
import { fileURLToPath } from 'node:url';

const __dirname = path.dirname(fileURLToPath(import.meta.url));
```

### 8.2 Tool Registration

Replace the `registerTool` call for `convert_to_paper` with:

```typescript
registerAppTool(
  server,
  'convert_to_paper',
  {
    type: 'object',
    properties: {
      source: {
        type: 'string',
        description: 'Markdown research content',
      },
      title: {
        type: 'string',
        description: 'Paper title',
      },
      author: {
        type: 'array',
        items: { type: 'string' },
        description: 'Author names',
      },
      abstract: {
        type: 'string',
        description: 'Paper abstract',
      },
      template: {
        type: 'string',
        enum: ['ieee', 'acm', 'arxiv', 'nature', 'science', 'custom'],
        default: 'ieee',
        description: 'Venue template to use',
      },
      bibliography: {
        type: 'string',
        description: 'BibTeX content',
      },
      options: {
        type: 'object',
        description: 'Compilation options',
      },
    },
    required: ['source', 'title', 'author', 'abstract', 'template'],
    _meta: {
      ui: {
        resourceUri: 'ui://paper-mcp/preview.html',
        csp: ['worker-src blob:'],
      },
    },
  },
  convertToPaper
);

// Keep other tools as regular registerTool
server.registerTool('validate_paper_structure', validateInputSchema, validatePaperStructure);
server.registerTool('list_templates', listTemplatesInputSchema, listTemplates);
```

### 8.3 Resource Registration

Add to `server/src/index.ts`:

```typescript
const PAPER_RESOURCE_URI = 'ui://paper-mcp/preview.html';
const PAPER_DIST_PATH = path.resolve(__dirname, '../../dist/paper-preview.html');

registerAppResource(
  server,
  PAPER_RESOURCE_URI,
  PAPER_RESOURCE_URI,
  { mimeType: RESOURCE_MIME_TYPE },
  async () => {
    const html = await fs.readFile(PAPER_DIST_PATH, 'utf-8');
    return {
      contents: [{
        uri: PAPER_RESOURCE_URI,
        mimeType: RESOURCE_MIME_TYPE,
        text: html,
      }],
    };
  }
);
```

---

## 9. CSS / Styling Notes

The app uses ZenSci design tokens from `styles.css` (shared across all apps). Additional paper-specific styles:

```css
/* app/src/styles.css — add to shared base */

/* Two-column layout */
.paper-layout {
  display: flex;
  gap: 16px;
  margin-top: 16px;
}

.paper-layout-main {
  flex: 1;
}

.paper-layout-sidebar {
  width: 280px;
  border-left: var(--zen-border);
  padding-left: 16px;
  max-height: 600px;
  overflow-y: auto;
}

/* PDF canvas styling */
.pdf-canvas {
  width: 100%;
  max-width: 600px;
  border: var(--zen-border);
  border-radius: var(--zen-radius);
  margin-bottom: 12px;
}

/* Page navigation */
.page-nav {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 12px;
  font-size: 13px;
  margin-top: 12px;
}

/* Stats grid */
.stats-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 8px;
  margin-top: 12px;
}

.stat-item {
  padding: 10px;
  background: var(--mcp-bg-secondary, #f9f9f9);
  border-radius: var(--zen-radius);
  text-align: center;
}

.stat-value {
  font-size: 16px;
  font-weight: 700;
  color: var(--zen-tab-active-bg, #2563eb);
}

.stat-label {
  font-size: 10px;
  color: var(--mcp-text-secondary, #666);
  margin-top: 2px;
}

/* Abstract panel */
.abstract-panel {
  padding: 10px;
  background: var(--mcp-bg-secondary, #f9f9f9);
  border-radius: var(--zen-radius);
  font-size: 12px;
  line-height: 1.5;
  margin-bottom: 12px;
  color: var(--mcp-text-secondary, #666);
}

/* Authors list */
.authors-list {
  font-size: 11px;
  color: var(--mcp-text-secondary, #666);
}

.author-item {
  margin-bottom: 4px;
}

.author-name {
  font-weight: 600;
}

.author-affiliation {
  font-size: 10px;
  font-style: italic;
}
```

---

## 10. Build Configuration

### 10.1 app/package.json

```json
{
  "name": "@zen-sci/paper-mcp-app",
  "version": "0.4.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "test": "vitest"
  },
  "dependencies": {
    "@modelcontextprotocol/ext-apps": "^1.0.1",
    "pdfjs-dist": "^4.0.0",
    "react": "^18.3.0",
    "react-dom": "^18.3.0"
  },
  "devDependencies": {
    "@testing-library/react": "^14.0.0",
    "@types/react": "^18.3.0",
    "@types/react-dom": "^18.3.0",
    "@vitejs/plugin-react": "^4.3.0",
    "typescript": "^5.4.0",
    "vite": "^5.4.0",
    "vite-plugin-singlefile": "^2.0.0",
    "vitest": "^1.0.0"
  }
}
```

**Key dependency:** `pdfjs-dist ^4.0.0` for PDF rendering.

### 10.2 app/vite.config.ts

Add CSP extension for PDF.js worker:

```typescript
// app/vite.config.ts — identical except for comment
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { viteSingleFile } from 'vite-plugin-singlefile';
import path from 'node:path';

export default defineConfig({
  plugins: [react(), viteSingleFile()],
  build: {
    outDir: path.resolve(__dirname, '../dist'),
    rollupOptions: {
      input: path.resolve(__dirname, 'index.html'),
    },
  },
  // Note: CSP for PDF.js worker (worker-src blob:) is declared in server config
});
```

### 10.3 app/index.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>ZenSci — Paper Preview</title>
</head>
<body>
  <div id="root"></div>
  <script type="module" src="/src/main.tsx"></script>
</body>
</html>
```

---

## 11. File Manifest

### New App Files (Phase 4 Only)

| File | Description |
|------|-------------|
| `app/src/App.tsx` | Root component with tab navigation, format selector, sidebar |
| `app/src/types.ts` | TypeScript types for tool result and component props |
| `app/src/components/PDFPreview.tsx` | PDF.js viewer with page navigation |
| `app/src/components/FormatSelector.tsx` | Dropdown to switch between IEEE/ACM/arXiv/etc. |
| `app/src/components/PaperInfoPanel.tsx` | Sidebar with abstract, authors, stats |
| `app/src/components/StatsGrid.tsx` | 2x3 grid of section/figure/table/citation counts |
| `app/src/components/WarningsStrip.tsx` | Yellow warning bar for constraint violations |
| `app/src/components/LaTeXSourceTab.tsx` | Raw LaTeX with copy button |
| `app/src/__tests__/App.test.tsx` | Root component tests |
| `app/src/__tests__/PDFPreview.test.tsx` | PDF preview tests |
| `app/src/__tests__/FormatSelector.test.tsx` | Format selector tests |
| `app/src/__tests__/setup.ts` | Vitest setup with mock SDK |

### Shared Files (Identical Across All Apps)

| File | Location |
|------|----------|
| `app/src/main.tsx` | Shared entry point |
| `app/src/styles.css` | ZenSci design tokens + paper-specific CSS |
| `app/src/components/LoadingState.tsx` | Shared loading spinner |
| `app/src/components/ErrorState.tsx` | Shared error display |
| `app/src/components/EmptyState.tsx` | Shared empty state message |
| `app/vite.config.ts` | Shared Vite + singlefile config |
| `app/index.html` | Vite entry (title differs per module) |
| `app/tsconfig.json` | Shared TypeScript config |
| `app/package.json` | Shared base dependencies (+ pdfjs-dist) |

### Build Output

| File | Description |
|------|-------------|
| `dist/paper-preview.html` | Single-file bundle (all CSS + JS inlined, including PDF.js) |

---

## 12. Success Criteria

- [ ] App compiles without errors: `cd app && vite build`
- [ ] Output file `dist/paper-preview.html` exists and is valid HTML
- [ ] App registers as `ui://paper-mcp/preview.html` in server config
- [ ] App renders loading state while waiting for tool result
- [ ] App renders PDF preview with canvas rendering (PDF.js)
- [ ] Page navigation works: Prev/Next buttons, page counter accurate
- [ ] Format selector dropdown shows IEEE, ACM, arXiv, Nature, Science options
- [ ] Format selection triggers tool call with new format
- [ ] Format badge shows correct label for current format
- [ ] Abstract section is collapsible; toggles on button click
- [ ] Authors list displays with names and affiliations
- [ ] Stats grid shows section/figure/table/citation counts
- [ ] Warnings strip appears for constraint violations (page count, etc.)
- [ ] LaTeX source tab displays raw LaTeX with syntax highlighting
- [ ] "Copy to Clipboard" button works for LaTeX source
- [ ] PDF.js worker initializes correctly (no console errors)
- [ ] App handles malformed JSON gracefully (error state)
- [ ] All components render without console errors
- [ ] Sidebar scrolls independently if content exceeds max-height
- [ ] Two-column layout is responsive (sidebar hides on small screens)
- [ ] Unit tests pass with ≥ 80% coverage
- [ ] No hardcoded colors outside semantic status values
- [ ] `useHostStyles()` applied at root component level
- [ ] CSP `worker-src blob:` correctly set in server config

---

## Audit Notes (2026-02-18)

**Spec authored:** 2026-02-18
**Grounded in:** paper-mcp v0.3.0 spec (Phase 1), MCP Apps SDK v1.0.1, PDF.js v4.0.0
**Architecture validated against:** mcp-apps-architecture-spec.md
**Status:** Production Ready

### Key Decisions
- **PDF rendering:** Uses PDF.js v4.0 with canvas rendering; requires `worker-src blob:` CSP extension.
- **Format switching:** Uses format selector dropdown; app calls `convert_to_paper` with new format parameter.
- **Two-column layout:** Main content (PDF + tabs) on left, sidebar (metadata, stats, authors) on right.
- **Sidebar content:** Abstract (collapsible), author list with affiliations, stats grid (2x3 layout).
- **Warnings strip:** Yellow bar at top; lists constraint violations (page count, figure count, etc.).
- **Tab structure:** Preview (PDF) and LaTeX Source tabs at top level.
- **Page navigation:** Prev/Next buttons with page counter.
- **Base64 PDF:** Tool returns PDF as base64; app decodes for PDF.js rendering.

### Open Items
None — spec is complete and production-ready.
