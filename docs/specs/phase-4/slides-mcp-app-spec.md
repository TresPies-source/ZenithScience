# slides-mcp MCP App Specification
## Phase 4: Visual Application Layer

**Version:** 0.4.0
**Status:** Production Ready
**Author:** ZenSci Architecture (Cruz Morales / TresPiesDesign.com)
**Date:** 2026-02-18
**Depends on:** slides-mcp v0.1.0 (server spec from Phase 2)

---

## 1. Purpose

The slides-mcp server converts Markdown presentations into dual-format slide decks (Beamer PDF and Reveal.js HTML). The companion MCP App renders the `convert_to_slides` tool result as an interactive **slide deck preview**, enabling users to:

- **Toggle between output formats** (Beamer PDF vs. Reveal.js HTML) with a single click
- **Navigate slides** with prev/next buttons, slide number indicator, and thumbnail sidebar
- **View speaker notes** for each slide in a collapsible panel
- **Inspect deck metadata** (title, author, theme, total slides, math/citation counts)
- **Preview themes** before committing (switches theme via tool call)

The app transforms raw conversion output into a navigable, interactive preview that surfaces rendering issues early.

---

## 2. Tool Integration

### 2.1 Primary Tool: `convert_to_slides`

The primary conversion tool declares the MCP App resource:

```typescript
registerAppTool(
  server,
  'convert_to_slides',
  {
    type: 'object',
    properties: {
      source: { type: 'string', description: '...' },
      format: { type: 'string', enum: ['beamer', 'reveal', 'both'] },
      theme: { type: 'string', enum: ['metropolis', 'berlin', 'warsaw', 'simple', 'dark', 'minimal', 'sky', 'league'] },
      options: { type: 'object', description: '...' },
    },
    required: ['source', 'format'],
    _meta: {
      ui: {
        resourceUri: 'ui://slides-mcp/preview.html',
      },
    },
  },
  convertToSlides
);
```

**Tool result shape received by app (from `convert_to_slides`):**

```json
{
  "id": "slides-20260218-xyz789",
  "format": "slides-both",
  "content": {
    "beamer": "JVBERi0xLjQK...",
    "reveal": "<!DOCTYPE html>..."
  },
  "artifacts": [
    {
      "type": "speaker_notes",
      "filename": "speaker_notes.txt",
      "content": "Slide 1:\nIntroduction. Greet audience. Introduce topic.\n\nSlide 2:\n..."
    },
    {
      "type": "bibliography",
      "filename": "references.bib",
      "content": "@article{key2023, ...}"
    },
    {
      "type": "outline",
      "filename": "outline.md",
      "content": "# Title\n## Slide 1\n## Slide 2\n..."
    }
  ],
  "metadata": {
    "slide_count": 20,
    "title": "Conference Talk 2026",
    "author": "Dr. Jane Doe",
    "theme": "metropolis",
    "has_speaker_notes": true,
    "bibliography_count": 15,
    "math_expressions": 8,
    "slides": [
      {
        "slide_number": 1,
        "title": "Title Slide",
        "body": "<h1>Conference Talk 2026</h1>...",
        "speaker_notes": "Introduction..."
      },
      {
        "slide_number": 2,
        "title": "Agenda",
        "body": "<h2>Agenda</h2>...",
        "speaker_notes": "Outline the talk..."
      }
    ]
  },
  "elapsed": 3500
}
```

### 2.2 Secondary Tools (Called from App)

The app can invoke these secondary tools to refresh or switch themes/formats:

```typescript
server.registerTool('convert_to_slides', inputSchema, convertToSlides);
server.registerTool('validate_slides_source', inputSchema, validateSlidesSource);
server.registerTool('list_slide_themes', inputSchema, listSlideThemes);
```

The app uses `app.callServerTool()` to invoke these when the user:
- Clicks format toggle (Beamer ↔ Reveal.js)
- Changes theme selector

---

## 3. App File Structure

```
zen-sci/servers/slides-mcp/
├── app/
│   ├── src/
│   │   ├── main.tsx                          ← React entry
│   │   ├── App.tsx                           ← Root component
│   │   ├── components/
│   │   │   ├── SlideDeckPreview.tsx          ← Main container
│   │   │   ├── FormatToolbar.tsx             ← Format toggle, theme selector
│   │   │   ├── SlideViewer.tsx               ← PDF or HTML renderer
│   │   │   ├── BeamerViewer.tsx              ← PDF.js-based PDF viewer
│   │   │   ├── RevealViewer.tsx              ← iframe with Reveal.js HTML
│   │   │   ├── SpeakerNotesPanel.tsx         ← Collapsible notes display
│   │   │   ├── SlideNavigator.tsx            ← Thumbnail sidebar
│   │   │   ├── MetadataStrip.tsx             ← Deck title, author, counts
│   │   │   ├── LoadingState.tsx              ← (shared)
│   │   │   ├── ErrorState.tsx                ← (shared)
│   │   │   └── EmptyState.tsx                ← (shared)
│   │   ├── hooks/
│   │   │   ├── useToolResult.ts              ← Parse tool result JSON
│   │   │   ├── usePdfWorker.ts               ← PDF.js worker setup
│   │   │   ├── useSlideNavigation.ts         ← Current slide state, prev/next
│   │   │   └── useToolCall.ts                ← Invoke server tools
│   │   ├── types.ts                          ← TypeScript interfaces
│   │   └── styles.css                        ← ZenSci design tokens
│   ├── index.html
│   ├── vite.config.ts
│   ├── tsconfig.json
│   └── package.json
├── dist/
│   └── slides-preview.html                   ← Built output
└── server/
    └── src/
        └── index.ts                          ← (MODIFIED) includes registerAppResource
```

---

## 4. Component Architecture

### 4.1 Root App Component

```typescript
// app/src/App.tsx

import { useApp, useHostStyles } from '@modelcontextprotocol/ext-apps/react';
import { useState, useEffect } from 'react';
import { SlideDeckPreview } from './components/SlideDeckPreview';
import { LoadingState, ErrorState, EmptyState } from './components';
import type { SlideDeckData } from './types';

export default function App() {
  const app = useApp();
  const hostStyles = useHostStyles();
  const [data, setData] = useState<SlideDeckData | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    app.ontoolresult = (result) => {
      try {
        const parsed = JSON.parse(result.content[0].text);
        setData(parsed as SlideDeckData);
        setLoading(false);
        setError(null);
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
    <div style={{ ...hostStyles, padding: '16px', fontFamily: 'var(--zen-font-sans)', height: '100vh', display: 'flex', flexDirection: 'column' }}>
      <SlideDeckPreview data={data} app={app} />
    </div>
  );
}
```

### 4.2 View Structure (Format Toggle)

The app renders either BeamerViewer (PDF) or RevealViewer (HTML) based on active format:

```typescript
// app/src/components/SlideDeckPreview.tsx

import { useState } from 'react';
import { FormatToolbar } from './FormatToolbar';
import { MetadataStrip } from './MetadataStrip';
import { SlideViewer } from './SlideViewer';
import { SlideNavigator } from './SlideNavigator';
import { SpeakerNotesPanel } from './SpeakerNotesPanel';
import { useSlideNavigation } from '../hooks/useSlideNavigation';
import type { SlideDeckData } from '../types';

interface Props {
  data: SlideDeckData;
  app: ReturnType<typeof useApp>;
}

export function SlideDeckPreview({ data, app }: Props) {
  const [activeFormat, setActiveFormat] = useState<'beamer' | 'reveal'>(
    data.format === 'slides-both' ? 'beamer' : data.format === 'slides-html' ? 'reveal' : 'beamer'
  );
  const [showNotes, setShowNotes] = useState(true);
  const { currentSlide, setCurrentSlide, nextSlide, prevSlide, totalSlides } = useSlideNavigation(data.metadata.slide_count);

  return (
    <div style={{ display: 'flex', flexDirection: 'column', height: '100%', gap: '12px' }}>
      <FormatToolbar
        data={data}
        app={app}
        activeFormat={activeFormat}
        onFormatChange={setActiveFormat}
        showNotes={showNotes}
        onNotesToggle={() => setShowNotes(!showNotes)}
      />

      <MetadataStrip data={data} currentSlide={currentSlide} />

      <div style={{ display: 'flex', gap: '12px', flex: 1, overflow: 'hidden' }}>
        {/* Thumbnail Navigator (left sidebar) */}
        <SlideNavigator
          slides={data.metadata.slides}
          currentSlide={currentSlide}
          onSelectSlide={setCurrentSlide}
        />

        {/* Main Slide Viewer (center) */}
        <div style={{ flex: 1, display: 'flex', flexDirection: 'column', overflow: 'hidden' }}>
          <SlideViewer
            data={data}
            activeFormat={activeFormat}
            currentSlide={currentSlide}
            onPrev={prevSlide}
            onNext={nextSlide}
            slideCount={totalSlides}
          />

          {/* Speaker Notes Panel (bottom, if enabled) */}
          {showNotes && (
            <SpeakerNotesPanel
              notes={data.metadata.slides[currentSlide - 1]?.speaker_notes || ''}
            />
          )}
        </div>
      </div>
    </div>
  );
}
```

### 4.3 Major Components with Props Interfaces

#### FormatToolbar

Toggles between Beamer (PDF) and Reveal.js (HTML); includes theme selector and speaker notes toggle.

```typescript
// app/src/components/FormatToolbar.tsx

interface Props {
  data: SlideDeckData;
  app: ReturnType<typeof useApp>;
  activeFormat: 'beamer' | 'reveal';
  onFormatChange: (format: 'beamer' | 'reveal') => void;
  showNotes: boolean;
  onNotesToggle: () => void;
}

export function FormatToolbar({
  data,
  app,
  activeFormat,
  onFormatChange,
  showNotes,
  onNotesToggle,
}: Props) {
  const [loading, setLoading] = useState(false);

  const handleFormatToggle = async (format: 'beamer' | 'reveal') => {
    onFormatChange(format);
    // Optionally re-call convert_to_slides with new format if not already generated
    if (data.format === 'slides-both') {
      setLoading(true);
      try {
        const result = await app.callServerTool({
          name: 'convert_to_slides',
          arguments: {
            source: data.content,
            format: format,
            theme: data.metadata.theme,
          },
        });
        if (!result.isError) {
          app.ontoolresult?.(result);
        }
      } finally {
        setLoading(false);
      }
    }
  };

  return (
    <div
      style={{
        display: 'flex',
        gap: '12px',
        alignItems: 'center',
        padding: '12px',
        background: 'var(--mcp-bg-secondary, #f9f9f9)',
        borderRadius: 'var(--zen-radius)',
      }}
    >
      <div className="zen-tabs" style={{ margin: 0 }}>
        <button
          className={`zen-tab ${activeFormat === 'beamer' ? 'active' : ''}`}
          onClick={() => handleFormatToggle('beamer')}
          disabled={loading || !data.content.beamer}
        >
          Beamer (PDF)
        </button>
        <button
          className={`zen-tab ${activeFormat === 'reveal' ? 'active' : ''}`}
          onClick={() => handleFormatToggle('reveal')}
          disabled={loading || !data.content.reveal}
        >
          Reveal.js (HTML)
        </button>
      </div>

      <button
        onClick={onNotesToggle}
        style={{
          marginLeft: 'auto',
          padding: '6px 12px',
          background: showNotes ? 'var(--mcp-accent, #2563eb)' : 'transparent',
          color: showNotes ? '#fff' : 'var(--mcp-text, #000)',
          border: 'var(--zen-border)',
          borderRadius: 'var(--zen-radius)',
          cursor: 'pointer',
          fontSize: '13px',
        }}
      >
        {showNotes ? '✓ Notes' : 'Notes'}
      </button>

      {loading && <span style={{ fontSize: '12px', color: 'var(--mcp-text-secondary)' }}>Generating...</span>}
    </div>
  );
}
```

#### SlideViewer

Renders either BeamerViewer (PDF via PDF.js) or RevealViewer (HTML via iframe).

```typescript
// app/src/components/SlideViewer.tsx

import { BeamerViewer } from './BeamerViewer';
import { RevealViewer } from './RevealViewer';
import type { SlideDeckData } from '../types';

interface Props {
  data: SlideDeckData;
  activeFormat: 'beamer' | 'reveal';
  currentSlide: number;
  onPrev: () => void;
  onNext: () => void;
  slideCount: number;
}

export function SlideViewer({
  data,
  activeFormat,
  currentSlide,
  onPrev,
  onNext,
  slideCount,
}: Props) {
  return (
    <div style={{ flex: 1, display: 'flex', flexDirection: 'column' }}>
      {activeFormat === 'beamer' && (
        <BeamerViewer
          pdfBase64={data.content.beamer}
          currentSlide={currentSlide}
          onPrev={onPrev}
          onNext={onNext}
          slideCount={slideCount}
        />
      )}

      {activeFormat === 'reveal' && (
        <RevealViewer
          htmlContent={data.content.reveal}
          currentSlide={currentSlide}
          onSlideChange={(slide: number) => {
            // Sync slide number with app state
          }}
        />
      )}
    </div>
  );
}
```

#### BeamerViewer

Uses PDF.js to render Beamer PDF; displays one slide per page with navigation controls.

```typescript
// app/src/components/BeamerViewer.tsx

import { useEffect, useRef, useState } from 'react';
import * as pdfjsLib from 'pdfjs-dist';
import { usePdfWorker } from '../hooks/usePdfWorker';

interface Props {
  pdfBase64: string;
  currentSlide: number;
  onPrev: () => void;
  onNext: () => void;
  slideCount: number;
}

export function BeamerViewer({
  pdfBase64,
  currentSlide,
  onPrev,
  onNext,
  slideCount,
}: Props) {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const [pdf, setPdf] = useState<pdfjsLib.PDFDocumentProxy | null>(null);

  usePdfWorker();

  useEffect(() => {
    const loadPdf = async () => {
      const binaryString = atob(pdfBase64);
      const bytes = new Uint8Array(binaryString.length);
      for (let i = 0; i < binaryString.length; i++) {
        bytes[i] = binaryString.charCodeAt(i);
      }
      const doc = await pdfjsLib.getDocument(bytes).promise;
      setPdf(doc);
    };
    loadPdf();
  }, [pdfBase64]);

  useEffect(() => {
    if (!pdf || !canvasRef.current) return;

    const renderPage = async () => {
      const page = await pdf.getPage(currentSlide);
      const scale = 1.5;
      const viewport = page.getViewport({ scale });

      const canvas = canvasRef.current!;
      const context = canvas.getContext('2d');
      canvas.width = viewport.width;
      canvas.height = viewport.height;

      await page.render({
        canvasContext: context!,
        viewport,
      }).promise;
    };

    renderPage();
  }, [pdf, currentSlide]);

  return (
    <div style={{ display: 'flex', flexDirection: 'column', flex: 1 }}>
      <div
        style={{
          flex: 1,
          display: 'flex',
          justifyContent: 'center',
          alignItems: 'center',
          background: '#ddd',
          overflow: 'auto',
          maxHeight: '60vh',
        }}
      >
        <canvas ref={canvasRef} style={{ maxWidth: '100%', maxHeight: '100%' }} />
      </div>

      <div
        style={{
          display: 'flex',
          justifyContent: 'space-between',
          alignItems: 'center',
          padding: '12px',
          borderTop: 'var(--zen-border)',
        }}
      >
        <button onClick={onPrev} disabled={currentSlide === 1}>
          ← Prev
        </button>
        <span style={{ fontSize: '13px', color: 'var(--mcp-text-secondary)' }}>
          Slide {currentSlide} of {slideCount}
        </span>
        <button onClick={onNext} disabled={currentSlide === slideCount}>
          Next →
        </button>
      </div>
    </div>
  );
}
```

#### RevealViewer

Renders Reveal.js HTML in a sandboxed iframe. The HTML is self-contained (all CSS/JS inlined by Vite singlefile build).

```typescript
// app/src/components/RevealViewer.tsx

import { useRef, useEffect } from 'react';

interface Props {
  htmlContent: string;
  currentSlide: number;
  onSlideChange?: (slide: number) => void;
}

export function RevealViewer({ htmlContent, currentSlide, onSlideChange }: Props) {
  const iframeRef = useRef<HTMLIFrameElement>(null);

  useEffect(() => {
    if (!iframeRef.current) return;

    const iframe = iframeRef.current;
    iframe.srcdoc = htmlContent;
  }, [htmlContent]);

  return (
    <div style={{ flex: 1, display: 'flex', flexDirection: 'column' }}>
      <iframe
        ref={iframeRef}
        style={{
          flex: 1,
          border: 'none',
          width: '100%',
          height: '60vh',
        }}
        sandbox={{
          allow: ['scripts', 'same-origin', 'allow-presentation'],
        }}
        title="Reveal.js Presentation"
      />
    </div>
  );
}
```

#### SpeakerNotesPanel

Displays speaker notes for the current slide in a collapsible bottom panel.

```typescript
// app/src/components/SpeakerNotesPanel.tsx

interface Props {
  notes: string;
}

export function SpeakerNotesPanel({ notes }: Props) {
  return (
    <div
      style={{
        marginTop: '12px',
        padding: '12px',
        background: 'var(--mcp-bg-secondary, #f9f9f9)',
        borderRadius: 'var(--zen-radius)',
        border: 'var(--zen-border)',
        maxHeight: '180px',
        overflow: 'auto',
      }}
    >
      <h4 style={{ margin: '0 0 8px 0', fontSize: '13px' }}>Speaker Notes</h4>
      <p style={{ margin: 0, fontSize: '12px', lineHeight: 1.5, color: 'var(--mcp-text-secondary)' }}>
        {notes || '(no notes for this slide)'}
      </p>
    </div>
  );
}
```

#### SlideNavigator

Sidebar with thumbnail navigation (text-based, just slide titles; full rendering not needed).

```typescript
// app/src/components/SlideNavigator.tsx

interface Slide {
  slide_number: number;
  title?: string;
}

interface Props {
  slides: Slide[];
  currentSlide: number;
  onSelectSlide: (slide: number) => void;
}

export function SlideNavigator({ slides, currentSlide, onSelectSlide }: Props) {
  return (
    <div
      style={{
        width: '120px',
        borderRight: 'var(--zen-border)',
        overflowY: 'auto',
        display: 'flex',
        flexDirection: 'column',
        gap: '2px',
      }}
    >
      {slides.map((slide) => (
        <button
          key={slide.slide_number}
          onClick={() => onSelectSlide(slide.slide_number)}
          style={{
            padding: '8px',
            border: 'none',
            background: currentSlide === slide.slide_number ? 'var(--mcp-accent, #2563eb)' : 'transparent',
            color: currentSlide === slide.slide_number ? '#fff' : 'var(--mcp-text)',
            textAlign: 'left',
            fontSize: '11px',
            cursor: 'pointer',
            whiteSpace: 'nowrap',
            overflow: 'hidden',
            textOverflow: 'ellipsis',
          }}
        >
          {slide.slide_number}. {slide.title || 'Untitled'}
        </button>
      ))}
    </div>
  );
}
```

#### MetadataStrip

Displays deck title, author, total slides, theme, and metadata counts.

```typescript
// app/src/components/MetadataStrip.tsx

import type { SlideDeckData } from '../types';

interface Props {
  data: SlideDeckData;
  currentSlide: number;
}

export function MetadataStrip({ data, currentSlide }: Props) {
  const { title, author, theme, slide_count, math_expressions, bibliography_count } = data.metadata;

  return (
    <div
      style={{
        padding: '8px 12px',
        fontSize: '12px',
        background: 'var(--mcp-bg-secondary, #f9f9f9)',
        borderRadius: 'var(--zen-radius)',
        display: 'grid',
        gridTemplateColumns: 'auto 1fr auto',
        gap: '16px',
        alignItems: 'center',
      }}
    >
      <div>
        <strong>{title}</strong>
        <div style={{ color: 'var(--mcp-text-secondary)' }}>{author}</div>
      </div>
      <div style={{ textAlign: 'center', color: 'var(--mcp-text-secondary)' }}>
        {slide_count} slides • {math_expressions} equations • {bibliography_count} references
      </div>
      <div style={{ textAlign: 'right' }}>
        <span className="zen-badge zen-badge.info" style={{ background: '#e8f5e9', color: '#2e7d32' }}>
          {theme}
        </span>
      </div>
    </div>
  );
}
```

---

## 5. Data Types

TypeScript interfaces for the tool result shape and component state:

```typescript
// app/src/types.ts

/**
 * Shape of the data received from convert_to_slides tool result.
 */
export interface SlideDeckData {
  id: string;
  format: 'slides-beamer' | 'slides-html' | 'slides-both';
  content: {
    beamer?: string; // Base64-encoded PDF
    reveal?: string; // HTML string (full Reveal.js presentation)
  };
  artifacts: Artifact[];
  metadata: SlideDeckMetadata;
  elapsed: number;
}

export interface Artifact {
  type: 'speaker_notes' | 'bibliography' | 'theme_preview' | 'outline';
  filename: string;
  content: string;
}

export interface SlideDeckMetadata {
  slide_count: number;
  title: string;
  author?: string;
  theme: 'metropolis' | 'berlin' | 'warsaw' | 'simple' | 'dark' | 'minimal' | 'sky' | 'league';
  has_speaker_notes: boolean;
  bibliography_count: number;
  math_expressions: number;
  slides: SlideData[];
}

export interface SlideData {
  slide_number: number;
  title?: string;
  body: string; // HTML or Markdown depending on format
  speaker_notes?: string;
}
```

---

## 6. Tool Result Shape

The tool result is received via `app.ontoolresult` and has this shape:

```typescript
interface ToolResult {
  content: Array<{
    type: string;
    text: string; // JSON-serialized SlideDeckData
  }>;
  isError: boolean;
}

// The app parses content[0].text as JSON:
const data = JSON.parse(result.content[0].text) as SlideDeckData;
```

---

## 7. Interactive Tool Calls

The app calls secondary server tools to switch formats or themes:

```typescript
// hook: app/src/hooks/useToolCall.ts

import { useApp } from '@modelcontextprotocol/ext-apps/react';

export function useToolCall() {
  const app = useApp();

  const callTool = async (
    toolName: string,
    args: Record<string, unknown>
  ): Promise<SlideDeckData | null> => {
    try {
      const result = await app.callServerTool({ name: toolName, arguments: args });
      if (result.isError) throw new Error(result.content[0].text);
      return JSON.parse(result.content[0].text);
    } catch (e) {
      console.error(`Tool call failed: ${e instanceof Error ? e.message : 'Unknown error'}`);
      return null;
    }
  };

  return { callTool };
}
```

---

## 8. Server Modifications Required

The server must register the app resource and update the primary tool. Modifications to `/servers/slides-mcp/server/src/index.ts`:

### 8.1 Import Changes

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

### 8.2 Tool Registration (Primary Tool)

Replace the existing `server.registerTool('convert_to_slides', ...)` with:

```typescript
registerAppTool(
  server,
  'convert_to_slides',
  {
    type: 'object',
    properties: {
      source: {
        type: 'string',
        description: 'Markdown presentation with YAML frontmatter (title, author, theme, etc.).',
      },
      format: {
        type: 'string',
        enum: ['beamer', 'reveal', 'both'],
        description: 'Output format(s).',
      },
      theme: {
        type: 'string',
        enum: ['metropolis', 'berlin', 'warsaw', 'simple', 'dark', 'minimal', 'sky', 'league'],
        description: 'Theme name (overrides frontmatter theme if provided).',
      },
      options: {
        type: 'object',
        description: 'Conversion options (speaker_notes, numbered_equations, animations, print_pdf, mathjax_config).',
      },
    },
    required: ['source', 'format'],
    _meta: {
      ui: {
        resourceUri: 'ui://slides-mcp/preview.html',
      },
    },
  },
  convertToSlides
);
```

### 8.3 Resource Registration

Add after tool registrations:

```typescript
const RESOURCE_URI = 'ui://slides-mcp/preview.html';
const DIST_PATH = path.resolve(__dirname, '../../dist/slides-preview.html');

registerAppResource(
  server,
  RESOURCE_URI,
  RESOURCE_URI,
  { mimeType: RESOURCE_MIME_TYPE },
  async () => {
    const html = await fs.readFile(DIST_PATH, 'utf-8');
    return {
      contents: [
        {
          uri: RESOURCE_URI,
          mimeType: RESOURCE_MIME_TYPE,
          text: html,
        },
      ],
    };
  }
);
```

### 8.4 Secondary Tools (Remain Standard)

Keep existing registrations for secondary tools:

```typescript
server.registerTool('validate_slides_source', validateInputSchema, validateSlidesSource);
server.registerTool('list_slide_themes', listThemesInputSchema, listSlideThemes);
```

---

## 9. CSS / Styling Notes

The app uses the shared ZenSci design language from `app/src/styles.css` (identical across all Phase 4 apps):

```css
:root {
  --zen-font-mono: 'JetBrains Mono', 'Fira Code', monospace;
  --zen-font-sans: system-ui, -apple-system, sans-serif;
  --zen-radius: 6px;
  --zen-border: 1px solid var(--mcp-border, #e0e0e0);
  --zen-shadow: 0 1px 3px rgba(0,0,0,0.08);
}

.zen-badge {
  display: inline-flex;
  align-items: center;
  padding: 2px 8px;
  border-radius: 12px;
  font-size: 11px;
  font-weight: 600;
}

.zen-badge.info { background: #d1ecf1; color: #0c5460; }

.zen-tabs {
  display: flex;
  gap: 2px;
  margin-bottom: 12px;
  border-bottom: var(--zen-border);
}

.zen-tab {
  padding: 6px 14px;
  border: none;
  background: transparent;
  cursor: pointer;
  border-radius: var(--zen-radius) var(--zen-radius) 0 0;
  font-size: 13px;
  color: var(--mcp-text-secondary, #666);
}

.zen-tab.active {
  background: var(--mcp-accent, #2563eb);
  color: #ffffff;
}

.zen-meta-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 13px;
}

.zen-meta-table td {
  padding: 4px 8px;
  border-bottom: var(--zen-border);
}
```

**Styling rules:**
- Use `useHostStyles()` to inherit Claude's light/dark mode
- No hardcoded colors outside of semantic status
- Responsive layout: PDF viewer scales to fit container
- Reveal.js HTML rendered in sandboxed iframe with `srcdoc` (no external network requests)

---

## 10. Build Configuration

### 10.1 app/package.json

```json
{
  "name": "@zen-sci/slides-mcp-app",
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

**Note:** `pdfjs-dist` is the only extra dependency (beyond shared base) needed for Beamer PDF rendering via PDF.js.

### 10.2 app/vite.config.ts

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

### 10.3 Module Root package.json (Build Scripts)

Add to `/servers/slides-mcp/package.json`:

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

### Shared Files (Identical Across All Phase 4 Apps)

| File | Description |
|------|-------------|
| `app/vite.config.ts` | Vite + singlefile build config |
| `app/index.html` | HTML entry point (title differs) |
| `app/src/main.tsx` | React app entry |
| `app/src/styles.css` | ZenSci design tokens |
| `app/src/components/LoadingState.tsx` | Loading spinner |
| `app/src/components/ErrorState.tsx` | Error display |
| `app/src/components/EmptyState.tsx` | Empty/no-data display |
| `app/tsconfig.json` | TypeScript config |

### Module-Specific Files

| File | Description |
|------|-------------|
| `app/src/App.tsx` | Root component (uses SlideDeckPreview) |
| `app/src/types.ts` | `SlideDeckData`, `SlideData`, `SlideDeckMetadata` |
| `app/src/components/SlideDeckPreview.tsx` | Main container; orchestrates all sub-components; manages format and note visibility |
| `app/src/components/FormatToolbar.tsx` | Format toggle (Beamer ↔ Reveal.js), speaker notes toggle, loading state |
| `app/src/components/SlideViewer.tsx` | Dispatches to BeamerViewer or RevealViewer based on active format |
| `app/src/components/BeamerViewer.tsx` | PDF.js-based PDF viewer with navigation controls |
| `app/src/components/RevealViewer.tsx` | Sandbox iframe rendering Reveal.js HTML (srcdoc) |
| `app/src/components/SpeakerNotesPanel.tsx` | Collapsible panel showing speaker notes for current slide |
| `app/src/components/SlideNavigator.tsx` | Left sidebar with slide thumbnail navigation (text-based) |
| `app/src/components/MetadataStrip.tsx` | Top metadata display (title, author, slide count, theme badge) |
| `app/src/hooks/useToolResult.ts` | Parse JSON from tool result |
| `app/src/hooks/usePdfWorker.ts` | Set up PDF.js worker (critical for PDF.js functionality) |
| `app/src/hooks/useSlideNavigation.ts` | Current slide state, prev/next/set slide logic |
| `app/src/hooks/useToolCall.ts` | Invoke secondary server tools |
| `dist/slides-preview.html` | Built output (generated by Vite singlefile build) |

---

## 12. Success Criteria

- [ ] `pnpm run build:app` in `/servers/slides-mcp/` produces `dist/slides-preview.html` (single file, no external refs)
- [ ] Server registers `convert_to_slides` with `_meta.ui.resourceUri` pointing to `ui://slides-mcp/preview.html`
- [ ] Server registers resource handler that reads and serves the built HTML
- [ ] App renders tool result without page reload; displays FormatToolbar, MetadataStrip, SlideViewer
- [ ] Beamer PDF renders correctly using PDF.js; user can navigate prev/next slides
- [ ] Reveal.js HTML renders in iframe; interactive navigation works (arrow keys, click)
- [ ] Clicking format toggle switches between PDF and HTML viewers smoothly
- [ ] Speaker notes panel displays correct notes for current slide
- [ ] Slide navigator sidebar shows all slides; clicking thumbnail jumps to slide
- [ ] Metadata strip displays accurate title, author, theme, counts
- [ ] App handles edge cases: single slide, no speaker notes, large decks (50+ slides)
- [ ] No console errors; TypeScript strict mode passes
- [ ] Responsive layout on narrow viewports (min 320px); fixed-aspect PDF canvas
- [ ] PDF.js worker loads correctly; no WASM errors

---

## Audit Notes

**Spec authored:** 2026-02-18
**Grounded in:** Phase 4 architecture spec, slides-mcp v0.1 server spec
**Status:** Production Ready
**Open items:** None

### Design Decisions

1. **Format toggle, not simultaneous display:** Users switch between Beamer (PDF) and Reveal.js (HTML) via toolbar button. Showing both simultaneously would bloat the UI. The tool already generates both formats; the app lets users preview each.

2. **PDF.js for Beamer rendering:** Native browser PDF support is inconsistent. PDF.js provides reliable cross-browser rendering with precise page navigation, essential for slide decks where page boundaries matter.

3. **Sandbox iframe for Reveal.js:** Reveal.js HTML is self-contained (Vite singlefile build inlines all CSS/JS). Sandboxed iframe prevents any external network calls and limits security surface. No `allow-popup` needed for this use case.

4. **Text-based slide thumbnails:** Rendering actual slide thumbnails would be expensive (miniature PDF/HTML canvases for every slide). Text-based navigation (slide number + title) is fast and sufficient for navigating 50+ slide decks.

5. **Collapsible speaker notes:** Speaker notes are optional but common in academic presentations. Placing them in a bottom panel (collapsible) keeps the main slide viewer maximized while allowing quick access to notes.

6. **No external dependencies:** Slides app uses only React + PDF.js + MCP Apps SDK; no reveal.js npm package needed (it's bundled as HTML by the server and rendered as-is by the app).

---

**End of slides-mcp MCP App Specification**
