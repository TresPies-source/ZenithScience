# ZenSci Phase 4 Implementation Prompt
## Visual Application Layer — All 6 MCP App Companions

**Date:** 2026-02-18
**Phase:** 4 of 4
**Author:** ZenSci Architecture (Cruz Morales / TresPiesDesign.com)
**Target Agent:** Claude Code (Opus-level)
**Estimated files:** ~70 new files, 6 modified server entry points

---

## 0. Context and Grounding

You are implementing Phase 4 of ZenSci — the visual application layer that adds a companion MCP App to every ZenSci server module.

**Read these documents in full before writing a single line of code:**

| Priority | File | Why |
|----------|------|-----|
| CRITICAL | `ZenithScience/specs/phase-4/mcp-apps-architecture-spec.md` | Shared architecture, build pipeline, SDK patterns, design language — applies to ALL 6 modules |
| CRITICAL | `ZenithScience/handoffs/handoff_zero.md` | Full project context, constraints, identity |
| CRITICAL | `ZenithScience/CONTEXT.md` | Architecture decisions — especially D1 (factory pattern), D2 (strict enums), D3 (error split) |
| MODULE | `ZenithScience/specs/phase-4/latex-mcp-app-spec.md` | latex-mcp app: PDF preview, PDF.js, citation panel |
| MODULE | `ZenithScience/specs/phase-4/blog-mcp-app-spec.md` | blog-mcp app: HTML preview, SEO tab, OG card |
| MODULE | `ZenithScience/specs/phase-4/grant-mcp-app-spec.md` | grant-mcp app: compliance dashboard, agency selector |
| MODULE | `ZenithScience/specs/phase-4/slides-mcp-app-spec.md` | slides-mcp app: Beamer/Reveal.js dual preview |
| MODULE | `ZenithScience/specs/phase-4/newsletter-mcp-app-spec.md` | newsletter-mcp app: email client preview (srcdoc, 3 viewports) |
| MODULE | `ZenithScience/specs/phase-4/paper-mcp-app-spec.md` | paper-mcp app: academic format preview, format switching |
| CONTEXT | `ZenithScience/specs/infrastructure/packages-sdk-spec.md` | `createZenSciServer()` factory and `ZenSciContext` — server modification context |

---

## 1. Prerequisites

Before starting Phase 4, verify all of the following are true:

- [ ] `zen-sci/packages/core` builds (`pnpm run build` exits 0, tests pass)
- [ ] `zen-sci/packages/sdk` builds and tests pass
- [ ] `zen-sci/servers/latex-mcp/server/` has passing unit tests
- [ ] `zen-sci/servers/blog-mcp/server/` has passing unit tests
- [ ] `zen-sci/servers/grant-mcp/server/` has passing unit tests
- [ ] `zen-sci/servers/slides-mcp/server/` has passing unit tests
- [ ] `zen-sci/servers/newsletter-mcp/server/` has passing unit tests
- [ ] `zen-sci/servers/paper-mcp/server/` has passing unit tests

**If any server is not yet complete, stop and complete Phases 1–3 first.** Phase 4 modifies existing server entry points — they must be stable before modification.

---

## 2. Monorepo Additions (Do First)

### 2.1 Root pnpm workspace

Add the app directories to the pnpm workspace. Edit `zen-sci/pnpm-workspace.yaml`:

```yaml
# zen-sci/pnpm-workspace.yaml
packages:
  - 'packages/*'
  - 'servers/*/server'
  - 'servers/*/app'    # ← ADD THIS LINE
```

### 2.2 Root build scripts

Add to `zen-sci/package.json` scripts:

```json
{
  "scripts": {
    "build:apps": "pnpm --filter './servers/*/app' run build",
    "build:all":  "pnpm run build && pnpm run build:apps",
    "test:apps":  "pnpm --filter './servers/*/app' run test"
  }
}
```

### 2.3 Shared tsconfig for apps

Create `zen-sci/tsconfig.app.json` (base config for all app packages):

```json
{
  "extends": "./tsconfig.base.json",
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true
  }
}
```

---

## 3. Implementation Order

Implement modules in this sequence. Each module follows the same 4-step pattern (detailed in § 4).

1. **latex-mcp** — Implement first. Flagship app; validates PDF.js + ext-apps integration.
2. **blog-mcp** — Second. Validates HTML preview + SEO tab pattern. No extra deps.
3. **grant-mcp** — Third. Validates compliance dashboard pattern. No extra deps.
4. **slides-mcp** — Fourth. Reuses PDF.js from latex-mcp; adds Reveal.js iframe.
5. **newsletter-mcp** — Fifth. Validates email-in-srcdoc iframe pattern.
6. **paper-mcp** — Sixth. Reuses PDF.js pattern from latex-mcp; adds format switcher.

**Rule:** Complete each module fully (app builds, server modified, tests pass) before starting the next.

---

## 4. Per-Module Implementation Pattern (Applies to All 6)

For each module, execute these 4 steps:

### Step 4.1 — Create `app/` directory

```
zen-sci/servers/{module}/app/
├── src/
│   ├── main.tsx
│   ├── App.tsx
│   ├── types.ts
│   ├── styles.css
│   ├── components/
│   │   ├── LoadingState.tsx
│   │   ├── ErrorState.tsx
│   │   ├── EmptyState.tsx
│   │   └── [module-specific components — see module spec § 4]
│   └── hooks/
│       └── useToolResult.ts
├── index.html
├── vite.config.ts
├── tsconfig.json
└── package.json
```

Use the exact file templates from `mcp-apps-architecture-spec.md` §§ 4, 5, 6, 7 for the shared files. Module-specific components are defined in each module's spec.

### Step 4.2 — Implement the React app

Read the module spec § 4 (Component Architecture) and § 6 (Tool Result Shape). Implement:
- Root `App.tsx` following the pattern in architecture spec § 6.1
- All module-specific components listed in the spec
- `types.ts` with all TypeScript interfaces from spec § 5
- `styles.css` with ZenSci tokens from architecture spec § 7

### Step 4.3 — Modify server `index.ts`

Read the module spec § 8 (Server Modifications Required). Apply:
- Import `registerAppTool`, `registerAppResource`, `RESOURCE_MIME_TYPE` from `@modelcontextprotocol/ext-apps/server`
- Replace the primary conversion tool's `server.registerTool()` with `registerAppTool()` (adding `_meta.ui`)
- Add `registerAppResource()` call to serve the built HTML
- DO NOT modify secondary tool registrations (validate, check, generate)

### Step 4.4 — Build and verify

```bash
# From module root (zen-sci/servers/{module}/)
pnpm run build:app        # Builds app/ → dist/
ls dist/                  # Verify {module}-app.html exists
pnpm run build            # Full build (server + app)
pnpm run test             # All tests pass
```

---

## 5. Module-Specific Requirements

### 5A. latex-mcp

**Read:** `specs/phase-4/latex-mcp-app-spec.md` (fully)

Key requirements:
- PDF.js renders `convert_to_pdf` result (base64 PDF decoded to `Uint8Array` → `pdfjsLib.getDocument({ data })`)
- Two tabs: **Preview** (PDF.js viewer) and **LaTeX** (source code `<pre>`)
- Metadata panel: title, authors, page count, citation resolution status (`7/8 resolved`), unresolved citation names shown as red badges
- Warnings strip: yellow bar if `warnings.length > 0`, lists warnings below toolbar
- Interactive: TeX engine dropdown (`pdflatex` / `xelatex` / `lualatex`) + "Re-render" button calls `convert_to_pdf` with updated `options.engine`; "Check Citations" button calls `check_citations`, updates citation panel
- PDF.js worker: `pdfjsLib.GlobalWorkerOptions.workerSrc = URL.createObjectURL(new Blob([...worker code...]))` or use bundled worker from `pdfjs-dist/build/pdf.worker.min.js`
- CSP: `worker-src blob:` required in `registerAppTool` `_meta.ui.csp`
- Extra dep: `pdfjs-dist ^4.0.0`
- resourceUri: `ui://latex-mcp/preview.html`
- dist: `dist/latex-preview.html`

### 5B. blog-mcp

**Read:** `specs/phase-4/blog-mcp-app-spec.md` (fully)

Key requirements:
- Three tabs: **Preview** (sandboxed iframe), **SEO** (OG card + Twitter card + meta table), **Source** (raw HTML)
- Preview: set iframe `srcdoc` attribute to the HTML string — never use `innerHTML`
- Mobile/Desktop toggle: change iframe inline style `width` between `'375px'` (centered) and `'100%'`
- SEO tab: visual OG card (gray border box showing `og:image`, `og:title`, `og:description`); Twitter card preview; validation table (✓ green for present required fields, ✗ red for absent required, − gray for optional)
- Required OG fields: `og:type`, `og:title`, `og:description`, `og:url` — `og:type` must be `'article'` (never `'blog'`)
- "Generate Feed" button calls `generate_feed`, shows XML result in a modal or collapsible panel
- No extra deps
- resourceUri: `ui://blog-mcp/preview.html`
- dist: `dist/blog-preview.html`

### 5C. grant-mcp

**Read:** `specs/phase-4/grant-mcp-app-spec.md` (fully)

Key requirements:
- Two tabs: **Compliance Checker** and **Agency Requirements**
- Overview panel: agency badge (`NIH` / `NSF` / `ERC`, color-coded), overall compliance badge (`PASS`/`WARN`/`FAIL`), page count vs limit, word count vs limit, budget vs limit
- Section checklist: scrollable list; each row shows section name, ✓/⚠/✗ status, current vs required page/word count, color-coded row background
- Attachments table: required attachments with `SubmissionFileRole` strict union type, present/missing status
- Agency Requirements tab: formatted table per agency — page limits, fonts, margins, submission platform (`nsf-research-gov` / `nih-era-commons` / `erc-sygma`)
- Agency selector dropdown re-calls `format_grant_proposal` with new `platform` argument
- `SubmissionFileRole` must match the strict union from `packages-core-spec.md` — no `| string` escape
- No extra deps
- resourceUri: `ui://grant-mcp/dashboard.html`
- dist: `dist/grant-dashboard.html`

### 5D. slides-mcp

**Read:** `specs/phase-4/slides-mcp-app-spec.md` (fully)

Key requirements:
- Format toggle toolbar: "Beamer (PDF)" / "Reveal.js (HTML)" — top-level app state
- Beamer view: PDF.js renders compiled Beamer PDF (same pattern as latex-mcp app). Each PDF page = one slide.
- Reveal.js view: full Reveal.js HTML in `srcdoc` iframe at fixed height (60vh). The HTML from the tool is self-contained.
- Speaker notes panel: collapsible panel below the viewer; shows `slide.speaker_notes` for current slide. Toggle button in toolbar.
- Slide navigator: left sidebar showing slide number + title text for each slide. Click to jump (Beamer: go to PDF page; Reveal.js: cannot navigate programmatically — just show which slide is "active")
- Metadata strip: deck title, author, theme name, total slide count, equation count, reference count
- Format toggle calls `convert_to_slides` with `format: 'beamer' | 'revealjs'`
- Extra dep: `pdfjs-dist ^4.0.0` (Beamer view only)
- resourceUri: `ui://slides-mcp/preview.html`
- dist: `dist/slides-preview.html`

### 5E. newsletter-mcp

**Read:** `specs/phase-4/newsletter-mcp-app-spec.md` (fully)

Key requirements:
- Three client view tabs: **Desktop** (iframe 600px wide), **Mobile** (iframe 375px, centered), **Dark Mode** (iframe with dark styles prepended to HTML)
- Dark Mode: prepend `<style>@media (prefers-color-scheme: dark) { body { background: #1a1a1a !important; color: #e0e0e0 !important; } }</style>` to the HTML string before setting `srcdoc`
- Metadata strip: subject line, preview text, sender name, size badge (green < 60KB, yellow 60-100KB, red > 100KB with warning icon)
- CAN-SPAM compliance panel (collapsible): footer present ✓/✗, unsubscribe link ✓/✗, sender address ✓/✗ → overall PASS/FAIL badge
- Links panel (collapsible): table of all hrefs in the email (link text + URL)
- MJML Source tab: raw MJML in `<pre>` with copy button
- All email HTML renders via `srcdoc` on an iframe — NEVER `innerHTML` on the page DOM
- Inner iframe sandbox: `allow-scripts allow-same-origin allow-popups`
- No extra deps
- resourceUri: `ui://newsletter-mcp/preview.html`
- dist: `dist/newsletter-preview.html`

### 5F. paper-mcp

**Read:** `specs/phase-4/paper-mcp-app-spec.md` (fully)

Key requirements:
- PDF.js renders the compiled academic paper PDF (same pattern as latex-mcp app)
- Format selector dropdown: `IEEE Two-Column` / `ACM Article` / `arXiv Preprint` — calls `convert_to_paper` with new `format` argument, re-renders PDF
- Format badge in toolbar shows human-readable format name
- Paper info sidebar (or panel): abstract in a styled box (collapsible via toggle), author list with affiliations, stats grid — section count, figure count, table count, citation count (2×3 label-number grid)
- Warnings strip: yellow bar if `warnings.length > 0`
- LaTeX Source tab: raw LaTeX in `<pre>` with copy button
- Page navigation: prev/next, page X of Y indicator
- CSP: `worker-src blob:` required
- Extra dep: `pdfjs-dist ^4.0.0`
- resourceUri: `ui://paper-mcp/preview.html`
- dist: `dist/paper-preview.html`

---

## 6. File Manifest

**New files across all 6 modules: ~72 files**
**Modified files: 6 server `index.ts` files + 2 workspace config files**

### Shared new files per module (×6 = 48 files)

| File (relative to `servers/{module}/`) | Description |
|----------------------------------------|-------------|
| `app/package.json` | App package deps (ext-apps, react, vite, singlefile) |
| `app/tsconfig.json` | Extends `../../../tsconfig.app.json` |
| `app/vite.config.ts` | Vite + react + singlefile; outDir = `../dist` |
| `app/index.html` | Vite entry; `<div id="root">` + `<script src="/src/main.tsx">` |
| `app/src/main.tsx` | React root render |
| `app/src/App.tsx` | Root component with `useApp`, `useHostStyles`, state machine |
| `app/src/types.ts` | TypeScript interfaces for tool result shapes |
| `app/src/styles.css` | ZenSci design tokens |

### Module-specific new files per module

**latex-mcp** (+7 = 15 total new):
`app/src/components/PdfViewer.tsx`, `app/src/components/LatexSource.tsx`, `app/src/components/MetadataPanel.tsx`, `app/src/components/CitationBadge.tsx`, `app/src/components/EngineSelector.tsx`, `app/src/components/WarningsStrip.tsx`, `app/src/hooks/usePdfJs.ts`

**blog-mcp** (+9 = 17 total new):
`app/src/components/HtmlPreview.tsx`, `app/src/components/ViewportToggle.tsx`, `app/src/components/SeoTab.tsx`, `app/src/components/OgCardPreview.tsx`, `app/src/components/TwitterCardPreview.tsx`, `app/src/components/MetaTagTable.tsx`, `app/src/components/HtmlSource.tsx`, `app/src/components/FeedModal.tsx`, `app/src/hooks/useFeedCall.ts`

**grant-mcp** (+7 = 15 total new):
`app/src/components/OverviewPanel.tsx`, `app/src/components/SectionChecklist.tsx`, `app/src/components/AttachmentsPanel.tsx`, `app/src/components/AgencyRequirementsTab.tsx`, `app/src/components/ComplianceToolbar.tsx`, `app/src/components/ComplianceBadge.tsx`, `app/src/hooks/useAgencySwitch.ts`

**slides-mcp** (+8 = 16 total new):
`app/src/components/FormatToolbar.tsx`, `app/src/components/BeamerViewer.tsx`, `app/src/components/RevealViewer.tsx`, `app/src/components/SpeakerNotesPanel.tsx`, `app/src/components/SlideNavigator.tsx`, `app/src/components/MetadataStrip.tsx`, `app/src/hooks/usePdfJs.ts`, `app/src/hooks/useFormatSwitch.ts`

**newsletter-mcp** (+7 = 15 total new):
`app/src/components/EmailViewer.tsx`, `app/src/components/ClientViewTabs.tsx`, `app/src/components/MetadataStrip.tsx`, `app/src/components/CanSpamPanel.tsx`, `app/src/components/LinksPanel.tsx`, `app/src/components/MjmlSource.tsx`, `app/src/hooks/useDarkMode.ts`

**paper-mcp** (+7 = 15 total new):
`app/src/components/PaperViewer.tsx`, `app/src/components/FormatSelector.tsx`, `app/src/components/PaperInfoPanel.tsx`, `app/src/components/AbstractBox.tsx`, `app/src/components/StatsGrid.tsx`, `app/src/components/LatexSource.tsx`, `app/src/hooks/usePdfJs.ts`

### Modified files

| File | Change |
|------|--------|
| `zen-sci/pnpm-workspace.yaml` | Add `servers/*/app` to workspace packages |
| `zen-sci/package.json` | Add `build:apps`, `build:all`, `test:apps` scripts |
| `zen-sci/servers/latex-mcp/server/src/index.ts` | `registerAppTool` + `registerAppResource` |
| `zen-sci/servers/blog-mcp/server/src/index.ts` | `registerAppTool` + `registerAppResource` |
| `zen-sci/servers/grant-mcp/server/src/index.ts` | `registerAppTool` + `registerAppResource` |
| `zen-sci/servers/slides-mcp/server/src/index.ts` | `registerAppTool` + `registerAppResource` |
| `zen-sci/servers/newsletter-mcp/server/src/index.ts` | `registerAppTool` + `registerAppResource` |
| `zen-sci/servers/paper-mcp/server/src/index.ts` | `registerAppTool` + `registerAppResource` |

---

## 7. Critical DO NOTs

- **DO NOT** import `@zen-sci/core` or `@zen-sci/sdk` from any `app/` package — apps depend only on `@modelcontextprotocol/ext-apps`, React, and their own module code
- **DO NOT** use `innerHTML` to render email HTML in newsletter-mcp — always use `srcdoc` on a sandboxed iframe
- **DO NOT** use `innerHTML` to render blog HTML in blog-mcp — always use `srcdoc` on a sandboxed iframe
- **DO NOT** hardcode colors (except semantic pass/warn/fail status values) — use `useHostStyles()` and `var(--mcp-*)` CSS variables
- **DO NOT** skip the `app.connect()` call — `useApp()` calls it automatically when using the React hook, but verify this in any manual `new App()` usage
- **DO NOT** call `server.registerTool()` for the primary conversion tool in Phase 4 — use `registerAppTool()` instead (it wraps `registerTool` and adds app metadata)
- **DO NOT** add external CDN script tags to app HTML — Vite singlefile bundles everything; external scripts break the single-file constraint
- **DO NOT** use `localStorage` or `sessionStorage` in any app — not supported in sandboxed MCP App iframes
- **DO NOT** duplicate `usePdfJs.ts` hook code across modules — copy the same implementation (latex-mcp, slides-mcp, paper-mcp all need it; write it once per module but keep it identical)

---

## 8. Success Criteria

### Per-module (must pass for each of the 6 modules):

- [ ] `pnpm run build:app` in module root exits 0
- [ ] `dist/{module}-app.html` is a single self-contained HTML file (no external `<script src>` or `<link href>` references)
- [ ] Server `index.ts` uses `registerAppTool` for primary conversion tool with correct `_meta.ui.resourceUri`
- [ ] Server registers resource handler via `registerAppResource` pointing to the correct dist file
- [ ] App renders initial tool result from `ontoolresult` without page reload
- [ ] App makes at least one interactive tool call from user interaction
- [ ] Unit tests pass: `LoadingState`, `ErrorState`, `EmptyState`, root `App` component render
- [ ] No TypeScript errors (`tsc --noEmit` exits 0 in app directory)

### Global (must pass for the full phase):

- [ ] `pnpm run build:apps` from workspace root builds all 6 apps without errors
- [ ] All 6 apps tested manually in the `basic-host` test harness against running servers
- [ ] `pnpm run test:apps` passes all tests
- [ ] `useHostStyles()` applied at root `div` of every App.tsx
- [ ] No hardcoded colors outside of semantic status values (grep for `#[0-9a-fA-F]{3,6}` should return only the semantic pass/warn/fail values and the dark mode patch in newsletter-mcp)
- [ ] `SCHEMAS.md` updated: Phase 4 types (tool result shapes from each module's `types.ts`) noted as implemented
- [ ] `STATUS.md` updated: Phase 4 complete, implementation prompt location noted, all 6 apps marked ✅

---

## 9. Architecture Grounding

This implementation is grounded in:

- **Official MCP Apps protocol:** Launched January 26, 2026. Spec at `modelcontextprotocol.io/docs/extensions/apps`
- **`@modelcontextprotocol/ext-apps` v1.0.1:** Official Anthropic SDK for building MCP Apps
- **AgenticGateway v1.1.0 App Bridge spec:** `ZenflowProjects/AgenticStackOrchestration/specs/v1.1.0/v110-mcp-apps-bridge-spec.md` — Gateway hosts ZenSci apps via this infrastructure
- **ZenSci Phase 4 specs:** `ZenithScience/specs/phase-4/` — 7 files (1 architecture + 6 module specs)
- **ZenSci server specs (Phases 1–3):** Each app spec references its server spec for tool result shapes

---

*Prompt prepared by: ZenSci Architecture (Cruz Morales / TresPiesDesign.com)*
*Date: 2026-02-18*
*Prerequisites: Phases 0–3 complete*
