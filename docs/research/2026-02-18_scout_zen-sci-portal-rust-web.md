# Strategic Scout: zen-sci-portal — Rust Backend + Web Frontend Architecture
**Date:** 2026-02-18
**Status:** Strategic Scouting (No Code)
**Audience:** Cruz Morales, Architecture Decision Makers

---

## Executive Summary

This scout explores the strategic decision space for building **zen-sci-portal** — a **social, cloud, multi-user knowledge platform** using **Tauri** (Rust backend + native webview frontend). The core tension: *What does a Rust backend uniquely provide for a social knowledge platform when the Gateway (Go) and ZenSci servers (TypeScript) already cover massive ground?*

The answer lies in **three critical capabilities Rust owns naturally**:
1. **Local-first persistence & search** — Rust's file-system and memory efficiency make local indexing/sync native
2. **Real-time collaboration at scale** — Tokio + CRDT libraries enable WebSocket-driven presence/sync without external infra
3. **Lightweight desktop deployment** — Tauri's native webview + Rust core offers 5-10x smaller footprint than Electron, crucial for adoption

**Recommended hybrid approach:** *Rust owns local search + session management; frontend (React + Vite for consistency with MCP Apps) handles social UI; Tauri IPC bridges them with careful async patterns to avoid string serialization bottlenecks.*

---

## Part 1: The Strategic Tensions

### Tension 1: Capability Overlap
**The Problem:**
- **AgenticGateway (Go)** already has: memory management, tool orchestration, provider routing, OTEL observability, SQLite persistence
- **ZenSci (TypeScript)** already has: document parsing, citation validation, format rendering, 6 MCP servers
- **Tauri (Rust)** layer would add: native file-system access, real-time sockets, platform integration

**Question:** What's the minimal Rust surface that adds value without duplicating Gateway/ZenSci work?

### Tension 2: Multi-User Requirements
The portal is **social/cloud/multi-user** (not solo-user/local-first). This requires:
- Session management & JWT validation
- Presence awareness (who's online? what are they doing?)
- Collaborative editing (multiple users editing the same document)
- Cloud sync (local changes → cloud persistence)

*Go + TypeScript can handle the cloud layer; Rust must own the **local replica** and **real-time link** back to cloud.*

### Tension 3: Performance vs. Abstraction
- IPC (Inter-Process Communication) in Tauri suffers from string serialization overhead
- Each Rust → frontend call must be carefully designed to avoid death-by-a-thousand-cuts
- Frontend framework choice (React vs. Svelte vs. Vue) impacts latency profile

*Solution: Batch requests, use WebSockets for high-frequency updates, reserve IPC for control flow.*

---

## Part 2: Track 1 — Rust Backend Routes

### Route A: Local Search Index (Tantivy-Powered)

**What it does:**
Rust maintains a full-text search index over all user documents and ZenSci outputs (PDFs, blog posts, grant proposals). Users type "photosynthesis mechanism"—Rust searches the local index instantly, then fetches full document from Gateway as needed.

**Why Rust excels:**
- **Tantivy** (Lucene-inspired CRDT-free search library) compiles to native code; sub-50ms latency on 10K+ documents
- File-system integration is native (no Python subprocess)
- Memory efficiency: Tantivy uses FSTs (Finite State Transducers) + integer compression; typical 100MB index for 50K documents
- Index persists locally; optional cloud sync to Gateway's SQLite

**Which ZenSci clusters benefit:**
1. **Parsing** — Index feeds back document metadata (title, sections, citations) to UI
2. **Validation** — Search index validates cross-references against indexed documents
3. **Structure** — Index tracks document hierarchy for navigation UI
4. **Output** — Search treats rendered PDFs/HTML as indexable text

**Risk profile:**
- **Low implementation risk:** Tantivy is production-grade (used by Quickwit, ParadeDB)
- **Medium integration risk:** Must keep index in sync with Gateway's SQLite on cloud write
- **Mitigation:** Tantivy + SQLite write hooks; prioritize local-first correctness over cloud sync speed

**Key Rust crates:**
- `tantivy` (0.22+) — full-text search engine
- `tokio` (async I/O) — background index updates
- `serde_json` / `bincode` — index serialization
- `dashmap` — lock-free concurrent index state

**Effort estimate:** 6–8 weeks (index schema design + sync strategy)

---

### Route B: Native File System & Cloud Sync

**What it does:**
Rust watches the user's local filesystem (`~/Documents/zen-sci/`). When a markdown file changes, Rust queues it for ZenSci processing, then syncs outputs (PDFs, HTML, slides) back to the folder. Cloud sync layer keeps these files in sync with Gateway's cloud storage.

**Why Rust excels:**
- **notify** crate provides OS-level file-system events (inotify on Linux, FSEvents on macOS, ReadDirectoryChangesW on Windows)
- Native symlink & permission handling
- Can run watchers in background threads without blocking UI
- Tauri's file system APIs pair perfectly

**Which ZenSci clusters benefit:**
1. **Parsing** — local file system is the source of truth
2. **Conversion** — outputs written back to same folder automatically
3. **Configuration** — per-folder settings in `.zen-config.yaml`
4. **Rendering & Output** — final artifacts (PDFs) appear locally first

**Risk profile:**
- **Low implementation risk:** notify crate is mature; well-tested on all platforms
- **Medium UX risk:** User expectations around folder structure & sync timing
- **Mitigation:** Clear sync status indicator; conflict resolution strategy (local version wins, cloud versioned)

**Key Rust crates:**
- `notify` (5.2+) — OS-level file-system watching
- `async-fs` + `tokio::fs` — async file operations
- `ignore` — .gitignore-aware file enumeration
- `dirs` — platform-specific document folder paths

**Effort estimate:** 4–6 weeks (watcher + sync reconciliation)

---

### Route C: PDF/Document Native Rendering Pipeline

**What it does:**
Instead of relying on Python subprocess (`latex_engine.py`) or pandoc overhead, Rust handles the final PDF rendering step natively. Takes LaTeX/HTML from ZenSci servers, renders to PDF using Rust libraries, returns to webview for display.

**Why Rust excels:**
- `printpdf` (pure Rust PDF generation) or `lopdf` (lower-level) avoid subprocess overhead
- Can render inline in the Rust backend; no process fork, no GC pressure
- Easier to sandbox and control in desktop context
- WASM-compatible if future mobile support needed

**Which ZenSci clusters benefit:**
1. **Math Rendering** — embedded math objects in PDF (already LaTeX)
2. **Bibliography Rendering** — PDF bookmarks from citation structure
3. **Rendering & Output** — final PDF generation step

**Risk profile:**
- **High implementation risk:** PDF generation is complex; `printpdf` is less mature than pandoc
- **Medium feature risk:** LaTeX ↔ PDF translation is non-trivial; LaTeX subset support may be limited
- **Mitigation:** Phase 1 focuses on simple documents; keep pandoc as fallback

**Key Rust crates:**
- `printpdf` (0.8+) — pure Rust PDF generation
- `lopdf` (0.34+) — lower-level PDF object library
- `pdfium-render` (0.8+) — read/manipulate existing PDFs
- `texcraft` (if LaTeX parsing needed)

**Effort estimate:** 8–12 weeks (LaTeX → PDF subsetting is complex)

---

### Route D: Auth & Session Management (JWT + Keychain)

**What it does:**
Rust handles the entire auth layer: JWT generation/validation, session token storage in platform keychain (macOS Keychain, Windows DPAPI, Linux Secret Service), credential refresh, and multi-account support.

**Why Rust excels:**
- `jsonwebtoken` (mature, audited library) for JWT operations
- Platform keychain integration via `security-framework` (macOS) / `windows-credentials` (Windows) / `secret-service` (Linux)
- Centralized session state prevents scattered token management in webview
- Can enforce device/app-binding in JWT claims

**Which ZenSci clusters benefit:**
1. **Parsing** — all document operations require auth context
2. **Validation** — validates user permissions for documents
3. **Structure** — user identity shapes document access (shared vs. private)
4. All downstream operations require session context

**Risk profile:**
- **Low implementation risk:** JWT libraries are production-grade
- **Medium platform risk:** Keychain APIs are OS-specific; must handle all three platforms
- **Mitigation:** Fallback to encrypted file storage if keychain unavailable; test extensively on all OSes

**Key Rust crates:**
- `jsonwebtoken` — JWT creation/validation
- `security-framework` — macOS Keychain (must use on macOS only)
- `windows-credentials` — Windows DPAPI (Windows only)
- `secret-service` — Linux Secret Service (D-Bus based)
- `ring` or `aws-lc-rs` — crypto backend (choose one)

**Effort estimate:** 4–6 weeks (platform-specific testing is the long pole)

---

### Route E: Real-Time Collaboration (Tokio + CRDT)

**What it does:**
Rust maintains a WebSocket server (via Tokio) for real-time collaboration. Users see presence awareness (who's editing?), live cursor positions, and collaborative editing with automatic conflict resolution using **Automerge** (CRDT library).

**Why Rust excels:**
- **Tokio** async runtime handles thousands of concurrent WebSocket connections on one thread
- **Automerge 3** (released 2025, 10x memory reduction) provides CRDT-based document merging
- Rust's type system enforces memory safety across concurrent operations
- Natural fit with Tauri's event loop; no garbage collection hiccups

**Which ZenSci clusters benefit:**
1. **Structure** — CRDT maintains consistent document tree across peers
2. **Conversion** — when document changes (via CRDT), trigger re-render automatically
3. **Rendering & Output** — multiple users see live rendering updates
4. Enables true **social** knowledge creation

**Risk profile:**
- **High implementation risk:** CRDT semantics are non-intuitive; Automerge 3 is very new
- **Medium latency risk:** Conflict resolution can be expensive; must batch updates carefully
- **Medium architectural risk:** Breaks the "stateless server" pattern; Rust maintains session state

**Key Rust crates:**
- `tokio` (1.40+) — async runtime
- `tokio-tungstenite` — WebSocket protocol over Tokio
- `automerge` (0.4+) — CRDT document representation
- `serde` / `bincode` — serialization of CRDT state
- `dashmap` — lock-free session state

**Effort estimate:** 12–16 weeks (CRDT semantics + conflict recovery is substantial)

---

## Part 2 Summary: Route Comparison Table

| Route | Complexity | Value Add | ZenSci Clusters | Timeline | Dependencies |
|-------|-----------|-----------|-----------------|----------|--------------|
| **A: Search Index** | Medium | High (UX-critical) | Parsing, Validation, Structure | 6–8w | Tantivy, tokio |
| **B: File Sync** | Low | Medium (convenience) | Parsing, Conversion, Output | 4–6w | notify, async-fs |
| **C: PDF Rendering** | High | Low (nice-to-have) | Math, Bibliography, Output | 8–12w | printpdf/lopdf |
| **D: Auth & Keychain** | Medium | High (security/UX) | All (cross-cutting) | 4–6w | jsonwebtoken, OS-specific |
| **E: Real-Time Collab** | Very High | Very High (product diff) | Structure, Conversion, Output | 12–16w | Tokio, Automerge, WebSocket |

---

## Part 3: Track 2 — Web Frontend Routes

### Frontend Route A: React + Vite

**Framework choice:**
- **React 18+** with hooks-based architecture
- **Vite** build tool (fast HMR, excellent tree-shaking)
- **TypeScript** (consistency with MCP Apps)

**MCP App embedding:**
- MCP Apps (Phase 4) are already Vite + single-file HTML bundles
- React can iframe them directly via `<iframe srcDoc={appHtml} />`
- Consistent tooling: same `vite.config.ts` patterns

**Real-time features:**
- **React Query** or **SWR** for server state
- **Socket.IO** or raw WebSocket + React state for presence
- **Zustand** or **Jotai** for client state (lighter than Redux)

**Tauri IPC compatibility:**
- React's virtual DOM adds ~10-15ms latency to IPC responses; acceptable for control flow
- High-frequency updates (presence, typing indicators) must use WebSocket, not IPC
- `@tauri-apps/api` client library has good React bindings

**DX & build complexity:**
- Largest ecosystem; most tutorials & StackOverflow answers
- HMR works well; large bundle size (300-400KB gzipped for simple app)
- Excellent component library support (MUI, Chakra, Shadcn)

**Risk profile:**
- **Low risk:** Proven pattern for Tauri apps
- **Medium risk:** React ecosystem is large; dependency management can be chaotic
- **Mitigation:** Use `npm audit` regularly; pin transitive dependencies

**Key dependencies:**
- `react` ^18
- `vite` ^5
- `@tauri-apps/api` ^2
- `@tauri-apps/plugin-websocket` (for WebSocket)
- `zustand` or `jotai` (state)
- `@tanstack/react-query` (server state)

**Effort estimate:** 8–12 weeks (including Tauri IPC integration)

---

### Frontend Route B: SvelteKit

**Framework choice:**
- **SvelteKit** with **static adapter** (required for Tauri; no SSR)
- **Svelte 5** (fine-grained reactivity, no virtual DOM)
- **Vite** (same as React, but faster initial build)

**MCP App embedding:**
- Svelte can iframe MCP Apps, but less native integration than React
- `svelte/store` for global state; works well with IPC

**Real-time features:**
- Svelte's fine-grained reactivity makes real-time updates **extremely efficient**
- WebSocket integration is cleaner: bind directly to stores without React hooks
- Presence indicators, typing cursors have minimal overhead

**Tauri IPC compatibility:**
- Svelte's compiled output is smaller (~50-100KB); faster IPC serialization
- Store-based updates integrate naturally with Tauri command responses
- Less "glue code" than React

**DX & build complexity:**
- Smaller community than React; fewer third-party component libraries
- Excellent documentation; learning curve is gentler than React
- SvelteKit's file-based routing is intuitive (like Next.js)

**Risk profile:**
- **Low implementation risk:** SvelteKit is stable (v2.0 released)
- **Medium ecosystem risk:** Fewer pre-built components; may need custom components
- **Mitigation:** Use shadcn (works with Svelte now); Svelte is gaining adoption

**Key dependencies:**
- `svelte` ^5
- `sveltekit` ^2
- `@tauri-apps/api` ^2
- `@tauri-apps/plugin-websocket`
- `svelte/store` (built-in)

**Effort estimate:** 6–10 weeks (faster than React due to less framework overhead)

---

### Frontend Route C: SolidJS

**Framework choice:**
- **SolidJS** (ultra-fine-grained reactivity, compiled to direct DOM updates)
- **Vite** (first-class Solid support)
- **TypeScript** (if strict typing desired; optional in Solid)

**MCP App embedding:**
- SolidJS's portals can embed iframe-based MCP Apps
- Custom elements wrapper possible for tighter integration

**Real-time features:**
- **Fastest** of the three frameworks for real-time updates
- No virtual DOM overhead; reactive primitives map perfectly to WebSocket events
- Excellent for presence indicators, live cursors, live rendering

**Tauri IPC compatibility:**
- Smallest bundle size (~20-50KB); lightning-fast serialization
- `createSignal()` / `createEffect()` integrate naturally with async Tauri calls
- Excellent async handling via `createResource()`

**DX & build complexity:**
- Very small community; limited third-party integrations
- Learning curve steeper than Svelte/React (different mental model)
- Excellent performance for power users willing to learn a new paradigm

**Risk profile:**
- **Low implementation risk:** Solid is production-ready (v1.8+)
- **High hiring/maintenance risk:** Fewer developers familiar with Solid
- **Medium ecosystem risk:** Fewer component libraries; more DIY components

**Key dependencies:**
- `solid-js` ^1.8
- `solid-start` (optional meta-framework)
- `@tauri-apps/api` ^2
- `@tauri-apps/plugin-websocket`

**Effort estimate:** 8–12 weeks (plus learning curve)

---

### Frontend Route D: Vue 3 + Vite

**Framework choice:**
- **Vue 3** with Composition API
- **Vite** (excellent Vue 3 support, official)
- **TypeScript** optional (works well in Vue 3)

**MCP App embedding:**
- Vue's component model works naturally with iframe-based MCP Apps
- Can wrap iframes in custom components with scoped styling

**Real-time features:**
- Composition API provides reactive primitives similar to Svelte
- `ref()` / `computed()` integrate well with WebSocket updates
- Good balance between performance and ease of learning

**Tauri IPC compatibility:**
- Vue's reactivity layer is efficient; no virtual DOM overhead in most cases
- Good TypeScript support via `<script setup>` syntax
- Community has solid Tauri examples

**DX & build complexity:**
- Growing ecosystem; Vite + Vue integration is first-class
- Single-File Components (`.vue`) are intuitive
- Smaller than React ecosystem, but larger than Svelte/Solid

**Risk profile:**
- **Low implementation risk:** Vue 3 is stable and widely used
- **Medium ecosystem risk:** Smaller than React, but steadily growing
- **Mitigation:** Use Ant Design Vue or PrimeVue for component libraries

**Key dependencies:**
- `vue` ^3.5
- `vite` ^5
- `@tauri-apps/api` ^2
- `pinia` (state, Vuex alternative)
- `@vueuse/core` (composition utilities)

**Effort estimate:** 8–10 weeks

---

### Frontend Route E: Vanilla TypeScript + Web Components

**Framework choice:**
- No framework; vanilla **TypeScript** + Web Components
- **Vite** for bundling and HMR
- Custom event system for inter-component communication

**MCP App embedding:**
- **Perfect fit:** MCP Apps are already Web Components / single-file HTML
- Can embed as `<custom-mcp-app src="ui://latex-mcp/preview.html"></custom-mcp-app>`
- Zero abstraction mismatch

**Real-time features:**
- Full control over rendering pipeline
- WebSocket updates can directly manipulate DOM
- No framework overhead; minimal memory usage

**Tauri IPC compatibility:**
- Direct `@tauri-apps/api` usage; no framework layer
- Extremely efficient command dispatch
- Minimal serialization overhead

**DX & build complexity:**
- Steepest learning curve; must hand-code many patterns (routing, state, etc.)
- Excellent for power users with strong JavaScript/TypeScript skills
- Vite's HMR still works; watch-mode development is smooth

**Risk profile:**
- **High implementation risk:** No guardrails; must build everything from scratch
- **Very high maintenance risk:** Code brittleness without framework structure
- **Hiring risk:** Hard to find developers comfortable with vanilla TypeScript

**Key dependencies:**
- `typescript`
- `vite` ^5
- `@tauri-apps/api` ^2
- Custom routing lib (or none)

**Effort estimate:** 12–16 weeks (more design time, less framework time)

---

## Part 3 Summary: Frontend Comparison Table

| Route | Framework | Bundle Size | Latency | DX | Tauri IPC | Real-Time | Effort |
|-------|-----------|-------------|---------|-----|-----------|-----------|--------|
| **A: React** | Proven | 300–400KB | 10–15ms | Excellent | Native | Good | 8–12w |
| **B: SvelteKit** | Modern | 50–150KB | 5–10ms | Great | Good | Excellent | 6–10w |
| **C: SolidJS** | Cutting-edge | 20–50KB | 2–5ms | Good | Good | Excellent | 8–12w |
| **D: Vue 3** | Growing | 150–250KB | 5–10ms | Great | Good | Good | 8–10w |
| **E: Vanilla TS** | Minimal | <50KB | 1–3ms | Hard | Native | Excellent | 12–16w |

---

## Part 4: Synthesis — Route Pairings

### Pairing 1: Search Index (Route A) + React (Route A)
**Rationale:**
- React's virtual DOM fine for IPC-driven search results
- Search queries batch well; serialize as `{query: "", filters: []}` → wait for Tantivy results
- MCP App embedding (Phase 4) already assumes React
- **Consistency across portal + App UIs**

**Data flow:**
```
User types search query
  → React sends via Tauri IPC: `invoke("search", {query, filters})`
  → Rust: Tantivy search → returns 50 results
  → React: renders paginated list, virtual scrolling (efficient)
  → User clicks result → fetch full doc from Gateway
```

**Effort:** 8–12w (frontend) + 6–8w (Rust search) = 14–20w total
**Risk:** Medium (Tantivy + React integration is straightforward)

---

### Pairing 2: File Sync (Route B) + SvelteKit (Route B)
**Rationale:**
- File system events pair beautifully with Svelte's reactivity
- User adds `myresearch.md` → Rust detects → triggers ZenSci → Svelte UI auto-updates
- Lighter bundle perfect for instant HMR during development
- Svelte stores maintain folder state elegantly

**Data flow:**
```
User saves file ~/zen-sci/myresearch.md
  → Rust notify event fires
  → Queues ZenSci: convert_to_pdf, convert_to_html
  → On completion, Rust emits Tauri event
  → SvelteKit listener updates store
  → UI reactively shows new PDF, HTML in sidebar
```

**Effort:** 4–6w (file sync) + 6–10w (SvelteKit) = 10–16w total
**Risk:** Low (notify crate is battle-tested; Svelte is reactive by design)

---

### Pairing 3: Auth + Keychain (Route D) + SolidJS (Route C)
**Rationale:**
- Auth is a control-plane operation (low frequency)
- Solid's fine-grained reactivity overkill for auth, but excellent for presence/session state
- Smallest bundle maximizes adoption on lower-end machines
- Solid's stores integrate perfectly with encrypted keychain state

**Data flow:**
```
First launch: no credentials
  → Rust checks platform keychain → empty
  → Solid renders login form
  → User logs in → Rust validates JWT → stores in keychain
  → Solid store updates; app unlocked
  → Every subsequent launch: Rust reads keychain, silent auth
```

**Effort:** 4–6w (auth) + 8–12w (SolidJS) = 12–18w total
**Risk:** Medium (Solid is less familiar; auth is security-critical)

---

### Pairing 4: Real-Time Collab (Route E) + SvelteKit (Route B)
**Rationale:**
- Real-time collaboration is **the product differentiator** for a social platform
- Svelte's fine-grained reactivity + Tokio WebSocket = efficient collaboration at scale
- Automerge CRDT semantics map cleanly to Svelte stores
- SvelteKit's progressive enhancement (if needed) pairs well with real-time updates

**Data flow:**
```
User A edits "Introduction" section
  → Rust Tokio WebSocket server receives Automerge delta
  → Broadcasts to User B's Rust instance
  → User B's Svelte reactive store updates
  → UI re-renders user B's document instantly
  → Conflict-free merge via Automerge CRDT
```

**Effort:** 12–16w (real-time collab) + 6–10w (SvelteKit) = 18–26w total
**Risk:** Very High (Automerge 3 is new; CRDT semantics require careful testing)

---

## Part 5: Recommended Direction

### Winning Hybrid Approach: **Search + Auth + SvelteKit**

**What you build:**
1. **Rust (Tauri Backend):**
   - **Search index (Route A)** — Tantivy-powered local search over all documents/outputs
   - **Auth layer (Route D)** — JWT + platform keychain for multi-user device context
   - **IPC bridge** — OpenAI-compatible chat ← this already exists in Gateway, expose via IPC

2. **Frontend (SvelteKit):**
   - **Consistency with MCP Apps** (which are Vite + single-file HTML)
   - **Real-time features** via SvelteKit stores + optional WebSocket for presence
   - **File sync dashboard** — shows pending/completed conversions
   - **Search UI** — integrates Tantivy results with semantic browsing

3. **Why this wins:**
   - **Clear separation of concerns:** Rust = local state + security; Frontend = UI + social features
   - **Tauri IPC used sparingly:** Only for search queries, auth state, document metadata
   - **No WebSocket on LAN** (optional; can be added later for real-time collab without breaking foundation)
   - **Shipping velocity:** Search + Auth are 10–12 weeks; SvelteKit is 6–10 weeks = **achievable in Q2 2026**
   - **Graceful degradation:** If Tantivy/Keychain unavailable, app falls back to Gateway search + env-var auth

**Architecture diagram:**

```
┌─────────────────────────────────────────────────────┐
│  zen-sci-portal (Tauri App)                         │
│                                                      │
│  ┌──────────────────┐      ┌─────────────────────┐  │
│  │  SvelteKit UI    │      │  Rust Backend       │  │
│  │                  │◄────►│  (Tauri Core)       │  │
│  │ - Social view    │ IPC  │                     │  │
│  │ - File browser   │      │ ┌─────────────────┐ │  │
│  │ - Search results │      │ │ Tantivy Index   │ │  │
│  │ - Presence       │      │ │ (local search)  │ │  │
│  │ - Settings       │      │ └─────────────────┘ │  │
│  │                  │      │                     │  │
│  │ Stores:          │      │ ┌─────────────────┐ │  │
│  │ - search results │      │ │ JWT + Keychain  │ │  │
│  │ - user session   │      │ │ (auth state)    │ │  │
│  │ - doc metadata   │      │ └─────────────────┘ │  │
│  │                  │      │                     │  │
│  └──────────────────┘      │ ┌─────────────────┐ │  │
│                            │ │ File Watcher    │ │  │
│                            │ │ (notify, async) │ │  │
│                            │ └─────────────────┘ │  │
│                            └─────────────────────┘  │
│                                      ▲               │
└──────────────────────────────────────┼───────────────┘
                                       │
                    ┌──────────────────┴──────────────────┐
                    │                                     │
              ┌─────▼──────┐                      ┌──────▼─────┐
              │  AgenticGW  │                      │ Cloud Sync  │
              │  (Go)       │                      │ (optional)  │
              │             │                      │             │
              │ - Chat API  │                      │ - S3/DB     │
              │ - Memory    │                      │ - Versioning
              │ - Tools     │                      │             │
              └─────┬──────┘                       └─────────────┘
                    │
              ┌─────▼──────────────┐
              │  ZenSci Servers    │
              │  (TypeScript MCP)  │
              │                    │
              │ - latex-mcp        │
              │ - blog-mcp         │
              │ - grant-mcp        │
              │ - slides-mcp       │
              │ - newsletter-mcp   │
              │ - paper-mcp        │
              └────────────────────┘
```

**Why SvelteKit > React:**
- Smaller bundle (faster startup on slower machines)
- Svelte stores integrate seamlessly with real-time updates
- SvelteKit's file-based routing is intuitive for feature team development
- No virtual DOM overhead when rendering 100-item search results

**Why Tantivy search > Gateway search for local use:**
- Sub-50ms latency (user types, results instantly)
- Works offline (crucial for knowledge workers)
- Index persists locally; can sync to Gateway on cloud write
- User data stays on device (privacy win)

**Why keychain auth > env-var auth:**
- Multi-user device support (different people log in)
- Credentials never written to disk unencrypted
- Zero manual secrets management
- Native platform integration wins adoption

---

## Part 6: Open Questions for Cruz

Before locking architecture, answer these:

### Q1: Real-Time Collaboration Scope
**Current assumption:** Phase 1 supports single-user or async multi-user (Google Docs style revision history).
**Decision needed:** Should Phase 2 include live cursor positions + presence? This gates Routes E heavily.
**Impact:** Route E (Collab) adds 12–16 weeks; can defer to v1.1 if not critical.

### Q2: Cloud Sync Strategy
**Current assumption:** Local state is primary; cloud is backup.
**Decision needed:** What's the conflict resolution policy? (Last write wins? User chooses? CRDT?)
**Impact:** Affects file sync (Route B) complexity and authentication (Route D) scope.

### Q3: Offline-First vs. Always-Connected
**Current assumption:** Users expect offline work in Tauri, but most operations assume Gateway connection.
**Decision needed:** What features must work offline? (Search? Document conversion? Auth refresh?)
**Impact:** Determines local cache strategy and IPC bandwidth.

### Q4: Multi-User Portal or Multi-Tenant?
**Current assumption:** Each user runs their own Tauri instance (like Notion Desktop app).
**Decision needed:** Future: shared portal (web) vs. per-user Tauri instances?
**Impact:** Auth strategy (single-device vs. org-wide) and scaling assumptions.

### Q5: Timeline Priority
**Decision needed:** Which comes first?
1. **Search + Auth** (v1.0 — 10–12 weeks, shipping value immediately)
2. **Real-time collab** (v1.1 — 12–16 weeks, product differentiator)
3. **File sync** (v1.1 — 4–6 weeks, convenience feature)

### Q6: Platform Support
**Current assumption:** macOS + Windows + Linux via Tauri.
**Decision needed:** Mobile (iOS/Android) in scope? Tauri v2 supports it, but adds complexity.
**Impact:** Affects auth (keychain vs. mobile platform defaults) and real-time (WebSocket on mobile).

---

## Part 7: Risk & Mitigation Summary

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| **Tantivy index falls out of sync with Gateway** | Medium | High | Implement write-through cache; design index as read-only replica. Test sync on every doc operation. |
| **Tauri IPC becomes bottleneck** | Medium | Medium | Use WebSocket for high-freq updates (presence, typing). Batch search queries. |
| **Keychain integration fails on Linux** | Low | Medium | Support fallback: encrypted file storage. Test on all three platforms early. |
| **Svelte ecosystem too small** | Low | Low | If needed, pivot to React (same IPC interface). No UI changes needed. |
| **Automerge 3 has stability issues** | Medium | Very High | Defer real-time collab to v1.1. Ship async collaboration first (Routes A+D+B). |
| **User expectations for "cloud" not met** | Medium | High | Clear UX: show local ← → cloud sync status. Educate in docs. |

---

## Part 8: Recommended Phase Roadmap

### Phase 4.1 (Portal v1.0): Foundation — 10–12 weeks
**What ships:**
- Rust: Tantivy search index + file watcher
- Rust: JWT auth + platform keychain
- Frontend: SvelteKit UI (search, auth, file browser, settings)
- Tauri IPC bridge (search, auth, file metadata)

**Success criteria:**
- User opens app → logs in via keychain
- User browses local files → search across all documents
- User clicks result → fetches full doc from Gateway
- Offline search works when Gateway unavailable

### Phase 4.2 (Portal v1.1): Collaboration — 12–16 weeks
**What ships (optional, based on Q1 decision):**
- Rust: Tokio WebSocket server + Automerge CRDT
- Real-time presence (who's editing?)
- Live collaborative editing
- Conflict-free merging

**Success criteria:**
- Two users on same doc → edits merge automatically
- Latency < 500ms for presence updates
- Conflicts resolve without user intervention

### Phase 4.3 (Portal v1.2): Scale — 6–8 weeks
**What ships:**
- Cloud sync via Gateway (S3 + metadata store)
- Document versioning
- Permission/sharing controls
- Team workspaces

---

## Conclusion

**The winning path for zen-sci-portal is a **pragmatic layering**:**

1. **Rust owns the local, security-critical, performance-sensitive layers** (search, auth, file watching)
2. **SvelteKit owns the social, real-time, multi-user UI layers** (presence, collaboration, sharing)
3. **Tauri IPC is reserved for control flow only** (search queries, auth state, doc metadata)
4. **ZenSci + Gateway handle the heavy lifting** (conversion, storage, reasoning)

**This approach:**
- Delivers v1.0 in 10–12 weeks (search + auth + SvelteKit)
- Keeps UX responsive via local search index
- Enables multi-user support with clear auth boundaries
- Leaves room for real-time collab in v1.1 without rearchitecting
- Respects the existing ZenSci semantic clusters; enhances without duplicating

**Next step:** Confirm decisions Q1–Q6, then commission Rust backend (Tantivy + JWT + notify watchers) in parallel with SvelteKit frontend scaffolding.

---

## Appendix: Crate Reference

### Rust Ecosystem (Tauri Backend)
- **Tantivy 0.22+** — Full-text search index
- **Tokio 1.40+** — Async runtime (search, file watching, WebSocket)
- **Jsonwebtoken** — JWT creation/validation
- **Security-framework** (macOS) / **windows-credentials** (Windows) / **secret-service** (Linux) — Platform keychain
- **Notify 5.2+** — File-system watching
- **Automerge 0.4+** — CRDT for real-time collab (v1.1)
- **Serde / Bincode** — Serialization
- **Dashmap** — Lock-free concurrent state

### Frontend Ecosystem (SvelteKit)
- **SvelteKit 2.0+** — Meta-framework (routing, SSG)
- **Svelte 5.0+** — Component framework (fine-grained reactivity)
- **Vite 5.0+** — Build tool
- **@tauri-apps/api ^2** — Tauri IPC client
- **@tauri-apps/plugin-websocket** — WebSocket plugin (real-time collab)
- **TailwindCSS** / **shadcn** — Styling

### Development Tools
- **Goreleaser** — Multi-platform Rust binary builds
- **cargo-watch** — Live reload
- **npm run dev** — SvelteKit HMR

---

**Document prepared by:** Strategic Scout
**Date:** 2026-02-18
**Status:** Ready for decision & architecture lockdown
**Approval gate:** Q1–Q6 decisions + timeline confirmation
