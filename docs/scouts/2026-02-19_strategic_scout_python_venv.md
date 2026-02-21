# Strategic Scout: Python venv Activation for ZenSci Gateway
**Date:** 2026-02-19
**Owner:** Cruz Morales / TresPiesDesign.com
**Scout:** Strategic Thinking Agent (claude-sonnet-4-6)
**Status:** Decision ready

---

## Tension Statement

The ZenSci stack requires that every Python subprocess spawned by a Node.js MCP server finds
`pypandoc`, `sympy`, `bibtexparser`, and `pylatexenc` on the Python path. These packages live
exclusively inside `/Users/alfonsomorales/ZenflowProjects/ZenithScience/zen-sci/.venv`. The
gateway that owns those subprocesses is a compiled Go binary launched in four completely
different environment contexts — developer terminal, CI pipeline, and Tauri desktop sidecar —
none of which guarantee the venv is active. The current workaround (manually `source`-ing the
venv before launching the gateway) is invisible to Tauri and will silently break the near-term
demo unless replaced by a durable mechanism.

---

## What the Code Actually Does

Before scouting routes it is worth being precise about the call chain, because each layer is a
potential intervention point.

```
Tauri desktop app (Rust)
  └─ spawns: agentic-gateway binary (sidecar)
       └─ reads: gateway-config.yaml
            └─ starts MCP servers via stdio transport
                 └─ node zen-sci/servers/latex-mcp/dist/index.bundle.js
                      └─ createZenSciServer() → PythonEngine
                           └─ spawn(this.pythonPath, [scriptPath], …)
                                // this.pythonPath defaults to 'python3'
                                // inherits the process environment of the node process
                                // which inherits from agentic-gateway
                                // which inherits from Tauri's launch environment
```

Key files:
- `zen-sci/packages/sdk/src/integration/python-engine.ts` — `PythonEngine` constructor:
  `this.pythonPath = pythonPath ?? 'python3'` — no venv awareness today.
- `zen-sci/packages/sdk/src/factory/create-server.ts` — `createZenSciServer()`:
  `const pythonEngine = new PythonEngine(logger)` — no `pythonPath` argument passed.
- `gateway-config.yaml` (in `AgenticGatewayByDojoGenesis/`) — MCP server transport blocks
  support an `env:` map that the Go config loader expands with `${VAR}` syntax.
- `mcp/config.go` — `expandEnvVars()` runs before YAML is unmarshalled; the `env` map on each
  transport is passed to the spawned Node process as its environment.

The venv Python path is:
`/Users/alfonsomorales/ZenflowProjects/ZenithScience/zen-sci/.venv/bin/python3`

---

## Routes

### Route A — Environment Variable Injection via gateway-config.yaml (Zero-code)

**Mechanism:** Add `PYTHON_PATH` (or `ZEN_SCI_PYTHON`) to the `env:` map on each MCP server
transport block in `gateway-config.yaml`. The Node.js process receives this env var. The MCP
server reads it at startup and passes it to `PythonEngine`. Since `PythonEngine` already accepts
an optional `pythonPath` constructor argument, the only code change needed is in `create-server.ts`
to read `process.env.PYTHON_PATH ?? 'python3'`.

**Change surface:**
- `gateway-config.yaml` — add one `env:` line per server block (6 servers = 6 lines, or a
  YAML anchor to de-duplicate).
- `zen-sci/packages/sdk/src/factory/create-server.ts` — one line:
  `new PythonEngine(logger, process.env.PYTHON_PATH)`.
- No Go gateway changes, no Tauri changes, no Python changes.

**For Tauri production:** Set `ZEN_SCI_PYTHON` as a Tauri environment variable in `tauri.conf.json`
under `bundle.resources` or via a shell wrapper that Rust writes before spawning the gateway.
Alternatively, set it via `process.env` in the Tauri `before-build` command or Rust's
`std::env::set_var` before `Command::new("agentic-gateway").spawn()`.

**Upside:**
- Near-zero code. The gateway YAML config already uses env expansion (`${VAR}` expansion is live
  in `mcp/config.go`).
- Works in dev terminal immediately (export the var once in `.zshrc` or an `.env` file).
- Works in CI (set env var in workflow yaml).
- The gateway binary is untouched.
- `PythonEngine.pythonPath` is already an injectable field — this is completing the design intent.

**Downside:**
- Requires someone to set `PYTHON_PATH` (or equivalent) correctly for every launch context.
  For the developer that is a one-time shell config change. For Tauri that requires a Rust-side
  mechanism to detect and inject the path, which is a non-trivial Tauri change.
- The venv path is machine-local and absolute. CI and other machines need the same structure or
  a different value of the variable.
- The variable name `PYTHON_PATH` collides with an obscure Python interpreter internal. Prefer
  `ZEN_SCI_PYTHON` or `ZENSCI_PYTHON_BIN`.

---

### Route B — Absolute venv Python Path Hard-coded in gateway-config.yaml (Simplest)

**Mechanism:** Change the `command:` field for each MCP server transport from `"node"` to the
absolute venv Python path. Wait — that is wrong direction. The Node process is what is launched;
Python is a child of Node. The correct equivalent is: set `PYTHON_PATH` to the absolute venv
path directly in `gateway-config.yaml` without any `${VAR}` expansion — i.e., hard-code the
path string.

**Change surface:**
- `gateway-config.yaml` — add `PYTHON_PATH: /Users/alfonsomorales/ZenflowProjects/ZenithScience/zen-sci/.venv/bin/python3`
  under each server's `env:` block.
- `zen-sci/packages/sdk/src/factory/create-server.ts` — same one-line change as Route A.

**Upside:**
- Even simpler than Route A for the developer. No environment variable to remember to set.
- Works in dev terminal and CI immediately after the config change.
- Zero gateway changes.

**Downside:**
- The venv path is machine-local and absolute. The config file becomes un-portable. CI, other
  developers, and Tauri production environments will have different paths. This is the most
  fragile option for anything beyond a solo dev machine.
- Hard to commit to source control without encoding a machine-specific assumption in the repo.
- Fails completely in production Tauri where the venv does not exist at a predictable path.

**Verdict:** Acceptable as a temporary dev shortcut. Not a production answer.

---

### Route C — Venv Python Path Auto-Discovery in PythonEngine (SDK change, portable)

**Mechanism:** At `PythonEngine` construction time, if no `pythonPath` is provided, walk a
known relative path from the script being executed back to the venv. Specifically: given that
every Python script lives at `servers/{module}/engine/*.py` relative to the monorepo, and the
venv lives at `zen-sci/.venv`, the engine can compute:

```
script: /path/to/zen-sci/servers/latex-mcp/engine/latex_engine.py
venv:   /path/to/zen-sci/.venv/bin/python3
```

The SDK walks `dirname(scriptPath)` up 3 levels and checks for `.venv/bin/python3`. If found
and executable, use it. Otherwise fall back to `process.env.PYTHON_PATH ?? 'python3'`.

**Change surface:**
- `zen-sci/packages/sdk/src/integration/python-engine.ts` — add a static `resolveVenvPython()`
  method, called from `runJSON()` when `this.pythonPath === 'python3'`.
- No gateway changes, no config changes, no Tauri changes.

**Upside:**
- Zero configuration: works automatically on any machine where the monorepo structure is intact
  and the venv is built.
- Works in dev terminal, CI, and any Node process regardless of its inherited environment.
- Self-documenting: the code explains the venv expectation.
- Portable: the relative path calculation survives repo moves as long as internal structure is
  preserved.

**Downside:**
- Breaks in Tauri production. The MCP server bundles are esbuild single-file JS bundles
  (`dist/index.bundle.js`). The Python engine scripts are bundled alongside as Tauri resources.
  The relative path from bundle to venv will not exist in the production app bundle — the venv
  is a dev artifact, not a shipped dependency.
- The path-walking logic adds fragility if the monorepo structure changes.
- CI: the CI machine must have the venv built at the same relative path. This is achievable but
  must be documented.

**Verdict:** Excellent for dev and CI. Incomplete for production. Pairs well with Route A as a
fallback chain: try env var first, then path-walk, then system python3.

---

### Route D — Venv Python Baked into the esbuild Bundle via a Wrapper Script (Production-grade, no venv in prod)

**Mechanism:** Acknowledge that the venv is a dev artifact and that production must not depend
on it. Instead, for production builds:

1. At bundle time (esbuild step), do NOT include Python engine scripts. Instead, ship the Python
   dependencies as a native binary or pre-compiled wheel bundle, or use a wrapper approach.
2. More practically for a near-term demo: ship the Python scripts as Tauri resources alongside a
   known Python binary path discovered at Tauri startup.

The Tauri Rust layer, when launching the gateway as a sidecar, can write a small env-injection
wrapper: before spawning `agentic-gateway`, Rust code calls `std::env::set_var("PYTHON_PATH", resolved_venv_path)` where `resolved_venv_path` is computed from
`tauri::api::path::resource_dir()` or a bundled `python3` binary path.

Alternatively, the Tauri `beforeDevCommand` and build pipeline can set `PYTHON_PATH` as a
build-time injected environment variable.

**Change surface:**
- `zen-sci-portal/src-tauri/src/main.rs` (or wherever the sidecar is spawned) — set env var
  before spawning gateway.
- `zen-sci/packages/sdk/src/factory/create-server.ts` — read `PYTHON_PATH` env var.
- `tauri.conf.json` — bundle Python scripts as resources alongside the gateway binary.
- No gateway Go changes.

**Upside:**
- True production readiness. The mechanism works without a shell, without an active venv, and
  without any developer ceremony.
- Rust-side environment injection is the cleanest approach for a sidecar pattern: Tauri owns the
  launch environment.
- Compatible with app sandboxing on macOS.

**Downside:**
- Requires Rust changes in the portal (source changes to `src-tauri/src/`).
- In production, if the Python binary is bundled, it adds significant app size (Python runtime
  is ~60-80MB for a minimal venv with numpy-free deps like sympy/pypandoc).
- `pdflatex` is not bundled — it must be installed on the user's machine. This is a deeper
  dependency problem that no venv strategy fully solves. The venv problem and the pdflatex
  problem are separate concerns.
- More work than needed for a near-term demo.

---

### Route E — Absolute Python Path via a Launch Script (Zero-code, dev-only bridge)

**Mechanism:** Create a `scripts/start-gateway.sh` in `AgenticGatewayByDojoGenesis/` that:
1. Sources the venv: `. /path/to/.venv/bin/activate`
2. Exports `PYTHON_PATH` to the venv python
3. Launches the gateway binary

For CI, the workflow YAML does the same. For Tauri, the `beforeDevCommand` in `tauri.conf.json`
runs this script before starting the dev server.

**Change surface:**
- One shell script in `AgenticGatewayByDojoGenesis/scripts/start-gateway.sh`.
- `zen-sci/packages/sdk/src/factory/create-server.ts` — read `PYTHON_PATH` (same one-line
  change needed by Routes A, C, D).
- No gateway changes, no Go changes.

**Upside:**
- Fastest to ship for a demo. One shell file plus one TypeScript line.
- The gateway binary itself is untouched.
- The script is a natural place to document the venv requirement.

**Downside:**
- Does not solve the Tauri production sidecar problem — there is no shell to run the script.
- Adds yet another file that developers must know to use instead of running the binary directly.
- Does not survive `cd AgenticGatewayByDojoGenesis && ./agentic-gateway` style invocation.

---

## Tradeoff Comparison

| Route | Code Changes | Dev Terminal | CI | Tauri Dev | Tauri Prod | Effort | Durability |
|-------|-------------|-------------|-----|-----------|------------|--------|------------|
| A — env var in gateway-config.yaml | SDK: 1 line; Config: 6 lines | Set var in .zshrc | Set in workflow yaml | Set in Tauri env | Needs Rust env injection | Low | High |
| B — hard-coded path in config | SDK: 1 line; Config: 6 lines | Works immediately | Works if path matches | Works | Breaks | Very Low | Very Low |
| C — path-walk in PythonEngine | SDK: ~20 lines | Works automatically | Works if venv built | Works | Breaks in bundle | Low-Medium | Medium |
| D — Rust env injection in Tauri | SDK: 1 line; Rust: 10-20 lines | Same as A | Same as A | Native | Native | Medium | Very High |
| E — launch script | Shell: 1 file; SDK: 1 line | Must use script | Must use script | Limited | Breaks | Very Low | Low |

### Constraint Filters Applied

| Constraint | Filters |
|-----------|---------|
| No gateway Go changes desired | All routes pass (none touch Go) |
| Node.js MCP servers: source changes OK | Routes A, C, D, E all require the same 1-line change to create-server.ts |
| Near-term demo focus | Routes A + E are fastest; C is close behind |
| Tauri v2 sidecar production | Only Route D + Route A-with-Rust-injection fully solve this |
| Solo dev (Cruz) | Lower ceremony routes preferred |
| CI must work | Routes A, C, D all work; B is fragile |
| pdflatex is a separate dependency | No route solves pdflatex availability — that is user machine state |

---

## Merged Recommendation

**Recommended approach: Route A + Route C as a layered resolution chain, with Route D
deferred to post-demo production hardening.**

### Immediate actions (demo-ready in under 1 hour)

**Step 1 — SDK change (one line, `create-server.ts`):**

In `zen-sci/packages/sdk/src/factory/create-server.ts`, line 40, change:

```typescript
const pythonEngine = new PythonEngine(logger);
```

to:

```typescript
const pythonEngine = new PythonEngine(logger, process.env.ZENSCI_PYTHON_BIN);
```

This threads the env var through to `PythonEngine.pythonPath`. When the var is unset, the
existing fallback to `'python3'` continues to apply.

**Step 2 — gateway-config.yaml (6 lines, using a YAML anchor to stay DRY):**

In `gateway-config.yaml`, add a YAML anchor at the top of the `servers:` section and reference
it in each server's `env:` block:

```yaml
x-zen-sci-env: &zen-sci-env
  LOG_LEVEL: "info"
  ZENSCI_PYTHON_BIN: "${ZENSCI_PYTHON_BIN}"
```

Then in each server transport block, change:
```yaml
env:
  LOG_LEVEL: "info"
```
to:
```yaml
env: *zen-sci-env
```

Now the gateway passes `ZENSCI_PYTHON_BIN` through to each Node process from its own
environment.

**Step 3 — Developer shell config (one export in `.zshrc` or `.env`):**

```sh
export ZENSCI_PYTHON_BIN="/Users/alfonsomorales/ZenflowProjects/ZenithScience/zen-sci/.venv/bin/python3"
```

This survives all terminal launches, Makefile invocations, and any tool that inherits the
developer shell environment.

**Step 4 — Route C fallback in PythonEngine (adds ~15 lines):**

In `zen-sci/packages/sdk/src/integration/python-engine.ts`, modify `runJSON()` to auto-detect
the venv when no explicit path is configured. The resolution order becomes:

1. `pythonPath` constructor argument (explicit, wins always)
2. `process.env.ZENSCI_PYTHON_BIN` (already handled by Step 1 via the constructor)
3. Path-walk from script location up to `.venv/bin/python3` (dev/CI safety net)
4. System `python3` (final fallback, will fail with exit 2 if packages missing)

The path-walk is a guard net, not the primary mechanism. It ensures that even if someone forgets
to set the env var in a new shell or a new CI pipeline, the system silently falls into the
correct Python when the monorepo structure is intact.

**Step 5 — CI (one line in the workflow YAML):**

```yaml
env:
  ZENSCI_PYTHON_BIN: "${{ github.workspace }}/zen-sci/.venv/bin/python3"
```

Combined with a `pip install -r requirements.txt` step in the venv, this is deterministic
across runners.

### Post-demo production hardening (Route D, deferred)

For the Tauri production sidecar, the cleanest solution is:

In `zen-sci-portal/src-tauri/src/` (wherever the gateway sidecar is launched via Tauri's
`Command` API), inject `ZENSCI_PYTHON_BIN` as an environment variable on the sidecar command
before spawning it. The value would be computed from `tauri::api::path::resource_dir()` plus
a known relative path to a bundled Python binary.

This is deferred because:
1. It requires a decision about whether to bundle Python or require macOS system Python / the
   user to have Python with the packages installed.
2. The demo runs on Cruz's machine where the venv already exists. Route A + Route C is
   sufficient to make the demo deterministic.
3. The pdflatex dependency is a larger unsolved problem for production distribution that will
   dictate whether a full Python runtime bundle is even necessary.

### Why not Route B

Route B (hard-coding the absolute path in `gateway-config.yaml`) encodes a machine-specific
assumption into a shared config file. It will break the moment someone else runs the project or
CI uses a different workspace path. It is not worth the risk even for a demo.

### Why not Route E alone

Route E (launch script) does not fix the Tauri sidecar context. It also adds a ceremony layer
that is less durable than a shell config variable. It has a role as a convenience wrapper for
new contributors but should not be the primary mechanism.

---

## Decision Summary

| Question | Answer |
|---------|--------|
| Primary mechanism | Env var `ZENSCI_PYTHON_BIN`, set in developer shell and CI workflow yaml, threaded through `gateway-config.yaml` → Node process → `PythonEngine` |
| Code change in gateway (Go) | None |
| Code changes in MCP SDK | 1 line in `create-server.ts` + ~15 lines in `python-engine.ts` |
| Config changes | `gateway-config.yaml`: add env var to each server transport block |
| Tauri production | Deferred — Rust env injection in portal sidecar launch, post-demo |
| Env var name | `ZENSCI_PYTHON_BIN` (avoids collision with Python's own `PYTHON_PATH` internal) |
| pdflatex availability | Out of scope for this scout — separate dependency management problem |

---

## Affected Files

| File | Change Type | Required For |
|------|------------|-------------|
| `zen-sci/packages/sdk/src/factory/create-server.ts` | 1-line edit | All contexts |
| `zen-sci/packages/sdk/src/integration/python-engine.ts` | ~15-line addition | Dev/CI safety net |
| `AgenticGatewayByDojoGenesis/gateway-config.yaml` | 6-line addition (env map) | All gateway-launched contexts |
| Developer `.zshrc` / `.env` | 1-line export | Developer terminal |
| CI workflow yaml (when created) | 1-line env var | CI |
| `zen-sci-portal/src-tauri/src/` | Rust env injection ~20 lines | Production Tauri (deferred) |

---

## What This Does NOT Solve

1. **`pdflatex` availability.** The `latex_engine.py` spawns `pdflatex` as a subprocess. If
   `pdflatex` is not on the system PATH (it is not inside the venv), the engine will fall back
   to LaTeX-source-only output regardless of which Python is used. This is a distribution
   problem separate from the venv activation problem. For the demo, Cruz's machine presumably
   has TeX Live installed. For production distribution, this requires a separate strategy (bundle
   a TeX distribution, require an installer, or document the system dependency).

2. **Venv freshness across installs.** If dependencies in `engine/requirements.txt` change, the
   venv must be rebuilt. The env var approach gives no signal about whether the pointed-to venv
   is current. A CI step that always recreates the venv from `requirements.txt` is the right
   guard.

3. **Windows compatibility.** The venv path uses Unix-style separators. On Windows, the path to
   the venv Python would be `.venv/Scripts/python.exe`. The path-walk in Route C should account
   for this. Not relevant for the current macOS + darwin target but worth noting for future
   cross-platform work.
