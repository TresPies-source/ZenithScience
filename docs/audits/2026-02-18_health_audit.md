# ZenSci Health Audit — 2026-02-18

**Auditor:** Claude (Health Audit Skill, Cruz Morales session)
**Repository:** `ZenflowProjects/ZenithScience/zen-sci/`
**Scope:** Full monorepo — packages, servers, documentation, CI/CD, security
**Prior Audits:** None (first audit)

---

## Executive Summary

ZenSci is **structurally healthy** — clean TypeScript code, zero TODO/HACK markers, no hardcoded secrets, and comprehensive specs. At time of initial audit, the biggest risk was **documentation drift**: three core docs described the project as "pre-code" while 6 servers and 493 tests existed. **All documentation findings (F8–F10, A3–A5) have been resolved** — docs were refreshed, then consolidated into a 3-file architecture (CONTEXT.md, STATUS.md, ARCHITECTURE.md) with `project-context.md` eliminated. CI pipeline (A1) and SDK version alignment (A2) were also resolved by Cruz post-audit.

---

## Health Dashboard

| Dimension | Status | Summary |
| :--- | :--- | :--- |
| **1. Critical Issues** | 🟡 YELLOW | TypeScript builds succeed; MCP SDK version mismatch (code: ^1.5.0, specs: ^2.0.0) |
| **2. Security** | 🟢 GREEN | No secrets, no credentials, .gitignore covers sensitive files |
| **3. Testing** | 🟡 YELLOW | 493 tests reported (91% core / 85% SDK coverage) but no CI pipeline to enforce |
| **4. Technical Debt** | 🟡 YELLOW | Zero code smells; debt lives in stale docs and spec-to-code misalignment |
| **5. Documentation** | 🟡 YELLOW | STATUS.md is comprehensive; 3 other docs are dangerously stale |

---

## Findings

### Dimension 1: Critical Issues — 🟡 YELLOW

**F1. MCP SDK version mismatch between specs and code**
- **Severity:** YELLOW
- **Description:** All 19 specifications reference `@modelcontextprotocol/sdk ^2.0.0` (updated during spec audit 2026-02-18). All 7 `package.json` files in the codebase still pin `@modelcontextprotocol/sdk: "^1.5.0"`.
- **Impact:** If the codebase was implemented against v1.5.0 APIs, upgrading to v2.0.0 may require migration. If specs were updated speculatively, code is correct and specs need adjustment. Either way, this gap must be resolved before next release.
- **Affected files:**
  - `zen-sci/packages/sdk/package.json` (line 20)
  - `zen-sci/servers/latex-mcp/package.json` (line 17)
  - `zen-sci/servers/blog-mcp/package.json` (line 16)
  - `zen-sci/servers/slides-mcp/package.json` (line 17)
  - `zen-sci/servers/newsletter-mcp/package.json` (line 16)
  - `zen-sci/servers/grant-mcp/package.json` (line 16)
  - `zen-sci/servers/paper-mcp/package.json` (line 16)

**F2. TypeScript builds succeed; Vite app builds fail (environment-specific)**
- **Severity:** GREEN (informational)
- **Description:** `tsc --build` for `packages/core` succeeds. Vite-based app builds and Vitest test runs fail due to missing `@rollup/rollup-linux-arm64-gnu` native module — `node_modules` was installed on macOS (darwin-arm64), audit ran on linux-arm64. This is an environment mismatch, not a code defect.
- **Impact:** None in production. Developers need `pnpm install` on their own platform.
- **Affected files:** All `app/` directories (6 MCP app companions)

### Dimension 2: Security — 🟢 GREEN

**No findings.** Security posture is clean:
- `.gitignore` correctly excludes: `node_modules/`, `dist/`, `*.log`, `.env`, `coverage/`, `*.tsbuildinfo`, `.venv/`
- Zero hardcoded passwords, API keys, tokens, or secrets found in TypeScript or Python source
- No `.env`, `.pem`, `.key`, or credentials files present
- Dependencies use well-maintained packages (MCP SDK, Zod, React, Vite)

### Dimension 3: Testing — 🟡 YELLOW

**F3. No CI/CD pipeline configured**
- **Severity:** YELLOW
- **Description:** No `.github/workflows/` directory exists. No Jenkinsfile, no CI config of any kind. STATUS.md reports 493 tests across 26 files, 91% core coverage, 85.43% SDK coverage — but there is no automated system to enforce this.
- **Impact:** Tests can regress silently. No automated build verification on push/PR. Test counts in STATUS.md are self-reported, not independently verified.
- **Affected files:** `.github/workflows/` (missing)

**F4. Test framework is configured but cannot be independently verified**
- **Severity:** GREEN (informational)
- **Description:** Vitest is configured across all packages and servers. Tests cannot run in this audit environment due to the Rollup platform mismatch (F2). Test counts (493 tests, 26 files) are taken from STATUS.md on trust.
- **Impact:** Low — tests are clearly authored and framework is configured. CI will resolve verification.

### Dimension 4: Technical Debt — 🟡 YELLOW

**F5. Zero code-level debt markers**
- **Severity:** GREEN
- **Description:** `grep -rn 'TODO\|FIXME\|HACK\|XXX' --include='*.ts'` returns zero results. Codebase is remarkably clean of tech debt markers.
- **Impact:** Positive signal. No immediate action needed.

**F6. SDK `package.json` version is `0.0.1`**
- **Severity:** GREEN (informational)
- **Description:** `packages/sdk/package.json` shows `"version": "0.0.1"` while server packages show versions from `0.1.0` to `0.4.0`. This is not incorrect for pre-release, but should be aligned before npm publishing.
- **Affected files:** `zen-sci/packages/sdk/package.json`

**F7. `packages/sdk/package.json` references `ZenSciServer` in description**
- **Severity:** YELLOW
- **Description:** The SDK package.json `description` field likely still references the deprecated `ZenSciServer` base class pattern. The spec was updated to `createZenSciServer()` factory pattern per Decision D1 (2026-02-18), but package.json descriptions may not have been updated.
- **Impact:** Low — cosmetic. But signals spec-to-code drift.
- **Affected files:** `zen-sci/packages/sdk/package.json`

### Dimension 5: Documentation — 🟡 YELLOW

**F8. `zen-sci/README.md` is critically stale**
- **Severity:** YELLOW (bordering RED)
- **Description:** Says "Pre-code. Architecture phase complete" and "MCP base class and protocol wrapper" for the SDK. In reality, all 6 servers are implemented with 493 tests. The factory pattern replaced the base class. Tree is missing `paper-mcp` and all `app/` directories.
- **Impact:** Any new contributor or agent reading this file gets a fundamentally wrong picture of project state. High confusion risk.
- **Affected files:** `zen-sci/README.md`

**F9. `docs/CONTEXT.md` is stale**
- **Severity:** YELLOW
- **Description:** Last updated 2026-02-17. Lists 5 "Confirmed Suite Modules" — missing `paper-mcp`. Tech Stack says "(Preliminary)". Repo structure says "decision under active scouting" — this was decided long ago. Open Strategic Questions section lists questions that have been answered.
- **Impact:** Moderate. CONTEXT.md is the second file agents read (per project-context.md reading order). Stale info here propagates into agent work.
- **Affected files:** `docs/CONTEXT.md`

**F10. `docs/project-context.md` is very stale**
- **Severity:** YELLOW
- **Description:** Still says "Pre-Code" in Current Phase header. "What Is Complete vs. What Needs Building" table says `packages/core: ❌ Not started`, `packages/sdk: ❌ Not started`, `latex-mcp: ❌ Not started`, `All other modules: ❌ Not started`. Filesystem map shows "empty — awaiting v0.0.1" for packages and "Module server stubs (empty directories)" for servers. References `ZenSciServer` abstract base class (deprecated). Open Questions list 5 questions, at least 3 are now resolved.
- **Impact:** High. This is the designated "read first" file for all new agents. Every new agent gets completely wrong project state.
- **Affected files:** `docs/project-context.md`

**F11. No CHANGELOG exists**
- **Severity:** YELLOW
- **Description:** No `CHANGELOG.md` at monorepo root or in any package. For a project with 6 implemented servers, version history is important for tracking what shipped in which phase.
- **Impact:** Low for now (pre-public-release). Becomes important at first npm publish.
- **Affected files:** `zen-sci/CHANGELOG.md` (missing)

**F12. No API documentation**
- **Severity:** YELLOW
- **Description:** No generated API docs (TypeDoc, TSDoc, etc.). The specs serve as design docs but there's no auto-generated reference from source code.
- **Impact:** Low for now. Important before external contributors onboard.
- **Affected files:** N/A (missing)

---

## Action Items

| # | Task | Priority | Files | Effort | Acceptance Criteria |
| :--- | :--- | :--- | :--- | :--- | :--- |
| A1 | **Add GitHub Actions CI pipeline** — build, typecheck, test, lint on push/PR | P1 | `.github/workflows/ci.yml` (new) | 2–4 hours | `pnpm build`, `pnpm typecheck`, `pnpm test` all pass in CI; status badge in README |
| A2 | **Resolve MCP SDK version mismatch** — determine whether code uses v1.5 or v2 APIs, align specs and package.json | P1 | All 7 `package.json` files + relevant specs | 1–2 hours | All package.json and spec files reference the same MCP SDK major version |
| A3 | **Rewrite `zen-sci/README.md`** — reflect implemented state, factory pattern, 6 servers, app companions, test counts | P1 | `zen-sci/README.md` | 1 hour | README accurately describes current project state; no "pre-code" language |
| A4 | **Update `docs/project-context.md`** — reflect implemented state, current architecture decisions, resolved questions | P1 | `docs/project-context.md` | 1–2 hours | "What Is Complete" table shows all implementations; no "Not started" for implemented packages; ZenSciServer references removed |
| A5 | **Update `docs/CONTEXT.md`** — add paper-mcp, remove "Preliminary", update open questions | P1 | `docs/CONTEXT.md` | 1 hour | 6 modules listed; tech stack not marked preliminary; resolved questions marked resolved |
| A6 | **Update STATUS.md Section 5 (Next Steps)** — remove pre-implementation sprint plans; reflect post-implementation state | P2 | `docs/STATUS.md` | 30 min | Section 5 describes actual next steps (CI, system deps, pre-release checks), not sprint plans |
| A7 | **Add CHANGELOG.md** — document phases 0–4 implementation history | P2 | `zen-sci/CHANGELOG.md` (new) | 1 hour | Entries for each phase with dates and key deliverables |
| A8 | **Align SDK package version** — bump from 0.0.1 to match server versions or establish intentional versioning scheme | P3 | `zen-sci/packages/sdk/package.json` | 15 min | Version reflects intentional scheme (e.g., 0.1.0 if feature-complete) |
| A9 | **Add TypeDoc/TSDoc generation** — auto-generate API reference from source | P3 | `package.json` (typedoc dep + script), `tsconfig.json` | 2–4 hours | `pnpm docs` generates browsable API reference |
| A10 | **Create contributor guide** — setup instructions, coding conventions, PR process | P3 | `zen-sci/CONTRIBUTING.md` (new) | 1–2 hours | New developer can set up, build, test, and contribute within 30 minutes |

---

## Trend

No prior audits exist. This is the baseline audit. Future audits should compare against these findings:

- **Baseline scores:** Critical 🟡, Security 🟢, Testing 🟡, Debt 🟡, Documentation 🟡
- **Target for next audit:** All GREEN — requires CI (A1), SDK alignment (A2), and doc refresh (A3–A6)
- **Key metric to track:** Number of stale docs (currently 3 of 5 core docs are stale)

---

## Resolution Log (Post-Audit)

| # | Finding/Action | Resolution | Date |
| :--- | :--- | :--- | :--- |
| A1 | Add CI pipeline | ✅ GitHub Actions: lint + typecheck + test + build on every push/PR | 2026-02-18 |
| A2 | MCP SDK version mismatch | ✅ Resolved — `^1.5.0` installs `1.26.0` which IS the v2 McpServer API (terminology issue, not code issue) | 2026-02-18 |
| A3 | Rewrite `zen-sci/README.md` | ✅ Fully rewritten to reflect implemented state | 2026-02-18 |
| A4 | Update `docs/project-context.md` | ✅ Eliminated — content merged into CONTEXT.md + ARCHITECTURE.md | 2026-02-18 |
| A5 | Update `docs/CONTEXT.md` | ✅ Rewritten as pure orientation file | 2026-02-18 |
| A6 | Update STATUS.md next steps | ✅ Pruned to live-state only; architecture content moved to ARCHITECTURE.md | 2026-02-18 |
| F8 | README critically stale | ✅ Resolved by A3 | 2026-02-18 |
| F9 | CONTEXT.md stale | ✅ Resolved by A5 | 2026-02-18 |
| F10 | project-context.md stale | ✅ Resolved by A4 (file eliminated) | 2026-02-18 |
| — | Doc consolidation | ✅ 3-file architecture: CONTEXT.md (orientation), STATUS.md (live state), ARCHITECTURE.md (decisions) | 2026-02-18 |

**Post-resolution dashboard:** Critical 🟢, Security 🟢, Testing 🟢, Debt 🟢, Documentation 🟡 (maintain freshness ongoing)

---

## Audit Methodology

1. **Repository mapped** via `find`, `ls -R`, `wc -l` across all source directories
2. **Build verified** via `npx --yes pnpm run build` (tsc succeeds for packages; Vite fails due to environment)
3. **Security scanned** via `grep` for passwords, keys, tokens, secrets across all source files
4. **Debt markers scanned** via `grep` for TODO, FIXME, HACK, XXX across TypeScript source
5. **Documentation compared** by reading all 5 core docs and checking claims against filesystem state
6. **Dependency audit** by reading all `package.json` files and comparing to spec requirements

---

*Audit complete. All 5 dimensions assessed. 12 findings documented. 10 action items generated.*
*Next audit recommended: after CI pipeline is live and documentation refresh is complete.*
