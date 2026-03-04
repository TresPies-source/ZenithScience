# Zen-Sci Portal — Full Stack Testing Run

> Three terminals, one stack. This guide boots the complete v1.2 stack and exercises every surface.

## Prerequisites

- Ollama running with `qwen3:8b` (or any model loaded)
- Node.js 20+, Rust toolchain, Go 1.22+
- MCP servers built (the 6 `zen-sci/servers/*` directories)

---

## Terminal 1 — Gateway (Go backend on :8080)

```bash
cd ~/ZenflowProjects/AgenticGatewayByDojoGenesis

# Clear port 8080 if a previous gateway is still running
kill $(lsof -ti:8080) 2>/dev/null; sleep 1

# Build fresh binary
make build

# Set MCP servers root and start
export ZEN_SCI_SERVERS_ROOT=~/ZenflowProjects/ZenithScience/zen-sci/servers
./bin/agentic-gateway
```

**Watch for:**
- `Listening on :8080`
- `MCP: 6 servers connected, 17 tools`
- `Provider ollama healthy`

**Smoke tests (run from another pane or wait for Terminal 2):**
```bash
# Health
curl -s http://localhost:8080/health | jq .status
# → "healthy"

# MCP servers
curl -s http://localhost:8080/admin/mcp/status | jq .total_tools
# → 17

# List tools (should include all 6 namespaces)
curl -s http://localhost:8080/v1/gateway/tools | jq '.tools[].name'
```

---

## Terminal 2 — Portal Frontend (SvelteKit dev server on :1420)

```bash
cd ~/ZenflowProjects/ZenithScience/zen-sci-portal

# Install deps if needed
npm install

# Run dev server (Vite + SvelteKit)
npm run dev
```

**Watch for:**
- `VITE v6.x.x ready in XXXms`
- `Local: http://localhost:1420/`

> **Note:** The dev server only serves the frontend. Tauri commands (document CRUD, search indexing, gateway proxy) won't work without the Tauri shell. Use Terminal 3 for the full experience.

---

## Terminal 3 — Tauri Desktop App (Rust + Sidecar + Webview)

```bash
cd ~/ZenflowProjects/ZenithScience/zen-sci-portal

# Ensure the fresh gateway binary is in place
cp ~/ZenflowProjects/AgenticGatewayByDojoGenesis/bin/agentic-gateway \
   src-tauri/binaries/agentic-gateway-aarch64-apple-darwin

# Build and run the full desktop app
npm run tauri-dev
```

**Watch for:**
- `Compiling zen-sci-portal v0.1.0`
- Webview window opens
- Gateway sidecar starts automatically (you can stop Terminal 1 if using sidecar)

> **Sidecar vs standalone:** If Terminal 1's gateway is already running on :8080, the sidecar will detect it and skip launching a second instance. To test sidecar auto-start, kill Terminal 1 first.

---

## Testing Checklist

### 1. Authentication
- [ ] Open the app → redirected to `/auth`
- [ ] Click "Use locally" → session created as `local-user`
- [ ] Sidebar shows email + role

### 2. Workspace (Document Management)
- [ ] Navigate to `/workspace`
- [ ] Create a new document (try each format: LaTeX, Blog, Grant, Slides, Newsletter, Paper)
- [ ] Click into a document → `/workspace/[docId]`
- [ ] Type content → auto-save indicator appears (3s debounce)
- [ ] Click "+ Section" to add sections
- [ ] Click "Organize sections" → modal with drag-drop reorder
- [ ] Click "Convert" → DocConverter panel slides open
- [ ] Try each conversion format (PDF, Slides, Blog, etc.)

### 3. Agent Chat (v1.2 Streaming)
- [ ] From `/workspace`, the right-panel AgentChat should be visible
- [ ] Type a message and send
- [ ] **Streaming test:** Watch for live text appearing character-by-character (not all at once)
- [ ] If agent uses tools, tool invocation badges should appear
- [ ] Switch provider via dropdown (if multiple available)
- [ ] Navigate to `/chat` for standalone full-page chat

### 4. Memory
- [ ] Navigate to `/memory`
- [ ] **Browse tab:** See paginated memory items
- [ ] **Search tab:** Search for a term
- [ ] Create a new memory item (try types: note, insight, fact, observation, question, reference)
- [ ] Click an item to edit it

### 5. Tools
- [ ] Navigate to `/tools`
- [ ] See tool grid with namespace filters
- [ ] Search for a tool (e.g., "validate")
- [ ] Click a tool → ToolInvoker modal opens
- [ ] Fill in inputs and invoke (e.g., `zen_sci_latex:validate_document` with sample LaTeX)

### 6. Pipelines
- [ ] Navigate to `/pipelines`
- [ ] Build a pipeline using presets (or custom)
- [ ] Submit → DAG execution starts
- [ ] Watch DAG viewer show node states (planned → running → completed)

### 7. Search
- [ ] Use Cmd/Ctrl+K shortcut from any page → search bar focuses
- [ ] Type a query → navigated to `/search?q=...`
- [ ] See results with file type badges
- [ ] Use filter sidebar (file type, date range)

### 8. Settings
- [ ] Navigate to `/settings`
- [ ] See gateway status (up/version/uptime)
- [ ] Try "Restart Gateway" button
- [ ] See LLM providers list
- [ ] Enter/update an API key for anthropic or openai
- [ ] Verify "Configured" badge appears after save
- [ ] Log out → redirected to `/auth`

### 9. SSE Streaming Verification (curl)
```bash
# Create agent
AGENT_ID=$(curl -s -X POST http://localhost:8080/v1/gateway/agents \
  -H 'Content-Type: application/json' \
  -d '{"workspace_root": "/tmp/test"}' | jq -r .agent_id)

# Test non-streaming (baseline)
curl -s -X POST "http://localhost:8080/v1/gateway/agents/$AGENT_ID/chat" \
  -H 'Content-Type: application/json' \
  -d '{"message": "Say hello"}' | jq .

# Test streaming (SSE)
curl -N -X POST "http://localhost:8080/v1/gateway/agents/$AGENT_ID/chat" \
  -H 'Content-Type: application/json' \
  -d '{"message": "Say hello", "stream": true}'
```

**Expected SSE output:**
```
event: thinking
data: {"type":"thinking","data":{"message":"Processing your request..."},...}

event: response_chunk
data: {"type":"response_chunk","data":{"content":"Hello! How can I..."},...}

event: complete
data: {"type":"complete","data":{"usage":{...}},...}

data: [DONE]
```

### 10. Memory Garden (Backend-only, curl)
```bash
# These endpoints exist but aren't wired to the frontend yet
curl -s http://localhost:8080/v1/garden/stats | jq .
curl -s http://localhost:8080/v1/seeds | jq .
```

---

## Shutdown

1. **Terminal 3:** `Ctrl+C` (stops Tauri dev + sidecar)
2. **Terminal 2:** `Ctrl+C` (stops Vite dev server)
3. **Terminal 1:** `Ctrl+C` (stops standalone gateway)

---

## Known Behaviors

| Behavior | Explanation |
|----------|-------------|
| Ollama takes 30-90s for first response | qwen3:8b has a "thinking" phase; subsequent responses are faster |
| `thinking` event appears twice in SSE | Duplicate emit in `handleGatewayAgentChatStream`; cosmetic, non-blocking |
| MCP App route (`/app/[appId]`) shows empty iframe | `ui://` protocol not resolved without MCP Apps enabled |
| Search results aren't clickable | ResultsList renders items but click-to-open isn't wired |
| DocConverter shows raw output | No preview rendering; output is raw tool result text |
