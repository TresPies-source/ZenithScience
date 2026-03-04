# ZenSci Portal v1.2: The Thinking Workspace

**Author:** Look & Feel Agent (ZenithScience)
**Status:** Draft
**Created:** 2026-02-20
**Grounded In:** v1.1 durability patch (shipped 2026-02-20), UIUX.md synthesis (8 themes, 14 sources), zen-sci-portal-v1.1-spec.md, strategic scouts (desktop, web, mobile), health and durability audits

---

## 1. Vision

> Transform ZenSci from a competent developer dashboard into a scholar's workshop — calm, typographically rich, responsive, and spatially coherent — where the agent is a trusted collaborator visible at the periphery and the researcher's thinking comes forward.

**The Core Insight:**
v1.1 closed all three architectural durability gaps: the agent now knows what document is open, remembers prior conversations, and persists its identity across restarts. The infrastructure of the thinking workspace exists. What does not yet exist is the experience of it. v1.2 closes that gap — not by adding features, but by making the existing capabilities visible, trustworthy, and calm.

The primary failure mode of v1.1 is not a missing feature. It is the frozen interface: a researcher asks the agent a question, watches a bouncing dot spinner for 30–120 seconds (particularly on local Ollama models), and cannot tell whether the system is working, failed, or offline. Every other UX improvement is downstream of fixing this. Trust is the foundation.

**What Makes This Different:**
Most document tools treat UX polish as a late-stage concern — functionality first, feel later. ZenSci's user is an academic researcher: someone who spends hours in a single document, demands typographic rigor, and uses AI as a thinking partner, not a search engine. The experience requirements for this user are categorically different from a general productivity tool. v1.2 encodes those requirements as first-class engineering commitments.

The eight improvements in v1.2 are not cosmetic. Each is grounded in a specific documented tension from the strategic scouts, identified UX pain points in the codebase analysis, or production failures in the durability audit. Every design decision in this spec cites its source.

---

## 2. Goals & Success Criteria

**Primary Goals:**
1. **Responsive agent interaction** — Streaming responses eliminate the frozen-interface perception for all supported providers (Anthropic, OpenAI, Ollama).
2. **Spatial coherence** — The agent's awareness of the document is visible in the workspace; memory is ambient, not hidden.
3. **Scholarly visual identity** — Typography, spacing, and hierarchy convey calm, focused, scholarly work.
4. **Honest capability communication** — Offline vs. online modes are always explicit; no silent failures.

**Success Criteria:**
- ✅ First streaming token appears in ≤2 seconds for Anthropic/OpenAI providers
- ✅ Agent responses render incrementally — no blank wait periods >3 seconds for any provider
- ✅ Offline mode is visually distinct; gateway-required features are visually disabled with explanation
- ✅ A custom typeface pair is applied across all routes; system font stack is removed
- ✅ When agent references a document section, that section is highlighted in the editor
- ✅ Patch-by-intent proposals render as visual diffs, not plain text
- ✅ Sidebar navigation has 3-tier visual hierarchy (primary/secondary/utility)
- ✅ Every major empty state has a designed illustration + message + primary action
- ✅ Agent memory citations appear in chat messages; referenced memories are linkable
- ✅ MCP app views have explicit entry/exit transitions with Escape-to-return shortcut

**Non-Goals (Out of Scope):**
- ❌ Web portal or mobile app changes (deferred to v2.0)
- ❌ Multi-user collaboration features (deferred to v2.0)
- ❌ New MCP server modules or conversion formats
- ❌ CRDT or real-time sync infrastructure
- ❌ Game format generation (mobile scope)
- ❌ Public sharing or publishing features

---

## 3. Technical Architecture

### 3.1 System Overview

v1.2 is a **frontend-only release** with one Go gateway change (SSE endpoint for streaming). All changes are in the SvelteKit + Tauri portal. No MCP server changes. No database schema changes beyond an optional `cited_memory_ids` column for chat messages.

```
Researcher
    │
    ▼
zen-sci-portal (SvelteKit 5 + Tauri v2)
    │
    ├── AgentChat (streaming via SSE)          [CHANGED]
    ├── DocumentEditor (section highlight)      [CHANGED]
    ├── MemoryPanel (ambient citations)         [CHANGED]
    ├── Sidebar (3-tier IA hierarchy)           [CHANGED]
    ├── Typography System (Inter + JetBrains)   [NEW]
    ├── EmptyState components                   [NEW]
    ├── SkeletonLoader components               [NEW]
    ├── OnlineStatus indicator                  [NEW]
    └── MCP App Shell (transition design)       [CHANGED]
    │
    ▼
AgenticGateway (Go) — adds SSE stream endpoint  [MINIMAL CHANGE]
    │
    ├── Anthropic (native streaming)
    ├── OpenAI (native streaming)
    └── Ollama (chunk streaming)
```

**Key Architectural Principle:**
All eight improvements are additive. Nothing in v1.2 requires changing SQLite schema, IPC command signatures, or MCP server APIs. The gateway SSE endpoint is the only backend change. This minimizes regression risk for a polish release.

---

### 3.2 Feature 1: Streaming Agent Responses

**Purpose:**
Eliminate the frozen interface. Show the first token within 2 seconds and render text incrementally. This is the single highest-ROI change in v1.2.

**Source:** UIUX.md Theme 1 (highest-priority recommendation R1), v1.1 spec (tagged as v1.2 scope), framework research (tokio-tungstenite WebSocket production-ready)

**Gateway Change — SSE Endpoint (Go):**

Add `GET /v1/gateway/agent/:agentId/stream` alongside the existing `POST /v1/gateway/agent/:agentId/message`. The streaming endpoint accepts the same request body and returns `text/event-stream`.

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

    // Set SSE headers
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
                Type:    "error",
                Content: err.Error(),
            }))
            flusher.Flush()
            return
        case <-r.Context().Done():
            return
        }
    }
}
```

**Provider Normalization for Streaming:**

```go
// gateway/providers/stream.go

// Each provider implements StreamingProvider
type StreamingProvider interface {
    StreamCompletion(ctx context.Context, req NormalizedRequest) (<-chan StreamChunk, error)
}

// Anthropic: Uses native SSE (claude-3-5-sonnet, claude-opus-4-6)
func (p *AnthropicProvider) StreamCompletion(ctx context.Context, req NormalizedRequest) (<-chan StreamChunk, error) {
    ch := make(chan StreamChunk, 64)
    go func() {
        defer close(ch)
        stream := p.client.Messages.Stream(ctx, anthropic.MessageStreamParams{...})
        for stream.Next() {
            event := stream.Current()
            if delta, ok := event.Delta.(anthropic.TextDelta); ok {
                ch <- StreamChunk{Type: "delta", Content: delta.Text}
            }
        }
    }()
    return ch, nil
}

// Ollama: Text chunking via line-delimited JSON
func (p *OllamaProvider) StreamCompletion(ctx context.Context, req NormalizedRequest) (<-chan StreamChunk, error) {
    ch := make(chan StreamChunk, 64)
    go func() {
        defer close(ch)
        resp, _ := p.client.Post(ctx, "/api/chat", req)
        scanner := bufio.NewScanner(resp.Body)
        for scanner.Scan() {
            var chunk struct { Message struct { Content string } `json:"message"` }
            json.Unmarshal(scanner.Bytes(), &chunk)
            if chunk.Message.Content != "" {
                ch <- StreamChunk{Type: "delta", Content: chunk.Message.Content}
            }
        }
    }()
    return ch, nil
}
```

**Tauri IPC Command (Rust):**

```rust
// src-tauri/src/commands/agent.rs

#[tauri::command]
pub async fn chat_with_agent_stream(
    agent_id: String,
    message: String,
    document_id: Option<String>,
    app_handle: tauri::AppHandle,
) -> Result<(), String> {
    let gateway_url = format!(
        "http://127.0.0.1:8080/v1/gateway/agent/{}/stream",
        agent_id
    );

    let client = reqwest::Client::new();
    let mut resp = client
        .get(&gateway_url)
        .json(&serde_json::json!({
            "message": message,
            "document_id": document_id
        }))
        .send()
        .await
        .map_err(|e| e.to_string())?;

    let mut buffer = String::new();
    while let Some(chunk) = resp.chunk().await.map_err(|e| e.to_string())? {
        let text = String::from_utf8_lossy(&chunk);
        for line in text.lines() {
            if let Some(data) = line.strip_prefix("data: ") {
                app_handle
                    .emit("agent-stream-chunk", data)
                    .map_err(|e| e.to_string())?;
            }
        }
    }
    Ok(())
}
```

**Frontend — AgentChat.svelte (streaming render):**

```typescript
// src/lib/components/AgentChat.svelte (streaming additions)

import { listen } from '@tauri-apps/api/event';
import { invoke } from '@tauri-apps/api/core';

let streamingMessage = $state<string>('');
let isStreaming = $state(false);
let streamUnlisten: (() => void) | null = null;

async function sendMessage(content: string) {
    isStreaming = true;
    streamingMessage = '';

    // Listen for stream chunks before invoking
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

function finalizeStream() {
    isStreaming = false;
    if (streamingMessage) {
        messages.push({ role: 'assistant', content: streamingMessage, timestamp: Date.now() });
        streamingMessage = '';
    }
    streamUnlisten?.();
    streamUnlisten = null;
}
```

**Streaming Message Render:**

The in-progress streaming message renders in a distinct visual state — same bubble style as completed messages, with a blinking cursor appended while streaming:

```svelte
{#if isStreaming && streamingMessage}
  <div class="message-bubble assistant streaming">
    {@html renderMarkdown(streamingMessage)}<span class="cursor" aria-hidden="true">▋</span>
  </div>
{/if}
```

```css
/* Streaming cursor pulse */
.cursor {
  animation: blink 0.8s step-end infinite;
  color: var(--color-brand-500);
  font-weight: 400;
}
@keyframes blink {
  0%, 100% { opacity: 1; }
  50% { opacity: 0; }
}

/* Streaming state: subtle left border */
.message-bubble.streaming {
  border-left: 2px solid var(--color-brand-500);
  padding-left: 0.75rem;
}
```

**Tool Call Visualization During Streaming:**

When the agent executes a tool during streaming, a compact tool call badge appears inline:

```svelte
<!-- ToolCallBadge.svelte — appears inline during streaming -->
<div class="tool-badge" role="status" aria-live="polite">
  <span class="tool-icon">⚙</span>
  <span class="tool-name">{toolName}</span>
  <span class="tool-status">{status}</span>  <!-- "running..." | "done" | "failed" -->
</div>
```

**Performance Target:** First token renders within 2 seconds for Anthropic/OpenAI. Ollama local models render first token within 5 seconds (dependent on model load time).

---

### 3.3 Feature 2: Online/Offline Mode Indicator

**Purpose:**
Make it explicit which features work offline (document editing, local search, memory browse) and which require the Gateway (agent chat, conversions, tools). Eliminate silent failures.

**Source:** UIUX.md Theme 3 (R2), scouts 2 (portal, mobile)

The existing `gatewayStatus` store tracks `up | down | restarting`. v1.2 promotes this to a first-class UI surface and connects it to feature availability.

**GatewayStatusBar.svelte — Sidebar Header:**

```svelte
<!-- Replaces the current status-pill in sidebar -->
<script lang="ts">
  import { gatewayStatus } from '$lib/stores/gateway';

  const statusConfig = {
    up: { label: 'Connected', icon: '●', class: 'status-online' },
    down: { label: 'Offline', icon: '○', class: 'status-offline' },
    restarting: { label: 'Reconnecting', icon: '◌', class: 'status-reconnecting' },
  };

  $: config = statusConfig[$gatewayStatus] ?? statusConfig.down;
</script>

<div class="gateway-status {config.class}" role="status" aria-live="polite">
  <span class="status-dot" aria-hidden="true">{config.icon}</span>
  <span class="status-label">{config.label}</span>
  {#if $gatewayStatus === 'down'}
    <button class="status-help" on:click={showOfflineHelp} aria-label="What works offline?">
      What can I do?
    </button>
  {/if}
</div>
```

**OfflineCapabilitySheet.svelte — Triggered on "What can I do?":**

```svelte
<div class="offline-sheet" role="dialog" aria-label="Offline capabilities">
  <h2>Working offline</h2>

  <section>
    <h3>Available offline</h3>
    <ul>
      <li>✓ Edit and save documents</li>
      <li>✓ Browse saved documents</li>
      <li>✓ Search document titles and content (local index)</li>
      <li>✓ Browse stored memories</li>
      <li>✓ Review conversion history</li>
    </ul>
  </section>

  <section>
    <h3>Requires Gateway (currently offline)</h3>
    <ul>
      <li>✗ Agent chat and reasoning</li>
      <li>✗ Document conversions (PDF, blog, slides…)</li>
      <li>✗ Memory search and save via agent</li>
      <li>✗ Tool execution</li>
      <li>✗ Pipeline orchestration</li>
    </ul>
  </section>

  <button on:click={retryGateway}>Retry connection</button>
</div>
```

**Feature Guard Pattern:**

All gateway-requiring UI elements use a consistent disabled pattern when offline:

```svelte
<!-- AgentChat textarea — disabled when offline -->
<textarea
  disabled={$gatewayStatus !== 'up'}
  placeholder={$gatewayStatus === 'up'
    ? 'Ask your research assistant…'
    : 'Agent unavailable — Gateway is offline'}
  class:offline={$gatewayStatus !== 'up'}
/>
```

```css
.offline {
  cursor: not-allowed;
  opacity: 0.5;
  background: repeating-linear-gradient(
    45deg,
    transparent,
    transparent 4px,
    var(--color-gray-100) 4px,
    var(--color-gray-100) 8px
  );
}
```

**CSS Variables for Status States:**

```css
@theme {
  --color-status-online: #10b981;     /* brand-500 emerald */
  --color-status-offline: #6b7280;    /* gray-500 */
  --color-status-reconnecting: #f59e0b; /* amber-400 */
  --color-status-error: #ef4444;      /* red-500 */
}
```

---

### 3.4 Feature 3: Typography System

**Purpose:**
Replace the system font stack with a selected typeface pair that communicates scholarly calm. Typography is the single highest-leverage design investment for researcher UX.

**Source:** UIUX.md Theme 5 (R3)

**Selected Typefaces:**
- **UI Font:** [Inter](https://rsms.me/inter/) — Optimized for screens, neutral, high legibility at small sizes, excellent Unicode and math symbol coverage
- **Mono Font:** [JetBrains Mono](https://www.jetbrains.com/legalnotices/jetbrains-mono/) — Designed for code reading, ligatures, excellent at 14px, open source

Both are self-hosted via `@fontsource` packages (no external CDN dependency, works offline).

**Installation:**

```bash
pnpm add @fontsource-variable/inter @fontsource/jetbrains-mono
```

**app.css — Global font imports and type scale:**

```css
/* Font imports */
@import '@fontsource-variable/inter';
@import '@fontsource/jetbrains-mono/400.css';
@import '@fontsource/jetbrains-mono/700.css';

@theme {
  /* Type scale */
  --font-sans: 'InterVariable', 'Inter', system-ui, sans-serif;
  --font-mono: 'JetBrains Mono', 'Fira Code', monospace;

  /* Scale (6 steps) */
  --text-xs:   0.6875rem;  /* 11px — captions, timestamps, badges */
  --text-sm:   0.8125rem;  /* 13px — secondary body, labels */
  --text-base: 0.9375rem;  /* 15px — primary body, editor content */
  --text-lg:   1.0625rem;  /* 17px — section headings */
  --text-xl:   1.25rem;    /* 20px — page headings */
  --text-2xl:  1.5rem;     /* 24px — modal titles, large headings */

  /* Line heights */
  --leading-tight:  1.3;
  --leading-normal: 1.6;
  --leading-relaxed: 1.8;  /* Editor content — generous for reading */

  /* Letter spacing */
  --tracking-tight: -0.01em;  /* Headings */
  --tracking-normal: 0em;
  --tracking-wide: 0.04em;    /* Labels, captions */
  --tracking-widest: 0.08em;  /* ALL CAPS badges */
}

/* Global application */
body {
  font-family: var(--font-sans);
  font-size: var(--text-base);
  line-height: var(--leading-normal);
  font-feature-settings: 'kern' 1, 'liga' 1, 'calt' 1;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

/* Editor content — relaxed leading for long reading sessions */
.prose, .document-editor textarea {
  font-size: var(--text-base);
  line-height: var(--leading-relaxed);
  font-family: var(--font-sans);
}

/* Code, IDs, conversion outputs */
code, pre, .mono {
  font-family: var(--font-mono);
  font-size: 0.875em;  /* Slightly smaller relative to surrounding text */
}

/* Headings */
h1 { font-size: var(--text-2xl); font-weight: 600; letter-spacing: var(--tracking-tight); }
h2 { font-size: var(--text-xl);  font-weight: 600; letter-spacing: var(--tracking-tight); }
h3 { font-size: var(--text-lg);  font-weight: 500; }
h4 { font-size: var(--text-base); font-weight: 500; }

/* Labels and metadata */
.label {
  font-size: var(--text-xs);
  font-weight: 500;
  letter-spacing: var(--tracking-wide);
  text-transform: uppercase;
}
```

**Tailwind v4 Config Update:**

```css
/* tailwind.css additions */
@theme {
  --default-font-family: var(--font-sans);
  --default-mono-font-family: var(--font-mono);
}
```

**Dark Mode Typography:**
In dark mode, Inter renders at slightly higher font-weight to compensate for reduced contrast. Apply via:

```css
.dark body {
  font-weight: 400;  /* Inter renders heavier on dark backgrounds */
  -webkit-font-smoothing: subpixel-antialiased;
}
```

---

### 3.5 Feature 4: Agent-Document Visual Connection

**Purpose:**
When the agent references a document section, highlight that section in the editor. When a patch-by-intent proposal arrives, render it as a visual diff, not plain text in chat.

**Source:** UIUX.md Theme 2 (R4), v1.1 spec (patch-by-intent design), portal v1.1 scout (Route C recommended)

**Section Highlighting:**

The agent's response payload already includes `section_id` references in tool call results from `get_document`. v1.2 emits these as Tauri events to the frontend.

```typescript
// src/lib/stores/documentHighlight.ts
import { writable } from 'svelte/store';

export interface SectionHighlight {
    sectionId: string;
    reason: 'agent-reference' | 'patch-target' | 'search-result';
    expiresAt: number;  // timestamp — highlights fade after 5 seconds
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

```svelte
<!-- DocumentEditor.svelte — section highlight integration -->
<script lang="ts">
  import { highlightedSections } from '$lib/stores/documentHighlight';

  // Scroll to section when highlighted by agent
  $effect(() => {
    const active = $highlightedSections.find(h => h.sectionId === sectionId);
    if (active) {
      sectionEl?.scrollIntoView({ behavior: 'smooth', block: 'nearest' });
    }
  });
</script>

<div
  class="section-block"
  class:highlighted={$highlightedSections.some(h => h.sectionId === sectionId)}
  class:patch-target={$highlightedSections.some(
    h => h.sectionId === sectionId && h.reason === 'patch-target'
  )}
  bind:this={sectionEl}
>
  <!-- section content -->
</div>
```

```css
/* Section highlight animations */
.section-block {
  transition: background-color 0.3s ease, border-left 0.2s ease;
  border-left: 3px solid transparent;
}

.section-block.highlighted {
  background-color: color-mix(in oklch, var(--color-brand-500) 8%, transparent);
  border-left-color: var(--color-brand-500);
  animation: highlight-pulse 0.4s ease-out;
}

.section-block.patch-target {
  background-color: color-mix(in oklch, var(--color-amber-400) 12%, transparent);
  border-left-color: var(--color-amber-400);
}

@keyframes highlight-pulse {
  0%   { background-color: color-mix(in oklch, var(--color-brand-500) 20%, transparent); }
  100% { background-color: color-mix(in oklch, var(--color-brand-500) 8%, transparent); }
}
```

**Patch-by-Intent Diff Overlay:**

Patch-by-intent proposals (from the agent's footer in chat) now render as visual diffs instead of markdown code blocks.

```svelte
<!-- PatchDiff.svelte — replaces plain text rendering of PATCH_INTENT -->
<script lang="ts">
  interface PatchIntent {
    operation: 'replace' | 'insert' | 'append';
    sectionId: string;
    content: string;
    description: string;
    originalContent?: string;  // fetched from document store
  }

  let { patch }: { patch: PatchIntent } = $props();

  function applyPatch() {
    // Emit highlight + apply via IPC
    addHighlight(patch.sectionId, 'patch-target');
    invoke('apply_patch_intent', { patch });
  }
</script>

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

```css
.patch-diff-card {
  border: 1px solid var(--color-border);
  border-radius: var(--radius-lg);
  overflow: hidden;
  margin: 0.75rem 0;
  font-family: var(--font-mono);
  font-size: var(--text-sm);
}
.diff-remove {
  background: color-mix(in oklch, #ef4444 8%, transparent);
  color: #991b1b;
  padding: 0.5rem 0.75rem;
}
.diff-add {
  background: color-mix(in oklch, #10b981 8%, transparent);
  color: #065f46;
  padding: 0.5rem 0.75rem;
}
.diff-marker {
  font-weight: 700;
  margin-right: 0.5rem;
  user-select: none;
}
.dark .diff-remove { color: #fca5a5; }
.dark .diff-add    { color: #6ee7b7; }
```

---

### 3.6 Feature 5: Information Architecture Hierarchy

**Purpose:**
Restructure the sidebar to communicate primary/secondary/utility hierarchy. Flat navigation creates cognitive load. As the product grows (web, mobile), IA confusion compounds.

**Source:** UIUX.md Theme 4 (R5), scout 3 (web portal navigation)

**Three-Tier Sidebar IA:**

```
PRIMARY (always visible, bold weight)
  ├── Workspace          [primary document hub]

SECONDARY (always visible, normal weight, slightly indented)
  ├── Chat               [agent companion surface]
  └── Memory             [knowledge surface]

UTILITY (collapsible group, smaller text, divider above)
  ├── Search
  ├── Tools
  ├── Pipelines
  └── Settings
```

**Sidebar.svelte — restructured navigation:**

```svelte
<nav class="sidebar" aria-label="Main navigation">
  <!-- Brand / Logo -->
  <div class="sidebar-brand">
    <span class="brand-mark">ZenSci</span>
    <GatewayStatusBar />
  </div>

  <!-- PRIMARY: Workspace (full weight, prominent) -->
  <section class="nav-tier nav-primary" aria-label="Primary">
    <NavItem href="/workspace" icon="document" label="Workspace" tier="primary" />
  </section>

  <!-- SECONDARY: Companion surfaces -->
  <section class="nav-tier nav-secondary" aria-label="Companions">
    <NavItem href="/chat" icon="chat" label="Chat" tier="secondary" />
    <NavItem href="/memory" icon="memory" label="Memory" tier="secondary" />
  </section>

  <!-- UTILITY: Tools and config (collapsible) -->
  <section class="nav-tier nav-utility" aria-label="Utilities">
    <button class="utility-toggle" aria-expanded={utilityOpen} on:click={toggleUtility}>
      <span>Tools & Settings</span>
      <span aria-hidden="true">{utilityOpen ? '▴' : '▾'}</span>
    </button>
    {#if utilityOpen}
      <div transition:slide>
        <NavItem href="/search" icon="search" label="Search" tier="utility" />
        <NavItem href="/tools" icon="tool" label="Tools" tier="utility" />
        <NavItem href="/pipelines" icon="pipeline" label="Pipelines" tier="utility" />
        <NavItem href="/settings" icon="settings" label="Settings" tier="utility" />
      </div>
    {/if}
  </section>
</nav>
```

**NavItem tier styles:**

```css
/* Primary nav items — full visual weight */
.nav-primary .nav-item {
  font-size: var(--text-base);
  font-weight: 600;
  padding: 0.625rem 0.75rem;
  border-radius: var(--radius-lg);
}

/* Secondary nav items — standard weight */
.nav-secondary .nav-item {
  font-size: var(--text-sm);
  font-weight: 500;
  padding: 0.5rem 0.75rem;
  padding-left: 1.25rem;  /* visual indent */
  border-radius: var(--radius-md);
}

/* Utility nav items — lighter, smaller */
.nav-utility .nav-item {
  font-size: var(--text-xs);
  font-weight: 400;
  padding: 0.375rem 0.75rem;
  color: var(--color-muted-foreground);
}

.nav-item.active {
  background: color-mix(in oklch, var(--color-brand-500) 12%, transparent);
  color: var(--color-brand-600);
}

/* Divider between secondary and utility */
.nav-utility {
  border-top: 1px solid var(--color-border);
  margin-top: 0.5rem;
  padding-top: 0.5rem;
}
```

---

### 3.7 Feature 6: Empty States and Loading Skeletons

**Purpose:**
Design first impressions and error states. These are where researchers form their product mental model.

**Source:** UIUX.md Theme 5 (R6)

**EmptyState.svelte — Universal Component:**

```svelte
<!-- src/lib/components/EmptyState.svelte -->
<script lang="ts">
  interface Props {
    variant: 'no-documents' | 'no-results' | 'no-memories' | 'agent-offline' | 'no-output';
    primaryAction?: { label: string; onClick: () => void };
    secondaryAction?: { label: string; href: string };
  }
  let { variant, primaryAction, secondaryAction }: Props = $props();

  const configs: Record<Props['variant'], { icon: string; title: string; message: string }> = {
    'no-documents': {
      icon: '📄',
      title: 'Start your first document',
      message: 'Your workspace is ready. Create a document to begin writing.',
    },
    'no-results': {
      icon: '🔍',
      title: 'No results found',
      message: 'Try adjusting your search terms or browse your documents directly.',
    },
    'no-memories': {
      icon: '🌱',
      title: 'Your memory garden is empty',
      message: 'Save key insights from your agent conversations to build a knowledge base.',
    },
    'agent-offline': {
      icon: '◌',
      title: 'Agent is offline',
      message: 'The research assistant is unavailable. Start the Gateway to continue.',
    },
    'no-output': {
      icon: '⚙',
      title: 'No output yet',
      message: 'Ask your agent to convert this document or select a format above.',
    },
  };

  $: config = configs[variant];
</script>

<div class="empty-state" role="status">
  <span class="empty-icon" aria-hidden="true">{config.icon}</span>
  <h3 class="empty-title">{config.title}</h3>
  <p class="empty-message">{config.message}</p>
  {#if primaryAction}
    <button class="btn-primary" on:click={primaryAction.onClick}>{primaryAction.label}</button>
  {/if}
  {#if secondaryAction}
    <a href={secondaryAction.href} class="btn-text">{secondaryAction.label}</a>
  {/if}
</div>
```

```css
.empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 0.75rem;
  padding: 3rem 1.5rem;
  text-align: center;
  color: var(--color-muted-foreground);
}
.empty-icon { font-size: 2.5rem; line-height: 1; }
.empty-title { font-size: var(--text-lg); font-weight: 600; color: var(--color-foreground); }
.empty-message { font-size: var(--text-sm); max-width: 28ch; line-height: var(--leading-relaxed); }
```

**Skeleton.svelte — Loading States:**

```svelte
<!-- src/lib/components/Skeleton.svelte -->
<script lang="ts">
  interface Props {
    variant: 'document-list' | 'memory-panel' | 'message' | 'section';
    count?: number;
  }
  let { variant, count = 3 }: Props = $props();
</script>

{#if variant === 'document-list'}
  {#each Array(count) as _}
    <div class="skeleton-row">
      <div class="skeleton-line w-2/3 h-4"></div>
      <div class="skeleton-line w-1/3 h-3 mt-2"></div>
    </div>
  {/each}

{:else if variant === 'memory-panel'}
  {#each Array(count) as _}
    <div class="skeleton-memory">
      <div class="skeleton-dot"></div>
      <div class="skeleton-line w-3/4 h-3"></div>
    </div>
  {/each}

{:else if variant === 'message'}
  <div class="skeleton-message">
    <div class="skeleton-avatar"></div>
    <div class="skeleton-bubble">
      <div class="skeleton-line w-full h-3"></div>
      <div class="skeleton-line w-4/5 h-3 mt-2"></div>
      <div class="skeleton-line w-2/3 h-3 mt-2"></div>
    </div>
  </div>
{/if}
```

```css
.skeleton-line {
  background: linear-gradient(90deg,
    var(--color-gray-200) 25%,
    var(--color-gray-100) 50%,
    var(--color-gray-200) 75%
  );
  background-size: 200% 100%;
  animation: shimmer 1.5s infinite;
  border-radius: var(--radius-sm);
}

@keyframes shimmer {
  0%   { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}

.dark .skeleton-line {
  background: linear-gradient(90deg,
    var(--color-gray-800) 25%,
    var(--color-gray-700) 50%,
    var(--color-gray-800) 75%
  );
  background-size: 200% 100%;
  animation: shimmer 1.5s infinite;
}
```

---

### 3.8 Feature 7: Memory Ambient Context

**Purpose:**
Surface memory as a living layer on top of the document and agent chat, not a hidden database at `/memory`.

**Source:** UIUX.md Theme 6 (R7), v1.1 spec (memory panel), scout 3 portal

**Memory Citation in Chat Messages:**

When the gateway injects memories as context, it now returns `cited_memory_ids[]` alongside the response. The frontend renders these as inline citations.

```typescript
// Gateway response type extension
interface AgentResponse {
  content: string;
  tool_calls?: ToolCall[];
  cited_memory_ids?: string[];  // NEW — IDs of memories used in this response
}
```

```svelte
<!-- MemoryCitation.svelte — appears below assistant messages -->
{#if message.citedMemoryIds?.length}
  <div class="memory-citations" role="note" aria-label="Sources from memory">
    <span class="citations-label">From your memory</span>
    {#each message.citedMemoryIds as memoryId}
      <button
        class="citation-chip"
        on:click={() => openMemoryDetail(memoryId)}
        aria-label="View memory {memoryId}"
      >
        <span class="citation-dot" style="background: {typeColor(memory.type)}"></span>
        {memory.title ?? memory.content.slice(0, 40) + '…'}
      </button>
    {/each}
  </div>
{/if}
```

**Memory Counter in Chat Header:**

```svelte
<!-- In AgentChat header, always visible -->
<div class="agent-context-bar">
  <span class="context-item">
    <span class="context-icon" aria-hidden="true">⚡</span>
    {$agentStore.provider}
  </span>
  {#if currentDocumentId}
    <span class="context-item">
      <span class="context-icon" aria-hidden="true">📄</span>
      {documentTitle}
    </span>
  {/if}
  <span class="context-item context-memory">
    <span class="context-icon" aria-hidden="true">🌿</span>
    {$memoryCount} memories
  </span>
</div>
```

**Document Margin Memory Indicators:**

When a document section has a saved memory tagged to it, a subtle leaf indicator appears in the gutter:

```svelte
<!-- SectionGutter.svelte — document editor left gutter -->
{#if sectionMemories.length > 0}
  <button
    class="gutter-indicator"
    on:click={() => showSectionMemories(sectionId)}
    aria-label="{sectionMemories.length} saved memories for this section"
    title="{sectionMemories.length} memories"
  >
    <span aria-hidden="true">🌿</span>
    {#if sectionMemories.length > 1}
      <span class="gutter-count">{sectionMemories.length}</span>
    {/if}
  </button>
{/if}
```

```css
.gutter-indicator {
  position: absolute;
  left: -1.75rem;
  top: 0.125rem;
  font-size: 0.75rem;
  opacity: 0.5;
  cursor: pointer;
  transition: opacity 0.15s;
  background: none;
  border: none;
}
.gutter-indicator:hover { opacity: 1; }
.gutter-count {
  font-size: 0.6rem;
  font-family: var(--font-mono);
  background: var(--color-brand-500);
  color: white;
  border-radius: 99px;
  padding: 0 3px;
  vertical-align: super;
}
```

---

### 3.9 Feature 8: MCP App Transition Experience

**Purpose:**
MCP app views (PDF viewer, blog preview, slide deck, newsletter preview) should feel like a focused mode of the workspace, not a context switch into a separate product.

**Source:** UIUX.md Theme 7 (R8), scout 2 (portal — React-in-Svelte-in-Tauri), Phase 4 MCP Apps architecture spec

**AppShell.svelte — wraps all `/app/[appId]` routes:**

```svelte
<!-- src/routes/app/[appId]/+page.svelte -->
<script lang="ts">
  import { page } from '$app/stores';
  import { goto } from '$app/navigation';
  import { fade, fly } from 'svelte/transition';

  let { appId } = $page.params;
  let exitTarget = $page.url.searchParams.get('from') ?? '/workspace';

  function exitApp() { goto(exitTarget, { replaceState: false }); }

  // Global Escape key → exit focused mode
  function handleKeydown(e: KeyboardEvent) {
    if (e.key === 'Escape') exitApp();
  }
</script>

<svelte:window on:keydown={handleKeydown} />

<div class="app-shell" transition:fade={{ duration: 150 }}>
  <!-- Focused mode header bar -->
  <header class="app-shell-header">
    <button class="exit-btn" on:click={exitApp} aria-label="Return to workspace (Escape)">
      <span aria-hidden="true">←</span>
      Back to workspace
    </button>
    <span class="app-title">{appMeta[appId]?.label ?? appId}</span>
    <div class="app-shell-actions">
      <!-- App-specific toolbar items injected via postMessage -->
    </div>
  </header>

  <!-- MCP App iframe in focused mode -->
  <main class="app-iframe-container">
    <iframe
      src="ui://{appId}"
      title="{appMeta[appId]?.label ?? appId} output"
      class="app-iframe"
      sandbox="allow-scripts allow-same-origin"
    />
  </main>
</div>
```

```css
.app-shell {
  position: fixed;
  inset: 0;
  z-index: 50;
  display: flex;
  flex-direction: column;
  background: var(--color-background);
}

.app-shell-header {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding: 0.75rem 1rem;
  border-bottom: 1px solid var(--color-border);
  background: var(--color-card);
  height: 3rem;
}

.exit-btn {
  font-size: var(--text-sm);
  color: var(--color-brand-600);
  background: none;
  border: none;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 0.375rem;
  font-weight: 500;
}
.exit-btn:hover { text-decoration: underline; }

.app-iframe-container {
  flex: 1;
  overflow: hidden;
}

.app-iframe {
  width: 100%;
  height: 100%;
  border: none;
}
```

**Entry Transition:**
When navigating to `/app/[appId]`, the workspace fades out (150ms) and the app shell fades in (150ms). The transition signals intentional mode shift.

**Shared Design Tokens in MCP Apps:**
Each Phase 4 MCP app bundle receives the design tokens via postMessage on load:

```typescript
// appBridge.ts — shared with all Phase 4 MCP app bundles
window.addEventListener('message', (e) => {
    if (e.data.type === 'zen-sci:theme-tokens') {
        const { brandColor, fontSans, fontMono, radius, dark } = e.data.tokens;
        document.documentElement.style.setProperty('--brand', brandColor);
        document.documentElement.style.setProperty('--font-sans', fontSans);
        document.documentElement.style.setProperty('--font-mono', fontMono);
        if (dark) document.documentElement.classList.add('dark');
    }
});
```

---

## 4. Implementation Plan

### 4.1 Phased Approach

**Timeline:** 3 weeks

| Phase | Duration | Focus | Deliverables |
|-------|----------|-------|--------------|
| 1 | Week 1 | Foundation | Streaming SSE, typography system, online/offline indicator |
| 2 | Week 2 | Spatial Coherence | Agent-document connection, IA hierarchy, empty states |
| 3 | Week 3 | Ambient Layer | Memory citations, app shell transition, polish pass |

### 4.2 Week-by-Week Breakdown

**Week 1: Foundation (Trust & Identity)**

- [ ] Add `GET /v1/gateway/agent/:id/stream` SSE endpoint in Go gateway
- [ ] Implement `chatWithAgentStream` Tauri IPC command (Rust)
- [ ] Add `listen('agent-stream-chunk')` in AgentChat.svelte; render streaming message
- [ ] Add streaming cursor animation (blink) and streaming message border
- [ ] Add tool call badge during streaming (visual inline indicator)
- [ ] Install `@fontsource-variable/inter` and `@fontsource/jetbrains-mono`
- [ ] Apply typography scale to `app.css` — replace all `font-sans: system-ui` references
- [ ] Audit all components for font-size usage; normalize to new type scale
- [ ] Build `GatewayStatusBar.svelte` (replaces status-pill in sidebar)
- [ ] Build `OfflineCapabilitySheet.svelte` (modal sheet listing offline vs. online features)
- [ ] Apply feature guard pattern (disabled + hatched background) to gateway-requiring UI

**Success Criteria:** Agent chat streams text in real-time. Inter font is applied. Gateway status is visually prominent.

**Week 2: Spatial Coherence (Document & Navigation)**

- [ ] Add `highlightedSections` store to document editor
- [ ] Connect agent response parsing to emit `agent-stream-chunk` events with `section_id` references
- [ ] Implement section highlight animation in `DocumentEditor.svelte`
- [ ] Build `PatchDiff.svelte` component (diff view for patch-by-intent proposals)
- [ ] Update `AgentChat.svelte` to detect PATCH_INTENT payloads and render `PatchDiff` instead of code block
- [ ] Add `apply_patch_intent` Tauri IPC command (Rust — applies patch to document store)
- [ ] Restructure `Sidebar.svelte` into 3-tier IA (primary/secondary/utility with collapsible utility section)
- [ ] Update `NavItem.svelte` with tier-specific styles
- [ ] Build `EmptyState.svelte` with 5 variants
- [ ] Apply empty states to: workspace (no documents), search (no results), memory (no items), agent (offline), output (no conversion)
- [ ] Build `Skeleton.svelte` with 3 variants; apply to document list load, memory panel load, initial chat load

**Success Criteria:** Section highlighting works end-to-end. Patch-by-intent renders as diff. Sidebar has clear hierarchy. All routes have empty states.

**Week 3: Ambient Layer (Memory & Transitions)**

- [ ] Extend gateway response type to include `cited_memory_ids[]`
- [ ] Build `MemoryCitation.svelte` (inline chips below agent messages)
- [ ] Build `AgentContextBar.svelte` (provider, document, memory count — always visible in chat header)
- [ ] Add `SectionGutter.svelte` with memory indicator dots (requires section → memory_id join query)
- [ ] Build `AppShell.svelte` for `/app/[appId]` routes (header + iframe + Escape handler)
- [ ] Implement fade transition on app shell entry/exit
- [ ] Add `appBridge.ts` postMessage handler for design token injection into MCP app iframes
- [ ] Polish pass: audit all spacing for consistency with new type scale
- [ ] Dark mode audit: verify all 8 new components work in dark mode
- [ ] Accessibility pass: verify ARIA labels, keyboard navigation, focus management for all new components

**Success Criteria:** Memory citations appear in chat. App shell transition is smooth. All new components pass dark mode and keyboard accessibility check.

### 4.3 Dependencies & Prerequisites

**Required Before Starting:**
- ✅ v1.1 durability patch shipped (agent ID persists, chat history re-hydrates) — closed 2026-02-20
- ✅ Gateway v1.1.0 healthy (`/health` returns `{"status":"healthy","version":"1.1.0"}`)
- ✅ `chatWithAgent` IPC command working (non-streaming fallback exists to roll back to)
- ✅ Memory panel implemented (MemoryPanel.svelte, `GET /v1/memory` endpoint)
- ✅ Phase 4 MCP apps built and accessible at `ui://` URIs

**Parallel Work:**
- Typography system (Week 1) can be developed in parallel with SSE streaming (no dependencies)
- Empty states and skeletons (Week 2) can be built before or after section highlighting

**Blocking Dependencies:**
- MCP app design token injection requires app shell to exist first
- Memory citations require gateway response type change (cited_memory_ids) before frontend work

### 4.4 Testing Strategy

**Unit Tests:**
- `highlightedSections` store: add, expire, clear behavior — target 100% coverage
- `EmptyState.svelte`: all 5 variants render with correct content
- `Skeleton.svelte`: all 3 variants render without error
- `PatchDiff.svelte`: renders with/without originalContent, fires apply/dismiss correctly

**Integration Tests (Tauri):**
- `chatWithAgentStream` IPC: verify events emitted for each SSE chunk type (delta, tool_start, done, error)
- `apply_patch_intent` IPC: verify document content updated and sidecar written
- Gateway SSE endpoint: verify streaming works for Anthropic, OpenAI, and Ollama providers

**E2E Tests (Playwright):**
- Researcher sends message → streaming text appears within 3 seconds → message finalizes on "done"
- Gateway goes offline → offline indicator appears → gateway-requiring elements show disabled state
- Agent references section → section highlighted in editor within 1 second
- Researcher navigates to `/app/latex` → app shell renders → Escape key returns to workspace

**Performance Tests:**
- First streaming token latency: ≤2s (Anthropic/OpenAI), ≤5s (Ollama)
- Inter font load: FOUT (flash of unstyled text) ≤100ms with self-hosted fontsource
- Section highlight animation: 60fps (no jank on low-power hardware)

**Manual QA:**
- Dark mode: all 8 new components verified in dark mode
- Keyboard navigation: Tab order logical through new sidebar tiers, app shell, patch diff
- Screen reader: ARIA labels correct for status indicators, empty states, memory citations

---

## 5. Risk Assessment & Mitigation

| Risk | Likelihood | Impact | Mitigation Strategy |
|------|------------|--------|---------------------|
| Ollama streaming latency >5s | Medium | Medium | Add SSE timeout indicator after 5s; show "Generating..." pulsing message |
| SSE connection drops mid-stream | Medium | Medium | Implement reconnect on `error` event; fall back to non-streaming `chatWithAgent` |
| Inter font FOUT on startup | Low | Low | Preload Inter in `<head>` with `<link rel="preload">`; use `font-display: swap` |
| Section ID mismatch between editor and agent | Medium | High | Gate section highlight on verified section_id from document store; fail silently if missing |
| Patch-by-intent silent corruption | Low | Critical | Apply button writes to sidecar + emits Tauri event; never auto-apply; always require user click |
| MCP app iframe CSP violation | Medium | Medium | Test each Phase 4 app with `sandbox="allow-scripts allow-same-origin"`; validate postMessage origin |
| Memory citation IDs stale (deleted memory) | Low | Low | Graceful fallback: hide citation chip if memory_id not found; no error thrown |
| Typography regression in dark mode | Medium | Medium | Run dark mode visual regression in Playwright before merge |

## 6. Rollback & Contingency

**Feature Flags (Svelte store-based, toggled in Settings):**
- `streaming_enabled`: Controls SSE streaming, default `true`. Falls back to `chatWithAgent` (non-streaming) if `false`.
- `typography_v2`: Controls Inter/JetBrains fonts. Default `true`. Reverts to system fonts if `false`.
- `section_highlighting`: Controls agent-document highlight. Default `true`. No-op if `false`.

**Rollback Procedure:**
1. Set `streaming_enabled = false` in Settings → disables SSE, uses non-streaming IPC
2. Set `typography_v2 = false` → reverts to system font stack (no rebuild required — CSS variable swap)
3. MCP app shell: reverts to direct route navigation (no transition) with single config change

**Monitoring:**
- SSE `error` event rate: Alert if >10% of stream sessions end in error
- IPC command latency: Alert if `chatWithAgentStream` takes >10s before first chunk
- Font load errors: Console errors for failed @fontsource imports (visible in Tauri devtools)

---

## 7. Documentation & Communication

**User-Facing Documentation:**
- [ ] Update README: "What works offline" section with clear table
- [ ] Add keyboard shortcut reference to Settings page: Cmd+K (search), Escape (exit app), Shift+Enter (newline in chat)
- [ ] Add onboarding tooltip for memory ambient indicators (gutter leaf icons) on first document open

**Developer Documentation:**
- [ ] Document SSE stream event format (`StreamChunk` type) in ARCHITECTURE.md
- [ ] Document postMessage token injection protocol in Phase 4 MCP Apps section
- [ ] Add `PatchDiff` and `EmptyState` to component library README with usage examples

**Release Notes for v1.2:**
- Streaming agent responses — real-time text as the agent thinks
- Online/offline clarity — always know what you can do
- Scholarly typography — Inter + JetBrains Mono across all surfaces
- Agent-document connection — see which sections your agent is reading and editing
- Memory citations — know which of your saved insights the agent used
- Focused app views — full-screen output previews with smooth transitions

---

## 8. Appendices

### 8.1 Related Work

- `UIUX.md` (claudeplugins, 2026-02-20): 14-source synthesis this spec is grounded in
- `zen-sci-portal-v1.1-spec.md` (2026-02-20): Foundation this release extends
- `2026-02-20_durability_audit.md`: Three production gaps this release assumes are closed
- `2026-02-18_scout3_zen-sci-portal.md`: AgenticGateway underutilization analysis
- `2026-02-18_wide_desktop-native-frameworks.md`: Tauri/Rust ecosystem validation

### 8.2 Future Considerations (v1.3 / v2.0 Candidates)

**v1.3 candidates (build directly on v1.2):**
- `search_memory` as registered agent tool (agent can self-query memory during reasoning loop)
- `pipeline_proposal` field (agent proposes a DAG; user confirms; pipeline launches)
- Tantivy full-text search integration (offline document content search, not just title)

**v2.0 candidates (require multi-surface sync infrastructure):**
- Web portal editing surface (requires portal-owned DB and sync model from spec-review doc)
- Real-time collaboration via tokio-tungstenite WebSocket
- Mobile consumption surfaces (reading view, adaptive quiz from ZenSci content)

### 8.3 Open Questions

- [ ] Which Ollama models support reliable streaming? (qwen3:8b, llama3.1, llama3.2 — needs testing to confirm chunk format consistency)
- [ ] Should the memory gutter indicator use leaf emoji (🌿) or a custom SVG icon for production?
- [ ] Should utility nav (Search, Tools, Pipelines) be collapsed by default or open by default?
- [ ] Does `cited_memory_ids[]` require a gateway change (Go) or can it be inferred client-side from injected context?

### 8.4 Pre-Implementation Checklist

- [ ] Confirm Gateway v1.1.0 running: `curl http://127.0.0.1:8080/health`
- [ ] Confirm `.venv` Python active for PDF generation tests
- [ ] Confirm Inter + JetBrains Mono available via `@fontsource` (no external CDN needed)
- [ ] Confirm Phase 4 MCP app bundles accessible at `ui://latex`, `ui://blog`, `ui://slides`
- [ ] Read `UIUX.md` for rationale behind each design decision before implementation
