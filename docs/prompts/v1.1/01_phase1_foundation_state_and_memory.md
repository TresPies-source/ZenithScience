# Zenflow Commission: Phase 1 — Foundation: Agent Store, Persistent Chat & Memory Panel

**Objective:** Lift agent and chat state out of component-local `$state` into a persistent Svelte store, add SQLite-backed chat history, fix the memory counter, and upgrade the memory panel with a paginated list view and detail/edit capability — all without touching the Go gateway or introducing new dependencies.

---

## 1. Context & Grounding

**Primary Specification:**
- `docs/specs/zen-sci-portal-v1.1-spec.md` — Full v1.1 spec. Phase 1 covers §3.3 (Agent Store + Persistent Chat), §3.5 (Live Memory Counter), and §3.6 (Memory Panel Pagination + Edit).

**Pattern Files (Follow these examples):**
- `zen-sci-portal/src/lib/stores/activeDocument.ts`: Follow this pattern for writable store shape — use `writable<T>` from `'svelte/store'`, export a named singleton. The agent store is more complex (has methods), so use the function-factory pattern.
- `zen-sci-portal/src-tauri/src/db/mod.rs`: Follow this exact pattern for all SQLite work — `Mutex<Connection>`, `execute_batch` for schema init in a `const SCHEMA: &str`, `params![]` macro for parameterized queries, `rusqlite::Result` error mapping.
- `zen-sci-portal/src-tauri/src/commands/document.rs`: Follow this pattern for all new Tauri commands — `#[tauri::command]`, `State<'_, AppState>`, `Result<T, String>`, `map_err(|e| e.to_string())`. Register commands in `lib.rs` `invoke_handler`.
- `zen-sci-portal/src/lib/api/tauri.ts`: Follow this exact pattern for all new IPC wrappers — `invoke<T>('command_name', { camelCaseArgs })`. All multi-word parameter keys must be camelCase (Tauri v2 deserializes with `rename_all = "camelCase"`).
- `zen-sci-portal/src/lib/components/MemoryPanel.svelte`: This is the component to enhance. Read it fully before touching it — it uses Svelte 5 runes (`$state`, `$derived`, `$props`), the `card` CSS class, `btn-primary`/`btn-secondary` utility classes, and `typeBadgeClass()` for type-colored badges.

**Files to Read Before Starting:**
- `zen-sci-portal/src/lib/components/AgentChat.svelte` — 392-line component to migrate. Agent state is local `$state`; `messages` is local array; `saveToMemory()` calls `storeMemory()` but never updates `memoryItems`.
- `zen-sci-portal/src/lib/types/index.ts` — All shared types. `MemoryItem` (from `storeMemory`) and `MemorySearchItem` (from `searchMemory`) are different types. The new `ChatMessage` interface currently lives only in `AgentChat.svelte` — it needs to move to the store.
- `zen-sci-portal/src/lib/api/tauri.ts` — Existing IPC layer. `storeMemory()` returns `MemoryItem`. `searchMemory()` takes `(query, projectId)`. New `listMemories()` and `updateMemory()` must be added here.
- `zen-sci-portal/src-tauri/src/lib.rs` — Register all new Rust commands in `invoke_handler![]`.
- `zen-sci-portal/src-tauri/src/state.rs` — `AppState` struct. `document_store` is `Arc<DocumentStore>`. New commands use `state.document_store` for SQLite access.
- `zen-sci-portal/src-tauri/src/models/gateway.rs` — `MemoryItem` Rust model. The new gateway `GET /v1/memory` and `PUT /v1/memory/:id` endpoints don't exist yet — the Rust `list_memories` and `update_memory` commands call new `GatewayClient` methods.
- `zen-sci-portal/src-tauri/src/gateway/client.rs` — `GatewayClient` struct. Add `list_memories()` and `update_memory()` methods here following the exact `post_json`/`get_json` helper pattern already used.

---

## 2. Detailed Requirements

### 2.1 Create `src/lib/stores/agent.ts` (NEW FILE)

Create `zen-sci-portal/src/lib/stores/agent.ts` with the following exact shape:

```typescript
import { writable, get } from 'svelte/store';
import type { Agent } from '$lib/types/index.js';

export interface ChatMessage {
  role: 'user' | 'assistant' | 'system';
  content: string;
  timestamp: number;
  toolCalls?: { tool: string; args: unknown; result?: unknown }[] | undefined;
}

interface AgentStoreState {
  agent: Agent | null;
  messages: ChatMessage[];
  activeDocumentId: string | null;
  priorContext: import('$lib/types/index.js').MemoryItem[];
  loading: boolean;
  error: string | null;
}

function createAgentStore() {
  const { subscribe, set, update } = writable<AgentStoreState>({
    agent: null,
    messages: [],
    activeDocumentId: null,
    priorContext: [],
    loading: false,
    error: null,
  });

  return {
    subscribe,
    init: async (projectId: string, provider: string) => { /* ... */ },
    setActiveDocument: (documentId: string | null) => { /* ... */ },
    addMessage: (message: ChatMessage) => { /* ... */ },
    setPriorContext: (items: import('$lib/types/index.js').MemoryItem[]) => { /* ... */ },
    clearMessages: () => { /* ... */ },
    reset: () => { /* ... */ },
  };
}

export const agentStore = createAgentStore();
```

- `init()`: Checks `get({ subscribe }).agent` — if already set, returns immediately (idempotent). Otherwise sets `loading: true`, calls `createAgent(projectId, 'Research Agent', provider)`, updates store with result.
- `addMessage()`: Appends to `messages` array immutably.
- All other methods are simple state updates.

### 2.2 Migrate `AgentChat.svelte` to use `agentStore`

Modify `zen-sci-portal/src/lib/components/AgentChat.svelte`:

1. **Import** `agentStore` from `'$lib/stores/agent.js'` and the `ChatMessage` interface from it.
2. **Replace** all local `let agent = $state<Agent | null>(null)` with `$agentStore.agent`.
3. **Replace** all local `let messages = $state<ChatMessage[]>([])` with `$agentStore.messages`.
4. **Replace** all local `let agentLoading = $state(true)` with `$agentStore.loading`.
5. **Replace** all local `let agentError = $state('')` with `$agentStore.error ?? ''`.
6. In `onMount`, call `await agentStore.init(projectId, activeProvider)` instead of directly calling `createAgent()`.
7. In `sendMessage()`, call `agentStore.addMessage(...)` instead of `messages = [...messages, ...]`.
8. In `clearChat()`, call `agentStore.clearMessages()`.
9. **Keep** local `$state` for: `inputValue`, `sending`, `showMemory`, `savingMemory`, `showProviderDropdown`, `providers`, `activeProvider`, `memoryItems`, `chatEndRef`. These remain component-local.
10. **DO NOT** add `fullScreen` prop or document context in this phase — those are Phase 2.

### 2.3 Fix Memory Counter in `AgentChat.svelte`

In `saveToMemory()`, after the `await storeMemory(...)` call succeeds, append the returned `MemoryItem` to `memoryItems` immediately:

```typescript
async function saveToMemory(content: string) {
  savingMemory = true;
  try {
    const stored = await storeMemory(content, 'conversation', 'private');
    // Immediately reflect in memory list — no search round-trip
    memoryItems = [
      {
        id: stored.id,
        memory_type: stored.memory_type,
        content: stored.content,
        relevance_score: undefined,
      },
      ...memoryItems,
    ].slice(0, 20);
  } catch {
    // Non-critical
  } finally {
    savingMemory = false;
  }
}
```

The memory counter in the template reads `memoryItems.length` — it now increments immediately.

### 2.4 Add SQLite Chat History (Rust)

Add a `chat_messages` table to the existing schema in `zen-sci-portal/src-tauri/src/db/mod.rs`:

Append to the `SCHEMA` constant (inside the existing `r#"..."#` block):

```sql
CREATE TABLE IF NOT EXISTS chat_messages (
    id TEXT PRIMARY KEY DEFAULT (lower(hex(randomblob(16)))),
    agent_id TEXT NOT NULL,
    role TEXT NOT NULL CHECK (role IN ('user', 'assistant', 'system')),
    content TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX IF NOT EXISTS idx_chat_messages_agent_id ON chat_messages(agent_id);
```

Add two new methods to `DocumentStore` in `zen-sci-portal/src-tauri/src/db/mod.rs`, following the exact same `Mutex<Connection>`, `params![]`, and `AppError` patterns:

```rust
pub async fn save_chat_message(
    &self,
    agent_id: &str,
    role: &str,
    content: &str,
) -> Result<(), AppError> {
    let conn = self.conn.lock().await;
    conn.execute(
        "INSERT INTO chat_messages (agent_id, role, content) VALUES (?1, ?2, ?3)",
        params![agent_id, role, content],
    )
    .map_err(|e| AppError::IoError(format!("Save chat message failed: {e}")))?;
    Ok(())
}

pub async fn load_chat_history(
    &self,
    agent_id: &str,
    limit: usize,
) -> Result<Vec<serde_json::Value>, AppError> {
    let conn = self.conn.lock().await;
    let mut stmt = conn
        .prepare(
            "SELECT role, content, created_at FROM chat_messages \
             WHERE agent_id = ?1 ORDER BY created_at DESC LIMIT ?2",
        )
        .map_err(|e| AppError::IoError(e.to_string()))?;

    let rows: Vec<serde_json::Value> = stmt
        .query_map(params![agent_id, limit as i64], |row| {
            Ok(serde_json::json!({
                "role": row.get::<_, String>(0)?,
                "content": row.get::<_, String>(1)?,
                "created_at": row.get::<_, String>(2)?,
            }))
        })
        .map_err(|e| AppError::IoError(e.to_string()))?
        .filter_map(|r| r.ok())
        .collect();

    Ok(rows)
}
```

### 2.5 Add Tauri Commands for Chat History (Rust)

Add two new commands to `zen-sci-portal/src-tauri/src/commands/gateway.rs`, following the exact same `#[tauri::command]`, `State<'_, AppState>`, `Result<T, String>` patterns used by existing commands in that file:

```rust
#[tauri::command]
pub async fn save_chat_message(
    state: State<'_, AppState>,
    agent_id: String,
    role: String,
    content: String,
) -> Result<(), String> {
    state
        .document_store
        .save_chat_message(&agent_id, &role, &content)
        .await
        .map_err(|e| e.to_string())
}

#[tauri::command]
pub async fn load_chat_history(
    state: State<'_, AppState>,
    agent_id: String,
) -> Result<Vec<serde_json::Value>, String> {
    state
        .document_store
        .load_chat_history(&agent_id, 200)
        .await
        .map_err(|e| e.to_string())
}
```

Register both commands in `zen-sci-portal/src-tauri/src/lib.rs` `invoke_handler![]` block, under the `// Gateway` section.

Export both from `zen-sci-portal/src-tauri/src/commands/mod.rs` if a `mod.rs` exists (check first).

### 2.6 Add IPC Wrappers for Chat History (TypeScript)

Add to `zen-sci-portal/src/lib/api/tauri.ts`:

```typescript
// ─── Chat History ─────────────────────────────────────────────────────────────

export const saveChatMessage = (agentId: string, role: string, content: string): Promise<void> =>
  invoke<void>('save_chat_message', { agentId, role, content });

export const loadChatHistory = (agentId: string): Promise<{ role: string; content: string; created_at: string }[]> =>
  invoke<{ role: string; content: string; created_at: string }[]>('load_chat_history', { agentId });
```

### 2.7 Add Gateway Client Methods for Memory List/Edit (Rust)

Add to `zen-sci-portal/src-tauri/src/gateway/client.rs`, following the exact `get_json`/`post_json` helper pattern:

```rust
/// GET /v1/memory?page=N&limit=N — paginated memory list.
/// Gateway returns: {"items": [...], "total": N, "page": N, "limit": N, "total_pages": N}
pub async fn list_memories(
    &self,
    token: &str,
    page: u32,
    limit: u32,
    memory_type: Option<&str>,
) -> Result<serde_json::Value, AppError> {
    let mut url = format!("{}/v1/memory?page={}&limit={}", self.base_url, page, limit);
    if let Some(t) = memory_type {
        url.push_str(&format!("&type={}", t));
    }
    self.get_json(token, &url).await
}

/// PUT /v1/memory/:id — update memory content.
/// Gateway returns the updated MemoryItem.
pub async fn update_memory(
    &self,
    token: &str,
    memory_id: &str,
    content: &str,
) -> Result<MemoryItem, AppError> {
    let url = format!("{}/v1/memory/{}", self.base_url, memory_id);
    let body = serde_json::json!({ "content": content });
    // Use PUT — requires a custom method call
    let resp = self
        .http_client
        .put(&url)
        .header("Authorization", Self::auth_header(token))
        .json(&body)
        .send()
        .await?;
    let val = self.handle_response(resp).await?;
    serde_json::from_value(val).map_err(|e| AppError::SerializationError(e.to_string()))
}
```

### 2.8 Add Tauri Commands for Memory List/Edit (Rust)

Add to `zen-sci-portal/src-tauri/src/commands/gateway.rs`:

```rust
#[tauri::command]
pub async fn list_memories(
    state: State<'_, AppState>,
    page: Option<u32>,
    limit: Option<u32>,
    memory_type: Option<String>,
) -> Result<serde_json::Value, String> {
    let token = get_token(&state).await?;
    state
        .gateway_client
        .list_memories(&token, page.unwrap_or(1), limit.unwrap_or(20), memory_type.as_deref())
        .await
        .map_err(|e| {
            tracing::error!("list_memories failed: {}", e);
            e.to_string()
        })
}

#[tauri::command]
pub async fn update_memory(
    state: State<'_, AppState>,
    memory_id: String,
    content: String,
) -> Result<MemoryItem, String> {
    let token = get_token(&state).await?;
    state
        .gateway_client
        .update_memory(&token, &memory_id, &content)
        .await
        .map_err(|e| {
            tracing::error!("update_memory failed: {}", e);
            e.to_string()
        })
}
```

Register `list_memories` and `update_memory` in `zen-sci-portal/src-tauri/src/lib.rs`.

### 2.9 Add IPC Wrappers for Memory List/Edit (TypeScript)

Add to `zen-sci-portal/src/lib/api/tauri.ts`:

```typescript
// ─── Memory List & Edit ───────────────────────────────────────────────────────

export interface MemoryListResponse {
  items: MemoryItem[];
  total: number;
  page: number;
  limit: number;
  total_pages: number;
}

export const listMemories = (
  page: number = 1,
  limit: number = 20,
  memoryType?: string
): Promise<MemoryListResponse> =>
  invoke<MemoryListResponse>('list_memories', { page, limit, memoryType: memoryType ?? null });

export const updateMemory = (memoryId: string, content: string): Promise<MemoryItem> =>
  invoke<MemoryItem>('update_memory', { memoryId, content });
```

Note: `MemoryListResponse` is a new interface — add it here in `tauri.ts`, not in `types/index.ts`, since it's gateway-response-specific and not a core model type.

### 2.10 Upgrade `MemoryPanel.svelte` with Browse Tab, Pagination, and Detail/Edit View

Modify `zen-sci-portal/src/lib/components/MemoryPanel.svelte`:

**New state variables to add:**
```typescript
// Browse tab state
let activeTab = $state<'browse' | 'search'>('browse');
let browseItems = $state<MemoryItem[]>([]);
let browsePage = $state(1);
let browseTotal = $state(0);
let browseTotalPages = $state(1);
let browseLoading = $state(false);

// Detail/edit view
let selectedItem = $state<MemoryItem | null>(null);
let editContent = $state('');
let saving = $state(false);
let saveError = $state('');
```

**Add `loadBrowsePage()` function:**
```typescript
async function loadBrowsePage(page: number) {
  browseLoading = true;
  try {
    const resp = await listMemories(page, 20);
    browseItems = resp.items;
    browseTotal = resp.total;
    browseTotalPages = resp.total_pages;
    browsePage = resp.page;
  } catch (err) {
    // Non-critical: show empty state
  } finally {
    browseLoading = false;
  }
}
```

Call `loadBrowsePage(1)` on `onMount`.

**Tab selector UI:** Add two tab buttons below the header — "Browse" and "Search" — switching `activeTab`. Use the `btn-secondary` / active variant pattern (see `AgentChat.svelte`'s provider dropdown for a similar selected-state pattern with conditional classes).

**Browse tab:** Show `browseItems` as cards using the existing `typeBadgeClass()` helper and `card` CSS class pattern already in this component. Each card shows: type badge, truncated content (first 120 chars), a right-arrow `›` button to open detail view. Show a total count: `{browseTotal} memories`. Add prev/next pagination buttons when `browseTotalPages > 1`.

**Detail/edit view:** When `selectedItem !== null`, render a slide-in panel (CSS `transform: translateX(...)` or simply replacing the list with a detail view). Show:
- Back arrow button to return to list
- Memory type badge and `created_at` date (use existing `formatDate()`)
- Editable `<textarea>` bound to `editContent`, initialized from `selectedItem.content`
- Cancel and Save buttons
- On Save: call `await updateMemory(selectedItem.id, editContent)`, then clear `selectedItem`, reload browse page.

**DO NOT** remove the existing Search tab functionality — preserve all existing `executeSearch`, `handleStore`, filter chips logic.

---

## 3. File Manifest

**Create:**
- `zen-sci-portal/src/lib/stores/agent.ts`

**Modify:**
- `zen-sci-portal/src/lib/components/AgentChat.svelte` — migrate to agentStore, fix memory counter
- `zen-sci-portal/src/lib/components/MemoryPanel.svelte` — add browse tab, pagination, detail/edit
- `zen-sci-portal/src/lib/api/tauri.ts` — add `saveChatMessage`, `loadChatHistory`, `listMemories`, `updateMemory`, `MemoryListResponse`
- `zen-sci-portal/src-tauri/src/db/mod.rs` — add `chat_messages` table to SCHEMA, add `save_chat_message` and `load_chat_history` methods
- `zen-sci-portal/src-tauri/src/commands/gateway.rs` — add `save_chat_message`, `load_chat_history`, `list_memories`, `update_memory` commands
- `zen-sci-portal/src-tauri/src/gateway/client.rs` — add `list_memories` and `update_memory` methods
- `zen-sci-portal/src-tauri/src/lib.rs` — register all 4 new Tauri commands in `invoke_handler![]`

**Do not modify:**
- `zen-sci-portal/src/lib/types/index.ts` — types stay stable in Phase 1 (new types go in `tauri.ts`)
- `zen-sci-portal/src/routes/memory/+page.svelte` — thin wrapper, no changes needed
- `zen-sci-portal/src-tauri/src/state.rs` — no new state needed
- Any files in `zen-sci/` or `docs/`

---

## 4. Success Criteria

- [ ] Navigate to `/workspace/[docId]`, open `AgentChat`, send a message, navigate away (to `/memory`), navigate back — chat history is intact in the store (messages still visible).
- [ ] Save an agent response to memory → the memory counter in `AgentChat` header increments immediately from `(N)` to `(N+1)` without any search query being issued.
- [ ] Open `/memory`, click "Browse" tab → paginated list of all stored memory items appears (requires gateway `GET /v1/memory` to be live; if not yet live, show graceful empty state "No memories to browse yet").
- [ ] Click a memory item in Browse → detail view opens with editable textarea; edit content, click Save → item updated, returns to list.
- [ ] Build compiles: `cargo build` in `zen-sci-portal/src-tauri/` passes with no errors.
- [ ] The `agentStore.init()` is idempotent: calling it twice (e.g., component remounts) does not create a second agent.

---

## 5. Constraints & Non-Goals

- **DO NOT** add document context, `documentId` props, or `priorContext` injection to `AgentChat.svelte` — that is Phase 2.
- **DO NOT** create a `/chat` full-screen route — that is Phase 3.
- **DO NOT** add the `PATCH_INTENT:` system or patch confirmation UI — that is Phase 2.
- **DO NOT** add provider key management UI — that is Phase 3.
- **DO NOT** modify any Go gateway source — the gateway endpoints `GET /v1/memory` and `PUT /v1/memory/:id` are expected to be added by the gateway team separately; the Rust client and Tauri commands are the portal's side of that contract.
- **DO NOT** modify `src/lib/types/index.ts` — keep it stable.
- **DO NOT** introduce any new npm or Cargo dependencies.
- **DO NOT** modify files outside the File Manifest.
