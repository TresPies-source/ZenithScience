# zen-sci-portal Status Audit — 2026-02-19

**Target:** `zen-sci-portal/` — Tauri v2 Desktop Application
**Auditor:** System Health Auditor (claude-sonnet-4-5)
**Commission ref:** `docs/workspace/2026-02-18_zen-sci-portal-desktop-v1.0-commission.md`

---

## Overall Health Score: 82 / 100

---

## Traffic-Light Summary

| Dimension | Status | Score | One-line verdict |
| :--- | :---: | :---: | :--- |
| File Structure | YELLOW | 14/15 | All 34 specified files present; 1 missing integration test (auth flow) |
| Build Health | YELLOW | 13/15 | Frontend builds clean; Tauri production build broken (frontendDist mismatch) |
| Test Health | GREEN | 18/20 | 20 Rust + 2 integration + 15 frontend = 37 total; 0 failures |
| Lint Health | GREEN | 15/15 | `cargo clippy -- -D warnings` exits clean; zero lint errors |
| Dependency Health | GREEN | 13/15 | All deps present and pinned; 1 dev-only concern |
| Code Quality | YELLOW | 9/15 | 5 legitimate stubs (spec-allowed); 1 hardcoded secret in defaults; production unwrap/expect in lib.rs |
| Documentation Drift | YELLOW | 8/10 | STATUS.md and CONTEXT.md do not mention zen-sci-portal at all |

---

## Dimension 1 — File Structure

**Status: YELLOW**

All 34 files from the commission file manifest are present. The directory layout matches Section 3 of the commission spec exactly.

**Missing files (1):**

The commission (Group 6) specifies three integration tests:
- `test_auth_flow` — login → keychain → check_session → logout
- `test_search_flow` — present and passing
- `test_gateway_client` — present and passing

`test_auth_flow` is **absent** from `src-tauri/tests/integration_test.rs`. The file only contains the search and gateway client tests.

**Unexpected files:**
- `build/` directory — generated artifact, not in manifest but expected from adapter-static output; not a problem.
- `dist/` directory — does NOT exist (relevant to the build gap below).
- `.svelte-kit/` — generated SvelteKit cache; expected.
- `node_modules/`, `src-tauri/target/` — expected build artifacts.
- `package-lock.json` — present alongside pnpm usage; minor inconsistency (pnpm projects should only have `pnpm-lock.yaml`, but this does not affect builds since `node_modules/` was populated via pnpm).

**Action:** Add `test_auth_flow` integration test to `src-tauri/tests/integration_test.rs`. Severity: LOW.

---

## Dimension 2 — Build Health

**Status: YELLOW**

### Frontend (pnpm run build)

Result: PASS. 209 modules transformed, zero TypeScript errors, `build/` written via adapter-static.

Warnings emitted (non-blocking, Svelte 5 advisory):
- `src/routes/app/[appId]/+page.svelte:11` — link missing `aria-label` (a11y)
- `src/routes/workspace/[docId]/+page.svelte:124` — link missing `aria-label` (a11y)
- `src/routes/search/+page.svelte:47,62,67` — three `<label>` elements missing `for`/`htmlFor` association (a11y)
- `src/lib/components/ResultsList.svelte:15` — `containerRef` not declared with `$state(...)` (reactivity)
- `src/lib/components/SectionOrganizer.svelte:14` — `document.sections` reference captures initial value only (reactivity)

### Rust backend (cargo check)

Result: PASS. Compiles clean in 5.22s with zero warnings and zero errors.

### Critical gap — Tauri production build (tauri build)

`tauri.conf.json` sets `"frontendDist": "../dist"` but SvelteKit adapter-static writes output to `build/` (the default when no `outDir` override is set in `vite.config.ts` or `svelte.config.js`).

Running `tauri build` would bundle the wrong (or absent) directory. The app binary would launch to a blank window.

**This is a pre-release blocker for any packaged binary distribution.** The dev server path (`tauri dev`) is unaffected since it uses the live Vite dev server at `devUrl`.

**Action (P0):** Either set `"frontendDist": "../build"` in `tauri.conf.json` OR add `build: { outDir: '../dist' }` to `vite.config.ts`. Severity: HIGH — blocks all release packaging.

---

## Dimension 3 — Test Health

**Status: GREEN**

| Suite | Tests | Failures | Notes |
| :--- | :---: | :---: | :--- |
| Rust unit tests | 20 | 0 | error, config, auth, search, gateway, file modules |
| Rust integration tests | 2 | 0 | search_flow, gateway_client |
| Frontend (Vitest) | 15 | 0 | 8 store tests, 4 DocumentEditor, 3 SearchBar |
| **Total** | **37** | **0** | |

**Coverage gaps (observed, not measured):**
- No test for `commands/document.rs` (stub behavior, but stubs should be tested for their return contracts)
- No frontend test for `SectionOrganizer.svelte`, `AgentChat.svelte`, `ResultsList.svelte`
- Missing `test_auth_flow` integration test (see Dimension 1)

---

## Dimension 4 — Lint Health

**Status: GREEN**

`cargo clippy -- -D warnings` passes clean with zero diagnostics. Compilation completes in 3.93s.

No warnings, no lint errors, no dead-code violations surfaced.

---

## Dimension 5 — Dependency Health

**Status: GREEN**

### Cargo.toml

All dependencies are present and version-pinned. Cross-referencing against the commission spec Section 1.3:

| Crate | Specified | Present | Notes |
| :--- | :---: | :---: | :--- |
| tauri | 2 | 2 | features = [] (minimal, correct) |
| tauri-plugin-shell | 2 | 2 | required for sidecar |
| tokio | 1.40 full | 1.40 full | matches spec |
| serde / serde_json | 1.0 | 1.0 | with derive |
| tantivy | 0.22 | 0.22 | |
| notify | 6 | 6 | |
| jsonwebtoken | 9 | 9 | |
| keyring | 3 | 3 | |
| reqwest | 0.12 json+rustls | 0.12 | |
| tracing / tracing-subscriber | 0.1 / 0.3 | present | |
| uuid | 1.0 v4+serde | 1.0 | |
| chrono | 0.4 serde | 0.4 | |
| toml | 0.8 | 0.8 | |
| dirs | 5.0 | 5.0 | |
| walkdir | 2.4 | 2.4 | |
| tauri-build | 2 (build-dep) | 2 | correct placement |

Dev deps: `tokio-test`, `tempfile`, `mockito` — all present and appropriate.

**Minor note:** `dashmap = "5.5"` and `async-trait = "0.1"` appear in `Cargo.toml` but are not referenced in any source module. These are dead dependencies. Severity: LOW.

### package.json

All frontend dependencies from commission Section 1.4 are present:
- `@tauri-apps/api ^2.0.0`, `svelte ^5.0.0`, `@sveltejs/kit ^2.0.0`, `tailwindcss ^3.4.0`, `@tailwindcss/typography ^0.5.10`
- `@sveltejs/adapter-static ^3.0.0`, `vitest ^2.0.0`, `@testing-library/svelte ^5.0.0`

**Note:** `@tauri-apps/plugin-websocket: ^2.0.0` is listed as a runtime dependency but has no usage in `src/`. This is dead code at the package level. Severity: LOW.

---

## Dimension 6 — Code Quality Signals

**Status: YELLOW**

### Stubs (spec-sanctioned, documented)

The following stubs are **explicitly required by the commission** (Section 5, Constraints & Non-Goals) and are correctly marked as v1 limitations:

| Location | Stub | Spec-allowed? |
| :--- | :--- | :---: |
| `commands/document.rs:list_documents` | Returns `vec![]` — local catalog not implemented | Yes |
| `commands/document.rs:get_document` | Returns Err("not found") | Yes |
| `commands/document.rs:transition_to_structured` | Returns Err("Gateway sync not yet implemented") | Yes |
| `commands/document.rs:save_document_version` | Logs, returns Ok(()) | Yes |
| `commands/gateway.rs:suggest_game_format` | Returns `GameFormatSuggestion::stub()` | Yes — explicitly spec'd |

### Production `unwrap`/`expect` in non-test code

| Location | Usage | Risk |
| :--- | :--- | :--- |
| `lib.rs:33` | `.expect("Failed to initialize application state")` | Process abort on startup if dirs can't be created or Tantivy can't open. Acceptable at startup boundary, but error should be logged before panic. |
| `lib.rs:83` | `.expect("error while running tauri application")` | Standard Tauri boilerplate — unavoidable. |
| `commands/gateway.rs:25` | `.unwrap_or(Value::Null)` | Safe fallback — not a panic risk. |

All other `unwrap()` calls are inside `#[cfg(test)]` blocks and are correct test-only usage.

### Hardcoded default secret

`config.rs:35`:
```rust
jwt_secret: std::env::var("JWT_SECRET")
    .unwrap_or_else(|_| "dev-secret-change-in-production".to_string()),
```

This is a well-named fallback but it means any developer who forgets to set `JWT_SECRET` in production will silently use a known secret string. The `dev-secret-change-in-production` label is helpful, but the application should log a prominent `warn!()` when the fallback fires and ideally refuse to start in release profile without an explicit `JWT_SECRET`.

**Action (P1):** Add a `tracing::warn!` when the JWT_SECRET fallback fires. In release builds, consider returning `AppError::AuthError` if the env var is missing. Severity: MEDIUM — security implication in packaged release builds.

### Svelte reactivity warnings (from build output)

Two Svelte 5 reactivity issues surfaced in the build:

1. `ResultsList.svelte:15` — `containerRef` is not `$state(...)`. Scroll-position binding may silently fail if the element is re-mounted.
2. `SectionOrganizer.svelte:14` — `let sections = $state([...document.sections])` captures `document.sections` at mount time. If the parent passes a new `document` prop, `sections` will not update.

**Action (P2):** Wrap `containerRef` in `$state()`. Derive `sections` inside a `$derived()` or add a reactive `$effect()` that re-initializes when `document` changes. Severity: LOW-MEDIUM — causes subtle UI bugs under prop updates.

### IPC signature gap: `logout` takes `user_id` parameter

The commission spec (Group 5) specifies:
```
logout(state) -> Result<(), String>
```
The Rust implementation takes `user_id: String` as an additional parameter. The TS IPC client correctly passes `{ user_id: userId }` to match. The settings route calls `logout($userSession.user_id)`, which is correct.

This is a benign spec deviation that works correctly end-to-end — the extra parameter makes the implementation more explicit. No action needed, but note the divergence from the spec signature.

### `externalBin` platform paths

The commission spec Section 1.2 calls for platform-specific binary paths:
> macOS (arm + x86), Windows (MSVC x64), Linux (gnu x86)

`tauri.conf.json` registers only:
```json
"externalBin": ["binaries/agentic-gateway"]
```

Tauri v2 resolves platform-specific binary names automatically from the base name using the target triple suffix convention (e.g., `agentic-gateway-aarch64-apple-darwin`). The stub binary at `src-tauri/binaries/agentic-gateway-aarch64-apple-darwin` confirms this convention is in use. This approach is valid and more maintainable than enumerating paths explicitly. The spec language is slightly misleading but the implementation is correct.

---

## Dimension 7 — Documentation Drift

**Status: YELLOW**

### docs/STATUS.md

`STATUS.md` was last updated 2026-02-18. It covers Phases 0–4 of the `zen-sci/` MCP monorepo thoroughly. However, **zen-sci-portal receives zero mention**. There is no entry for:
- zen-sci-portal status (complete, tests passing)
- Test count (37 tests — 20 unit, 2 integration, 15 frontend)
- The Tauri v2 stack
- The portal's open items (frontendDist mismatch, missing auth integration test, stubs)

### docs/CONTEXT.md

`CONTEXT.md` likewise does not reference `zen-sci-portal/` in the filesystem map, the tech stack, or the implemented modules section. A new agent reading the project entry point would not know this application exists.

### docs/handoffs/

The handoff file `2026-02-18_handoff_phase-3-complete-to-phase-4.md` is present but this audit did not read it — it may reference the portal if it was written after Phase 4. The commission date and Phase 4 completion suggest the portal was built as part of the Phase 4+ work described in `docs/workspace/2026-02-18_zen-sci-portal-desktop-v1.0-commission.md`.

**Actions (P2):**
1. Add a `zen-sci-portal` section to `STATUS.md` (status, test count, open issues).
2. Add `zen-sci-portal/` to the filesystem map in `CONTEXT.md`.
3. Add the portal to the tech stack section of `CONTEXT.md` (Rust/Tauri/SvelteKit).

---

## Prioritized Issue List

| Priority | ID | Dimension | Issue | Severity | Action |
| :---: | :--- | :--- | :--- | :--- | :--- |
| P0 | PORTAL-01 | Build | `frontendDist: "../dist"` in `tauri.conf.json` does not match SvelteKit adapter-static output `build/` — `tauri build` will package a blank app | HIGH | Set `frontendDist` to `"../build"` in `tauri.conf.json` OR add `build: { outDir: '../dist' }` to `vite.config.ts` |
| P1 | PORTAL-02 | Code Quality | `JWT_SECRET` fallback fires silently in production with a known default string | MEDIUM | Add `tracing::warn!` on fallback; in release profile, consider hard-failing if `JWT_SECRET` not set |
| P1 | PORTAL-03 | File Structure | `test_auth_flow` integration test specified in commission but absent from `src-tauri/tests/integration_test.rs` | LOW-MEDIUM | Add test: login → keychain write → check_session → logout → check_session returns None |
| P2 | PORTAL-04 | Code Quality | `ResultsList.svelte` `containerRef` not `$state(...)` — scroll binding may silently fail | LOW | Declare `let containerRef = $state<HTMLDivElement | undefined>(undefined)` |
| P2 | PORTAL-05 | Code Quality | `SectionOrganizer.svelte` `sections` state captures initial `document.sections` only | LOW | Use `$effect` or `$derived` to re-sync when `document` prop changes |
| P2 | PORTAL-06 | Documentation | `STATUS.md` and `CONTEXT.md` have no mention of `zen-sci-portal` | LOW | Add portal section to both files |
| P3 | PORTAL-07 | Dependencies | `dashmap` and `async-trait` in `Cargo.toml` are unused by any source module | LOW | Remove dead dependencies |
| P3 | PORTAL-08 | Dependencies | `@tauri-apps/plugin-websocket` in `package.json` is unused | LOW | Remove or document intent |
| P3 | PORTAL-09 | Build | 6 Svelte build warnings (4 a11y, 2 reactivity) do not block builds but indicate quality debt | LOW | Add `aria-label` to icon-only links; associate `<label>` with controls |
| P3 | PORTAL-10 | Testing | No tests for `commands/document.rs` stub contracts; no tests for 3 frontend components | LOW | Add command tests asserting stub return shapes; add component tests for SectionOrganizer, AgentChat |

---

## What Is Working Well

- Rust backend compiles clean with zero warnings under `cargo check` and `cargo clippy -- -D warnings`.
- All 20 Rust unit tests and 2 integration tests pass, covering auth, search, gateway, file, and error modules.
- All 15 frontend tests pass with Vitest + @testing-library/svelte.
- Every IPC command specified in the commission is registered in `lib.rs` — the full `invoke_handler!` list is complete.
- Spec-sanctioned stubs are clearly commented, narrowly scoped, and return appropriate error messages (not silent failures).
- `tauri.conf.json` identifier, window dimensions, CSP for `ui://`, and shell sidecar plugin are all correct.
- `adapter-static` with `fallback: 'index.html'` is configured correctly for Tauri's static file serving.
- `AppConfig` loads from `~/.zen-sci/config.toml` with graceful fallback to defaults.
- Feature flags for cloud sync (off) and CRDT (off) are explicitly declared in `config.rs`.
- The `suggest_game_format` stub returns the exact hardcoded shape the spec requires.

---

*Report written to `docs/audits/2026-02-19_portal_status_audit.md`*
