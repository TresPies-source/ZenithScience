# Handoff: Phases 0–3 Complete → Phase 4 Implementation
**From:** Claude Sonnet 4.6 (implementation agent) | **To:** Supervisor / next implementation agent | **Date:** 2026-02-18

---

## Objective

> Phases 0–3 are fully implemented, tested, and spec-conformant; the receiving agent must implement Phase 4 (MCP Apps) using the prompt at `docs/workspace/2026-02-18_phase-4-implementation-prompt.md`.

---

## Required Context

### Project Background
ZenithScience (`zen-sci/`) is a TypeScript monorepo of MCP servers that convert Markdown to publication formats (LaTeX/PDF, HTML, email, slides, academic papers, grant proposals). It uses the `@zen-sci/sdk` factory pattern for all MCP servers. All Phase 0–3 work is complete: 8 packages build cleanly, 493 tests pass.

### Filesystem Layout
```
ZenithScience/
├── zen-sci/               ← monorepo root (NOT zen-sci-mcp/)
│   ├── packages/
│   │   ├── core/          ← shared parsing, math, citations
│   │   └── sdk/           ← MCP factory, Python bridge, utils
│   └── servers/
│       ├── latex-mcp/
│       ├── blog-mcp/
│       ├── grant-mcp/
│       ├── slides-mcp/
│       ├── newsletter-mcp/
│       └── paper-mcp/
├── docs/
│   ├── specs/             ← all 19 specs (phase-0 through phase-4)
│   ├── workspace/         ← phase prompts and audit findings
│   └── STATUS.md
└── handoffs/              ← this file
```

**Path mapping** (spec references → actual):
- `ZenithScience/CONTEXT.md` → `docs/CONTEXT.md`
- `workspace/` → `docs/workspace/`

### Tech Stack & Conventions
| Concern | Choice |
|---|---|
| Language | TypeScript (strict, `exactOptionalPropertyTypes: true`) |
| Package manager | pnpm 10.28.2 (workspaces) |
| MCP SDK | `@modelcontextprotocol/sdk` v2 — `server.tool()` only, never `setRequestHandler` |
| Tool names | snake_case |
| npm scope | `@zen-sci` |
| SDK pattern | `createZenSciServer()` factory → `ctx.server.tool()` |
| Error types | `ConversionErrorData` (wire) + `ConversionError` (runtime) |
| Python bridge | `PythonEngine.runJSON()` via stdin/stdout JSON |
| License | Apache 2.0 |
| Build | `cd zen-sci && pnpm run build` |
| Test | `cd zen-sci && pnpm run test` |

### Critical TypeScript Pattern
```typescript
// exactOptionalPropertyTypes: true — NEVER assign T | undefined to T?
// Pattern: build required fields first, then conditionally assign optionals
const args: SomeArgs = { required: rawArgs.required };
if (rawArgs.optional !== undefined) args.optional = rawArgs.optional;
```

### Key Implementation Patterns Confirmed Across Phases 0–3
- `MathValidator.validate()` is **async**, takes 2 args: `(latex, mode: 'inline'|'display')`
- `gray-matter` auto-parses ISO dates to `Date` objects — `escapeHtml` must handle non-string
- `GrantSubmissionFormat` types are defined locally in `grant-mcp`, not in `core`
- `MathValidator` has `unavailable` flag for graceful degradation when Python not installed
- Static signal handler registry in `TempFileManager` avoids `MaxListenersExceededWarning`
- All tool result types include `elapsed_ms: number`

---

## State of the Codebase at Handoff

### Test Counts (493 total, all passing)
| Package | Tests |
|---|---|
| packages/core | 129 |
| packages/sdk | 68 |
| servers/latex-mcp | 37 |
| servers/blog-mcp | 76 |
| servers/grant-mcp | 50 |
| servers/slides-mcp | 34 |
| servers/newsletter-mcp | 63 |
| servers/paper-mcp | 36 |

### Bug Fixes Applied This Session (not in original implementation)
| # | File | Fix |
|---|---|---|
| 1 | `latex-mcp/tools/convert-to-pdf.ts` | `generateBasicLatex` — frontmatter state machine to skip YAML lines |
| 2 | `slides-mcp/rendering/beamer-renderer.ts` | `escapeLatex` — all 9 LaTeX special chars |
| 3 | `newsletter-mcp/rendering/block-renderer.ts` | `renderList()` for `<ul>`/`<ol>` |
| 4 | `newsletter-mcp/rendering/mjml-builder.ts` | XSS: `escapeHtml` before markdown regex; list detection |
| 5 | `paper-mcp/templates/template-registry.ts` | Strip `paper-` prefix from dir names |
| 6 | `paper-mcp/tools/convert-to-paper.ts` | Format-specific bib styles (ieeetr/acm/plainnat) |
| 7 | `sdk/integration/python-engine.ts` | `checkAvailable()` — removed dead broken calls |
| 8 | `sdk/utils/temp-file-manager.ts` | Static signal handler registry |

### Spec Conformance Fixes Applied This Session
| Fix | Files |
|---|---|
| `elapsed_ms` added to all 7 result types | `convert-to-pdf`, `convert-to-html`, `generate-proposal`, `convert-to-email`, `convert-to-slides`, `validate-deck`, `convert-to-paper` |
| `elapsed_ms` added to `validate_document` | `latex-mcp/tools/validate-document.ts` |
| `elapsed_ms` added to `check_citations` | `latex-mcp/tools/check-citations.ts` |
| `fromName`, `fromEmail`, `replyTo`, `platform` added | `newsletter-mcp/tools/convert-to-email.ts`, `newsletter-mcp/src/index.ts` |
| `'both'` format added to `convert_to_slides` | `slides-mcp/tools/convert-to-slides.ts`, `slides-mcp/src/index.ts` |
| `list_slide_themes` tool added | `slides-mcp/tools/list-slide-themes.ts`, `slides-mcp/src/index.ts` |
| `list_templates` tool added | `paper-mcp/tools/list-templates.ts`, `paper-mcp/src/index.ts` |
| grant-mcp spec updated to canonize sections-array model | `docs/specs/phase-2/grant-mcp-spec.md` §3.2 |

### Registered Tool Names (final, canonical)
| Server | Tool Names |
|---|---|
| latex-mcp | `convert_to_pdf`, `validate_document`, `check_citations` |
| blog-mcp | `convert_to_html`, `generate_feed`, `validate_post` |
| grant-mcp | `generate_proposal`, `validate_compliance`, `check_format` |
| slides-mcp | `convert_to_slides`, `validate_deck`, `list_slide_themes` |
| newsletter-mcp | `convert_to_email`, `validate_email` |
| paper-mcp | `convert_to_paper`, `validate_submission`, `list_templates` |

---

## Task Definition for Receiver

Implement **Phase 4: MCP Apps** using the spec and prompt already prepared:

1. **Read the Phase 4 prompt:** `docs/workspace/2026-02-18_phase-4-implementation-prompt.md`
2. **Read the Phase 4 specs:** `docs/specs/phase-4/` (all files)
3. **Read project context:** `docs/CONTEXT.md`, `docs/STATUS.md`
4. **Follow the same patterns** used in Phases 0–3 (see Tech Stack above)
5. **Build and test** after each package: `cd zen-sci && pnpm run build && pnpm run test`
6. **All existing 493 tests must continue to pass** after Phase 4 additions

**Do not** begin implementation until you have read all Phase 4 specs and the phase prompt fully.

---

## Definition of Done

- [ ] All Phase 4 packages specified in `docs/workspace/2026-02-18_phase-4-implementation-prompt.md` are implemented
- [ ] `cd zen-sci && pnpm run build` exits 0 with no TypeScript errors
- [ ] `cd zen-sci && pnpm run test` shows all tests passing (≥493 existing + new Phase 4 tests)
- [ ] No regressions in existing 8 packages
- [ ] `docs/STATUS.md` updated to reflect Phase 4 completion
- [ ] All new MCP tools follow snake_case naming and `server.tool()` registration pattern
- [ ] All new result types include `elapsed_ms: number`
- [ ] `exactOptionalPropertyTypes` pattern followed throughout (no `T | undefined` assigned to `T?`)

---

## Constraints & Boundaries

- **DO NOT** use `setRequestHandler` — use `server.tool()` only
- **DO NOT** add `| string` escape hatches to enums
- **DO NOT** modify any existing Phase 0–3 files unless a Phase 4 requirement explicitly requires it
- **DO NOT** add new top-level npm dependencies without checking if `@zen-sci/core` or `@zen-sci/sdk` already provides the functionality
- **MUST** follow the `createZenSciServer()` factory pattern for all new MCP servers
- **MUST** use the `exactOptionalPropertyTypes` conditional-assignment pattern
- **MUST** run `pnpm run build` before `pnpm run test` (build artifacts required)
- **MUST** keep the monorepo root at `zen-sci/` — do not rename or restructure

---

## Key Resources

| Resource | Path |
|---|---|
| Phase 4 implementation prompt | `docs/workspace/2026-02-18_phase-4-implementation-prompt.md` |
| Phase 4 specs | `docs/specs/phase-4/` |
| Project context | `docs/CONTEXT.md` |
| Status tracker | `docs/STATUS.md` |
| Audit findings | `docs/workspace/2026-02-18_audit-findings.md` |
| SDK factory pattern | `zen-sci/packages/sdk/src/factory/create-server.ts` |
| Example server (most complete) | `zen-sci/servers/blog-mcp/src/` |
| Example server (most recent) | `zen-sci/servers/paper-mcp/src/` |
| Agent memory | `/Users/alfonsomorales/.claude/projects/-Users-alfonsomorales-ZenflowProjects-ZenithScience/memory/MEMORY.md` |

---

## Next Steps After Completion

1. Update `docs/STATUS.md` — mark Phase 4 complete, record test count
2. Update agent `MEMORY.md` — record Phase 4 state and any new patterns discovered
3. Run a final `pnpm run build && pnpm run test` from `zen-sci/` and confirm count
4. Create a new handoff if further phases exist, or notify the supervisor that the project is complete
