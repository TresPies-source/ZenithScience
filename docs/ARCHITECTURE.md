# ZenSci Architecture & Decisions

**Owner:** TresPiesDesign.com / Cruz Morales
**Last updated:** 2026-02-20

This file is the canonical record of all architecture decisions, the semantic cluster model, and the specification audit trail. If a decision is recorded here and somewhere else, this file wins.

---

## Core Principles

- **Quality over speed** — Solve hard problems (LaTeX, math, proof structures) first; everything else follows
- **Shared infrastructure first** — Build core parsing, citation, validation layers once; reuse across all modules
- **AI as thinking partner** — Augment human reasoning before rendering, not replace human judgment
- **Multi-format coherence** — The same source document should render identically across LaTeX, HTML, email, slides
- **Durable state at the root** — All user data lives under `~/.zen-sci/`; app bundle updates must never lose it

---

## Architecture Decisions

| # | Decision | Date | Detail |
|---|----------|------|--------|
| 1 | **IP Ownership** | 2026-02-17 | TresPiesDesign.com / Cruz Morales own all IP |
| 2 | **Product Name** | 2026-02-17 | ZenSci (product), ZenithScience.org (domain) |
| 3 | **GitHub Org & Repo** | 2026-02-17 | TresPies-source/ZenithScience (monorepo, not "typecraft") |
| 4 | **Working Directory** | 2026-02-17 | `zen-sci/` (kebab-case) for code; `ZenithScience/` root for decision artifacts |
| 5 | **npm Scope** | 2026-02-17 | `@zen-sci` (not `@typecraft`) |
| 6 | **Monorepo Structure** | 2026-02-17 | Shared core library + independent MCP servers (not separate repos) |
| 7 | **Full Spec Pass** | 2026-02-18 | 19 specs written across 3 phases + infrastructure in a single parallel pass |
| 8 | **SDK Pattern** | 2026-02-18 | Composition over inheritance: `createZenSciServer()` factory + standalone utilities, not `ZenSciServer` abstract base class |
| 9 | **Strict Enums** | 2026-02-18 | All union types (OutputFormat, BibliographyStyle, aspectRatio, SubmissionFileRole) are strict — no `\| string` escape hatches |
| 10 | **Error Identity Split** | 2026-02-18 | `ConversionErrorData` (wire interface) + `ConversionError` (runtime class) with `fromData()` / `toData()` bridge |
| 11 | **Module Shipping Order** | 2026-02-18 | latex-first. Sequence: latex → blog → slides + newsletter (v0.3) → grant (v0.4) → paper (v0.5). Cruz confirmed. |
| 12 | **Open-Source License** | 2026-02-18 | Apache 2.0. Patent grant protects institutional users (NIH/NSF); trademark clause protects ZenSci/ZenithScience brand. |
| 13 | **MCP Apps — Phase 4** | 2026-02-18 | Companion apps using `@modelcontextprotocol/ext-apps` SDK (v1.0.1). Each module gets `app/` sibling. Vite + `vite-plugin-singlefile` single-file HTML bundles at `ui://` URIs. |
| 14 | **PatchIntent field schema** | 2026-02-20 | Spec said `section_title + action`. Shipped `operation + section_id + content + description` (richer, more actionable). All three tiers aligned: Go struct, Rust model, TypeScript interface. |
| 15 | **Document context in chat** | 2026-02-20 | Spec said send section index only. Shipped full `document_content` text dump with `document_id`. Gateway injects it as `--- CURRENT DOCUMENT (id: %s) ---` block in the system prompt. Richer context wins. |
| 16 | **Memory search seed** | 2026-02-20 | Spec said use `document:{docId}` prefix query. Shipped free-form `documentTitle ?? projectId` query. More natural; the gateway semantic search handles it. |
| 17 | **Durable agent registry** | 2026-02-20 | Agent IDs are persisted in `agents` table in `~/.zen-sci/zensci.db` (one row per project). On init, portal verifies agent is still live at the gateway before reusing. On gateway death, portal creates a new agent and re-hydrates from the last 30 SQLite chat messages. Provider keys stored in `~/.zen-sci/provider_keys.json` (Rust side writes it directly; not via gateway endpoint). |
| 18 | **Document sidecar sync** | 2026-02-20 | `GET /v1/gateway/documents/:id` reads `~/.zen-sci/documents/<id>.json`. Portal writes this sidecar on every content-changing operation (create, section add/update, draft update). SQLite is the source of truth; sidecar is a derived, always-current copy for gateway reads. |

---

## Semantic Architecture (Behavioral Clusters)

The system comprises 9 behavioral clusters spanning all 6 implemented modules. This reveals where shared infrastructure pays dividends and where modules must diverge.

**Tier 1 — Shared Across All Modules (core library):**

1. **Parsing** — Markdown → AST; frontmatter extraction
2. **Citation & References** — BibTeX/CSL parsing, cross-reference resolution
3. **Validation** — Schema validation, dead-link detection, math syntax checking

**Tier 2 — Format-Specific (module variants):**

4. **Structure** — Document tree (outline, sections, hierarchy)
5. **Conversion** — Target format code generation (LaTeX, HTML, email, etc.)
6. **Math Rendering** — TeX math → target format (PDF, MathML, SVG)
7. **Bibliography Rendering** — CSL styles → target format
8. **Configuration** — Module-specific settings
9. **Rendering & Output** — Final artifact generation and validation

**Architectural Insight:** Clusters 1, 2, 3 must be shared (duplication cost is high). Clusters 4–9 have module-specific variants but inherit from core abstractions.

**Full cluster analysis:** `architecture/2026-02-17_semantic-clusters.md`

---

## Portal Three-Tier Stack Architecture

```
zen-sci-portal (SvelteKit 5 + Tauri v2)
│
├── src/lib/stores/agent.ts          ← 4-step durable init: SQLite → verify → create → re-hydrate
├── src/lib/api/tauri.ts             ← typed IPC wrapper for all Tauri commands
├── src/lib/components/AgentChat.svelte
│
└── src-tauri/ (Rust)
    ├── commands/document.rs         ← write_document_sidecar() on every content write
    ├── commands/gateway.rs          ← chat_with_agent, get_stored_agent, verify_and_persist_agent
    ├── db/mod.rs                    ← SQLite: documents + sections + chat_messages + agents
    ├── gateway/client.rs            ← HTTP: verify_agent(), chat_with_agent(), list_tools(), etc.
    └── config.rs                    ← data_dir = ~/.zen-sci/; gateway_url = 127.0.0.1:8080
            ↓
AgenticGateway (Go, :8080)
    ├── handle_gateway.go            ← runAgentLoop(), parsePatchIntent(), handleGetDocument()
    ├── router.go                    ← /health, /v1/gateway/agents/:id, /v1/gateway/documents/:id
    └── services/providers/
        ├── anthropic.go             ← tool_use block format
        ├── openai_compat.go         ← tool_calls format
        └── ollama.go                ← modelSupportsTools() + text fallback
            ↓
zen-sci MCP servers (Node.js, esbuild bundles)
    latex-mcp, blog-mcp, grant-mcp, slides-mcp, newsletter-mcp, paper-mcp
```

**Durable storage layout (`~/.zen-sci/`):**

```
~/.zen-sci/
├── zensci.db                        ← SQLite: documents, sections, chat_messages, agents
├── documents/
│   └── <id>.json                    ← sidecar written by portal on every content change
├── provider_keys.json               ← {"anthropic": "sk-...", "ollama": null}
├── index/                           ← Tantivy full-text search index
└── outputs/                         ← rendered artifacts
```

---

## Specification Audit Summary (2026-02-18)

**Full report:** `workspace/2026-02-18_audit-findings.md`
**Result: 17 issues found (3 Critical, 5 High, 5 Medium, 4 Advisory). No spec requires rewrite.**

### Key Decisions From Audit (Cruz, 2026-02-18)

| # | Decision | Route | Specs Modified |
| :--- | :--- | :--- | :--- |
| D1 | **Drop ZenSciServer, use native McpServer** — Composition over inheritance | B | packages-sdk-spec (rewritten) |
| D2 | **Strict enums everywhere** | A | packages-core-spec, grant-mcp |
| D3 | **Split ConversionError identity** — Interface `ConversionErrorData` + class `ConversionError` | A | packages-core-spec, packages-sdk-spec |
| D4 | **Align blog-mcp SEOMetadata** — extends `WebMetadataSchema` from core | A | blog-mcp-v0.2-spec |

---

## v1.1 Portal Spec Divergences (Intentional — 2026-02-20)

Documented here so future agents don't flag these as bugs.

| Spec said | Shipped | Rationale |
| :--- | :--- | :--- |
| `section_title + action` in PatchIntent | `operation + section_id + content + description` | Richer; matches how the UI surfaces the patch |
| Send section index to gateway | Full `document_content` string | Agent needs full context to reason well |
| Memory search seed: `document:{docId}` | Free-form `documentTitle ?? projectId` | Gateway semantic search handles free-form; feels more natural |
| Provider key primary path: gateway endpoint | Rust writes `provider_keys.json` directly | Simpler; endpoint exists but isn't the UI entry point |
| Pipelines page uses `pipelineStore` | Pipelines page uses local state | Store exists and is correct; page diverged in implementation |

---

## Strategic Position

- Not just a converter — the AI helps structure and formalize thinking *before* rendering
- LaTeX output is evidence that thinking was done rigorously
- Suite creates network effects: same input format, different publication targets
- Portal is the thinking workspace: document-grounded agent chat, patch proposals, and memory as context
