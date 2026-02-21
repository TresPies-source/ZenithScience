# zen-sci-portal — Spec Review Summary
**Date:** 2026-02-18
**Status:** Ready for Cruz review — NOT commissioned for implementation

All three portal specs have been revised. This document summarizes every locked decision, what changed in each spec, and any open questions remaining before implementation.

---

## Locked Architectural Decisions

| # | Decision | Choice |
|---|----------|--------|
| 1 | Database ownership | Portal owns its own PostgreSQL DB |
| 2 | Sync strategy | Git-like async + AI-assisted merges (via Gateway DAG) |
| 3 | Publishing model | Explicit publish button → public URL |
| 4 | Collaboration scope | Section-level co-author editing from day one |
| 5 | Auth architecture | Gateway as identity provider (JWT endpoints added to Gateway in v1) |
| 6 | Real-time editing | v1 ships git-like async; CRDT reserved for v2 (forward-looking `crdt_state` column already in schema) |
| 7 | AgenticGateway role | Center of gravity — not a sidecar. Rust layer is a sync/cache/credential/search facade in front of it |
| 8 | Document catalog | Cloud (Gateway) owns canonical catalog; desktop holds git-like clones with version hashes |
| 9 | Games | Adaptive Quiz as working game demo; games are existing content + game mechanics layer |
| 10 | Ship strategy | v1 full architecture — no bootstrapping |
| 11 | Section boundaries | Draft mode (free-form) → structured mode (ownership assigned) — explicit owner-triggered transition |
| 12 | Orphaned sections | Owner reclaim mechanism when co-author leaves; sections show "unowned" warning |

---

## What Changed in Each Spec

### Desktop — `zen-sci-portal-v0.1-spec.md`
**Was:** Document launcher with AgenticGateway as a sidecar utility (MCP document conversion only)
**Now:** Agentic workspace where Gateway is the reasoning engine

Specific additions in Revision R1 (lines 1291–1888):

- **Full IPC surface** — 8 new IPC commands: `create_agent`, `chat_with_agent`, `submit_dag`, `get_dag_status`, `search_memory`, `store_memory`, `list_providers`, `switch_provider`. These expose the Gateway's agent chat, DAG orchestration, memory, and provider routing (Ollama ↔ Kimi K2.5 ↔ Anthropic).
- **Document catalog model** — Documents have `version_hash`, `upstream_url`, `last_synced_at`. Sync is explicit push/pull. Conflicts trigger AI-assisted merge via Gateway DAG.
- **Collaboration data model** — `Document` and `Section` schemas with `mode: draft|structured`, `crdt_state: Option<Bytes>` (null in v1), `owner_id`, `co_authors[]`.
- **Draft → structured transition** — Owner clicks "Organize sections" → section assignment UI → ownership is locked.
- **Orphaned sections** — Detection, notification, `reclaim_section(section_id, new_owner_id)` IPC command.
- **Auth integration** — Desktop stores JWT in platform keychain (via `keyring` crate). All Gateway calls use Bearer token. Gateway needs `/auth/register`, `/auth/login`, `/auth/refresh` endpoints added.

---

### Web Portal — `zen-sci-web-v0.1-spec.md`
**Was:** Thin client display layer for desktop outputs
**Now:** Full stateful peer workspace — Notion for academic work + GitHub Pages for group outputs + social layer

Specific additions in Revision R1 (lines 1237–1848):

- **Architecture reframe** — Portal and desktop are peers. Portal works directly in canonical source. Desktop works offline with local Ollama.
- **Full database schema** — 9 tables: `users`, `groups`, `group_members`, `projects`, `documents`, `sections`, `document_versions`, `comments`, `suggestions`, `activity_events`. Documents have `version_hash`, `mode`, `published_at`, `publish_url`, `crdt_state BYTEA` (null in v1).
- **Auth endpoints** — `POST /auth/register`, `POST /auth/login`, `POST /auth/refresh` added to Gateway. Portal stores JWT in httpOnly cookie.
- **Section ownership** — Draft/structured mode, section assignment UI (drag-and-drop), co-author color coding.
- **Git-like version history** — Every save creates a `document_versions` record. Version history UI: commit list, diff view, revert.
- **AI-assisted merge** — On sync conflict: portal returns both versions + common ancestor → desktop triggers Gateway DAG merge → author reviews.
- **Explicit publish** — "Publish" button → `published_at` set → public URL generated at `portal.zensci.io/groups/:slug/outputs/:doc-slug`. Read-only, no auth required.
- **Orphaned sections** — `PATCH /api/sections/:id/owner` endpoint. Warning banner in document when sections are unowned.
- **Forward-looking CRDT** — `crdt_state BYTEA` column reserved. v2 migration path: populate from `document_versions.content_snapshot` when WebSocket added to Gateway.

---

### Mobile — `zen-sci-mobile-v0.1-spec.md`
**Was:** Content consumption (Quiz, CardDeck, Reading, VideoScript) — no games
**Now:** Same content types + Adaptive Quiz as working game demo

Specific additions in Revision R1 (lines 2341–3751):

- **Games architecture** — Games are content types + game mechanics layer (not a parallel track). `gameFormat` field added to `MobileContent`. `AdaptiveQuizContent` extends `MobileContent` with `game_rules: GameRules` and `game_state?: GameSession`.
- **Type definitions** — `GameRules`, `AdaptiveQuizContent`, `GameSession`, `Achievement` TypeScript interfaces. `QuizQuestion` extended with `difficulty: 'easy'|'medium'|'hard'` field.
- **Game state storage** — Rust IPC: `save_game_session`, `get_game_session`, `get_achievements`, `unlock_achievement`. SQLite: `game_sessions` and `achievements` tables. Local only in v1 (no cloud sync).
- **AdaptiveQuizPlayer.svelte** — Full working component (~670 lines). Reactive state, adaptive difficulty algorithm (streak ≥ 3 → bump up; 2 wrong → bump down), 5-screen flow (Start → Question → Result → Achievement toast → Summary), score animations, flame streak indicator, progress bar.
- **Transformation DAG** — `generate_adaptive_quiz` DAG spec in YAML: paper-mcp output → concept extraction → question generation → difficulty tagging → achievement generation → `AdaptiveQuizContent` JSON.
- **Demo verification checklist** — 20-point checklist confirming the game is end-to-end playable from real paper-mcp content.
- **v2 reserved** — MatchingGame, TimelineSort, BranchingNarrative noted as reserved formats. Leaderboards, spaced repetition, timers noted for v1.1.

---

## Architecture Coherence Check

The three surfaces now form a coherent system:

```
                    AgenticGateway (Go)
                    ┌─────────────────────────────────────┐
                    │  JWT Auth (+ new /auth/* endpoints)  │
                    │  8 LLM providers (Ollama, Kimi, etc) │
                    │  DAG orchestration                    │
                    │  SQLite memory / agent sessions       │
                    │  MCP servers (ZenSci 6 servers)       │
                    │  App Bridge (MCP Apps)                │
                    └──────────┬──────────────┬────────────┘
                               │              │
              ┌────────────────▼──┐    ┌──────▼────────────────────┐
              │  Desktop (Tauri)   │    │  Web Portal (SvelteKit)    │
              │  Rust facade layer │◄──►│  PostgreSQL DB             │
              │  Local Tantivy idx │    │  Canonical document source │
              │  Git-like clone    │    │  Section ownership UI      │
              │  Keychain JWT      │    │  Publish → public URLs     │
              │  Offline Ollama    │    │  Social: groups, activity  │
              └───────────────────┘    └───────────────────────────┘
                                                    │
                                       ┌────────────▼──────────────┐
                                       │  Mobile (Tauri v2)         │
                                       │  Content transformation    │
                                       │  Adaptive Quiz game demo   │
                                       │  Local SQLite cache        │
                                       │  iOS + Android             │
                                       └───────────────────────────┘
```

**Data ownership is clear:**
- Agent memory, sessions, LLM state → Gateway SQLite
- Canonical documents, social, versions → Portal PostgreSQL
- Local search index, credentials, offline cache → Desktop/Mobile local storage

**Auth flows through one place:** Gateway issues JWTs. Desktop stores in keychain. Web portal stores in httpOnly cookie. Mobile stores in platform keychain (iOS Keychain / Android Keystore). All three surfaces speak the same token.

**Sync model is consistent across desktop and mobile:** Both hold git-like clones (version hashes), sync explicitly, detect conflicts via hash comparison, resolve via AI-assisted merge or manual diff review.

---

## Open Questions Before Implementation

These are not blockers for spec review, but will need answers before implementation begins:

1. **Gateway auth endpoints scope** — ✅ LOCKED: Part of the portal v1 spec. The `/auth/register`, `/auth/login`, `/auth/refresh` endpoints are a portal v1 deliverable, implemented in AgenticGateway alongside the portal build.

2. **Portal hosting** — ✅ LOCKED: Self-hosted first (Docker Compose). Stack: portal + Gateway + PostgreSQL + optional Ollama. Managed cloud is a v1.1 addition. Ollama opt-in via Docker Compose profile.

3. **Mobile transformation trigger** — ✅ LOCKED: AI recommendation on open. Gateway reads content type + metadata → suggests format via non-blocking SuggestionCard → one-tap confirm. No background auto-generation. No wasted API credits on unengaged content.

4. **Section boundary definition on first open** — ✅ LOCKED: Any group member creates sections in draft mode. Owner approves structure at the draft → structured transition (merge, rename, assign in "Organize sections" modal).

---

## Spec Files

| Surface | File | Lines | Status |
|---------|------|-------|--------|
| Desktop | `docs/specs/portal/zen-sci-portal-v0.1-spec.md` | 1,888 | Revised (R1 appended) |
| Web Portal | `docs/specs/portal/zen-sci-web-v0.1-spec.md` | 1,848 | Revised (R1 appended) |
| Mobile | `docs/specs/portal/zen-sci-mobile-v0.1-spec.md` | 3,751 | Revised (R1 appended) |

---

*This review document is a companion to the specs — not a replacement. Read the R1 sections in each spec for full detail. No implementation or commissioning until Cruz approves.*
