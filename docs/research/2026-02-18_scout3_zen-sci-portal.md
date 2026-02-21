# Strategic Scout 3: zen-sci-portal Architecture Gap Analysis
## Where the Desktop Spec Underutilizes AgenticGateway and Needs Elevation to Stateful Agentic Workspace

**Date:** 2026-02-18
**Status:** Architecture Reconnaissance (No Code)
**Audience:** Cruz Morales, Product Leadership
**Purpose:** Identify concrete gaps between the current v0.1 spec and a full stateful agentic workspace

---

## Executive Summary

The zen-sci-portal v0.1 spec is functionally complete as a **local-first document creation tool**. However, it severely underutilizes AgenticGateway's core strengths and misses the architectural opportunity to create a **stateful agentic workspace** — a persistent, context-aware agent partner that can orchestrate multi-step knowledge-work tasks, reason across your document corpus, and maintain memory across sessions.

**The core reframe:** The portal is currently designed as a **frontend-to-backend plumbing layer** (user clicks button → Tauri command → Gateway MCP call → result). It should be designed as a **first-class agent workspace** where the agent has persistent memory of your research, can run DAG orchestrations, and surfaces agentic reasoning as first-class UI elements.

**Three major architectural gaps identified:**

1. **Gateway capabilities are hidden, not surfaced** — DAG orchestration, memory persistence, multi-step planning all exist in AgenticGateway but are never exposed to the user or the portal's UI.

2. **Document catalog ownership is ambiguous** — The spec treats local Tantivy as primary, but AgenticGateway owns canonical document storage. No reconciliation strategy exists.

3. **Session memory is ephemeral** — Each portal session is stateless. No persistent agent context across launches, no ability to resume interrupted workflows, no reasoning history.

---

## Scout 1: What AgenticGateway Capabilities Are Missing from the Portal Spec?

### Context: AgenticGateway v1.0.0 Production Capabilities

**AgenticGateway (Go, 188 files, ~39K LOC)** provides:

| Capability | Specification | Integration in Portal Spec |
|-----------|---|---|
| **8 LLM providers** (Anthropic, OpenAI, Google, Groq, Mistral, Kimi, DeepSeek, Ollama) | OpenAI-compatible `/v1/chat/completions` endpoint | **NOT USED** — spec makes no mention of agent chat/reasoning |
| **DAG orchestration** | Multi-step agentic pipelines as directed acyclic graphs | **NOT USED** — no mention of workflow or task composition |
| **33 registered tools** | File ops, web ops, computation, planning, research, visual | **PARTIALLY USED** — only MCP tools (document conversion) are invoked |
| **44 tiered skills** (Tiers 0-3, meta-skill invocation) | Reusable reasoning patterns with call-depth enforcement | **NOT USED** — no skill invocation in portal |
| **SQLite-backed conversation memory** | Persistent across sessions, embeddings-based retrieval | **NOT USED** — portal has no conversation history or context persistence |
| **Langfuse OTEL observability** | End-to-end tracing of agentic actions | **NOT USED** — no telemetry integration in portal |
| **App Bridge (`apps/` module)** | Registers MCP Apps at `ui://` URIs, renders in webview | **PARTIALLY USED** — spec mentions Phase 4 app iframes but doesn't show how they integrate with agent reasoning |
| **MCP Host manager** | Full Model Context Protocol host, manages all 6 ZenSci servers | **USED** — only as a tool dispatcher, not as a reasoning engine |

### What's Implemented in Portal Spec

**The Rust backend defines exactly 11 IPC commands:**

```rust
search(query, limit)           // Local Tantivy search
convert_document(source, format)     // Route to Gateway MCP tool
login(email, password)         // Auth
logout()                       // Auth
check_session()                // Auth
list_documents(folder)         // File system listing
create_document(name, format)  // File creation
get_gateway_status()           // Health check
get_mcp_app(appId)            // Load Phase 4 app
watch_folder(path)             // File watcher control
index_reindex()                // Index maintenance
```

**All are synchronous request-response patterns.** Each command:
- Serializes to JSON
- Crosses Tauri IPC boundary
- Rust processes, returns result
- Frontend handles response
- No agentic reasoning, no multi-step composition, no memory

### What's Missing: Agentic Capabilities

**1. No Agent Chat Endpoint**
- **What exists in Gateway:** `/v1/chat/completions` (OpenAI-compatible) + memory context
- **What portal needs:** SvelteKit component that can invoke agent chat, send prompts like "cite the photosynthesis mechanism from my papers," get agentic responses
- **Spec gap:** Zero mention of chat UI, no agent reasoning display, no prompt input
- **Concrete gap location:** Section 3.4 (SvelteKit Frontend) lists routes `/`, `/auth`, `/workspace`, `/search`, `/settings`, `/app/*` — **no `/chat` or `/agent` route**

**2. No DAG Orchestration Surface**
- **What exists in Gateway:** DAG execution engine with state tracking, conditional branching, tool chaining
- **What portal needs:** UI to compose workflows ("take draft → run citation check → simulate reviewer feedback → suggest revisions")
- **Spec gap:** No workflow builder, no visual DAG representation, no task queue display
- **Concrete gap location:** Section 3.6 (Tauri IPC Commands) — **no `compose_workflow` or `execute_dag` command**

**3. No Persistent Agentic Memory Integration**
- **What exists in Gateway:** SQLite conversation store + embeddings for semantic search + tiered context construction
- **What portal needs:** Access to stored conversation history, ability to reference past reasoning, resume interrupted workflows
- **Spec gap:** Each session starts fresh; no reference to Gateway's memory system
- **Concrete gap location:** Section 3.3.2 (AuthManager) manages session tokens but **not conversation context** — compare to AuthManager's `SessionState` struct which only stores `user_id`, `email`, `token_expiry` — **no `conversation_id` or `memory_context` field**

**4. No Skill or Tool Visibility in UI**
- **What exists in Gateway:** 44 tiered skills, 33 tools, all discoverable via `/tools` endpoint
- **What portal needs:** Skills browser showing available reasoning patterns, tools marketplace, ability to invoke skills from UI
- **Spec gap:** Only MCP tools (document conversion) are visible; gateway skills are invisible
- **Concrete gap location:** Section 3.6 lists `list_documents`, `list_tools` never mentioned — **no `get_available_skills` or `get_tool_catalog` command**

**5. No Multi-Provider Routing or Fallback UI**
- **What exists in Gateway:** 8 LLM providers, automatic fallback on rate limit or failure
- **What portal needs:** User control over which provider to use ("use Ollama locally", "fallback to Claude if Ollama unavailable"), cost tracking, provider health dashboard
- **Spec gap:** Gateway choice is opaque; user has no control
- **Concrete gap location:** Section 3.1 (System Overview) and Section 3.5 (AgenticGateway Integration) **never mention provider configuration** — Rust backend launches Gateway with unknown config file path

**6. No Agentic Task Queue or Background Job Visibility**
- **What exists in Gateway:** Task queuing, state tracking, ability to pause/resume/cancel
- **What portal needs:** UI showing background conversion jobs, ability to manage queue, view progress/errors
- **Spec gap:** File watcher queues conversions, but queue is invisible to user
- **Concrete gap location:** Section 3.3.3 (FileWatcher) defines `FileEvent` enum and "queue for ZenSci processing" — **but no queue management UI, no Tauri command to list/cancel jobs**

### Gap Summary: Capabilities Exposed vs. Hidden

| Capability | Exposed? | How | Portal UX Impact |
|-----------|----------|-----|------------------|
| Agent Chat | ❌ No | Not exposed | User can't converse with agent; tool calls only |
| DAG Workflows | ❌ No | Not exposed | No multi-step task composition |
| Memory System | ❌ No | Only session tokens | No conversation history, no context carryover |
| Skills/Tools | ⚠️ Partial | Only MCP document tools | User doesn't know what the system can do |
| Provider Routing | ❌ No | Hidden in Gateway config | No user control, no fallback visibility |
| Task Queue | ❌ No | Opaque to frontend | User can't manage conversions, no progress tracking |

---

## Scout 2: What Does "Stateful Agentic Workspace" Look Like Concretely?

### What It Is NOT (Current Spec)

The current spec is a **stateless document forge**:
- User creates/edits/searches documents
- Each action is atomic (edit → save → convert)
- No agent reasoning or memory
- Gateway is a tool invocation engine, not a reasoning partner
- Resetting the app → full restart

### What It IS (Desired Architecture)

A **stateful agentic workspace** is:

**1. Persistent Agent Memory Context**

```
Portal Launch
  ↓
Rust backend retrieves agent session:
  - Last active research project
  - Conversation history (last 50 exchanges)
  - Current task state (if paused)
  - Available context documents
  ↓
SvelteKit initializes agent panel with:
  - "Welcome back. You were working on [project]. Last task: [paused task]."
  - Quick-jump buttons to recent reasoning threads
  - Relevant document suggestions based on past interactions
  ↓
User can immediately resume where they left off without re-explaining context
```

**Concrete requirement:** Portal must invoke `get_session_memory()` IPC command on startup, retrieve agent context from Gateway's SQLite, and surface it in UI.

---

**2. Agentic DAG Workflows as First-Class UI Elements**

```
User clicks "Prepare Paper for Submission"
  ↓
SvelteKit shows DAG builder (visual node flow):
  [Draft Paper] → [Citation Check] → [Reviewer Sim] → [Feedback Integration] → [Output PDF]
  ↓
User configures each node (e.g., reviewer personas, citation rules)
  ↓
User clicks "Execute"
  ↓
Rust sends DAG to Gateway: `execute_workflow({nodes, edges, config})`
  ↓
Gateway processes multi-step task:
  - Task 1: Citation validation (check every cite in Gateway's knowledge base)
  - Task 2: Simulate 3 reviewers (invoke chat endpoint with different personas)
  - Task 3: Synthesize feedback (aggregate, rank by severity)
  - Task 4: Auto-generate revision suggestions
  ↓
Portal UI shows real-time progress:
  [✓ Done] Citation Check (5 issues found)
  [⏳ In Progress] Reviewer Simulation
  [⏳ Queued] Feedback Integration
  [⏳ Queued] Output PDF
  ↓
Results stream back with actionable suggestions (agent-generated, not just tool output)
```

**Concrete requirement:** Portal must support:
- `compose_dag({nodes, edges})` — save DAG as template
- `execute_dag({dag_id, params})` — run multi-step task, stream results
- `get_dag_templates()` — browse saved workflows
- Real-time event stream (WebSocket) showing task progress

---

**3. Tool Calls as Visible, Agentic Actions**

**Current spec:** Hidden plumbing
```
User clicks "Convert to PDF"
  ↓ [invisible to user]
Rust calls Gateway: convert_to_pdf(markdown)
  ↓ [invisible to user]
Gateway calls latex-mcp tool
  ↓ [invisible to user]
Result returns to Rust
  ↓ [invisible to user]
UI shows "PDF ready"
```

**Desired spec:** Visible, traceable, agentic actions
```
User clicks "Convert to PDF"
  ↓ [VISIBLE] SvelteKit shows "Agent planning action..."
  ↓ [VISIBLE] Action panel appears:
    "Agent decided to: convert_document
     Reason: User requested PDF output
     Tool: latex-mcp (v0.1)
     Estimated time: 3–5 seconds
     Provider: DeepSeek (reasoning), pdflatex (rendering)"
  ↓ [VISIBLE] Real-time output stream:
    "- Validating markdown... ✓
     - Detecting math expressions... ✓
     - Resolving citations... ✓
     - Rendering LaTeX... ⏳
     - Generating PDF... ⏳"
  ↓ [VISIBLE] Result with agent explanation:
    "PDF ready. Found 3 inline equations, 12 citations.
     Recommend: Add subject headings to Table of Contents (2 missing).
     Similar papers from your corpus: [3 suggestions]"
```

**Concrete requirement:** Portal must support:
- `invoke_tool_with_reasoning({tool, args, explain})` — call tool + explain why
- `stream_tool_output()` — WebSocket stream of tool execution
- Action panel component showing: reason, tool, progress, suggestions
- Citation of similar documents in corpus (agent memory system)

---

**4. Persistent Cross-Session Task State**

**Current spec:** Ephemeral
- User starts task, app crashes → task lost
- User pauses workflow, closes app → workflow forgotten
- No way to resume interrupted work

**Desired spec:** Persistent
```
User starts "Prepare Paper for Submission" DAG
  (step 1-3 complete, step 4 paused)
  ↓
User closes portal
  ↓
Portal saves task state to Gateway:
  - DAG execution ID
  - Completed steps (with results)
  - Current step (paused)
  - User context
  ↓
[Next day, user reopens portal]
  ↓
Agent displays: "You were working on Paper Submission task.
  3 of 5 steps complete. Resume?"
  ↓
User clicks "Resume"
  ↓
Gateway continues from step 4 using prior context
  ↓
Result: seamless task continuation
```

**Concrete requirement:** Portal must support:
- `save_task_state({task_id, progress})` — checkpoint workflow
- `resume_task({task_id})` — pick up where left off
- Task history in sidebar: "Recent Tasks" with status

---

**5. Memory-Aware Document Discovery**

**Current spec:** Keyword search only (Tantivy)
```
User types "photosynthesis"
  ↓
Tantivy returns 10 documents matching keyword
```

**Desired spec:** Agentic semantic search across memory
```
User types "photosynthesis mechanism"
  ↓
Gateway searches:
  1. Local Tantivy (fast keyword match)
  2. Conversation memory embeddings (semantic search)
  3. Citation graph (what papers cite this concept?)
  4. Agent suggestions (papers you've discussed this with, in other contexts)
  ↓
SvelteKit displays results with agent explanation:
  [Document 1: "Light Reactions in C3 Plants"]
    Relevance: 92% (keyword match + concept proximity)
    Agent note: "This was cited in your 2024 photosynthesis review.
                 You noted Z-scheme dynamics were incomplete."
    Quick action: [View] [Add to current doc] [Create note]

  [Document 2: "Energy Transfer in Photosystems"]
    Relevance: 87% (semantic relevance)
    Agent note: "Not previously opened, but highly cited by
                 papers you've read. Complements your draft."
    Quick action: [View] [Cite] [Add to reading list]
```

**Concrete requirement:** Portal must support:
- `semantic_search({query, context})` — agent-powered discovery
- `get_memory_insights({doc_id})` — agent suggestions based on history
- Memory metadata shown in search results (prior interactions, related concepts)

---

### Concrete Data Model Changes Required

**Current AuthManager session state:**
```rust
pub struct StoredSession {
    pub token: String,
    pub refresh_token: String,
    pub user_id: String,
    pub email: String,
    pub expires_at: i64,
}
```

**Desired agentic session state:**
```rust
pub struct AgenticSessionState {
    // Auth (as before)
    pub token: String,
    pub user_id: String,
    pub email: String,

    // Agent context (new)
    pub agent_session_id: String,
    pub last_project_id: String,
    pub active_task_id: Option<String>,
    pub task_state: TaskExecutionState,  // paused/running/completed
    pub conversation_history: Vec<Message>,
    pub memory_context: MemoryContext,
    pub last_memory_refresh: i64,
}

pub struct TaskExecutionState {
    pub dag_id: String,
    pub current_step: usize,
    pub completed_steps: Vec<StepResult>,
    pub progress_percent: u8,
    pub paused_at: Option<i64>,
}

pub struct MemoryContext {
    pub recent_papers: Vec<DocumentRef>,
    pub active_topics: Vec<String>,
    pub recent_conversations: Vec<ConversationSummary>,
    pub suggested_next_actions: Vec<AgentSuggestion>,
}
```

**Concrete Tauri IPC commands needed:**
```rust
// Agent session
get_agent_session() → AgenticSessionState
save_agent_session({state}) → void
clear_agent_session() → void

// Agentic chat
invoke_agent_chat({prompt, context}) → ChatResponse (streaming)
get_conversation_history({limit}) → Vec<Message>

// DAG workflows
compose_dag({nodes, edges}) → DAG
execute_dag({dag_id, params}) → TaskExecution (streaming)
resume_task({task_id}) → TaskExecution
get_task_history() → Vec<TaskMetadata>

// Memory-aware search
semantic_search({query, memory_context}) → SearchResults (with agent explanations)
get_memory_insights({doc_id}) → AgentInsights

// Tool visibility
get_available_tools() → Vec<ToolDefinition>
get_available_skills() → Vec<SkillDefinition>

// Provider control
get_gateway_providers() → Vec<Provider>
set_preferred_provider({provider_id}) → void
```

---

## Scout 3: Cloud-Owned Document Catalog with Git-Like Local Clones

### What the Spec Assumes (Ambiguously)

**Section 3.3.1 (SearchIndexer):**
> "Index a document (PDF, HTML, email, etc.) ... all formats indexed to Tantivy within 2 seconds."

**Section 3.3.3 (FileWatcher):**
> "Monitor ~/.zen-sci/ folder for document changes. Trigger conversions on file save."

**Section 3, Data Flow:**
> "Rust backend writes PDF to ~/.zen-sci/outputs/"

**The implicit model:** Local filesystem is the source of truth. Tantivy indexes local outputs. Files sync to cloud optionally (v1.1).

### The Problem: Two Sources of Truth

**Scenario 1: User creates document on web portal (v2.0+)**
- Document stored in cloud Gateway + S3
- Desktop portal opens
- Local Tantivy has no knowledge of this document
- User searches, doesn't find it

**Scenario 2: User edits locally while offline**
- Edit saved locally
- App comes back online
- Cloud has a different version of the document
- Conflict: which version wins?

**Scenario 3: User deletes document locally**
- Local filesystem: document gone
- Cloud: document still exists
- Tantivy index: document gone
- Cloud sync on next startup: does it resurrect the local copy?

### What "Cloud-Owned with Git-Like Local Clones" Means

**Document catalog ownership:**
- **Cloud (AgenticGateway)** owns the canonical catalog: document IDs, metadata, versions, permissions, sharing
- **Local (Portal)** maintains git-like clones: pull to sync, push to publish, merge conflict resolution
- **Tantivy index** is the local working copy: read-only replica of cloud catalog

**Git analogy:**
```
GitHub (cloud)              Local machine (portal)
┌──────────────┐           ┌──────────────────┐
│ Repo master  │    ←→    │ Local git clone   │
│ (canonical)  │    pull  │ (working copy)    │
│              │   push   │                   │
│ - documents  │ ←merge→  │ - files           │
│ - metadata   │ ↑ ↓      │ - .git (sync info)│
│ - versions   │          │ - Tantivy index   │
│ - sharing    │          │   (searchable)    │
└──────────────┘          └──────────────────┘
```

### Implementation: Document Catalog Model

**Gateway owns (cloud store):**
```json
{
  "document_id": "doc-123",
  "title": "Photosynthesis Mechanisms",
  "author": "user-456",
  "version": 5,
  "last_modified": 1708294800,
  "content_hash": "abc123def456",
  "outputs": [
    {"format": "pdf", "hash": "xyz789", "size": 1240000},
    {"format": "html", "hash": "xyz790", "size": 450000}
  ],
  "permissions": {"user-456": "owner", "user-789": "read"},
  "sync_status": "published"
}
```

**Portal maintains locally (clone):**
```json
{
  "document_id": "doc-123",
  "local_path": "~/zen-sci/photosynthesis-mechanisms/",
  "source_file": "doc-123.md",
  "outputs": {
    "pdf": "doc-123.pdf",
    "html": "doc-123.html"
  },
  "cloud_version": 5,
  "local_version": 5,
  "last_synced": 1708294800,
  "sync_status": "synced",
  "changes": []
}
```

### Sync Protocol: Pull, Push, Merge

**On portal startup:**
```
1. Check Gateway for remote changes:
   GET /v1/gateway/catalog?since=last_sync_timestamp

2. For each remote document:
   - If local_version < cloud_version:
     → PULL (download new version)
   - If local_version > cloud_version:
     → PUSH pending (mark for upload)
   - If both changed:
     → MERGE conflict (auto-merge or user chooses)

3. Update Tantivy index with merged catalog
```

**When user saves a local file:**
```
1. User edits ~/zen-sci/doc-123.md, saves

2. FileWatcher detects change

3. Compute content_hash (SHA256)

4. Compare with cloud version:
   - If cloud unchanged: PUSH immediately
   - If cloud changed: queue for manual merge

5. On next sync:
   - If conflict: show merge UI
   - User chooses: local wins / cloud wins / manual merge
   - Apply CRDT resolution if configured
```

**When user deletes a file:**
```
1. User deletes ~/zen-sci/doc-123.md

2. FileWatcher detects deletion

3. Mark in local sync tracker: "deletion_pending"

4. On next sync to cloud:
   - Ask user: "Confirm delete doc-123 from cloud?"
   - If yes: DELETE on Gateway
   - If no: re-download from cloud
```

### Tantivy as a Read-Only Replica

**Key design principle:** Tantivy is NOT the source of truth; it's the **local search replica**.

**Index sync algorithm:**
```
1. On app startup:
   GET /v1/gateway/catalog → fetch remote catalog metadata

2. For each document:
   - Compute local index hash
   - Compare with remote content hash

3. If mismatch or new document:
   - Download or recompute
   - Reindex in Tantivy

4. Add metadata to Tantivy:
   - doc_id (remote ID)
   - cloud_version (for conflict detection)
   - last_synced (timestamp)
   - sync_status (synced/pending_pull/pending_push)

5. Optional: Tag index with version number
   - Index v5 = matches cloud catalog version 5
   - On startup, if index_version < cloud_version → reindex
```

### Conflict Resolution Strategies

**Three modes (user configurable):**

**Mode 1: Local Wins (Default for desktop)**
- User's local edits are authoritative
- Cloud gets updated with local version on push
- Risk: Lose cloud changes

**Mode 2: Cloud Wins (Default for web)**
- Server's version is authoritative
- Local gets overwritten on pull
- Risk: Lose local edits

**Mode 3: CRDT Merge (Requires Automerge)**
- Merge conflicts automatically using CRDT semantics
- Both changes preserved if non-overlapping
- User reviews overlapping sections
- Risk: Requires user education on CRDT behavior

**Concrete Tauri IPC commands needed:**
```rust
// Catalog sync
sync_catalog() → SyncResult (lists changes, conflicts)
get_catalog_status() → CatalogStatus
get_sync_conflicts() → Vec<SyncConflict>

// Merge resolution
resolve_conflict({doc_id, strategy}) → void
merge_document_crdt({doc_id}) → MergeResult

// Document lifecycle
pull_document({doc_id}) → Document
push_document({doc_id}) → PushResult
delete_document({doc_id, confirm}) → void
```

---

## Scout 4: V1 Architecture Gaps

### What V1 Should Include (Full Architecture)

**The current spec (v0.1) is a **bootstrap**. V1 needs:**

**1. Auth Model Across Surfaces**

*Current spec gap:* JWT stored locally in keychain. No mention of cross-device auth, device registration, or multi-surface session.

*V1 requirement:*
```
User logs in on desktop portal
  ↓
Rust registers device: POST /v1/gateway/devices/register
  {device_name, device_id, os, os_version}
  ← returns device_token (unique credential)

User logs in on web portal
  ↓
Same user credential works
  ↓
User's session shows: "Logged in on 2 devices"
  [Desktop: macOS, last seen 2 hours ago]
  [Web: Chrome, last seen now]

Admin can revoke device access if needed
```

*Concrete Rust changes:*
- Device registration API
- Device tracking in session state
- Device token validation
- Cross-surface session sync

**2. Catalog Sync Protocol (Specified Above)**

*Current spec gap:* No sync strategy between local and cloud.

*V1 requirement:* Git-like pull/push/merge as detailed in Scout 3.

**3. Agentic Session Persistence (Specified Above)**

*Current spec gap:* Each portal launch is a fresh session.

*V1 requirement:* Agent memory, task state, conversation history persisted across launches.

**4. Real-Time Collaboration (Deferred in Spec)**

*Current spec explicitly marks:* ❌ "Real-time collaboration in v1.0."

*V1 requirement (if included):*
- WebSocket-based presence + live cursor
- Automerge CRDT for conflict-free merging
- Tokio WebSocket server in Rust backend
- Svelte-based collaborative editor

**5. Provider Management & Fallback**

*Current spec gap:* Provider choice is opaque.

*V1 requirement:*
```
SvelteKit settings panel:
  [Select LLM Provider]
  - Claude (default)
  - GPT-4 (if API key configured)
  - Local Ollama (if running)
  - DeepSeek (if API key configured)

  [Fallback strategy]
  ☑ Auto-fallback on rate limit
  ☑ Auto-fallback on error

  [Cost tracking]
  This month: $XX.XX (Claude: $15, GPT-4: $8)
```

*Concrete Rust changes:*
- Gateway config file management
- Provider enumeration
- Fallback policy enforcement
- Cost tracking via Gateway telemetry

**6. MCP App Integration Protocol**

*Current spec mentions* Phase 4 apps in iframes but provides zero message protocol.

*V1 requirement:*
```
Formal protocol for SvelteKit ← → MCP App iframe communication

SvelteKit → MCP App:
  postMessage({
    type: "requestDocument" | "requestSearch" | "executeTool",
    payload: {...}
  })

MCP App → SvelteKit:
  postMessage({
    type: "documentLoaded" | "searchResults" | "toolResult",
    payload: {...}
  })

Error handling, auth, CSP policy defined
```

*Concrete spec requirement:* "MCP App Developer Guide" with protocol, examples, security model.

---

### Gaps in Current Spec vs. V1 Full Architecture

| Component | v0.1 Spec | V1 Requirement | Impact |
|-----------|-----------|----------------|--------|
| **Auth scope** | Single device, keychain | Multi-device, device registration | Cross-surface UX broken |
| **Document catalog** | Local Tantivy as source of truth | Cloud-owned, git-like sync | Data loss on conflict, no cloud handoff |
| **Agent memory** | None | Persistent session, task state | No context carryover, can't resume tasks |
| **DAG workflows** | Not mentioned | Specified, with UI | No multi-step orchestration |
| **Provider control** | Hidden in Gateway | User-controlled with fallback | No user agency, no cost tracking |
| **Real-time collab** | Deferred to v1.1 | WebSocket + Automerge (optional) | Single-user only, no live editing |
| **MCP App protocol** | Mentioned, not specified | Formal protocol + guide | Phase 4 apps can't integrate |
| **Task queue visibility** | Opaque | UI shows progress, can manage jobs | User frustration (invisible processing) |

---

## Scout 5: The Reframe — What's the Core Tension?

**The hidden tension in the spec:**

The portal is designed as a **"local-first single-user document creation tool with optional cloud sync"** but positioned as **"Create surface in a 3-surface social knowledge platform."**

These are fundamentally different products:

### Product A: Standalone Local-First Tool
- User creates documents locally
- Optional cloud backup
- No collaboration
- No social features
- Offline-capable
- Competitor: Obsidian Desktop

### Product B: Create Surface of Social Knowledge Platform
- User creates documents, shares with collaborators
- Documents live in cloud by default
- Local copies are clones (git-like)
- Real-time collaboration
- Social activity feeds, presence awareness
- Offline-capable but cloud-connected
- Competitor: Notion Desktop, Google Docs Offline

**The spec conflates both.**

### What the Reframe Reveals

> **The real strategic choice is not "should we build a desktop app?" but "is this a single-user tool or a social platform's Create surface?"**

**If Product A (Standalone):**
- Keep local Tantivy as source of truth
- Cloud sync is optional, v1.1+
- No collaboration features needed
- Focus on local UX (search speed, file organization)
- 10-12 week timeline (current spec)

**If Product B (Platform Create Surface):**
- Cloud owns the catalog, local syncs as clone
- Collaboration is v1.0 feature
- User management, permissions, sharing are core
- Agentic workspace becomes a product differentiator
- 16-20 week timeline (requires sync, auth, DAG, memory)

**The current spec claims Product B positioning but implements Product A architecture.**

---

## Summary of Gaps

### 1. Capability Gaps (Scout 1)

| Gateway Capability | Current Spec | Needed for Agentic Workspace |
|---|---|---|
| Chat/Reasoning | Not exposed | Required (agent conversation UI) |
| DAG Orchestration | Not exposed | Required (workflow builder + execution UI) |
| Memory System | Not exposed | Required (persistent session + task state) |
| Skills/Tools | Partially exposed (MCP only) | Required (tools browser, skill discovery) |
| Provider Routing | Hidden | Required (user control + fallback UI) |
| Task Queue | Opaque | Required (progress UI, job management) |

### 2. Data Model Gaps (Scout 2)

| Data | Current Spec | Needed for Agentic Workspace |
|---|---|---|
| Session state | JWT + keychain | + agent_session_id + task_state + memory_context |
| Document metadata | Local file properties | + cloud_version + sync_status + conflicts |
| Task tracking | None | Required (task_id + execution_state + pause/resume) |
| Workflow templates | None | Required (save DAGs, compose, version) |

### 3. Protocol Gaps (Scout 2 & 4)

| Protocol | Current Spec | Needed for V1 |
|---|---|---|
| Agent chat | Not mentioned | `/agent/chat` endpoint + streaming |
| DAG execution | Not mentioned | `/workflow/execute` endpoint + progress events |
| Catalog sync | Not mentioned | `/sync` protocol (pull/push/merge) |
| Device auth | Not mentioned | Device registration + multi-device sessions |
| MCP App messaging | Mentioned, not spec'd | Formal `postMessage` protocol + CSP |

### 4. UI Gaps (Scout 2)

| UI Surface | Current Spec | Needed for Agentic Workspace |
|---|---|---|
| Chat panel | Not mentioned | Required (prompt input, reasoning display) |
| DAG builder | Not mentioned | Required (workflow visual composition) |
| Task queue | Not mentioned | Required (progress, cancellation) |
| Sync status | Not mentioned | Required (conflict alerts, merge UI) |
| Memory/Context | Not mentioned | Required (conversation history, suggested actions) |
| Tools browser | Not mentioned | Required (discover tools, invoke skills) |
| Provider selector | Not mentioned | Required (LLM choice, cost tracking) |

---

## Conclusion: The Reframe

**The core reframe:** The portal is not a "local-first document tool that happens to connect to the cloud." It is a **stateful agentic workspace** that must unify local and cloud state, expose agentic reasoning as a first-class UI, and persist agent memory across sessions.

**This reframe changes everything:**

1. **Auth:** From "keychain tokens" to "multi-device session management"
2. **Documents:** From "local files with optional cloud sync" to "cloud-owned with git-like local clones"
3. **Agent:** From "hidden tool dispatcher" to "visible reasoning partner"
4. **Memory:** From "ephemeral per-session" to "persistent across launches"
5. **Workflows:** From "single atomic actions" to "multi-step DAG orchestration"

**The current spec is 60% complete for Product A (standalone tool) but 20% complete for Product B (platform Create surface + agentic workspace).**

**Recommended path forward:**

1. **Decide which product:** Standalone tool (10-12 weeks) or platform Create surface (16-20 weeks)?
2. **If platform:** Add 12 new Tauri IPC commands (Scout 2 list)
3. **If platform:** Add 5 new SvelteKit routes (chat, workflows, sync, tools, memory)
4. **If platform:** Redesign session state model to include agent context
5. **If platform:** Specify catalog sync protocol (Scout 3)
6. **If platform:** Write MCP App protocol specification (Scout 4)

**Timeline implications:**
- **Product A (current spec):** 8-12 weeks, ships v0.1 as-is
- **Product B (agentic workspace):** 16-20 weeks, requires architectural redesign before implementation

---

**Scout prepared by:** Strategic Scout
**Status:** Ready for product decision
**Next gate:** Clarify Product A vs. Product B positioning
**Output files:**
- This scout: `2026-02-18_scout3_zen-sci-portal.md`
