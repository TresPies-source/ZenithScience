# Zenflow Commission: zen-sci-portal Desktop v1.0 — The Forge

**Objective:** Build the `zen-sci-portal` Tauri v2 desktop application from scratch — a complete agentic knowledge-work forge where researchers create, search, and collaborate on academic documents powered by a bundled AgenticGateway sidecar and the 6 ZenSci MCP servers.

**Commissioned by:** Cruz Morales / TresPiesDesign.com
**Date:** 2026-02-18
**License:** Apache 2.0
**Prereqs:** Rust 1.75+, Node.js 20 LTS, pnpm 8+, Tauri CLI v2, AgenticGateway binary built and available at `src-tauri/binaries/`

---

## 1. Context & Grounding

**Primary Specification (read fully before writing a single line):**
- `ZenithScience/docs/specs/portal/zen-sci-portal-v1.0-spec.md` — Full application spec including Sections 1–8 (base spec) + Revision R1 (AgenticGateway integration, document catalog, collaboration model, section ownership, orphaned sections, auth) + Revision R2 (section boundaries in draft mode, new IPC commands). This is the authoritative source for every module name, IPC command signature, data model, and directory name.

**Architecture context (read before writing Rust):**
- `ZenithScience/docs/research/2026-02-18_scout_zen-sci-portal-rust-web.md` — Framework decisions, Rust crate selections, architecture rationale.
- `ZenflowProjects/AgenticGatewayByDojoGenesis/server/middleware/auth.go` — JWT auth patterns to follow for the Rust AuthManager (GatewayClaims struct, Bearer token extraction, role-based claims).
- `ZenflowProjects/AgenticGatewayByDojoGenesis/server/router.go` — Full Gateway API surface. Every route listed here is callable via the Rust gateway client. The `create_agent`, `chat_with_agent`, `submit_dag`, `search_memory` IPC commands must map to exactly these endpoints.

**Pattern files (follow these for structure and style):**
- `ZenithScience/docs/workspace/2026-02-18_phase-0-implementation-prompt.md` — Follow the implementation order pattern from prior ZenSci phases: scaffold first, types second, implementation third, tests last.
- `ZenflowProjects/AgenticGatewayByDojoGenesis/server/middleware/auth.go` — HMAC JWT validation, role extraction, keychain-compatible token structure.
- `ZenflowProjects/AgenticGatewayByDojoGenesis/server/router.go` — REST endpoint map for the Rust HTTP client (`reqwest`) to call.

**Status:**
- `ZenithScience/docs/STATUS.md` — Read before creating files. The `zen-sci-portal/` directory does not yet exist anywhere. This is a greenfield Tauri project.

---

## 2. Detailed Requirements

Implement in the order below. Do not skip ahead. Each numbered group must compile and pass tests before proceeding.

---

### Group 1 — Tauri Project Scaffolding

**1.1 Initialize the project**

Create the directory `ZenflowProjects/ZenithScience/zen-sci-portal/` and initialize a Tauri v2 project with SvelteKit frontend:

```
pnpm create tauri-app zen-sci-portal --template svelte-ts
```

Or scaffold manually if the CLI is unavailable. The project must match the directory structure in spec Section 3.2 exactly.

**1.2 `src-tauri/tauri.conf.json`**

Use the exact configuration from spec Section 3.2 (Key Tauri Configuration). Key fields:
- `identifier`: `com.trespies.zen-sci-portal`
- `externalBin`: AgenticGateway binary registered as `agentic-gateway` with platform-specific paths for macOS (arm + x86), Windows (MSVC x64), Linux (gnu x86).
- CSP: allows `ui://` protocol for MCP App iframe rendering.
- Window: 1200×800, min 800×600.

**1.3 `src-tauri/Cargo.toml`**

Use the exact dependencies from spec Section 3.2 (Cargo.toml). Pin versions. All features must be explicit. Do not add dependencies not listed in the spec without a clear reason stated in a code comment.

**1.4 `package.json`**

Use the exact frontend dependencies from spec Section 3.2 (Package.json). Add `@tailwindcss/typography` for document editor prose styling.

---

### Group 2 — Rust Backend Foundation

**2.1 `src-tauri/src/error.rs`**

Define a unified `AppError` enum covering:
- `GatewayUnavailable(String)`
- `AuthError(String)`
- `SearchError(String)`
- `KeychainError(String)`
- `IoError(String)`
- `SerializationError(String)`
- `NotFound(String)`

Implement `std::fmt::Display`, `From<std::io::Error>`, `From<serde_json::Error>`. Make it serializable via `serde` so IPC commands can return `Result<T, String>` (Tauri requires string errors).

**2.2 `src-tauri/src/config.rs`**

Define `AppConfig` struct with:
- `gateway_port: u16` (default: 8080)
- `gateway_host: String` (default: "127.0.0.1")
- `data_dir: PathBuf` (default: `~/.zen-sci/`)
- `index_dir: PathBuf` (default: `~/.zen-sci/index/`)
- `outputs_dir: PathBuf` (default: `~/.zen-sci/outputs/`)
- `gateway_binary: String` (default: "agentic-gateway")
- `log_level: String` (default: "info")

Load from `~/.zen-sci/config.toml` if present; fall back to defaults. Use `toml` crate for parsing.

**2.3 `src-tauri/src/state.rs`**

Define `AppState` as a Tauri-managed state struct:
```rust
pub struct AppState {
    pub config: Arc<AppConfig>,
    pub search_indexer: Arc<Mutex<SearchIndexer>>,
    pub auth_manager: Arc<Mutex<AuthManager>>,
    pub gateway_sidecar: Arc<Mutex<GatewaySidecar>>,
    pub file_watcher: Arc<Mutex<FileWatcher>>,
    pub http_client: Arc<reqwest::Client>,
}
```

Implement `AppState::new(config: AppConfig) -> Result<Self, AppError>` that initializes all subsystems.

**2.4 `src-tauri/src/main.rs` and `lib.rs`**

Wire `AppState` into Tauri's `.manage()`. Register all IPC commands (see Group 5). Initialize the Gateway sidecar on app startup via `tauri::Builder::default().setup(|app| { ... })`. Handle app shutdown: stop file watcher, stop gateway sidecar.

---

### Group 3 — Rust Subsystems

Implement each module in `src-tauri/src/`. Each must have its own `mod.rs`, implementation file(s), and unit tests.

**3.1 Auth Module (`src-tauri/src/auth/`)**

Files: `mod.rs`, `manager.rs`, `keychain.rs`, `session.rs`

`AuthManager`:
- `login(email: &str, password: &str, gateway_url: &str) -> Result<Session, AppError>`
  - POSTs to `{gateway_url}/auth/login` (the new Gateway auth endpoint from R1)
  - Stores returned JWT in platform keychain using service name `"zen-sci-portal"`, account key `"jwt-{user_id}"`
  - Returns populated `Session`
- `logout(user_id: &str) -> Result<(), AppError>`
  - Deletes JWT from platform keychain
  - Clears in-memory session
- `check_session() -> Result<Option<Session>, AppError>`
  - Reads JWT from keychain
  - Validates via `jsonwebtoken` crate (HMAC HS256, `JWT_SECRET` from env or config)
  - Returns `Some(Session)` if valid, `None` if absent or expired
- `get_token() -> Result<String, AppError>`
  - Returns current valid JWT for use in Gateway HTTP calls

`keychain.rs`: Platform-specific implementations. Use `keyring` crate (cross-platform). Service: `"zen-sci-portal"`.

`Session` struct: `{ user_id: String, email: String, username: String, role: String, expires_at: i64 }`

Unit tests: mock keychain writes/reads, validate expired token returns `None`, validate malformed token returns error.

**3.2 Search Module (`src-tauri/src/search/`)**

Files: `mod.rs`, `schema.rs`, `indexer.rs`, `searcher.rs`

`schema.rs` — Define Tantivy schema with fields:
- `id` (TEXT, STORED, INDEXED) — document UUID
- `title` (TEXT, STORED, INDEXED) — document title
- `body` (TEXT, STORED, INDEXED) — full text content
- `file_type` (TEXT, STORED, FAST) — "pdf" | "html" | "email" | "slides" | "latex"
- `created_at` (DATE, STORED, FAST) — creation timestamp
- `updated_at` (DATE, STORED, FAST) — last modified timestamp
- `author_id` (TEXT, STORED, FAST) — owner user ID

`indexer.rs` — `SearchIndexer`:
- `new(index_dir: &Path) -> Result<Self, AppError>` — opens or creates Tantivy index
- `index_document(doc: &IndexableDocument) -> Result<(), AppError>` — adds/updates a document
- `remove_document(id: &str) -> Result<(), AppError>` — removes by ID
- `commit() -> Result<(), AppError>` — flushes pending writes

`searcher.rs` — `Searcher`:
- `search(query: &str, limit: usize) -> Result<Vec<SearchResult>, AppError>`
  - BM25 scoring across `title` + `body` fields
  - Returns `SearchResult { id, title, snippet, file_type, score, created_at }`
  - Snippet: 150-char context window around first match

Unit tests: index 3 docs, search for a term present in one, verify exact result; search for absent term returns empty; index → remove → search returns empty.

**3.3 Gateway Module (`src-tauri/src/gateway/`)**

Files: `mod.rs`, `sidecar.rs`, `client.rs`, `config.rs`

`sidecar.rs` — `GatewaySidecar`:
- `start(binary_path: &str, port: u16, app_handle: &tauri::AppHandle) -> Result<(), AppError>`
  - Spawns AgenticGateway binary via `tauri::api::process::Command`
  - Sets env: `PORT`, `ENVIRONMENT=production`, `JWT_SECRET`
  - Waits up to 5 seconds for `/health` to return 200
- `stop() -> Result<(), AppError>` — sends SIGTERM, waits for clean exit
- `health_check(gateway_url: &str) -> Result<bool, AppError>` — GETs `/health`, returns true if 200
- `restart(app_handle: &tauri::AppHandle) -> Result<(), AppError>` — stop + start with exponential backoff (1s, 2s, 4s, max 3 retries)
- Background health monitor: spawn a Tokio task that health-checks every 10 seconds; auto-restart on failure; emit Tauri event `"gateway_status_changed"` with `{ status: "up" | "down" | "restarting" }`

`client.rs` — `GatewayClient` (wraps `reqwest::Client`):
All methods take `&self, token: &str` and use `Authorization: Bearer {token}` header.
- `chat_completions(body: serde_json::Value) -> Result<serde_json::Value, AppError>` → `POST /v1/chat/completions`
- `create_agent(project_id: &str, name: &str, provider: &str) -> Result<Agent, AppError>` → `POST /v1/gateway/agents`
- `chat_with_agent(agent_id: &str, message: &str) -> Result<AgentResponse, AppError>` → `POST /v1/gateway/agents/{id}/chat`
- `submit_dag(pipeline: serde_json::Value) -> Result<DagJob, AppError>` → `POST /v1/gateway/orchestrate`
- `get_dag_status(job_id: &str) -> Result<DagJob, AppError>` → `GET /v1/gateway/orchestrate/{id}/dag`
- `search_memory(query: &str, project_id: &str) -> Result<Vec<MemoryItem>, AppError>` → `POST /v1/memory/search`
- `store_memory(content: &str, tags: Vec<String>, project_id: &str) -> Result<MemoryItem, AppError>` → `POST /v1/memory`
- `list_providers() -> Result<Vec<Provider>, AppError>` → `GET /v1/providers`
- `list_tools() -> Result<Vec<Tool>, AppError>` → `GET /v1/gateway/tools`

**3.4 File Module (`src-tauri/src/file/`)**

Files: `mod.rs`, `watcher.rs`, `metadata.rs`

`watcher.rs` — `FileWatcher`:
- `watch(dir: &Path, tx: Sender<FileEvent>) -> Result<(), AppError>`
  - Uses `notify` crate (RecommendedWatcher)
  - Emits `FileEvent { path, kind: Created | Modified | Deleted }` on channel
  - On `Created` or `Modified`: queue document for re-indexing (debounce 500ms)
- `stop() -> Result<(), AppError>`

`metadata.rs` — `LocalFile`:
- `{ id: String, path: PathBuf, file_type: FileType, size_bytes: u64, created_at: DateTime<Utc>, updated_at: DateTime<Utc>, indexed: bool }`
- `scan_directory(dir: &Path) -> Result<Vec<LocalFile>, AppError>` — walks dir, returns all supported files

**3.5 Models (`src-tauri/src/models/`)**

Files: `mod.rs`, `document.rs`, `section.rs`, `user.rs`, `search_result.rs`, `gateway.rs`, `game.rs`

`document.rs`:
```rust
pub struct Document {
    pub id: String,
    pub title: String,
    pub version_hash: String,
    pub upstream_url: Option<String>,
    pub last_synced_at: Option<DateTime<Utc>>,
    pub owner_id: String,
    pub co_authors: Vec<String>,
    pub mode: DocumentMode,  // Draft | Structured
    pub crdt_state: Option<Vec<u8>>,  // null in v1; forward-looking for v2 CRDT
    pub sections: Vec<Section>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

pub enum DocumentMode { Draft, Structured }
```

`section.rs`:
```rust
pub struct Section {
    pub id: String,
    pub document_id: String,
    pub title: String,
    pub content: String,
    pub owner_id: Option<String>,  // None = orphaned
    pub order: i32,
    pub locked: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}
```

`user.rs`: `{ id, email, username, role, avatar_url }`
`search_result.rs`: `{ id, title, snippet, file_type, score, created_at }`
`gateway.rs`: `Agent`, `AgentResponse`, `DagJob`, `MemoryItem`, `Provider`, `Tool` — match Gateway API response shapes from `router.go`.
`game.rs`: `GameSession`, `Achievement`, `GameFormatSuggestion` — from mobile spec R1 (for shared type consistency).

---

### Group 4 — SvelteKit Frontend

**4.1 Global setup**

- `src/app.css` — TailwindCSS base + typography plugin. Dark mode via `class` strategy. CSS variables for brand colors.
- `svelte.config.js` — `adapter-static` with `fallback: 'index.html'` (required for Tauri).
- `src/lib/types/index.ts` — Export all TypeScript types mirroring the Rust models (Document, Section, User, SearchResult, Agent, DagJob, Provider, GameFormatSuggestion).

**4.2 IPC client (`src/lib/api/tauri.ts`)**

Typed wrapper around `@tauri-apps/api/core invoke`. Export one function per IPC command:
```typescript
// Auth
export const checkSession = (): Promise<Session | null>
export const login = (email: string, password: string): Promise<Session>
export const logout = (): Promise<void>

// Documents
export const listDocuments = (): Promise<Document[]>
export const createDocument = (title: string): Promise<Document>
export const getDocument = (id: string): Promise<Document>
export const createSection = (documentId: string, title: string, position?: number): Promise<Section>
export const mergeSections = (sectionIds: string[], mergedTitle: string): Promise<Section>
export const transitionToStructured = (documentId: string): Promise<Document>
export const reclaimSection = (sectionId: string, newOwnerId: string): Promise<Section>

// Search
export const search = (query: string, limit?: number): Promise<SearchResult[]>

// Gateway
export const getGatewayStatus = (): Promise<{ status: string; version: string }>
export const listProviders = (): Promise<Provider[]>
export const switchProvider = (providerName: string): Promise<void>
export const createAgent = (projectId: string, name: string, provider: string): Promise<Agent>
export const chatWithAgent = (agentId: string, message: string): Promise<AgentResponse>
export const submitDag = (pipeline: object): Promise<DagJob>
export const getDagStatus = (jobId: string): Promise<DagJob>
export const searchMemory = (query: string, projectId: string): Promise<MemoryItem[]>
export const storeMemory = (content: string, tags: string[], projectId: string): Promise<MemoryItem>
export const listTools = (): Promise<Tool[]>
export const suggestGameFormat = (contentId: string, contentType: string, metadata: object): Promise<GameFormatSuggestion>
```

**4.3 Svelte stores (`src/lib/stores/`)**

- `userSession.ts` — `writable<Session | null>(null)`. Initialized from `checkSession()` on app mount.
- `searchResults.ts` — `writable<SearchResult[]>([])`. Updated by `SearchBar`.
- `documents.ts` — `writable<Document[]>([])`. Loaded on workspace mount.
- `gatewayStatus.ts` — `writable<'up' | 'down' | 'restarting'>('up')`. Updated by Tauri event listener on `gateway_status_changed`.
- `activeDocument.ts` — `writable<Document | null>(null)`. Set when user opens a document.

**4.4 Routes**

`src/routes/+layout.svelte`:
- App shell: left sidebar (nav + search bar) + main content area.
- On mount: `checkSession()` → redirect to `/auth` if unauthenticated.
- Listens for `gateway_status_changed` Tauri event → updates `gatewayStatus` store → shows status pill in sidebar.

`src/routes/+page.svelte`:
- Redirects to `/workspace` if authenticated, `/auth` if not.

`src/routes/auth/+page.svelte`:
- Email + password form.
- Calls `login(email, password)` → on success: sets `userSession` store, navigates to `/workspace`.
- Shows error message on failure.
- "Use locally (no login)" option: sets a local-only session flag, skips auth, navigates to `/workspace`.

`src/routes/workspace/+page.svelte`:
- Shows list of recent documents from `documents` store.
- "New Document" button → calls `createDocument(title)` → navigates to `/workspace/{docId}`.
- Format selector: LaTeX Paper | Blog Post | Grant Proposal | Slides | Newsletter | Academic Paper.
- AgentWorkspace panel (right column): shows active agent for the current project, chat input, provider selector.

`src/routes/workspace/[docId]/+page.svelte`:
- Loads document via `getDocument(docId)` → sets `activeDocument` store.
- If `document.mode === 'Draft'`: free-form markdown editor. Any user can add sections. Shows "Organize sections" button for document owner.
- If `document.mode === 'Structured'`: section-based editor. Each section shows owner badge. Non-owners see comment/suggest controls but not the edit cursor.
- "Organize sections" modal (owner only): drag-and-drop section list, merge button, co-author assignment dropdown, "Confirm structure" button → calls `transitionToStructured(docId)`.
- Orphaned section warning banner: if any `section.owner_id === null`, show "N sections are unowned. [Assign owners]" banner with link to the organize modal.
- Convert button: dropdown (PDF / HTML / Slides / Email) → calls `submitDag` with appropriate ZenSci pipeline.

`src/routes/search/+page.svelte`:
- Full-page search results. Receives `?q=` query param.
- Displays `SearchResult[]` from `searchResults` store.
- Filters sidebar: by file_type, by date range.

`src/routes/settings/+page.svelte`:
- Gateway status + restart button.
- Provider list (`listProviders()`) with active indicator and `switchProvider()` action.
- Data directory path + "Open in Finder/Explorer" button.
- "Log out" button.

`src/routes/app/[appId]/+page.svelte`:
- Renders a `ui://` MCP App in an iframe (Phase 4 support).
- Loads app metadata via `getGatewayStatus()` / app registry.

**4.5 Components**

`src/lib/components/SearchBar.svelte`:
- Debounced (250ms) input. On input: calls `search(query)` → updates `searchResults` store.
- Keyboard shortcut: Cmd/Ctrl+K focuses the bar from anywhere.
- Shows spinner while searching. Clears on empty input.

`src/lib/components/DocumentEditor.svelte`:
- Props: `document: Document`, `sectionId: string | null`, `readonly: boolean`
- Textarea-based markdown editor (no CodeMirror dependency in v1 — keep it simple).
- Auto-saves content every 5 seconds to local state. "Save version" button creates a commit (calls `saveDocumentVersion`).
- Shows section owner badge if `mode === 'Structured'`.

`src/lib/components/ResultsList.svelte`:
- Props: `results: SearchResult[]`
- Virtual scroll (no library; implement simple windowed list for 50 items).
- Each result: title, snippet, file type badge, date.

`src/lib/components/AgentChat.svelte`:
- Props: `projectId: string`
- Creates an agent on mount if none exists (`createAgent`).
- Chat interface: message list + input. Calls `chatWithAgent`.
- Provider pill showing active provider. Click → opens `switchProvider` dropdown.
- Memory panel: shows recent `searchMemory` results for the current project context.

`src/lib/components/SectionOrganizer.svelte`:
- Props: `document: Document`, `members: User[]`, `onConfirm: () => void`
- Drag-and-drop section list (use native HTML5 drag-and-drop, no library).
- Each row: section title (editable), owner dropdown (assign to member), delete button, merge checkbox.
- "Merge selected" button → calls `mergeSections`.
- "Confirm structure" → calls `transitionToStructured`.

---

### Group 5 — IPC Command Registration

In `src-tauri/src/commands/`, implement all `#[tauri::command]` functions. Each must:
- Accept `state: tauri::State<'_, AppState>` as the first argument.
- Return `Result<T, String>` (Tauri requires `String` errors; convert `AppError` with `.to_string()`).
- Log errors with `tracing::error!` before returning.

Commands to implement (one file per logical group):

`commands/auth.rs`:
- `check_session(state) -> Result<Option<Session>, String>`
- `login(state, email: String, password: String) -> Result<Session, String>`
- `logout(state) -> Result<(), String>`

`commands/search.rs`:
- `search(state, query: String, limit: Option<usize>) -> Result<Vec<SearchResult>, String>`

`commands/document.rs`:
- `list_documents(state) -> Result<Vec<Document>, String>`
- `create_document(state, title: String) -> Result<Document, String>`
- `get_document(state, id: String) -> Result<Document, String>`
- `create_section(state, document_id: String, title: String, position: Option<usize>) -> Result<Section, String>`
- `merge_sections(state, section_ids: Vec<String>, merged_title: String) -> Result<Section, String>`
- `transition_to_structured(state, document_id: String) -> Result<Document, String>`
- `reclaim_section(state, section_id: String, new_owner_id: String) -> Result<Section, String>`
- `save_document_version(state, document_id: String, commit_message: String) -> Result<(), String>`

`commands/gateway.rs`:
- `get_gateway_status(state) -> Result<GatewayStatus, String>`
- `list_providers(state) -> Result<Vec<Provider>, String>`
- `switch_provider(state, provider_name: String) -> Result<(), String>`
- `create_agent(state, project_id: String, name: String, provider: String) -> Result<Agent, String>`
- `chat_with_agent(state, agent_id: String, message: String) -> Result<AgentResponse, String>`
- `submit_dag(state, pipeline: serde_json::Value) -> Result<DagJob, String>`
- `get_dag_status(state, job_id: String) -> Result<DagJob, String>`
- `search_memory(state, query: String, project_id: String) -> Result<Vec<MemoryItem>, String>`
- `store_memory(state, content: String, tags: Vec<String>, project_id: String) -> Result<MemoryItem, String>`
- `list_tools(state) -> Result<Vec<Tool>, String>`
- `suggest_game_format(state, content_id: String, content_type: String, metadata: serde_json::Value) -> Result<GameFormatSuggestion, String>`

Register all commands in `main.rs` via `tauri::Builder::default().invoke_handler(tauri::generate_handler![...])`.

---

### Group 6 — Tests

**Rust unit tests** — Each module must have `#[cfg(test)]` blocks:
- `auth/manager.rs`: test `check_session` with valid token, expired token, missing token.
- `search/indexer.rs`: test `index_document` + `search` returns correct result.
- `search/searcher.rs`: test BM25 scoring, snippet generation.
- `gateway/sidecar.rs`: test health check returns true for mocked 200 response, false for 503.
- `gateway/client.rs`: test request construction (headers, body format) using `wiremock` or a simple mock.
- `file/watcher.rs`: test event emission on file creation in temp dir.

**Integration tests** (`src-tauri/tests/`):
- `test_auth_flow`: login → keychain write → check_session returns valid session → logout → check_session returns None.
- `test_search_flow`: index 3 documents → search term present in one → verify correct result returned.
- `test_gateway_client`: start mock HTTP server → call `list_providers` → verify correct deserialization.

**Frontend tests** (`src/tests/`):
- Component tests for `SearchBar` (vitest + @testing-library/svelte): typing triggers debounced search call.
- Component test for `DocumentEditor`: content changes update local state; "Save version" calls IPC.
- Store tests: `userSession` updates correctly on login/logout.

---

## 3. File Manifest

**Create (new project — all files):**

```
ZenflowProjects/ZenithScience/zen-sci-portal/
├── package.json
├── tsconfig.json
├── vite.config.ts
├── svelte.config.js
├── tailwind.config.js
├── postcss.config.js
│
├── src/
│   ├── app.css
│   ├── app.html
│   ├── routes/
│   │   ├── +layout.svelte
│   │   ├── +page.svelte
│   │   ├── auth/+page.svelte
│   │   ├── workspace/+page.svelte
│   │   ├── workspace/[docId]/+page.svelte
│   │   ├── search/+page.svelte
│   │   ├── settings/+page.svelte
│   │   └── app/[appId]/+page.svelte
│   └── lib/
│       ├── api/tauri.ts
│       ├── stores/userSession.ts
│       ├── stores/searchResults.ts
│       ├── stores/documents.ts
│       ├── stores/gatewayStatus.ts
│       ├── stores/activeDocument.ts
│       ├── types/index.ts
│       └── components/
│           ├── SearchBar.svelte
│           ├── DocumentEditor.svelte
│           ├── ResultsList.svelte
│           ├── AgentChat.svelte
│           └── SectionOrganizer.svelte
│
├── src-tauri/
│   ├── Cargo.toml
│   ├── tauri.conf.json
│   ├── binaries/            (empty dir; Gateway binary placed here at build time)
│   └── src/
│       ├── main.rs
│       ├── lib.rs
│       ├── error.rs
│       ├── config.rs
│       ├── state.rs
│       ├── auth/mod.rs
│       ├── auth/manager.rs
│       ├── auth/keychain.rs
│       ├── auth/session.rs
│       ├── search/mod.rs
│       ├── search/schema.rs
│       ├── search/indexer.rs
│       ├── search/searcher.rs
│       ├── gateway/mod.rs
│       ├── gateway/sidecar.rs
│       ├── gateway/client.rs
│       ├── gateway/config.rs
│       ├── file/mod.rs
│       ├── file/watcher.rs
│       ├── file/metadata.rs
│       ├── models/mod.rs
│       ├── models/document.rs
│       ├── models/section.rs
│       ├── models/user.rs
│       ├── models/search_result.rs
│       ├── models/gateway.rs
│       ├── models/game.rs
│       ├── commands/mod.rs
│       ├── commands/auth.rs
│       ├── commands/search.rs
│       ├── commands/document.rs
│       └── commands/gateway.rs
```

**Do NOT modify** any existing files outside `zen-sci-portal/`. The ZenSci monorepo (`zen-sci/`), AgenticGateway (`AgenticGatewayByDojoGenesis/`), and existing specs are read-only references.

---

## 4. Success Criteria

- [ ] `cargo build --release` completes with zero errors and zero warnings in `src-tauri/`.
- [ ] `pnpm run build` completes with zero TypeScript errors.
- [ ] `tauri dev` launches the app window with the workspace route visible.
- [ ] Auth flow: entering valid credentials (tested against a running Gateway instance) stores JWT in keychain; app reopens without prompting for login again.
- [ ] Search: indexing a markdown file and searching for a term in it returns the file in results within 50ms.
- [ ] Gateway sidecar: app startup launches the bundled `agentic-gateway` binary and the status pill shows "up" within 5 seconds.
- [ ] Agent chat: typing a message in `AgentChat.svelte` sends to Gateway and displays the response.
- [ ] Section ownership: creating a document → adding two sections → switching to structured mode → assigning sections to co-authors persists the assignment.
- [ ] Orphaned section: removing a co-author from a document with owned sections shows the "N sections unowned" warning banner.
- [ ] All Rust unit tests pass: `cargo test` reports 0 failures.
- [ ] All Svelte component tests pass: `pnpm run test` reports 0 failures.
- [ ] `cargo clippy -- -D warnings` produces zero lint errors.
- [ ] App window loads in under 500ms from launch.

---

## 5. Constraints & Non-Goals

- **DO NOT** implement real-time CRDT collaboration. The `crdt_state` field in `Document` is a forward-looking column that must exist in the model and be `None` in all v1 operations. Do not wire any WebSocket sync.
- **DO NOT** implement the mobile transformation trigger (`suggest_game_format` IPC command must exist but may return a hardcoded stub in v1 — `{ format: "AdaptiveQuiz", reason: "stub", concept_count: 10, estimated_minutes: 5 }`).
- **DO NOT** add any third-party UI component libraries (Radix, shadcn, etc.). Use Tailwind utility classes only.
- **DO NOT** implement the file sync (`file/sync.rs`) beyond the stub. File watching (`file/watcher.rs`) is required; cloud sync is v1.1.
- **DO NOT** modify the AgenticGateway source. The portal consumes Gateway HTTP endpoints only.
- **DO NOT** add `localStorage` usage anywhere. Session state lives in the Rust keychain and in-memory Svelte stores only.
- **DO NOT** implement the `activity_feed` / `GroupFeed.svelte` component. Group social features belong to the web portal spec, not the desktop.
- The Gateway auth endpoints (`/auth/login`, `/auth/register`) are a **separate Gateway deliverable** (part of the portal v1 spec per decision 1). If they are not yet live, the `login` IPC command should return a clear `AuthError("Gateway auth endpoints not yet available")` rather than a silent failure.

---

*Commission complete. Do not start implementation until you have read the full spec (including R1 and R2 sections) and the AgenticGateway router.go file. The spec is the source of truth; this commission is the execution plan.*
