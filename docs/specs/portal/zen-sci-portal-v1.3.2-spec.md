# ZenSci Portal v1.3.2: The Scholar's Workshop

**Author:** Look & Feel Agent + Alfonso Morales (ZenithScience)
**Status:** Draft
**Created:** 2026-02-20
**Grounded In:** v1.1 durability patch (shipped 2026-02-20), v1.2 UX/UI spec (UIUX.md synthesis, 14 sources), v1.3 surface completion spec (60+ endpoint audit), 3 strategic scouts (desktop/web/mobile), 3 audits (health/portal-status/durability)
**Supersedes:** `zen-sci-portal-v1.2-spec.md`, `v1.3-surface-completion-specification.md`
**Versioning Note:** Numbered v1.3.2 to reserve v1.3.0 and v1.3.1 for anticipated bugfix patches discovered during implementation.

---

## 1. Vision

> Wire every existing route, component, and backend endpoint into a coherent, usable surface — then make that surface feel like a scholar's workshop: calm, typographically rich, responsive, and spatially coherent.

**The Core Insight:**

ZenSci's portal sits on top of a powerful backend: 57 registered gateway routes, 6 MCP servers exposing 17 tools, a memory garden system, MCP app lifecycle, document fetch, tracing, and provider key management. The frontend consumes only 9 of those 57 endpoints. Every route exists but several have stubbed interactions, disconnected data flows, or missing context wiring. And the routes that *do* work feel like a developer dashboard, not a research environment.

v1.3.2 closes both gaps in a single release:
- **Surface completion** (from v1.3): wire 13 additional gateway endpoints, make every button do what the user expects, connect every component to the data it needs.
- **Experience layer** (from v1.2): streaming responses, scholarly typography, spatial coherence between agent and document, ambient memory, honest offline/online communication.

Neither half is sufficient alone. A complete surface with a frozen chat is broken. A beautiful interface with dead buttons is dishonest. v1.3.2 ships both.

**What Makes This Different:**

This is neither a feature release nor a polish release. It is a **completion + experience** release. Every change either connects to an existing endpoint or makes an existing capability visible, trustworthy, and calm. The only new Go code is a single SSE streaming endpoint. The only new Rust code is thin IPC wrappers. The bulk of the work is in SvelteKit components, stores, and CSS — the layer the researcher actually touches.

The 12 workstreams map to documented gaps: 8 from the UIUX.md synthesis (14 research sources), 6 from the v1.3 gateway audit (57 endpoints vs. 9 consumed). Where the two specs overlap (streaming, MCP apps, patch intent, search), this spec takes the richer design from whichever source is more complete.

---

## 1.5 Current State (Audit Results — 2026-02-20)

**Frontend (SvelteKit + Tauri):**

| Metric | Count |
|--------|-------|
| Source files (`.ts` + `.svelte`) | 35 |
| Routes (`+page.svelte`) | 11 |
| Shared components | 8 |
| Svelte stores | 7 |
| ARIA/role accessibility instances | 105 |
| Frontend API functions (`tauri.ts`) | 35 |
| Tauri commands (Rust) | 34 |
| Rust test functions | 14 (9 unit + 5 integration) |

**Backend (Go — AgenticGateway):**

| Metric | Count |
|--------|-------|
| Go source files | 200 |
| Go test files | 167 |
| Registered routes | 57 |
| MCP servers | 6 |
| MCP tools | 17 |
| **Endpoints consumed by portal** | **9 of 57 (16%)** |

**Key Gap:** Portal consumes 16% of available gateway endpoints. v1.3.2 targets raising this to **39% (22 of 57)** while simultaneously applying the experience layer across all surfaces.

---

## 2. Goals & Success Criteria

**Primary Goals:**
1. **Complete the surface** — Wire all 6 frontend routes to their full backend capabilities. Zero stub routes remaining. Endpoint consumption 9 → 22.
2. **Responsive agent interaction** — Streaming responses eliminate the frozen-interface perception for all providers (Anthropic, OpenAI, Ollama).
3. **Spatial coherence** — The agent's awareness of the document is visible in the workspace; memory is ambient, not hidden.
4. **Scholarly visual identity** — Typography, spacing, and hierarchy convey calm, focused, scholarly work.
5. **Honest capability communication** — Offline vs. online modes are always explicit; no silent failures. Every component interaction completes its data round-trip.

**Success Criteria:**

*Surface Completion:*
- ✅ `/workspace/[docId]` supports draft → structured mode transition via UI button
- ✅ `/workspace/[docId]` shows version history sidebar with diff view
- ✅ `/app/[appId]` launches MCP apps via `POST /v1/gateway/apps/launch` and renders in sandboxed iframe with entry/exit transitions
- ✅ `/search` results are clickable and navigate to `/workspace/[docId]` or preview panel
- ✅ `/search` offers "Search memories" toggle using `POST /v1/memory/search`
- ✅ `/settings` shows MCP server connection status from `GET /admin/mcp/servers`
- ✅ `/settings` shows system health metrics from `GET /admin/health`
- ✅ AgentChat personalizes system prompt based on document format (latex, blog, grant, etc.)
- ✅ AgentChat applies `patch_intent` events to the active document via `updateSectionContent`
- ✅ ToolInvoker auto-fills `content` parameter from active document when invoked from workspace
- ✅ ToolInvoker result can be saved to memory via `POST /v1/memory`

*Experience Layer:*
- ✅ First streaming token appears in ≤2 seconds for Anthropic/OpenAI providers
- ✅ Agent responses render incrementally — no blank wait periods >3 seconds for any provider
- ✅ Offline mode is visually distinct; gateway-required features are visually disabled with explanation
- ✅ A custom typeface pair (Inter + JetBrains Mono) is applied across all routes; system font stack is removed
- ✅ When agent references a document section, that section is highlighted in the editor
- ✅ Patch-by-intent proposals render as visual diffs, not plain text
- ✅ Sidebar navigation has 3-tier visual hierarchy (primary/secondary/utility)
- ✅ Every major empty state has a designed message + primary action
- ✅ Agent memory citations appear in chat messages; referenced memories are linkable
- ✅ `cargo test` and `go test ./...` pass with 0 regressions
- ✅ `npm run check` reports 0 errors

**Non-Goals (Out of Scope):**
- ❌ Web portal or mobile app changes (deferred to v2.0)
- ❌ Multi-user collaboration / CRDT (deferred to v2.0)
- ❌ New MCP server modules or conversion formats
- ❌ Team/permissions system
- ❌ Knowledge graph visualization (v1.4 candidate)
- ❌ Memory Garden frontend — seeds, snapshots (v1.4 candidate)
- ❌ Offline queue / sync (v2.0)
- ❌ Game format generation (mobile scope)

---

## 3. Technical Architecture

### 3.1 System Overview

v1.3.2 touches three layers: heavy frontend, light Rust wrappers, minimal Go:

```
┌──────────────────────────────────────────────────────────┐
│  Frontend (SvelteKit 5)                                  │
│  ├─ 6 routes enhanced (workspace, app, search,           │  ← BULK OF WORK
│  │   settings, chat, memory)                             │
│  ├─ 12 components enhanced or new                        │
│  ├─ Typography system (Inter + JetBrains Mono)           │
│  ├─ Empty states, skeletons, offline indicator           │
│  └─ 3-tier sidebar IA hierarchy                          │
├──────────────────────────────────────────────────────────┤
│  Rust Backend (Tauri v2)                                 │
│  └─ 6 new thin command wrappers (MCP, health, apps,     │  ← THIN WRAPPERS
│      versions, streaming bridge)                         │
├──────────────────────────────────────────────────────────┤
│  Go Backend (AgenticGateway)                             │
│  └─ 1 new endpoint: GET /v1/gateway/agent/:id/stream    │  ← SINGLE ADDITION
│     (SSE streaming for agent chat)                       │
└──────────────────────────────────────────────────────────┘
```

**Newly consumed endpoints (13 wired for the first time):**

| Endpoint | Method | v1.3.2 Consumer |
|----------|--------|-----------------|
| `/v1/gateway/agent/:id/stream` | GET | AgentChat streaming (NEW Go endpoint) |
| `/admin/mcp/servers` | GET | Settings MCP panel |
| `/admin/health` | GET | Settings health dashboard |
| `/v1/settings/providers` | POST | Settings key save |
| `/v1/settings/providers` | GET | Settings key status |
| `/v1/gateway/apps/launch` | POST | App companion launch |
| `/v1/gateway/apps/close` | POST | App companion close |
| `/v1/gateway/apps` | GET | App companion list |
| `/v1/gateway/documents/:id` | GET | Agent document grounding |
| `/v1/memory/search` | POST | Search page toggle (also used by agent) |
| `/v1/memory/:id` | DELETE | Memory panel delete |
| `/v1/memory/:id` | GET | Memory detail fetch |
| `/v1/garden/stats` | GET | Settings garden stats |

---

### 3.2 Feature 1: Streaming Agent Responses (SSE)

**Purpose:** Eliminate the frozen interface. First token within 2 seconds.

**Source:** UIUX.md Theme 1 (R1 — highest priority), v1.1 spec (tagged v1.2 scope), v1.3 spec (streaming polish)

**This is the only new Go code in the entire release.**

**Gateway — SSE Endpoint (Go):**

```go
// gateway/handlers/stream.go

type StreamChunk struct {
    Type    string `json:"type"`    // "delta" | "done" | "error" | "tool_start" | "tool_end"
    Content string `json:"content"` // text delta or tool name
    Index   int    `json:"index"`   // chunk sequence number
}

func (h *Handler) StreamAgentMessage(w http.ResponseWriter, r *http.Request) {
    agentID := chi.URLParam(r, "agentId")
    var req AgentMessageRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid request", http.StatusBadRequest)
        return
    }

    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")
    w.Header().Set("X-Accel-Buffering", "no")
    flusher, ok := w.(http.Flusher)
    if !ok {
        http.Error(w, "streaming not supported", http.StatusInternalServerError)
        return
    }

    chunkCh := make(chan StreamChunk, 64)
    errCh := make(chan error, 1)
    go h.runStreamingLoop(r.Context(), agentID, req, chunkCh, errCh)

    for {
        select {
        case chunk, open := <-chunkCh:
            if !open {
                fmt.Fprintf(w, "data: %s\n\n", mustJSON(StreamChunk{Type: "done"}))
                flusher.Flush()
                return
            }
            fmt.Fprintf(w, "data: %s\n\n", mustJSON(chunk))
            flusher.Flush()
        case err := <-errCh:
            fmt.Fprintf(w, "data: %s\n\n", mustJSON(StreamChunk{
                Type: "error", Content: err.Error(),
            }))
            flusher.Flush()
            return
        case <-r.Context().Done():
            return
        }
    }
}
```

**Provider streaming normalization:** Each provider implements `StreamingProvider`:
- **Anthropic:** Native SSE via `client.Messages.Stream()` — emits `TextDelta` events
- **OpenAI:** Native SSE via `client.Chat.Completions.Stream()` — emits `ChoiceDelta` events
- **Ollama:** Line-delimited JSON via `POST /api/chat` with `stream: true` — parsed line by line

**Tauri IPC bridge (Rust):**

```rust
#[tauri::command]
pub async fn chat_with_agent_stream(
    agent_id: String,
    message: String,
    document_id: Option<String>,
    app_handle: tauri::AppHandle,
) -> Result<(), String> {
    let gateway_url = format!(
        "http://127.0.0.1:8080/v1/gateway/agent/{}/stream", agent_id
    );
    let client = reqwest::Client::new();
    let mut resp = client
        .get(&gateway_url)
        .json(&serde_json::json!({
            "message": message, "document_id": document_id
        }))
        .send().await.map_err(|e| e.to_string())?;

    while let Some(chunk) = resp.chunk().await.map_err(|e| e.to_string())? {
        let text = String::from_utf8_lossy(&chunk);
        for line in text.lines() {
            if let Some(data) = line.strip_prefix("data: ") {
                app_handle.emit("agent-stream-chunk", data)
                    .map_err(|e| e.to_string())?;
            }
        }
    }
    Ok(())
}
```

**Frontend — AgentChat.svelte streaming:**

```typescript
import { listen } from '@tauri-apps/api/event';
import { invoke } from '@tauri-apps/api/core';

let streamingMessage = $state<string>('');
let isStreaming = $state(false);
let streamUnlisten: (() => void) | null = null;

async function sendMessage(content: string) {
    isStreaming = true;
    streamingMessage = '';

    streamUnlisten = await listen<string>('agent-stream-chunk', (event) => {
        const chunk = JSON.parse(event.payload);
        if (chunk.type === 'delta') {
            streamingMessage += chunk.content;
        } else if (chunk.type === 'tool_start') {
            streamingMessage += `\n\`[Calling ${chunk.content}...]\`\n`;
        } else if (chunk.type === 'done' || chunk.type === 'error') {
            finalizeStream();
        }
    });

    try {
        await invoke('chat_with_agent_stream', {
            agentId: $agentStore.agentId,
            message: content,
            documentId: currentDocumentId ?? null,
        });
    } catch (err) {
        streamingMessage = `Error: ${err}`;
        finalizeStream();
    }
}
```

**Streaming message render — blinking cursor + left border accent:**

```svelte
{#if isStreaming && streamingMessage}
  <div class="message-bubble assistant streaming">
    {@html renderMarkdown(streamingMessage)}<span class="cursor" aria-hidden="true">▋</span>
  </div>
{/if}
```

```css
.cursor { animation: blink 0.8s step-end infinite; color: var(--color-brand-500); }
@keyframes blink { 0%, 100% { opacity: 1; } 50% { opacity: 0; } }
.message-bubble.streaming { border-left: 2px solid var(--color-brand-500); padding-left: 0.75rem; }
```

**Tool call badge during streaming:**

```svelte
<div class="tool-badge" role="status" aria-live="polite">
  <span class="tool-icon">⚙</span>
  <span class="tool-name">{toolName}</span>
  <span class="tool-status">{status}</span>
</div>
```

**Performance targets:** ≤2s first token (Anthropic/OpenAI), ≤5s first token (Ollama, model-load dependent).

---

### 3.3 Feature 2: Context-Aware Agent with Personas

**Purpose:** Specialize the agent per document format. Apply patch intents to the document. Surface memory citations.

**Source:** v1.3 Workstream E (personas, patch intent apply), UIUX.md Theme 2 (R4 — agent-document connection), Theme 6 (R7 — ambient memory)

**Agent persona system (frontend-only):**

```typescript
// src/lib/agent/personas.ts
export interface AgentPersona {
  name: string;
  systemPromptSuffix: string;
  suggestedTools: string[];
}

export const PERSONAS: Record<string, AgentPersona> = {
  latex: {
    name: 'LaTeX Research Assistant',
    systemPromptSuffix: 'You are helping write a LaTeX academic document. Suggest citations, validate equations, and help with paper structure. You can use zen_sci_latex tools.',
    suggestedTools: ['zen_sci_latex:validate_document', 'zen_sci_latex:check_citations']
  },
  blog: {
    name: 'Blog Writing Assistant',
    systemPromptSuffix: 'You are helping write a blog post. Focus on readability, SEO, and engagement. You can use zen_sci_blog tools.',
    suggestedTools: ['zen_sci_blog:convert_to_html', 'zen_sci_blog:validate_post']
  },
  grant: {
    name: 'Grant Proposal Assistant',
    systemPromptSuffix: 'You are helping write a grant proposal. Focus on compliance, budget justification, and impact narrative.',
    suggestedTools: ['zen_sci_grant:generate_proposal', 'zen_sci_grant:validate_compliance']
  },
  slides: {
    name: 'Presentation Assistant',
    systemPromptSuffix: 'You are helping create a presentation. Focus on slide flow, visual clarity, and key messages.',
    suggestedTools: ['zen_sci_slides:convert_to_slides']
  },
  newsletter: {
    name: 'Newsletter Editor',
    systemPromptSuffix: 'You are helping draft a newsletter. Focus on audience engagement, clear CTAs, and email best practices.',
    suggestedTools: ['zen_sci_newsletter:convert_to_email']
  },
  paper: {
    name: 'Academic Paper Assistant',
    systemPromptSuffix: 'You are helping write an academic paper. Focus on methodology rigor, literature review, and citation format.',
    suggestedTools: ['zen_sci_paper:convert_to_paper']
  }
};
```

**Patch intent application — unified design (merges v1.2 diff overlay + v1.3 apply logic):**

```svelte
<!-- PatchDiff.svelte — visual diff + Apply/Dismiss (from v1.2) -->
<div class="patch-diff-card" role="region" aria-label="Proposed edit">
  <header class="patch-header">
    <span class="patch-icon">✎</span>
    <span class="patch-description">{patch.description}</span>
    <span class="patch-section">Section: {patch.sectionId}</span>
  </header>

  {#if patch.originalContent}
    <div class="diff-view">
      <div class="diff-remove">
        <span class="diff-marker">−</span>
        <span class="diff-text">{patch.originalContent}</span>
      </div>
      <div class="diff-add">
        <span class="diff-marker">+</span>
        <span class="diff-text">{patch.content}</span>
      </div>
    </div>
  {:else}
    <div class="diff-add">
      <span class="diff-marker">+</span>
      <span class="diff-text">{patch.content}</span>
    </div>
  {/if}

  <footer class="patch-actions">
    <button class="btn-primary btn-sm" on:click={applyPatch}>Apply</button>
    <button class="btn-secondary btn-sm" on:click={() => dismissPatch(patch)}>Dismiss</button>
  </footer>
</div>
```

**Apply logic (from v1.3 — writes to document store):**

```typescript
async function applyPatch() {
  if (!pendingPatchIntent || !$activeDocument) return;
  const { operation, section_id, content } = pendingPatchIntent;

  addHighlight(section_id, 'patch-target');  // v1.2 section highlight

  if (operation === 'replace' && section_id) {
    await updateSectionContent(section_id, $activeDocument.id, content);
  } else if (operation === 'append') {
    const lastSection = $activeDocument.sections.at(-1);
    if (lastSection) {
      await updateSectionContent(lastSection.id, $activeDocument.id,
        lastSection.content + '\n\n' + content);
    }
  } else if (operation === 'insert' && section_id) {
    await updateSectionContent(section_id, $activeDocument.id, content);
  }

  const updated = await getDocument($activeDocument.id);
  activeDocument.set(updated);
  pendingPatchIntent = null;
}
```

**Section highlighting (from v1.2):**

```typescript
// src/lib/stores/documentHighlight.ts
export interface SectionHighlight {
    sectionId: string;
    reason: 'agent-reference' | 'patch-target' | 'search-result';
    expiresAt: number;
}
export const highlightedSections = writable<SectionHighlight[]>([]);

export function addHighlight(sectionId: string, reason: SectionHighlight['reason']) {
    const expiresAt = Date.now() + 5000;
    highlightedSections.update(hs => [
        ...hs.filter(h => h.sectionId !== sectionId),
        { sectionId, reason, expiresAt }
    ]);
    setTimeout(() => {
        highlightedSections.update(hs => hs.filter(h => h.sectionId !== sectionId));
    }, 5200);
}
```

```css
.section-block.highlighted {
  background-color: color-mix(in oklch, var(--color-brand-500) 8%, transparent);
  border-left: 3px solid var(--color-brand-500);
  animation: highlight-pulse 0.4s ease-out;
}
.section-block.patch-target {
  background-color: color-mix(in oklch, var(--color-amber-400) 12%, transparent);
  border-left-color: var(--color-amber-400);
}
```

**Memory citations in chat (from v1.2):**

```svelte
{#if message.citedMemoryIds?.length}
  <div class="memory-citations" role="note" aria-label="Sources from memory">
    <span class="citations-label">From your memory</span>
    {#each message.citedMemoryIds as memoryId}
      <button class="citation-chip" on:click={() => openMemoryDetail(memoryId)}>
        <span class="citation-dot" style="background: {typeColor(memory.type)}"></span>
        {memory.title ?? memory.content.slice(0, 40) + '...'}
      </button>
    {/each}
  </div>
{/if}
```

**Agent context bar (always visible in chat header):**

```svelte
<div class="agent-context-bar">
  <span class="context-item"><span class="context-icon">⚡</span>{$agentStore.provider}</span>
  {#if currentDocumentId}
    <span class="context-item"><span class="context-icon">📄</span>{documentTitle}</span>
  {/if}
  <span class="context-item"><span class="context-icon">🌿</span>{$memoryCount} memories</span>
  {#if persona}
    <span class="context-item persona-badge">{persona.name}</span>
  {/if}
</div>
```

---

### 3.4 Feature 3: Typography System

**Purpose:** Replace system font stack with scholarly typeface pair. Single highest-leverage design investment.

**Source:** UIUX.md Theme 5 (R3)

**Selected:** Inter (UI) + JetBrains Mono (code). Self-hosted via `@fontsource` (offline-capable).

```bash
pnpm add @fontsource-variable/inter @fontsource/jetbrains-mono
```

```css
@import '@fontsource-variable/inter';
@import '@fontsource/jetbrains-mono/400.css';
@import '@fontsource/jetbrains-mono/700.css';

@theme {
  --font-sans: 'InterVariable', 'Inter', system-ui, sans-serif;
  --font-mono: 'JetBrains Mono', 'Fira Code', monospace;

  --text-xs:   0.6875rem;  /* 11px — captions, timestamps */
  --text-sm:   0.8125rem;  /* 13px — secondary body, labels */
  --text-base: 0.9375rem;  /* 15px — primary body, editor */
  --text-lg:   1.0625rem;  /* 17px — section headings */
  --text-xl:   1.25rem;    /* 20px — page headings */
  --text-2xl:  1.5rem;     /* 24px — modal titles */

  --leading-tight:  1.3;
  --leading-normal: 1.6;
  --leading-relaxed: 1.8;
}

body {
  font-family: var(--font-sans);
  font-size: var(--text-base);
  line-height: var(--leading-normal);
  font-feature-settings: 'kern' 1, 'liga' 1, 'calt' 1;
  -webkit-font-smoothing: antialiased;
}

.prose, .document-editor textarea {
  line-height: var(--leading-relaxed);
}

code, pre, .mono { font-family: var(--font-mono); font-size: 0.875em; }

h1 { font-size: var(--text-2xl); font-weight: 600; letter-spacing: -0.01em; }
h2 { font-size: var(--text-xl);  font-weight: 600; letter-spacing: -0.01em; }
h3 { font-size: var(--text-lg);  font-weight: 500; }

.dark body { font-weight: 400; -webkit-font-smoothing: subpixel-antialiased; }
```

---

### 3.5 Feature 4: Online/Offline Mode Indicator

**Purpose:** Make capability boundaries explicit. No silent failures.

**Source:** UIUX.md Theme 3 (R2)

```svelte
<!-- GatewayStatusBar.svelte — sidebar header -->
<div class="gateway-status {config.class}" role="status" aria-live="polite">
  <span class="status-dot">{config.icon}</span>
  <span class="status-label">{config.label}</span>
  {#if $gatewayStatus === 'down'}
    <button class="status-help" on:click={showOfflineHelp}>What can I do?</button>
  {/if}
</div>
```

**Feature guard pattern — applied to all gateway-requiring UI:**

```css
.offline {
  cursor: not-allowed;
  opacity: 0.5;
  background: repeating-linear-gradient(45deg, transparent, transparent 4px,
    var(--color-gray-100) 4px, var(--color-gray-100) 8px);
}
```

```css
@theme {
  --color-status-online: #10b981;
  --color-status-offline: #6b7280;
  --color-status-reconnecting: #f59e0b;
  --color-status-error: #ef4444;
}
```

---

### 3.6 Feature 5: 3-Tier Sidebar Navigation

**Purpose:** Replace flat route list with primary/secondary/utility hierarchy.

**Source:** UIUX.md Theme 4 (R5)

```
PRIMARY (bold, prominent)
  └── Workspace

SECONDARY (standard weight, slight indent)
  ├── Chat
  └── Memory

UTILITY (collapsible, smaller, divider above)
  ├── Search
  ├── Tools
  ├── Pipelines
  └── Settings
```

```css
.nav-primary .nav-item { font-size: var(--text-base); font-weight: 600; }
.nav-secondary .nav-item { font-size: var(--text-sm); font-weight: 500; padding-left: 1.25rem; }
.nav-utility .nav-item { font-size: var(--text-xs); font-weight: 400; color: var(--color-muted-foreground); }
.nav-utility { border-top: 1px solid var(--color-border); margin-top: 0.5rem; }
```

---

### 3.7 Feature 6: Empty States & Loading Skeletons

**Purpose:** Design first impressions and error states.

**Source:** UIUX.md Theme 5 (R6)

**EmptyState.svelte — 5 variants:**

| Variant | Icon | Title | Message |
|---------|------|-------|---------|
| `no-documents` | 📄 | Start your first document | Your workspace is ready. Create a document to begin writing. |
| `no-results` | 🔍 | No results found | Try adjusting your search terms or browse your documents directly. |
| `no-memories` | 🌱 | Your memory garden is empty | Save key insights from your agent conversations to build a knowledge base. |
| `agent-offline` | ◌ | Agent is offline | The research assistant is unavailable. Start the Gateway to continue. |
| `no-output` | ⚙ | No output yet | Ask your agent to convert this document or select a format above. |

**Skeleton.svelte — shimmer animation, 3 variants:** `document-list`, `memory-panel`, `message`

```css
.skeleton-line {
  background: linear-gradient(90deg, var(--color-gray-200) 25%, var(--color-gray-100) 50%, var(--color-gray-200) 75%);
  background-size: 200% 100%;
  animation: shimmer 1.5s infinite;
}
@keyframes shimmer { 0% { background-position: 200% 0; } 100% { background-position: -200% 0; } }
```

---

### 3.8 Feature 7: Workspace Document View Completion

**Purpose:** Complete `/workspace/[docId]` with mode transitions, version history, section ownership.

**Source:** v1.3 Workstream A

**New UI elements in `/workspace/[docId]/+page.svelte`:**

```svelte
<!-- Mode transition button (draft -> structured) -->
{#if isOwner && doc.mode === 'draft' && doc.sections.length >= 2}
  <button onclick={handleTransitionToStructured} class="btn-primary text-xs px-3 py-1">
    Go Structured
  </button>
{/if}

<!-- Version history toggle -->
<button onclick={() => showVersionHistory = !showVersionHistory} class="btn-secondary text-xs px-3 py-1">
  History
</button>
```

**New component: `VersionHistory.svelte` — sidebar showing saved versions with timestamps, diff view, restore button.**

**New Tauri commands:**
```rust
#[tauri::command]
pub async fn list_document_versions(doc_id: String, ...) -> Result<Vec<DocumentVersion>, AppError>

#[tauri::command]
pub async fn get_document_version(version_id: String, ...) -> Result<DocumentVersion, AppError>
```

**New types:**
```typescript
export interface DocumentVersion {
  id: string;
  document_id: string;
  content: string;
  message: string;
  created_at: string;
}
```

**Document margin memory indicators (from v1.2):**

```svelte
{#if sectionMemories.length > 0}
  <button class="gutter-indicator" on:click={() => showSectionMemories(sectionId)}>
    <span>🌿</span>
    {#if sectionMemories.length > 1}
      <span class="gutter-count">{sectionMemories.length}</span>
    {/if}
  </button>
{/if}
```

---

### 3.9 Feature 8: MCP App Companions (Unified)

**Purpose:** Wire `/app/[appId]` to gateway lifecycle API with designed entry/exit transitions.

**Source:** v1.3 Workstream B (gateway wiring) + v1.2 Feature 8 (transition design)

This is where the two specs overlap most. v1.3.2 takes the **gateway lifecycle wiring from v1.3** and wraps it in the **AppShell transition design from v1.2**:

```svelte
<!-- AppShell.svelte — unified: gateway launch + transition UX -->
<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import { page } from '$app/stores';
  import { goto } from '$app/navigation';
  import { launchApp, closeApp } from '$lib/api/tauri.js';
  import { fade } from 'svelte/transition';

  let appId = $derived($page.params.appId);
  let instanceId = $state<string | null>(null);
  let appUrl = $state('');
  let loading = $state(true);
  let exitTarget = $page.url.searchParams.get('from') ?? '/workspace';

  function exitApp() { goto(exitTarget); }
  function handleKeydown(e: KeyboardEvent) { if (e.key === 'Escape') exitApp(); }

  onMount(async () => {
    try {
      const result = await launchApp(appId!, 'local');
      instanceId = result.instance_id;
      appUrl = `http://127.0.0.1:8080/v1/gateway/resources?uri=ui://${appId}/preview.html`;
    } catch (err) {
      error = `App not available: ${err}`;
    } finally {
      loading = false;
    }
  });

  onDestroy(async () => {
    if (instanceId) await closeApp(instanceId).catch(() => {});
  });
</script>

<svelte:window on:keydown={handleKeydown} />

<div class="app-shell" transition:fade={{ duration: 150 }}>
  <header class="app-shell-header">
    <button class="exit-btn" on:click={exitApp}>← Back to workspace</button>
    <span class="app-title">{appMeta[appId]?.label ?? appId}</span>
  </header>
  <main class="app-iframe-container">
    {#if loading}
      <Skeleton variant="message" count={3} />
    {:else if error}
      <EmptyState variant="no-output" />
    {:else}
      <iframe src={appUrl} title="{appId} output" class="app-iframe"
        sandbox="allow-scripts allow-same-origin" />
    {/if}
  </main>
</div>
```

**Design token injection via postMessage:**

```typescript
// appBridge.ts — injected into MCP app iframes
window.addEventListener('message', (e) => {
    if (e.data.type === 'zen-sci:theme-tokens') {
        const { brandColor, fontSans, fontMono, dark } = e.data.tokens;
        document.documentElement.style.setProperty('--brand', brandColor);
        document.documentElement.style.setProperty('--font-sans', fontSans);
        if (dark) document.documentElement.classList.add('dark');
    }
});
```

**New Tauri commands:**
```rust
#[tauri::command]
pub async fn launch_app(app_id: String, session_id: String, ...) -> Result<AppInstance, AppError>
#[tauri::command]
pub async fn close_app(instance_id: String, ...) -> Result<(), AppError>
#[tauri::command]
pub async fn list_apps(session_id: String, ...) -> Result<Vec<AppInstance>, AppError>
```

---

### 3.10 Feature 9: Deep Search (Unified)

**Purpose:** Connect search to both full-text (Tantivy) and semantic memory (gateway), make results clickable.

**Source:** v1.3 Workstream C

```svelte
<!-- Search mode toggle -->
<div class="flex gap-2 mb-3">
  <button onclick={() => searchMode = 'documents'}
    class={searchMode === 'documents' ? 'btn-primary' : 'btn-secondary'}>Documents</button>
  <button onclick={() => searchMode = 'memories'}
    class={searchMode === 'memories' ? 'btn-primary' : 'btn-secondary'}>Memories</button>
  <button onclick={() => searchMode = 'all'}
    class={searchMode === 'all' ? 'btn-primary' : 'btn-secondary'}>All</button>
</div>
```

```typescript
async function executeSearch(query: string, mode: SearchMode) {
  const results: UnifiedResult[] = [];
  if (mode === 'documents' || mode === 'all') {
    const docResults = await search(query);
    results.push(...docResults.results.map(r => ({
      ...r, source: 'document' as const, href: `/workspace/${r.document_id}`
    })));
  }
  if (mode === 'memories' || mode === 'all') {
    const memResults = await searchMemory(query, projectId);
    results.push(...memResults.map(r => ({
      id: r.id, title: r.content.slice(0, 80), snippet: r.content,
      source: 'memory' as const, relevance_score: r.relevance_score,
      href: `/memory?highlight=${r.id}`
    })));
  }
  return results.sort((a, b) => (b.relevance_score ?? 0) - (a.relevance_score ?? 0));
}
```

**ResultsList.svelte — clickable results with source badges:**

```svelte
<a href={result.href} class="block hover:bg-gray-50 dark:hover:bg-gray-800">
  <!-- existing result rendering -->
  <span class="text-xs px-1.5 py-0.5 rounded {
    result.source === 'memory' ? 'bg-purple-100 text-purple-700' : 'bg-blue-100 text-blue-700'
  }">{result.source}</span>
</a>
```

---

### 3.11 Feature 10: Settings Surface Completion

**Purpose:** Expose MCP server status, detailed health, and garden stats.

**Source:** v1.3 Workstream D

**New sections in `/settings/+page.svelte`:**

| Section | Endpoint | Content |
|---------|----------|---------|
| MCP Servers | `GET /admin/mcp/servers` | Server name, connection dot (green/red), tool count |
| System Health | `GET /admin/health` | Active agents, total tools, memory (MB), goroutines |
| Garden Stats | `GET /v1/garden/stats` | Total memories, types breakdown |

**New Tauri commands:**
```rust
#[tauri::command]
pub async fn get_mcp_servers(...) -> Result<McpServersResponse, AppError>
#[tauri::command]
pub async fn get_detailed_health(...) -> Result<DetailedHealth, AppError>
```

**New types:**
```typescript
export interface McpServer {
  server_id: string; display_name: string;
  state: 'connected' | 'disconnected' | 'error';
  tool_count: number; last_error: string;
}
export interface DetailedHealth {
  system: { go_version: string; goroutines: number; num_cpu: number };
  memory: { alloc_mb: number; total_alloc_mb: number; sys_mb: number };
  tools: { total: number; namespaces: string[] };
  agents: { active_count: number };
  mcp: { enabled: boolean; server_count: number; total_tools: number };
}
```

---

### 3.12 Feature 11: Connected ToolInvoker

**Purpose:** Make ToolInvoker context-aware (auto-fill from document) and store results to memory.

**Source:** v1.3 Workstream F

```svelte
<!-- Auto-fill from document -->
{#if documentContent && selectedTool?.parameters?.some(p => p.name === 'content')}
  <div class="bg-blue-50 dark:bg-blue-900/20 rounded p-2 mb-3">
    <label class="flex items-center gap-2 text-xs">
      <input type="checkbox" bind:checked={useDocumentContent} />
      Use current document as input
    </label>
  </div>
{/if}

<!-- Save result to memory -->
{#if result}
  <div class="flex gap-2 mt-2">
    <button onclick={handleCopyResult} class="btn-secondary text-xs">Copy</button>
    <button onclick={handleSaveToMemory} class="btn-secondary text-xs">Save to Memory</button>
  </div>
{/if}
```

---

## 4. Implementation Plan

### 4.1 Phased Approach

**Timeline:** 4 weeks (Week 1–2: foundation + experience; Week 3: surface completion; Week 4: integration + polish)

| Phase | Duration | Focus | Deliverables |
|-------|----------|-------|--------------|
| 1 | Week 1 | **Foundation** | SSE streaming (Go + Rust + Svelte), typography system, online/offline indicator |
| 2 | Week 2 | **Spatial Coherence** | Agent personas, section highlighting, PatchDiff + apply logic, IA hierarchy, empty states, skeletons |
| 3 | Week 3 | **Surface Completion** | Deep search, settings surface, workspace version history + mode transition, MCP app lifecycle, connected ToolInvoker |
| 4 | Week 4 | **Integration & Polish** | Memory ambient layer (citations, gutter, context bar), MCP app transitions + token injection, dark mode audit, accessibility pass, full regression |

### 4.2 Week-by-Week Breakdown

**Week 1: Foundation (Trust & Identity)**

- [ ] Add `GET /v1/gateway/agent/:id/stream` SSE endpoint in Go gateway
- [ ] Implement `StreamingProvider` for Anthropic, OpenAI, Ollama
- [ ] Implement `chat_with_agent_stream` Tauri IPC command (Rust)
- [ ] Add `listen('agent-stream-chunk')` in AgentChat.svelte; render streaming message with cursor
- [ ] Add tool call badge during streaming
- [ ] Install `@fontsource-variable/inter` and `@fontsource/jetbrains-mono`
- [ ] Apply typography scale to `app.css`; replace all `font-sans: system-ui` references
- [ ] Audit all components for font-size usage; normalize to 6-step type scale
- [ ] Build `GatewayStatusBar.svelte` (replaces status-pill in sidebar)
- [ ] Build `OfflineCapabilitySheet.svelte`
- [ ] Apply feature guard pattern (disabled + hatched background) to gateway-requiring UI

**Success Criteria:** Agent chat streams text. Inter font applied. Offline indicator prominent.

**Week 2: Spatial Coherence (Agent + Navigation)**

- [ ] Create `src/lib/agent/personas.ts` with 6 format-specific personas
- [ ] Add `documentFormat` prop to AgentChat; display persona name in context bar
- [ ] Add `highlightedSections` store; connect to agent response parsing
- [ ] Implement section highlight animation in DocumentEditor.svelte
- [ ] Build `PatchDiff.svelte` (diff overlay for patch-by-intent proposals)
- [ ] Implement `applyPatch()` — writes to document store via `updateSectionContent`
- [ ] Build `AgentContextBar.svelte` (provider, document, memory count, persona)
- [ ] Restructure `Sidebar.svelte` into 3-tier IA
- [ ] Build `EmptyState.svelte` with 5 variants
- [ ] Apply empty states to all 6 routes
- [ ] Build `Skeleton.svelte` with 3 variants; apply to document list, memory panel, chat

**Success Criteria:** Agent specializes by format. Section highlighting works. Patch intent renders as diff. Sidebar has hierarchy. All routes have empty states.

**Week 3: Surface Completion (Backend Wiring)**

- [ ] Add `UnifiedSearchResult` type; search mode toggle (documents/memories/all)
- [ ] Wire `searchMemory()` into `/search`; make results clickable with source badges
- [ ] Add `getMcpServers()` and `getDetailedHealth()` Tauri commands
- [ ] Add MCP Servers + System Health cards to `/settings`
- [ ] Add "Go Structured" button to `/workspace/[docId]`
- [ ] Add `listDocumentVersions()` and `getDocumentVersion()` Tauri commands
- [ ] Build `VersionHistory.svelte` component
- [ ] Add `launchApp()`, `closeApp()`, `listApps()` Tauri commands
- [ ] Build unified `AppShell.svelte` (gateway launch + transition UX)
- [ ] Verify `MCP_APPS_ENABLED` env var in sidecar config
- [ ] Add `documentContent` prop to ToolInvoker; auto-fill checkbox; "Save to Memory" button

**Success Criteria:** All 13 new endpoints wired. Search returns document + memory results. Settings shows MCP servers. Version history works. App lifecycle works.

**Week 4: Integration & Polish**

- [ ] Extend gateway response type to include `cited_memory_ids[]`
- [ ] Build `MemoryCitation.svelte` (inline chips below agent messages)
- [ ] Add `SectionGutter.svelte` with memory indicator dots
- [ ] Add `appBridge.ts` postMessage handler for design token injection
- [ ] Polish pass: audit all spacing for consistency with type scale
- [ ] Dark mode audit: verify all 12 new/changed components work in dark mode
- [ ] Accessibility pass: ARIA labels, keyboard navigation, focus management
- [ ] Full regression: `cargo test`, `npm run check`, `go test ./...`
- [ ] Update TESTING-RUN.md with v1.3.2 test cases
- [ ] Update STATUS.md and CONTEXT.md

**Success Criteria:** Memory citations appear. Dark mode clean. Accessibility clean. Zero regressions. Docs updated.

### 4.3 Dependencies & Prerequisites

**Required Before Starting:**
- ✅ v1.1 durability patch shipped (agent ID persists, chat history re-hydrates) — closed 2026-02-20
- ✅ Gateway v1.1.0 healthy (`/health` returns `{"status":"healthy","version":"1.1.0"}`)
- ✅ `chatWithAgent` IPC command working (non-streaming fallback)
- ✅ Memory panel implemented (MemoryPanel.svelte, `GET /v1/memory` endpoint)
- ✅ Phase 4 MCP apps built and accessible at `ui://` URIs
- ✅ All 6 MCP servers connected, 17 zen_sci tools registered
- ✅ `MCP_APPS_ENABLED=true` env var set for MCP app workstream

**Parallel Work:**
- Week 1 tasks (streaming, typography, offline) are fully independent of each other
- Week 2 tasks (personas, highlighting, empty states) are independent of each other
- Week 3 workstreams (search, settings, workspace, apps, tools) are fully independent
- Week 4 memory citations require gateway response type change first

**Blocking Dependencies:**
- SSE streaming Go endpoint must be built before Rust bridge and Svelte consumer
- MCP app design token injection requires AppShell to exist first
- Memory citations require `cited_memory_ids[]` in gateway response (Week 4 Go change or client-side inference)
- Version history requires verifying `document_versions` table exists in SQLite schema

### 4.4 Testing Strategy

**Unit Tests:**
- `personas.ts` — 6 formats have valid personas
- `highlightedSections` store — add, expire, clear behavior
- `EmptyState.svelte` — all 5 variants render
- `Skeleton.svelte` — all 3 variants render
- `PatchDiff.svelte` — renders with/without originalContent, apply/dismiss
- `UnifiedSearchResult` type — both source types construct

**Integration Tests (Rust):**
- `chat_with_agent_stream` IPC: events emitted for each SSE chunk type
- `apply_patch_intent` IPC: document content updated, sidecar written
- `test_mcp_servers_endpoint` — `/admin/mcp/servers` deserialization
- `test_detailed_health_endpoint` — `/admin/health` deserialization
- `test_launch_app_endpoint` — `/v1/gateway/apps/launch` deserialization

**Integration Tests (Go):**
- SSE endpoint: streaming works for Anthropic, OpenAI, Ollama providers
- `StreamChunk` serialization matches frontend parse expectations

**E2E Tests (Playwright):**
- Send message → streaming text within 3s → finalizes on "done"
- Gateway offline → indicator → disabled elements
- Agent references section → section highlighted within 1s
- Navigate to `/app/latex` → AppShell renders → Escape returns to workspace
- Search query → results from both sources → click navigates
- Settings → MCP servers green → health metrics render

**Performance Tests:**
- First streaming token: ≤2s (Anthropic/OpenAI), ≤5s (Ollama)
- Inter font FOUT: ≤100ms with self-hosted fontsource
- Section highlight animation: 60fps

**Manual QA:**
- Dark mode: all 12 new/changed components
- Keyboard: tab order through sidebar tiers, app shell, patch diff
- Screen reader: ARIA labels on status indicators, empty states, citations

---

## 5. Risk Assessment & Mitigation

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Ollama streaming latency >5s | Medium | Medium | Timeout indicator after 5s; "Generating..." pulse; non-streaming fallback |
| SSE connection drops mid-stream | Medium | Medium | Reconnect on `error`; fall back to `chatWithAgent` (non-streaming) |
| `document_versions` table doesn't exist | Medium | Medium | Check migration files Day 1 of Week 3; add migration in Rust if missing |
| MCP Apps gateway endpoints not working | Medium | Low | Test with curl before Week 3; defer to v1.4 if broken |
| Inter font FOUT on startup | Low | Low | Preload in `<head>` with `font-display: swap` |
| Section ID mismatch editor vs. agent | Medium | High | Gate highlight on verified section_id from store; fail silently |
| Patch-by-intent silent corruption | Low | Critical | Never auto-apply; always require user click |
| MCP app iframe CSP violation | Medium | Medium | Test each Phase 4 app with `sandbox="allow-scripts allow-same-origin"` |
| `exactOptionalPropertyTypes` TS errors on new types | High | Low | Apply `?? null` / spread pattern from v1.2 fixes |
| Patch intent parsing differs streaming vs. non-streaming | Low | Medium | Test both paths; streaming emits dedicated event |
| Memory citation IDs stale (deleted) | Low | Low | Hide chip if memory_id not found; no error |
| Typography regression in dark mode | Medium | Medium | Visual regression in Playwright before merge |

---

## 6. Rollback & Contingency

**Feature Flags (Svelte store-based, toggled in Settings):**
- `streaming_enabled` — default `true`. Falls back to non-streaming `chatWithAgent`.
- `typography_v2` — default `true`. Reverts to system font stack via CSS variable swap.
- `section_highlighting` — default `true`. No-op if `false`.

**Workstream Isolation:**
Each of the 12 features is self-contained. If any is blocked, the others ship independently:

| Feature | Rollback |
|---------|----------|
| Streaming | Flag `streaming_enabled = false` → non-streaming IPC |
| Typography | Flag `typography_v2 = false` → system fonts |
| Offline indicator | Revert GatewayStatusBar → status-pill |
| Sidebar IA | Revert Sidebar.svelte → flat navigation |
| Empty states | Additive; no prior behavior to restore |
| Agent personas | Remove import; fallback to generic "Research Agent" |
| PatchDiff + apply | Revert to amber text card (v1.1 behavior) |
| Section highlighting | Flag off; no visual change |
| Deep search | Remove toggle; results remain non-clickable |
| Settings surface | Remove new cards; still shows gateway + providers |
| Workspace modes/versions | Additive buttons; remove if broken |
| MCP app lifecycle | Revert to `ui://` direct iframe (current) |
| ToolInvoker context | Remove `documentContent` prop; standalone mode |
| Memory citations | Hide citations; no breaking change |

**Monitoring:**
- SSE `error` event rate: alert if >10% of sessions
- `chatWithAgentStream` latency: alert if >10s before first chunk
- `cargo test`: 14 existing + 5 new integration = 19+ must pass
- `npm run check`: 0 errors (8 pre-existing warnings OK)
- `go test ./...`: 10 packages pass

---

## 7. Documentation & Communication

**User-Facing:**
- [ ] Update README: "What works offline" table
- [ ] Add keyboard shortcut reference: Cmd+K (search), Escape (exit app), Shift+Enter (newline)
- [ ] Update TESTING-RUN.md with v1.3.2 test cases

**Developer Documentation:**
- [ ] Document SSE `StreamChunk` type in ARCHITECTURE.md
- [ ] Document persona system in `src/lib/agent/personas.ts` header
- [ ] Document postMessage token injection protocol in Phase 4 MCP Apps section
- [ ] Add `PatchDiff`, `EmptyState`, `VersionHistory` to component README
- [ ] Add API endpoint mapping table (consumed vs. available) to README
- [ ] Update types/index.ts with JSDoc on all new interfaces

**Release Notes:**
```
v1.3.2 — The Scholar's Workshop

Streaming:
- Real-time agent responses — text appears as the agent thinks
- Tool call badges during streaming (⚙ running/done/failed)

Visual Identity:
- Inter + JetBrains Mono typography across all surfaces
- 6-step type scale (11px–24px) with scholarly spacing
- Dark mode refinements

Agent Intelligence:
- Agent personalizes by document format (LaTeX, Blog, Grant, Slides, Newsletter, Paper)
- Patch-by-intent proposals appear as visual diffs — Apply or Dismiss
- Section highlighting when agent references your document
- Memory citations show which saved insights the agent used

Surface Completion:
- Search queries documents AND memories with source badges
- Search results are clickable — navigate directly
- Settings shows MCP server status and system health metrics
- Documents support draft → structured mode transition
- Version history sidebar with saved versions
- ToolInvoker auto-fills from active document
- Tool results save to memory with one click
- MCP app lifecycle (launch/close) via gateway

Navigation & Status:
- 3-tier sidebar (Workspace → Chat/Memory → Utilities)
- Online/offline indicator with capability explanation
- Empty states for all major surfaces
- Loading skeletons for document list, memory panel, chat

Technical:
- 13 additional gateway endpoints consumed (9 → 22)
- 1 new Go endpoint (SSE streaming)
- 6 new Tauri command wrappers
- 12 components new or enhanced
```

---

## 8. Appendices

### 8.1 Full Endpoint Consumption Map (v1.3.2)

| # | Endpoint | v1.1 | v1.3.2 |
|---|----------|------|--------|
| 1 | `POST /auth/login` | ✅ | ✅ |
| 2 | `POST /auth/register` | ✅ | ✅ |
| 3 | `POST /auth/refresh` | ✅ | ✅ |
| 4 | `POST /v1/gateway/agents` | ✅ | ✅ |
| 5 | `GET /v1/gateway/agents/:id` | ✅ | ✅ |
| 6 | `POST /v1/gateway/agents/:id/chat` | ✅ | ✅ |
| 7 | `GET /v1/gateway/agent/:id/stream` | — | ✅ NEW (Go) |
| 8 | `GET /v1/gateway/tools` | ✅ | ✅ |
| 9 | `POST /v1/tools/:name/invoke` | ✅ | ✅ |
| 10 | `GET /v1/providers` | ✅ | ✅ |
| 11 | `POST /v1/memory` | ✅ | ✅ |
| 12 | `GET /v1/memory` | ✅ | ✅ |
| 13 | `PUT /v1/memory/:id` | ✅ | ✅ |
| 14 | `POST /v1/memory/search` | ✅ | ✅ (+ search page) |
| 15 | `POST /v1/gateway/orchestrate` | ✅ | ✅ |
| 16 | `GET /v1/gateway/orchestrate/:id/dag` | ✅ | ✅ |
| 17 | `GET /admin/mcp/servers` | — | ✅ NEW |
| 18 | `GET /admin/health` | — | ✅ NEW |
| 19 | `POST /v1/settings/providers` | — | ✅ NEW |
| 20 | `GET /v1/settings/providers` | — | ✅ NEW |
| 21 | `POST /v1/gateway/apps/launch` | — | ✅ NEW |
| 22 | `POST /v1/gateway/apps/close` | — | ✅ NEW |
| 23 | `GET /v1/gateway/apps` | — | ✅ NEW |
| 24 | `GET /v1/gateway/documents/:id` | — | ✅ NEW |
| 25 | `DELETE /v1/memory/:id` | — | ✅ NEW |
| 26 | `GET /v1/memory/:id` | — | ✅ NEW |
| 27 | `GET /v1/garden/stats` | — | ✅ NEW |

**Total: 9 → 27 endpoints consumed (47% of 57).**

### 8.2 Files Changed (Predicted — 19 modified, 6 created)

**Modified (19):**
1. `gateway/handlers/stream.go` — NEW SSE endpoint (only Go change)
2. `gateway/providers/anthropic.go` — `StreamCompletion` method
3. `gateway/providers/openai.go` — `StreamCompletion` method
4. `gateway/providers/ollama.go` — `StreamCompletion` method
5. `src-tauri/src/commands/agent.rs` — `chat_with_agent_stream`
6. `src-tauri/src/commands/gateway.rs` — MCP, health, app commands
7. `src-tauri/src/commands/document.rs` — version list/get
8. `src-tauri/src/commands/mod.rs` — register new commands
9. `src-tauri/src/lib.rs` — register in invoke_handler
10. `src/lib/api/tauri.ts` — new command wrappers
11. `src/lib/types/index.ts` — new interfaces
12. `src/routes/workspace/[docId]/+page.svelte` — mode transition, version toggle, agent format prop
13. `src/routes/app/[appId]/+page.svelte` — unified AppShell
14. `src/routes/search/+page.svelte` — search mode toggle, memory integration
15. `src/routes/settings/+page.svelte` — MCP servers, health, garden cards
16. `src/lib/components/AgentChat.svelte` — streaming, persona, patch intent, citations
17. `src/lib/components/ToolInvoker.svelte` — document auto-fill, save to memory
18. `src/lib/components/ResultsList.svelte` — clickable, source badge
19. `src/app.css` — typography, skeleton keyframes, status colors, diff styles

**Created (6):**
1. `src/lib/agent/personas.ts` — 6 format-specific agent personas
2. `src/lib/stores/documentHighlight.ts` — section highlight store
3. `src/lib/components/VersionHistory.svelte` — version history sidebar
4. `src/lib/components/EmptyState.svelte` — 5-variant empty state
5. `src/lib/components/Skeleton.svelte` — 3-variant loading skeleton
6. `src/lib/components/PatchDiff.svelte` — visual diff for patch-by-intent

### 8.3 Source Spec Deduplication Log

Where v1.2 and v1.3 overlap, this spec resolved as follows:

| Area | v1.2 Design | v1.3 Design | v1.3.2 Resolution |
|------|-------------|-------------|-------------------|
| **Streaming** | Full SSE Go + Rust + Svelte | "v1.2 wired, v1.3 polish" | Takes v1.2 full implementation |
| **MCP App shell** | Transition design (fade, Escape, token injection) | Gateway lifecycle wiring (launch/close/list) | Both — gateway lifecycle inside transition shell |
| **Patch intent** | Visual diff overlay (PatchDiff.svelte) | Apply logic (writes to document store) | Both — diff UI triggers apply logic |
| **Search** | Not in scope | Unified search (documents + memories) | Takes v1.3 |
| **Settings** | Not in scope | MCP servers + health dashboard | Takes v1.3 |
| **Agent personas** | Not in scope | Format-specific personas | Takes v1.3 |
| **Typography** | Full system (Inter + JetBrains + scale) | Not in scope | Takes v1.2 |
| **Offline indicator** | Full design (StatusBar + Sheet + guards) | Not in scope | Takes v1.2 |
| **Sidebar IA** | 3-tier hierarchy | Not in scope | Takes v1.2 |
| **Empty states** | 5-variant component + skeletons | Not in scope | Takes v1.2 |
| **Memory ambient** | Citations, gutter dots, context bar | Not in scope | Takes v1.2 |
| **Section highlighting** | Full store + animation | Not in scope | Takes v1.2 |

### 8.4 v1.4 Candidates (Deferred)

- `search_memory` as registered agent tool (agent self-queries memory during reasoning loop)
- `pipeline_proposal` field (agent proposes DAG; user confirms; pipeline launches)
- Tantivy full-text search integration (offline document content search)
- Memory Garden frontend (seeds, snapshots, garden context display)
- Knowledge graph visualization (D3.js for memory relationships)
- Tracing dashboard (`/v1/gateway/traces/:id` rendered as execution timeline)
- Notification system (toasts for async completions)
- Offline queue (buffer operations when gateway down)

### 8.5 Open Questions

- [ ] Does `document_versions` table exist in current SQLite schema? Check migration files before Week 3.
- [ ] Should document format be stored on `Document` model (new field) or inferred from metadata?
- [ ] Should agent personas be configurable per-user in settings, or fixed per format?
- [ ] Which Ollama models support reliable streaming? (qwen3:8b, llama3.1, llama3.2 need testing)
- [ ] Does `cited_memory_ids[]` require a gateway change or can it be inferred client-side?
- [ ] Should utility nav (Search, Tools, Pipelines) be collapsed by default?
- [ ] Should memory gutter indicator use leaf emoji or custom SVG for production?
- [ ] For MCP Apps: are `ui://` resources actually built and serving, or do they need future work?

### 8.6 Pre-Implementation Checklist

- [ ] Confirm Gateway v1.1.0 running: `curl http://127.0.0.1:8080/health`
- [ ] Confirm `.venv` Python active for PDF generation tests
- [ ] Confirm Inter + JetBrains Mono available via `@fontsource`
- [ ] Confirm Phase 4 MCP app bundles accessible at `ui://latex`, `ui://blog`, etc.
- [ ] Confirm `MCP_APPS_ENABLED=true` env var set
- [ ] Verify `document_versions` SQLite table exists (or plan migration)
- [ ] Read `UIUX.md` for design rationale before implementation
- [ ] Read `v1.3-surface-completion-specification.md` for endpoint details

### 8.7 Related Work

| Document | Purpose |
|----------|---------|
| `claudeplugins/UIUX.md` | 14-source UX research synthesis — rationale for all experience layer decisions |
| `zen-sci-portal-v1.2-spec.md` | Original UX/UI spec (superseded, preserved for lineage) |
| `v1.3-surface-completion-specification.md` | Original surface completion spec (superseded, preserved for lineage) |
| `zen-sci-portal-v1.1-spec.md` | Foundation this release extends |
| `2026-02-20_durability_audit.md` | Three production gaps this release assumes are closed |
| `2026-02-18_scout3_zen-sci-portal.md` | AgenticGateway underutilization analysis |
| `2026-02-18_wide_desktop-native-frameworks.md` | Tauri/Rust ecosystem validation |
