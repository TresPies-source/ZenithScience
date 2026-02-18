# Implementation Commission: Phase 1 — packages/sdk + latex-mcp v0.1

**Objective:** Implement `packages/sdk` (the MCP transport layer and shared utilities) and `servers/latex-mcp` (the flagship module — markdown to publication-ready PDF via LaTeX), establishing the end-to-end conversion pipeline from MCP tool invocation to PDF artifact.

**Commissioned by:** Cruz Morales / TresPiesDesign.com
**Date:** 2026-02-18
**License:** Apache 2.0
**Prereqs:** Phase 0 complete (`pnpm --filter @zen-sci/core run test` passes with ≥ 80% coverage)
**System deps:** Node.js >= 20 LTS, pnpm >= 8, Python >= 3.11, pandoc >= 3.0, TeX Live (full), SymPy, pypandoc, bibtexparser

---

## 1. Context & Grounding

**Primary Specifications (read fully before writing):**
- `ZenithScience/specs/infrastructure/packages-sdk-spec.md` — Complete SDK architecture, `createZenSciServer()` factory, all class APIs, testing strategy. **Read all sections including §3 (class APIs) and §4 (public API surface).**
- `ZenithScience/specs/phase-1/latex-mcp-v0.1-spec.md` — LaTeX MCP vision, all MCP tool definitions (§3.2), Python engine architecture (§3.3), testing strategy (§5). **Read in full before implementing any server code.**
- `ZenithScience/specs/infrastructure/packages-core-spec.md §2.1` — Public API surface to import from `@zen-sci/core`.
- `ZenithScience/CONTEXT.md` — Decisions D1 (composition over inheritance), D2 (strict enums), D3 (ConversionError split).

**Pattern Files:**
- `zen-sci/packages/core/src/utils/logger.ts` — Follow same error-handling and logging patterns in sdk logger.
- `zen-sci/packages/core/src/pipeline/conversion-pipeline.ts` — Understand ConversionPipeline usage pattern before implementing `runConversionPipeline()` in sdk.
- `packages-sdk-spec.md §3.1` — `createZenSciServer()` factory: follow the exact return shape (`ZenSciContext`).

**Critical Architecture Note:** packages/sdk uses **composition over inheritance**. There is NO `ZenSciServer` abstract base class. Instead, modules call `createZenSciServer()` which returns a `ZenSciContext` object containing a pre-configured `McpServer` instance plus utility singletons. Modules then call `server.registerTool()` on the returned server.

---

## 2. Detailed Requirements

### 2.1 — Scaffold packages/sdk

1. Create `zen-sci/packages/sdk/` directory structure:
   ```
   zen-sci/packages/sdk/
   ├── src/
   │   ├── index.ts
   │   ├── types.ts
   │   ├── factory/
   │   │   ├── create-server.ts
   │   │   ├── module-manifest.ts
   │   │   └── tool-builder.ts
   │   ├── integration/
   │   │   ├── python-engine.ts
   │   │   ├── pandoc-wrapper.ts
   │   │   └── subprocess-pool.ts
   │   ├── transport/
   │   │   └── stdio-transport.ts
   │   ├── errors/
   │   │   ├── mcp-error-handler.ts
   │   │   └── error-mapper.ts
   │   ├── logging/
   │   │   ├── logger.ts
   │   │   └── pipeline-monitor.ts
   │   └── utils/
   │       ├── temp-file-manager.ts
   │       ├── artifact-manager.ts
   │       └── request-id-generator.ts
   ├── tests/
   │   ├── factory.test.ts
   │   ├── integration.test.ts
   │   └── error-handling.test.ts
   ├── package.json
   ├── tsconfig.json
   └── README.md
   ```

2. Create `zen-sci/packages/sdk/package.json`:
   ```json
   {
     "name": "@zen-sci/sdk",
     "version": "0.0.1",
     "description": "ZenSci MCP transport layer and shared server utilities",
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
       "typecheck": "tsc --noEmit"
     },
     "dependencies": {
       "@modelcontextprotocol/sdk": "^1.5.0",
       "@zen-sci/core": "workspace:*",
       "pino": "^8.18.0",
       "uuid": "^9.0.0"
     },
     "devDependencies": {
       "@types/node": "^20.0.0",
       "typescript": "^5.3.0",
       "vitest": "^1.2.0"
     }
   }
   ```

3. Create `zen-sci/packages/sdk/tsconfig.json` extending `../../tsconfig.base.json` with references to `../core`.

### 2.2 — Implement packages/sdk core classes

4. Implement `src/utils/request-id-generator.ts`:
   - `generateRequestId(): string` — returns a UUID v4 string

5. Implement `src/utils/temp-file-manager.ts` — `TempFileManager` class per spec §3.4:
   - Constructor: `(moduleName: string)` — creates a temp dir prefix like `/tmp/zen-sci-{module}-{pid}`
   - `createTempFile(extension: string, content?: string | Buffer): Promise<string>` — creates temp file, returns path
   - `createTempDir(): Promise<string>` — creates temp subdirectory, returns path
   - `cleanup(): Promise<void>` — deletes all temp files created by this instance
   - Register cleanup on process 'exit', 'SIGINT', 'SIGTERM' signals

6. Implement `src/utils/artifact-manager.ts` — `ArtifactManager` class per spec §3.5:
   - `register(artifact: Artifact): void` — tracks artifact by name
   - `getAll(): Artifact[]` — returns all registered artifacts
   - `getByName(name: string): Artifact | undefined`
   - `readBuffer(name: string): Promise<Buffer>` — reads artifact file into buffer
   - `clear(): void` — removes all registered artifacts

7. Implement `src/logging/logger.ts` — structured logger wrapping pino per spec §3.6:
   - Constructor: `(moduleName: string)` — creates child logger with module name
   - `info(msg: string, data?: Record<string, unknown>): void`
   - `warn(msg: string, data?: Record<string, unknown>): void`
   - `error(msg: string, data?: Record<string, unknown>): void`
   - `debug(msg: string, data?: Record<string, unknown>): void`
   - Log to stderr (pino default). Level controlled by `LOG_LEVEL` env var.

8. Implement `src/logging/pipeline-monitor.ts` — `PipelineMonitor` class:
   - Constructor: `(logger: Logger)`
   - `track(pipeline: ConversionPipeline): void` — logs stage transitions at debug level
   - `logCompletion(pipeline: ConversionPipeline): void` — logs final status + elapsed at info level
   - `logFailure(pipeline: ConversionPipeline, error: ConversionErrorData): void` — logs error at error level

9. Implement `src/errors/error-mapper.ts`:
   - `mapErrorCode(code: string): string` — maps machine codes to human-friendly messages
   - Common mappings: `'schema-validation-failed'` → `'Input validation failed: check required fields'`, `'pandoc-conversion-failed'` → `'Document conversion failed: check markdown syntax'`, `'tex-compilation-failed'` → `'LaTeX compilation failed: check math expressions and environments'`

10. Implement `src/errors/mcp-error-handler.ts` — `MCPErrorHandler` class per spec §3.3:
    - `toMCPError(error: ConversionErrorData): { code: number; message: string }` — maps ConversionErrorData to MCP error format (code 500 for internal, code 400 for validation)
    - `wrap<T>(fn: () => Promise<T>, logger: Logger): Promise<T>` — wraps async function, catches errors, logs, rethrows as MCP-compatible error

11. Implement `src/integration/subprocess-pool.ts` — simple concurrency limiter:
    - `SubprocessPool` class with `maxConcurrent: number` (default 4)
    - `run<T>(fn: () => Promise<T>): Promise<T>` — queues execution, respects concurrency limit

12. Implement `src/integration/pandoc-wrapper.ts` — `PandocWrapper` class per spec §3.8:
    - Constructor: `(logger: Logger)`
    - `convert(content: string, fromFormat: string, toFormat: string, extraArgs?: string[]): Promise<string>` — spawns `pandoc` process with stdin input
    - `convertFile(inputPath: string, toFormat: string, outputPath: string, extraArgs?: string[]): Promise<void>` — converts file to file
    - `version(): Promise<string>` — returns pandoc version string
    - Timeout: 30 seconds. On timeout, kill process and throw ConversionError with code 'pandoc-timeout'.
    - Check pandoc >= 3.0 at construction time; throw if version mismatch.

13. Implement `src/integration/python-engine.ts` — `PythonEngine` class per spec §3.7:
    - Constructor: `(logger: Logger, pythonPath?: string)` — defaults `pythonPath` to `python3`
    - `run(scriptPath: string, args: string[], input?: string): Promise<{ stdout: string; stderr: string; exitCode: number }>` — spawns Python subprocess
    - `runJSON<T>(scriptPath: string, inputData: unknown): Promise<T>` — serializes input to JSON on stdin, parses JSON from stdout
    - `checkAvailable(): Promise<boolean>` — checks Python and required packages (sympy, pypandoc) are importable
    - Timeout: 60 seconds for compilation tasks, 5 seconds for validation tasks (pass timeout as arg)

14. Implement `src/transport/stdio-transport.ts`:
    - Thin wrapper re-exporting `StdioServerTransport` from `@modelcontextprotocol/sdk/server/stdio.js`
    - No additional logic; this exists as an indirection point if transport changes in future

15. Implement `src/factory/module-manifest.ts` — `ModuleManifest` interface and registry per spec §3.2:
    - `ModuleManifest` interface with: `name`, `version`, `description`, `supportedFormats: OutputFormat[]`, `capabilities: string[]`, `pythonDependencies: string[]`, `systemDependencies: string[]`
    - `MCPToolDefinition` interface with: `name`, `description`, `inputSchema`, `outputSchema`

16. Implement `src/factory/tool-builder.ts` — `ToolBuilder` helper class:
    - `tool(name: string, description: string, inputSchema: object, outputSchema: object): MCPToolDefinition` — convenience builder
    - `withDependencies(manifest: ModuleManifest): ToolBuilder` — validates format support before building

17. Implement `src/factory/create-server.ts` — `createZenSciServer()` factory per spec §3.1:
    - Import `McpServer` from `@modelcontextprotocol/sdk/server/mcp.js`
    - Import `StdioServerTransport` from `@modelcontextprotocol/sdk/server/stdio.js`
    - Create all utility instances (Logger, TempFileManager, ArtifactManager, PythonEngine, MCPErrorHandler)
    - Return `ZenSciContext` with all utilities and `start()` function
    - `start()` must: `await server.connect(new StdioServerTransport())`
    - Export `mapOutputFormatToRenderTarget()` standalone function
    - Export `runConversionPipeline()` standalone async function (see spec §3.9 for full implementation)

18. Implement `src/types.ts` — SDK-specific types:
    - `ZenSciServerConfig` interface
    - `ZenSciContext` interface
    - Re-export relevant types from `@zen-sci/core` that modules commonly need

19. Implement `src/index.ts` matching public API surface from `packages-sdk-spec.md §2.2`:
    - Export: `createZenSciServer`, `ModuleManifest`, `MCPToolDefinition`, `ToolBuilder`, `PythonEngine`, `PandocWrapper`, `MCPErrorHandler`, `TempFileManager`, `ArtifactManager`, `Logger`, type exports from `./types.js`

### 2.3 — Write packages/sdk tests

20. Implement `tests/factory.test.ts`:
    - `createZenSciServer()` returns a ZenSciContext with all expected properties
    - Returned `server` is an instance of McpServer
    - Logger is initialized with the module name
    - TempFileManager is initialized

21. Implement `tests/integration.test.ts`:
    - `PandocWrapper.version()` returns a string (integration, requires pandoc installed)
    - `PandocWrapper.convert()` converts a simple markdown string to HTML
    - `PythonEngine.checkAvailable()` returns boolean (should not throw)

22. Implement `tests/error-handling.test.ts`:
    - `MCPErrorHandler.toMCPError()` maps ConversionErrorData correctly to MCP error shape
    - `MCPErrorHandler.wrap()` catches errors and re-throws in expected shape
    - Pandoc timeout produces 'pandoc-timeout' error code

23. Run `pnpm --filter @zen-sci/sdk run build` → must exit 0.
    Run `pnpm --filter @zen-sci/sdk run test` → must exit 0.

### 2.4 — Scaffold servers/latex-mcp

24. Create `zen-sci/servers/latex-mcp/` directory structure:
    ```
    zen-sci/servers/latex-mcp/
    ├── src/
    │   ├── index.ts          (MCP server entry point)
    │   ├── tools/
    │   │   ├── convert-to-pdf.ts
    │   │   ├── validate-document.ts
    │   │   └── check-citations.ts
    │   └── manifest.ts       (ModuleManifest for latex-mcp)
    ├── engine/
    │   ├── latex_engine.py   (Primary Python engine)
    │   ├── math_validator.py (SymPy validation)
    │   ├── citation_engine.py (BibTeX + citeproc)
    │   └── requirements.txt
    ├── tests/
    │   ├── unit/
    │   │   ├── tools.test.ts
    │   │   └── manifest.test.ts
    │   └── integration/
    │       └── end-to-end.test.ts
    ├── fixtures/
    │   ├── sample.md         (Sample markdown with math + citations)
    │   ├── sample.bib        (Sample BibTeX file)
    │   └── expected-warnings.json
    ├── package.json
    ├── tsconfig.json
    └── README.md
    ```

25. Create `zen-sci/servers/latex-mcp/package.json`:
    ```json
    {
      "name": "@zen-sci/latex-mcp",
      "version": "0.1.0",
      "description": "ZenSci LaTeX MCP: markdown to publication-ready PDF",
      "type": "module",
      "main": "./dist/index.js",
      "bin": {
        "zen-sci-latex": "./dist/cli.js"
      },
      "scripts": {
        "build": "tsc --build",
        "start": "node dist/index.js",
        "test": "vitest run",
        "test:integration": "vitest run tests/integration",
        "typecheck": "tsc --noEmit"
      },
      "dependencies": {
        "@modelcontextprotocol/sdk": "^1.5.0",
        "@zen-sci/core": "workspace:*",
        "@zen-sci/sdk": "workspace:*"
      },
      "devDependencies": {
        "@types/node": "^20.0.0",
        "typescript": "^5.3.0",
        "vitest": "^1.2.0"
      }
    }
    ```

26. Create `zen-sci/servers/latex-mcp/engine/requirements.txt`:
    ```
    pypandoc>=1.12
    sympy>=1.12
    bibtexparser>=1.4.0
    pylatexenc>=2.10
    ```

### 2.5 — Implement servers/latex-mcp TypeScript

27. Implement `src/manifest.ts` — defines the `ModuleManifest` for latex-mcp:
    - `name: 'latex-mcp'`, `version: '0.1.0'`
    - `supportedFormats: ['latex', 'paper-ieee', 'paper-acm', 'paper-arxiv', 'thesis', 'patent']`
    - `capabilities: ['math-validation', 'citation-resolution', 'cross-references', 'toc-generation', 'pdf-compilation']`
    - `pythonDependencies: ['pypandoc>=1.12', 'sympy>=1.12', 'bibtexparser>=1.4.0']`
    - `systemDependencies: ['pandoc>=3.0', 'pdflatex OR xelatex OR lualatex']`

28. Implement `src/tools/convert-to-pdf.ts` — handler for the `convert_to_pdf` MCP tool:
    - Accept `ConvertToPdfArgs` (source, title, author[], bibliography?, bibliography_style?, options?: {engine, toc, geometry, font, draft_mode}, latex_preamble?, output_dir?)
    - Build `DocumentRequest` from args (using `@zen-sci/core` types)
    - Call `runConversionPipeline()` from `@zen-sci/sdk` with custom `render` and `compile` steps:
      - `render`: pass to Python engine's `latex_engine.py` via PythonEngine.runJSON()
      - `compile`: Python engine returns TeX source + PDF path; read PDF into Buffer
    - Return tool result: `{ pdf_path, pdf_base64, latex_source, page_count, metadata, warnings, citations }`
    - Match output schema exactly as specified in `latex-mcp-v0.1-spec.md §3.2`

29. Implement `src/tools/validate-document.ts` — handler for `validate_document` MCP tool:
    - Accept `source`, `bibliography?`
    - Run SchemaValidator + MarkdownParser + MathValidator + CitationManager from `@zen-sci/core`
    - Return: `{ valid, errors, warnings, citationStats: { total, resolved, unresolved[] }, mathStats: { total, valid, invalid[] } }`

30. Implement `src/tools/check-citations.ts` — handler for `check_citations` MCP tool:
    - Accept `source`, `bibliography`
    - Run CitationManager from `@zen-sci/core`
    - Return: `{ resolved: CitationRecord[], unresolved: string[], bibliography_entries: number }`

31. Implement `src/index.ts` — the MCP server entry point:
    ```typescript
    import { createZenSciServer } from '@zen-sci/sdk';
    import { latexManifest } from './manifest.js';
    import { convertToPdf } from './tools/convert-to-pdf.js';
    import { validateDocument } from './tools/validate-document.js';
    import { checkCitations } from './tools/check-citations.js';

    const { server, logger, start } = createZenSciServer({
      name: 'latex-mcp',
      version: '0.1.0',
      manifest: latexManifest,
    });

    // Register tools using MCP SDK v2 high-level API
    server.registerTool('convert_to_pdf', { ... inputSchema ... }, convertToPdf);
    server.registerTool('validate_document', { ... }, validateDocument);
    server.registerTool('check_citations', { ... }, checkCitations);

    await start();
    ```
    - Tool input schemas must match exactly what `latex-mcp-v0.1-spec.md §3.2` specifies (snake_case names per MCP convention)
    - Use `@modelcontextprotocol/sdk` v2 `server.registerTool()` API (not deprecated `setRequestHandler` pattern)
    - See spec §3.2 for full inputSchema and outputSchema definitions

### 2.6 — Implement Python engine

32. Implement `engine/latex_engine.py` — primary conversion engine:
    - Entry point: reads JSON from stdin (`{ request_id, source, frontmatter, bibliography, options }`)
    - Step 1: Write source to temp .md file
    - Step 2: Write bibliography to temp .bib file (if provided)
    - Step 3: Call `pypandoc.convert_file(md_path, 'latex', outputfile=tex_path, extra_args=[...])` to get .tex source
    - Step 4: Post-process .tex to inject frontmatter metadata (title, author, date) and custom preamble
    - Step 5: Inject `\usepackage{microtype}`, `\usepackage{hyperref}`, `\usepackage{bookmark}` if not present
    - Step 6: Compile: `subprocess.run(['pdflatex', '-interaction=nonstopmode', tex_path], ...)` — run twice for cross-refs
    - Step 7: Read PDF → base64 encode
    - Step 8: Write JSON to stdout: `{ pdf_base64, latex_source, page_count, warnings, citations: {total, resolved, unresolved} }`
    - On any error: write JSON to stdout with `{ error: { code, message, details } }` and exit(1)
    - Temp files in `/tmp/zen-sci-latex-{request_id}/`; cleanup on exit

33. Implement `engine/math_validator.py`:
    - Reads JSON from stdin: `{ expressions: [{ id, expression, context }] }`
    - For each expression: `sympy.parse_expr(expression)` — if it raises, mark as invalid
    - Writes JSON to stdout: `{ results: [{ id, valid, error? }] }`

34. Implement `engine/citation_engine.py`:
    - Reads JSON from stdin: `{ bibliography_content, citation_keys }`
    - Parses BibTeX with bibtexparser
    - Resolves each key, collects unresolved
    - Writes JSON: `{ resolved: [{key, entry}], unresolved: [key] }`

### 2.7 — Write latex-mcp tests

35. Implement `tests/unit/manifest.test.ts`:
    - latexManifest has all required fields
    - supportedFormats includes 'latex' and 'paper-ieee'
    - capabilities includes 'math-validation' and 'citation-resolution'

36. Implement `tests/unit/tools.test.ts` (mock PythonEngine):
    - `convertToPdf` returns expected shape when PythonEngine.runJSON() returns mocked response
    - Missing required field `source` produces validation error before Python is called
    - Invalid `OutputFormat` produces validation error

37. Implement `tests/integration/end-to-end.test.ts` (requires pandoc + TeX Live):
    - Simple markdown with one heading and one paragraph → valid PDF returned as base64
    - Markdown with `$E = mc^2$` math → PDF produced (math validation passes)
    - Markdown with `[@smith2023]` citation + .bib with 'smith2023' entry → citation resolved in PDF
    - Markdown with unresolved citation `[@missing2023]` → warning in response, PDF still produced
    - Skip tests if `pandoc` or `pdflatex` not available (`beforeAll` check)

38. Run `pnpm --filter @zen-sci/latex-mcp run build` → must exit 0.
    Run `pnpm --filter @zen-sci/latex-mcp run test` → must exit 0. Integration tests may skip if TeX not available.

---

## 3. File Manifest

### Create (new files):
```
zen-sci/packages/sdk/package.json
zen-sci/packages/sdk/tsconfig.json
zen-sci/packages/sdk/README.md
zen-sci/packages/sdk/src/index.ts
zen-sci/packages/sdk/src/types.ts
zen-sci/packages/sdk/src/factory/create-server.ts
zen-sci/packages/sdk/src/factory/module-manifest.ts
zen-sci/packages/sdk/src/factory/tool-builder.ts
zen-sci/packages/sdk/src/integration/python-engine.ts
zen-sci/packages/sdk/src/integration/pandoc-wrapper.ts
zen-sci/packages/sdk/src/integration/subprocess-pool.ts
zen-sci/packages/sdk/src/transport/stdio-transport.ts
zen-sci/packages/sdk/src/errors/mcp-error-handler.ts
zen-sci/packages/sdk/src/errors/error-mapper.ts
zen-sci/packages/sdk/src/logging/logger.ts
zen-sci/packages/sdk/src/logging/pipeline-monitor.ts
zen-sci/packages/sdk/src/utils/temp-file-manager.ts
zen-sci/packages/sdk/src/utils/artifact-manager.ts
zen-sci/packages/sdk/src/utils/request-id-generator.ts
zen-sci/packages/sdk/tests/factory.test.ts
zen-sci/packages/sdk/tests/integration.test.ts
zen-sci/packages/sdk/tests/error-handling.test.ts
zen-sci/servers/latex-mcp/package.json
zen-sci/servers/latex-mcp/tsconfig.json
zen-sci/servers/latex-mcp/README.md
zen-sci/servers/latex-mcp/src/index.ts
zen-sci/servers/latex-mcp/src/manifest.ts
zen-sci/servers/latex-mcp/src/tools/convert-to-pdf.ts
zen-sci/servers/latex-mcp/src/tools/validate-document.ts
zen-sci/servers/latex-mcp/src/tools/check-citations.ts
zen-sci/servers/latex-mcp/engine/latex_engine.py
zen-sci/servers/latex-mcp/engine/math_validator.py
zen-sci/servers/latex-mcp/engine/citation_engine.py
zen-sci/servers/latex-mcp/engine/requirements.txt
zen-sci/servers/latex-mcp/tests/unit/tools.test.ts
zen-sci/servers/latex-mcp/tests/unit/manifest.test.ts
zen-sci/servers/latex-mcp/tests/integration/end-to-end.test.ts
zen-sci/servers/latex-mcp/fixtures/sample.md
zen-sci/servers/latex-mcp/fixtures/sample.bib
zen-sci/servers/latex-mcp/fixtures/expected-warnings.json
```

---

## 4. Success Criteria

- [ ] `pnpm --filter @zen-sci/sdk run build` exits 0
- [ ] `pnpm --filter @zen-sci/sdk run test` exits 0 (all sdk tests pass)
- [ ] `pnpm --filter @zen-sci/latex-mcp run build` exits 0
- [ ] `pnpm --filter @zen-sci/latex-mcp run test` exits 0 (unit tests pass; integration may skip if TeX unavailable)
- [ ] `createZenSciServer()` returns a `ZenSciContext` with all required properties (server, logger, tempFileManager, artifactManager, pythonEngine, errorHandler, start)
- [ ] No `ZenSciServer` abstract base class exists anywhere in packages/sdk
- [ ] `server.registerTool()` (MCP SDK v2 API) used — NOT deprecated `setRequestHandler` pattern
- [ ] Tool names use snake_case: `convert_to_pdf`, `validate_document`, `check_citations`
- [ ] `convert_to_pdf` input schema matches `latex-mcp-v0.1-spec.md §3.2` exactly (all fields, types, enums)
- [ ] Integration test: simple markdown → PDF (base64 returned, not null/empty)
- [ ] Integration test: math expression `$E = mc^2$` → PDF produced without error
- [ ] Integration test: resolved citation → no unresolved warnings in response
- [ ] Integration test: unresolved citation `[@missing]` → warning in response, PDF still produced (not error)
- [ ] Processing time < 5 seconds for the sample.md fixture (on a standard developer machine)
- [ ] `STATUS.md` updated: packages/sdk ✅, latex-mcp v0.1 ✅

---

## 5. Constraints & Non-Goals

- **DO NOT** use `ZenSciServer` abstract base class — composition via `createZenSciServer()` factory only (Decision D1)
- **DO NOT** use `setRequestHandler` + `ListToolsRequestSchema`/`CallToolRequestSchema` pattern — use MCP SDK v2 `server.registerTool()` high-level API
- **DO NOT** implement blog-mcp, slides-mcp, newsletter-mcp, or grant-mcp — those are Phase 2+
- **DO NOT** implement GUI, watch mode, or real-time preview — out of scope for v0.1 (spec non-goals)
- **DO NOT** auto-install LaTeX packages — users manage their TeX installation
- **DO NOT** embed pandoc binary — call system pandoc as subprocess
- **DO NOT** return 'pdf_base64' as empty string on error — errors must propagate as ConversionErrorData
- **DO NOT** swallow Python subprocess errors — surface them with proper error codes
- **DO NOT** hardcode temp file paths — always use TempFileManager
- **DO NOT** modify files in `ZenithScience/specs/` or `ZenithScience/CONTEXT.md`
- `latex-mcp` tool naming: use **snake_case** for all MCP tool names (spec §3.2 convention)

---

## 6. Architecture Grounding

**MCP SDK version:** Use `@modelcontextprotocol/sdk` v1.5.x. The correct import paths are:
- `@modelcontextprotocol/sdk/server/mcp.js` → `{ McpServer }`
- `@modelcontextprotocol/sdk/server/stdio.js` → `{ StdioServerTransport }`
- `@modelcontextprotocol/sdk` does NOT have `@anthropic-ai/sdk` as a dependency

**Python ↔ TypeScript bridge pattern:**
- TypeScript calls `PythonEngine.runJSON(scriptPath, inputData)` — serializes to JSON on stdin
- Python reads from stdin, processes, writes JSON to stdout
- TypeScript reads from stdout, parses JSON
- All errors are captured from stderr and returned as `ConversionErrorData`
- Never use file-based IPC (use stdin/stdout JSON)

**PDF output strategy:** The Python engine returns PDF as base64-encoded string in JSON stdout. TypeScript converts base64 to Buffer. This avoids file path coordination between processes.

**pandoc version pinning:** Check `pandoc --version` output at PandocWrapper construction. Require >= 3.0. If lower version detected, throw `ConversionError` with code 'pandoc-version-mismatch' and message explaining the requirement.

---

*Spec version: packages-sdk-spec.md v1.0.0, latex-mcp-v0.1-spec.md*
*Phase: 1 of 3*
*Prereq: Phase 0 complete (packages/core tests passing)*
*Next: Phase 2 — blog-mcp v0.2 + grant-mcp v0.4*
