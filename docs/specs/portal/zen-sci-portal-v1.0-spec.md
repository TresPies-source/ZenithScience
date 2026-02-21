# zen-sci-portal v1.0: The Forge
**Author:** Architecture & Design (TresPiesDesign.com)
**Status:** Release Specification
**Date:** 2026-02-18
**Grounded In:** Strategic Scout (2026-02-18), Tauri Desktop Framework Research (2026-02-18), AgenticGateway v1.0.0 (production-ready), ZenSci 6-server suite (493 tests passing)

---

## 1. Vision

> **The local-first, knowledge-work forge:** A Tauri desktop app where academics, researchers, and students create publication-ready documents (papers, grants, slides, newsletters, blogs) using zen-sci's 6 MCP servers, backed by local full-text search, JWT auth via keychain, and optional cloud sync.

### The Core Insight

zen-sci-portal is **not** a web app. It's a native desktop application that owns three capabilities Tauri + Rust uniquely provide:

1. **Local-first persistence and search** — Tantivy full-text index indexed against all user document outputs (PDFs, HTML, emails, slides). Sub-50ms search latency. Offline-first knowledge discovery.

2. **Secure credential management** — JWT tokens stored in platform keychain (macOS Keychain, Windows DPAPI, Linux Secret Service). Multi-user device support without shipping unencrypted secrets to localStorage.

3. **Lightweight desktop deployment** — 10MB bundle, 30–50MB idle memory, <0.5s startup. Adoption-critical footprint vs. Electron (100MB+, 150–300MB idle, 1–2s startup).

The **frontend** (SvelteKit) is consistent with MCP Apps (Phase 4) and handles social, collaborative, and real-time UI concerns. The **Rust backend** (Tauri core) owns local search, auth, file watching, and AgenticGateway sidecar bundling. The **cloud layer** (AgenticGateway + ZenSci) is unchanged — the portal is a new surface, not a platform overhaul.

---

## 1.5 Current State

### Existing Assets Ready to Build On

**AgenticGateway v1.0.0** (Production-Ready, Go)
- 8 LLM providers (Anthropic, OpenAI, Google, Groq, Mistral, Kimi, DeepSeek, Ollama)
- 33 registered tools (file ops, web ops, computation, planning, research, visual)
- 44 tiered skills (Tiers 0-3, meta-skill invocation with call depth enforcement)
- Memory system: SQLite + embeddings + compression + tiered context construction
- OTEL observability integrated with Langfuse
- All endpoints OpenAI-compatible; MCP integration complete
- **Status:** Ready to bundle as Tauri sidecar

**ZenithScience 6 MCP Servers** (Phases 0–3 Complete, TypeScript)
- **latex-mcp** (v0.1): 37 tests. Markdown → LaTeX → PDF with math validation + citation management
- **blog-mcp** (v0.2): 76 tests. Markdown → HTML + RSS + SEO validation
- **grant-mcp** (v0.4): 50 tests. NIH/NSF proposal generation + compliance checking
- **slides-mcp** (v0.3): 34 tests. Markdown → Beamer + Reveal.js
- **newsletter-mcp** (v0.3): 63 tests. Markdown → MJML + email rendering
- **paper-mcp** (v0.5): 36 tests. IEEE/ACM/arXiv template rendering
- **Total:** 493 tests passing, 91%+ statement coverage, production-ready
- **Phase 4 (MCP Apps):** Specced (interactive PDF.js viewers, SEO dashboards, compliance panels). Not yet implemented; portal v1.0 will support Phase 4 app rendering in webview iframes.

**Rust Crate Ecosystem (Production-Ready)**
- **Tantivy** (0.22+): Full-text search, Lucene-equivalent, <10ms startup, 2x faster than Lucene
- **Tokio** (1.40+): Async runtime for background indexing, file watching, WebSocket real-time (v1.1+)
- **notify** (5.2+): OS-level file-system events (inotify/FSEvents/ReadDirectoryChangesW)
- **jsonwebtoken**: JWT creation, validation, rotation
- **security-framework** / **windows-credentials** / **secret-service**: Platform keychain integration
- **rusqlite** / **tokio-rusqlite**: SQLite embedded database
- **serde** / **bincode**: Serialization
- **reqwest**: HTTP client for Gateway communication
- **pulldown-cmark** / **comrak**: Markdown parsing

### Versions & Pinning

**Tauri:** v2.x (native webview, multi-platform, sidecar support)
**SvelteKit:** v2.0+ (stable, file-based routing, SSG adapter for Tauri)
**Svelte:** v5.0+ (fine-grained reactivity, minimal overhead for Tauri IPC)
**Node.js:** v20 LTS (minimum)
**Rust:** 1.75+ (MSRV for tokio 1.40+, tantivy 0.22+)

---

## 2. Goals & Success Criteria

### Primary Goals

1. **Deliver a self-contained Tauri desktop app** that users download once and launch with one click. All dependencies (AgenticGateway sidecar, ZenSci servers) bundled. No manual setup required.

2. **Local-first knowledge work:** Users open the app, create/edit markdown documents, and instantly search across all outputs (PDFs, HTML, emails, slides) via local Tantivy index. Offline search works when Gateway unavailable.

3. **Multi-user device support:** Each family member (or coworker sharing a machine) logs in with their own credentials. JWT stored in platform keychain, not localStorage. Each user sees their own documents and cloud sync status.

4. **Zero-friction onboarding:** First launch → login (optional, can use locally) → browse sample documents → create new document → pick format (LaTeX/blog/grant/slides/newsletter/paper) → receive output in <5 seconds.

5. **Prove Tauri viability for knowledge platforms:** Demonstrate that 10MB footprint + native webview + Rust backend is the right architecture for offline-first, collaborative knowledge work at scale.

### Success Criteria

- ✅ **App distribution:** Signed, notarized installers for macOS (Intel + ARM), Windows (MSVC x64), Linux (AppImage). <10MB download. Zero external dependencies (all bundled).
- ✅ **Search indexing:** User creates document → converted to PDF/HTML/email/slides → all formats indexed to Tantivy within 2 seconds. Search query returns results in <50ms.
- ✅ **Auth:** First launch → user optionally logs in via JWT. Token stored in platform keychain. Subsequent launches: silent auth from keychain. No password re-entry.
- ✅ **AgenticGateway sidecar:** App bundles Go binary (cross-platform naming per Tauri spec). Launches on app startup. Health-checked every 10 seconds. If crashes, auto-restart with exponential backoff.
- ✅ **MCP App embedding:** Phase 4 apps (PDF.js viewers, SEO panels, compliance dashboards) render in webview iframes. iFrame src points to bundled HTML via `ui://` protocol.
- ✅ **File sync (optional v1.1):** User can watch a local folder (`~/zen-sci/` by default). File changes → queued for ZenSci processing → outputs synced back to folder. Tauri file watcher (notify crate) powers this.
- ✅ **Real-time collaboration (optional v1.1):** Multiple users on same document → changes sync via WebSocket + Automerge CRDT. Live presence indicators. Conflict-free merging.
- ✅ **Performance:** App launches in <0.5 seconds. Search query < 50ms. Document conversion < 5 seconds (typical ~20 pages). Idle memory < 50MB.
- ✅ **Accessibility & polish:** Keyboard navigation throughout UI. Dark mode support. Responsive design for 13" to 27" displays. All error messages have actionable guidance.
- ✅ **Test coverage:** Unit tests for all Rust modules (search indexer, auth manager, file watcher, sidecar launcher). Integration tests for SvelteKit UI. E2E tests for happy-path workflows.

### Explicit Non-Goals

- ❌ **Real-time collaboration in v1.0.** (deferred to v1.1; async via revision history sufficient for v1.0)
- ❌ **Mobile (iOS/Android).** (Tauri supports it, but out of scope for v1.0; desktop-first focus)
- ❌ **Custom LaTeX package auto-install.** (Users can add preamble; we don't vendor TeX packages)
- ❌ **Web-based portal.** (SvelteKit is desktop-only in v1.0; web is v2.0+ scope)
- ❌ **GDPR/HIPAA compliance by default.** (Data lives locally; cloud sync is optional. Security is user's responsibility if using cloud.)
- ❌ **Symbolic math CAS.** (ZenSci's SymPy integration handles math validation; we don't build a math engine.)

---

## 3. Technical Architecture

### 3.1 System Overview

```
┌──────────────────────────────────────────────────────────────┐
│  zen-sci-portal (Tauri v2 Application)                       │
│                                                              │
│  ┌────────────────────────────┐  ┌──────────────────────┐   │
│  │  SvelteKit Frontend        │  │  Rust Backend        │   │
│  │  (WebView)                 │  │  (Tauri Core)        │   │
│  │                            │  │                      │   │
│  │  Routes:                   │  │  Modules:            │   │
│  │  - /auth (login/logout)    │◄─┼─┤ SearchIndexer      │   │
│  │  - /workspace (folder nav) │  │ │ (Tantivy)          │   │
│  │  - /editor (doc editor)    │  │ │                    │   │
│  │  - /search (results)       │  │ │ AuthManager        │   │
│  │  - /settings               │  │ │ (JWT + keychain)   │   │
│  │  - /app/* (MCP App iframes)│  │ │                    │   │
│  │                            │  │ │ FileWatcher        │   │
│  │  Components:               │  │ │ (notify)           │   │
│  │  - SearchBar               │  │ │                    │   │
│  │  - DocumentEditor          │  │ │ GatewaySidecar     │   │
│  │  - ResultsList (virtual)   │  │ │ (tokio::process)   │   │
│  │  - GroupFeed (async)       │  │ │                    │   │
│  │                            │  │ │ FileIndexer        │   │
│  │  Stores:                   │  │ │ (background)       │   │
│  │  - searchResults           │  │ │                    │   │
│  │  - userSession             │  │ │ IpcBridge          │   │
│  │  - documentMetadata        │  │ │ (Tauri commands)   │   │
│  │  - groupActivity           │  │ └──────────────────┘    │
│  │                            │                              │
│  └────────────────────────────┘                              │
│          ▲                                                    │
│          │ Tauri IPC Commands & Events                      │
│          ▼                                                    │
└──────────────────────────────────────────────────────────────┘
         ▲                      ▲                    ▲
         │                      │                    │
         │                      │                    │
    ┌────┴─────┐        ┌───────┴────────┐    ┌────┴────────┐
    │ Gateway   │        │ Cloud Sync     │    │ MCP Apps    │
    │ (Go v1.0)│        │ (S3/DB, v1.1)  │    │ (Phase 4)   │
    │           │        │                │    │             │
    │ - Chat    │        │ - Versioning   │    │ - PDF.js    │
    │ - Memory  │        │ - Sharing      │    │ - SEO panel │
    │ - Tools   │        │ - Permissions  │    │ - Compiler  │
    └───────────┘        └────────────────┘    │             │
         ▲                                       │ Bundled    │
         │                                       │ HTML       │
    ┌────┴──────────────────┐                  └─────────────┘
    │  ZenSci Servers       │
    │  (TypeScript MCP)     │
    │                       │
    │ - latex-mcp (PDF)     │
    │ - blog-mcp (HTML)     │
    │ - grant-mcp (LaTeX)   │
    │ - slides-mcp (reveal) │
    │ - newsletter-mcp      │
    │ - paper-mcp (IEEE)    │
    └───────────────────────┘
```

### Data Flow: User Creates a Document

```
User clicks "New Document" in SvelteKit UI
        ↓
SvelteKit opens DocumentEditor component
        ↓
User types markdown in editor
        ↓
User clicks "Convert to PDF" button
        ↓
SvelteKit invokes Tauri IPC: invoke("convert_document", {source, format: "pdf"})
        ↓
Rust backend receives command
        ↓
Rust routes request to Gateway (via sidecar HTTP)
        ↓
Gateway routes to latex-mcp (MCP tool call)
        ↓
latex-mcp processes markdown → LaTeX → PDF (via pandoc + pdflatex)
        ↓
PDF returned to Gateway, then to Rust backend
        ↓
Rust backend:
  - Writes PDF to local folder (~/.zen-sci/outputs/)
  - Triggers Tantivy index update (background task)
  - Emits Tauri event: "conversion_complete" to SvelteKit
        ↓
SvelteKit receives event, updates store with new document metadata
        ↓
UI shows "PDF ready" with preview (PDF.js iframe if Phase 4 app available)
        ↓
[Background task continues: Tantivy indexes PDF content + metadata]
```

### Data Flow: User Searches

```
User types "photosynthesis mechanism" in SearchBar
        ↓
SvelteKit debounces input (250ms), invokes Tauri IPC: invoke("search", {query, limit: 50})
        ↓
Rust backend queries local Tantivy index
        ↓
Tantivy returns 50 results, sorted by relevance (BM25 score)
        ↓
Each result includes: document ID, title, snippet, file type (PDF/HTML/email/etc), relevance score
        ↓
Rust returns JSON array to SvelteKit
        ↓
SvelteKit populates ResultsList component (virtual scroller for 50 items)
        ↓
User clicks a result → SvelteKit fetches full document from local disk
        ↓
Document opens in editor or preview (PDF.js for PDFs, HTML iframe for blogs, etc.)
```

### Data Flow: User Logs In (First Time)

```
App launches → SvelteKit checks Rust for session state
        ↓
invoke("check_session", {})
        ↓
Rust backend:
  - Checks platform keychain for JWT token
  - If found: validates token (jsonwebtoken crate)
  - If valid: returns {authenticated: true, user: {...}}
  - If not found or invalid: returns {authenticated: false}
        ↓
SvelteKit receives response
        ↓
If authenticated: skip login, show /workspace
If not: show /auth login form
        ↓
User enters email + password, clicks "Login"
        ↓
SvelteKit invokes invoke("login", {email, password})
        ↓
Rust backend:
  - Sends credentials to Gateway (/v1/gateway/auth/login endpoint)
  - Gateway validates, returns JWT + refresh token
  - Rust stores JWT in platform keychain (encrypted at rest)
  - Rust returns {success: true, user: {...}} to SvelteKit
        ↓
SvelteKit stores minimal session state in memory (not localStorage)
        ↓
User logged in; workspace available
```

---

### 3.2 Tauri Application Structure

```
zen-sci-portal/
├── package.json                    # SvelteKit + Tauri frontend
├── tsconfig.json
├── vite.config.ts
├── svelte.config.js
│
├── src/                            # SvelteKit frontend code
│   ├── routes/
│   │   ├── +layout.svelte         # App shell (header, sidebar)
│   │   ├── +page.svelte           # Landing / workspace
│   │   ├── auth/
│   │   │   └── +page.svelte       # Login/signup flow
│   │   ├── workspace/
│   │   │   ├── +page.svelte       # File browser, recent docs
│   │   │   └── [docId]/
│   │   │       └── +page.svelte   # Document editor
│   │   ├── search/
│   │   │   └── +page.svelte       # Search results
│   │   ├── settings/
│   │   │   └── +page.svelte       # User preferences, auth settings
│   │   └── app/
│   │       └── [appId]/           # MCP App iframe route (Phase 4)
│   │           └── +page.svelte
│   │
│   ├── lib/
│   │   ├── components/
│   │   │   ├── SearchBar.svelte
│   │   │   ├── DocumentEditor.svelte
│   │   │   ├── ResultsList.svelte
│   │   │   ├── GroupFeed.svelte
│   │   │   └── ...
│   │   │
│   │   ├── stores/
│   │   │   ├── searchResults.ts   # Svelte store for search results
│   │   │   ├── userSession.ts     # Current user session
│   │   │   ├── documents.ts       # Workspace documents
│   │   │   └── groupActivity.ts   # Activity feed (v1.1)
│   │   │
│   │   ├── api/
│   │   │   └── tauri.ts           # IPC client wrapper
│   │   │       ├── search(query)
│   │   │       ├── convertDocument(source, format)
│   │   │       ├── login(email, password)
│   │   │       ├── logout()
│   │   │       ├── checkSession()
│   │   │       ├── listDocuments()
│   │   │       ├── getGatewayStatus()
│   │   │       └── ...
│   │   │
│   │   └── types/
│   │       └── index.ts           # TypeScript interfaces
│   │
│   └── app.css                    # Global styles (TailwindCSS)
│
├── src-tauri/                     # Tauri Rust backend
│   ├── src/
│   │   ├── main.rs                # App entry point
│   │   ├── lib.rs                 # Library root
│   │   │
│   │   ├── search/
│   │   │   ├── mod.rs
│   │   │   ├── indexer.rs         # Tantivy indexing logic
│   │   │   ├── searcher.rs        # Query execution
│   │   │   └── schema.rs          # Tantivy schema definition
│   │   │
│   │   ├── auth/
│   │   │   ├── mod.rs
│   │   │   ├── manager.rs         # JWT validation, keychain ops
│   │   │   ├── keychain.rs        # Platform-specific keychain bindings
│   │   │   └── session.rs         # Session state management
│   │   │
│   │   ├── gateway/
│   │   │   ├── mod.rs
│   │   │   ├── sidecar.rs         # Spawn, health check, restart logic
│   │   │   ├── client.rs          # HTTP client to Gateway (reqwest)
│   │   │   └── config.rs          # Sidecar configuration
│   │   │
│   │   ├── file/
│   │   │   ├── mod.rs
│   │   │   ├── watcher.rs         # notify crate integration
│   │   │   ├── sync.rs            # File sync logic (v1.1)
│   │   │   └── metadata.rs        # Track local files
│   │   │
│   │   ├── commands/
│   │   │   ├── mod.rs             # IPC command definitions
│   │   │   ├── search.rs          # #[tauri::command] search(...)
│   │   │   ├── convert.rs         # #[tauri::command] convert_document(...)
│   │   │   ├── auth.rs            # #[tauri::command] login(...), logout(), check_session(...)
│   │   │   ├── document.rs        # #[tauri::command] list_documents(...), create_document(...)
│   │   │   ├── gateway.rs         # #[tauri::command] get_gateway_status()
│   │   │   └── app.rs             # #[tauri::command] get_mcp_app(appId)
│   │   │
│   │   ├── models/
│   │   │   ├── mod.rs
│   │   │   ├── document.rs        # Document struct
│   │   │   ├── search_result.rs   # SearchResult struct
│   │   │   ├── user.rs            # User struct
│   │   │   └── gateway_response.rs # Gateway response types
│   │   │
│   │   ├── error.rs               # Error types (anyhow / custom)
│   │   ├── config.rs              # App configuration
│   │   └── state.rs               # AppState struct
│   │
│   ├── Cargo.toml                 # Rust dependencies
│   ├── tauri.conf.json            # Tauri configuration
│   └── icons/                     # App icons (PNG, ICNS, ICO)
│
├── build/                         # Build artifacts (auto-generated)
├── dist/                          # SvelteKit SSG output (static assets)
│
└── README.md
```

### Key Tauri Configuration (`src-tauri/tauri.conf.json`)

```json
{
  "productName": "zen-sci-portal",
  "version": "0.1.0",
  "identifier": "com.trespies.zen-sci-portal",
  "build": {
    "beforeDevCommand": "npm run dev",
    "devUrl": "http://localhost:5173",
    "frontendDist": "../dist"
  },
  "app": {
    "windows": [
      {
        "title": "zen-sci-portal",
        "width": 1200,
        "height": 800,
        "minWidth": 800,
        "minHeight": 600,
        "resizable": true,
        "decorations": true,
        "fullscreen": false
      }
    ],
    "security": {
      "csp": "default-src 'self' ui:// data: https:; script-src 'self' 'wasm-unsafe-eval'; style-src 'self' 'unsafe-inline' ui://; img-src 'self' data: https: ui://; font-src 'self' data: ui://",
      "dangerousRemoteDomainInjectionPolicy": ["ui://"]
    }
  },
  "systemTray": {
    "iconPath": "icons/tray-icon.png",
    "iconAsTemplate": true,
    "menuOnLeftClick": false,
    "tooltip": "zen-sci-portal"
  },
  "bundle": {
    "active": true,
    "targets": ["msi", "dmg", "appimage", "deb"],
    "deb": {
      "depends": ["libgtk-3-0", "libwebkit2gtk-4.0-0"]
    },
    "macOS": {
      "signingIdentity": "Developer ID Application",
      "entitlements": null
    }
  },
  "plugins": {
    "shell": {
      "open": true
    },
    "websocket": {
      "all": {
        "allowHost": ["localhost", "127.0.0.1", "*.local"]
      }
    }
  },
  "security": {
    "csp": {
      "default-src": ["'self'", "ui://", "data:", "https:"],
      "script-src": ["'self'", "'wasm-unsafe-eval'"],
      "style-src": ["'self'", "'unsafe-inline'", "ui://"],
      "img-src": ["'self'", "data:", "https:", "ui://"],
      "font-src": ["'self'", "data:", "ui://"]
    }
  },
  "externalBin": [
    {
      "name": "agentic-gateway",
      "path": "binaries/agentic-gateway",
      "versions": ["1.0.0"],
      "windows": [
        {
          "path": "binaries/agentic-gateway-x86_64-pc-windows-msvc.exe"
        }
      ],
      "linux": [
        {
          "path": "binaries/agentic-gateway-x86_64-unknown-linux-gnu"
        }
      ],
      "macOS": [
        {
          "path": "binaries/agentic-gateway-aarch64-apple-darwin"
        },
        {
          "path": "binaries/agentic-gateway-x86_64-apple-darwin"
        }
      ]
    }
  ]
}
```

### Package.json (Frontend)

```json
{
  "name": "zen-sci-portal",
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "tauri": "tauri",
    "tauri-dev": "tauri dev",
    "tauri-build": "tauri build",
    "test": "vitest",
    "test:e2e": "playwright test"
  },
  "dependencies": {
    "@tauri-apps/api": "^2.0.0",
    "@tauri-apps/plugin-websocket": "^2.0.0",
    "svelte": "^5.0.0",
    "sveltekit": "^2.0.0",
    "tailwindcss": "^3.4.0",
    "typescript": "^5.3.0"
  },
  "devDependencies": {
    "@sveltejs/adapter-static": "^3.0.0",
    "@tauri-apps/cli": "^2.0.0",
    "vite": "^5.0.0"
  }
}
```

### Cargo.toml (Rust Backend)

```toml
[package]
name = "zen-sci-portal"
version = "0.1.0"
edition = "2021"
rust-version = "1.75"

[dependencies]
tauri = { version = "2", features = ["shell-sidecar", "shell-open", "protocol-asset", "http-client", "window-create"] }
tokio = { version = "1.40", features = ["full"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
tantivy = "0.22"
notify = "5.2"
jsonwebtoken = "9"
reqwest = { version = "0.12", features = ["json", "rustls-tls"] }
rusqlite = { version = "0.32", features = ["bundled"] }
tokio-rusqlite = "0.6"
anyhow = "1.0"
thiserror = "1.0"
log = "0.4"
env_logger = "0.11"
uuid = { version = "1.0", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
dashmap = "5.5"
async-trait = "0.1"

[target.'cfg(target_os = "macos")'.dependencies]
security-framework = "2.9"

[target.'cfg(target_os = "windows")'.dependencies]
windows-credentials = "0.3"

[target.'cfg(target_os = "linux")'.dependencies]
secret-service = "3.0"
zeroize = "1.7"

[profile.release]
opt-level = 3
lto = true
codegen-units = 1
```

---

### 3.3 Rust Backend Layer

The Rust backend (Tauri core) has **four key modules**, each with clearly defined scope and API:

#### 3.3.1 SearchIndexer (Tantivy)

**Purpose:** Index all user document outputs locally. Provide sub-50ms search.

**Dependencies:** `tantivy`, `tokio`, `serde`, `dashmap`

**API Surface:**

```rust
// src-tauri/src/search/mod.rs

pub struct SearchIndexer {
    index: Arc<Mutex<Index>>,
    writer: Arc<Mutex<IndexWriter>>,
    searcher: Searcher,
}

impl SearchIndexer {
    /// Initialize index with schema
    pub async fn new(index_dir: &Path) -> Result<Self>;

    /// Index a document (PDF, HTML, email, etc.)
    pub async fn index_document(
        &self,
        doc_id: &str,
        content: &str,
        metadata: DocumentMetadata,
    ) -> Result<()>;

    /// Search documents by query
    pub async fn search(
        &self,
        query: &str,
        limit: usize,
        offset: usize,
    ) -> Result<SearchResults>;

    /// Remove document from index
    pub async fn delete_document(&self, doc_id: &str) -> Result<()>;

    /// Reindex all documents (background maintenance)
    pub async fn reindex_all(&self) -> Result<u64>;
}

pub struct SearchResults {
    pub total_hits: u64,
    pub results: Vec<SearchResult>,
    pub query_time_ms: u128,
}

pub struct SearchResult {
    pub doc_id: String,
    pub title: String,
    pub snippet: String,
    pub score: f32,
    pub file_type: String,  // "pdf", "html", "email", "slide", etc.
    pub created_at: String,
}
```

**Implementation Details:**
- **Schema:** Fields = [doc_id (String, stored), title (Text, stored), content (Text), metadata (JSON, stored), file_type (Facet), created_at (Date)]
- **Tokenizer:** Standard tokenizer + Latin stemming (17 language stems)
- **Scoring:** BM25 (default Tantivy scoring)
- **Concurrency:** Multiple searchers can query in parallel; single IndexWriter for mutations (background task)
- **Persistence:** Index stored in `~/.zen-sci/search-index/` with WAL durability

#### 3.3.2 AuthManager (JWT + Keychain)

**Purpose:** JWT validation, session management, credential storage in platform keychain.

**Dependencies:** `jsonwebtoken`, `security-framework` (macOS), `windows-credentials` (Windows), `secret-service` (Linux), `ring` (crypto)

**API Surface:**

```rust
// src-tauri/src/auth/mod.rs

pub struct AuthManager {
    jwt_secret: String,
    keychain: PlatformKeychain,
    session_store: Arc<Mutex<SessionState>>,
}

impl AuthManager {
    /// Initialize auth manager
    pub async fn new(jwt_secret: &str) -> Result<Self>;

    /// Validate JWT token
    pub fn validate_token(&self, token: &str) -> Result<TokenClaims>;

    /// Check if user session exists in keychain
    pub async fn get_cached_session(&self) -> Result<Option<StoredSession>>;

    /// Store JWT in platform keychain
    pub async fn store_session(
        &self,
        token: &str,
        refresh_token: &str,
        user_id: &str,
        email: &str,
    ) -> Result<()>;

    /// Retrieve session from keychain
    pub async fn retrieve_session(&self) -> Result<Option<StoredSession>>;

    /// Clear session from keychain (logout)
    pub async fn clear_session(&self) -> Result<()>;

    /// Refresh expired token (call Gateway)
    pub async fn refresh_token(&self, refresh_token: &str) -> Result<String>;
}

pub struct TokenClaims {
    pub sub: String,        // user ID
    pub email: String,
    pub iat: i64,           // issued at
    pub exp: i64,           // expiration
}

pub struct StoredSession {
    pub token: String,
    pub refresh_token: String,
    pub user_id: String,
    pub email: String,
    pub expires_at: i64,
}
```

**Implementation Details:**
- **Platform-specific:** Use `security-framework` on macOS (Keychain), `windows-credentials` on Windows (DPAPI), `secret-service` on Linux (D-Bus Secret Service)
- **Keychain service name:** `com.trespies.zen-sci-portal`
- **Keychain account:** `{user_email}`
- **Token storage:** JWT stored as UTF-8 string in keychain (platform encrypts at rest)
- **In-memory session:** Minimal session state cached in `SessionState` (user_id, email, token expiry). No passwords stored.
- **Token rotation:** If token expires and refresh fails → user logged out; next login shows auth form

#### 3.3.3 FileWatcher (notify)

**Purpose:** Monitor `~/.zen-sci/` folder for document changes. Trigger conversions on file save.

**Dependencies:** `notify`, `async-fs`, `tokio::fs`, `ignore`

**API Surface:**

```rust
// src-tauri/src/file/mod.rs

pub struct FileWatcher {
    watcher: RecommendedWatcher,
    watch_dir: PathBuf,
    event_sender: mpsc::Sender<FileEvent>,
}

impl FileWatcher {
    /// Start watching a directory for changes
    pub async fn new(watch_dir: &Path) -> Result<Self>;

    /// Handle file events (called by notify internally)
    async fn handle_file_event(&self, event: notify::Event) -> Result<()>;

    /// List documents in watch directory
    pub async fn list_documents(&self) -> Result<Vec<Document>>;

    /// Watch for specific file type changes (e.g., .md files only)
    pub async fn watch_extensions(&self, extensions: &[&str]) -> Result<()>;
}

pub enum FileEvent {
    Created { path: PathBuf, timestamp: i64 },
    Modified { path: PathBuf, timestamp: i64 },
    Deleted { path: PathBuf, timestamp: i64 },
    Renamed { from: PathBuf, to: PathBuf },
}

pub struct Document {
    pub id: String,
    pub path: PathBuf,
    pub name: String,
    pub ext: String,
    pub size_bytes: u64,
    pub modified_at: i64,
}
```

**Implementation Details:**
- **Watch directory:** `~/.zen-sci/` (platform-specific via `dirs` crate)
- **Ignored patterns:** `.git/`, `node_modules/`, `target/`, `.DS_Store`, etc. (via `ignore` crate)
- **Event debouncing:** Coalesce multiple writes to same file within 500ms
- **Async runtime:** Tokio-based; file events posted to channel for background processing
- **Conversion trigger:** When `.md` file saved → queue for ZenSci conversion

#### 3.3.4 GatewaySidecar (tokio::process)

**Purpose:** Bundle, launch, health-check, and manage AgenticGateway subprocess.

**Dependencies:** `tokio`, `reqwest`, `serde_json`

**API Surface:**

```rust
// src-tauri/src/gateway/mod.rs

pub struct GatewaySidecar {
    process: Child,
    base_url: String,
    health_check_interval: Duration,
}

impl GatewaySidecar {
    /// Spawn AgenticGateway subprocess
    pub async fn spawn(config: SidecarConfig) -> Result<Self>;

    /// Check if Gateway is healthy (GET /health)
    pub async fn health_check(&self) -> Result<HealthStatus>;

    /// Start background health monitoring (auto-restart on failure)
    pub async fn start_health_monitor(self: Arc<Self>) -> Result<()>;

    /// Send request to Gateway (HTTP client)
    pub async fn call_tool(
        &self,
        tool_name: &str,
        args: serde_json::Value,
    ) -> Result<serde_json::Value>;

    /// List available tools from Gateway
    pub async fn list_tools(&self) -> Result<Vec<ToolDefinition>>;

    /// Shutdown Gateway gracefully
    pub async fn shutdown(&mut self) -> Result<()>;
}

pub struct SidecarConfig {
    pub binary_path: String,
    pub port: u16,
    pub mcp_config_path: PathBuf,
}

pub struct HealthStatus {
    pub healthy: bool,
    pub uptime_ms: u64,
    pub mcp_servers_connected: usize,
}
```

**Implementation Details:**
- **Binary bundling:** Tauri `externalBin` config downloads correct binary per platform (Windows MSVC, Linux GNU, macOS Universal)
- **Launch:** `shell().sidecar("agentic-gateway")` with config file path
- **Port selection:** Default 8080; fallback to 8081–8090 if in use
- **Health check:** Every 10 seconds via GET http://localhost:8080/health
- **Restart logic:** If unhealthy for >3 checks, kill process and restart with exponential backoff (1s, 2s, 4s, max 30s)
- **Shutdown:** On app exit, gracefully close Gateway process (SIGTERM, wait 5s, SIGKILL)

---

### 3.4 SvelteKit Frontend

**Framework:** SvelteKit v2.0+ with Svelte 5 (fine-grained reactivity, minimal overhead for Tauri IPC)

**Routing:** File-based routing in `src/routes/`

**State Management:** Svelte stores (`svelte/store`) for reactive state

**Key Routes:**

| Route | Purpose | Components |
|-------|---------|-----------|
| `/` | Landing/workspace root | WorkspaceBrowser, RecentDocuments |
| `/auth` | Login/signup form | LoginForm, SignupForm |
| `/workspace` | Document folder browser | FolderTree, DocumentList |
| `/workspace/[docId]` | Document editor | DocumentEditor, Preview, MCP App frame |
| `/search?q=...` | Search results | SearchBar, ResultsList (virtual scroller) |
| `/settings` | User preferences & auth | AuthSettings, IndexSettings, Sync Status |
| `/app/[appId]` | MCP App iframe container | AppBridge (Phase 4) |

**Key Stores:**

```typescript
// src/lib/stores/userSession.ts
export const userSession = writable<UserSession | null>(null);

interface UserSession {
  userId: string;
  email: string;
  tokenExpiresAt: number;
  authenticated: boolean;
}

// src/lib/stores/searchResults.ts
export const searchResults = writable<SearchResults>(null);
export const searchQuery = writable<string>("");

interface SearchResults {
  totalHits: number;
  results: SearchResult[];
  queryTimeMs: number;
  hasMore: boolean;
}

// src/lib/stores/documents.ts
export const documents = writable<Document[]>([]);
export const currentDocument = writable<Document | null>(null);

interface Document {
  id: string;
  name: string;
  path: string;
  ext: string;
  modifiedAt: number;
  sizeBytes: number;
  outputFormats: string[];  // "pdf", "html", "email", "slide", etc.
}
```

**Key Components:**

| Component | Purpose |
|-----------|---------|
| **SearchBar** | Input field with debounced search. Triggers Tauri IPC search command. |
| **DocumentEditor** | Monaco editor instance. Syntax highlighting for markdown. Auto-save to local disk. |
| **ResultsList** | Virtual scroller (svelte-virtualizer) for 50+ results. Click → open document. |
| **GroupFeed** | Activity feed showing recent docs, shared updates (v1.0 = local; v1.1 = cloud). |
| **AppBridge** | iframe container for Phase 4 MCP Apps. Handles messaging + CSP sandboxing. |

**Tauri IPC Wrapper:**

```typescript
// src/lib/api/tauri.ts
import { invoke } from "@tauri-apps/api/tauri";

export async function search(query: string, limit: number = 50) {
  return invoke<SearchResults>("search", { query, limit });
}

export async function convertDocument(source: string, format: string) {
  return invoke<ConversionResult>("convert_document", { source, format });
}

export async function login(email: string, password: string) {
  return invoke<LoginResult>("login", { email, password });
}

export async function logout() {
  return invoke<void>("logout", {});
}

export async function checkSession() {
  return invoke<SessionStatus>("check_session", {});
}

export async function listDocuments(folder?: string) {
  return invoke<Document[]>("list_documents", { folder });
}

export async function getGatewayStatus() {
  return invoke<GatewayStatus>("get_gateway_status", {});
}

export async function getMCPApp(appId: string) {
  return invoke<AppBundle>("get_mcp_app", { appId });
}
```

---

### 3.5 AgenticGateway Integration

**Bundling:** Tauri `externalBin` config (see Section 3.2) bundles the Go binary for all platforms.

**Launch:** On app startup, Rust backend spawns Gateway subprocess via Tauri sidecar. If Gateway crashes, auto-restart.

**HTTP Communication:**
- Frontend (SvelteKit) never talks to Gateway directly
- All Gateway requests go through Rust backend (Tauri commands)
- Rust backend makes HTTP calls to Gateway at `http://localhost:8080`

**Example: Document Conversion Flow**

```
1. User clicks "Convert to PDF" in SvelteKit UI
   ↓
2. SvelteKit calls: invoke("convert_document", { source: "# Title\n...", format: "pdf" })
   ↓
3. Rust backend receives command, calls Gateway:
   POST http://localhost:8080/v1/gateway/tools/call
   {
     "tool": "latex_mcp:convert_to_pdf",
     "arguments": {
       "source": "# Title\n...",
       "bibliography_style": "ieee"
     }
   }
   ↓
4. Gateway routes to latex-mcp MCP server (via MCPHostManager)
   ↓
5. latex-mcp processes markdown → LaTeX → PDF
   ↓
6. PDF returned to Gateway, then to Rust backend
   ↓
7. Rust backend:
   - Writes PDF to ~/.zen-sci/outputs/
   - Queues Tantivy index update
   - Returns path to SvelteKit
   ↓
8. SvelteKit shows PDF preview
```

---

### 3.6 Tauri IPC Commands

**All commands are defined in `src-tauri/src/commands/` and registered in `src-tauri/src/main.rs`.**

| Command | Input | Output | Purpose |
|---------|-------|--------|---------|
| `search` | `{query: string, limit: number}` | `SearchResults` | Query local Tantivy index |
| `convert_document` | `{source: string, format: string, ...options}` | `ConversionResult` | Route to Gateway, convert markdown to target format |
| `login` | `{email: string, password: string}` | `{success: bool, user: User}` | Authenticate to Gateway, store JWT in keychain |
| `logout` | `{}` | `{success: bool}` | Clear JWT from keychain |
| `check_session` | `{}` | `SessionStatus` | Check if valid JWT in keychain |
| `list_documents` | `{folder?: string}` | `Document[]` | List markdown files in watch directory |
| `create_document` | `{name: string, format: string}` | `{doc_id: string, path: string}` | Create new markdown file |
| `get_gateway_status` | `{}` | `GatewayStatus` | Check if sidecar healthy; return uptime + tool count |
| `get_mcp_app` | `{appId: string}` | `AppBundle` | Load Phase 4 MCP App HTML (for iframe) |
| `watch_folder` | `{path: string}` | `{success: bool}` | Start file watcher on folder |
| `stop_watcher` | `{}` | `{success: bool}` | Stop file watcher |
| `index_reindex` | `{}` | `{indexed: number, duration_ms: number}` | Rebuild search index |

---

### 3.7 Data Models

**Rust Structs:**

```rust
// src-tauri/src/models/document.rs
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Document {
    pub id: String,
    pub name: String,
    pub path: String,
    pub ext: String,
    pub size_bytes: u64,
    pub modified_at: i64,
    pub output_formats: Vec<String>,
}

// src-tauri/src/models/search_result.rs
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SearchResult {
    pub doc_id: String,
    pub title: String,
    pub snippet: String,
    pub score: f32,
    pub file_type: String,
    pub created_at: String,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct SearchResults {
    pub total_hits: u64,
    pub results: Vec<SearchResult>,
    pub query_time_ms: u128,
    pub has_more: bool,
}

// src-tauri/src/models/user.rs
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct User {
    pub id: String,
    pub email: String,
    pub name: Option<String>,
}

// src-tauri/src/models/gateway_response.rs
#[derive(Debug, Serialize, Deserialize)]
pub struct ConversionResult {
    pub success: bool,
    pub output_path: Option<String>,
    pub output_base64: Option<String>,
    pub error: Option<String>,
    pub warnings: Vec<String>,
    pub elapsed_ms: u128,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct GatewayStatus {
    pub healthy: bool,
    pub uptime_ms: u64,
    pub version: String,
    pub mcp_servers_connected: usize,
    pub tools_available: usize,
}
```

**TypeScript Interfaces:**

```typescript
// src/lib/types/index.ts
export interface User {
  id: string;
  email: string;
  name?: string;
}

export interface Document {
  id: string;
  name: string;
  path: string;
  ext: string;
  sizeBytes: number;
  modifiedAt: number;
  outputFormats: string[];
}

export interface SearchResult {
  docId: string;
  title: string;
  snippet: string;
  score: number;
  fileType: string;
  createdAt: string;
}

export interface ConversionResult {
  success: boolean;
  outputPath?: string;
  outputBase64?: string;
  error?: string;
  warnings: string[];
  elapsedMs: number;
}

export interface GatewayStatus {
  healthy: boolean;
  uptimeMs: number;
  version: string;
  mcpServersConnected: number;
  toolsAvailable: number;
}
```

---

## 4. Implementation Plan

### Phase 1 (Week 1–2): Foundation
**Deliverables:** Tauri scaffold + AgenticGateway sidecar integration + Auth layer

- [ ] Initialize Tauri v2 project with SvelteKit static adapter
- [ ] Set up Rust backend directory structure (search, auth, gateway, file modules)
- [ ] Implement GatewaySidecar: spawn, health check, restart logic
- [ ] Implement AuthManager: JWT validation, keychain storage (platform-specific)
- [ ] Create core Tauri IPC commands: `login`, `logout`, `check_session`, `get_gateway_status`
- [ ] Build login/logout UI in SvelteKit; test credential flow
- [ ] Write integration tests for auth + sidecar startup
- [ ] **Success criteria:** App launches, spins up Gateway, user can login/logout

### Phase 2 (Week 3–4): Search & Document Management
**Deliverables:** Tantivy search index + document browsing

- [ ] Implement SearchIndexer: Tantivy schema, indexing, querying
- [ ] Implement FileWatcher: monitor `~/.zen-sci/` for markdown files
- [ ] Create `search` and `list_documents` Tauri commands
- [ ] Build SearchBar + ResultsList UI components (virtual scroller)
- [ ] Integrate search with file watcher: new files automatically indexed
- [ ] Write performance tests: <50ms search on 1000 documents
- [ ] **Success criteria:** User can type search query, see results in <100ms

### Phase 3 (Week 5–6): Document Conversion & MCP Integration
**Deliverables:** SvelteKit editor + gateway routing + Phase 4 app iframe support

- [ ] Build DocumentEditor component (Monaco editor with markdown syntax highlighting)
- [ ] Implement `convert_document` command: route to Gateway, handle response
- [ ] Create conversion UI: button states, progress indicator, error messages
- [ ] Implement `get_mcp_app` command for Phase 4 MCP App loading
- [ ] Build AppBridge component: iframe sandboxing, CSP, message passing
- [ ] Write E2E tests: create doc → convert to PDF/HTML/slides → preview
- [ ] **Success criteria:** User can edit markdown, click "Convert to PDF", receive PDF in <5s

### Phase 4 (Week 7–8): Optimization & E2E Testing
**Deliverables:** Performance tuning, comprehensive tests, release-ready build

- [ ] Profile search latency, optimize Tantivy schema if needed
- [ ] Optimize IPC message size (batch requests, streaming for large responses)
- [ ] Implement offline mode: search works when Gateway unavailable
- [ ] Write comprehensive E2E tests (Playwright): full user workflows
- [ ] Build production release: sign macOS/Windows, notarize, create installers
- [ ] Write user documentation: quick start, troubleshooting, architecture guide
- [ ] **Success criteria:** All tests passing, app <50MB bundle, <0.5s startup

---

## 5. Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| **Tauri sidecar multi-platform complexity** | Medium | High | Test cross-platform sidecar bundling early (Week 1). Use GitHub Actions matrix for CI. |
| **SvelteKit CSP for MCP App iframes** | Medium | Medium | Define restrictive CSP in `tauri.conf.json`. Test iframe messaging with sample Phase 4 app. Allow `ui://` protocol in CSP. |
| **Keychain integration platform-specific failures** | Medium | High | Test on macOS, Windows, Linux early. Implement encrypted file fallback if keychain unavailable. |
| **Tantivy index out-of-sync with Gateway storage** | Low | High | Design index as read-only replica. Reindex on app startup if metadata changed. Tag index version in config. |
| **Tauri IPC becomes bottleneck for high-frequency updates** | Medium | Medium | Use WebSocket plugin for real-time updates (v1.1). Batch IPC calls. Profile with 1000+ documents. |
| **Search latency degrades on 10K+ documents** | Low | Medium | Tantivy scales well to 100K docs. Implement pagination + lazy loading in UI. Monitor query times in telemetry. |
| **User expectations for "cloud" not met** | Medium | High | Clear UX messaging: local folder icon vs. cloud icon. Show sync status. Document v1.0 scope (local-first, no cloud yet). |
| **File watcher misses changes on network shares** | Low | Medium | Warn user if watch folder is remote. Implement fallback polling if notify event drops. |
| **AgenticGateway crashes during doc conversion** | Medium | Medium | Implement graceful fallback: show "Gateway offline" error. Retry with exponential backoff. Queue conversions for retry. |
| **Installer bloat for multi-arch binaries** | Low | Medium | Use Goreleaser to build slim binaries. Separate Windows MSI + macOS DMG. Use AppImage for Linux. |

---

## 6. Rollback & Contingency

**Feature Flags:**
```rust
// src-tauri/src/config.rs
pub const FEATURE_SEARCH: bool = true;     // Tantivy indexing
pub const FEATURE_AUTH: bool = true;       // JWT + keychain
pub const FEATURE_FILE_WATCH: bool = true; // File watcher (v1.1 flag)
pub const FEATURE_SYNC: bool = false;      // Cloud sync (v1.1 flag, disabled in v1.0)
```

**Rollback Procedures:**

- **Search index corrupted:** Delete `~/.zen-sci/search-index/`, app auto-reindexes on startup
- **Auth token expired:** User logs out, logs back in. Refresh token mechanism in place.
- **Gateway sidecar won't start:** App shows "Gateway offline" error. User can still browse local documents + search.
- **MCP App iframe fails to load:** Fallback UI message. Document still accessible in editor view.
- **File watcher drops events:** App can manual reindex via `index_reindex` command in settings.

---

## 7. Documentation & Communication

**User Documentation:**
- Quick start guide (5 minutes to first PDF)
- Architecture overview (local-first, offline-capable)
- Troubleshooting (Gateway not starting, search slow, keychain issues)
- FAQ (What's stored locally? Can I share documents? Is my data encrypted?)

**Developer Documentation:**
- Rust backend architecture (modules, IPC contract)
- SvelteKit component guide (stores, routing, Tauri bindings)
- Contribution guidelines (testing, code style, release process)

**Release Communication:**
- Changelog: features, bug fixes, breaking changes
- Migration guide: how to upgrade from earlier versions
- Known limitations: v1.0 does not include cloud sync (v1.1+), real-time collab (v1.1+), mobile (v2.0+)

---

## 8. Appendices

### 8.1 Rust Crate Versions (Cargo.toml)

```toml
tauri = "2.x"
tokio = "1.40"
serde = "1.0"
serde_json = "1.0"
tantivy = "0.22"
notify = "5.2"
jsonwebtoken = "9"
reqwest = "0.12"
rusqlite = "0.32"
tokio-rusqlite = "0.6"
anyhow = "1.0"
thiserror = "1.0"
log = "0.4"
env_logger = "0.11"
uuid = "1.0"
chrono = "0.4"
dashmap = "5.5"
async-trait = "0.1"

# Platform-specific
security-framework = "2.9"        # macOS
windows-credentials = "0.3"       # Windows
secret-service = "3.0"            # Linux
zeroize = "1.7"                   # Linux
```

### 8.2 SvelteKit Dependencies (package.json)

```json
{
  "@tauri-apps/api": "^2.0.0",
  "@tauri-apps/plugin-websocket": "^2.0.0",
  "@tauri-apps/cli": "^2.0.0",
  "svelte": "^5.0.0",
  "sveltekit": "^2.0.0",
  "@sveltejs/adapter-static": "^3.0.0",
  "typescript": "^5.3.0",
  "tailwindcss": "^3.4.0",
  "vite": "^5.0.0"
}
```

### 8.3 Future: Phase 1.1 (Real-Time Collaboration & Cloud Sync)

**Not in v0.1 scope, but architecture supports:**

- **Real-time updates:** Rust WebSocket server (tokio-tungstenite) + Automerge CRDT
- **Cloud sync:** S3 file storage + Gateway metadata store
- **Presence awareness:** Who's online, live cursor positions, typing indicators
- **Conflict resolution:** CRDT-based automatic merging

**Entry point:** `src-tauri/src/collab/` module (scaffolded but disabled in v0.1)

### 8.4 Open Questions

1. **File system layout:** Should each document have its own folder, or all in one? → Recommend per-document folders (`~/.zen-sci/{doc-id}/`)

2. **Search indexing latency:** Is <2 second index update acceptable, or need real-time? → v0.1: batch indexing (2s). v1.1: real-time via background tokio task.

3. **Offline auth:** Can user work without internet if they have cached JWT? → Yes. Gateway call fails gracefully; local features (search, edit, preview) all work offline.

4. **Cloud sync priority:** Does v1.0 include opt-in cloud backup? → No. v0.1 is local-first only. v1.1 adds optional cloud sync.

5. **MCP App security:** Should iframes be sandboxed further? → Current CSP is restrictive; `ui://` protocol only allows trusted bundled apps. Further hardening can wait for v1.1.

---

**Specification Prepared By:** TresPiesDesign.com / Cruz Morales
**Date:** 2026-02-18
**Status:** Ready for Implementation
**Estimated Timeline:** 8 weeks (2 months) to v0.1 release
**Next Steps:** Commission implementation prompt for Phase 1 (Week 1–2 foundation work)


---

## Revision R1: Architectural Corrections (2026-02-18)

This revision corrects the framing of AgenticGateway from a sidecar utility to the reasoning engine and orchestration core, updates the document catalog model to include cloud-owned canonical sources with git-like local sync, and introduces collaborative section ownership with draft/structured mode transitions.

### 1. Agentic Workspace Model

#### 1.1 Corrected Architecture: Gateway as Center of Gravity

The previous spec treated AgenticGateway as a document conversion utility (sidecar) for the Rust desktop layer. The corrected framing:

**AgenticGateway is the reasoning engine, session memory, and orchestration core.** The Tauri Rust backend is a sync/cache/credential/search facade in front of the Gateway. Every IPC command from the SvelteKit frontend either:
1. Talks directly to the Gateway via the Rust bridge, or
2. Prepares local data (files, fragments, search results) to send to the Gateway for reasoning

This inversion puts the stateful, agentic, tool-aware logic on the Gateway (where it can coordinate across users, projects, and tools) and leaves the desktop as the responsible owner of local-first search, credential management, and document sync orchestration.

#### 1.2 Architecture Diagram (Text)

```
┌─────────────────────────────────────────────────────────────────┐
│  SvelteKit Frontend (Web UI)                                     │
│  • Project workspace UI                                          │
│  • Document editor with section ownership sidebar                │
│  • Search & library browser                                      │
└─────────────────┬───────────────────────────────────────────────┘
                  │ Tauri IPC (JSON-RPC)
                  ↓
┌─────────────────────────────────────────────────────────────────┐
│  Tauri Rust Backend (Desktop Layer)                              │
│  • JWT credential storage (platform keychain)                    │
│  • Local Tantivy full-text search index                          │
│  • File watching & local change staging                          │
│  • Document sync orchestration (push/pull)                       │
│  • Cache layer for frequently accessed documents                 │
│  • IPC command router                                            │
└─────────────────┬───────────────────────────────────────────────┘
                  │ HTTPS + Bearer Token
                  ↓
┌─────────────────────────────────────────────────────────────────┐
│  AgenticGateway (Reasoning & Orchestration Core)                 │
│  • Named agents per project (LLM-backed)                         │
│  • Agent chat sessions with persistent context                   │
│  • DAG pipeline orchestration (ZenSci workflow execution)        │
│  • MCP tool management & invocation                              │
│  • Semantic memory storage & retrieval                           │
│  • Multi-provider LLM support (Anthropic, Ollama, Kimi, etc.)   │
│  • JWT auth & access control                                     │
│  • Canonical document catalog ownership                          │
└─────────────────┬───────────────────────────────────────────────┘
                  │ gRPC (MCP Server Discovery)
                  ↓
┌─────────────────────────────────────────────────────────────────┐
│  ZenSci 6-Server Suite                                           │
│  • LaTeX compiler, slides generator, PDF tool,                   │
│    email sender, blog publisher, document tools                  │
└─────────────────────────────────────────────────────────────────┘
```

#### 1.3 Gateway API Surface Exposed via Tauri IPC

The following endpoints (from `router.go`) are mapped to Tauri IPC commands:

| Gateway Route | IPC Command | Purpose |
|---|---|---|
| `POST /v1/gateway/agents` | `create_agent(project_id, name, provider)` | Create a named agent for a project |
| `GET /v1/gateway/agents/:id` | `get_agent(agent_id)` | Retrieve agent details |
| `POST /v1/gateway/agents/:id/chat` | `chat_with_agent(agent_id, message, context)` | Stream agent response with context |
| `POST /v1/gateway/orchestrate` | `submit_dag(pipeline_spec)` | Submit a DAG job (e.g., compile + publish) |
| `GET /v1/gateway/orchestrate/:id/dag` | `get_dag_status(job_id)` | Stream DAG progress events |
| `GET /v1/gateway/tools` | `list_tools(namespace?)` | List available MCP tools |
| `POST /v1/memory` | `store_memory(content, tags, project_id)` | Store a memory entry |
| `POST /v1/memory/search` | `search_memory(query, project_id, limit)` | Semantic search over memory |
| `GET /v1/garden/context` | `get_garden_context(session_id)` | Retrieve project context snapshot |
| `GET /v1/providers` | `list_providers()` | Return available LLM providers with status |
| (new endpoint) | `switch_provider(provider_name)` | Set active provider (Ollama vs. Kimi vs. Anthropic) |
| `POST /v1/chat/completions` | `llm_completion(prompt, provider?, model?)` | Direct LLM completion |

#### 1.4 IPC Command Signatures

All commands return either a JSON response or a stream (for chat/DAG progress):

```go
// Agent management
create_agent(project_id: string, name: string, provider: string) -> AgentResponse
get_agent(agent_id: string) -> AgentResponse
chat_with_agent(agent_id: string, message: string, context?: Context) -> Stream<ChatEvent>

// DAG orchestration
submit_dag(pipeline_spec: PipelineSpec) -> JobResponse { id, status }
get_dag_status(job_id: string) -> Stream<DAGProgressEvent>

// Tools
list_tools(namespace?: string) -> ToolList

// Memory
store_memory(content: string, tags: string[], project_id: string) -> MemoryResponse
search_memory(query: string, project_id: string, limit?: number) -> SearchResult[]

// Garden & context
get_garden_context(session_id: string) -> GardenContext

// Providers
list_providers() -> ProviderStatus[]
switch_provider(provider_name: string) -> ProviderStatus

// LLM
llm_completion(prompt: string, provider?: string, model?: string) -> Stream<CompletionEvent>
```

#### 1.5 Error Handling & Gateway Resilience

- If Gateway is unreachable, the Rust backend returns a cached response or a "Gateway offline" error with a retry suggestion
- Chat streams can be resumed by agent_id + session_id
- DAG job IDs are persistent; polling `/orchestrate/:id/dag` continues even if the desktop app restarts

---

### 2. Document Catalog & Sync Model

#### 2.1 Cloud-Owned Canonical Source

The **Gateway owns the canonical document catalog.** Each document has a canonical URL on the Gateway (e.g., `/v1/documents/{id}`). The desktop holds **git-like local clones** of documents.

A document in the Gateway includes:
- Metadata (title, owner, co-authors, mode)
- Full content + history
- Section ownership assignments (if in structured mode)
- CRDT state (reserved for v2; null in v1)

#### 2.2 Document Clone Metadata (Rust Local Store)

Each local clone stores:

```rust
pub struct DocumentClone {
    pub id: Uuid,
    pub gateway_id: Uuid,
    pub title: String,
    pub version_hash: String,          // SHA256(content + sections)
    pub upstream_url: String,          // https://gateway.local/v1/documents/{id}
    pub last_synced_at: DateTime<Utc>,
    pub local_changes_staged: bool,    // true = changes pending push
    pub conflict_state: Option<ConflictInfo>,
    pub sections: Vec<Section>,
}

pub struct ConflictInfo {
    pub local_version_hash: String,
    pub upstream_version_hash: String,
    pub conflicted_sections: Vec<usize>,  // section indices with conflicts
    pub merge_strategy: "manual" | "ai_assisted",
    pub merge_in_progress: bool,
}
```

#### 2.3 Sync Workflow

**Pull (Fetch Upstream Changes)**
1. Desktop calls `get_document(doc_id)` on Gateway
2. Gateway returns current `version_hash`, content, and sections
3. Rust layer compares `upstream_version_hash` with local `version_hash`
4. If hashes match: no-op
5. If local changes staged and upstream changed: **conflict detected** → create ConflictInfo
6. If no local changes: fast-forward local clone
7. If conflict: user prompted with merge UI (see Section 2.4)

**Push (Sync Local Changes)**
1. User clicks "Sync to Cloud"
2. Rust layer verifies no upstream changes since last pull; if changed, pulls first
3. Builds a `PutDocumentRequest` with updated content + sections + new `version_hash`
4. POSTs to Gateway `/v1/documents/{id}` with Bearer token
5. Gateway validates ownership/co-author permissions
6. If successful: updates `last_synced_at`, clears `local_changes_staged`
7. If conflict at Gateway (another user pushed between pull and push): Rust layer detects HTTP 409, triggers merge flow

#### 2.4 Conflict Detection & AI-Assisted Merge

When a conflict is detected:

1. UI shows "Conflict: Section 'Introduction' differs from upstream"
2. User can choose: **Manual Merge** or **AI-Assisted Merge**
3. **Manual Merge**: 3-way diff view (local | base | upstream), drag-to-accept
4. **AI-Assisted Merge**:
   - Rust layer calls `submit_dag` with a merge job:
     ```json
     {
       "name": "merge_conflict",
       "steps": [
         {
           "name": "analyze_conflict",
           "tool": "llm:merge_analyzer",
           "input": {
             "local_section": "...",
             "upstream_section": "...",
             "base_section": "..."
           }
         },
         {
           "name": "generate_merged",
           "tool": "llm:merge_generator",
           "input": {
             "analysis": "${analyze_conflict.output}",
             "style": "preserve_both_perspectives"
           }
         }
       ]
     }
     ```
   - Gateway executes DAG, streams progress to UI
   - Once complete, user reviews merged result and approves/rejects
   - On approval: local clone updated with merged content, ready to push

---

### 3. Collaboration Data Model: Document & Section Schemas

#### 3.1 Document Schema (Gateway)

```go
type Document struct {
    ID              uuid.UUID    `db:"id"`
    Title           string       `db:"title"`
    OwnerID         uuid.UUID    `db:"owner_id"`        // User who created/owns document
    CoAuthors       []uuid.UUID  `db:"co_authors"`      // List of co-author IDs
    Mode            string       `db:"mode"`            // "draft" or "structured"
    Content         string       `db:"content"`         // Full markdown/text content
    VersionHash     string       `db:"version_hash"`    // SHA256(content + sections JSON)
    LastModifiedAt  time.Time    `db:"last_modified_at"`
    CreatedAt       time.Time    `db:"created_at"`
    Sections        []Section    `gorm:"foreignKey:DocumentID"`
    CRDTState       *[]byte      `db:"crdt_state"`      // Null in v1; CRDT byte blob in v2
}
```

#### 3.2 Section Schema (Gateway)

```go
type Section struct {
    ID          uuid.UUID    `db:"id"`
    DocumentID  uuid.UUID    `db:"document_id"`
    Title       string       `db:"title"`
    Content     string       `db:"content"`
    OwnerID     *uuid.UUID   `db:"owner_id"`           // Co-author who owns this section; null if orphaned
    Order       int          `db:"order"`              // Ordinal position in document
    Locked      bool         `db:"locked"`             // true = owner actively editing, others can comment only
    CreatedAt   time.Time    `db:"created_at"`
    UpdatedAt   time.Time    `db:"updated_at"`
}
```

#### 3.3 Rust Local Mirror

The Rust backend mirrors Document and Section schemas locally, adding sync metadata:

```rust
pub struct LocalDocument {
    pub id: Uuid,
    pub gateway_id: Uuid,
    pub title: String,
    pub owner_id: Uuid,
    pub co_authors: Vec<Uuid>,
    pub mode: DocumentMode,           // Draft | Structured
    pub content: String,
    pub version_hash: String,
    pub upstream_url: String,
    pub last_synced_at: DateTime<Utc>,
    pub local_changes_staged: bool,
    pub conflict_state: Option<ConflictInfo>,
    pub sections: Vec<LocalSection>,
}

pub struct LocalSection {
    pub id: Uuid,
    pub title: String,
    pub content: String,
    pub owner_id: Option<Uuid>,        // None if orphaned
    pub order: usize,
    pub locked: bool,
    pub local_changes_staged: bool,
}

pub enum DocumentMode {
    Draft,
    Structured,
}
```

---

### 4. Draft Mode & Section Boundaries

#### 4.1 Mode Overview

Documents have two modes:

| Mode | Behavior | Section Ownership | Use Case |
|---|---|---|---|
| **Draft** | Anyone can edit freely; no section ownership | Not assigned | Early brainstorming, co-authors iterating freely |
| **Structured** | Sections are owned; only owner + comments allowed | Assigned to co-authors | Ready for division of labor, parallel writing |

#### 4.2 Draft → Structured Transition

1. **Document owner initiates transition:**
   - Clicks "Mark Ready for Collaboration" button
   - UI prompts: "Assign sections to co-authors"

2. **Section boundary assignment UI:**
   - Document displayed with section headers highlighted
   - Drag each section header to a co-author name in a sidebar
   - Once all sections assigned, click "Lock Collaboration"
   - Gateway receives `PATCH /v1/documents/{id}` with mode=structured + section assignments

3. **Gateway validates & persists:**
   - Confirms all sections are assigned (no unowned sections)
   - Updates Document.Mode = "structured"
   - Writes Section.OwnerID for each section
   - Broadcasts change to all active co-author sessions

4. **Desktop sync:**
   - Rust layer detects mode change on pull
   - Updates UI: shows section ownership sidebar, enables edit locks
   - Disables free editing in sections owned by other co-authors (read-only mode)

#### 4.3 Reverse Transition: Structured → Draft

Document owner can revert to draft mode:
1. Click "Revert to Free Edit"
2. Confirm: "All section locks will be removed"
3. Gateway receives `PATCH /v1/documents/{id}` with mode=draft, clears all Section.OwnerID
4. Desktop notifies co-authors: "Document is now in free-edit mode"

---

### 5. Orphaned Sections: Detection & Reclamation

#### 5.1 When Sections Become Orphaned

A section becomes orphaned when:
- A co-author is removed from the document's co_authors list
- A co-author leaves a group/project
- A section's owner_id is no longer in the document's co_authors list

#### 5.2 Detection & Notification

**On the Gateway (during sync or authorization check):**
- Before serving a document, validate all Section.OwnerID ⊆ Document.CoAuthors
- If orphaned sections detected, set Section.locked = true + Section.orphaned = true (add field to Section schema)
- Return document with flag: `"has_orphaned_sections": true`

**On the Desktop:**
- Rust layer notifies UI: "3 sections are unowned"
- UI displays badge on document in library
- Clicking document opens "Orphaned Sections" dialog

#### 5.3 Reclamation Flow

**UI Dialog: "Orphaned Sections (3)"**
- List each orphaned section (title, former owner, content preview)
- For each: dropdown to reassign to a co-author, or leave unowned (blocked)
- Button: "Reassign and Save"

**IPC Command:**
```go
reclaim_section(doc_id: Uuid, section_id: Uuid, new_owner_id: Uuid) -> SectionResponse
```

**Gateway Handler:**
1. Validates `new_owner_id` is in Document.CoAuthors
2. Updates Section.OwnerID = new_owner_id
3. If all sections now owned: clears orphaned flag
4. Returns updated Section + Document

**Rust Desktop:**
- Calls `reclaim_section` for each reassigned section
- Pushes changes to Gateway
- Updates UI: removes badge, reloads document view

#### 5.4 Preventing Orphaned Sections

- Gateway prevents removing a co-author if they own sections (error: "Cannot remove co-author with owned sections; reassign sections first")
- Owner must reclaim/reassign before removal

---

### 6. Auth Integration: Gateway JWT Endpoints & Keychain Storage

#### 6.1 New Gateway Auth Endpoints (to be added in v1)

The Gateway middleware (auth.go) already has JWT infrastructure. v1 adds these **public** endpoints (no auth required):

```
POST /auth/register
  request: { email: string, password: string, name: string }
  response: { user_id: uuid, token: string, expires_in: number }

POST /auth/login
  request: { email: string, password: string }
  response: { user_id: uuid, token: string, expires_in: number }

POST /auth/refresh
  request: { refresh_token: string }
  response: { token: string, expires_in: number }
  auth: Bearer (refresh_token)
```

All other Gateway endpoints require `Authorization: Bearer <jwt_token>`.

#### 6.2 Rust Desktop: JWT Storage in Platform Keychain

The Rust backend uses the `keyring` crate to store JWTs securely per-user:

```rust
use keyring::Entry;

pub struct AuthManager {
    service: String,  // "zen-sci-portal"
}

impl AuthManager {
    pub async fn store_token(&self, username: &str, token: &str) -> Result<()> {
        let entry = Entry::new(&self.service, username)?;
        entry.set_password(token)?;
        Ok(())
    }

    pub async fn retrieve_token(&self, username: &str) -> Result<String> {
        let entry = Entry::new(&self.service, username)?;
        Ok(entry.get_password()?)
    }

    pub async fn delete_token(&self, username: &str) -> Result<()> {
        let entry = Entry::new(&self.service, username)?;
        entry.delete_password()?;
        Ok(())
    }
}
```

Tokens are stored in:
- **macOS:** Keychain (~/Library/Keychains/)
- **Windows:** DPAPI-encrypted local store (%APPDATA%\zen-sci-portal\)
- **Linux:** Secret Service (org.freedesktop.secrets)

#### 6.3 Desktop Auth Flow

**Initial Login:**
1. User enters email + password in SvelteKit login form
2. Rust bridge calls `IPC::login(email, password)`
3. Rust layer POSTs to `https://gateway/auth/login` with credentials
4. Gateway returns JWT token + expiry
5. Rust stores token in keychain via `AuthManager.store_token()`
6. Rust returns success to SvelteKit; UI navigates to workspace

**Token Refresh:**
1. Before each Gateway API call, Rust layer checks token expiry
2. If expiry < 5 minutes: calls `IPC::refresh_token()`
3. Rust layer POSTs to `https://gateway/auth/refresh` with refresh_token (from keychain)
4. Gateway returns new access token
5. Rust updates keychain + returns to caller
6. API call proceeds with new token

**Logout:**
1. User clicks "Logout"
2. Rust layer calls `AuthManager.delete_token(username)`
3. Clears in-memory token cache
4. SvelteKit navigates to login screen

#### 6.4 Multi-User Device Support

The desktop app supports multiple users on the same device. Keychain entries are scoped per-username:

```rust
// Alice logs in
AuthManager::store_token("alice@example.com", "jwt_alice")
// Token stored in keychain under "alice@example.com"

// Bob logs in
AuthManager::store_token("bob@example.com", "jwt_bob")
// Token stored in keychain under "bob@example.com"

// User switches to Alice
let token = AuthManager::retrieve_token("alice@example.com")  // ✓ returns jwt_alice
```

SvelteKit maintains a `current_user_id` in state; Rust layer switches keychain lookup based on this.

#### 6.5 Bearer Token in All API Calls

All Rust-to-Gateway API calls include the JWT:

```rust
let token = AuthManager::retrieve_token(current_user)?;
let client = reqwest::Client::new();
let response = client
    .post("https://gateway/v1/documents")
    .header("Authorization", format!("Bearer {}", token))
    .json(&doc_payload)
    .send()
    .await?;
```

If a request returns 401 (Unauthorized), Rust layer automatically:
1. Attempts token refresh
2. Retries request with new token
3. If refresh fails, logs user out and prompts re-login

---

### 7. Integration Summary

#### 7.1 Request Flow Example: "Create Agent & Chat"

1. **SvelteKit UI:**
   - User selects project, enters agent name, picks provider (Ollama)
   - Clicks "Create Agent"

2. **Tauri IPC:**
   ```javascript
   const response = await invoke('create_agent', {
     project_id: '550e8400-e29b-41d4-a716-446655440000',
     name: 'Research Agent',
     provider: 'ollama'
   });
   ```

3. **Rust Backend:**
   - Retrieves JWT from keychain
   - POSTs to `https://gateway/v1/gateway/agents` with Bearer token:
     ```json
     {
       "project_id": "550e8400-e29b-41d4-a716-446655440000",
       "name": "Research Agent",
       "provider": "ollama"
     }
     ```

4. **Gateway:**
   - Validates JWT (user is authenticated + authorized for project)
   - Creates Agent record in DB
   - Returns `{ agent_id: "...", status: "ready" }`

5. **Rust Backend:**
   - Receives agent_id, stores in local cache
   - Returns to SvelteKit

6. **SvelteKit UI:**
   - Shows "Agent created: Research Agent"
   - Opens chat pane, ready for first message

7. **User types message:**
   - "Summarize the methodology in paper.pdf"

8. **Tauri IPC:**
   ```javascript
   const stream = await invoke('chat_with_agent', {
     agent_id: '...',
     message: 'Summarize the methodology in paper.pdf',
     context: {
       document_id: '...',
       search_results: [...]
     }
   });
   stream.onMessage(msg => updateChatUI(msg));
   ```

9. **Rust Backend:**
   - Streams POST to `https://gateway/v1/gateway/agents/{agent_id}/chat` (Server-Sent Events)
   - Receives chunks from Gateway (agent reasoning, tool calls, final response)
   - Forwards chunks to SvelteKit via IPC stream

10. **SvelteKit UI:**
    - Displays agent response in real-time, including any tool invocations

---

### 8. Phase Gates & Implementation Notes

#### 8.1 v1 (Phase 0: Foundation)
- Auth endpoints: register, login, refresh, keychain storage
- Document CRUD + local clone sync (pull/push)
- Conflict detection (no AI merge yet)
- Agent creation + chat streaming
- Section ownership UI (draft/structured mode)
- Orphaned section detection + reclaim IPC

#### 8.2 v2 (Phase 1: Collaboration)
- WebSocket-based real-time editing
- CRDT state serialization (Document.crdt_state)
- Presence awareness (who's editing which section)
- AI-assisted merge DAG execution
- Comment threads on sections

#### 8.3 v3 (Phase 2: Intelligence)
- Multi-agent workflows (agents coordinating on a document)
- Semantic indexing of all documents in memory garden
- Proactive suggestions (agent recommends section edits, restructures)
- Export to multiple formats (PDF, EPUB, HTML, Markdown)


## Revision R2: Locked Decisions (2026-02-18)

### Section Boundaries in Draft Mode — Final Decision

**Decision:** Any group member can create sections in draft mode. Owner approves structure at the draft → structured transition.

**Rationale:** Structure should emerge organically during early writing. Constraining section creation to the owner creates a bottleneck during ideation.

**Implementation detail:**
- In `mode: draft`, any co-author with access can call `create_section(document_id, title, position)` IPC command
- Sections created in draft mode have `owner_id: null` (unassigned) until the transition
- When the owner triggers "Organize sections" → structured mode transition, the assignment UI presents all existing sections (potentially overlapping or messily named) for the owner to: merge, rename, delete, or assign to co-authors
- The transition UI includes a "Merge sections" action: select two sections → confirm → content is concatenated, one section ID is retired
- After transition, `create_section` is restricted to document owner only

**New IPC command:**
```rust
#[tauri::command]
async fn create_section(
    document_id: String,
    title: String,
    position: Option<usize>, // None = append at end
) -> Result<Section, String>

#[tauri::command]
async fn merge_sections(
    section_ids: Vec<String>, // ordered: content concatenated in this order
    merged_title: String,
) -> Result<Section, String>
```
