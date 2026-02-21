# ZenSci Project Status

**Current Phase:** v1.1 Portal + Durability Patch — All Gaps Closed
**Last Updated:** 2026-02-20 (durability patch — agent registry, document sidecar sync, history re-hydration)

> For orientation (what, who, where) see **CONTEXT.md**. For decisions and architecture see **ARCHITECTURE.md**.

---

## ⚡ Durability Patch — SHIPPED (2026-02-20)

Three production-breaking gaps from the v1.1 verification closed in a single patch. All code compiles; `cargo test` 5/5; `svelte-check` 0 errors.

**Break 1 — Document sidecar (fixed):**
- `GET /v1/gateway/documents/:id` was 404ing for all real documents — gateway reads `~/.zen-sci/documents/<id>.json` but portal only wrote SQLite.
- Fix: `write_document_sidecar()` added to `commands/document.rs`, called after `create_document`, `create_section`, `update_section_content`, `update_document_content`. Sidecar is now written on every content-changing operation and stays in sync with the SQLite source of truth.

**Break 2 — Agent ID was ephemeral (fixed):**
- Gateway agents live in process memory only. Every gateway restart (OS sleep/wake, crash, app update) silently created a new bare agent — all agent configuration and context lost.
- Fix: `agents` table added to `~/.zen-sci/zensci.db`. `get_agent_for_project` / `upsert_agent` on `DocumentStore`. `GatewayClient::verify_agent()` calls `GET /v1/gateway/agents/:id`. Two new Tauri commands: `get_stored_agent`, `verify_and_persist_agent`.

**Break 3 — Chat history never seeded new agent (fixed):**
- When a new agent was forced (gateway death), the new agent started with empty context. `load_chat_history` existed but was never used during init.
- Fix: `agentStore.init()` now runs a 4-step protocol: check SQLite → verify alive at gateway → create fresh only if dead → re-hydrate with last 30 messages from SQLite as an amber system context card.

**Files changed:**
| File | Change |
| :--- | :--- |
| `src-tauri/src/commands/document.rs` | `write_document_sidecar()` + 4 callsites |
| `src-tauri/src/db/mod.rs` | `agents` table + `StoredAgent` + `get_agent_for_project` + `upsert_agent` |
| `src-tauri/src/gateway/client.rs` | `verify_agent()` method |
| `src-tauri/src/commands/gateway.rs` | `get_stored_agent` + `verify_and_persist_agent` commands |
| `src-tauri/src/lib.rs` | Register 2 new commands in `invoke_handler` |
| `src/lib/api/tauri.ts` | `StoredAgent` interface + `getStoredAgent` + `verifyAndPersistAgent` exports |
| `src/lib/stores/agent.ts` | Full 4-step durable init protocol |
| `src-tauri/tests/integration_test.rs` | Fixed pre-existing mock body mismatch in `test_gateway_client_list_providers_deserializes` |

---

## ⚡ Portal v1.1 — COMPLETE (2026-02-20)

Full v1.1 feature set shipped and verified against spec with live curl tests. 26 items confirmed with exact file:line evidence.

**v1.1 Features verified live:**
- `runAgentLoop()` — full LLM↔tool agentic loop, 8 max iterations (`handle_gateway.go:83`)
- `parsePatchIntent()` / `stripPatchIntent()` — fenced-block parser (`handle_gateway.go:52-72`)
- `PatchIntent` struct aligned across Go / TypeScript / Rust
- `DocumentContext` — `document_id` + `document_content` injected as `--- CURRENT DOCUMENT ---` block
- Provider normalization: Anthropic `tool_use`, OpenAI `tool_calls`, Ollama `modelSupportsTools()` + text fallback
- Memory durability confirmed: items survive gateway restarts
- `GET /v1/memory`, `PUT /v1/memory/:id`, `POST/GET /v1/settings/providers` all live
- `GET /v1/settings/providers` returns `{"providers":{"anthropic":true}}`
- Persistent chat (`save_chat_message` / `load_chat_history`) wired to SQLite
- Memory panel (paginated list, detail, editing) — `MemoryPanel.svelte`
- Provider key management — `set_provider_key` writes `~/.zen-sci/provider_keys.json`
- Full patch intent UI (amber confirmation card, Apply/Dismiss, `applypatch` custom event)
- Memory-as-history init from `searchMemory(documentTitle ?? projectId, projectId)`
- Agent store with `init`, `setActiveDocument`, `setPriorContext`, `addMessage`, `clearMessages`, `reset`
- Pipeline store (`jobs`, `submit()`, polling) — exists; pipelines page uses local state (intentional divergence)

**3 intentional spec divergences (catalogued, not gaps):**
- `PatchIntent` fields: spec said `section_title + action`, shipped `operation + section_id + content + description` (richer)
- Document context: spec said section index only, shipped full `document_content` text dump
- Memory search seed: spec said `document:{docId}` prefix, shipped free-form `documentTitle ?? projectId`

---

## ⚡ Three-Tier Stack — LIVE (2026-02-19)

```
zen-sci-portal (SvelteKit UI)
    → Tauri invoke_handler commands (Rust)
        → GatewayClient HTTP → POST /v1/tools/:name/invoke
            → AgenticGateway (Go, :8080, binary at binaries/agentic-gateway)
                → zen-sci MCP servers (Node.js, esbuild bundles, stdio bridge)
```

**Confirmed live (2026-02-20):**
- Gateway `/health` returns `{"status":"healthy","version":"1.1.0"}` (binary rebuilt 13:15 Feb 20)
- 6 MCP servers connected, 17 zen_sci tools registered (49 total including gateway tools)
- `zen_sci_latex:convert_to_pdf` invoked end-to-end — `isError: false`, 85ms
- Memory durable across gateway restarts (confirmed with durability test item)

---

## 1. Current State

All Phases 0–4 (MCP suite) + Portal v1.1 + Durability Patch implemented and green.

| Area | Status | Notes |
| :--- | :--- | :--- |
| **packages/core** | ✅ | 129 tests, 91% stmt coverage |
| **packages/sdk** | ✅ | 68 tests, 85% coverage |
| **servers/latex-mcp** | ✅ | 37 tests |
| **servers/blog-mcp** | ✅ | 76 tests |
| **servers/grant-mcp** | ✅ | 50 tests |
| **servers/slides-mcp** | ✅ | 34 tests |
| **servers/newsletter-mcp** | ✅ | 63 tests |
| **servers/paper-mcp** | ✅ | 36 tests |
| **Phase 4 (MCP Apps)** | ✅ | 6 companion HTML bundles at `ui://` URIs |
| **zen-sci-portal v1.1** | ✅ | Tauri v2 + SvelteKit 5 + Rust. Agentic loop, patch intent, memory panel, provider key mgmt. 5/5 Rust tests. |
| **Durability patch** | ✅ | Sidecar sync, agent registry (SQLite), history re-hydration. Cargo + svelte-check clean. |
| **Gateway deserialization audit** | ✅ | All 9 client methods verified against Go source. 7 bugs fixed. 8 tests added. |

---

## 2. Active Workstreams

1. ✅ **System deps** — pandoc 3.9 + TeX Live 2025 installed
2. ✅ **Python deps** — sympy, pypandoc, bibtexparser in `zen-sci/.venv`
3. ✅ **CI pipeline** — GitHub Actions on every push/PR
4. ✅ **Three-tier stack** — live, smoke tested
5. ✅ **Portal v1.1** — full feature set shipped
6. ✅ **Durability patch** — all three production breaks closed
7. 📋 **Binary PDF verification** — confirm pdflatex step completes with `.venv` active in gateway env
8. 📋 **v1.2 planning** — streaming (SSE), `search_memory` as registered tool, `pipeline_proposal` field

---

## 3. Blockers & Dependencies

**No critical blockers.**

| Decision | Impact | Owner | Status |
| :--- | :--- | :--- | :--- |
| Binary PDF verification | Medium | Cruz | Run with `.venv` active; confirm `pdf_base64` in response |
| Trademark/domain availability | Medium | Operations | Check ZenSci, ZenithScience.org, @zen-sci before v0.1 release |
| v1.2 scope decision | Low | Cruz | Streaming vs. tool vs. pipeline_proposal — pick one focus |

---

## 4. Next Steps

**Immediate (P1):**
1. **Binary PDF verification** — activate `.venv` before starting the gateway, re-run smoke test, confirm `pdf_base64` field is present
   ```bash
   source zen-sci/.venv/bin/activate
   # then start gateway and invoke zen_sci_latex:convert_to_pdf
   ```
2. **`patch_intent` from Ollama** — qwen3:8b doesn't reliably emit the fenced block. Prompt engineering fix: add an explicit example of the fenced JSON format to the system prompt in `handleGatewayAgentChat()`.

**Near-term (P2 — v1.2 candidates):**
1. **Streaming (SSE)** — biggest UX gap; chat feels frozen during Ollama completions
2. **`search_memory` as a registered tool** — agent can self-query memory during the agentic loop
3. **`pipeline_proposal` field** — agent proposes a DAG; user confirms; submit to orchestration

**Backlog (P3):**
1. Replace manual `export ZEN_SCI_SERVERS_ROOT=...` with `.env.local` / `direnv`
2. Add TypeDoc/TSDoc auto-generated API reference
3. Align SDK package version with server versioning scheme
4. MCP registry namespace decision (`io.github.trespies-source`)

---

## 5. Module Roadmap

| Module | Output | Phase | Status | Tests |
| :--- | :--- | :--- | :--- | :--- |
| **latex-mcp** | LaTeX + PDF | v0.1 | ✅ | 37 |
| **blog-mcp** | HTML blog | v0.2 | ✅ | 76 |
| **slides-mcp** | Beamer + Reveal.js | v0.3 | ✅ | 34 |
| **newsletter-mcp** | MJML + HTML email | v0.3 | ✅ | 63 |
| **grant-mcp** | LaTeX / Word | v0.4 | ✅ | 50 |
| **paper-mcp** | IEEE/ACM/arXiv LaTeX | v0.5 | ✅ | 36 |

**Portal versions:**

| Version | Status | Key feature |
| :--- | :--- | :--- |
| v1.0 | ✅ | Three-tier stack live, tool invocation, SQLite persistence |
| v1.1 | ✅ | Agentic loop, patch intent, memory panel, provider key mgmt, document context |
| v1.1 patch | ✅ | Durable agent registry, document sidecar sync, history re-hydration |
| v1.2 | 📋 | Streaming SSE, search_memory tool, pipeline_proposal |

---

## 6. Key Metrics & Constraints

- All 6 MCP servers implemented; 493 tests passing across 8 packages
- Portal: 5/5 Rust tests green; 0 TypeScript errors; 0 svelte-check errors
- Team size: Solo (Cruz); auditing/scouting support available
- System deps installed: pandoc 3.9, TeX Live 2025, sympy/pypandoc/bibtexparser in venv
- CI: GitHub Actions enforces lint + typecheck + test + build on every push/PR
- All durable state lives under `~/.zen-sci/` — survives Tauri bundle updates

**Quality Targets:**
- LaTeX output must be publication-ready
- All formats must preserve semantic meaning (math, citations, structure)
- Portal must be durable across OS sleep/wake, gateway restarts, and app bundle updates

---

## 7. Risk Assessment

| Risk | Probability | Impact | Mitigation |
| :--- | :--- | :--- | :--- |
| Core abstraction is too LaTeX-specific | Medium | High | blog-mcp v0.2 validated generality |
| Python engine subprocess bridge is fragile | Medium | Medium | Test error handling extensively |
| Math rendering varies too much across formats | Low | Medium | SymPy + MathML + target-format conversion in core |
| Trademark/domain not available | Low | Medium | Check before v0.1 release; have backup names ready |
| `patch_intent` unreliable on weak Ollama models | Medium | Low | Architecture is correct; prompt engineering fix in v1.2 |

---

## 8. Success Criteria (v0 Suite)

- ✅ **v0.1 (latex-mcp)** — Markdown with math/citations → publication-ready PDF
- ✅ **v0.2 (blog-mcp)** — Same source → HTML blog
- ✅ **v0.3 (slides-mcp + newsletter-mcp)** — Same source → Beamer/Reveal.js + email
- ✅ **v0.4 (grant-mcp)** — Same source → grant proposal
- ✅ **Core Quality** — All 5 formats from 1 markdown source; no format-specific rewrites
- ✅ **Portal v1.1** — Agentic loop live; document-grounded agent chat; patch intent UI
- ✅ **Durability** — All durable state in `~/.zen-sci/`; survives gateway restarts and Tauri updates

---

## 9. Health Audit (2026-02-20)

**Last full report:** `audits/2026-02-20_durability_audit.md`

| Dimension | Status |
| :--- | :--- |
| Critical Issues | 🟢 GREEN — All three durability breaks closed |
| Security | 🟢 GREEN — No secrets in repo; provider keys in plaintext local file (documented) |
| Testing | 🟢 GREEN — 493 MCP + 5 Rust integration + 8 gateway unit; CI on every push/PR |
| Technical Debt | 🟢 GREEN — Gateway deser audit complete; durability patch complete; 0 silent failures |
| Documentation | 🟢 GREEN — STATUS, ARCHITECTURE, CONTEXT, audit, handoff all current as of 2026-02-20 |

**Top remaining action:** Binary PDF verification (`.venv` active when starting gateway).

---

## 10. Notes for Future Audits

- Re-audit after v1.2 — verify streaming doesn't break the patch intent extraction pipeline
- Check `agents` table growth — add `last_seen_at` housekeeping if > 100 projects
- Quarterly thinking-partnership review — is ZenSci meeting the "augment human reasoning" thesis?
- Forward compatibility: when Phase 2 modules ship, re-run cluster analysis against implementation
