# E2E Preparedness Plan — zen-sci-portal × AgenticGateway × zen-sci

**Date:** 2026-02-19
**Status:** Wiring complete — first integrated run ready to attempt

---

## 1. What the Full Chain Looks Like

```
SvelteKit UI
   │  invoke("invoke_tool", { tool_name, inputs })
   ▼
Tauri command: invoke_tool                  [commands/gateway.rs]
   │  get_token() → AuthManager → keychain
   │  GatewayClient::invoke_tool(token, tool_name, inputs)
   ▼
POST http://127.0.0.1:8080/v1/tools/:name/invoke   [AgenticGateway sidecar]
   │  JWT validation → route to handleInvokeTool
   │  MCP bridge → spawn Node.js server via stdio
   ▼
zen-sci MCP server   e.g. latex-mcp/dist/index.js
   │  registerAppTool handler runs
   │  returns { tool_name, inputs, output, duration_ms, status }
   ▼
ToolInvocationResult deserialized in Rust    [models/gateway.rs]
   │  returned to SvelteKit UI
   ▼
Rendered result
```

**Fully-qualified tool name format:** `{namespace}:{tool_name}`

| Server | Namespace | Primary tool |
|---|---|---|
| latex-mcp | `zen_sci_latex` | `zen_sci_latex:convert_to_pdf` |
| blog-mcp | `zen_sci_blog` | `zen_sci_blog:convert_to_html` |
| slides-mcp | `zen_sci_slides` | `zen_sci_slides:convert_to_slides` |
| newsletter-mcp | `zen_sci_newsletter` | `zen_sci_newsletter:convert_to_email` |
| grant-mcp | `zen_sci_grant` | `zen_sci_grant:generate_proposal` |
| paper-mcp | `zen_sci_paper` | `zen_sci_paper:convert_to_paper` |

---

## 2. What Is Now Wired (Completed This Session)

| Layer | File | Change |
|---|---|---|
| Model | `src/models/gateway.rs` | Added `ToolInvocationResult` struct |
| HTTP client | `src/gateway/client.rs` | Added `invoke_tool()` method → `POST /v1/tools/:name/invoke` |
| Tauri command | `src/commands/gateway.rs` | Added `invoke_tool` command with `tracing::debug` + `tracing::error` |
| Registration | `src/lib.rs` | Imported + registered in `invoke_handler!` |

Previously wired (prior session or pre-existing):

- `GatewayClient::list_tools()` → `GET /v1/gateway/tools`
- `AuthManager` full login/check/logout cycle
- `gateway-config.yaml` entries for all 6 zen-sci MCP servers
- Gateway sidecar startup/health monitoring

---

## 3. Prerequisites for First Integrated Run

### 3a. Gateway binary + zen-sci server dist files must be present

The Gateway is bundled as `binaries/agentic-gateway` (Tauri `externalBin`). The zen-sci Node.js dist files need to be either:

- Bundled as Tauri resources (production path — Option A, confirmed), **or**
- Available at `ZEN_SCI_SERVERS_ROOT` env var (dev path)

For the first integrated smoke test, the simplest path is dev mode:

```bash
# In terminal 1: build all zen-sci servers
cd zen-sci/servers
for s in latex-mcp blog-mcp slides-mcp newsletter-mcp grant-mcp paper-mcp; do
  (cd $s && npm install && npm run build)
done

# Set the env var so Gateway can find them
export ZEN_SCI_SERVERS_ROOT="$(pwd)"
```

### 3b. Gateway must be running and healthy

```bash
# Start the Gateway binary directly for dev
./binaries/agentic-gateway --config AgenticGatewayByDojoGenesis/gateway-config.yaml

# Verify health
curl http://127.0.0.1:8080/health
```

Expected: `{"status":"ok","version":"...","uptime_ms":...}`

### 3c. A valid JWT must be obtainable

The portal's auth flow hits `POST /auth/login` on the Gateway. If Gateway auth endpoints are not yet live, `invoke_tool` will fail at `get_token()` with `"No active session"`.

**Workaround for first smoke test:** Mint a dev JWT directly and inject via keychain, or temporarily bypass `get_token()` with a hardcoded dev token in a local branch.

### 3d. Tool input schema

Each tool expects specific inputs. For smoke testing `zen_sci_latex:convert_to_pdf`:

```json
{
  "tool_name": "zen_sci_latex:convert_to_pdf",
  "inputs": {
    "content": "\\documentclass{article}\\begin{document}Hello World\\end{document}",
    "filename": "test.pdf"
  }
}
```

Consult each server's `src/index.ts` → `registerAppTool(server, { name, schema })` for the exact schema.

---

## 4. E2E Test Strategy

### Phase 1: Manual smoke test (immediate)

1. Start Gateway with zen-sci config
2. Verify `GET /v1/gateway/tools` returns all zen-sci tools
3. Manually `curl POST /v1/tools/zen_sci_latex:convert_to_pdf/invoke` with a minimal LaTeX body
4. Confirm `{ status: "success", output: {...}, duration_ms: ... }` shape matches `ToolInvocationResult`

### Phase 2: Rust integration test

Add `test_invoke_tool_smoke` to `tests/integration_test.rs`:

```rust
// Sketch — add after test_auth_flow
#[tokio::test]
async fn test_invoke_tool_smoke() {
    // mockito server mimicking Gateway's /v1/tools/:name/invoke response
    let mut server = Server::new_async().await;
    let _mock = server
        .mock("POST", "/v1/tools/zen_sci_latex:convert_to_pdf/invoke")
        .with_status(200)
        .with_header("content-type", "application/json")
        .with_body(r#"{
            "tool_name": "zen_sci_latex:convert_to_pdf",
            "inputs": {"content": "\\documentclass{article}..."},
            "output": {"pdf_path": "/tmp/test.pdf"},
            "duration_ms": 450,
            "status": "success"
        }"#)
        .create_async().await;

    let client = GatewayClient::new(server.url(), reqwest::Client::new());
    let result = client.invoke_tool(
        "test-token",
        "zen_sci_latex:convert_to_pdf",
        serde_json::json!({"content": "\\documentclass{article}...", "filename": "test.pdf"}),
    ).await.unwrap();

    assert_eq!(result.tool_name, "zen_sci_latex:convert_to_pdf");
    assert_eq!(result.status, "success");
    assert!(result.duration_ms >= 0);
    _mock.assert_async().await;
}
```

### Phase 3: Live integration test (once Gateway + zen-sci are running)

Run against a real local Gateway instance. Tag with `#[cfg(feature = "live_integration")]` so they don't run in CI without the sidecar present.

### Phase 4: Tauri e2e (Playwright/WebdriverIO)

Once a UI surface calls `invoke_tool`, add a Playwright test that:

1. Renders a document
2. Triggers export to PDF
3. Asserts the result panel shows a success response

---

## 5. Observability Anchors

The `invoke_tool` Tauri command emits structured logs at two levels:

```
DEBUG  invoke_tool: tool=zen_sci_latex:convert_to_pdf inputs={...}
ERROR  invoke_tool failed: tool=zen_sci_latex:convert_to_pdf error=...
```

The Gateway's `handleInvokeTool` records `duration_ms` on every response — this flows back through `ToolInvocationResult.duration_ms` to the UI and is available for display, metrics, and tracing.

For full trace visibility, the Gateway's `gateway-config.yaml` entries include:

```yaml
timeouts:
  startup: 15
  tool_default: 120    # LaTeX servers
  health_check: 5
health_check:
  enabled: true
  interval_sec: 60
retry_policy:
  max_attempts: 2
```

Any tool invocation that exceeds `tool_default` will be surfaced as a timeout error in `ToolInvocationResult.status`.

---

## 6. Open Items / Ownership

| Item | Priority | Notes |
|---|---|---|
| `test_invoke_tool_smoke` integration test | P1 | Can be written now (mockito, no live Gateway needed) |
| Gateway auth endpoint live (`POST /auth/login`) | P1 | Blocks real token flow; dev bypass needed until live |
| zen-sci server dist builds in CI | P1 | Required for any live integration test |
| Tauri resource bundling of zen-sci dist files | P2 | Production packaging; dev uses `ZEN_SCI_SERVERS_ROOT` |
| UI surface calling `invoke_tool` | P2 | Frontend work — not yet started |
| P2 Svelte reactivity bugs | P2 | `ResultsList.svelte` containerRef, `SectionOrganizer.svelte` sections |
| Live Playwright e2e test | P3 | Requires UI surface + live Gateway |

---

## 7. Summary: What "First Integrated Run" Requires

The Rust wiring is complete. The shortest path to a successful end-to-end invocation is:

1. Build zen-sci server dist files
2. Set `ZEN_SCI_SERVERS_ROOT`
3. Start the Gateway with `gateway-config.yaml`
4. Obtain or mint a valid JWT
5. `curl POST /v1/tools/zen_sci_latex:convert_to_pdf/invoke`

Everything between that curl and the SvelteKit UI is now wired. The gap is the runtime environment, not the code.
