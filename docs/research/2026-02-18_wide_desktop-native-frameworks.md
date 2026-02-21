# WIDE RESEARCH: Desktop/Native App Frameworks & Rust Ecosystem for zen-sci-portal

**Research Date:** 2026-02-18
**Project:** zen-sci-portal (Tauri desktop app + Rust backend for knowledge work)
**Prepared For:** Cruz Morales, ZenithScience / TresPiesDesign.com

---

## Landscape Brief

**Scope Boundaries:**
- **Track A:** Desktop/native app framework landscape — what exists beyond "just a browser web app"
- **Track B:** Rust ecosystem maturity for knowledge work, document processing, search, PDF/math, collaboration
- **Architectural Context:** AgenticGateway (v1.0.0 Go, 8 LLM providers) + ZenSci (6 TS/MCP servers) + Tauri v2 (chosen frontend)
- **Out of Scope:** Mobile-only frameworks, full server deployment patterns, enterprise licensing

**Strategic Questions Answered:**
1. What does the native/desktop landscape uniquely offer vs. browser apps?
2. What Rust crates are production-ready for zen-sci-portal's domain?
3. Where should zen-sci-portal invest Rust capability first?

---

## Source Scan Table

| # | Source | Relevance | Key Takeaway |
|---|--------|-----------|--------------|
| 1 | DoltHub: Electron vs. Tauri (2025) | 5 | Tauri: <10MB bundle, 30–50MB idle memory; Electron: 100MB+, 150–300MB idle. Tauri startup <0.5s vs 1–2s. |
| 2 | RaftLabs: Tauri vs Electron practical guide | 5 | Electron: Node.js + Chromium (years of ecosystem). Tauri: Rust + native webview (smaller, faster). |
| 3 | Tauri v2 Docs: Sidecar (Node.js) | 5 | Sidecar bundling: embed external binaries (Go, Python, Node.js) with multi-arch support. Multi-target naming convention. |
| 4 | Tauri v2 Docs: Embedding Binaries | 5 | `externalBin` config, cross-platform naming (e.g., `-x86_64-unknown-linux-gnu`), runtime via `shell().sidecar()`. |
| 5 | Wails: Introduction | 4 | Go + webview desktop framework. Auto-generate TypeScript models for Go structs. Native rendering, small footprint. |
| 6 | Wails GitHub (~20K stars) | 4 | Go desktop apps with web frontend. Svelte, React, Vue, Preact templates. Dev mode auto-reload. |
| 7 | Flutter Desktop Docs (3.38, Nov 2025) | 4 | Flutter 3.38 + Dart 3.10. Desktop adoption: 24.1% macOS, 20.1% Windows, 11.2% Linux. AI Toolkit added. |
| 8 | Flutter 2026 roadmap | 3 | v3.41 expected Feb 2026. Focus: modularization, Impeller, Material 3, desktop parity. 2.8M monthly devs. |
| 9 | Qt/Desktop comparison 2026 | 4 | Qt (C++/QML): fastest startup, lowest memory. Mature enterprise tools. Legacy but solid. |
| 10 | .NET MAUI Overview | 3 | Windows/macOS/Linux from single C# codebase. Microsoft backing. Slightly slower than Qt. |
| 11 | Capacitor/Ionic cross-platform | 2 | Web + native bridge. Good for standard device access, not desktop-first. |
| 12 | PWA vs Native 2026 | 3 | PWAs 40–60% cheaper. Service workers/caching. Still can't fully serve offline knowledge work. |
| 13 | Tantivy GitHub (~8.5K stars) | 5 | Full-text search Lucene-equivalent. 2x faster than Lucene. <10ms startup. Immutable data. |
| 14 | Tantivy Docs (docs.rs) | 5 | Configurable tokenizer, 17 Latin language stems, phrase queries, stable Rust. |
| 15 | lopdf GitHub | 4 | Rust PDF read/write. v0.38.0 (Rust 2024 edition). PDF 1.5+ object streams. 11–61% file size reduction. |
| 16 | pulldown-cmark GitHub | 5 | Rust CommonMark parser. v0.13.0. Very fast, low memory. Used by rustc, cargo doc. |
| 17 | comrak GitHub | 5 | GitHub Flavored Markdown. 100% CommonMark compat. Used by crates.io, docs.rs, lib.rs, GitLab, Deno. |
| 18 | tokio-tungstenite GitHub | 5 | WebSocket + Tokio. Production use (video conferencing, real-time). v0.26.2+. TLS support. |
| 19 | rusqlite crates.io | 4 | SQLite wrapper for Rust. Sync API. tokio-rusqlite for async. Standard embedded DB choice. |
| 20 | sled lib.rs | 4 | Pure-Rust transactional embedded DB. Non-blocking reads/writes. flush_async support. |

---

## Patterns & Themes

### Cluster 1: Bundle Size & Performance (Mature)

Tauri <10MB vs Electron 100MB+. Idle memory 30–50MB vs 150–300MB. Startup <0.5s vs 1–2s. For an always-on knowledge platform, this gap is decisive. Users expect instant launch for ambient tools.

### Cluster 2: Language Ecosystem Lock-In (Mature)

Framework choice equals language choice for the backend layer:

| Framework | Frontend | Backend | Fit for zen-sci-portal |
|-----------|----------|---------|------------------------|
| **Tauri** | Web tech | **Rust** | ✅ Best fit — Rust for hard problems, sidecar for Go/TS |
| **Electron** | Web tech | **Node.js** | ❌ Redundant with existing TS; no Rust advantage |
| **Wails** | Web tech | **Go** | ⚠️ Go is good but Tauri's polyglot sidecar wins |
| **Flutter** | **Dart** | Dart | ❌ Forces Dart; no benefit over existing stack |
| **Qt** | C++/QML | C++ | ❌ Legacy tooling; no Rust alignment |

Tauri forcing Rust is not a bug — it's a feature. Rust + Go (Gateway) + TypeScript (ZenSci) gives clean separation of concerns.

### Cluster 3: Sidecar/Subprocess Patterns (Growing → Mature)

Tauri v2 `externalBin` supports bundling any binary (Go, Python, Node.js) with multi-arch selection at runtime. AgenticGateway (Go binary) can be shipped as a sidecar. ZenSci servers (Node.js) can also be sidecared. Single installer, one-click launch.

### Cluster 4: Real-Time Collaboration via Native Desktop (Growing)

Notable Tauri apps doing real-time: Noor (deep work chat), HuLa (IM + Vue3), Banban (Kanban + markdown), Vector (E2EE messenger). Pattern: tokio-tungstenite WebSocket + local SQLite + background sync. Native desktop enables lower latency than cloud-only for shared knowledge work.

### Cluster 5: Rust Knowledge Work Ecosystem (Growing → Mature)

Production-ready tier for zen-sci-portal's domain:

| Crate | Category | Maturity | Fit |
|-------|----------|----------|-----|
| pulldown-cmark | Markdown → HTML/AST | ✅ Mature | Parse ZenSci markdown |
| comrak | Markdown → GFM | ✅ Mature | GitHub-flavored markdown |
| tantivy | Full-text search | ✅ Mature | Index all documents locally |
| lopdf | PDF read/write | ✅ Stable | Extract/annotate PDFs |
| tokio-tungstenite | WebSocket real-time | ✅ Mature | Multi-user sync |
| rusqlite | Local SQLite | ✅ Mature | Embed document store |
| notify | Filesystem watching | ✅ Mature | Watch for local changes |
| reqwest | HTTP client | ✅ Mature | Sync to AgenticGateway |
| jsonwebtoken | JWT auth | ✅ Mature | Auth to Gateway |
| rustls | TLS crypto | ✅ Mature | Secure connections |

**Gap:** Symbolic math libraries in Rust (nalgebra, rusymbols) are early-stage. For publication-quality LaTeX math, ZenSci's TeX → Python engine (sympy via pandoc) remains the right path. Don't try to re-solve symbolic math in Rust.

### Outliers & Innovations

- **Wails** is the closest alternative to Tauri for this specific polyglot stack (Go gateway + web frontend). Worth watching as a fallback if Rust proves a bottleneck. But Tauri's sidecar support is broader.
- **tantivy** is genuinely 2x faster than Lucene with <10ms startup — meaningfully better than building search on top of the Gateway's SQLite.
- **Tauri social apps** exist (HuLa, Vector) — the "Tauri can't do social" concern is unfounded.

### Gaps in the Landscape

- No mature Rust symbolic math CAS (gap vs. Python/SymPy)
- No standard Rust CRDT library dominating (automerge-rs and diamond-types compete)
- PDF *rendering* in Rust is limited (lopdf handles structure but not visual rendering — use PDF.js in the webview for that)

---

## Opportunity Matrix

```
                    HIGH IMPACT
                         │
  ┌──────────────────────┼──────────────────────┐
  │   QUICK WINS         │      BIG BETS         │
  │                      │                       │
  │ • Tantivy search     │ • Real-time CRDT collab│
  │ • Sidecar bundling   │ • Full PDF render      │
  │   (AgenticGateway)   │   pipeline in Rust     │
  │ • pulldown-cmark     │ • Symbolic math CAS    │
  │ • rusqlite local DB  │                       │
  │ • JWT/keychain auth  │                       │
  │ • notify file watch  │                       │
  │                      │                       │
HIGH FEASIBILITY ────────┼──────── LOW FEASIBILITY
  │                      │                       │
  │   FILL LATER         │      AVOID            │
  │                      │                       │
  │ • Image processing   │ • Flutter (Dart)       │
  │ • reqwest HTTP sync  │ • Qt (C++ mismatch)    │
  │ • comrak GFM         │ • NativeScript (mobile)│
  │                      │ • PWA-only (offline)   │
  │                      │                       │
  └──────────────────────┴──────────────────────┘
                    LOW IMPACT
```

---

## Recommended Focus Areas

### #1: Tantivy Full-Text Search

Knowledge platforms live or die on search. Local full-text search is offline-first, <10ms startup, Lucene-equivalent quality, and enables search-as-you-type UX. Competitive parity with Obsidian, Roam Research. The Rust layer indexes all ZenSci outputs (PDFs, HTML, email, slides) — a cross-cluster capability that enhances Parsing, Validation, and all Rendering clusters simultaneously.

**Effort:** Medium (1–2 sprints). **Payoff:** High.

### #2: Sidecar Bundling of AgenticGateway

Without sidecar bundling, users must launch AgenticGateway manually. With it: single installer, one-click launch, all binaries managed by Tauri, cross-platform. Tauri's `externalBin` makes this medium effort with massive UX payoff — the portal becomes a self-contained product rather than an integration burden.

**Effort:** Medium (once per platform). **Payoff:** Very high (product vs. integration).

### #3: tokio-tungstenite Real-Time Sync

The social/multi-user requirement means document changes must propagate. WebSocket + local-first SQLite + background sync is the proven Tauri pattern (validated by HuLa, Noor, Vector). Positions zen-sci-portal for v1.1 collaborative editing without rearchitecting the base.

**Effort:** High (requires CRDT policy for v1.1). **Payoff:** High (must-have for social layer).

### #4: JWT + Keychain Auth (Platform Credential Storage)

Multi-user = auth boundary. Rust's `jsonwebtoken` + platform keychain (Tauri `keyring` plugin or `keychain-services`) gives native credential storage without shipping passwords in localStorage. This is the security foundation for the social layer.

**Effort:** Medium. **Payoff:** High (required for multi-user).

---

## Framework Verdict: Why Tauri (Confirmed)

| Criterion | Tauri | Electron | Wails | Flutter |
|-----------|-------|----------|-------|---------|
| Bundle size | <10MB | 100MB+ | ~10MB | ~20MB |
| Idle memory | 30–50MB | 150–300MB | ~30MB | ~50MB |
| Startup | <0.5s | 1–2s | <0.5s | ~0.5s |
| Backend language | **Rust** | Node.js | **Go** | Dart |
| Sidecar support | ✅ Polyglot | ⚠️ Node only | ⚠️ Go only | ❌ |
| MCP App embed | ✅ WebView | ✅ Chromium | ✅ WebView | ❌ |
| Offline-first | ✅ | ✅ | ✅ | ✅ |
| Social apps exist | ✅ | ✅ | ⚠️ | ⚠️ |

Tauri wins on every dimension relevant to zen-sci-portal. Wails is close but loses on Rust alignment and polyglot sidecar breadth.

---

## Sources

- [DoltHub: Electron vs. Tauri](https://www.dolthub.com/blog/2025-11-13-electron-vs-tauri/)
- [RaftLabs: Tauri vs Electron Guide](https://raftlabs.medium.com/tauri-vs-electron-a-practical-guide-to-picking-the-right-framework-5df80e360f26)
- [Tauri v2: Node.js Sidecar](https://v2.tauri.app/learn/sidecar-nodejs/)
- [Tauri v2: Embedding Binaries](https://v2.tauri.app/develop/sidecar/)
- [Wails Introduction](https://wails.io/docs/introduction/)
- [Wails GitHub](https://github.com/wailsapp/wails)
- [Flutter Desktop Support](https://flutter.dev/multi-platform/desktop)
- [Flutter 2026 Roadmap](https://devnewsletter.com/p/state-of-flutter-2026/)
- [Qt Desktop 2026](https://tibicle.com/blog/best-framework-for-desktop-application-in-2026)
- [Tantivy GitHub](https://github.com/quickwit-oss/tantivy)
- [Tantivy Docs](https://docs.rs/tantivy/)
- [lopdf GitHub](https://github.com/J-F-Liu/lopdf)
- [pulldown-cmark GitHub](https://github.com/pulldown-cmark/pulldown-cmark)
- [comrak GitHub](https://github.com/kivikakk/comrak)
- [tokio-tungstenite GitHub](https://github.com/snapview/tokio-tungstenite)
- [rusqlite crates.io](https://crates.io/crates/rusqlite)
- [sled lib.rs](https://lib.rs/crates/sled)
- [Local-First AI with Rust and Tauri](https://medium.com/@Musbell008/a-technical-blueprint-for-local-first-ai-with-rust-and-tauri-b9211352bc0e)
