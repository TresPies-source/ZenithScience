# Zenflow Commission: Phase 2 — Intelligence: Document Context, Tool-Calling Loop & Patch-by-Intent

**Objective:** Wire the agent to the open document (via a lightweight section index in the system prompt + `get_document` tool call), implement the normalized tool-calling loop in the Go gateway, and add the `PATCH_INTENT:` write-back protocol with human confirmation in `AgentChat.svelte`.

**Prerequisite:** Phase 1 must be complete. `agentStore` exists, `AgentChat.svelte` reads from the store, and the memory counter fix is live.

---

## 1. Context & Grounding

**Primary Specification:**
- `docs/specs/zen-sci-portal-v1.1-spec.md` — Full v1.1 spec. Phase 2 covers §3.2 (Agent-Workspace Context), §3.4 (Tool-Calling Agent Loop). Implementation order: Q3 (Patch-by-Intent) first, then Q1 (Document Context), then Q2 (Tool Loop).

**Pattern Files (Follow these examples):**
- `zen-sci-portal/src/lib/api/tauri.ts` — Pattern for extending `chatWithAgent`. Currently: `invoke<AgentResponse>('chat_with_agent', { agentId, message })`. Must be extended to pass document context. Follow the camelCase parameter convention exactly.
- `zen-sci-portal/src-tauri/src/commands/gateway.rs` — Pattern for `chat_with_agent` Tauri command. Currently takes `agent_id: String, message: String`. Must be extended to accept optional document context fields.
- `zen-sci-portal/src-tauri/src/gateway/client.rs` — Pattern for `chat_with_agent` HTTP method. Currently sends `{"message": message}`. Must be extended to include document context in the request body.
- `zen-sci-portal/src-tauri/src/models/gateway.rs` — Pattern for response models. `AgentResponse` is the existing struct — it needs two new optional fields: `patch_intent` and `tool_calls` (the latter already exists as `Vec<ToolCall>` but is never populated by the gateway). The `ToolCall` struct also needs renaming/extending.
- `zen-sci-portal/src/lib/components/AgentChat.svelte` — The component being enhanced. Currently 392 lines. Read fully — it uses Svelte 5 runes, has `{#if pendingPatch}` block to add, `agentStore` from Phase 1 is now active.
- `zen-sci-portal/src/routes/workspace/[docId]/+page.svelte` — Shows how `activeDocument` store is read and how the document is structured. `AgentChat` is embedded here and must receive `documentId` and `documentTitle` as props.

**Files to Read Before Starting:**
- `zen-sci-portal/src/lib/types/index.ts` — `AgentResponse` interface (needs `patch_intent?: PatchIntent` added). `Section` interface (shows `id`, `title`, `content`, `owner_id`).
- `zen-sci-portal/src/lib/stores/agent.ts` (Phase 1 output) — The `AgentStoreState` — add `priorContext: MemoryItem[]` field and `setPriorContext()` method if not already there.
- `zen-sci-portal/src-tauri/src/models/gateway.rs` — `AgentResponse` Rust struct and `ToolCall` struct. Add `patch_intent: Option<serde_json::Value>` to `AgentResponse`.
- `docs/specs/zen-sci-portal-v1.1-spec.md` §3.2.5 — Exact `PATCH_INTENT:` format, `parsePatchIntent()` Go function, `PatchIntent` struct (Go). This is Go gateway code.
- `docs/specs/zen-sci-portal-v1.1-spec.md` §3.4 — Full `runAgentLoop()` Go function, `executeTool()`, `ToolCallResult`, normalized provider pattern.

---

## 2. Detailed Requirements

### Implementation Order (Follow This Exactly)

1. **Step A: Patch-by-Intent** — frontend + gateway changes for write-back
2. **Step B: Document Context Injection** — system prompt enrichment
3. **Step C: Tool-Calling Loop** — normalized gateway loop + provider implementations

---

### Step A: Patch-by-Intent (PATCH_INTENT: Write-Back Protocol)

#### A.1 Add `PatchIntent` type to `src/lib/types/index.ts`

```typescript
export interface PatchIntent {
  section_title: string;
  action: 'create' | 'update';
}
```

#### A.2 Extend `AgentResponse` in `src/lib/types/index.ts`

Add to the existing `AgentResponse` interface:
```typescript
patch_intent?: PatchIntent;
pipeline_proposal?: PipelineProposal;  // placeholder for Phase 3 — add as optional unknown for now
```

#### A.3 Extend Rust `AgentResponse` model in `src-tauri/src/models/gateway.rs`

Add to the existing `AgentResponse` struct:
```rust
pub patch_intent: Option<serde_json::Value>,
```

#### A.4 Add `pendingPatch` state and `applyPatch()` to `AgentChat.svelte`

Add new local state:
```typescript
let pendingPatch = $state<import('$lib/types/index.js').PatchIntent | null>(null);
```

After the `agentStore.addMessage(...)` call in `sendMessage()`, check:
```typescript
if (result.patch_intent) {
  pendingPatch = result.patch_intent;
}
```

Add `applyPatch()` function that:
1. Gets the last assistant message content from `$agentStore.messages`.
2. If `intent.action === 'create'`: calls `createSection(documentId, intent.section_title)`, then `updateSectionContent(section.id, documentId, content)`.
3. If `intent.action === 'update'`: finds existing section by title (case-insensitive), calls `updateSectionContent(section.id, documentId, content)`.
4. After applying: calls `saveToMemory(...)` with a brief summary string, sets `pendingPatch = null`, reloads document via `getDocument(documentId)` and updates `activeDocument` store.
5. Requires `documentId` prop (see A.5) — guard with `if (!documentId) return`.

Import `createSection`, `updateSectionContent`, `getDocument` from `'$lib/api/tauri.js'`. Import `activeDocument` from `'$lib/stores/activeDocument.js'`.

#### A.5 Add `documentId`, `documentTitle`, `doc` props to `AgentChat.svelte`

Change the `Props` interface:
```typescript
interface Props {
  projectId: string;
  documentId?: string;
  documentTitle?: string;
  doc?: import('$lib/types/index.js').Document;
  fullScreen?: boolean;  // Phase 3 prep — default false, no behavior change yet
}

let { projectId, documentId, documentTitle, doc, fullScreen = false }: Props = $props();
```

`AgentChat` is currently used in `workspace/[docId]/+page.svelte` — update that page to pass `documentId={docId}`, `documentTitle={doc?.title}`, `doc={doc}` (using the existing `doc` derived value on that page).

#### A.6 Add patch confirmation UI to `AgentChat.svelte`

Below the messages list (`{#each messages...}` block), before the input area, add:

```svelte
{#if pendingPatch}
  <div class="mx-4 mb-2 p-3 rounded-lg bg-zinc-800 border border-amber-600/40">
    <p class="text-xs text-amber-400 font-medium mb-1">Proposed edit</p>
    <p class="text-sm text-zinc-300">
      {pendingPatch.action === 'create' ? 'Create' : 'Update'} section:
      <span class="font-mono text-amber-300">"{pendingPatch.section_title}"</span>
    </p>
    <p class="text-xs text-zinc-500 mt-1">
      The response above will be written to this section.
    </p>
    <div class="flex gap-2 mt-3">
      <button
        onclick={() => applyPatch(pendingPatch!)}
        class="px-3 py-1 text-xs rounded bg-amber-700 hover:bg-amber-600 text-white transition-colors"
      >
        Apply
      </button>
      <button
        onclick={() => pendingPatch = null}
        class="px-3 py-1 text-xs rounded bg-zinc-700 text-zinc-300 hover:bg-zinc-600 transition-colors"
      >
        Discard
      </button>
    </div>
  </div>
{/if}
```

Note: Use `bg-zinc-800`, `border-amber-600/40`, `text-amber-400`, `text-amber-300` — these match the spec and dark theme. The existing `AgentChat` uses Tailwind classes; add the dark equivalents.

---

### Step B: Document Context Injection

#### B.1 Extend `chatWithAgent` in `src/lib/api/tauri.ts`

```typescript
export interface AgentChatContext {
  documentId?: string;
  documentTitle?: string;
  documentSectionIndex?: string[];   // section titles only — not content
  documentMode?: 'draft' | 'structured';
  priorContext?: import('$lib/types/index.js').MemoryItem[];
}

export const chatWithAgent = (
  agentId: string,
  message: string,
  context?: AgentChatContext
): Promise<AgentResponse> =>
  invoke<AgentResponse>('chat_with_agent', { agentId, message, context: context ?? null });
```

Note: The existing `chatWithAgent` signature `(agentId: string, message: string)` changes to include optional `context`. This is backward compatible since `context` is optional.

#### B.2 Extend Rust `chat_with_agent` command in `src-tauri/src/commands/gateway.rs`

```rust
#[derive(Debug, serde::Deserialize)]
pub struct AgentChatContext {
    pub document_id: Option<String>,
    pub document_title: Option<String>,
    pub document_section_index: Option<Vec<String>>,
    pub document_mode: Option<String>,
    pub prior_context: Option<Vec<serde_json::Value>>,
}

#[tauri::command]
pub async fn chat_with_agent(
    state: State<'_, AppState>,
    agent_id: String,
    message: String,
    context: Option<AgentChatContext>,
) -> Result<AgentResponse, String> {
    let token = get_token(&state).await?;
    state
        .gateway_client
        .chat_with_agent(&token, &agent_id, &message, context.as_ref())
        .await
        .map_err(|e| {
            tracing::error!("chat_with_agent failed: {}", e);
            e.to_string()
        })
}
```

#### B.3 Extend `GatewayClient::chat_with_agent` in `src-tauri/src/gateway/client.rs`

Update the existing `chat_with_agent` method to accept and forward context:

```rust
pub async fn chat_with_agent(
    &self,
    token: &str,
    agent_id: &str,
    message: &str,
    context: Option<&crate::commands::gateway::AgentChatContext>,
) -> Result<AgentResponse, AppError> {
    let url = format!("{}/v1/gateway/agents/{}/chat", self.base_url, agent_id);
    let mut body = serde_json::json!({ "message": message });

    if let Some(ctx) = context {
        if let Some(doc_id) = &ctx.document_id {
            body["document_id"] = serde_json::Value::String(doc_id.clone());
        }
        if let Some(title) = &ctx.document_title {
            body["document_title"] = serde_json::Value::String(title.clone());
        }
        if let Some(sections) = &ctx.document_section_index {
            body["document_section_index"] = serde_json::to_value(sections).unwrap_or_default();
        }
        if let Some(mode) = &ctx.document_mode {
            body["document_mode"] = serde_json::Value::String(mode.clone());
        }
        if let Some(prior) = &ctx.prior_context {
            body["prior_context"] = serde_json::to_value(prior).unwrap_or_default();
        }
    }

    let val = self.post_json(token, &url, body).await?;
    serde_json::from_value(val).map_err(|e| AppError::SerializationError(e.to_string()))
}
```

#### B.4 Update `sendMessage()` in `AgentChat.svelte` to pass context

In `sendMessage()`, update the `chatWithAgent` call:

```typescript
const result = await chatWithAgent(agent.id, text, {
  documentId,
  documentTitle,
  documentSectionIndex: doc?.sections.map(s => s.title) ?? [],
  documentMode: doc?.mode,
  priorContext: $agentStore.priorContext,
});
```

#### B.5 Add memory-as-history session init to `AgentChat.svelte`

In `onMount`, after `agentStore.init(...)` completes and `documentId` is set:

```typescript
if (documentId) {
  agentStore.setActiveDocument(documentId);
  try {
    const priorMemory = await searchMemory(`document:${documentId}`, projectId);
    agentStore.setPriorContext(priorMemory as unknown as MemoryItem[]);
  } catch {
    // Non-critical — empty prior context is fine
  }
}
```

Note: `searchMemory` returns `MemorySearchItem[]` not `MemoryItem[]` — cast as needed; the shape is compatible for system prompt injection purposes.

#### B.6 Tag `saveToMemory()` calls with `document_id` metadata (Go gateway side)

The `storeMemory` Tauri command currently sends `{content, type, context_type}` to the gateway. To add `document_id` metadata, the existing `store_memory` gateway command needs to pass metadata.

**Extend `store_memory` Tauri command** in `src-tauri/src/commands/gateway.rs`:
```rust
#[tauri::command]
pub async fn store_memory(
    state: State<'_, AppState>,
    content: String,
    memory_type: String,
    context_type: String,
    metadata: Option<serde_json::Value>,  // NEW — optional metadata map
) -> Result<MemoryItem, String> {
    let token = get_token(&state).await?;
    state
        .gateway_client
        .store_memory(&token, &content, &memory_type, &context_type, metadata)
        .await
        .map_err(|e| {
            tracing::error!("store_memory failed: {}", e);
            e.to_string()
        })
}
```

**Extend `GatewayClient::store_memory`** in `src-tauri/src/gateway/client.rs`:
```rust
pub async fn store_memory(
    &self,
    token: &str,
    content: &str,
    memory_type: &str,
    context_type: &str,
    metadata: Option<serde_json::Value>,
) -> Result<MemoryItem, AppError> {
    let url = format!("{}/v1/memory", self.base_url);
    let mut body = serde_json::json!({
        "content": content,
        "type": memory_type,
        "context_type": context_type
    });
    if let Some(meta) = metadata {
        body["metadata"] = meta;
    }
    let val = self.post_json(token, &url, body).await?;
    serde_json::from_value(val).map_err(|e| AppError::SerializationError(e.to_string()))
}
```

**Update `storeMemory` in `src/lib/api/tauri.ts`**:
```typescript
export const storeMemory = (
  content: string,
  memoryType: string = 'note',
  contextType: string = 'private',
  metadata?: Record<string, unknown>
): Promise<MemoryItem> =>
  invoke<MemoryItem>('store_memory', {
    content,
    memoryType,
    contextType,
    metadata: metadata ?? null,
  });
```

**Update `saveToMemory()` in `AgentChat.svelte`** to pass `document_id` in metadata:
```typescript
async function saveToMemory(content: string) {
  savingMemory = true;
  try {
    const stored = await storeMemory(content, 'conversation', 'private',
      documentId ? { document_id: documentId, document_title: documentTitle } : undefined
    );
    memoryItems = [
      { id: stored.id, memory_type: stored.memory_type, content: stored.content, relevance_score: undefined },
      ...memoryItems,
    ].slice(0, 20);
  } catch {
    // Non-critical
  } finally {
    savingMemory = false;
  }
}
```

---

### Step C: Tool-Calling Loop (Go Gateway)

**Note:** The tool-calling loop lives entirely in the Go AgenticGateway binary, not in the Svelte/Rust portal. The portal changes for Step C are limited to displaying normalized `tool_calls` in the response, which the frontend already partially handles.

#### C.1 Go Gateway: `runAgentLoop()` in `server/handle_gateway.go`

Implement the `runAgentLoop()` function exactly as specified in `docs/specs/zen-sci-portal-v1.1-spec.md` §3.4.1. Key requirements:

1. Build `providerTools` from `tools.ListTools()` — all registered tools including `get_document`.
2. Initialize `messages` with system prompt and user message.
3. Loop up to `maxToolIterations = 5`:
   - Call `llmProvider.GenerateCompletion(ctx, req)` with `Tools: providerTools`.
   - If `resp.ToolCalls` is empty → return `resp.Content` as final answer.
   - For each tool call: call `s.executeTool(ctx, tc.Name, tc.Arguments)`.
   - Append assistant message + tool result message to `messages`.
4. Return the final response with all accumulated `allToolCalls`.

#### C.2 Go Gateway: Register `get_document` tool in `tools/registry.go`

Register `get_document` as a gateway-native tool (not an MCP tool) at gateway startup. See §3.2.1 for the full `RegisterGetDocumentTool()` implementation.

The `get_document` function implementation for v1.1 local-first: receive the document structure from the chat request body's `document_section_index` field (already sent by the portal in Step B). The tool returns a JSON object with `{document_id, title, mode, sections: [{title}]}` — structural index only, as specified.

#### C.3 Go Gateway: `buildSystemPrompt()` update in `server/handle_gateway.go`

Update `buildSystemPrompt()` to accept a `DocumentContext` and `providerNativeTools bool` parameter. See §3.2.4 and §3.4.3 for the full implementation. Key points:
- Inject document ID, title, mode, and section titles (not content) into system prompt.
- Inject prior context items from `PriorContext` field.
- Only inject `TOOL_CALL:` text-mode tool instructions when `!providerNativeTools`.

#### C.4 Go Gateway: Provider implementations

**Anthropic provider** (`server/services/providers/anthropic.go`): Map `Tools[]` → `tools` array in API request. Parse `tool_use` content blocks → normalized `ToolCalls[]`. See §3.4.2.

**OpenAI provider** (`server/services/providers/openai.go`): Map `Tools[]` → `tools` array in OpenAI function-calling format. Parse `tool_calls` in response → normalized `ToolCalls[]`.

**Ollama provider** (`server/services/providers/ollama.go`): Add `modelSupportsTools(model string) bool` probe using `toolCapablePatterns`. Use Ollama's native `tools` field on capable models. Add `generateWithTextToolFallback()` for non-capable models. Both paths return normalized `ToolCalls[]`. See §3.4.2 for full implementation.

#### C.5 Go Gateway: `parsePatchIntent()` and `stripPatchIntent()`

In `server/handle_gateway.go`, implement `parsePatchIntent()` and `stripPatchIntent()` exactly as specified in §3.2.5. Add `patch_intent` to the final `c.JSON(http.StatusOK, ...)` response alongside `response`, `tool_calls`, and `disposition`.

#### C.6 Go Gateway: `GET /v1/gateway/documents/:id` endpoint

Add `handleGetDocument` to `server/handle_gateway.go` and register the route. See §3.2.1 for implementation. For v1.1 local-first, the `documentProxy.GetDocument` can return the document structure from the `DocumentContext` that was passed in the chat request body — no round-trip needed.

#### C.7 Feature flags in gateway config

Add to `gateway-config.yaml` (or equivalent config file):
```yaml
features:
  tool_calling: true
  get_document_tool: true
  document_context: true
  patch_intent: true
  memory_as_history: true
```

All features default to `true` for v1.1. Setting `tool_calling: false` reverts to direct LLM call (v1.0 behavior).

#### C.8 Portal: Display normalized `tool_calls` (Rust model update)

The existing `AgentResponse` Rust struct already has `pub tool_calls: Vec<ToolCall>`. The `ToolCall` struct has `tool`, `args`, `result`. The Go gateway now returns richer tool call results — update the Rust `ToolCall` model to match:

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ToolCall {
    pub tool: String,
    pub args: serde_json::Value,
    pub result: Option<serde_json::Value>,
    pub error: Option<String>,   // NEW
}
```

The frontend `AgentChat.svelte` already renders `{tc.tool}` and `{#if tc.result}` — this requires no template change.

---

## 3. File Manifest

**Modify (Portal — TypeScript/Svelte):**
- `zen-sci-portal/src/lib/types/index.ts` — add `PatchIntent`, extend `AgentResponse`
- `zen-sci-portal/src/lib/api/tauri.ts` — extend `chatWithAgent`, extend `storeMemory`, add `AgentChatContext`
- `zen-sci-portal/src/lib/components/AgentChat.svelte` — add props, `pendingPatch` state, `applyPatch()`, patch UI, session init with prior context, document context in `sendMessage()`
- `zen-sci-portal/src/routes/workspace/[docId]/+page.svelte` — pass `documentId`, `documentTitle`, `doc` props to `AgentChat`

**Modify (Portal — Rust):**
- `zen-sci-portal/src-tauri/src/models/gateway.rs` — extend `AgentResponse` with `patch_intent`, extend `ToolCall` with `error`
- `zen-sci-portal/src-tauri/src/commands/gateway.rs` — extend `chat_with_agent` command (add `AgentChatContext` struct, `context` param), extend `store_memory` command (add `metadata` param)
- `zen-sci-portal/src-tauri/src/gateway/client.rs` — extend `chat_with_agent` method, extend `store_memory` method

**Modify (Go AgenticGateway):**
- `server/handle_gateway.go` — `runAgentLoop()`, `executeTool()`, `buildSystemPrompt()`, `parsePatchIntent()`, `stripPatchIntent()`, `handleGetDocument()`
- `server/services/providers/anthropic.go` — normalize `Tools[]` → Anthropic tools, parse `tool_use` → `ToolCalls[]`
- `server/services/providers/openai.go` — normalize `Tools[]` → OpenAI tools, parse `tool_calls` → `ToolCalls[]`
- `server/services/providers/ollama.go` — `modelSupportsTools()`, `generateWithTextToolFallback()`, native tools path
- `tools/registry.go` — `RegisterGetDocumentTool()`
- `gateway-config.yaml` — feature flags

**Do not modify:**
- `zen-sci-portal/src/lib/stores/agent.ts` — Phase 1 output; only add `setPriorContext()` if not already present
- `zen-sci-portal/src-tauri/src/db/mod.rs` — Phase 1 already complete
- `zen-sci-portal/src/routes/memory/+page.svelte`
- Any `zen-sci/` MCP server files

---

## 4. Success Criteria

- [ ] Open a document, type "summarize this document" in AgentChat → gateway logs show `get_document` tool being called → agent response references actual section titles from the document.
- [ ] Type "draft an Introduction section" → response contains prose followed by `PATCH_INTENT: {"section_title": "Introduction", "action": "create"}` (stripped from response text) → `patch_intent` field present in response → Apply button appears in chat UI.
- [ ] Click Apply → new "Introduction" section created in document (visible after page reloads) → memory counter increments (accepted edit saved to memory with `document_id` tag).
- [ ] Type "convert this to PDF" with Anthropic provider → gateway logs show `zen_sci_latex:convert_to_pdf` tool being called → tool result appears in chat message's tool calls area.
- [ ] Restart app, open same document, new chat → system prompt in gateway logs includes prior context items tagged with that `document_id`.
- [ ] With Ollama + `qwen3:8b`: tool calling uses native Ollama tools field (verify in gateway logs: `"using_native_tools": true`).
- [ ] With Ollama + a non-qwen model: `generateWithTextToolFallback()` is used (gateway log shows `"tool_fallback": true`).
- [ ] Set `tool_calling: false` in gateway config, restart gateway → chat works as before (direct LLM call, no tool loop).
- [ ] Build compiles: `cargo build` in `zen-sci-portal/src-tauri/` passes with no new errors.

---

## 5. Constraints & Non-Goals

- **DO NOT** implement streaming/SSE responses — that is v1.2.
- **DO NOT** add the `/chat` full-screen route — that is Phase 3.
- **DO NOT** add provider key management UI — that is Phase 3.
- **DO NOT** implement `search_memory` as a registered tool — that is v1.2.
- **DO NOT** add any new npm or Cargo dependencies.
- **DO NOT** silently write to the document without `PATCH_INTENT:` footer and user confirmation via the Apply button. Human confirmation is mandatory.
- **DO NOT** modify any `zen-sci/` MCP server source files.
- **DO NOT** modify `zen-sci-portal/src-tauri/src/lib.rs` unless a new Tauri command is added (only `store_memory` signature change, not a new command).
- The `AgentChatContext` struct lives in `commands/gateway.rs` for now — do not create a separate models file for it.
