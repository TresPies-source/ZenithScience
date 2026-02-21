# blog-mcp MCP App Specification
## Phase 4: Visual Application Layer

**Version:** 0.4.0
**Status:** Draft
**Author:** ZenSci Architecture (Cruz Morales / TresPiesDesign.com)
**Date:** 2026-02-18
**Depends on:** `blog-mcp` v0.2.0, `@modelcontextprotocol/ext-apps` ^1.0.1

---

## 1. Purpose

The **blog-mcp-app** provides a rich visual interface for the `blog-mcp` server's primary conversion tool, `convert_to_html`. When a user calls `convert_to_html` from Claude, the app receives the tool result (HTML content, frontmatter, SEO metadata, reading time, word count, and citation status) and renders it as an interactive preview panel with three tabs: **Preview** (live HTML), **SEO** (metadata cards and validation), and **Source** (raw HTML).

**Key interactions:**
- View the rendered HTML blog post in a sandboxed iframe
- Toggle between mobile (375px) and desktop (100%) viewport widths
- Inspect SEO metadata with OG card preview and Twitter card preview
- Review all meta tags with validation checkmarks for required fields
- Copy raw HTML source
- Call `generate_feed` to generate RSS/Atom feed from the post

---

## 2. Tool Integration

### 2.1 Primary Tool: `convert_to_html`

The `convert_to_html` tool is registered with the app's visual layer via `_meta.ui`. Its output is the data source for the entire app.

**Tool registration in server/src/index.ts:**
```typescript
registerAppTool(
  server,
  'convert_to_html',
  {
    type: 'object',
    properties: {
      markdown_source: { type: 'string', description: 'Markdown blog post content' },
      options: {
        type: 'object',
        properties: {
          theme: { type: 'string', enum: ['light', 'dark', 'auto'] },
          format: { type: 'string', enum: ['html', 'html5'] },
        },
      },
    },
    _meta: {
      ui: {
        resourceUri: 'ui://blog-mcp/preview.html',
        // No extra CSP needed — DOMParser and srcdoc are safe
      },
    },
  },
  convertToHtml
);
```

**Tool result shape (received via `app.ontoolresult`):**
```json
{
  "html": "<article><h1>My Blog Post</h1><p>Content here...</p></article>",
  "frontmatter": {
    "title": "My Blog Post",
    "date": "2026-02-18",
    "tags": ["tech", "news"],
    "slug": "my-blog-post"
  },
  "seo": {
    "og": {
      "title": "My Blog Post",
      "description": "A brief summary of the post",
      "type": "article",
      "image": "https://example.com/image.jpg"
    },
    "twitter": {
      "card": "summary_large_image",
      "title": "My Blog Post",
      "description": "A brief summary of the post"
    },
    "schema_org": {
      "@type": "Article",
      "headline": "My Blog Post",
      "author": "Alice Chen",
      "datePublished": "2026-02-18"
    }
  },
  "word_count": 1240,
  "reading_time_minutes": 6,
  "citations": {
    "total": 3,
    "resolved": 3,
    "unresolved": []
  },
  "warnings": []
}
```

### 2.2 Secondary Tools

Called interactively from the app:

**`generate_feed`**
- Input: `{ html: string; frontmatter: {...} }`
- Output: RSS/Atom XML feed as a string
- Called when user clicks "Generate Feed" button
- Shows resulting XML in a modal or expandable panel

**`convert_to_html` (regenerate)**
- Input: `{ markdown_source: string; options: {...} }`
- Output: full tool result (same shape as primary)
- Called when user clicks "Regenerate" button
- Replaces entire post data with new render

---

## 3. App File Structure

```
servers/blog-mcp/
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
│   │   │   ├── PreviewTab.tsx          ← Sandboxed iframe with HTML + viewport toggle
│   │   │   ├── SeoTab.tsx              ← OG card, Twitter card, meta table, validation
│   │   │   ├── SourceTab.tsx           ← Raw HTML with copy button
│   │   │   ├── OgCardPreview.tsx       ← Visual card mimicking link unfurl
│   │   │   ├── TwitterCardPreview.tsx  ← Twitter card preview
│   │   │   ├── MetaTagTable.tsx        ← All meta tags with validation
│   │   │   ├── FeedModal.tsx           ← RSS/Atom XML display
│   │   │   └── ControlsPanel.tsx       ← Regenerate, generate feed buttons
│   │   ├── hooks/
│   │   │   └── useViewportWidth.ts     ← Mobile/desktop toggle state
│   │   └── utils/
│   │       ├── formatMetaTags.ts       ← Extract/format SEO metadata
│   │       └── validateOgTags.ts       ← Check required OG fields
│   ├── index.html                      ← Vite entry (title: "ZenSci — Blog Preview")
│   ├── vite.config.ts                  ← Shared Vite + singlefile config
│   ├── tsconfig.json                   ← Shared TypeScript config
│   └── package.json                    ← App deps (no extra deps beyond base)
├── dist/
│   └── blog-preview.html               ← Built output (self-contained)
└── package.json                        ← Module root
```

---

## 4. Component Architecture

### 4.1 Root App Component

```typescript
// app/src/App.tsx
import { useApp, useHostStyles } from '@modelcontextprotocol/ext-apps/react';
import { useState, useEffect } from 'react';
import type { ConvertToHtmlResult } from './types';
import { PreviewTab } from './components/PreviewTab';
import { SeoTab } from './components/SeoTab';
import { SourceTab } from './components/SourceTab';
import { ControlsPanel } from './components/ControlsPanel';
import { LoadingState, ErrorState, EmptyState } from './components/shared';

type TabName = 'preview' | 'seo' | 'source';

export default function App() {
  const app = useApp();
  const hostStyles = useHostStyles();
  const [data, setData] = useState<ConvertToHtmlResult | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(true);
  const [activeTab, setActiveTab] = useState<TabName>('preview');

  useEffect(() => {
    app.ontoolresult = (result) => {
      try {
        const parsed = JSON.parse(result.content[0].text) as ConvertToHtmlResult;
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
          className={`zen-tab ${activeTab === 'seo' ? 'active' : ''}`}
          onClick={() => setActiveTab('seo')}
        >
          SEO
        </button>
        <button
          className={`zen-tab ${activeTab === 'source' ? 'active' : ''}`}
          onClick={() => setActiveTab('source')}
        >
          Source
        </button>
      </div>

      {/* Tab content */}
      <div style={{ marginTop: '12px' }}>
        {activeTab === 'preview' && <PreviewTab data={data} />}
        {activeTab === 'seo' && <SeoTab data={data} />}
        {activeTab === 'source' && <SourceTab html={data.html} />}
      </div>
    </div>
  );
}
```

### 4.2 PreviewTab Component

```typescript
// app/src/components/PreviewTab.tsx
import { useState } from 'react';
import type { ConvertToHtmlResult } from '../types';

interface PreviewTabProps {
  data: ConvertToHtmlResult;
}

export function PreviewTab({ data }: PreviewTabProps) {
  const [viewportWidth, setViewportWidth] = useState<'mobile' | 'desktop'>('desktop');

  const width = viewportWidth === 'mobile' ? '375px' : '100%';
  const containerStyle: React.CSSProperties =
    viewportWidth === 'mobile'
      ? { maxWidth: '375px', margin: '0 auto' }
      : { width: '100%' };

  return (
    <div style={{ display: 'flex', flexDirection: 'column', gap: '12px' }}>
      {/* Metadata chips */}
      <div style={{ display: 'flex', gap: '12px', fontSize: '12px', flexWrap: 'wrap' }}>
        <span
          style={{
            background: 'var(--mcp-bg-secondary, #f0f0f0)',
            padding: '4px 8px',
            borderRadius: 'var(--zen-radius)',
          }}
        >
          📖 {data.reading_time_minutes} min read
        </span>
        <span
          style={{
            background: 'var(--mcp-bg-secondary, #f0f0f0)',
            padding: '4px 8px',
            borderRadius: 'var(--zen-radius)',
          }}
        >
          ✍ {data.word_count} words
        </span>
      </div>

      {/* Viewport toggle */}
      <div style={{ display: 'flex', gap: '8px', fontSize: '12px' }}>
        <button
          onClick={() => setViewportWidth('mobile')}
          style={{
            padding: '4px 12px',
            background: viewportWidth === 'mobile' ? 'var(--mcp-accent, #2563eb)' : 'transparent',
            color: viewportWidth === 'mobile' ? '#fff' : 'var(--mcp-text-primary)',
            border: '1px solid var(--mcp-border)',
            borderRadius: 'var(--zen-radius)',
            cursor: 'pointer',
          }}
        >
          📱 Mobile (375px)
        </button>
        <button
          onClick={() => setViewportWidth('desktop')}
          style={{
            padding: '4px 12px',
            background: viewportWidth === 'desktop' ? 'var(--mcp-accent, #2563eb)' : 'transparent',
            color: viewportWidth === 'desktop' ? '#fff' : 'var(--mcp-text-primary)',
            border: '1px solid var(--mcp-border)',
            borderRadius: 'var(--zen-radius)',
            cursor: 'pointer',
          }}
        >
          🖥 Desktop (100%)
        </button>
      </div>

      {/* HTML preview in sandboxed iframe */}
      <div
        style={{
          ...containerStyle,
          border: 'var(--zen-border)',
          borderRadius: 'var(--zen-radius)',
          overflow: 'auto',
          maxHeight: '600px',
          background: '#fff',
        }}
      >
        <iframe
          srcDoc={data.html}
          style={{
            width: '100%',
            height: '100%',
            border: 'none',
            minHeight: '400px',
            display: 'block',
          }}
          sandbox="allow-same-origin"
          title="Blog post preview"
        />
      </div>
    </div>
  );
}
```

### 4.3 SeoTab Component

```typescript
// app/src/components/SeoTab.tsx
import { useState } from 'react';
import type { ConvertToHtmlResult } from '../types';
import { OgCardPreview } from './OgCardPreview';
import { TwitterCardPreview } from './TwitterCardPreview';
import { MetaTagTable } from './MetaTagTable';

interface SeoTabProps {
  data: ConvertToHtmlResult;
}

export function SeoTab({ data }: SeoTabProps) {
  return (
    <div style={{ display: 'flex', flexDirection: 'column', gap: '16px' }}>
      {/* OG Card Preview */}
      <div>
        <h3 style={{ margin: '0 0 8px 0', fontSize: '13px', fontWeight: 600 }}>
          Open Graph Preview
        </h3>
        <OgCardPreview og={data.seo.og} />
      </div>

      {/* Twitter Card Preview */}
      <div>
        <h3 style={{ margin: '0 0 8px 0', fontSize: '13px', fontWeight: 600 }}>
          Twitter Card Preview
        </h3>
        <TwitterCardPreview twitter={data.seo.twitter} />
      </div>

      {/* All Meta Tags */}
      <div>
        <h3 style={{ margin: '0 0 8px 0', fontSize: '13px', fontWeight: 600 }}>
          Meta Tags
        </h3>
        <MetaTagTable seo={data.seo} />
      </div>
    </div>
  );
}
```

### 4.4 OgCardPreview Component

```typescript
// app/src/components/OgCardPreview.tsx
interface OgCardProps {
  og: {
    title?: string;
    description?: string;
    image?: string;
    type?: string;
  };
}

export function OgCardPreview({ og }: OgCardProps) {
  return (
    <div
      style={{
        border: 'var(--zen-border)',
        borderRadius: 'var(--zen-radius)',
        overflow: 'hidden',
        background: '#fff',
        maxWidth: '500px',
        boxShadow: 'var(--zen-shadow)',
      }}
    >
      {og.image && (
        <img
          src={og.image}
          alt="OG preview"
          style={{
            width: '100%',
            height: '260px',
            objectFit: 'cover',
            display: 'block',
          }}
          onError={(e) => {
            (e.target as HTMLImageElement).style.display = 'none';
          }}
        />
      )}
      <div style={{ padding: '12px' }}>
        {og.title && (
          <h4 style={{ margin: '0 0 4px 0', fontSize: '14px', fontWeight: 600 }}>
            {og.title}
          </h4>
        )}
        {og.description && (
          <p
            style={{
              margin: '4px 0 0 0',
              fontSize: '12px',
              color: 'var(--mcp-text-secondary, #666)',
              lineHeight: '1.4',
            }}
          >
            {og.description}
          </p>
        )}
        {og.type && (
          <p
            style={{
              margin: '8px 0 0 0',
              fontSize: '11px',
              color: 'var(--mcp-text-secondary, #999)',
            }}
          >
            Type: {og.type}
          </p>
        )}
      </div>
    </div>
  );
}
```

### 4.5 TwitterCardPreview Component

```typescript
// app/src/components/TwitterCardPreview.tsx
interface TwitterCardProps {
  twitter: {
    card?: string;
    title?: string;
    description?: string;
  };
}

export function TwitterCardPreview({ twitter }: TwitterCardProps) {
  return (
    <div
      style={{
        border: '1px solid #ccc',
        borderRadius: var(--zen-radius),
        padding: '12px',
        background: '#f9f9f9',
        maxWidth: '500px',
        fontSize: '12px',
      }}
    >
      <div
        style={{
          fontSize: '10px',
          color: '#666',
          marginBottom: '4px',
        }}
      >
        X (Twitter) Preview
      </div>
      {twitter.title && (
        <h4 style={{ margin: '0 0 4px 0', fontSize: '13px', fontWeight: 600 }}>
          {twitter.title}
        </h4>
      )}
      {twitter.description && (
        <p
          style={{
            margin: '4px 0 0 0',
            fontSize: '12px',
            color: '#666',
            lineHeight: '1.4',
          }}
        >
          {twitter.description}
        </p>
      )}
      {twitter.card && (
        <div style={{ marginTop: '8px', fontSize: '11px', color: '#999' }}>
          Card type: {twitter.card}
        </div>
      )}
    </div>
  );
}
```

### 4.6 MetaTagTable Component

```typescript
// app/src/components/MetaTagTable.tsx
interface MetaTagTableProps {
  seo: {
    og: Record<string, any>;
    twitter: Record<string, any>;
    schema_org: Record<string, any>;
  };
}

export function MetaTagTable({ seo }: MetaTagTableProps) {
  const requiredOgFields = ['title', 'description', 'image', 'type'];

  // Flatten all SEO fields into a list of {name, value, required?, present?}
  const rows: Array<{ name: string; value: string; namespace: string; required?: boolean; present: boolean }> = [];

  // OG tags
  Object.entries(seo.og || {}).forEach(([key, value]) => {
    rows.push({
      name: `og:${key}`,
      value: String(value || ''),
      namespace: 'OG',
      required: requiredOgFields.includes(key),
      present: !!value,
    });
  });

  // Twitter tags
  Object.entries(seo.twitter || {}).forEach(([key, value]) => {
    rows.push({
      name: `twitter:${key}`,
      value: String(value || ''),
      namespace: 'Twitter',
      present: !!value,
    });
  });

  // Schema.org (first 3-4 fields only to keep table short)
  Object.entries(seo.schema_org || {})
    .slice(0, 4)
    .forEach(([key, value]) => {
      rows.push({
        name: key,
        value: typeof value === 'string' ? value : JSON.stringify(value),
        namespace: 'Schema.org',
        present: !!value,
      });
    });

  return (
    <div style={{ overflowX: 'auto' }}>
      <table
        className="zen-meta-table"
        style={{
          width: '100%',
          borderCollapse: 'collapse',
          fontSize: '12px',
          border: 'var(--zen-border)',
          borderRadius: 'var(--zen-radius)',
          overflow: 'hidden',
        }}
      >
        <thead>
          <tr style={{ background: 'var(--mcp-bg-secondary, #f9f9f9)' }}>
            <th style={{ textAlign: 'left', padding: '8px', borderBottom: 'var(--zen-border)', fontWeight: 600 }}>
              Meta Tag
            </th>
            <th style={{ textAlign: 'left', padding: '8px', borderBottom: 'var(--zen-border)', fontWeight: 600 }}>
              Value
            </th>
            <th style={{ textAlign: 'center', padding: '8px', borderBottom: 'var(--zen-border)', fontWeight: 600, width: '60px' }}>
              Status
            </th>
          </tr>
        </thead>
        <tbody>
          {rows.map((row, i) => (
            <tr key={i}>
              <td style={{ padding: '8px', borderBottom: 'var(--zen-border)', fontWeight: 500 }}>
                {row.name}
              </td>
              <td
                style={{
                  padding: '8px',
                  borderBottom: 'var(--zen-border)',
                  maxWidth: '300px',
                  overflow: 'hidden',
                  textOverflow: 'ellipsis',
                  whiteSpace: 'nowrap',
                  color: row.present ? 'inherit' : 'var(--mcp-text-secondary)',
                  fontSize: '11px',
                }}
              >
                {row.value || '(empty)'}
              </td>
              <td
                style={{
                  padding: '8px',
                  borderBottom: 'var(--zen-border)',
                  textAlign: 'center',
                  fontSize: '14px',
                }}
              >
                {row.present ? (
                  <span style={{ color: 'green' }}>✓</span>
                ) : row.required ? (
                  <span style={{ color: 'red' }}>✗</span>
                ) : (
                  <span style={{ color: '#999' }}>−</span>
                )}
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

### 4.7 SourceTab Component

```typescript
// app/src/components/SourceTab.tsx
import { useState } from 'react';

interface SourceTabProps {
  html: string;
}

export function SourceTab({ html }: SourceTabProps) {
  const [copied, setCopied] = useState(false);

  const handleCopy = () => {
    navigator.clipboard.writeText(html).then(() => {
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
          margin: 0,
        }}
      >
        {html}
      </pre>
    </div>
  );
}
```

### 4.8 ControlsPanel Component

```typescript
// app/src/components/ControlsPanel.tsx
import { useState } from 'react';
import type { ConvertToHtmlResult } from '../types';
import { FeedModal } from './FeedModal';

interface ControlsPanelProps {
  data: ConvertToHtmlResult;
  app: any; // useApp() return type
}

export function ControlsPanel({ data, app }: ControlsPanelProps) {
  const [loading, setLoading] = useState(false);
  const [feedXml, setFeedXml] = useState<string | null>(null);

  const handleRegenerate = async () => {
    setLoading(true);
    try {
      const result = await app.callServerTool({
        name: 'convert_to_html',
        arguments: {
          markdown_source: data.markdown_source || '',
        },
      });
      if (result.isError) {
        console.error('Regenerate failed:', result.content[0].text);
      }
    } catch (e) {
      console.error('Tool call error:', e);
    } finally {
      setLoading(false);
    }
  };

  const handleGenerateFeed = async () => {
    setLoading(true);
    try {
      const result = await app.callServerTool({
        name: 'generate_feed',
        arguments: {
          html: data.html,
          frontmatter: data.frontmatter,
        },
      });
      if (!result.isError) {
        setFeedXml(result.content[0].text);
      } else {
        console.error('Generate feed failed:', result.content[0].text);
      }
    } catch (e) {
      console.error('Tool call error:', e);
    } finally {
      setLoading(false);
    }
  };

  return (
    <>
      <div style={{ display: 'flex', gap: '8px', flexWrap: 'wrap' }}>
        <button
          onClick={handleRegenerate}
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
          {loading ? 'Processing…' : 'Regenerate'}
        </button>

        <button
          onClick={handleGenerateFeed}
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
          Generate Feed
        </button>
      </div>

      {feedXml && <FeedModal xml={feedXml} onClose={() => setFeedXml(null)} />}
    </>
  );
}
```

### 4.9 FeedModal Component

```typescript
// app/src/components/FeedModal.tsx
import { useState } from 'react';

interface FeedModalProps {
  xml: string;
  onClose: () => void;
}

export function FeedModal({ xml, onClose }: FeedModalProps) {
  const [copied, setCopied] = useState(false);

  const handleCopy = () => {
    navigator.clipboard.writeText(xml).then(() => {
      setCopied(true);
      setTimeout(() => setCopied(false), 2000);
    });
  };

  return (
    <div
      style={{
        position: 'fixed',
        inset: 0,
        background: 'rgba(0,0,0,0.5)',
        display: 'flex',
        alignItems: 'center',
        justifyContent: 'center',
        zIndex: 1000,
      }}
      onClick={onClose}
    >
      <div
        style={{
          background: '#fff',
          borderRadius: 'var(--zen-radius)',
          padding: '16px',
          maxWidth: '600px',
          maxHeight: '80vh',
          overflow: 'auto',
          boxShadow: '0 20px 25px -5px rgba(0,0,0,0.3)',
        }}
        onClick={(e) => e.stopPropagation()}
      >
        <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center', marginBottom: '12px' }}>
          <h3 style={{ margin: 0, fontSize: '14px', fontWeight: 600 }}>
            Feed XML
          </h3>
          <button
            onClick={onClose}
            style={{
              background: 'transparent',
              border: 'none',
              fontSize: '20px',
              cursor: 'pointer',
              padding: 0,
            }}
          >
            ×
          </button>
        </div>

        <button
          onClick={handleCopy}
          style={{
            padding: '6px 12px',
            background: 'var(--mcp-accent, #2563eb)',
            color: '#fff',
            border: 'none',
            borderRadius: 'var(--zen-radius)',
            cursor: 'pointer',
            fontSize: '12px',
            marginBottom: '12px',
          }}
        >
          {copied ? '✓ Copied' : 'Copy XML'}
        </button>

        <pre
          style={{
            background: 'var(--mcp-bg-secondary, #f5f5f5)',
            padding: '12px',
            borderRadius: 'var(--zen-radius)',
            border: 'var(--zen-border)',
            overflow: 'auto',
            maxHeight: '400px',
            fontFamily: 'var(--zen-font-mono)',
            fontSize: '11px',
            lineHeight: '1.4',
            margin: 0,
          }}
        >
          {xml}
        </pre>
      </div>
    </div>
  );
}
```

---

## 5. Data Types

```typescript
// app/src/types.ts
export interface ConvertToHtmlResult {
  html: string;
  frontmatter: {
    title?: string;
    date?: string;
    tags?: string[];
    slug?: string;
    [key: string]: any;
  };
  seo: {
    og: {
      title?: string;
      description?: string;
      type?: string;
      image?: string;
      [key: string]: any;
    };
    twitter: {
      card?: string;
      title?: string;
      description?: string;
      [key: string]: any;
    };
    schema_org: {
      '@type'?: string;
      headline?: string;
      author?: string;
      datePublished?: string;
      [key: string]: any;
    };
  };
  word_count: number;
  reading_time_minutes: number;
  citations: {
    total: number;
    resolved: number;
    unresolved?: string[];
  };
  warnings?: string[];
  markdown_source?: string; // For regeneration
}

export interface Frontmatter {
  title?: string;
  date?: string;
  tags?: string[];
  slug?: string;
  [key: string]: any;
}

export interface SeoMetadata {
  og: Record<string, string | undefined>;
  twitter: Record<string, string | undefined>;
  schema_org: Record<string, any>;
}

export type ViewportMode = 'mobile' | 'desktop';
```

---

## 6. Tool Result Shape

The app receives the `convert_to_html` tool result via `app.ontoolresult`. The host serializes it as JSON text in `result.content[0].text`. The App component parses this and passes it to sub-components.

**Shape (TypeScript interface):**
```typescript
{
  html: string;                         // Full HTML string (safe to render in iframe)
  frontmatter: {
    title?: string;
    date?: string;
    tags?: string[];
    slug?: string;
    [key: string]: any;
  };
  seo: {
    og: {
      title?: string;
      description?: string;
      type?: string;                    // Usually "article"
      image?: string;                   // URL to image
      [key: string]: any;
    };
    twitter: {
      card?: string;                    // e.g., "summary_large_image"
      title?: string;
      description?: string;
      [key: string]: any;
    };
    schema_org: {
      '@type'?: string;                 // e.g., "Article"
      headline?: string;
      author?: string;
      datePublished?: string;
      [key: string]: any;
    };
  };
  word_count: number;
  reading_time_minutes: number;
  citations: {
    total: number;
    resolved: number;
    unresolved?: string[];
  };
  warnings?: string[];
  markdown_source?: string;             // Optional; used by regenerate
}
```

---

## 7. Interactive Tool Calls

The app makes two types of interactive tool calls:

### 7.1 Regenerate Post

**From:** ControlsPanel "Regenerate" button
**Calls:** `convert_to_html` again
**Payload:**
```json
{
  "name": "convert_to_html",
  "arguments": {
    "markdown_source": "# My Blog Post\n\nContent here..."
  }
}
```

**Handler:**
```typescript
const result = await app.callServerTool({
  name: 'convert_to_html',
  arguments: {
    markdown_source: data.markdown_source || '',
  },
});
```

### 7.2 Generate Feed

**From:** ControlsPanel "Generate Feed" button
**Calls:** `generate_feed`
**Payload:**
```json
{
  "name": "generate_feed",
  "arguments": {
    "html": "<article>...</article>",
    "frontmatter": {
      "title": "My Blog Post",
      "date": "2026-02-18",
      "tags": ["tech"]
    }
  }
}
```

**Handler:**
```typescript
const result = await app.callServerTool({
  name: 'generate_feed',
  arguments: {
    html: data.html,
    frontmatter: data.frontmatter,
  },
});
```

The result is raw XML text, displayed in a modal for user to copy.

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

Replace the existing `server.registerTool('convert_to_html', ...)` call with:

```typescript
registerAppTool(
  server,
  'convert_to_html',
  {
    type: 'object',
    properties: {
      markdown_source: {
        type: 'string',
        description: 'Markdown blog post content',
      },
      options: {
        type: 'object',
        properties: {
          theme: {
            type: 'string',
            enum: ['light', 'dark', 'auto'],
            description: 'Color theme for HTML output',
          },
          format: {
            type: 'string',
            enum: ['html', 'html5'],
            description: 'HTML version',
          },
        },
      },
    },
    required: ['markdown_source'],
    _meta: {
      ui: {
        resourceUri: 'ui://blog-mcp/preview.html',
        // No extra CSP needed — DOMParser and srcdoc are safe
      },
    },
  },
  convertToHtml
);
```

### 8.3 Secondary Tools

Keep as standard `server.registerTool()`:

```typescript
server.registerTool(
  'generate_feed',
  {
    type: 'object',
    properties: {
      html: {
        type: 'string',
        description: 'HTML blog post content',
      },
      frontmatter: {
        type: 'object',
        description: 'Blog post metadata',
      },
      format: {
        type: 'string',
        enum: ['rss', 'atom'],
        description: 'Feed format',
      },
    },
    required: ['html', 'frontmatter'],
  },
  generateFeed
);

server.registerTool(
  'validate_html',
  {
    type: 'object',
    properties: {
      html: {
        type: 'string',
        description: 'HTML to validate',
      },
    },
    required: ['html'],
  },
  validateHtml
);
```

### 8.4 Resource Registration

Add this after all tool registrations:

```typescript
const RESOURCE_URI = 'ui://blog-mcp/preview.html';
const DIST_PATH = path.resolve(__dirname, '../../dist/blog-preview.html');

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

**Shared styles** (from `mcp-apps-architecture-spec.md` § 7) are in `app/src/styles.css`. The app additionally uses:

- **Iframe container:** Uses `srcdoc` attribute to avoid external file references; set `sandbox="allow-same-origin"` for safety
- **Viewport toggle buttons:** Simple inline buttons with background color indicating active state
- **Metadata chips:** Light background with padding and border-radius; display as flex row with gap
- **OG card preview:** Mimics link unfurl styling — image at top, text below with padding
- **Twitter card preview:** Simpler text-only preview with metadata at bottom
- **Meta tag table:** Uses `.zen-meta-table` class with status indicators (✓ green, ✗ red, − gray)
- **Feed modal:** Fixed position overlay with semi-transparent backdrop; closes on outer click
- **All buttons and controls:** Inherit accent color from host; disabled state via opacity

**No hardcoded colors** except semantic status. All inherit from host via `useHostStyles()`.

---

## 10. Build Configuration

### 10.1 `app/package.json`

```json
{
  "name": "@zen-sci/blog-mcp-app",
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

**No extra dependencies** beyond the shared base — DOMParser and iframe handling are built-in to the browser.

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

Add to `servers/blog-mcp/package.json`:

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
| `app/package.json` | App dependencies (no extras beyond base) |
| `app/index.html` | Vite entry (title: "ZenSci — Blog Preview") |
| `app/vite.config.ts` | Vite + singlefile config (shared template) |
| `app/tsconfig.json` | TypeScript config (shared template) |
| `app/src/main.tsx` | React entry point (shared template) |
| `app/src/styles.css` | ZenSci design tokens (shared) |
| `app/src/App.tsx` | Root component with 3-tab layout |
| `app/src/types.ts` | TypeScript interfaces (ConvertToHtmlResult, etc.) |
| `app/src/components/LoadingState.tsx` | Loading UI (shared) |
| `app/src/components/ErrorState.tsx` | Error UI (shared) |
| `app/src/components/EmptyState.tsx` | Empty state UI (shared) |
| `app/src/components/PreviewTab.tsx` | HTML preview in iframe + viewport toggle |
| `app/src/components/SeoTab.tsx` | SEO metadata views (OG, Twitter, meta tags) |
| `app/src/components/SourceTab.tsx` | Raw HTML with copy button |
| `app/src/components/OgCardPreview.tsx` | Visual OG card (mimics link unfurl) |
| `app/src/components/TwitterCardPreview.tsx` | Twitter card preview |
| `app/src/components/MetaTagTable.tsx` | All meta tags with validation checkmarks |
| `app/src/components/ControlsPanel.tsx` | Regenerate + generate feed buttons |
| `app/src/components/FeedModal.tsx` | Modal for displaying RSS/Atom XML |
| `app/src/hooks/useViewportWidth.ts` | Mobile/desktop viewport toggle (optional) |
| `app/src/utils/formatMetaTags.ts` | Extract/format SEO metadata (optional) |
| `dist/blog-preview.html` | Built output (generated by `npm run build:app`) |

---

## 12. Success Criteria

- [ ] `pnpm run build:app` from `servers/blog-mcp/` builds without errors
- [ ] `dist/blog-preview.html` is a single self-contained HTML file
- [ ] Server registers `convert_to_html` with `registerAppTool` and `_meta.ui.resourceUri = 'ui://blog-mcp/preview.html'`
- [ ] Server registers app resource via `registerAppResource`
- [ ] App renders HTML preview in sandboxed iframe when `convert_to_html` is called
- [ ] Viewport toggle (mobile/desktop) works; iframe width changes correctly
- [ ] Reading time and word count chips display correctly
- [ ] SEO tab shows OG card preview with image and text
- [ ] SEO tab shows Twitter card preview
- [ ] SEO tab shows meta tag table with validation (✓ for present, ✗ for missing required, − for optional)
- [ ] Source tab shows raw HTML with working copy button
- [ ] "Regenerate" button calls `convert_to_html` again
- [ ] "Generate Feed" button calls `generate_feed` and displays XML in modal
- [ ] Modal closes when clicking outside or × button
- [ ] Feed modal has working copy button
- [ ] No hardcoded colors outside semantic status
- [ ] `useHostStyles()` applied at root
- [ ] Iframe sandbox attribute is set to `allow-same-origin` (safe)

---

## Audit Notes (2026-02-18)

**Spec authored:** 2026-02-18
**Author:** ZenSci Architecture (Cruz Morales)
**Grounded in:** `mcp-apps-architecture-spec.md` (Phase 4 reference architecture), HTML5 `srcdoc` API, DOMParser API
**Status:** Draft
**Key decisions:**
- `srcdoc` iframe attribute for rendering HTML (no external file references)
- Three-tab layout: Preview (HTML), SEO (metadata), Source (raw HTML)
- Mobile (375px) and desktop (100%) viewport toggle
- Metadata chips for reading time and word count (no sidebar needed)
- OG + Twitter card previews to show how link would appear when shared
- Meta tag table with validation for required OG fields (title, description, image, type)
- Feed generation as secondary tool call with modal display
- No external CSS libraries; all styling uses ZenSci design tokens
