# Next Steps — Implementation Prompts
**Project:** zen-sci-portal × AgenticGateway × zen-sci
**Date:** 2026-02-19
**Context:** P0/P1 bugs fixed. invoke_tool wired. Gateway registered with 6 zen-sci MCP servers. E2e plan written. These are the ready-to-execute next tracks.

---

## Track 1 — P1: test_invoke_tool_smoke (Integration Test)
**File:** `zen-sci-portal/src-tauri/tests/integration_test.rs`
**Effort:** ~30 min. No live Gateway needed — uses mockito.

### What to do

Add a fourth integration test `test_invoke_tool_smoke` immediately after `test_gateway_client_list_providers_deserializes`. The test must:

1. Spin up a mockito server
2. Register a mock for `POST /v1/tools/zen_sci_latex:convert_to_pdf/invoke` returning a valid `ToolInvocationResult` JSON shape
3. Construct a `GatewayClient` pointed at the mockito URL
4. Call `client.invoke_tool("test-token", "zen_sci_latex:convert_to_pdf", inputs).await`
5. Assert `result.tool_name == "zen_sci_latex:convert_to_pdf"`, `result.status == "success"`, `result.duration_ms >= 0`
6. Assert `result.output` contains the expected key (e.g., `"pdf_path"`)
7. Call `_mock.assert_async().await` to confirm the mock was hit exactly once

### Exact mock response shape

```json
{
  "tool_name": "zen_sci_latex:convert_to_pdf",
  "inputs": { "content": "\\documentclass{article}\\begin{document}Hello\\end{document}", "filename": "smoke.pdf" },
  "output": { "pdf_path": "/tmp/smoke.pdf", "page_count": 1 },
  "duration_ms": 312,
  "status": "success"
}
```

### What to verify in the test

```rust
assert_eq!(result.tool_name, "zen_sci_latex:convert_to_pdf");
assert_eq!(result.status, "success");
assert!(result.duration_ms >= 0);
assert!(result.output.get("pdf_path").is_some());
_mock.assert_async().await;
```

### Also: add a second test for error case

Add `test_invoke_tool_error_shape` immediately after the smoke test. Mock the same endpoint returning HTTP 500 with a JSON error body. Assert that `client.invoke_tool()` returns an `Err(...)` and the error string is non-empty. This validates that `AppError::SerializationError` or gateway HTTP error propagates correctly.

### Imports to add at the top of the test block

```rust
use zen_sci_portal_lib::gateway::GatewayClient;
// (mockito::Server already used in test_auth_flow above)
```

---

## Track 2 — P2: Svelte Reactivity Fixes
**Files:**
- `zen-sci-portal/src/lib/components/ResultsList.svelte`
- `zen-sci-portal/src/lib/components/SectionOrganizer.svelte`
**Effort:** ~20 min. Surgical — two lines in each file.

### Fix 1: `ResultsList.svelte` — containerRef missing `$state()`

**Line 15, current:**
```svelte
let containerRef: HTMLDivElement | undefined;
```

**Line 15, fix:**
```svelte
let containerRef = $state<HTMLDivElement | undefined>(undefined);
```

**Why:** In Svelte 5's rune system, variables used with `bind:this` must be declared with `$state()` for the binding to participate in the reactivity graph. Without it, the binding is inert — `containerRef` is set by the DOM engine but never triggers downstream updates. If any future code reads `containerRef` reactively (e.g., for measuring height or scrolling programmatically), it will silently get `undefined`.

### Fix 2: `SectionOrganizer.svelte` — sections doesn't re-sync when document prop refreshes

**Current (line 14):**
```svelte
let sections = $state<Section[]>([...document.sections]);
```

This is correct for initial render but the local `sections` state is a one-time copy. If the parent component passes a refreshed `document` prop after a save (e.g., after `transitionToStructured` succeeds and the parent re-fetches), the modal will display stale section data.

**Fix — add an `$effect` immediately after the `$state` declaration (after line 14):**
```svelte
let sections = $state<Section[]>([...document.sections]);

// Re-sync when the parent document prop changes (e.g. after an external save).
// Guard: only reset if not mid-drag, to avoid interrupting in-progress reorders.
$effect(() => {
    if (dragIndex === null) {
        sections = [...document.sections];
    }
});
```

**Why:** The `$effect` runs whenever `document.sections` changes (Svelte 5 tracks it as a dependency). The `dragIndex === null` guard prevents a mid-drag reset if the parent happens to refresh at the same time. Without this, the organizer silently shows old section order / old section count if the document was modified externally between modal open and confirm.

---

## Track 3 — P2: Tauri Resource Bundling of zen-sci Dist Files
**Files:**
- `zen-sci-portal/src-tauri/tauri.conf.json`
- `zen-sci-portal/src-tauri/src/state.rs` (or wherever the sidecar is launched)
**Effort:** ~1 hour. Requires all 6 zen-sci servers to have `npm run build` dist files present.

### What to do

1. **Build all zen-sci servers to dist:**
   ```bash
   cd zen-sci/servers
   for s in latex-mcp blog-mcp slides-mcp newsletter-mcp grant-mcp paper-mcp; do
     (cd $s && npm install && npm run build)
   done
   ```

2. **Add zen-sci dist files as Tauri resources in `tauri.conf.json`:**
   ```json
   "bundle": {
     "resources": {
       "../zen-sci/servers/latex-mcp/dist": "zen-sci-servers/latex-mcp/dist",
       "../zen-sci/servers/blog-mcp/dist": "zen-sci-servers/blog-mcp/dist",
       "../zen-sci/servers/slides-mcp/dist": "zen-sci-servers/slides-mcp/dist",
       "../zen-sci/servers/newsletter-mcp/dist": "zen-sci-servers/newsletter-mcp/dist",
       "../zen-sci/servers/grant-mcp/dist": "zen-sci-servers/grant-mcp/dist",
       "../zen-sci/servers/paper-mcp/dist": "zen-sci-servers/paper-mcp/dist"
     }
   }
   ```

3. **Set `ZEN_SCI_SERVERS_ROOT` at portal launch:**
   In the Tauri setup closure in `lib.rs` (or in the sidecar launch code), resolve the resource directory and set the env var before starting the Gateway sidecar:
   ```rust
   // In setup closure, before starting the Gateway sidecar:
   let resource_dir = app.path().resource_dir()
       .expect("resource dir must be resolvable");
   let zen_sci_root = resource_dir.join("zen-sci-servers");
   std::env::set_var("ZEN_SCI_SERVERS_ROOT", zen_sci_root);
   ```

4. **Verify the Gateway can find servers:**
   After Tauri bundle, confirm the Gateway's `mcp/bridge.go` expands `${ZEN_SCI_SERVERS_ROOT}` correctly in the `args` path from `gateway-config.yaml`.

### Notes
- `node_modules` should NOT be included in the bundle — only `dist/`. Each zen-sci server bundles its dependencies via esbuild/tsup into a single `dist/index.js`.
- The path structure inside the Tauri bundle must exactly match what `gateway-config.yaml` expects after `ZEN_SCI_SERVERS_ROOT` is substituted.

---

## Track 4 — Future: invoke_tool UI Surface (Frontend)
**Files:** New component(s) in `zen-sci-portal/src/lib/components/`
**Effort:** ~2–3 hours. Depends on Track 3 (resource bundling) and a live Gateway.
**Prerequisite:** Auth flow must be live (real JWT from Gateway, not mocked).

### What to build

A `ToolInvoker.svelte` component (or integrate into the existing document editor) that:

1. Calls `list_tools()` on mount to fetch the tool registry
2. Renders a tool selector — namespace group (`zen_sci_latex`, `zen_sci_blog`, etc.) + tool name
3. Accepts a JSON inputs textarea (or per-field form if schema is known)
4. Calls `invoke("invoke_tool", { tool_name, inputs })` on submit
5. Displays the `ToolInvocationResult`:
   - `status` badge (success = green, error = red)
   - `duration_ms` in a subtle annotation
   - `output` rendered as JSON or a file download link if `output.pdf_path` / `output.html_path` etc.
6. Shows an error state if the invoke rejects

### Tauri API call (in `$lib/api/tauri.ts`)

```typescript
import { invoke } from '@tauri-apps/api/core';
import type { ToolInvocationResult } from '$lib/types/index.js';

export async function invokeTool(
    toolName: string,
    inputs: Record<string, unknown>
): Promise<ToolInvocationResult> {
    return invoke<ToolInvocationResult>('invoke_tool', {
        toolName,   // camelCase — Tauri serializes to snake_case for Rust
        inputs,
    });
}
```

Note: Tauri's `invoke` bridge expects camelCase JS → snake_case Rust parameter names by default. Confirm the exact serialization behavior with a quick test before building the full component.

### Type to add to `$lib/types/index.ts`

```typescript
export interface ToolInvocationResult {
    tool_name: string;
    inputs: Record<string, unknown>;
    output: Record<string, unknown>;
    duration_ms: number;
    status: string;
}
```

---

## Priority Order

| Track | Priority | Blocker | Can start now? |
|---|---|---|---|
| 1 — test_invoke_tool_smoke | P1 | None | ✅ Yes |
| 2 — Svelte reactivity fixes | P2 | None | ✅ Yes |
| 3 — Tauri resource bundling | P2 | zen-sci npm builds | Needs npm build first |
| 4 — invoke_tool UI surface | Future | Auth live + Track 3 | No |
