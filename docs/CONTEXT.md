# ZenSci — Project Context

**Owner:** TresPiesDesign.com / Cruz Morales
**Domain:** ZenithScience.org
**GitHub:** github.com/TresPies-source/zen-sci
**npm scope:** @zen-sci
**License:** Apache 2.0
**Last updated:** 2026-02-20

> **New agent? Read this file first. Then STATUS.md (live state). Then ARCHITECTURE.md (decisions). Then the file(s) for your task.**

---

## What ZenSci Is

ZenSci is a suite of MCP servers that converts structured markdown and AI-assisted reasoning into publication-ready artifacts — LaTeX PDFs, HTML blog posts, grant proposals, slide decks, academic papers, and email newsletters. The AI acts as a thinking partner first (helping structure reasoning, validate math, organize arguments) and a format converter second. One input schema (`DocumentRequest`), many output formats.

The **zen-sci-portal** is the desktop workspace: a Tauri v2 + SvelteKit 5 app where researchers write, chat with a document-grounded agent, review and apply AI-proposed edits, and submit conversion pipelines — all running locally on their machine.

---

## Implemented Modules (MCP Suite)

| Module | Output | Version | Tests |
|--------|--------|---------|-------|
| `latex-mcp` | LaTeX + PDF | v0.1 | 37 |
| `blog-mcp` | HTML blog | v0.2 | 76 |
| `slides-mcp` | Beamer + Reveal.js | v0.3 | 34 |
| `newsletter-mcp` | MJML + HTML email | v0.3 | 63 |
| `grant-mcp` | LaTeX / Word | v0.4 | 50 |
| `paper-mcp` | IEEE/ACM/arXiv LaTeX | v0.5 | 36 |

All 6 modules have MCP App companions — single-file HTML bundles at `ui://` URIs.

---

## Tech Stack

```
zen-sci-portal (Desktop app — SvelteKit 5 + Tauri v2 + Rust)
  └── SQLite (~/.zen-sci/zensci.db)          ← documents, sections, chat_messages, agents
  └── document sidecars (~/.zen-sci/documents/)  ← JSON files for gateway reads
  └── provider keys (~/.zen-sci/provider_keys.json)
  └── AgentChat.svelte + agentStore           ← 4-step durable init, history re-hydration
        ↓
AgenticGateway (Go binary, :8080)
  └── runAgentLoop()                          ← LLM↔tool loop, 8 max iterations
  └── parsePatchIntent()                      ← fenced JSON block extraction
  └── Provider layer: Anthropic / OpenAI / Ollama normalized
        ↓
zen-sci MCP servers (Node.js, esbuild bundles, stdio bridge)
  └── latex-mcp, blog-mcp, grant-mcp, slides-mcp, newsletter-mcp, paper-mcp

TypeScript (Node.js >= 20)
  └── @modelcontextprotocol/sdk       ← MCP protocol, tool registration
  └── @modelcontextprotocol/ext-apps  ← MCP App companion bundles
  └── zod                             ← Schema validation
  └── packages/sdk                    ← createZenSciServer() factory + runConversionPipeline()

Python (>= 3.11)
  └── pandoc (>= 3.0)
  └── TeX Live (pdflatex/xelatex)
  └── sympy, bibtexparser, pypandoc

Build: pnpm >= 8 workspaces, tsc --build, Vitest, cargo
```

---

## Filesystem Map

```
ZenflowProjects/ZenithScience/
│
├── docs/
│   ├── CONTEXT.md                           ← YOU ARE HERE — orientation
│   ├── STATUS.md                            ← Live state: what's done, what's next, what's blocked
│   ├── ARCHITECTURE.md                      ← Decisions, portal arch diagram, audit record
│   │
│   ├── audits/
│   │   ├── 2026-02-18_health_audit.md       ← Baseline health audit
│   │   ├── 2026-02-19_portal_status_audit.md
│   │   └── 2026-02-20_durability_audit.md   ← Durability patch audit (latest)
│   ├── architecture/
│   │   └── 2026-02-17_semantic-clusters.md
│   ├── scouts/
│   │   ├── 2026-02-17_module-strategy-scout.md
│   │   └── 2026-02-20_portal_v1.1_strategic_scout.md
│   ├── research/
│   │   └── 2026-02-17_naming-research.md
│   ├── schemas/
│   │   └── SCHEMAS.md
│   ├── specs/
│   │   ├── infrastructure/                  ← packages-core-spec, packages-sdk-spec
│   │   ├── phase-1/ phase-2/ phase-3/       ← module specs
│   │   ├── phase-4/                         ← MCP Apps architecture + 6 companion specs
│   │   ├── portal/                          ← portal specs (if any)
│   │   └── zen-sci-portal-v1.1-spec.md      ← v1.1 feature spec
│   ├── handoffs/
│   │   ├── handoff_zero.md
│   │   ├── 2026-02-18_handoff_phase-3-complete-to-phase-4.md
│   │   └── 2026-02-20_v1.1-durability-patch.md  ← latest handoff
│   └── workspace/                           ← Implementation prompts, audit findings
│
├── zen-sci/                                 ← MCP suite (ALL PHASES IMPLEMENTED)
│   ├── packages/
│   │   ├── core/                            ← 129 tests, 91% coverage
│   │   └── sdk/                             ← 68 tests, 85% coverage
│   └── servers/
│       ├── latex-mcp/      (+app/)          ← 37 tests
│       ├── blog-mcp/       (+app/)          ← 76 tests
│       ├── slides-mcp/     (+app/)          ← 34 tests
│       ├── newsletter-mcp/ (+app/)          ← 63 tests
│       ├── grant-mcp/      (+app/)          ← 50 tests
│       └── paper-mcp/      (+app/)          ← 36 tests
│
├── zen-sci-portal/                          ← Desktop app (v1.1 + durability patch)
│   ├── src/
│   │   └── lib/
│   │       ├── stores/agent.ts              ← 4-step durable init protocol
│   │       ├── api/tauri.ts                 ← typed IPC wrapper
│   │       ├── components/AgentChat.svelte  ← chat UI + patch intent banner
│   │       └── types/index.ts               ← PatchIntent, AgentResponse, StoredAgent
│   └── src-tauri/src/
│       ├── commands/document.rs             ← write_document_sidecar()
│       ├── commands/gateway.rs              ← chat, agent registry, memory, tools
│       ├── db/mod.rs                        ← SQLite: documents + agents table
│       ├── gateway/client.rs               ← GatewayClient with verify_agent()
│       └── lib.rs                           ← Tauri entry point + invoke_handler
│
└── AgenticGatewayByDojoGenesis/             ← Go gateway (separate repo, external)
    └── server/
        ├── handle_gateway.go               ← runAgentLoop, parsePatchIntent, handleGetDocument
        └── router.go                       ← all routes including /health, /v1/gateway/*
```

---

## Durable Storage Layout

All user data lives under `~/.zen-sci/`. This directory is **outside the Tauri bundle** and survives app updates, reinstalls, and macOS system migrations.

```
~/.zen-sci/
├── zensci.db                   ← SQLite (documents, sections, chat_messages, agents)
├── documents/
│   └── <uuid>.json             ← sidecar for each document (written by portal on every content write)
├── provider_keys.json          ← {"anthropic": "sk-...", "openai": null, "ollama": null}
├── index/                      ← Tantivy full-text search index
└── outputs/                    ← rendered artifacts (PDFs, HTML, etc.)
```

---

## Reading Order by Agent Type

### Implementation Agent (portal)
1. This file (CONTEXT.md)
2. `docs/STATUS.md` — current state, next steps
3. `docs/specs/zen-sci-portal-v1.1-spec.md` — v1.1 feature spec
4. `zen-sci-portal/src-tauri/src/commands/gateway.rs` — command layer
5. `zen-sci-portal/src/lib/stores/agent.ts` — durability protocol
6. `ARCHITECTURE.md § Portal Three-Tier Stack` — architecture diagram

### Implementation Agent (MCP modules)
1. This file (CONTEXT.md)
2. `specs/infrastructure/packages-core-spec.md` — MUST read before any code
3. `specs/infrastructure/packages-sdk-spec.md` — MUST read before any server
4. Your assigned module spec
5. `zen-sci/` — inspect existing implementation

### Strategic / Architectural Agent
1. This file (CONTEXT.md)
2. `ARCHITECTURE.md` — all decisions + semantic clusters + portal arch
3. `STATUS.md` — current phase, blockers, workstreams
4. `architecture/2026-02-17_semantic-clusters.md`
5. `scouts/2026-02-20_portal_v1.1_strategic_scout.md`

### Audit / Review Agent
1. This file (CONTEXT.md)
2. `STATUS.md` — health audit dashboard
3. `ARCHITECTURE.md` — spec audit summary + v1.1 divergences
4. `audits/2026-02-20_durability_audit.md` — latest

---

## Open Questions

1. MCP registry namespace — `io.github.trespies-source` vs. alternatives → **Open**
2. Trademark/domain availability (ZenSci, ZenithScience.org, @zen-sci) → **Open, pre-release**
3. v1.2 scope — Streaming SSE vs. `search_memory` tool vs. `pipeline_proposal` → **Open**

All other strategic questions have been decided — see ARCHITECTURE.md for the full decision log.
