# ZenithScience Portal v1.1: The Thinking Workspace

**Author:** Cruz Morales
**Status:** Draft
**Created:** 2026-02-20
**Grounded In:** zen-sci-portal-v1.0-spec.md, ARCHITECTURE.md, CONTEXT.md, STATUS.md, live E2E session (2026-02-19/20)

---

## 1. Vision

> Transform ZenithScience from a document editor with a chat sidebar into a unified thinking workspace where the agent knows what you're writing, can reach your tools, and remembers everything.

**The Core Insight:**

v1.0 shipped all the right pieces — agent, memory, tools, DAG pipelines, document editor — but they don't talk to each other. The agent doesn't know what document is open. Memory saves work but the counter never updates. Tools are visible but unreachable from chat. The pipeline page is a functional prototype with no integration path. The result is a capable system that feels disconnected.

v1.1 is about connecting the wires. Every change in this release is an integration, not a net-new feature. The agent gets document context automatically. Memory becomes a live counter. The chat handler gets a tool-calling loop. The pipeline page graduates from test harness to first-class workflow. Providers get real key management. The workspace becomes the place where the agent thinks alongside you.

**What Makes This Different:**

ZenSci's philosophy is *AI as thinking partner, not AI as button.* That means the agent shouldn't wait to be handed content — it should already know what document is open, what section you're in, what tools are available, and what you've saved to memory. v1.1 implements this philosophy at the integration layer: no new MCP servers, no new UI paradigms, just wiring the existing stack together so the agent can actually do its job.

The other key principle is *local-first*. Provider key management in v1.1 uses the platform keychain (already available via Tauri's secure storage), not a cloud vault. Chat history persists to SQLite (already present). Nothing new leaves the machine.

---

## 2. Goals & Success Criteria

**Primary Goals:**

1. **Agent-Workspace Integration** — The agent knows what document is open, can reference sections, and can write output back into the document
2. **Live Memory** — Memory counter reflects actual saved items; panel supports pagination, detail view, and editing
3. **Tool-Calling Agent** — Agent can invoke MCP tools (`zen_sci_latex:convert_to_pdf`, etc.) in response to user requests, via a tool loop in the gateway handler
4. **Persistent Chat** — Message history survives navigation; full-screen `/chat` route available
5. **Provider Key Management** — Anthropic, OpenAI, and other providers configurable via gateway settings UI or `.env`
6. **Pipelines First-Class** — Pipeline page integrated with agent chat; agent can propose and submit DAGs; pipeline results flow back to workspace

**Success Criteria:**

- ✅ Open a document, ask "summarize this document" → agent calls `get_document`, returns a summary referencing actual section content
- ✅ Ask agent to "draft an Introduction" → response + PATCH_INTENT footer → Apply button appears → click Apply → section created in document
- ✅ Ask "convert this to PDF" → agent invokes `zen_sci_latex:convert_to_pdf` via normalized tool loop, result visible in workspace
- ✅ Save an agent response → memory counter increments immediately (no search required); item tagged with `document_id`
- ✅ Restart app, open same document, start new chat → agent's system prompt includes top-5 prior memory items for that document
- ✅ Navigate away from workspace and back → chat history is intact (Svelte agent store)
- ✅ `/chat` route loads as full-screen chat with sidebar showing current document
- ✅ Add an Anthropic API key in Settings → gateway registers the provider, agent can use Claude with native tool calling
- ✅ Submit a pipeline from chat ("run Write+PDF pipeline") → DAG submitted, status visible in pipeline page
- ✅ Memory panel shows paginated list; click item → detail view with edit capability

**Non-Goals (Out of Scope for v1.1):**

- ❌ Real-time collaborative editing (multi-user)
- ❌ New MCP servers (beyond the existing 6)
- ❌ Streaming/SSE chat responses (deferred to v1.2)
- ❌ Agent-to-agent orchestration
- ❌ Mobile or web portal
- ❌ OAuth provider login (e.g., GitHub Copilot as provider)

---

## 3. Technical Architecture

### 3.1 System Overview

```
zen-sci-portal (SvelteKit/Tauri)
│
├── /chat          ← NEW: full-screen persistent chat route
├── /workspace/[docId]
│   ├── AgentChat  ← ENHANCED: memory-as-history, PATCH_INTENT write-back
│   ├── DocConverter
│   └── DocumentEditor ← ENHANCED: agent proposes section edits (human confirms)
├── /memory        ← ENHANCED: paginated list, detail view, edit
├── /pipelines     ← ENHANCED: integrated with chat + agent
└── /settings      ← ENHANCED: provider key management
│
Tauri IPC (Rust)
│
AgenticGateway (Go :8080)
│
├── POST /v1/gateway/agents/:id/chat  ← ENHANCED: normalized tool loop + patch_intent
├── GET  /v1/gateway/documents/:id    ← NEW: get_document tool fulfillment endpoint
├── POST /v1/memory/search            ← existing (used for session init context)
├── POST /v1/memory                   ← existing (now tags document_id in metadata)
├── GET  /v1/memory                   ← NEW: list endpoint for pagination
├── PUT  /v1/memory/:id               ← NEW: edit endpoint
├── POST /v1/settings/providers       ← NEW: provider key registration
└── GET  /v1/settings/providers       ← NEW: list configured providers
│
zen-sci MCP servers (Node.js, 17 tools)     ← reached via normalized ToolCalls[]
+ get_document (gateway-native tool)         ← registered at gateway startup
```

### 3.2 Feature 1: Agent-Workspace Context

**Problem:** `AgentChat.svelte` creates an agent scoped to `projectId` but never passes document context. The gateway's `handleGatewayAgentChat` sends only the user's raw message to Ollama with a generic system prompt. The agent is blind to what the user is writing.

**Solution (grounded in the strategic scout — Q1 Route B + Cruz's clarification on "fully readable"):**

Two integrated layers:
1. **Memory-as-History**: Agents stay ephemeral (gateway sidecar restarts on every app launch — persisting agent IDs is futile). Continuity comes from the gateway memory system, which is already durable. On session start, search memory tagged with `document_id` and inject top-5 relevant items into the system prompt.
2. **`get_document` as a first-class tool**: The agent does not receive a text dump of the document in the system prompt. Instead, `get_document(document_id)` is registered as a tool in the tool registry. When the agent needs to read a section, check ownership, understand structure, or see the current draft mode, it calls `get_document` during its reasoning loop. This is the correct design — the document is a resource the agent queries, not data the host pre-injects.

#### 3.2.1 `get_document` Gateway Tool

A new internal tool is registered at gateway startup alongside the MCP tools. It is not backed by an MCP server — it is a first-class gateway-native tool that queries the Tauri document store via a new endpoint.

**Gateway tool registration (Go) — `tools/registry.go`:**

```go
// Register get_document as a gateway-native tool (not an MCP tool)
func RegisterGetDocumentTool(docFetcher DocumentFetcher) {
    Register(Tool{
        Name:        "get_document",
        Description: "Fetch the full structure of the currently open document, including title, mode (Draft/Structured), and all sections with their title, content, owner, and lock status. Call this whenever you need to read the document before answering or proposing edits.",
        Parameters: map[string]interface{}{
            "document_id": map[string]interface{}{
                "type":        "string",
                "description": "The document ID to fetch (provided in your system prompt)",
            },
        },
        Function: func(ctx context.Context, args map[string]interface{}) (interface{}, error) {
            docID, _ := args["document_id"].(string)
            return docFetcher.GetDocument(ctx, docID)
        },
    })
}
```

**New gateway endpoint — `GET /v1/gateway/documents/:id`:**

```go
// handle_gateway.go
func (s *Server) handleGetDocument(c *gin.Context) {
    docID := c.Param("id")
    // Fetch from Tauri-layer cache or forward to document store
    // The portal proxies document reads through the gateway for tool use
    doc, err := s.documentProxy.GetDocument(c.Request.Context(), docID)
    if err != nil {
        s.errorResponse(c, http.StatusNotFound, "document_not_found", err.Error())
        return
    }
    c.JSON(http.StatusOK, doc)
}
```

The `documentProxy` is a thin HTTP client that calls back to the Tauri portal's local REST-style bridge. In practice for local-first v1.1, the portal can expose a local document endpoint on a second port that the gateway can call, **or** — simpler — the `get_document` tool can be fulfilled entirely within the gateway by receiving the document structure at agent init time via the existing chat request body (see §3.2.3).

#### 3.2.2 Memory-as-History: Session Init

**`AgentChat.svelte` — on session init (simplified):**

```typescript
import { agentStore } from '$lib/stores/agent';
import { searchMemory, storeMemory } from '$lib/api/tauri';

// On mount, after agent is created:
onMount(async () => {
  await agentStore.init(projectId, activeProvider);

  if (documentId) {
    agentStore.setActiveDocument(documentId);

    // Retrieve prior memory tagged with this document
    const priorMemory = await searchMemory(`document:${documentId}`, 5);
    agentStore.setPriorContext(priorMemory.items);
  }
});
```

**`src/lib/stores/agent.ts` — `setPriorContext`:**

```typescript
setPriorContext: (items: MemoryItem[]) => {
  update(s => ({ ...s, priorContext: items }));
},
```

**Memory tagging — `saveToMemory()` in `AgentChat.svelte`:**

```typescript
async function saveToMemory(content: string) {
  const stored = await storeMemory(content, 'conversation', 'private', {
    document_id: documentId,   // TAG — enables future retrieval per document
    document_title: documentTitle,
  });
  memoryItems = [{ id: stored.id, ...stored }, ...memoryItems].slice(0, 20);
}
```

The `metadata` field on `store_memory` already exists in the gateway API — this is additive, not a schema change.

#### 3.2.3 Chat Request with Document Bootstrap

The system prompt does NOT contain the full document content. It contains only the document ID, title, mode, and section titles (a structural index). When the agent needs section content, it calls `get_document`.

**`AgentChat.svelte` — Modified `sendMessage()`:**

```typescript
async function sendMessage() {
  if (!userMessage.trim() || !agent || sending) return;
  sending = true;
  const text = userMessage.trim();
  userMessage = '';

  agentStore.addMessage({ role: 'user', content: text, timestamp: new Date() });

  try {
    const result = await chatWithAgent(agent.id, text, {
      documentId,
      documentTitle,
      documentSectionIndex: doc?.sections.map(s => s.title) ?? [],
      documentMode: doc?.mode,
      priorContext: $agentStore.priorContext,
    });

    agentStore.addMessage({
      role: 'assistant',
      content: result.response ?? '',
      timestamp: new Date(),
      toolCalls: result.tool_calls?.length ? result.tool_calls : undefined,
      patchIntent: result.patch_intent,
    });

    if (result.patch_intent) {
      // Show diff preview — see §3.2.5
      pendingPatch = result.patch_intent;
    }
  } catch (e) {
    // existing error handling
  } finally {
    sending = false;
  }
}
```

**`tauri.ts` — Updated `chatWithAgent` signature:**

```typescript
export interface AgentChatContext {
  documentId?: string;
  documentTitle?: string;
  documentSectionIndex?: string[];  // section titles only — no content
  documentMode?: 'Draft' | 'Structured';
  priorContext?: MemoryItem[];       // top-N memory items from session init search
}

export const chatWithAgent = (
  agentId: string,
  message: string,
  context?: AgentChatContext
): Promise<AgentResponse> =>
  invoke<AgentResponse>('chat_with_agent', { agentId, message, context });
```

#### 3.2.4 Gateway Handler: Minimal System Prompt + Tool Registration

**`server/handle_gateway.go` — `buildSystemPrompt()`:**

```go
func (s *Server) buildSystemPrompt(agent *gateway.AgentConfig, ctx *DocumentContext) string {
    base := fmt.Sprintf(
        "You are a scientific writing assistant embedded in ZenithScience. "+
            "Pacing: %s. Depth: %s.\n\n",
        agent.Pacing, agent.Depth,
    )

    if ctx != nil && ctx.DocumentID != "" {
        base += fmt.Sprintf(
            "## Active Document\n"+
                "- ID: %s\n"+
                "- Title: %s\n"+
                "- Mode: %s\n"+
                "- Sections: %s\n\n"+
                "Call get_document(\"%s\") to read section content before answering questions "+
                "about this document or proposing edits.\n\n",
            ctx.DocumentID,
            ctx.DocumentTitle,
            ctx.DocumentMode,
            strings.Join(ctx.SectionTitles, ", "),
            ctx.DocumentID,
        )
    }

    if len(ctx.PriorContext) > 0 {
        base += "## Prior Context (from memory)\n"
        for _, mem := range ctx.PriorContext {
            base += fmt.Sprintf("- %s\n", mem.Content)
        }
        base += "\n"
    }

    base += "## Available Tools\n"
    base += "Call tools by name when you need them. " +
        "Always call get_document before summarizing or editing the document.\n"

    return base
}
```

The document structure (ID + section titles + mode) is injected as a lightweight index. The agent calls `get_document` to get section content on demand. Prior memory items from the session init search are injected as a "Prior Context" section.

#### 3.2.5 Document Write-Back: Patch-by-Intent with Human Confirmation

**Scout Q3 decision:** The agent proposes edits via a `PATCH_INTENT:` footer on its response. The human always confirms. No silent writes.

**Agent instruction (appended to system prompt):**

```
## Document Write-Back
When you want to create or update a document section, end your response with:

PATCH_INTENT: {"section_title": "<title>", "action": "create"|"update"}

The user will see a preview and choose whether to apply it. The content to write is
the prose in your response immediately above the PATCH_INTENT line.
Never write to the document without a PATCH_INTENT footer.
```

**Gateway response — adds `patch_intent` field:**

```go
// After runAgentLoop returns responseText:
patchIntent := parsePatchIntent(responseText)

c.JSON(http.StatusOK, gin.H{
    "agent_id":     agentID,
    "response":     stripPatchIntent(responseText),  // clean prose, no footer
    "tool_calls":   toolCalls,
    "patch_intent": patchIntent,                      // nil if not present
    "disposition": map[string]interface{}{
        "pacing": agentConfig.Pacing,
        "depth":  agentConfig.Depth,
    },
})
```

**`parsePatchIntent()` in Go:**

```go
func parsePatchIntent(content string) *PatchIntent {
    marker := "PATCH_INTENT:"
    idx := strings.LastIndex(content, marker)
    if idx == -1 {
        return nil
    }
    jsonStr := strings.TrimSpace(content[idx+len(marker):])
    // Take only first line (the JSON object)
    if nl := strings.IndexByte(jsonStr, '\n'); nl != -1 {
        jsonStr = jsonStr[:nl]
    }
    var pi PatchIntent
    if err := json.Unmarshal([]byte(jsonStr), &pi); err != nil {
        return nil
    }
    return &pi
}

type PatchIntent struct {
    SectionTitle string `json:"section_title"`
    Action       string `json:"action"` // "create" | "update"
}
```

**`AgentChat.svelte` — Patch confirmation UI:**

```svelte
{#if pendingPatch}
  <div class="mt-3 p-3 rounded-lg bg-zinc-800 border border-amber-600/40">
    <p class="text-xs text-amber-400 font-medium mb-1">Proposed edit</p>
    <p class="text-sm text-zinc-300">
      {pendingPatch.action === 'create' ? 'Create' : 'Update'} section:
      <span class="font-mono text-amber-300">"{pendingPatch.sectionTitle}"</span>
    </p>
    <p class="text-xs text-zinc-500 mt-1">
      Content: the response above will be written to this section.
    </p>
    <div class="flex gap-2 mt-3">
      <button onclick={() => applyPatch(pendingPatch!)}
        class="px-3 py-1 text-xs rounded bg-amber-700 hover:bg-amber-600 text-white">
        Apply
      </button>
      <button onclick={() => pendingPatch = null}
        class="px-3 py-1 text-xs rounded bg-zinc-700 text-zinc-300">
        Discard
      </button>
    </div>
  </div>
{/if}
```

**`applyPatch()` in `AgentChat.svelte`:**

```typescript
async function applyPatch(intent: PatchIntent) {
  if (!doc || !documentId) return;
  const lastAssistantMsg = [...messages].reverse().find(m => m.role === 'assistant');
  const content = lastAssistantMsg?.content ?? '';

  if (intent.action === 'create') {
    const section = await createSection(documentId, intent.sectionTitle);
    await updateSectionContent(section.id, content);
  } else {
    const section = doc.sections.find(s =>
      s.title.toLowerCase() === intent.sectionTitle.toLowerCase()
    );
    if (section) {
      await updateSectionContent(section.id, content);
    }
  }

  // Auto-save this accepted edit to memory with document tag
  await saveToMemory(
    `Agent proposed and user accepted: ${intent.action} section "${intent.sectionTitle}" (${content.length} chars)`
  );

  pendingPatch = null;
  doc = await getDocument(documentId);  // reload
}
```

The `updateSectionContent` Tauri command is already live. This is a frontend-only change — no new gateway endpoints or Rust commands needed for write-back.

---

### 3.3 Feature 2: Persistent Chat & Full-Screen Route

**Problem:** Agent state (ID + messages) lives only in `AgentChat.svelte` local `$state`. Navigating away destroys it. No full-screen chat route exists.

**Solution:** Lift agent state to a Svelte store; add `/chat` route.

#### 3.3.1 Agent Store

**`src/lib/stores/agent.ts` — NEW:**

```typescript
import { writable, get } from 'svelte/store';
import type { Agent, ChatMessage } from '$lib/types';
import { createAgent, chatWithAgent } from '$lib/api/tauri';

interface AgentStore {
  agent: Agent | null;
  messages: ChatMessage[];
  activeDocumentId: string | null;
  loading: boolean;
  error: string | null;
}

function createAgentStore() {
  const { subscribe, set, update } = writable<AgentStore>({
    agent: null,
    messages: [],
    activeDocumentId: null,
    loading: false,
    error: null,
  });

  return {
    subscribe,
    init: async (projectId: string, provider: string) => {
      const state = get({ subscribe });
      if (state.agent) return; // already initialized
      update(s => ({ ...s, loading: true }));
      try {
        const agent = await createAgent(projectId, 'Research Agent', provider);
        update(s => ({ ...s, agent, loading: false }));
      } catch (e) {
        update(s => ({ ...s, error: String(e), loading: false }));
      }
    },
    setActiveDocument: (documentId: string | null) => {
      update(s => ({ ...s, activeDocumentId: documentId }));
    },
    addMessage: (message: ChatMessage) => {
      update(s => ({ ...s, messages: [...s.messages, message] }));
    },
    clearMessages: () => {
      update(s => ({ ...s, messages: [] }));
    },
    reset: () => {
      set({ agent: null, messages: [], activeDocumentId: null, loading: false, error: null });
    },
  };
}

export const agentStore = createAgentStore();
```

#### 3.3.2 Chat Persistence to SQLite

Message history persists across sessions via a new Tauri command backed by SQLite.

**New Tauri commands:**

```rust
// commands/agent.rs
#[tauri::command]
pub async fn load_chat_history(
    state: tauri::State<'_, AppState>,
    agent_id: String,
) -> Result<Vec<ChatMessage>, String> {
    state.document_store
        .load_chat_history(&agent_id)
        .await
        .map_err(|e| e.to_string())
}

#[tauri::command]
pub async fn save_chat_message(
    state: tauri::State<'_, AppState>,
    agent_id: String,
    role: String,
    content: String,
) -> Result<(), String> {
    state.document_store
        .save_chat_message(&agent_id, &role, &content)
        .await
        .map_err(|e| e.to_string())
}
```

**SQLite schema addition:**

```sql
CREATE TABLE IF NOT EXISTS chat_messages (
    id TEXT PRIMARY KEY DEFAULT (lower(hex(randomblob(16)))),
    agent_id TEXT NOT NULL,
    role TEXT NOT NULL CHECK (role IN ('user', 'assistant', 'system')),
    content TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_chat_messages_agent_id ON chat_messages(agent_id);
```

#### 3.3.3 Full-Screen Chat Route

**`src/routes/chat/+page.svelte` — NEW:**

```svelte
<script lang="ts">
  import AgentChat from '$lib/components/AgentChat.svelte';
  import { userSession } from '$lib/stores/auth';
  import { agentStore } from '$lib/stores/agent';

  const session = $derived($userSession);
</script>

<div class="flex h-screen bg-zinc-950">
  <!-- Narrow document context sidebar -->
  <aside class="w-64 border-r border-zinc-800 flex flex-col p-4 gap-3 overflow-y-auto">
    <h2 class="text-xs font-semibold text-zinc-400 uppercase tracking-wider">Context</h2>
    {#if $agentStore.activeDocumentId}
      <p class="text-sm text-zinc-300">Document linked</p>
      <!-- Document title + link -->
    {:else}
      <p class="text-xs text-zinc-500">No document open. Open a workspace to link context.</p>
    {/if}
  </aside>

  <!-- Full chat pane -->
  <main class="flex-1 flex flex-col min-h-0">
    {#if session}
      <AgentChat
        projectId={session.project_id}
        fullScreen={true}
      />
    {/if}
  </main>
</div>
```

**Navigation update** — add "Chat" link to sidebar nav in `+layout.svelte`.

---

### 3.4 Feature 3: Tool-Calling Agent Loop

**Problem:** `handleGatewayAgentChat` calls the LLM once and returns. The 17 MCP tools and the new `get_document` tool are registered but never invoked. `CompletionRequest.Tools` and `CompletionResponse.ToolCalls` are dead fields in all providers.

**Solution (grounded in the strategic scout — Q2 Route C: Normalize at the Provider Layer):**

The gateway tool loop is clean and normalized — it operates entirely on `Tools[]` and `ToolCalls[]` without knowing which provider or parsing strategy was used. Each provider implementation handles the translation between its native format and the normalized types. This is the minimum-delta path: the Go interface already has the right shape. Wiring it up in each provider fills in already-stubbed behavior.

#### 3.4.1 Normalized Tool Loop (Gateway)

**`server/handle_gateway.go` — `runAgentLoop()`:**

```go
const maxToolIterations = 5

type ToolCallResult struct {
    Tool   string                 `json:"tool"`
    Args   map[string]interface{} `json:"args"`
    Result interface{}            `json:"result"`
    Error  string                 `json:"error,omitempty"`
}

func (s *Server) runAgentLoop(
    ctx context.Context,
    llmProvider provider.ModelProvider,
    systemPrompt string,
    userMessage string,
) (string, []ToolCallResult, error) {

    // Build normalized tool list from registry
    registeredTools := tools.ListTools()
    providerTools := make([]provider.Tool, 0, len(registeredTools))
    for _, t := range registeredTools {
        providerTools = append(providerTools, provider.Tool{
            Name:        t.Name,
            Description: t.Description,
            Parameters:  t.Parameters,
        })
    }

    messages := []provider.Message{
        {Role: "system", Content: systemPrompt},
        {Role: "user", Content: userMessage},
    }
    var allToolCalls []ToolCallResult

    for i := 0; i < maxToolIterations; i++ {
        resp, err := llmProvider.GenerateCompletion(ctx, &provider.CompletionRequest{
            Messages: messages,
            Tools:    providerTools,  // populated — provider normalizes to its native format
        })
        if err != nil {
            return "", allToolCalls, fmt.Errorf("LLM error: %w", err)
        }

        // Normalized check: did the provider return tool calls?
        if len(resp.ToolCalls) == 0 {
            // No tool calls — final answer
            return resp.Content, allToolCalls, nil
        }

        // Execute each tool call
        for _, tc := range resp.ToolCalls {
            result, toolErr := s.executeTool(ctx, tc.Name, tc.Arguments)
            tcResult := ToolCallResult{Tool: tc.Name, Args: tc.Arguments}
            if toolErr != nil {
                tcResult.Error = toolErr.Error()
            } else {
                tcResult.Result = result
            }
            allToolCalls = append(allToolCalls, tcResult)

            // Feed result back — use provider-neutral "tool" role where supported,
            // or a user message with the result for text-only providers
            messages = append(messages,
                provider.Message{Role: "assistant", Content: resp.Content, ToolCalls: resp.ToolCalls},
                provider.Message{
                    Role:    "tool",
                    Content: formatToolResult(tc.Name, result, toolErr),
                    ToolCallID: tc.ID,
                },
            )
        }
    }

    return "I reached the maximum number of tool iterations.", allToolCalls, nil
}

func (s *Server) executeTool(ctx context.Context, toolName string, args map[string]interface{}) (interface{}, error) {
    toolDef, err := tools.GetTool(toolName)
    if err != nil {
        return nil, fmt.Errorf("tool not found: %s", toolName)
    }
    if toolDef.Function == nil {
        return nil, fmt.Errorf("tool has no function: %s", toolName)
    }
    return toolDef.Function(ctx, args)
}
```

The loop is provider-agnostic. It sends `Tools[]`, receives `ToolCalls[]`, and executes. The translation happens inside each provider's `GenerateCompletion`.

#### 3.4.2 Provider Implementations

**Anthropic provider — `server/services/providers/anthropic.go`:**

```go
func (p *AnthropicProvider) GenerateCompletion(ctx context.Context, req *CompletionRequest) (*CompletionResponse, error) {
    // Map normalized Tools[] → Anthropic tools array
    anthropicTools := make([]anthropic.Tool, 0, len(req.Tools))
    for _, t := range req.Tools {
        anthropicTools = append(anthropicTools, anthropic.Tool{
            Name:        t.Name,
            Description: t.Description,
            InputSchema: t.Parameters,
        })
    }

    // ... build Anthropic API request with tools ...

    // Parse response: tool_use content blocks → normalized ToolCalls[]
    var toolCalls []ToolCall
    var textContent string
    for _, block := range resp.Content {
        switch block.Type {
        case "tool_use":
            toolCalls = append(toolCalls, ToolCall{
                ID:        block.ID,
                Name:      block.Name,
                Arguments: block.Input,
            })
        case "text":
            textContent += block.Text
        }
    }

    return &CompletionResponse{
        Content:   textContent,
        ToolCalls: toolCalls,  // normalized — gateway loop doesn't care about source format
    }, nil
}
```

**OpenAI provider — `server/services/providers/openai.go`:**

```go
// Map normalized Tools[] → OpenAI tools array (function calling format)
// Parse tool_calls in response → normalized ToolCalls[]
// Pattern is identical in shape to Anthropic above.
```

**Ollama provider — `server/services/providers/ollama.go`:**

```go
func (p *OllamaProvider) GenerateCompletion(ctx context.Context, req *CompletionRequest) (*CompletionResponse, error) {
    model := p.resolveModel(ctx, "")

    // Probe tool support by model name pattern
    toolCapable := p.modelSupportsTools(model)

    if toolCapable && len(req.Tools) > 0 {
        // Use Ollama's native tools field in /api/chat
        ollamaTools := mapToOllamaTools(req.Tools)
        resp, err := p.chatWithTools(ctx, model, req.Messages, ollamaTools)
        if err == nil && len(resp.Message.ToolCalls) > 0 {
            return &CompletionResponse{
                Content:   resp.Message.Content,
                ToolCalls: normalizeOllamaToolCalls(resp.Message.ToolCalls),
            }, nil
        }
        // Fall through to text-mode fallback on error
    }

    // Text-mode fallback: inject TOOL_CALL: format into system prompt
    // and parse the marker from the LLM response
    // This path is invisible to the gateway loop — it still returns ToolCalls[]
    return p.generateWithTextToolFallback(ctx, model, req)
}

func (p *OllamaProvider) modelSupportsTools(model string) bool {
    toolCapablePatterns := []string{"qwen", "llama3.1", "llama3.2", "mistral-nemo"}
    model = strings.ToLower(model)
    for _, pattern := range toolCapablePatterns {
        if strings.Contains(model, pattern) {
            return true
        }
    }
    return false
}

func (p *OllamaProvider) generateWithTextToolFallback(
    ctx context.Context,
    model string,
    req *CompletionRequest,
) (*CompletionResponse, error) {
    // Inject TOOL_CALL: format instructions into the system message
    // Parse TOOL_CALL: marker from response text
    // Return as normalized ToolCalls[] — gateway loop never sees the text format
    // ...
    content := resp.Message.Content
    if tc := parseTextToolCall(content); tc != nil {
        return &CompletionResponse{
            Content:   stripTextToolCall(content),
            ToolCalls: []ToolCall{{Name: tc.Tool, Arguments: tc.Args}},
        }, nil
    }
    return &CompletionResponse{Content: content}, nil
}
```

The `TOOL_CALL:` text format is demoted to an Ollama-internal detail. It is never referenced in the gateway tool loop or the system prompt for Anthropic/OpenAI users.

#### 3.4.3 Tool Definitions in System Prompt

For text-mode Ollama fallback, the tool instructions are injected only if the model is not tool-capable. For native-tool providers, the provider API carries the tool definitions — no system prompt needed.

```go
func (s *Server) buildSystemPrompt(agent *gateway.AgentConfig, ctx *DocumentContext, providerNativeTools bool) string {
    // ... base prompt + document index + prior context ...

    if !providerNativeTools {
        // Text-mode fallback: list tools in system prompt
        base += "\n## Available Tools\n"
        base += "To use a tool, include this exact format in your response:\n"
        base += "TOOL_CALL: {\"tool\": \"<name>\", \"args\": {<arguments>}}\n\n"
        for _, t := range tools.ListTools() {
            base += fmt.Sprintf("- %s: %s\n", t.Name, t.Description)
        }
    }
    // For native-tool providers: tools are passed in CompletionRequest.Tools,
    // no system prompt injection needed.

    return base
}
```

#### 3.4.4 Frontend Tool Call Display

`AgentChat.svelte` already has a `toolCalls` display area. The gateway returns the normalized list:

```go
c.JSON(http.StatusOK, gin.H{
    "agent_id":     agentID,
    "response":     responseText,
    "tool_calls":   toolCalls,    // []ToolCallResult — populated from normalized loop
    "patch_intent": patchIntent,  // from §3.2.5, if present
    "disposition": map[string]interface{}{
        "pacing": agentConfig.Pacing,
        "depth":  agentConfig.Depth,
    },
})
```

---

### 3.5 Feature 4: Live Memory Counter

**Problem:** After calling `storeMemory()`, the `memoryItems` array in `AgentChat.svelte` is not updated. The counter shows "(0)" because it reads `memoryItems.length` which is only populated after a search.

**Solution:** After every `storeMemory()` call, append the returned `MemoryItem` directly to `memoryItems` in the store — no search needed.

#### 3.5.1 AgentChat.svelte Fix

```typescript
async function saveToMemory(content: string) {
  try {
    savingMemory = true;
    const stored = await storeMemory(content, 'conversation', 'private');
    // Immediately reflect in the memory list — no search round-trip needed
    memoryItems = [
      {
        id: stored.id,
        memory_type: stored.memory_type,
        content: stored.content,
        relevance_score: undefined,
      },
      ...memoryItems,
    ].slice(0, 20); // keep last 20 in sidebar
  } catch (e) {
    console.error('saveToMemory failed:', e);
  } finally {
    savingMemory = false;
  }
}
```

The memory counter in the component header reads `memoryItems.length`, which now increments immediately on save.

#### 3.5.2 Memory Store (Persistent Count)

For the full-screen `/memory` route to show an accurate count across sessions, the memory panel fetches the total from the gateway list endpoint (see Feature 5).

---

### 3.6 Feature 5: Memory Panel — Paginated List, Detail View, Edit

**Problem:** Memory panel only shows search results. No browsing, no detail view, no editing.

**Solution:** Add a `GET /v1/memory` gateway endpoint for paginated listing; update `MemoryPanel.svelte` with a paginated list view, a detail slide-out, and inline editing.

#### 3.6.1 Gateway — List Endpoint

**`server/handle_memory.go` — New handler:**

```go
// GET /v1/memory?page=1&limit=20&type=note
func (s *Server) handleListMemories(c *gin.Context) {
    page := queryIntDefault(c, "page", 1)
    limit := queryIntDefault(c, "limit", 20)
    memType := c.Query("type") // optional filter

    userID := getUserIDFromContext(c)
    items, total, err := s.memoryManager.List(c.Request.Context(), userID, page, limit, memType)
    if err != nil {
        s.errorResponse(c, http.StatusInternalServerError, "list_failed", err.Error())
        return
    }

    c.JSON(http.StatusOK, gin.H{
        "items":      items,
        "total":      total,
        "page":       page,
        "limit":      limit,
        "total_pages": int(math.Ceil(float64(total) / float64(limit))),
    })
}
```

**`server/handle_memory.go` — Edit handler (PUT /v1/memory/:id):**

```go
func (s *Server) handleUpdateMemory(c *gin.Context) {
    memID := c.Param("id")
    var req struct {
        Content string `json:"content" binding:"required"`
    }
    if err := c.ShouldBindJSON(&req); err != nil {
        s.errorResponse(c, http.StatusBadRequest, "invalid_request", err.Error())
        return
    }
    userID := getUserIDFromContext(c)
    updated, err := s.memoryManager.Update(c.Request.Context(), userID, memID, req.Content)
    if err != nil {
        s.errorResponse(c, http.StatusInternalServerError, "update_failed", err.Error())
        return
    }
    c.JSON(http.StatusOK, updated)
}
```

#### 3.6.2 Tauri API

```typescript
// tauri.ts additions
export const listMemories = (page: number = 1, limit: number = 20, type?: string): Promise<MemoryListResponse> =>
  invoke<MemoryListResponse>('list_memories', { page, limit, memoryType: type });

export const updateMemory = (memoryId: string, content: string): Promise<MemoryItem> =>
  invoke<MemoryItem>('update_memory', { memoryId, content });

export interface MemoryListResponse {
  items: MemoryItem[];
  total: number;
  page: number;
  limit: number;
  total_pages: number;
}
```

#### 3.6.3 MemoryPanel.svelte — New Layout

```
┌─────────────────────────────────────────────────┐
│  Knowledge Memory  (47)          [Search ___]   │
│  ─────────────────────────────────────────────  │
│  BROWSE  │  SEARCH                              │
│  ─────────────────────────────────────────────  │
│  ● note    The document uses BibTeX... ›        │
│  ● insight AI as thinking partner first...  ›   │
│  ● fact    qwen3:8b runs on 11434...        ›   │
│  ─────────────────────────────────────────────  │
│  ← 1 2 3 ... 5 →                               │
└─────────────────────────────────────────────────┘
                    ↓ click item
┌─────────────────────────────────────────────────┐
│  ← Back    note   2026-02-20                    │
│  ─────────────────────────────────────────────  │
│  [The document uses BibTeX for references.    ] │
│  [                                            ] │
│                              [Cancel] [Save]    │
└─────────────────────────────────────────────────┘
```

**Implementation:** Two view states — `list` and `detail`. Detail view is a slide-in panel (CSS transform) showing the full content in an editable `<textarea>`. Save calls `updateMemory()`.

---

### 3.7 Feature 6: Provider Key Management

**Problem:** Only Ollama works out of the box. To use Anthropic or OpenAI, users must set env vars manually. No UI exists for key management.

**Solution:** Add a Settings page section for provider API keys, stored via Tauri's secure storage (platform keychain), read by the gateway at request time via the existing `KeyResolver` pattern.

#### 3.7.1 Gateway — Provider Settings Endpoints

```go
// POST /v1/settings/providers
// Body: { "provider": "anthropic", "api_key": "sk-ant-..." }
func (s *Server) handleSetProviderKey(c *gin.Context) {
    var req struct {
        Provider string `json:"provider" binding:"required"`
        APIKey   string `json:"api_key" binding:"required"`
    }
    if err := c.ShouldBindJSON(&req); err != nil {
        s.errorResponse(c, http.StatusBadRequest, "invalid_request", err.Error())
        return
    }
    // Store in secure storage and register/refresh provider
    if err := s.pluginManager.SetProviderKey(req.Provider, req.APIKey); err != nil {
        s.errorResponse(c, http.StatusInternalServerError, "set_key_failed", err.Error())
        return
    }
    c.JSON(http.StatusOK, gin.H{"provider": req.Provider, "status": "configured"})
}

// GET /v1/settings/providers
// Returns list of providers with configured/unconfigured status (never returns keys)
func (s *Server) handleGetProviderSettings(c *gin.Context) {
    providers := s.pluginManager.GetProviderStatus()
    c.JSON(http.StatusOK, gin.H{"providers": providers})
}
```

#### 3.7.2 Settings Page — Provider Section

**`src/routes/settings/+page.svelte` — Provider keys section:**

```svelte
<section class="space-y-4">
  <h2 class="text-sm font-semibold text-zinc-300">AI Providers</h2>
  {#each providerDefs as provider}
    <div class="flex items-center gap-3 p-3 rounded-lg bg-zinc-900 border border-zinc-800">
      <div class="flex-1">
        <p class="text-sm font-medium text-zinc-200">{provider.displayName}</p>
        <p class="text-xs text-zinc-500">{provider.description}</p>
      </div>
      {#if provider.configured}
        <span class="text-xs text-emerald-400">● Configured</span>
        <button onclick={() => clearKey(provider.id)}
          class="text-xs text-zinc-500 hover:text-red-400">Remove</button>
      {:else}
        <button onclick={() => openKeyDialog(provider)}
          class="px-3 py-1 text-xs rounded bg-zinc-700 hover:bg-zinc-600 text-zinc-200">
          Add Key
        </button>
      {/if}
    </div>
  {/each}
</section>
```

**Providers to support at v1.1:**

| Provider | Display Name | Env Var | Status |
|----------|-------------|---------|--------|
| `ollama` | Ollama (Local) | `OLLAMA_HOST` | Auto-detected |
| `anthropic` | Anthropic (Claude) | `ANTHROPIC_API_KEY` | Key required |
| `openai` | OpenAI (GPT) | `OPENAI_API_KEY` | Key required |

**Key storage:** Stored in Tauri's `tauri-plugin-stronghold` or platform keychain. The gateway's `KeyResolver` pattern (already in `BaseProvider`) reads from secure storage at request time — no keys in logs or config files.

---

### 3.8 Feature 7: Pipelines — Chat Integration

**Problem:** Pipelines page is a functional DAG builder but completely disconnected from chat. Users can't ask the agent to run a pipeline, and pipeline results don't flow back to the workspace.

**Solution:** Three integration points:
1. Agent can propose and submit a pipeline from chat (via natural language)
2. Pipeline completion can push output back to workspace document
3. Chat can link to a pipeline job in-progress

#### 3.8.1 Pipeline Proposal from Chat

When the user types "run the Write+PDF pipeline on this document", the agent recognizes the intent and emits a `pipeline_proposal` in the response instead of (or alongside) text.

**Extended `AgentResponse`:**

```typescript
export interface AgentResponse {
  agent_id: string;
  response?: string;
  tool_calls: ToolCall[];
  finish_reason?: string;
  document_patch?: DocumentPatch;   // from Feature 1
  pipeline_proposal?: PipelineProposal; // NEW
}

export interface PipelineProposal {
  preset: string;          // "write_pdf" | "write_slides" | etc.
  steps: string[];         // tool names
  description: string;     // human-readable summary
}
```

**In `AgentChat.svelte`**, when a `pipeline_proposal` is returned, display a confirmation card:

```svelte
{#if message.pipelineProposal}
  <div class="mt-2 p-3 rounded-lg bg-zinc-800 border border-zinc-700">
    <p class="text-xs text-zinc-400 mb-2">Pipeline proposal</p>
    <p class="text-sm text-zinc-200">{message.pipelineProposal.description}</p>
    <div class="flex gap-2 mt-3">
      <button onclick={() => submitProposedPipeline(message.pipelineProposal)}
        class="px-3 py-1 text-xs rounded bg-emerald-700 hover:bg-emerald-600 text-white">
        Run Pipeline
      </button>
      <button class="px-3 py-1 text-xs rounded bg-zinc-700 text-zinc-300">Dismiss</button>
    </div>
  </div>
{/if}
```

#### 3.8.2 Pipeline Result → Workspace

When a pipeline completes and includes document output, the pipelines page emits an event to the agent store. The workspace listens and offers to import the result.

**`pipelines/+page.svelte` addition:**

```typescript
// When job completes with document output
async function checkJobCompletion(job: DagJob) {
  if (job.status === 'completed' && job.output_document) {
    agentStore.setPipelineResult({
      jobId: job.id,
      content: job.output_document,
      format: job.output_format,
    });
  }
}
```

#### 3.8.3 Pipelines Page Polish

Beyond chat integration, the pipelines page needs:
- Fix: "Run" button should be disabled when no steps are selected
- Fix: Job list should persist across navigation (move to a store, not local state)
- Add: "Open in Workspace" button on completed jobs that produced documents
- Add: Step output preview in a collapsible panel (already partially built)
- Add: Error state with retry button for failed steps

---

## 4. Implementation Plan

### 4.1 Phased Approach

**Timeline:** 3 weeks

| Phase | Duration | Focus | Deliverables |
|-------|----------|-------|--------------|
| 1 | Week 1 | Foundation — State & Memory | Agent store, persistent chat, memory counter fix, memory panel pagination |
| 2 | Week 2 | Intelligence — Context & Tools | Document context injection, tool-calling loop, workspace write-back |
| 3 | Week 3 | Integration — Providers & Pipelines | Provider key management, pipelines chat integration, full-screen chat route, polish |

### 4.2 Week-by-Week Breakdown

**Week 1: Foundation**

- [ ] Create `src/lib/stores/agent.ts` (agent store with persistent init)
- [ ] Add `chat_messages` SQLite table and Rust commands (`load_chat_history`, `save_chat_message`)
- [ ] Migrate `AgentChat.svelte` from local `$state` to agent store
- [ ] Fix memory counter: append stored item to `memoryItems` immediately after `storeMemory()`
- [ ] Add `GET /v1/memory` (list) and `PUT /v1/memory/:id` (edit) to gateway
- [ ] Add Rust IPC commands: `list_memories`, `update_memory`
- [ ] Update `MemoryPanel.svelte`: paginated list view, detail/edit slide-in

**Success Criteria:** Navigate away from workspace and back — chat history intact; save a response — counter increments; memory panel shows paginated list of all saved items with editable detail view.

**Week 2: Intelligence**

The three doc-related features ship in scout-recommended order: write-back (frontend only) first, then memory tagging, then full tool loop.

- [ ] **Q3 (Patch-by-Intent)** — frontend only, no gateway changes needed:
  - Add `parsePatchIntent()` in Go gateway handler; return `patch_intent` in response
  - Add `PATCH_INTENT:` instruction to system prompt in gateway
  - Render pending patch confirmation card in `AgentChat.svelte`
  - Implement `applyPatch()` calling existing `updateSectionContent` Tauri command
  - Auto-save accepted patch to memory with `document_id` tag
- [ ] **Q1 (Document Tool + Memory-as-History)** — lightweight, uses existing infrastructure:
  - Register `get_document` tool in gateway tool registry
  - Add `GET /v1/gateway/documents/:id` endpoint (or internal fulfillment path)
  - Update `buildSystemPrompt()` to inject section index (not content) + prior context
  - Update `chatWithAgent` to pass `documentSectionIndex`, `documentMode`, `priorContext`
  - Tag all `storeMemory` calls with `document_id` metadata
  - On session init: `searchMemory("document:{docId}", 5)` → inject as prior context
- [ ] **`runAgentLoop()` scaffolding** — wire normalized loop (Q2 prep):
  - Implement `runAgentLoop()` with `Tools[]` / `ToolCalls[]` normalized contract
  - Implement `executeTool()` against tool registry
  - Rebuild gateway binary and test: "call get_document" → gateway logs show tool execution

**Success Criteria:** Ask agent to draft an Introduction → prose response + PATCH_INTENT footer + Apply button appears; click Apply → section created; counter shows accepted edit saved to memory.

**Week 3: Integration + Q2 Provider Tool Wiring**

- [ ] **Q2 (Normalize at Provider Layer)** — fill in provider implementations:
  - Anthropic provider: map `Tools[]` → `tools` array in API request; parse `tool_use` blocks → `ToolCalls[]`
  - OpenAI provider: same pattern for `tool_calls` in response
  - Ollama provider: add `modelSupportsTools(model)` probe; use `tools` field on capable models; `generateWithTextToolFallback()` for others — both return normalized `ToolCalls[]`
  - Test: ask agent "convert this to PDF" → `zen_sci_latex:convert_to_pdf` invoked, result in chat
- [ ] Add `POST /v1/settings/providers` and `GET /v1/settings/providers` to gateway
- [ ] Implement secure key storage via Tauri stronghold/keychain
- [ ] Update Settings page with provider configuration UI (Anthropic, OpenAI)
- [ ] Create `src/routes/chat/+page.svelte` (full-screen chat with context sidebar)
- [ ] Add Chat link to nav layout
- [ ] Add `pipeline_proposal` to agent response type; render proposal card in `AgentChat`
- [ ] Move pipeline job list to a store (persist across navigation)
- [ ] Add "Open in Workspace" to completed pipeline jobs
- [ ] Fix pipelines page: disabled Run button, step retry, error states
- [ ] E2E checklist run (all 8 success criteria)

**Success Criteria:** All success criteria from Section 2 pass in sequence; Anthropic provider invokes `zen_sci_latex:convert_to_pdf` natively; Ollama (qwen3:8b) invokes the same tool via native tools field.

### 4.3 Dependencies & Prerequisites

**Required Before Starting:**
- ✅ Gateway binary rebuilt (done 2026-02-20)
- ✅ Chat 500 error fixed (done 2026-02-20)
- ✅ MCP servers connecting with correct path (zen-sci/servers)
- ✅ Ollama running with qwen3:8b
- ✅ Tauri HTTP timeout increased to 180s

**Parallel Work (Week 2 & 3 can partially overlap):**
- Memory panel pagination (Week 1) is independent of tool loop (Week 2)
- Provider key management UI (Week 3) is independent of pipeline integration

**Blocking Dependencies:**
- Agent store (Week 1) must be complete before full-screen chat route (Week 3)
- Tool loop (Week 2) must be complete before pipeline proposal from chat (Week 3)
- `GET /v1/memory` endpoint must exist before memory panel pagination works

### 4.4 Testing Strategy

**Unit Tests (Rust):**
- `load_chat_history` / `save_chat_message` round-trip
- `list_memories` pagination logic
- `update_memory` authorization check (user can only edit their own)

**Unit Tests (Go):**
- `parseToolCall()` — valid JSON, malformed JSON, no tool call, nested braces
- `runAgentLoop()` — mock LLM, assert tool executed and result fed back
- `buildSystemPrompt()` — with and without document context

**Unit Tests (TypeScript):**
- Agent store: init, persist, reset
- Memory counter: increments after storeMemory, doesn't decrement on navigate

**Integration Tests:**
- Full chat → tool call → result cycle against live gateway (Ollama)
- Document context injection: gateway receives document content in request
- Memory list + edit: create → list → edit → verify updated

**Manual QA Checklist (Week 3):**
- [ ] Send "summarize this document" in workspace → agent uses document content
- [ ] Send "convert this to PDF" → tool call fires, result shown
- [ ] Navigate workspace → `/chat` → workspace → chat history intact
- [ ] Save 3 responses → counter shows "(3)"
- [ ] Open Memory panel → paginated list → click item → edit → save
- [ ] Add Anthropic key in Settings → provider appears in agent dropdown
- [ ] Ask agent to "run the Write+PDF pipeline" → proposal card appears → confirm → pipeline starts

---

## 5. Risk Assessment & Mitigation

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Gateway memory backing store not durable across restarts | Medium | Medium | Verify on startup; if not durable, store memory items in SQLite via Tauri and construct context string client-side — bypasses gateway memory search entirely |
| Ollama model name probe misses tool-capable models | Low | Medium | Start with `qwen`, `llama3.1`, `llama3.2` patterns; add `OLLAMA_TOOL_CAPABLE=true` env override for non-matching model names |
| `PATCH_INTENT:` footer omitted by weak Ollama models | Medium | Low | Graceful degradation: response still useful as prose; write-back simply doesn't trigger; user can always manually copy to section |
| `get_document` endpoint adds round-trip latency | Low | Low | For local-first v1.1, bootstrap document structure into `DocumentContext` at chat request time; full endpoint is a v1.2 expansion |
| Tool loop timeout (Ollama + tool execution chained) | Medium | Medium | Max 5 iterations; 30s tool execution timeout separate from 3-min LLM timeout |
| SQLite chat history grows unbounded | Low | Low | Retain last 200 messages per agent; prune on init |
| Secure key storage not available on all platforms | Low | Medium | Fall back to gateway-side env var if stronghold unavailable; warn user in UI |
| Memory panel list endpoint not in gateway v1.0 | High | Medium | Already identified as needed; implement in Week 1 before panel work |

---

## 6. Rollback & Contingency

**Feature Flags (gateway-config.yaml additions):**

```yaml
features:
  tool_calling: true          # enables runAgentLoop with normalized Tools[]/ToolCalls[]; false = direct LLM call (v1.0 behavior)
  get_document_tool: true     # registers get_document as a tool; false = omit from tool registry
  document_context: true      # enables section index injection in system prompt
  patch_intent: true          # enables PATCH_INTENT: parsing and patch_intent in response
  provider_settings: true     # enables /v1/settings/providers endpoints
  memory_as_history: true     # enables document_id tagging on store_memory calls
```

**Rollback Procedure (if tool loop causes regressions):**
1. Set `tool_calling: false` in gateway config (+ `get_document_tool: false`, `patch_intent: false`)
2. `kill -HUP <gateway_pid>` to reload config (or restart gateway)
3. Chat returns to direct LLM call — no tool invocation, same as v1.0

**Rollback Procedure (if Patch-by-Intent causes issues):**
1. Set `patch_intent: false` in gateway config
2. Apply button never appears — agent responses are read-only prose; user copies manually

**Monitoring:**
- Log every tool call attempt: `slog.Info("tool_call", "tool", toolName, "agent", agentID)`
- Log every loop iteration count: alert if consistently hitting `maxToolIterations`
- Log memory counter value on save to catch regressions

---

## 7. Documentation & Communication

**User-Facing (for Cruz's own use):**
- [ ] Update README: correct `ZEN_SCI_SERVERS_ROOT` path (`zen-sci/servers`)
- [ ] Add "How the agent uses tools" section to README

**Developer Documentation:**
- [ ] Update `STATUS.md` after each week's milestones
- [ ] Document `TOOL_CALL:` format in gateway README
- [ ] Add agent store pattern to `ARCHITECTURE.md`

**Handoff document (after v1.1):**
- [ ] `docs/handoffs/2026-02-20_handoff_v1.0-to-v1.1.md`

---

## 8. Appendices

### 8.1 Scout Decisions Reference

All three architecture decisions below were made by the strategic scout (`docs/scouts/2026-02-20_portal_v1.1_strategic_scout.md`) and are grounded in the actual codebase state as of 2026-02-20.

| Question | Recommended Route | Key Rationale |
|:---------|:-----------------|:--------------|
| Q1 Agent Persistence + Document Readability | Route B — Memory-as-History + `get_document` as tool | Gateway sidecar restarts make agent ID persistence illusory; semantic memory survives; document is a resource the agent queries, not data the host pre-injects |
| Q2 Tool Calling | Route C — Normalize at Provider Layer | Existing Go types (`Tools[]`/`ToolCalls[]`) already designed for this; gateway loop stays clean and provider-agnostic |
| Q3 Document Write-Back | Route C — Patch-by-Intent with Human Confirmation | Human confirmation is correct for academic writing; no risk of silent corruption; uses existing `update_section_content` Tauri command |

**Implementation order (from scout):**
1. Q3 first (Week 2 start) — frontend only, highest user value, no gateway changes
2. Q1 second (Week 2 mid) — memory tagging + `get_document` tool, minimal new code
3. Q2 last (Week 3) — provider wiring, fills existing stubs in Go

### 8.2 Provider Tool Capability Reference

| Provider | Tool Calling | Format | Fallback |
|----------|-------------|--------|----------|
| Anthropic (Claude) | Native | `tools` array + `tool_use` content blocks | N/A — always capable |
| OpenAI (GPT-4+) | Native | `tools` array + `tool_calls` in response | N/A — always capable |
| Ollama (qwen3, llama3.1+) | Native | `/api/chat` `tools` field | Text-mode TOOL_CALL: marker |
| Ollama (other models) | Text-mode | `TOOL_CALL:` footer in response | Response without tool calls |

`TOOL_CALL:` text format is an Ollama-internal implementation detail in the provider layer. The gateway tool loop never sees it — only normalized `ToolCalls[]`.

### 8.3 v1.2 Candidates

The following are explicitly deferred from v1.1:

- **Streaming responses (SSE)** — LLM tokens stream to chat UI as they arrive
- **`search_memory` as a registered tool** — agent proactively retrieves its own prior work mid-conversation (Q2 normalization in v1.1 unlocks this cleanly)
- **Project-scoped agent** — one agent per project (multiple documents), context window management (scout Q1 Route C — wrong scope for v1.1, correct for v2)
- **Automatic write-back without confirmation** — for trusted workflows only; requires track record of correct section targeting
- **Collaborative section ownership** — multiple agents can own different sections
- **MCP Apps integration** — Phase 4 apps launched from agent chat

### 8.4 Open Items to Verify Before Week 2

From the strategic scout — must be answered before implementing Q1:

1. **Gateway memory backing store durability** — Is `POST /v1/memory` data durable across gateway process restarts? Check the gateway's memory manager backing store (likely in-memory map vs. SQLite). If not durable → implement SQLite fallback in Tauri layer.
2. **Ollama tool-capable model name patterns** — Confirm `qwen3:8b` responds correctly to the Ollama `/api/chat` tools field. Run: `curl -X POST http://localhost:11434/api/chat -d '{"model":"qwen3:8b","messages":[...],"tools":[{"type":"function","function":{"name":"test","description":"test","parameters":{}}}]}'` and verify `tool_calls` in response.
3. **Gateway tool loop current state** — `CompletionResponse.ToolCalls` is populated by the Ollama provider? (Currently confirmed: no — ignored. Q2 Week 3 fills this in.)

### 8.5 References

1. `docs/specs/zen-sci-portal-v1.0-spec.md` — Base portal spec (Revision R2)
2. `docs/scouts/2026-02-20_portal_v1.1_strategic_scout.md` — Scout decisions for Q1/Q2/Q3
3. `docs/ARCHITECTURE.md` — 13 architecture decisions, semantic cluster model
4. `docs/CONTEXT.md` — ZenSci module overview, AI-as-thinking-partner philosophy
5. `docs/STATUS.md` — Live project state as of 2026-02-19
6. `server/handle_gateway.go` — Current chat handler (direct LLM call, bypasses orchestration)
7. `server/services/providers/ollama.go` — Ollama provider (resolveModel auto-detect added 2026-02-20)
8. `src/lib/components/AgentChat.svelte` — Current 392-line chat component
9. `src/lib/components/MemoryPanel.svelte` — Current memory panel
10. `src/routes/pipelines/+page.svelte` — Current 673-line DAG builder
