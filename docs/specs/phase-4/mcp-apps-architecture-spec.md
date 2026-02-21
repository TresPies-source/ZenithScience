# ZenSci MCP Apps — Architecture Specification
## Phase 4: Visual Application Layer

**Version:** 0.4.0-apps
**Status:** Draft
**Author:** ZenSci Architecture (Cruz Morales / TresPiesDesign.com)
**Date:** 2026-02-18
**Depends on:** Phase 1 (packages/core + packages/sdk + latex-mcp), Phase 2 (blog-mcp + grant-mcp), Phase 3 (slides-mcp + newsletter-mcp + paper-mcp)

---

## 1. Purpose

Phase 4 adds a **visual application layer** to every ZenSci MCP server. Each module gains a companion MCP App — a bundled React UI that renders the tool's output as an interactive panel inside the Claude chat interface (and any other MCP Apps-capable host).

The MCP Apps protocol was launched January 26, 2026. It extends MCP from a tool-only interface into a full application platform: tools can now declare a `resourceUri` pointing to a bundled HTML/React application, which the host renders in a sandboxed iframe alongside the tool result.

**ZenSci Phase 4 goal:** Every module's primary conversion tool renders its output visually. A user calling `convert_to_pdf` sees a live PDF preview panel. A user calling `convert_to_html` sees a live HTML blog preview with SEO score. The text output of tools becomes the *data source* for a rich, interactive interface.

---

## 2. Protocol and SDK Versions

| Dependency | Version | Notes |
| :--- | :--- | :--- |
| `@modelcontextprotocol/ext-apps` | `^1.0.1` | Official MCP Apps SDK |
| `@modelcontextprotocol/ext-apps/react` | (same package) | React hooks: `useApp`, `useHostStyles` |
| `@modelcontextprotocol/ext-apps/server` | (same package) | `registerAppTool`, `registerAppResource`, `RESOURCE_MIME_TYPE` |
| `react` | `^18.3.0` | UI framework for all ZenSci apps |
| `react-dom` | `^18.3.0` | DOM rendering |
| `vite` | `^5.4.0` | Build tool |
| `@vitejs/plugin-react` | `^4.3.0` | React/JSX support in Vite |
| `vite-plugin-singlefile` | `^2.0.0` | Bundles CSS + JS into single HTML file |
| `typescript` | `^5.4.0` | Type-safe app development |

---

## 3. Directory Structure (Per Module)

Every ZenSci server module grows an `app/` sibling alongside its `server/` directory. The built output lives in `dist/` at the module root, where the server reads it to serve as a `ui://` resource.

```
zen-sci/servers/{module-name}/
├── server/                    ← existing MCP server (Phase 1/2/3)
│   ├── src/
│   │   └── index.ts           ← MCP server entry (MODIFIED in Phase 4)
│   └── package.json
├── app/                       ← NEW in Phase 4
│   ├── src/
│   │   ├── main.tsx           ← React app entry point
│   │   ├── App.tsx            ← Root component
│   │   ├── components/        ← Module-specific UI components
│   │   │   └── ...
│   │   ├── hooks/             ← Custom hooks (data parsing, tool calls)
│   │   │   └── useToolResult.ts
│   │   └── types.ts           ← TypeScript types for tool result shapes
│   ├── index.html             ← Vite entry point (references main.tsx)
│   ├── vite.config.ts         ← Vite + singlefile config
│   ├── tsconfig.json
│   └── package.json           ← App deps (ext-apps, react, vite)
├── dist/
│   └── {module}-app.html     ← Built output; served by server
└── package.json               ← Module root (workspace member)
```

**Naming convention for dist output:**

| Module | dist filename |
| :--- | :--- |
| latex-mcp | `latex-preview.html` |
| blog-mcp | `blog-preview.html` |
| grant-mcp | `grant-dashboard.html` |
| slides-mcp | `slides-preview.html` |
| newsletter-mcp | `newsletter-preview.html` |
| paper-mcp | `paper-preview.html` |

---

## 4. Build Pipeline (Vite + Single File)

Every app uses an identical Vite configuration that bundles the React app into a single self-contained HTML file. All CSS and JavaScript is inlined — no external file references in the bundle.

```typescript
// app/vite.config.ts (identical across all modules)
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

```html
<!-- app/index.html (identical across all modules; only title differs) -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>ZenSci — {Module Name}</title>
</head>
<body>
  <div id="root"></div>
  <script type="module" src="/src/main.tsx"></script>
</body>
</html>
```

```typescript
// app/src/main.tsx (identical across all modules)
import React from 'react';
import { createRoot } from 'react-dom/client';
import App from './App.tsx';
import './styles.css';

const root = createRoot(document.getElementById('root')!);
root.render(<React.StrictMode><App /></React.StrictMode>);
```

**Build command** added to each module's root `package.json`:

```json
{
  "scripts": {
    "build:app": "cd app && vite build",
    "build": "tsc && cd app && vite build"
  }
}
```

---

## 5. Server Modifications (All Modules)

Every Phase 1/2/3 server must be modified to:

1. Replace `server.registerTool()` with `registerAppTool()` for each tool that has a visual counterpart
2. Add `registerAppResource()` to serve the bundled HTML

### 5.1 Import Changes

```typescript
// BEFORE (Phase 1/2/3):
import { createZenSciServer } from '@zen-sci/sdk';

// AFTER (Phase 4 addition):
import { createZenSciServer } from '@zen-sci/sdk';
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

### 5.2 Tool Registration Pattern

```typescript
// BEFORE:
server.registerTool('convert_to_pdf', inputSchema, convertToPdf);

// AFTER: primary conversion tool gets _meta.ui
registerAppTool(
  server,
  'convert_to_pdf',
  {
    ...inputSchema,
    _meta: {
      ui: {
        resourceUri: 'ui://latex-mcp/preview.html',
        // CSP allowlist — only add entries each app actually needs:
        // csp: ['https://cdnjs.cloudflare.com'],
      },
    },
  },
  convertToPdf
);

// Secondary tools remain as regular registerTool (no visual layer):
server.registerTool('validate_document', inputSchema, validateDocument);
server.registerTool('check_citations', inputSchema, checkCitations);
```

**Rule:** Only the primary conversion tool (the one that produces the main artifact) gets `_meta.ui`. Secondary utility tools (`validate_document`, `check_citations`, `generate_feed`) remain standard registered tools callable from within the app.

### 5.3 Resource Registration Pattern

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

**`ui://` naming convention:** `ui://{module-name}/{output-name}.html`

| Module | resourceUri |
| :--- | :--- |
| latex-mcp | `ui://latex-mcp/preview.html` |
| blog-mcp | `ui://blog-mcp/preview.html` |
| grant-mcp | `ui://grant-mcp/dashboard.html` |
| slides-mcp | `ui://slides-mcp/preview.html` |
| newsletter-mcp | `ui://newsletter-mcp/preview.html` |
| paper-mcp | `ui://paper-mcp/preview.html` |

---

## 6. React App Architecture (All Modules)

All ZenSci apps follow the same structural pattern. Module-specific components live in `components/`; the SDK integration is always identical.

### 6.1 Root App Component Pattern

```typescript
// app/src/App.tsx — identical pattern across all modules
import { useApp, useHostStyles } from '@modelcontextprotocol/ext-apps/react';
import { useState, useEffect } from 'react';
import type { ToolResultData } from './types';

export default function App() {
  const app = useApp();                    // Auto-connects on mount
  const hostStyles = useHostStyles();      // Inherits Claude's CSS variables
  const [data, setData] = useState<ToolResultData | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Receive initial tool result pushed by host
    app.ontoolresult = (result) => {
      try {
        const parsed = JSON.parse(result.content[0].text) as ToolResultData;
        setData(parsed);
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
    <div style={{ ...hostStyles, padding: '16px' }}>
      {/* Module-specific UI components */}
    </div>
  );
}
```

### 6.2 Tool Call Pattern (Interactive Controls)

```typescript
// Standard pattern for user-triggered tool calls from within the app
const callTool = async (toolName: string, args: Record<string, unknown>) => {
  setLoading(true);
  try {
    const result = await app.callServerTool({ name: toolName, arguments: args });
    if (result.isError) throw new Error(result.content[0].text);
    return JSON.parse(result.content[0].text);
  } catch (e) {
    setError(e instanceof Error ? e.message : 'Tool call failed');
    return null;
  } finally {
    setLoading(false);
  }
};
```

### 6.3 Shared UI States

Every app implements these three states:

```typescript
// app/src/components/LoadingState.tsx
export function LoadingState() {
  return (
    <div style={{ display: 'flex', alignItems: 'center', gap: '8px', padding: '24px' }}>
      <div className="zen-spinner" />
      <span style={{ color: 'var(--mcp-text-secondary, #666)' }}>Loading…</span>
    </div>
  );
}

// app/src/components/ErrorState.tsx
export function ErrorState({ message }: { message: string }) {
  return (
    <div style={{ padding: '16px', color: 'var(--mcp-error, #c0392b)', border: '1px solid currentColor', borderRadius: '6px' }}>
      <strong>Error:</strong> {message}
    </div>
  );
}

// app/src/components/EmptyState.tsx
export function EmptyState() {
  return (
    <div style={{ padding: '24px', textAlign: 'center', color: 'var(--mcp-text-secondary, #666)' }}>
      No result yet. Call the tool to render output here.
    </div>
  );
}
```

---

## 7. ZenSci Design Language

All apps inherit the host's CSS variables via `useHostStyles()`, ensuring they match Claude's light/dark mode. Apps supplement with a minimal ZenSci token set:

```css
/* app/src/styles.css — applied to all modules */
:root {
  /* ZenSci tokens — fallback to sensible defaults if host doesn't set them */
  --zen-font-mono: 'JetBrains Mono', 'Fira Code', monospace;
  --zen-font-sans: system-ui, -apple-system, sans-serif;
  --zen-radius: 6px;
  --zen-border: 1px solid var(--mcp-border, #e0e0e0);
  --zen-shadow: 0 1px 3px rgba(0,0,0,0.08);
  --zen-tab-active-bg: var(--mcp-accent, #2563eb);
  --zen-tab-active-fg: #ffffff;
}

/* Tab navigation — used across all apps with multiple panels */
.zen-tabs {
  display: flex;
  gap: 2px;
  border-bottom: var(--zen-border);
  margin-bottom: 12px;
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
  background: var(--zen-tab-active-bg);
  color: var(--zen-tab-active-fg);
}

/* Status badge — used for compliance/validation status indicators */
.zen-badge {
  display: inline-flex;
  align-items: center;
  padding: 2px 8px;
  border-radius: 12px;
  font-size: 11px;
  font-weight: 600;
}
.zen-badge.pass  { background: #d4edda; color: #155724; }
.zen-badge.warn  { background: #fff3cd; color: #856404; }
.zen-badge.fail  { background: #f8d7da; color: #721c24; }
.zen-badge.info  { background: #d1ecf1; color: #0c5460; }

/* Metadata key-value table */
.zen-meta-table { width: 100%; border-collapse: collapse; font-size: 13px; }
.zen-meta-table td { padding: 4px 8px; border-bottom: var(--zen-border); }
.zen-meta-table td:first-child { color: var(--mcp-text-secondary, #666); width: 40%; }
```

**Design rules:**
- Use `useHostStyles()` values first; ZenSci tokens are supplements, never overrides
- No hardcoded colors other than semantic status colors (pass/warn/fail)
- Tab navigation for apps with multiple views
- `zen-badge` for all status indicators

---

## 8. Package Dependencies Per Module

Each `app/package.json` is identical in structure:

```json
{
  "name": "@zen-sci/{module}-app",
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

Additional dependencies per module (beyond the shared base):

| Module | Extra dependencies |
| :--- | :--- |
| latex-mcp | `pdfjs-dist` (PDF.js for PDF rendering in browser) |
| blog-mcp | *(none — uses DOMParser for HTML preview)* |
| grant-mcp | *(none)* |
| slides-mcp | `reveal.js` (preview of RevealJS slides) |
| newsletter-mcp | *(none — raw HTML render in sandboxed iframe)* |
| paper-mcp | `pdfjs-dist` (same PDF rendering as latex-mcp) |

---

## 9. Security Model

Apps run in a sandboxed iframe. The AgenticGateway `SecurityPolicy` component enforces:

- `sandbox="allow-scripts allow-same-origin"` — no form submissions, no top-level navigation
- CSP header: `default-src 'self'; script-src 'unsafe-inline'; style-src 'unsafe-inline'` by default
- Extended CSP per module only when required (e.g., `pdfjs-dist` worker needs `worker-src blob:`)
- Tool calls from the app are **proxied** through `ToolCallProxy` — the app never calls the MCP server directly

CSP extensions per module:

| Module | Extra CSP needed |
| :--- | :--- |
| latex-mcp | `worker-src blob:` (PDF.js worker) |
| blog-mcp | *(none)* |
| grant-mcp | *(none)* |
| slides-mcp | *(none — Reveal.js inlined by Vite)* |
| newsletter-mcp | `sandbox="allow-scripts allow-same-origin allow-popups"` |
| paper-mcp | `worker-src blob:` (PDF.js worker) |

---

## 10. Monorepo Build Integration

Phase 4 adds `build:app` scripts to each module and a root-level parallel build target:

```json
// zen-sci/package.json (root) — add to existing build scripts
{
  "scripts": {
    "build:apps": "pnpm --filter './servers/**' run build:app",
    "build:all": "pnpm run build && pnpm run build:apps"
  }
}
```

**Build dependency order remains unchanged.** Apps build after servers (they share no code with core/sdk; they only depend on `@modelcontextprotocol/ext-apps`). The dist HTML files are static assets read at runtime by the server.

---

## 11. Testing Strategy

**Unit tests (Vitest + React Testing Library):**
Each app has a `src/__tests__/` directory with:
- `App.test.tsx` — renders without crash; handles loading/error/data states
- `{MainComponent}.test.tsx` — renders tool result data correctly
- `useToolResult.test.ts` — hook parses tool result JSON correctly

**Mock pattern for `@modelcontextprotocol/ext-apps`:**
```typescript
// app/src/__tests__/setup.ts
vi.mock('@modelcontextprotocol/ext-apps/react', () => ({
  useApp: () => ({
    connect: vi.fn(),
    callServerTool: vi.fn().mockResolvedValue({
      content: [{ type: 'text', text: JSON.stringify(mockToolResult) }],
      isError: false,
    }),
    ontoolresult: null,
  }),
  useHostStyles: () => ({}),
}));
```

**Integration test:** Use the official `basic-host` test harness from `github.com/modelcontextprotocol/ext-apps`. Point at each module server and verify:
- App renders on tool call
- Interactive controls produce expected tool call payloads
- Error states render on malformed tool results

---

## 12. File Manifest (Shared Files)

These files are **identical across all 6 modules** — create once per module using the templates above:

| File | Notes |
|------|-------|
| `app/vite.config.ts` | Identical across all modules |
| `app/index.html` | Title differs per module |
| `app/src/main.tsx` | Identical across all modules |
| `app/src/styles.css` | ZenSci design tokens — identical |
| `app/src/components/LoadingState.tsx` | Identical |
| `app/src/components/ErrorState.tsx` | Identical |
| `app/src/components/EmptyState.tsx` | Identical |
| `app/package.json` | Name + extra deps differ per module |
| `app/tsconfig.json` | Identical across all modules |

Module-specific files are defined in each module's individual spec (§ 4 of each spec).

---

## 13. Success Criteria

- [ ] `pnpm run build:apps` from workspace root builds all 6 app bundles without errors
- [ ] Each `dist/{module}-app.html` is a single self-contained HTML file (no external file references)
- [ ] Each server registers its primary tool with `registerAppTool` and the correct `resourceUri`
- [ ] Each server registers a resource handler that reads and serves the built HTML
- [ ] Each app renders initial tool result without a page reload
- [ ] Each app makes at least one interactive tool call from user interaction
- [ ] All apps pass unit tests with ≥ 80% coverage
- [ ] All apps tested in the `basic-host` harness against a running server
- [ ] No hardcoded colors outside of semantic status values
- [ ] `useHostStyles()` applied at the root `div` of every app

---

## Audit Notes (2026-02-18)

**Spec authored:** 2026-02-18
**Grounded in:** `@modelcontextprotocol/ext-apps` v1.0.1 (official SDK), `vite-plugin-singlefile` v2.0.0, `/AgenticStackOrchestration/specs/v1.1.0/v110-mcp-apps-bridge-spec.md` (AgenticGateway App Bridge), `/AgenticStackOrchestration/scouts/2026-02-15_deep_mcp_apps_research.md`
**Status:** Production Ready
**Open items:** None — all architectural decisions resolved before spec was written.
