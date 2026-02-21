# Zenflow Commission: Phase 3 — Integration: Provider Key Management, Full-Screen Chat & Pipelines

**Objective:** Add provider API key management UI to Settings (Anthropic, OpenAI), create the `/chat` full-screen route, move pipeline job state to a persistent store, add chat-triggered pipeline proposals, and fix pipeline page quality issues.

**Prerequisite:** Phase 1 and Phase 2 must be complete. `agentStore` exists, `AgentChat.svelte` reads from the store, document context and tool-calling loop are live.

---

## 1. Context & Grounding

**Primary Specification:**
- `docs/specs/zen-sci-portal-v1.1-spec.md` — Full v1.1 spec. Phase 3 covers §3.7 (Provider Key Management), §3.3.3 (Full-Screen Chat Route), §3.8 (Pipelines Chat Integration), and §4.2 Week 3 checklist.

**Pattern Files (Follow these examples):**
- `zen-sci-portal/src/routes/settings/+page.svelte` — The Settings page to extend. Read it fully. It already loads providers via `listProviders()`, shows them in cards with `provider.status`. The new "AI Providers" section is a new card below the existing "LLM Providers" section. Follow the existing `card p-5 space-y-3` / `btn-primary` / `btn-secondary` Tailwind patterns exactly.
- `zen-sci-portal/src/routes/+layout.svelte` — Navigation layout. `navItems` array drives the sidebar nav. Add `{ href: '/chat', label: 'Chat', icon: 'chat' }` to this array. Follow the same object shape.
- `zen-sci-portal/src/lib/stores/activeDocument.ts` — Pattern for a simple writable store singleton. The new `pipelineStore.ts` follows this exact pattern but with a more complex state shape.
- `zen-sci-portal/src/routes/pipelines/+page.svelte` — The pipelines page to enhance. Currently 673 lines. Read it fully before touching it. `jobs` is local `$state<DagJob[]>([])` — needs migration to store. PRESETS array defines the 4 standard pipelines. `draftSteps` is the builder state.
- `zen-sci-portal/src/lib/components/AgentChat.svelte` (Phase 2 output) — Already has `pendingPatch` pattern. The new `pipelineProposal` follows the same pattern: a `pendingPipelineProposal` state variable + a confirmation card UI below messages, similar to the patch confirmation card.
- `zen-sci-portal/src-tauri/src/gateway/client.rs` — Pattern for adding new HTTP methods to GatewayClient. The provider settings endpoints (`POST /v1/settings/providers`, `GET /v1/settings/providers`) follow the same `post_json`/`get_json` helper pattern.
- `zen-sci-portal/src-tauri/src/auth/keychain.rs` — Keychain integration already exists. Read this file to understand the keychain API before implementing secure key storage.

**Files to Read Before Starting:**
- `zen-sci-portal/src/lib/types/index.ts` — All existing types. Add `PipelineProposal` interface here.
- `zen-sci-portal/src/lib/api/tauri.ts` (Phase 2 output) — Add `setProviderKey`, `getProviderSettings` IPC wrappers.
- `zen-sci-portal/src-tauri/src/commands/gateway.rs` — Add `set_provider_key`, `get_provider_settings` commands.
- `zen-sci-portal/src-tauri/src/auth/keychain.rs` — Understand existing keychain read/write API.
- `zen-sci-portal/src/routes/pipelines/+page.svelte` — Full 673-line file — read all of it before modifying.

---

## 2. Detailed Requirements

### 2.1 Create `src/lib/stores/pipeline.ts` (NEW FILE)

Create `zen-sci-portal/src/lib/stores/pipeline.ts`:

```typescript
import { writable } from 'svelte/store';
import type { DagJob } from '$lib/types/index.js';

export interface PipelineResult {
  jobId: string;
  content: string;
  format: string;
}

interface PipelineStoreState {
  jobs: DagJob[];
  pipelineResult: PipelineResult | null;
}

function createPipelineStore() {
  const { subscribe, update, set } = writable<PipelineStoreState>({
    jobs: [],
    pipelineResult: null,
  });

  return {
    subscribe,
    addJob: (job: DagJob) => {
      update(s => ({ ...s, jobs: [job, ...s.jobs] }));
    },
    updateJob: (updatedJob: DagJob) => {
      update(s => ({
        ...s,
        jobs: s.jobs.map(j => j.id === updatedJob.id ? updatedJob : j),
      }));
    },
    setPipelineResult: (result: PipelineResult | null) => {
      update(s => ({ ...s, pipelineResult: result }));
    },
    clearResult: () => {
      update(s => ({ ...s, pipelineResult: null }));
    },
  };
}

export const pipelineStore = createPipelineStore();
```

### 2.2 Migrate `pipelines/+page.svelte` to use `pipelineStore`

Modify `zen-sci-portal/src/routes/pipelines/+page.svelte`:

1. **Import** `pipelineStore` from `'$lib/stores/pipeline.js'`.
2. **Replace** local `let jobs = $state<DagJob[]>([])` with `$derived($pipelineStore.jobs)` (read-only derived value from store).
3. **Replace** all `jobs = [newJob, ...jobs]` mutations with `pipelineStore.addJob(newJob)`.
4. **Replace** all `jobs = jobs.map(...)` update mutations with `pipelineStore.updateJob(updatedJob)`.
5. **Keep** all other local state: `draftSteps`, `pipelineName`, `submitting`, `builderError`, `availableTools`, `pollingIds`, `selectedJobId`, `pollInterval` — these remain component-local.
6. **Fix**: Disable the "Run Pipeline" button when `draftSteps.length === 0`. Add `disabled={submitting || draftSteps.length === 0}` to the run button.
7. **Fix**: Add error state with retry for failed steps. In the job detail view, for each step where `step.status === 'failed'`, show a "Retry" button that calls `handleRetryStep(job.id, step.name)`. `handleRetryStep` is a stub that calls `getDagStatus(job.id)` to re-check (full retry requires gateway support — show a "Retry not supported in v1.1" toast if the gateway returns an error).
8. **Add** "Open in Workspace" button on completed jobs: In the completed jobs list, for each job where `job.status === 'completed'`, add an "Open in Workspace" button. On click: call `pipelineStore.setPipelineResult({ jobId: job.id, content: job.steps?.[job.steps.length-1]?.output as string ?? '', format: 'text' })`, then navigate to `/workspace` using `goto('/workspace')`.
9. **Add** step output preview: In the selected job detail view, for each step that has `step.output`, add a collapsible `<details>` element with `<summary>Output</summary>` and the output rendered as `<pre class="text-xs font-mono whitespace-pre-wrap">`.

### 2.3 Add `PipelineProposal` to Types and `AgentResponse`

Add to `zen-sci-portal/src/lib/types/index.ts`:

```typescript
export interface PipelineProposal {
  preset: string;           // e.g. "write_pdf"
  steps: string[];          // tool names in order
  description: string;      // human-readable summary
}
```

Extend the existing `AgentResponse` interface (already has `patch_intent?: PatchIntent` from Phase 2):
```typescript
pipeline_proposal?: PipelineProposal;
```

Extend the Rust `AgentResponse` model in `src-tauri/src/models/gateway.rs`:
```rust
pub pipeline_proposal: Option<serde_json::Value>,
```

### 2.4 Add Pipeline Proposal UI to `AgentChat.svelte`

Modify `zen-sci-portal/src/lib/components/AgentChat.svelte`:

**New state:**
```typescript
let pendingPipelineProposal = $state<import('$lib/types/index.js').PipelineProposal | null>(null);
```

**In `sendMessage()`**, after handling `patch_intent`:
```typescript
if (result.pipeline_proposal) {
  pendingPipelineProposal = result.pipeline_proposal;
}
```

**Import** `pipelineStore` from `'$lib/stores/pipeline.js'`.
**Import** `submitDag` from `'$lib/api/tauri.js'`.
**Import** `goto` from `'$app/navigation'`.

**Add `submitProposedPipeline()` function:**
```typescript
async function submitProposedPipeline(proposal: import('$lib/types/index.js').PipelineProposal) {
  try {
    const pipeline = {
      plan: {
        id: crypto.randomUUID(),
        name: proposal.preset,
        dag: proposal.steps.map((toolName, i) => ({
          id: `step-${i}`,
          tool_name: toolName,
          depends_on: i > 0 ? [`step-${i-1}`] : []
        }))
      }
    };
    const job = await submitDag(pipeline);
    pipelineStore.addJob(job);
    pendingPipelineProposal = null;
    goto('/pipelines');
  } catch (err) {
    // Show error in chat
    agentStore.addMessage({
      role: 'system',
      content: `Pipeline submission failed: ${String(err)}`,
      timestamp: Date.now(),
    });
  }
}
```

**Pipeline proposal confirmation card** — add below the patch confirmation card (`{#if pendingPatch}` block):

```svelte
{#if pendingPipelineProposal}
  <div class="mx-4 mb-2 p-3 rounded-lg bg-zinc-800 border border-zinc-700">
    <p class="text-xs text-zinc-400 mb-2">Pipeline proposal</p>
    <p class="text-sm text-zinc-200">{pendingPipelineProposal.description}</p>
    <div class="flex gap-2 mt-3">
      <button
        onclick={() => submitProposedPipeline(pendingPipelineProposal!)}
        class="px-3 py-1 text-xs rounded bg-emerald-700 hover:bg-emerald-600 text-white transition-colors"
      >
        Run Pipeline
      </button>
      <button
        onclick={() => pendingPipelineProposal = null}
        class="px-3 py-1 text-xs rounded bg-zinc-700 text-zinc-300 hover:bg-zinc-600 transition-colors"
      >
        Dismiss
      </button>
    </div>
  </div>
{/if}
```

### 2.5 Create Full-Screen Chat Route (`/chat`)

Create `zen-sci-portal/src/routes/chat/+page.svelte` (NEW FILE):

```svelte
<script lang="ts">
  import AgentChat from '$lib/components/AgentChat.svelte';
  import { userSession } from '$lib/stores/userSession.js';
  import { agentStore } from '$lib/stores/agent.js';

  let session = $derived($userSession);
  let projectId = $derived(session?.user_id ?? 'local');
</script>

<div class="flex h-full bg-white dark:bg-gray-900">
  <!-- Narrow context sidebar -->
  <aside class="w-56 flex-shrink-0 border-r border-gray-200 dark:border-gray-800 flex flex-col p-4 gap-3 overflow-y-auto">
    <h2 class="text-xs font-semibold text-gray-400 uppercase tracking-wider">Context</h2>
    {#if $agentStore.activeDocumentId}
      <div class="text-sm text-gray-300">
        <p class="text-xs text-gray-500 mb-1">Active document</p>
        <a
          href="/workspace/{$agentStore.activeDocumentId}"
          class="text-brand-400 hover:text-brand-300 text-xs underline"
        >
          Open document
        </a>
      </div>
    {:else}
      <p class="text-xs text-gray-500">No document open. Open a workspace to link context.</p>
    {/if}
  </aside>

  <!-- Full chat pane -->
  <main class="flex-1 flex flex-col min-h-0 overflow-hidden">
    {#if projectId}
      <AgentChat
        {projectId}
        fullScreen={true}
      />
    {:else}
      <div class="flex items-center justify-center h-full text-gray-400 text-sm">
        <a href="/auth" class="underline">Sign in</a> to use the agent.
      </div>
    {/if}
  </main>
</div>
```

The `fullScreen={true}` prop is already added to `AgentChat` in Phase 2 — no behavior change yet, just a flag for future use.

### 2.6 Add Chat Link to Sidebar Navigation

Modify `zen-sci-portal/src/routes/+layout.svelte`:

In the `navItems` array, add after the Workspace item:
```typescript
{ href: '/chat', label: 'Chat', icon: 'chat' },
```

The nav renders items with `{item.label}` — no icon rendering currently, so the `icon` field is cosmetic. Follow existing array order.

### 2.7 Provider Key Management — Go Gateway Endpoints

Add to the Go AgenticGateway:

**`POST /v1/settings/providers`** — Accepts `{"provider": "anthropic", "api_key": "sk-ant-..."}`. Validates provider name (must be one of: `anthropic`, `openai`). Stores key in secure storage and registers/refreshes the provider. Returns `{"provider": "<name>", "status": "configured"}`.

**`GET /v1/settings/providers`** — Returns list of providers with configured/unconfigured status. **Never** returns the key itself. Returns `{"providers": [{"name": "anthropic", "configured": true}, {"name": "openai", "configured": false}, {"name": "ollama", "configured": true, "auto": true}]}`.

Key storage: Use the gateway's existing secure storage mechanism (or the platform keychain available via the `KeyResolver` pattern in `BaseProvider`). Keys read at request time — no keys in logs or config files.

### 2.8 Provider Key Management — Rust (Tauri Side)

Add to `zen-sci-portal/src-tauri/src/gateway/client.rs`:

```rust
/// POST /v1/settings/providers — set a provider API key.
pub async fn set_provider_key(
    &self,
    token: &str,
    provider: &str,
    api_key: &str,
) -> Result<serde_json::Value, AppError> {
    let url = format!("{}/v1/settings/providers", self.base_url);
    let body = serde_json::json!({ "provider": provider, "api_key": api_key });
    self.post_json(token, &url, body).await
}

/// GET /v1/settings/providers — list provider configuration status.
pub async fn get_provider_settings(&self, token: &str) -> Result<serde_json::Value, AppError> {
    let url = format!("{}/v1/settings/providers", self.base_url);
    self.get_json(token, &url).await
}
```

Add to `zen-sci-portal/src-tauri/src/commands/gateway.rs`:

```rust
#[tauri::command]
pub async fn set_provider_key(
    state: State<'_, AppState>,
    provider: String,
    api_key: String,
) -> Result<serde_json::Value, String> {
    let token = get_token(&state).await?;
    state
        .gateway_client
        .set_provider_key(&token, &provider, &api_key)
        .await
        .map_err(|e| {
            tracing::error!("set_provider_key failed: provider={} error={}", provider, e);
            e.to_string()
        })
}

#[tauri::command]
pub async fn get_provider_settings(
    state: State<'_, AppState>,
) -> Result<serde_json::Value, String> {
    let token = get_token(&state).await?;
    state
        .gateway_client
        .get_provider_settings(&token)
        .await
        .map_err(|e| {
            tracing::error!("get_provider_settings failed: {}", e);
            e.to_string()
        })
}
```

Register both in `zen-sci-portal/src-tauri/src/lib.rs`.

### 2.9 Add IPC Wrappers for Provider Settings (TypeScript)

Add to `zen-sci-portal/src/lib/api/tauri.ts`:

```typescript
// ─── Provider Settings ────────────────────────────────────────────────────────

export interface ProviderSetting {
  name: string;
  configured: boolean;
  auto?: boolean;        // true for Ollama (auto-detected, no key needed)
}

export const setProviderKey = (provider: string, apiKey: string): Promise<{ provider: string; status: string }> =>
  invoke<{ provider: string; status: string }>('set_provider_key', { provider, apiKey });

export const getProviderSettings = (): Promise<{ providers: ProviderSetting[] }> =>
  invoke<{ providers: ProviderSetting[] }>('get_provider_settings');
```

### 2.10 Add Provider Key Management UI to Settings Page

Modify `zen-sci-portal/src/routes/settings/+page.svelte`:

**New state:**
```typescript
let providerSettings = $state<import('$lib/api/tauri.js').ProviderSetting[]>([]);
let showKeyDialog = $state<string | null>(null);  // provider name when dialog is open
let pendingApiKey = $state('');
let savingKey = $state(false);
let keyError = $state('');
```

**Load on mount:** Add to the existing `onMount` `Promise.all`:
```typescript
// Attempt to load provider settings (gateway must support this endpoint)
try {
  const settings = await getProviderSettings();
  providerSettings = settings.providers;
} catch {
  // Endpoint may not exist yet — graceful fallback
}
```

**Provider definitions (hardcoded list for UI):**
```typescript
const PROVIDER_DEFS = [
  { id: 'anthropic', displayName: 'Anthropic (Claude)', description: 'Requires ANTHROPIC_API_KEY' },
  { id: 'openai', displayName: 'OpenAI (GPT)', description: 'Requires OPENAI_API_KEY' },
];
```

**Add the "AI Providers" section card** below the existing "LLM Providers" card, following the same `card p-5 space-y-3` / `btn-primary` / `btn-secondary` Tailwind pattern:

```svelte
<!-- AI Provider Keys -->
<div class="card p-5 space-y-3">
  <h2 class="text-sm font-semibold text-gray-700 dark:text-gray-300">AI Provider Keys</h2>
  <p class="text-xs text-gray-400 dark:text-gray-500">
    Keys are stored securely on this device. Ollama runs locally and needs no key.
  </p>

  {#each PROVIDER_DEFS as provDef}
    {@const setting = providerSettings.find(p => p.name === provDef.id)}
    <div class="flex items-center gap-3 p-3 rounded-lg bg-gray-50 dark:bg-gray-900 border border-gray-200 dark:border-gray-800">
      <div class="flex-1">
        <p class="text-sm font-medium text-gray-700 dark:text-gray-200">{provDef.displayName}</p>
        <p class="text-xs text-gray-400 dark:text-gray-500">{provDef.description}</p>
      </div>
      {#if setting?.configured}
        <span class="text-xs text-emerald-500">● Configured</span>
        <button
          onclick={() => handleClearKey(provDef.id)}
          class="text-xs text-gray-400 hover:text-red-400 transition-colors"
        >
          Remove
        </button>
      {:else}
        <button
          onclick={() => { showKeyDialog = provDef.id; pendingApiKey = ''; keyError = ''; }}
          class="btn-secondary text-xs px-3 py-1"
        >
          Add Key
        </button>
      {/if}
    </div>
  {/each}

  <!-- Inline key entry dialog -->
  {#if showKeyDialog}
    {@const dialogProvDef = PROVIDER_DEFS.find(p => p.id === showKeyDialog)}
    <div class="p-4 rounded-lg border border-brand-200 dark:border-brand-800 bg-brand-50 dark:bg-brand-900/20 space-y-3">
      <p class="text-sm font-medium text-gray-700 dark:text-gray-300">
        Add key for {dialogProvDef?.displayName}
      </p>
      <input
        type="password"
        bind:value={pendingApiKey}
        placeholder="sk-..."
        class="input-field font-mono text-sm"
        aria-label="API key"
        autocomplete="off"
      />
      {#if keyError}
        <p class="text-xs text-red-500">{keyError}</p>
      {/if}
      <div class="flex gap-2">
        <button
          onclick={handleSaveKey}
          disabled={savingKey || !pendingApiKey.trim()}
          class="btn-primary text-xs px-3 py-1.5"
        >
          {savingKey ? 'Saving…' : 'Save Key'}
        </button>
        <button
          onclick={() => { showKeyDialog = null; pendingApiKey = ''; keyError = ''; }}
          class="btn-secondary text-xs px-3 py-1.5"
        >
          Cancel
        </button>
      </div>
    </div>
  {/if}
</div>
```

**Handler functions:**
```typescript
async function handleSaveKey() {
  if (!showKeyDialog || !pendingApiKey.trim()) return;
  savingKey = true;
  keyError = '';
  try {
    await setProviderKey(showKeyDialog, pendingApiKey.trim());
    // Refresh settings
    const updated = await getProviderSettings();
    providerSettings = updated.providers;
    showKeyDialog = null;
    pendingApiKey = '';
  } catch (err) {
    keyError = `Failed to save key: ${String(err)}`;
  } finally {
    savingKey = false;
  }
}

async function handleClearKey(providerId: string) {
  try {
    // Clear by sending empty key — gateway interprets this as removal
    await setProviderKey(providerId, '');
    const updated = await getProviderSettings();
    providerSettings = updated.providers;
  } catch {
    // Non-critical
  }
}
```

**Important security note:** The `<input type="password">` field ensures the key is not visible as the user types. Never log or display the key after entry. The `pendingApiKey` variable is cleared after save. Do not store the key in any Svelte store.

---

## 3. File Manifest

**Create:**
- `zen-sci-portal/src/lib/stores/pipeline.ts`
- `zen-sci-portal/src/routes/chat/+page.svelte`

**Modify:**
- `zen-sci-portal/src/routes/+layout.svelte` — add Chat nav item
- `zen-sci-portal/src/routes/settings/+page.svelte` — add Provider Keys section
- `zen-sci-portal/src/routes/pipelines/+page.svelte` — migrate jobs to store, fix disabled Run button, add step retry stubs, add "Open in Workspace" button, add step output preview
- `zen-sci-portal/src/lib/components/AgentChat.svelte` — add `pendingPipelineProposal`, proposal card UI, `submitProposedPipeline()`
- `zen-sci-portal/src/lib/types/index.ts` — add `PipelineProposal`, extend `AgentResponse`
- `zen-sci-portal/src/lib/api/tauri.ts` — add `setProviderKey`, `getProviderSettings`, `ProviderSetting`
- `zen-sci-portal/src-tauri/src/commands/gateway.rs` — add `set_provider_key`, `get_provider_settings` commands
- `zen-sci-portal/src-tauri/src/gateway/client.rs` — add `set_provider_key`, `get_provider_settings` methods
- `zen-sci-portal/src-tauri/src/models/gateway.rs` — add `pipeline_proposal: Option<serde_json::Value>` to `AgentResponse`
- `zen-sci-portal/src-tauri/src/lib.rs` — register `set_provider_key`, `get_provider_settings`

**Modify (Go AgenticGateway):**
- `server/handle_settings.go` (new file or append to existing) — `handleSetProviderKey`, `handleGetProviderSettings` handlers
- `server/router.go` (or equivalent) — register `POST /v1/settings/providers` and `GET /v1/settings/providers` routes
- Secure key storage integration (platform keychain or `KeyResolver` pattern)

**Do not modify:**
- `zen-sci-portal/src/lib/stores/agent.ts` — Phase 1/2 output; stable
- `zen-sci-portal/src/lib/components/MemoryPanel.svelte` — Phase 1 output; stable
- `zen-sci-portal/src/routes/workspace/[docId]/+page.svelte` — Phase 2 output; stable
- `zen-sci-portal/src-tauri/src/db/mod.rs` — Phase 1 output; stable
- Any `zen-sci/` MCP server files

---

## 4. Success Criteria

- [ ] Navigate to `/workspace/[docId]`, open chat, send messages, navigate to `/chat` route → same chat history visible (agent store persists across routes).
- [ ] `/chat` route shows context sidebar with "Open document" link when `agentStore.activeDocumentId` is set.
- [ ] Sidebar nav shows "Chat" link; clicking it navigates to `/chat` without full page reload.
- [ ] Settings page shows "AI Provider Keys" section with Anthropic and OpenAI rows.
- [ ] Click "Add Key" for Anthropic → inline key entry form appears → enter a key → click Save → row shows "● Configured" status.
- [ ] After adding an Anthropic key, create agent in workspace → agent can use Claude for responses (provider selection in AgentChat dropdown shows anthropic as available).
- [ ] Type "run the Write+PDF pipeline on this document" in AgentChat → (if gateway returns `pipeline_proposal`) proposal card appears with "Run Pipeline" button → click → DAG submitted → redirected to `/pipelines`.
- [ ] On `/pipelines`, "Run Pipeline" button is disabled when no steps are selected in the builder.
- [ ] Completed pipeline job shows "Open in Workspace" button; clicking it sets `pipelineStore.pipelineResult` and navigates to `/workspace`.
- [ ] Navigate away from `/pipelines` to `/memory` and back → job list is still visible (persisted in store).
- [ ] A step with output shows a collapsible "Output" section in the job detail view.
- [ ] Build compiles: `cargo build` in `zen-sci-portal/src-tauri/` passes with no errors.

---

## 5. E2E Validation Checklist (All v1.1 Success Criteria)

Run through all 10 success criteria from `docs/specs/zen-sci-portal-v1.1-spec.md` §2 in sequence:

- [ ] Open document → "summarize this document" → agent uses `get_document` → summary references actual section content
- [ ] "draft an Introduction" → prose response + PATCH_INTENT footer → Apply button → click Apply → section created
- [ ] "convert this to PDF" → `zen_sci_latex:convert_to_pdf` invoked via normalized tool loop → result visible
- [ ] Save agent response → memory counter increments immediately (no search required) → item tagged with `document_id`
- [ ] Restart app, open same document, new chat → agent's system prompt includes top-5 prior memory items for that document
- [ ] Navigate away from workspace and back → chat history intact (Svelte agent store)
- [ ] `/chat` route loads as full-screen chat with context sidebar
- [ ] Add Anthropic key in Settings → gateway registers provider → agent can use Claude
- [ ] "run Write+PDF pipeline" → DAG submitted → status visible in pipeline page
- [ ] Memory panel shows paginated list → click item → detail view with edit capability

---

## 6. Constraints & Non-Goals

- **DO NOT** implement OAuth provider login (e.g., GitHub Copilot) — not in v1.1 scope.
- **DO NOT** implement streaming/SSE responses — deferred to v1.2.
- **DO NOT** implement real-time collaborative editing — deferred.
- **DO NOT** add new MCP servers beyond the existing 6 — not in v1.1 scope.
- **DO NOT** add mobile or web portal support.
- **DO NOT** display API keys after entry. The `<input type="password">` is mandatory. Never put keys in Svelte stores.
- **DO NOT** hardcode API keys in any config file or source file.
- **DO NOT** introduce any new npm or Cargo dependencies.
- **DO NOT** modify any `zen-sci/` MCP server source files.
- **DO NOT** modify files outside the File Manifest.
- The `handleClearKey` clearing mechanism (sending empty string) is a v1.1 expedient — if the gateway rejects empty keys, implement a `DELETE /v1/settings/providers/:name` endpoint instead and update accordingly.
