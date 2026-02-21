# latex-mcp MCP App Specification
## Phase 4: Visual Application Layer

**Version:** 0.4.0
**Status:** Draft
**Author:** ZenSci Architecture (Cruz Morales / TresPiesDesign.com)
**Date:** 2026-02-18
**Depends on:** `latex-mcp` v0.1.0, `@modelcontextprotocol/ext-apps` ^1.0.1

---

## 1. Purpose

The **latex-mcp-app** provides a rich visual interface for the `latex-mcp` server's primary conversion tool, `convert_to_pdf`. When a user calls `convert_to_pdf` from Claude, the app receives the tool result (PDF path, base64-encoded PDF, LaTeX source, metadata, warnings, and citation status) and renders it as an interactive preview panel.

**Key interactions:**
- View the rendered PDF in an embedded, navigable viewer (PDF.js)
- Inspect the raw LaTeX source code
- Re-render with different TeX engines (pdflatex, xelatex, lualatex)
- Check citation resolution status and call `check_citations` to refresh
- Review warnings and metadata at a glance

---

## 2. Tool Integration

### 2.1 Primary Tool: `convert_to_pdf`

The `convert_to_pdf` tool is registered with the app's visual layer via `_meta.ui`. Its output is the data source for the entire app.

**Tool registration in server/src/index.ts:**
```typescript
registerAppTool(
  server,
  'convert_to_pdf',
  {
    type: 'object',
    properties: {
      latex_source: { type: 'string', description: 'Raw LaTeX document' },
      options: {
        type: 'object',
        properties: {
          engine: {
            type: 'string',
            enum: ['pdflatex', 'xelatex', 'lualatex'],
            description: 'TeX engine to use',
          },
          // ... other options
        },
      },
    },
    _meta: {
      ui: {
        resourceUri: 'ui://latex-mcp/preview.html',
        csp: ['worker-src blob:'],
      },
    },
  },
  convertToPdf
);
```

**Tool result shape (received via `app.ontoolresult`):**
```json
{
  "pdf_path": "/tmp/output-abc123.pdf",
  "pdf_base64": "JVBERi0xLjQKJeLjz9MNCjEgMCBvYmo=...",
  "latex_source": "\\documentclass{article}\\begin{document}...",
  "page_count": 12,
  "metadata": {
    "title": "My Research Paper",
    "author": ["Alice Chen", "Bob Smith"],
    "date": "2026-02-18"
  },
  "warnings": [
    "Overfull \\hbox (12.34pt too wide) detected in line 45",
    "Citation 'smith2020' not found in .bib file"
  ],
  "citations": {
    "total": 8,
    "resolved": 7,
    "unresolved": ["smith2020"]
  }
}
```

### 2.2 Secondary Tools

Called interactively from the app when the user clicks "Check Citations" or "Re-render":

**`check_citations`**
- Input: `{ latex_source: string; bib_files?: string[] }`
- Output: updated `citations` object (same shape as above)
- Called when user clicks "Check Citations" button
- Updates the metadata panel with fresh citation status

**`convert_to_pdf` (re-render)**
- Input: `{ latex_source: string; options: { engine: string; ... } }`
- Output: full tool result (same shape as primary)
- Called when user clicks "Re-render" button after changing engine or modifying options
- Replaces entire document state with new render

---

## 3. App File Structure

```
servers/latex-mcp/
├── server/
│   ├── src/
│   │   └── index.ts                    ← Server entry (MODIFIED for Phase 4)
│   └── package.json
├── app/                                ← NEW in Phase 4
│   ├── src/
│   │   ├── main.tsx                    ← React app entry (shared template)
│   │   ├── App.tsx                     ← Root component
│   │   ├── styles.css                  ← ZenSci design tokens (shared)
│   │   ├── types.ts                    ← TypeScript interfaces
│   │   ├── components/
│   │   │   ├── LoadingState.tsx        ← Shared across modules
│   │   │   ├── ErrorState.tsx          ← Shared across modules
│   │   │   ├── EmptyState.tsx          ← Shared across modules
│   │   │   ├── PdfViewer.tsx           ← PRIMARY: embedded PDF.js viewer
│   │   │   ├── PdfToolbar.tsx          ← Page nav + zoom controls
│   │   │   ├── LaTeXSource.tsx         ← Raw source with copy button
│   │   │   ├── MetadataPanel.tsx       ← Title, authors, warnings, citations
│   │   │   └── ControlsPanel.tsx       ← Re-render, engine dropdown, check citations
│   │   ├── hooks/
│   │   │   └── usePdfWorker.ts         ← PDF.js worker initialization
│   │   └── utils/
│   │       └── formatCitations.ts      ← Citation status formatting
│   ├── index.html                      ← Vite entry (title: "ZenSci — LaTeX Preview")
│   ├── vite.config.ts                  ← Shared Vite + singlefile config
│   ├── tsconfig.json                   ← Shared TypeScript config
│   └── package.json                    ← App deps (with pdfjs-dist)
├── dist/
│   └── latex-preview.html              ← Built output (self-contained)
└── package.json                        ← Module root
```

---

## 4. Component Architecture

### 4.1 Root App Component

```typescript
// app/src/App.tsx
import { useApp, useHostStyles } from '@modelcontextprotocol/ext-apps/react';
import { useState, useEffect } from 'react';
import type { ConvertToPdfResult } from './types';
import { PdfViewer } from './components/PdfViewer';
import { LaTeXSource } from './components/LaTeXSource';
import { MetadataPanel } from './components/MetadataPanel';
import { ControlsPanel } from './components/ControlsPanel';
import { LoadingState, ErrorState, EmptyState } from './components/shared';

type TabName = 'preview' | 'source';

export default function App() {
  const app = useApp();
  const hostStyles = useHostStyles();
  const [data, setData] = useState<ConvertToPdfResult | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(true);
  const [activeTab, setActiveTab] = useState<TabName>('preview');

  useEffect(() => {
    app.ontoolresult = (result) => {
      try {
        const parsed = JSON.parse(result.content[0].text) as ConvertToPdfResult;
        setData(parsed);
        setError(null);
        setLoading(false);
      } catch (e) {
        setError('Failed to parse tool result');
        setLoading(false);
      }
    };
  }, [app]);

  if (loading) return <LoadingState />;
  if (error) return <ErrorState message={error} />;
  if (!data) return <EmptyState />;

  return (
    <div style={{ ...hostStyles, padding: '16px', fontFamily: 'var(--zen-font-sans)' }}>
      {/* Warnings strip */}
      {data.warnings && data.warnings.length > 0 && (
        <div style={{
          background: '#fffacd',
          border: '1px solid #ffeb3b',
          color: '#333',
          padding: '8px 12px',
          borderRadius: 'var(--zen-radius)',
          marginBottom: '12px',
          fontSize: '12px',
        }}>
          <strong>⚠ {data.warnings.length} warning(s):</strong>
          <ul style={{ margin: '4px 0 0 0', paddingLeft: '20px' }}>
            {data.warnings.map((w, i) => <li key={i}>{w}</li>)}
          </ul>
        </div>
      )}

      {/* Controls */}
      <ControlsPanel data={data} app={app} />

      {/* Tab navigation */}
      <div className="zen-tabs" style={{ marginTop: '12px' }}>
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
          LaTeX
        </button>
      </div>

      {/* Tab content */}
      {activeTab === 'preview' && (
        <div style={{ display: 'flex', flexDirection: 'column', gap: '12px' }}>
          <PdfViewer pdfBase64={data.pdf_base64} pageCount={data.page_count} />
          <MetadataPanel data={data} app={app} />
        </div>
      )}
      {activeTab === 'source' && (
        <LaTeXSource source={data.latex_source} />
      )}
    </div>
  );
}
```

### 4.2 PdfViewer Component

```typescript
// app/src/components/PdfViewer.tsx
import { useEffect, useRef, useState } from 'react';
import * as pdfjs from 'pdfjs-dist';

interface PdfViewerProps {
  pdfBase64: string;
  pageCount: number;
}

export function PdfViewer({ pdfBase64, pageCount }: PdfViewerProps) {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const [currentPage, setCurrentPage] = useState(1);
  const [zoom, setZoom] = useState(100);
  const [pdf, setPdf] = useState<pdfjs.PDFDocument | null>(null);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    pdfjs.GlobalWorkerOptions.workerSrc = `data:application/javascript;base64,${btoa(
      'importScripts("https://cdnjs.cloudflare.com/ajax/libs/pdf.js/4.0.0/pdf.worker.min.js")'
    )}`;

    const loadPdf = async () => {
      try {
        const binaryString = atob(pdfBase64);
        const bytes = new Uint8Array(binaryString.length);
        for (let i = 0; i < binaryString.length; i++) {
          bytes[i] = binaryString.charCodeAt(i);
        }
        const doc = await pdfjs.getDocument({ data: bytes }).promise;
        setPdf(doc);
      } catch (e) {
        setError(e instanceof Error ? e.message : 'Failed to load PDF');
      }
    };

    loadPdf();
  }, [pdfBase64]);

  useEffect(() => {
    if (!pdf || !canvasRef.current) return;

    const renderPage = async () => {
      try {
        const page = await pdf.getPage(currentPage);
        const viewport = page.getViewport({ scale: zoom / 100 });
        const canvas = canvasRef.current;
        if (!canvas) return;

        canvas.width = viewport.width;
        canvas.height = viewport.height;
        const ctx = canvas.getContext('2d');
        if (!ctx) return;

        await page.render({
          canvasContext: ctx,
          viewport,
        }).promise;
      } catch (e) {
        setError(e instanceof Error ? e.message : 'Failed to render page');
      }
    };

    renderPage();
  }, [pdf, currentPage, zoom]);

  if (error) {
    return (
      <div style={{
        padding: '16px',
        background: '#f8d7da',
        color: '#721c24',
        borderRadius: 'var(--zen-radius)',
        fontSize: '13px',
      }}>
        <strong>PDF Error:</strong> {error}
        <p style={{ margin: '8px 0 0 0', fontSize: '12px' }}>
          Falling back to download. PDF path: (see tool result)
        </p>
      </div>
    );
  }

  return (
    <div style={{ display: 'flex', flexDirection: 'column', gap: '12px' }}>
      <PdfToolbar
        currentPage={currentPage}
        pageCount={pageCount}
        zoom={zoom}
        onPageChange={setCurrentPage}
        onZoomChange={setZoom}
      />
      <div style={{
        border: 'var(--zen-border)',
        borderRadius: 'var(--zen-radius)',
        overflow: 'auto',
        maxHeight: '600px',
        display: 'flex',
        justifyContent: 'center',
        background: '#f5f5f5',
      }}>
        <canvas ref={canvasRef} />
      </div>
    </div>
  );
}
```

### 4.3 PdfToolbar Component

```typescript
// app/src/components/PdfToolbar.tsx
interface PdfToolbarProps {
  currentPage: number;
  pageCount: number;
  zoom: number;
  onPageChange: (page: number) => void;
  onZoomChange: (zoom: number) => void;
}

export function PdfToolbar({
  currentPage,
  pageCount,
  zoom,
  onPageChange,
  onZoomChange,
}: PdfToolbarProps) {
  return (
    <div style={{
      display: 'flex',
      alignItems: 'center',
      gap: '12px',
      padding: '8px 12px',
      background: 'var(--mcp-bg-secondary, #f9f9f9)',
      borderRadius: 'var(--zen-radius)',
      border: 'var(--zen-border)',
      fontSize: '13px',
      flexWrap: 'wrap',
    }}>
      {/* Page navigation */}
      <button
        onClick={() => onPageChange(Math.max(1, currentPage - 1))}
        disabled={currentPage === 1}
        style={{
          padding: '4px 8px',
          cursor: currentPage === 1 ? 'default' : 'pointer',
          opacity: currentPage === 1 ? 0.5 : 1,
        }}
      >
        ← Prev
      </button>
      <span style={{ minWidth: '60px', textAlign: 'center', color: 'var(--mcp-text-secondary)' }}>
        {currentPage} / {pageCount}
      </span>
      <button
        onClick={() => onPageChange(Math.min(pageCount, currentPage + 1))}
        disabled={currentPage === pageCount}
        style={{
          padding: '4px 8px',
          cursor: currentPage === pageCount ? 'default' : 'pointer',
          opacity: currentPage === pageCount ? 0.5 : 1,
        }}
      >
        Next →
      </button>

      <div style={{ borderLeft: 'var(--zen-border)', height: '20px' }} />

      {/* Zoom controls */}
      <button onClick={() => onZoomChange(Math.max(50, zoom - 10))} style={{ padding: '4px 8px' }}>
        −
      </button>
      <input
        type="text"
        value={`${zoom}%`}
        onChange={(e) => {
          const num = parseInt(e.target.value) || zoom;
          onZoomChange(Math.min(300, Math.max(50, num)));
        }}
        style={{ width: '50px', textAlign: 'center', padding: '2px 4px' }}
      />
      <button onClick={() => onZoomChange(Math.min(300, zoom + 10))} style={{ padding: '4px 8px' }}>
        +
      </button>
      <button onClick={() => onZoomChange(100)} style={{ padding: '4px 8px', fontSize: '11px' }}>
        100%
      </button>
      <button onClick={() => onZoomChange(0)} style={{ padding: '4px 8px', fontSize: '11px' }} title="Fit to width">
        Fit
      </button>
    </div>
  );
}
```

### 4.4 LaTeXSource Component

```typescript
// app/src/components/LaTeXSource.tsx
import { useState } from 'react';

interface LaTeXSourceProps {
  source: string;
}

export function LaTeXSource({ source }: LaTeXSourceProps) {
  const [copied, setCopied] = useState(false);

  const handleCopy = () => {
    navigator.clipboard.writeText(source).then(() => {
      setCopied(true);
      setTimeout(() => setCopied(false), 2000);
    });
  };

  return (
    <div style={{ display: 'flex', flexDirection: 'column', gap: '8px' }}>
      <button
        onClick={handleCopy}
        style={{
          alignSelf: 'flex-end',
          padding: '6px 12px',
          background: 'var(--mcp-accent, #2563eb)',
          color: '#fff',
          border: 'none',
          borderRadius: 'var(--zen-radius)',
          cursor: 'pointer',
          fontSize: '12px',
        }}
      >
        {copied ? '✓ Copied' : 'Copy'}
      </button>
      <pre
        style={{
          background: 'var(--mcp-bg-secondary, #f5f5f5)',
          padding: '12px',
          borderRadius: 'var(--zen-radius)',
          border: 'var(--zen-border)',
          overflow: 'auto',
          maxHeight: '600px',
          fontFamily: 'var(--zen-font-mono)',
          fontSize: '11px',
          lineHeight: '1.5',
          color: 'var(--mcp-text-primary, #000)',
        }}
      >
        {source}
      </pre>
    </div>
  );
}
```

### 4.5 MetadataPanel Component

```typescript
// app/src/components/MetadataPanel.tsx
import type { ConvertToPdfResult } from '../types';

interface MetadataPanelProps {
  data: ConvertToPdfResult;
  app: any; // useApp() return type
}

export function MetadataPanel({ data }: MetadataPanelProps) {
  const warningCount = data.warnings?.length ?? 0;
  const citationResolvedPercent =
    data.citations.total > 0
      ? Math.round((data.citations.resolved / data.citations.total) * 100)
      : 100;

  return (
    <div style={{
      display: 'grid',
      gridTemplateColumns: '1fr 1fr',
      gap: '16px',
      padding: '12px',
      background: 'var(--mcp-bg-secondary, #f9f9f9)',
      borderRadius: 'var(--zen-radius)',
      border: 'var(--zen-border)',
    }}>
      {/* Left column: basic metadata */}
      <div>
        <h3 style={{ margin: '0 0 8px 0', fontSize: '13px', fontWeight: 600 }}>
          Document
        </h3>
        <table className="zen-meta-table">
          <tbody>
            {data.metadata.title && (
              <tr>
                <td>Title</td>
                <td style={{ fontWeight: 500 }}>{data.metadata.title}</td>
              </tr>
            )}
            {data.metadata.author && data.metadata.author.length > 0 && (
              <tr>
                <td>Author(s)</td>
                <td>{data.metadata.author.join(', ')}</td>
              </tr>
            )}
            {data.metadata.date && (
              <tr>
                <td>Date</td>
                <td>{data.metadata.date}</td>
              </tr>
            )}
            <tr>
              <td>Pages</td>
              <td>{data.page_count}</td>
            </tr>
          </tbody>
        </table>
      </div>

      {/* Right column: quality/status indicators */}
      <div>
        <h3 style={{ margin: '0 0 8px 0', fontSize: '13px', fontWeight: 600 }}>
          Status
        </h3>
        <div style={{ display: 'flex', flexDirection: 'column', gap: '8px', fontSize: '12px' }}>
          <div style={{ display: 'flex', alignItems: 'center', gap: '8px' }}>
            <span>Warnings</span>
            <span
              className={`zen-badge ${warningCount > 0 ? 'warn' : 'pass'}`}
              style={{ marginLeft: 'auto' }}
            >
              {warningCount}
            </span>
          </div>
          <div style={{ display: 'flex', alignItems: 'center', gap: '8px' }}>
            <span>Citations</span>
            <span
              className={`zen-badge ${citationResolvedPercent === 100 ? 'pass' : 'warn'}`}
              style={{ marginLeft: 'auto' }}
            >
              {data.citations.resolved}/{data.citations.total}
            </span>
          </div>
          {data.citations.unresolved && data.citations.unresolved.length > 0 && (
            <div style={{ fontSize: '11px', color: 'var(--mcp-text-secondary)', marginTop: '4px' }}>
              <strong>Unresolved:</strong> {data.citations.unresolved.join(', ')}
            </div>
          )}
        </div>
      </div>
    </div>
  );
}
```

### 4.6 ControlsPanel Component

```typescript
// app/src/components/ControlsPanel.tsx
import { useState } from 'react';
import type { ConvertToPdfResult } from '../types';

interface ControlsPanelProps {
  data: ConvertToPdfResult;
  app: any; // useApp() return type
}

export function ControlsPanel({ data, app }: ControlsPanelProps) {
  const [engine, setEngine] = useState<'pdflatex' | 'xelatex' | 'lualatex'>('pdflatex');
  const [loading, setLoading] = useState(false);

  const handleReRender = async () => {
    setLoading(true);
    try {
      const result = await app.callServerTool({
        name: 'convert_to_pdf',
        arguments: {
          latex_source: data.latex_source,
          options: { engine },
        },
      });
      if (result.isError) {
        console.error('Re-render failed:', result.content[0].text);
      }
    } catch (e) {
      console.error('Tool call error:', e);
    } finally {
      setLoading(false);
    }
  };

  const handleCheckCitations = async () => {
    setLoading(true);
    try {
      const result = await app.callServerTool({
        name: 'check_citations',
        arguments: {
          latex_source: data.latex_source,
        },
      });
      if (result.isError) {
        console.error('Citation check failed:', result.content[0].text);
      }
    } catch (e) {
      console.error('Tool call error:', e);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div style={{
      display: 'flex',
      gap: '8px',
      alignItems: 'center',
      padding: '8px 0',
      flexWrap: 'wrap',
    }}>
      <label style={{ fontSize: '12px' }}>
        TeX Engine:
        <select
          value={engine}
          onChange={(e) => setEngine(e.target.value as any)}
          disabled={loading}
          style={{ marginLeft: '6px', padding: '4px' }}
        >
          <option value="pdflatex">pdflatex</option>
          <option value="xelatex">xelatex</option>
          <option value="lualatex">lualatex</option>
        </select>
      </label>

      <button
        onClick={handleReRender}
        disabled={loading}
        style={{
          padding: '6px 12px',
          background: 'var(--mcp-accent, #2563eb)',
          color: '#fff',
          border: 'none',
          borderRadius: 'var(--zen-radius)',
          cursor: loading ? 'default' : 'pointer',
          opacity: loading ? 0.6 : 1,
          fontSize: '12px',
        }}
      >
        {loading ? 'Processing…' : 'Re-render'}
      </button>

      <button
        onClick={handleCheckCitations}
        disabled={loading}
        style={{
          padding: '6px 12px',
          background: 'var(--mcp-accent, #2563eb)',
          color: '#fff',
          border: 'none',
          borderRadius: 'var(--zen-radius)',
          cursor: loading ? 'default' : 'pointer',
          opacity: loading ? 0.6 : 1,
          fontSize: '12px',
        }}
      >
        Check Citations
      </button>
    </div>
  );
}
```

---

## 5. Data Types

```typescript
// app/src/types.ts
export interface ConvertToPdfResult {
  pdf_path: string;
  pdf_base64: string;
  latex_source: string;
  page_count: number;
  metadata: {
    title?: string;
    author?: string[];
    date?: string;
  };
  warnings?: string[];
  citations: {
    total: number;
    resolved: number;
    unresolved?: string[];
  };
}

export interface Citation {
  key: string;
  resolved: boolean;
}

export interface PdfMetadata {
  title?: string;
  author?: string[];
  date?: string;
}

export type TexEngine = 'pdflatex' | 'xelatex' | 'lualatex';

export interface RenderOptions {
  engine: TexEngine;
  format?: 'pdf' | 'ps' | 'dvi';
}
```

---

## 6. Tool Result Shape

The app receives the `convert_to_pdf` tool result via `app.ontoolresult`. The host serializes it as JSON text in `result.content[0].text`. The App component parses this and passes it to sub-components.

**Shape (TypeScript interface):**
```typescript
{
  pdf_path: string;               // File path on server (for download fallback)
  pdf_base64: string;             // Base64-encoded PDF binary
  latex_source: string;           // Original LaTeX string
  page_count: number;             // Total pages in PDF
  metadata: {
    title?: string;
    author?: string[];
    date?: string;
  };
  warnings?: string[];            // LaTeX warnings (Overfull, citations, etc.)
  citations: {
    total: number;                // Total citations in document
    resolved: number;             // Number resolved from .bib
    unresolved?: string[];        // Array of citation keys not found
  };
}
```

---

## 7. Interactive Tool Calls

The app makes two types of interactive tool calls (from ControlsPanel and MetadataPanel):

### 7.1 Re-render with Different Engine

**From:** ControlsPanel "Re-render" button
**Calls:** `convert_to_pdf` again
**Payload:**
```json
{
  "name": "convert_to_pdf",
  "arguments": {
    "latex_source": "\\documentclass{article}...",
    "options": {
      "engine": "xelatex"
    }
  }
}
```

**Handler:**
```typescript
const result = await app.callServerTool({
  name: 'convert_to_pdf',
  arguments: {
    latex_source: data.latex_source,
    options: { engine },
  },
});
```

### 7.2 Check Citations

**From:** MetadataPanel "Check Citations" button
**Calls:** `check_citations`
**Payload:**
```json
{
  "name": "check_citations",
  "arguments": {
    "latex_source": "\\documentclass{article}..."
  }
}
```

**Handler:**
```typescript
const result = await app.callServerTool({
  name: 'check_citations',
  arguments: {
    latex_source: data.latex_source,
  },
});
```

Both return updated results that the app must parse and merge back into its state (triggering a re-render).

---

## 8. Server Modifications Required

### 8.1 Imports in `server/src/index.ts`

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

Replace the existing `server.registerTool('convert_to_pdf', ...)` call with:

```typescript
registerAppTool(
  server,
  'convert_to_pdf',
  {
    type: 'object',
    properties: {
      latex_source: {
        type: 'string',
        description: 'Raw LaTeX document content',
      },
      options: {
        type: 'object',
        properties: {
          engine: {
            type: 'string',
            enum: ['pdflatex', 'xelatex', 'lualatex'],
            description: 'TeX rendering engine',
          },
          format: {
            type: 'string',
            enum: ['pdf', 'ps', 'dvi'],
            description: 'Output format',
          },
        },
        required: ['engine'],
      },
    },
    required: ['latex_source', 'options'],
    _meta: {
      ui: {
        resourceUri: 'ui://latex-mcp/preview.html',
        csp: ['worker-src blob:'],
      },
    },
  },
  convertToPdf
);
```

### 8.3 Secondary Tools

Keep as standard `server.registerTool()`:

```typescript
server.registerTool(
  'check_citations',
  {
    type: 'object',
    properties: {
      latex_source: {
        type: 'string',
        description: 'LaTeX document content',
      },
      bib_files: {
        type: 'array',
        items: { type: 'string' },
        description: 'Optional .bib file paths',
      },
    },
    required: ['latex_source'],
  },
  checkCitations
);

server.registerTool(
  'validate_document',
  {
    type: 'object',
    properties: {
      latex_source: {
        type: 'string',
        description: 'LaTeX document to validate',
      },
    },
    required: ['latex_source'],
  },
  validateDocument
);
```

### 8.4 Resource Registration

Add this after all tool registrations:

```typescript
const RESOURCE_URI = 'ui://latex-mcp/preview.html';
const DIST_PATH = path.resolve(__dirname, '../../dist/latex-preview.html');

registerAppResource(
  server,
  RESOURCE_URI,
  RESOURCE_URI,
  { mimeType: RESOURCE_MIME_TYPE },
  async () => {
    const html = await fs.readFile(DIST_PATH, 'utf-8');
    return {
      contents: [{
        uri: RESOURCE_URI,
        mimeType: RESOURCE_MIME_TYPE,
        text: html,
      }],
    };
  }
);
```

---

## 9. CSS / Styling Notes

**Shared styles** (from `mcp-apps-architecture-spec.md § 7) are in `app/src/styles.css`. The app additionally uses:

- **PDF.js canvas rendering:** Uses native canvas styling; border and shadow applied at container level
- **Page navigation buttons:** Simple inline button styling; disabled state via opacity
- **Zoom input:** `<input type="text">` for manual zoom entry; uses monospace font for numbers
- **Metadata table:** Uses `.zen-meta-table` class for consistent styling
- **Warning strip:** Yellow background (`#fffacd`), gold border (`#ffeb3b`), black text for contrast
- **Badges:** Reuses `.zen-badge.pass`, `.zen-badge.warn` from shared styles

**No hardcoded colors** except semantic status (pass/warn/fail). All inherit from host via `useHostStyles()`.

---

## 10. Build Configuration

### 10.1 `app/package.json`

```json
{
  "name": "@zen-sci/latex-mcp-app",
  "version": "0.4.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "@modelcontextprotocol/ext-apps": "^1.0.1",
    "pdfjs-dist": "^4.0.0",
    "react": "^18.3.0",
    "react-dom": "^18.3.0"
  },
  "devDependencies": {
    "@types/react": "^18.3.0",
    "@types/react-dom": "^18.3.0",
    "@vitejs/plugin-react": "^4.3.0",
    "typescript": "^5.4.0",
    "vite": "^5.4.0",
    "vite-plugin-singlefile": "^2.0.0"
  }
}
```

**Key difference:** Includes `pdfjs-dist` (v4.0.0+) for PDF.js rendering.

### 10.2 `vite.config.ts`

Use the identical shared template (from mcp-apps-architecture-spec.md § 4):

```typescript
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
});
```

### 10.3 Root `package.json` Script

Add to `servers/latex-mcp/package.json`:

```json
{
  "scripts": {
    "build:app": "cd app && vite build",
    "build": "tsc && cd app && vite build"
  }
}
```

---

## 11. File Manifest

All new files in `app/` directory:

| File | Description |
|------|-------------|
| `app/package.json` | App dependencies (includes pdfjs-dist) |
| `app/index.html` | Vite entry (title: "ZenSci — LaTeX Preview") |
| `app/vite.config.ts` | Vite + singlefile config (shared template) |
| `app/tsconfig.json` | TypeScript config (shared template) |
| `app/src/main.tsx` | React entry point (shared template) |
| `app/src/styles.css` | ZenSci design tokens (shared) |
| `app/src/App.tsx` | Root component |
| `app/src/types.ts` | TypeScript interfaces (ConvertToPdfResult, etc.) |
| `app/src/components/LoadingState.tsx` | Loading UI (shared) |
| `app/src/components/ErrorState.tsx` | Error UI (shared) |
| `app/src/components/EmptyState.tsx` | Empty state UI (shared) |
| `app/src/components/PdfViewer.tsx` | Main PDF rendering component (PDF.js) |
| `app/src/components/PdfToolbar.tsx` | Page nav + zoom controls |
| `app/src/components/LaTeXSource.tsx` | Raw LaTeX with copy button |
| `app/src/components/MetadataPanel.tsx` | Title, authors, status badges |
| `app/src/components/ControlsPanel.tsx` | Engine dropdown, re-render, check citations |
| `app/src/hooks/usePdfWorker.ts` | PDF.js worker initialization (optional) |
| `app/src/utils/formatCitations.ts` | Citation status formatting (optional) |
| `dist/latex-preview.html` | Built output (generated by `npm run build:app`) |

---

## 12. Success Criteria

- [ ] `pnpm run build:app` from `servers/latex-mcp/` builds without errors
- [ ] `dist/latex-preview.html` is a single self-contained HTML file (no external refs except CDN for PDF.js worker)
- [ ] Server registers `convert_to_pdf` with `registerAppTool` and `_meta.ui.resourceUri = 'ui://latex-mcp/preview.html'`
- [ ] Server registers app resource via `registerAppResource`
- [ ] App renders PDF preview when `convert_to_pdf` is called
- [ ] Page navigation works (prev/next buttons, page indicator updates)
- [ ] Zoom controls work (zoom in/out, custom %, fit-to-width)
- [ ] "Re-render" button calls `convert_to_pdf` with selected engine
- [ ] "Check Citations" button calls `check_citations` and updates metadata panel
- [ ] LaTeX tab shows raw source with working copy button
- [ ] Warnings strip displays yellow when warnings are present
- [ ] Metadata panel shows title, authors, citation status with badges
- [ ] App handles PDF rendering errors gracefully (error message + fallback info)
- [ ] No hardcoded colors outside semantic status badges
- [ ] `useHostStyles()` applied at root

---

## Audit Notes (2026-02-18)

**Spec authored:** 2026-02-18
**Author:** ZenSci Architecture (Cruz Morales)
**Grounded in:** `mcp-apps-architecture-spec.md` (Phase 4 reference architecture), `pdfjs-dist` v4.0.0 documentation
**Status:** Draft
**Key decisions:**
- PDF.js for browser-side rendering (no server round-trip for page views)
- PDF.js worker loaded via data URL to stay within CSP + sandbox
- Engine dropdown as a simple re-render control (stateless per tool call)
- Citation checking as secondary tool call (updates metadata panel)
- Single preview + source tabs (simple two-view design)
