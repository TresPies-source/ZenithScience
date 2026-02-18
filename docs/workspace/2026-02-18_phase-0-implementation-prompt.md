# Implementation Commission: Phase 0 — Monorepo Foundation + packages/core

**Objective:** Initialize the zen-sci pnpm monorepo with shared tooling and implement `packages/core` in full — all TypeScript types, class implementations, and tests — so that every subsequent module has a stable, tested foundation to build on.

**Commissioned by:** Cruz Morales / TresPiesDesign.com
**Date:** 2026-02-18
**License:** Apache 2.0
**Prereqs:** Node.js >= 20 LTS, pnpm >= 8, Python >= 3.11, pandoc >= 3.0, TeX Live (full), SymPy
**Blocking:** Phase 1 cannot start until `npm run build` and `npm run test` pass in `packages/core` with ≥ 80% coverage.

---

## 1. Context & Grounding

**Primary Specifications (read these fully before writing a single line):**
- `ZenithScience/specs/infrastructure/packages-core-spec.md` — Complete type definitions (§3.1–3.4), class APIs (§4), testing strategy (§5), package.json (§6). This is the authoritative source for every interface, class, method signature, and directory name in this phase.
- `ZenithScience/specs/infrastructure/packages-sdk-spec.md` — Read §1 and §2 only to understand what packages/sdk will need from packages/core. Do NOT implement sdk in this phase.
- `ZenithScience/CONTEXT.md` — Architecture decisions (kebab-case everywhere, strict enums, ConversionErrorData split, composition over inheritance).
- `ZenithScience/STATUS.md` — Current state; understand what already exists before creating files.

**Pattern Files:**
- `packages-core-spec.md §3.1` — Implements DocumentRequest / DocumentResponse types exactly as written; do not add fields.
- `packages-core-spec.md §4.1` — MarkdownParser class: follow the exact method signatures (`parse()`, `toAST()`, `toString()`).
- `packages-core-spec.md §4.3` — SchemaValidator: uses Zod v3; follow the exact validator factory pattern.

**Current State:**
- `ZenithScience/zen-sci/` exists with a README.md only. No package.json, no packages/, no servers/.
- No production code exists anywhere in the project. All prior work is architectural (specs, scouts, schemas).

---

## 2. Detailed Requirements

### 2.1 — Initialize the pnpm Monorepo

1. Create `zen-sci/package.json` as the workspace root (private: true, name: "zen-sci-monorepo", version: "0.0.1"). Include scripts: `build`, `test`, `lint`, `typecheck` that delegate to all workspaces via `pnpm -r run`.

2. Create `zen-sci/pnpm-workspace.yaml` declaring packages:
   ```yaml
   packages:
     - 'packages/*'
     - 'servers/*'
   ```

3. Create `zen-sci/tsconfig.base.json` with shared TypeScript config:
   - `"target": "ES2022"`, `"module": "NodeNext"`, `"moduleResolution": "NodeNext"`
   - `"strict": true`, `"noUncheckedIndexedAccess": true`, `"exactOptionalPropertyTypes": true`
   - `"declaration": true`, `"declarationMap": true`, `"sourceMap": true`
   - `"outDir": "./dist"`, `"rootDir": "./src"`

4. Create `zen-sci/vitest.config.ts` at workspace root with coverage provider `v8`, coverage thresholds of 80% for lines/statements/branches/functions.

5. Create `zen-sci/.eslintrc.json` with `@typescript-eslint/recommended` and `no-explicit-any` as `error`.

6. Create `zen-sci/.prettierrc` with `singleQuote: true`, `trailingComma: 'all'`, `printWidth: 100`.

7. Create `zen-sci/.gitignore` covering: `node_modules/`, `dist/`, `*.log`, `.env`, `coverage/`, `*.tsbuildinfo`.

### 2.2 — Scaffold packages/core directory

8. Create the following directory structure exactly:
   ```
   zen-sci/packages/core/
   ├── src/
   │   ├── index.ts
   │   ├── types.ts
   │   ├── parsing/
   │   │   ├── markdown-parser.ts
   │   │   ├── frontmatter-extractor.ts
   │   │   └── ast-node-types.ts
   │   ├── citations/
   │   │   ├── citation-manager.ts
   │   │   ├── bibtex-parser.ts
   │   │   └── csl-renderer.ts
   │   ├── validation/
   │   │   ├── schema-validator.ts
   │   │   ├── math-validator.ts
   │   │   └── link-checker.ts
   │   ├── pipeline/
   │   │   └── conversion-pipeline.ts
   │   └── utils/
   │       ├── logger.ts
   │       └── error-handler.ts
   ├── tests/
   │   ├── parsing.test.ts
   │   ├── citations.test.ts
   │   ├── validation.test.ts
   │   └── integration.test.ts
   ├── package.json
   ├── tsconfig.json
   └── README.md
   ```

9. Create `zen-sci/packages/core/package.json`:
   ```json
   {
     "name": "@zen-sci/core",
     "version": "0.0.1",
     "description": "ZenSci shared parsing, pipeline, citation, and validation library",
     "type": "module",
     "main": "./dist/index.js",
     "types": "./dist/index.d.ts",
     "exports": {
       ".": {
         "import": "./dist/index.js",
         "types": "./dist/index.d.ts"
       }
     },
     "scripts": {
       "build": "tsc --build",
       "test": "vitest run --coverage",
       "typecheck": "tsc --noEmit",
       "lint": "eslint src --ext .ts"
     },
     "dependencies": {
       "gray-matter": "^4.0.3",
       "remark": "^15.0.1",
       "remark-frontmatter": "^5.0.0",
       "remark-gfm": "^4.0.0",
       "remark-parse": "^11.0.0",
       "unified": "^11.0.4",
       "unist-util-visit": "^5.0.0",
       "zod": "^3.22.4",
       "bibtex-js": "^0.1.2"
     },
     "devDependencies": {
       "@types/node": "^20.0.0",
       "typescript": "^5.3.0",
       "vitest": "^1.2.0",
       "@vitest/coverage-v8": "^1.2.0"
     }
   }
   ```

10. Create `zen-sci/packages/core/tsconfig.json` extending `../../tsconfig.base.json` with `references` pointing to no other workspace packages (core has no internal dependencies).

### 2.3 — Implement src/types.ts

11. Implement ALL TypeScript interfaces exactly as specified in `packages-core-spec.md §3.1 through §3.4`. This includes:
    - `DocumentRequest`, `OutputFormat` (24-value union), `DocumentOptions`, `FrontmatterMetadata`, `BibliographyOptions`, `BibliographyStyle`, `MathOptions` (§3.1)
    - `DocumentResponse`, `Artifact`, `ConversionWarning`, `ConversionErrorData`, `DocumentLocation`, `ResponseMetadata`, `ConversionPipeline`, `PipelineStage` (§3.1 continued)
    - `ThinkingSession`, `ReasoningStep`, `SessionDecision`, `ValidationResult` (§3.1 continued)
    - `DocumentTree`, `DocumentSection`, `DocumentNode`, `HeadingNode`, `ParagraphNode`, `CodeBlockNode`, `MathNode`, `InlineMathNode`, `CitationNode`, `CrossReferenceNode`, `ImageNode`, `TableNode`, `FootnoteNode`, `FrontmatterNode` — all AST node types (§3.2)
    - `CitationRecord`, `ParsedBibEntry`, `CSLEntry`, `ResolvedCitation`, `BibliographyList` — citation types (§3.3)
    - `ConversionRule`, `FormatConstraint`, `ImageProcessingOptions`, `ImageProcessingResult`, `AccessibilityReport`, `AccessibilityViolation`, `AccessibilityWarning`, `WebMetadataSchema`, `OpenGraphMetadata`, `TwitterCardMetadata`, `SchemaOrgData`, `DocumentVersion`, `DocumentChange`, `DocumentVersionHistory` — extension types (§3.4)
    - **All strict enum constraints from the audit decisions apply**: no `| string` escape hatches, `validatedAt` is `string` not `Date`, `aspectRatio` is strict enum.
    - Export everything with `export type *` at bottom of file.

### 2.4 — Implement Parsing Cluster

12. Implement `src/parsing/ast-node-types.ts` — re-exports all AST node interfaces from types.ts plus any remark/unist utility types needed internally. This file acts as the AST type hub for the parsing module.

13. Implement `src/parsing/frontmatter-extractor.ts` — `FrontmatterExtractor` class per spec §4.2:
    - `extract(source: string): FrontmatterMetadata` — uses gray-matter to parse YAML frontmatter
    - `strip(source: string): string` — returns markdown with frontmatter removed
    - Handles missing frontmatter gracefully (returns empty `{}`)
    - Normalizes `author` field: accepts string or array, always returns string[]

14. Implement `src/parsing/markdown-parser.ts` — `MarkdownParser` class per spec §4.1:
    - `parse(source: string): DocumentTree` — uses remark + remark-gfm + remark-frontmatter to produce a DocumentTree
    - `toAST(source: string): unknown` — returns raw unified AST (mdast) for inspection
    - `toString(tree: DocumentTree): string` — serializes tree back to markdown (round-trip)
    - Strips frontmatter from content before tree building (delegates to FrontmatterExtractor)
    - All AST nodes map to the DocumentNode union type from types.ts

### 2.5 — Implement Citations Cluster

15. Implement `src/citations/bibtex-parser.ts` — `BibTeXParser` class per spec §4.4:
    - `parse(bibtexContent: string): ParsedBibEntry[]` — uses bibtex-js to parse .bib file content
    - `validate(entries: ParsedBibEntry[]): ValidationResult` — checks required fields per entry type (article needs author, title, year, journal; book needs author, title, year, publisher)
    - Handles malformed BibTeX gracefully (returns partial results with errors in ValidationResult)

16. Implement `src/citations/csl-renderer.ts` — `CSLRenderer` class per spec §4.5:
    - `render(entries: ParsedBibEntry[], style: BibliographyStyle, format: 'html' | 'latex' | 'plain'): BibliographyList` — renders citations in target format
    - `renderInline(key: string, entries: ParsedBibEntry[], style: BibliographyStyle): string` — renders a single inline citation (e.g., "[1]" for IEEE, "(Smith, 2023)" for APA)
    - For v0.1: implement IEEE and APA styles manually; use Pandoc's `--citeproc` flag via subprocess for other styles

17. Implement `src/citations/citation-manager.ts` — `CitationManager` class per spec §4.3:
    - Constructor: `(bibliographyContent: string)` — initializes BibTeXParser and CSLRenderer
    - `resolve(key: string): ParsedBibEntry | null` — looks up citation by BibTeX key
    - `extractKeysFromAST(tree: DocumentTree): string[]` — walks AST to find all CitationNode keys
    - `validateAll(tree: DocumentTree): ValidationResult` — checks that all citation keys resolve
    - `renderBibliography(style: BibliographyStyle, format: 'html' | 'latex' | 'plain'): BibliographyList`

### 2.6 — Implement Validation Cluster

18. Implement `src/validation/schema-validator.ts` — `SchemaValidator` class per spec §4.6:
    - Define Zod schemas for `DocumentRequest` and all nested types
    - `validate(request: unknown, supportedFormats: OutputFormat[]): ValidationResult` — validates and checks format is in supportedFormats list
    - `validateFrontmatter(frontmatter: unknown): ValidationResult` — validates YAML frontmatter shape
    - Static methods only (no instance needed); exported as `SchemaValidator` with static interface

19. Implement `src/validation/math-validator.ts` — `MathValidator` class per spec §4.7:
    - `validate(expression: string, context: 'inline' | 'display'): ValidationResult` — calls Python (SymPy) via subprocess to validate math syntax
    - Python script path: `src/validation/math_validator.py` (create this Python file)
    - `validateAll(tree: DocumentTree): ValidationResult` — finds all MathNode and InlineMathNode in tree, validates each
    - **Python subprocess bridge pattern:** spawn `python3 -c "import sympy; sympy.parse_expr(expr)"` or use a dedicated Python script. Capture stdout/stderr. If Python is not available, return a warning (not an error) and skip validation.
    - Timeout: 5 seconds per expression. If timeout, return warning.

20. Implement `src/validation/link-checker.ts` — `LinkChecker` class per spec §4.8:
    - `check(tree: DocumentTree): Promise<ValidationResult>` — finds all external URLs in tree, sends HEAD requests
    - `checkCrossRefs(tree: DocumentTree): ValidationResult` — checks internal cross-references (§ labels, figure refs) are defined
    - Rate-limit external checks to 10 concurrent requests
    - Timeout: 3 seconds per URL. Dead links are warnings, not errors.

### 2.7 — Implement Pipeline

21. Implement `src/pipeline/conversion-pipeline.ts` — `ConversionPipeline` class per spec §4.9. This is the **runtime class** (not the interface from types.ts):
    - Constructor: `(requestId: string)` — creates a new pipeline with unique UUID
    - `start(): void` — sets status to 'running', records startedAt
    - `startStage(name: PipelineStage['name']): void` — adds/starts a stage
    - `completeStage(name: PipelineStage['name']): void` — marks stage complete, records elapsed
    - `failStage(name: PipelineStage['name'], error: ConversionErrorData): void` — marks stage failed
    - `complete(): void` — sets overall status to 'completed'
    - `fail(error: ConversionErrorData): void` — sets overall status to 'failed'
    - `toJSON(): ConversionPipeline` — returns the wire-safe ConversionPipeline interface (from types.ts)
    - Export class as `ConversionPipeline` (same name as interface — TypeScript structural typing handles this)

### 2.8 — Implement Utilities

22. Implement `src/utils/logger.ts` — simple structured logger:
    - `Logger` class with `info()`, `warn()`, `error()`, `debug()` methods
    - Each method accepts: `(message: string, data?: Record<string, unknown>): void`
    - Output to stderr as JSON: `{ level, message, timestamp, ...data }`
    - Log level controlled by `LOG_LEVEL` environment variable (default: 'info')

23. Implement `src/utils/error-handler.ts` — `ConversionError` class (the throwable runtime version):
    - `class ConversionError extends Error` with `code: string`, `location?: DocumentLocation`, `suggestions?: string[]`
    - `static fromData(data: ConversionErrorData): ConversionError` — factory method
    - `toData(): ConversionErrorData` — serialization bridge
    - Constructor: `(code: string, message: string, options?: { location?: DocumentLocation; suggestions?: string[] })`

### 2.9 — Implement src/index.ts

24. Export all public classes and types from index.ts exactly matching the public API surface in `packages-core-spec.md §2.1`. Every export listed in that spec must be present. Nothing extra; nothing missing.

### 2.10 — Write Tests

25. Implement `tests/parsing.test.ts` — tests for MarkdownParser and FrontmatterExtractor:
    - Parse markdown with YAML frontmatter → DocumentTree has correct nodes
    - Headings at all levels h1–h6 appear as HeadingNode with correct depth
    - Code blocks with language fence → CodeBlockNode with `lang` property
    - Math fences (`$inline$`, `$$display$$`) → MathNode / InlineMathNode
    - Round-trip: `parser.toString(parser.parse(source))` preserves content
    - Edge cases: empty source, source with only frontmatter, unicode content

26. Implement `tests/citations.test.ts` — tests for BibTeXParser, CSLRenderer, CitationManager:
    - Valid BibTeX parses correctly → array of ParsedBibEntry
    - Malformed BibTeX returns partial results + errors in ValidationResult
    - IEEE rendering of article entry produces `[N]` inline + correct bibliography line
    - APA rendering produces `(Author, Year)` inline + correct bibliography line
    - CitationManager.extractKeysFromAST finds all citation nodes in a tree
    - Missing citation key produces warning in ValidationResult

27. Implement `tests/validation.test.ts` — tests for SchemaValidator, MathValidator, LinkChecker:
    - Valid DocumentRequest passes validation
    - Missing required `source` field produces error
    - Format not in supportedFormats produces error
    - Valid LaTeX math (`x^2 + y^2`) passes MathValidator (or skips if Python unavailable)
    - Invalid math (`\fracxy`) fails MathValidator with code 'invalid-math'
    - Internal cross-ref to undefined label produces warning in LinkChecker

28. Implement `tests/integration.test.ts` — end-to-end parsing pipeline:
    - Full markdown document (with frontmatter, headings, math, code, citations) → DocumentTree
    - SchemaValidator validates a complete DocumentRequest with all fields populated
    - ConversionPipeline tracks all stages from start to complete

### 2.11 — Final Verification

29. Run `pnpm install` from `zen-sci/` root. Verify no peer dependency errors.

30. Run `pnpm --filter @zen-sci/core run build`. Must exit 0. No TypeScript errors. No `any` type usage.

31. Run `pnpm --filter @zen-sci/core run test`. Must exit 0. Coverage must be ≥ 80% lines/statements/branches/functions.

32. Update `ZenithScience/STATUS.md`: mark `packages/core` workstream as ✅ with version 0.0.1. Note build and test status.

---

## 3. File Manifest

### Create (new files):
```
zen-sci/package.json
zen-sci/pnpm-workspace.yaml
zen-sci/tsconfig.base.json
zen-sci/vitest.config.ts
zen-sci/.eslintrc.json
zen-sci/.prettierrc
zen-sci/.gitignore
zen-sci/packages/core/package.json
zen-sci/packages/core/tsconfig.json
zen-sci/packages/core/README.md
zen-sci/packages/core/src/index.ts
zen-sci/packages/core/src/types.ts
zen-sci/packages/core/src/parsing/markdown-parser.ts
zen-sci/packages/core/src/parsing/frontmatter-extractor.ts
zen-sci/packages/core/src/parsing/ast-node-types.ts
zen-sci/packages/core/src/citations/citation-manager.ts
zen-sci/packages/core/src/citations/bibtex-parser.ts
zen-sci/packages/core/src/citations/csl-renderer.ts
zen-sci/packages/core/src/validation/schema-validator.ts
zen-sci/packages/core/src/validation/math-validator.ts
zen-sci/packages/core/src/validation/math_validator.py
zen-sci/packages/core/src/pipeline/conversion-pipeline.ts
zen-sci/packages/core/src/utils/logger.ts
zen-sci/packages/core/src/utils/error-handler.ts
zen-sci/packages/core/tests/parsing.test.ts
zen-sci/packages/core/tests/citations.test.ts
zen-sci/packages/core/tests/validation.test.ts
zen-sci/packages/core/tests/integration.test.ts
```

### Do not touch:
```
zen-sci/README.md  (exists; do not modify)
ZenithScience/specs/  (read-only)
ZenithScience/CONTEXT.md  (read-only)
ZenithScience/architecture/  (read-only)
ZenithScience/scouts/  (read-only)
```

---

## 4. Success Criteria

- [ ] `pnpm install` from `zen-sci/` exits 0 with no peer dependency errors
- [ ] `pnpm --filter @zen-sci/core run typecheck` exits 0 (no TypeScript errors, no `any` types)
- [ ] `pnpm --filter @zen-sci/core run build` exits 0 (dist/ populated with .js + .d.ts files)
- [ ] `pnpm --filter @zen-sci/core run test` exits 0 (all tests pass)
- [ ] Coverage: ≥ 80% lines, ≥ 80% statements, ≥ 80% branches, ≥ 80% functions
- [ ] `src/types.ts` exports all 50+ interfaces from §3.1–3.4 with no additions or omissions
- [ ] `src/index.ts` exports exactly what `packages-core-spec.md §2.1` specifies
- [ ] `ConversionPipeline` class and `ConversionPipeline` interface are both present and TypeScript compiles without conflict
- [ ] `ConversionError` class has `fromData()` and `toData()` bridge methods
- [ ] `ConversionErrorData` interface is distinct from `ConversionError` class
- [ ] No `any` types (ESLint `no-explicit-any: error` must pass)
- [ ] All OutputFormat union values are present (24 values)
- [ ] `BibliographyStyle` includes 'nature' and 'arxiv'
- [ ] `AspectRatio` in `FormatConstraint` is strict enum (not `string`)
- [ ] `AccessibilityReport.validatedAt` is `string` (not `Date`)
- [ ] `STATUS.md` updated to mark packages/core ✅

---

## 5. Constraints & Non-Goals

- **DO NOT** implement `packages/sdk` — that is Phase 1
- **DO NOT** implement any MCP server or tool registration — that is Phase 1
- **DO NOT** implement any servers/ module — that is Phase 1+
- **DO NOT** use abstract base classes — composition over inheritance (Decision D1)
- **DO NOT** add `| string` escape hatches to any enum/union type — all enums are strict (Decision D2)
- **DO NOT** use `any` types anywhere in source code
- **DO NOT** modify files in `ZenithScience/specs/`, `ZenithScience/architecture/`, `ZenithScience/scouts/`, or `ZenithScience/CONTEXT.md` — these are immutable decision artifacts
- **DO NOT** use `@typecraft` as npm scope — scope is `@zen-sci` (Decision 4)
- **DO NOT** use `ZenSciServer` abstract class — it does not exist in this architecture (Decision D1)
- **DO NOT** implement Python-heavy features in TypeScript — math validation calls Python as subprocess, TypeScript side only spawns the process
- **DO NOT** break the MathValidator if Python is unavailable — degrade gracefully to warning

---

## 6. Architecture Grounding

**Key architectural decisions that directly affect this phase:**

- **Decision D1 (Composition over inheritance):** No abstract base classes. `ConversionPipeline` is a class but not abstract. MathValidator, SchemaValidator, etc. are standalone classes with static or instance methods.
- **Decision D2 (Strict enums):** Every union type in types.ts must be a strict string literal union — no `| string` fallbacks.
- **Decision D3 (ConversionError split):** `ConversionErrorData` is the wire-safe interface (JSON serializable). `ConversionError` is the throwable class. They are separate and bridged by `fromData()` / `toData()`.
- **Transport agnostic:** packages/core has NO knowledge of MCP, HTTP, stdio, or any transport protocol. It operates purely on TypeScript data structures.
- **Python engine pattern:** TypeScript spawns Python subprocess with `child_process.spawn()`. Input/output via JSON on stdin/stdout. Timeout handling required.
- **OutputFormat completeness:** The 24 OutputFormat values come from all modules across all 3 phases. Do not add or remove values from the spec — this union is the coordination contract between all modules.

---

*Spec version: packages-core-spec.md v1.0.0*
*Phase: 0 of 3*
*Next: Phase 1 — packages/sdk + latex-mcp v0.1*
