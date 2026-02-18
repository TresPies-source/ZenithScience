# Agent Handoff Package — Handoff Zero

**From:** ZenSci Architecture (Cruz Morales / TresPiesDesign.com)
**To:** Any incoming agent (strategic or coding)
**Date:** 2026-02-18
**Subject:** ZenSci project initialization — full context transfer

---

## 1. Objective

> Onboard fully to the ZenSci project, orient yourself to the workspace, and execute your assigned role — either continuing strategic/architectural work or beginning implementation of `packages/core`.

---

## 2. Required Context

Read these files **in order** before doing any work. Do not skip ahead.

### Tier 1 — Must Read (All Agents)

| File | Why |
|------|-----|
| `ZenithScience/project_context.md` | **Start here.** Filesystem map, key decisions, vocabulary, tech stack, reading order by agent type. |
| `ZenithScience/CONTEXT.md` | All 5 formal architecture decisions + open strategic questions. Immutable. |
| `ZenithScience/STATUS.md` | Live project state — current phase, active workstreams, blockers. |

### Tier 2 — Must Read (Role-Dependent)

**Strategic agents → read all of these:**

| File | Why |
|------|-----|
| `ZenithScience/architecture/2026-02-17_semantic-clusters.md` | The 9 behavioral clusters that define what the system does. The system's behavioral identity. |
| `ZenithScience/scouts/2026-02-17_module-strategy-scout.md` | Shipping order rationale, 14-module ecosystem, Phase 1–3 breakdown. |
| `ZenithScience/research/2026-02-17_naming-research.md` | GitHub org / npm scope decisions. Read the ⚠️ DECISION APPLIED section at top. |
| `ZenithScience/schemas/SCHEMAS.md` | 35+ data contracts backlog with spec status. |

**Coding agents → read all of these:**

| File | Why |
|------|-----|
| `ZenithScience/specs/infrastructure/packages-core-spec.md` | **Critical.** All shared TypeScript types, `DocumentRequest`/`DocumentResponse`, `MarkdownParser`, `CitationManager`, `MathValidator`, `ConversionPipeline`. Every module depends on this. |
| `ZenithScience/specs/infrastructure/packages-sdk-spec.md` | **Critical.** `createZenSciServer()` factory → `ZenSciContext`. `PandocWrapper`, `PythonEngine`, `MCPErrorHandler`, `ModuleManifest`. Every server uses this (composition, not inheritance). |
| `ZenithScience/specs/phase-1/latex-mcp-v0.1-spec.md` | The flagship. First module to implement. Build this only after `packages/core` and `packages/sdk` are complete. |
| `ZenithScience/zen-sci/README.md` | Inspect the empty scaffold — understand the directory structure before writing code. |

### Tier 3 — Reference as Needed

| File | When to read |
|------|-------------|
| `specs/phase-1/blog-mcp-v0.2-spec.md` | When blog-mcp implementation begins |
| `specs/phase-1/lab-notebook-mcp-spec.md` | Strategic differentiator — read to understand thinking-partner positioning |
| `specs/phase-2/*.md` (8 files) | When planning Phase 2 or auditing cross-module consistency |
| `specs/phase-3/*.md` (5 files) | Lower priority; read only if Phase 3 triggers emerge |
| `workspace/2026-02-18_opus-onboarding-prompt.md` | If performing a spec audit pass |

---

## 3. Task Definition

### If you are a Strategic / Architectural Agent

Your job is to advance the pre-code architectural work. Depending on what Cruz assigns, your task may be one or more of:

**3A. Spec Audit (highest priority)**
Audit all 19 specs against the criteria in `workspace/2026-02-18_opus-onboarding-prompt.md`. Specifically:
- Verify `packages-core-spec.md` TypeScript types against current `@modelcontextprotocol/sdk` API (use Context7)
- Check cross-spec consistency: all modules must agree on `DocumentRequest`/`DocumentResponse` shapes
- Confirm `OutputFormat` enum in `packages-core-spec.md` covers every format each module needs
- Flag any spec with placeholder text, vague risk tables, or missing week-by-week breakdowns
- Write findings to `workspace/[date]_audit-findings.md`

**3B. Schema Formalization**
Formally write TypeScript `interface` definitions for priority schemas in `schemas/`. Start with:
1. `DocumentRequest` and `DocumentResponse` (from `packages-core-spec.md` — formalize into standalone `.ts` types)
2. `ConversionPipeline` and `PipelineStage`
3. `CitationRecord`

Write outputs to `zen-sci/packages/core/src/types/` with accompanying `.test.ts` files.

**3C. Shipping Order**
Cruz confirmed latex-first shipping order (2026-02-18). Sequence: `latex-mcp` (v0.1) → `blog-mcp` (v0.2) → `slides-mcp` + `newsletter-mcp` (v0.3, parallel) → `grant-mcp` (v0.4). The rationale: solve the hardest problem (TeX pipeline) first; blog-mcp validates the core abstraction generality as second test.

---

### If you are a Coding / Implementation Agent

**Start with `packages/core` — nothing else can be built without it.**

Your implementation sequence:

**Step 1: Scaffold `packages/core`**
```
zen-sci/packages/core/
├── package.json          (name: @zen-sci/core, version: 0.0.1)
├── tsconfig.json
├── src/
│   ├── index.ts          (public exports)
│   ├── types/
│   │   ├── document.ts   (DocumentRequest, DocumentResponse, OutputFormat, ...)
│   │   ├── pipeline.ts   (ConversionPipeline, PipelineStage, ...)
│   │   ├── citation.ts   (CitationRecord, BibliographyStyle, ...)
│   │   └── errors.ts     (ConversionError, ValidationError, ParseError, ...)
│   ├── parsing/
│   │   └── MarkdownParser.ts
│   ├── citations/
│   │   └── CitationManager.ts
│   └── validation/
│       ├── SchemaValidator.ts
│       └── MathValidator.ts
└── tests/
    └── ...
```

Read `specs/infrastructure/packages-core-spec.md` for exact type definitions and class APIs.

**Step 2: Scaffold `packages/sdk`**
```
zen-sci/packages/sdk/
├── package.json          (name: @zen-sci/sdk, depends on @zen-sci/core)
├── src/
│   ├── factory.ts         (createZenSciServer() → ZenSciContext)
│   ├── types/
│   │   └── engine.ts      (ModuleManifest, PandocJob, PandocResult)
│   ├── engines/
│   │   ├── PandocWrapper.ts
│   │   └── PythonEngine.ts
│   └── error/
│       └── MCPErrorHandler.ts
```

Read `specs/infrastructure/packages-sdk-spec.md` for the `createZenSciServer()` factory signature and `ZenSciContext` interface. There is NO `ZenSciServer` abstract base class — Decision D1 resolved this as composition over inheritance.

**Step 3: Implement `servers/latex-mcp`**
Only begin after `packages/core` and `packages/sdk` have passing unit tests.
Read `specs/phase-1/latex-mcp-v0.1-spec.md` fully before writing a single line.

---

## 4. Definition of Done

### For Strategic Agents (Spec Audit path)

- [ ] All 19 spec files reviewed; any with issues have an `## Audit Notes` section added at top
- [ ] `packages-core-spec.md` TypeScript types verified against `@modelcontextprotocol/sdk` current API
- [ ] Cross-spec `DocumentRequest` shape is consistent across all 19 specs
- [ ] `workspace/[date]_audit-findings.md` written with: total specs reviewed, issues by category, key improvements, open questions
- [ ] `STATUS.md` updated: spec status changed from "Draft" to "Reviewed" for passing specs
- [ ] `SCHEMAS.md` updated if audit reveals any missing data contracts

### For Coding Agents (`packages/core` path)

- [ ] `zen-sci/packages/core/package.json` created with correct name (`@zen-sci/core`), version, dependencies
- [ ] All TypeScript interfaces from `packages-core-spec.md` implemented (no `any` types)
- [ ] `MarkdownParser` class implemented with unit tests passing
- [ ] `CitationManager` class implemented with unit tests passing
- [ ] `SchemaValidator` class implemented using Zod, with tests
- [ ] `DocumentRequest` validates correctly against test fixtures
- [ ] `npm run build` in `packages/core` exits 0
- [ ] `npm run test` in `packages/core` passes with ≥80% coverage
- [ ] `STATUS.md` updated: `packages/core` workstream marked ✅ with version

---

## 5. Constraints & Boundaries

- **DO NOT** modify files in `architecture/`, `scouts/`, or `research/` — these are final decision artifacts
- **DO NOT** override architecture decisions in `CONTEXT.md` — they are set by Cruz Morales
- **DO NOT** change directory naming from kebab-case (Decision 5: `zen-sci/`, `latex-mcp/`, `@zen-sci`)
- **DO NOT** use `@typecraft` as npm scope — the decision is `@zen-sci` (see Decision 4)
- **MUST** update `STATUS.md` after completing significant work
- **MUST** use kebab-case for all new directories and package names
- **MUST** target `zen-sci/` for all code; decision artifacts stay in `ZenithScience/` root
- Coding agents: **do not start servers until `packages/core` tests pass**

---

## 6. Project Identity Quick Reference

```
Product:      ZenSci
Full name:    ZenithScience
Domain:       ZenithScience.org
GitHub org:   TresPies-source
Monorepo:     github.com/TresPies-source/ZenithScience
npm scope:    @zen-sci
IP Owner:     TresPiesDesign.com / Cruz Morales
License:      Apache 2.0
Phase:        Specifications Complete / Pre-Code
```

---

## 7. Behavioral Identity (Memorize This)

> "ZenithScience is a human-AI collaboration engine that converts unstructured thinking into publication-ready artifacts."

The **thinking partner** positioning is the differentiator — not the format conversion. Converters are commodities; ZenSci helps you think better, then proves it by rendering beautifully. Every architectural decision should reinforce this: the `ThinkingSession` schema, the `lab-notebook-mcp` as Phase 1 differentiator, the FORMALIZE verb in the semantic cluster vocabulary.

---

## 8. Semantic Cluster Vocabulary (Use Consistently in Code & Docs)

| Verb | What it maps to |
|------|----------------|
| PARSE | `MarkdownParser`, `CitationManager.parse()`, frontmatter extraction |
| FORMALIZE | `ThinkingSession`, reasoning capture, proof structuring |
| STRUCTURE | Template engine, section hierarchy, `DocumentSection` |
| CONVERT | `pandoc` subprocess calls, format-specific rendering rules |
| VALIDATE | `SchemaValidator`, `MathValidator`, `LinkChecker` |
| RENDER | TeX engine, MJML compiler, HTML generation |

---

## 9. Next Steps After Completion

**After strategic audit:** Hand off to Cruz Morales with `workspace/[date]_audit-findings.md`. Cruz will confirm shipping order and any spec changes before coding begins.

**After `packages/core` implementation:** Hand off to `packages/sdk` implementer (may be same agent). Include test coverage report and any deviations from spec that were made during implementation (with rationale).

**After `packages/sdk` implementation:** Hand off to `latex-mcp` implementer with both packages published to local npm workspace (`npm link` or workspace protocol).

---

---

## 10. Implementation Prompts (Coding Agent Entry Points)

All four implementation phases have complete, self-contained prompts ready for coding agent commission. **For coding agents, these prompts supersede the task definitions in §3 above** — they contain the full context, file manifests, success criteria, and DO NOT constraints needed for autonomous execution.

| Prompt | Scope | New Files | Key Requirements |
|--------|-------|-----------|-----------------|
| `workspace/2026-02-18_phase-0-implementation-prompt.md` | Monorepo init + packages/core | 29 | 32 |
| `workspace/2026-02-18_phase-1-implementation-prompt.md` | packages/sdk + latex-mcp v0.1 | 37 | 38 |
| `workspace/2026-02-18_phase-2-implementation-prompt.md` | blog-mcp v0.2 + grant-mcp v0.4 | 45 | — |
| `workspace/2026-02-18_phase-3-implementation-prompt.md` | slides-mcp v0.3 + newsletter-mcp v0.3 + paper-mcp | 49 | — |

**Execution order:** Phase 0 must complete before Phase 1 can start. Phase 1 must complete before Phase 2 and Phase 3. Phases 2 and 3 can run in parallel once Phase 1 passes.

**Architectural invariants enforced across all prompts (never violate):**
- MCP SDK v2 high-level API: `server.registerTool()` only — NOT `setRequestHandler` + schema handlers
- All MCP tool names in snake_case: `convert_to_pdf`, `validate_document`, `generate_feed`, etc.
- TypeScript ↔ Python bridge via `PythonEngine.runJSON()` (stdin/stdout JSON) — never file-based IPC
- `createZenSciServer()` factory pattern — no `ZenSciServer` abstract class anywhere
- Strict enums everywhere — no `| string` escape hatches
- `ConversionErrorData` (interface/JSON wire type) + `ConversionError` (runtime class) — always both present
- pandoc >= 3.0 checked at `PandocWrapper` construction time
- `og:type` is always `'article'` for blog posts — `'blog'` is not a valid OG type
- `nsf-research-gov` not `nsf-fastlane` (FastLane retired 2023)

---

*Handoff prepared by: ZenSci Architecture (Cruz Morales / TresPiesDesign.com)*
*Protocol: agent-orchestration:handoff-protocol v1.0*
*Date: 2026-02-18 (updated: 2026-02-18 — added §10, D1 SDK correction, license confirmed, shipping order confirmed)*
