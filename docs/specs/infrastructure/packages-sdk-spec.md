# packages/sdk Spec: ZenSci Module Server Base Class

**Author:** ZenSci Architecture (TresPiesDesign.com)
**Status:** Production Ready
**Created:** 2026-02-18
**Version:** 1.0.0
**Audience:** All module server developers; implemented by architecture team

---

## 1. Overview

**packages/sdk** provides shared utility classes and a factory function that *all* ZenSci module servers compose. It handles:

- **MCP protocol integration** — `createZenSciServer()` factory wraps `McpServer` from `@modelcontextprotocol/sdk` with ZenSci defaults
- **Error handling** — converting ConversionErrorData to MCP error responses
- **Logging & observability** — structured logging, pipeline monitoring
- **Common utilities** — temporary file management, subprocess orchestration, Python engine calls

**Key principle:** packages/sdk is transport-layer (MCP-aware) while packages/core is pure data transformation. Modules instantiate `McpServer` directly (via factory or manually) and compose shared utilities as standalone imports. **Composition over inheritance** — no abstract base class.

---

## 2. Architecture Overview

### 2.1 Directory Structure

```
packages/sdk/
├── src/
│   ├── index.ts                          # Public API exports
│   ├── factory/
│   │   ├── create-server.ts             # createZenSciServer() factory function
│   │   ├── module-manifest.ts           # ModuleManifest interface & registry
│   │   └── tool-builder.ts              # Helper to build MCP tools
│   ├── integration/
│   │   ├── python-engine.ts             # Python subprocess orchestration
│   │   ├── pandoc-wrapper.ts            # Pandoc execution abstraction
│   │   └── subprocess-pool.ts           # Thread pool for async subprocess calls
│   ├── transport/
│   │   └── stdio-transport.ts           # Wrapper around MCP's StdioServerTransport
│   ├── errors/
│   │   ├── mcp-error-handler.ts         # Convert ConversionError → MCP error
│   │   └── error-mapper.ts              # Error code → user-friendly messages
│   ├── logging/
│   │   ├── logger.ts                    # Structured logging (pino or similar)
│   │   └── pipeline-monitor.ts          # Track and log pipeline execution
│   ├── utils/
│   │   ├── temp-file-manager.ts         # Create & cleanup temp files
│   │   ├── artifact-manager.ts          # Manage secondary output files
│   │   └── request-id-generator.ts      # UUIDs or custom IDs
│   └── types.ts                         # SDK-specific types (ModuleManifest, MCPToolDef, etc.)
├── tests/
│   ├── base.test.ts
│   ├── integration.test.ts
│   └── error-handling.test.ts
├── package.json
├── tsconfig.json
└── README.md
```

### 2.2 Public API Surface

```typescript
// packages/sdk/src/index.ts
export { createZenSciServer } from './factory/create-server.js';
export type { ModuleManifest, MCPToolDefinition } from './factory/module-manifest.js';
export { ToolBuilder } from './factory/tool-builder.js';
export { PythonEngine } from './integration/python-engine.js';
export { PandocWrapper } from './integration/pandoc-wrapper.js';
export { MCPErrorHandler } from './errors/mcp-error-handler.js';
export { TempFileManager } from './utils/temp-file-manager.js';
export { ArtifactManager } from './utils/artifact-manager.js';
export { Logger } from './logging/logger.js';
export type * from './types.js';
```

---

## 3. Core Classes & Interfaces

### 3.1 createZenSciServer() Factory & ZenSciContext

```typescript
/**
 * ZenSciServerConfig: Configuration for createZenSciServer().
 * Modules provide this to get a pre-configured McpServer with shared utilities.
 */
export interface ZenSciServerConfig {
  /** Module name (e.g., 'paper-mcp', 'beamer-mcp') */
  name: string;

  /** Module version (semver) */
  version: string;

  /** Module manifest (capabilities, supported formats, etc.) */
  manifest: ModuleManifest;
}

/**
 * ZenSciContext: Shared utilities returned by createZenSciServer().
 * Modules compose these as needed — no inheritance required.
 */
export interface ZenSciContext {
  /** Pre-configured McpServer instance (MCP SDK v2) */
  server: McpServer;

  /** Error handler for MCP errors */
  errorHandler: MCPErrorHandler;

  /** Structured logger */
  logger: Logger;

  /** Temp file manager (auto-cleanup on process exit) */
  tempFileManager: TempFileManager;

  /** Artifact manager for secondary outputs */
  artifactManager: ArtifactManager;

  /** Python engine for SymPy, pandoc, etc. */
  pythonEngine: PythonEngine;

  /** Start the server (connect stdio transport) */
  start: () => Promise<void>;
}

/**
 * createZenSciServer: Factory function that creates a McpServer
 * with pre-configured ZenSci utilities.
 *
 * This replaces the former ZenSciServer abstract base class.
 * Modules call this once, then register their tools on the returned server.
 *
 * @param config Module configuration
 * @returns ZenSciContext with server + shared utilities
 *
 * @example
 * ```typescript
 * const { server, logger, pythonEngine, start } = createZenSciServer({
 *   name: 'paper-mcp',
 *   version: '2.0.0',
 *   manifest: paperManifest,
 * });
 *
 * server.registerTool('convert_paper', { ... }, async (args) => { ... });
 * await start();
 * ```
 */
export function createZenSciServer(config: ZenSciServerConfig): ZenSciContext {
  const { name, version } = config;

  // Initialize shared utilities
  const logger = new Logger(name);
  const errorHandler = new MCPErrorHandler();
  const tempFileManager = new TempFileManager(name);
  const artifactManager = new ArtifactManager();
  const pythonEngine = new PythonEngine(logger);

  // Create MCP Server (SDK v2 high-level API)
  const server = new McpServer({ name, version });

  // Start function: connect stdio transport
  const start = async () => {
    try {
      const transport = new StdioServerTransport();
      await server.connect(transport);
      logger.info(`${name} v${version} started and ready for requests`);
    } catch (error) {
      logger.error('Failed to start server', { error });
      process.exit(1);
    }
  };

  return {
    server,
    errorHandler,
    logger,
    tempFileManager,
    artifactManager,
    pythonEngine,
    start,
  };
}

/**
 * Standalone utility: Map OutputFormat to Pandoc render target.
 * Modules can use this directly or provide their own mapping.
 */
export function mapOutputFormatToRenderTarget(format: OutputFormat): string {
  const mapping: Record<OutputFormat, string> = {
    latex: 'latex',
    html: 'html5',
    email: 'html5',
    beamer: 'beamer',
    revealjs: 'revealjs',
    pptx: 'pptx',
    docs: 'docx',
    docx: 'docx',
    epub: 'epub3',
    mobi: 'mobi',
    'lab-notebook': 'html5',
    'policy-brief': 'latex',
    proposal: 'docx',
    'podcast-notes': 'html5',
    resume: 'latex',
    'grant-latex': 'latex',
    'paper-ieee': 'latex',
    'paper-acm': 'latex',
    'paper-arxiv': 'latex',
    thesis: 'latex',
    patent: 'latex',
    whitepaper: 'latex',
  };
  return mapping[format] || 'latex';
}

/**
 * Standalone utility: Run the standard ZenSci conversion pipeline.
 * Modules call this for the default validate → parse → transform → render → compile → output flow.
 * For custom pipelines, modules can use individual @zen-sci/core classes directly.
 */
export async function runConversionPipeline(
  request: DocumentRequest,
  ctx: ZenSciContext,
  options?: {
    render?: (tree: DocumentTree, request: DocumentRequest) => Promise<string | Buffer>;
    compile?: (rendered: string | Buffer, request: DocumentRequest) => Promise<string | Buffer>;
    supportedFormats: OutputFormat[];
  }
): Promise<DocumentResponse> {
  const { logger, errorHandler, tempFileManager, artifactManager } = ctx;
  const pipeline = new ConversionPipeline(request.id);
  pipeline.start();

  try {
    // Validate
    pipeline.startStage('validate');
    const { SchemaValidator, MarkdownParser, CitationManager } = await import('@zen-sci/core');
    const validationResult = SchemaValidator.validate(request, options?.supportedFormats || []);
    if (!validationResult.valid) {
      throw new Error(`Schema validation failed: ${validationResult.errors[0]?.message}`);
    }
    pipeline.completeStage('validate');

    // Parse
    pipeline.startStage('parse');
    const parser = new MarkdownParser();
    const tree = parser.parse(request.source);
    pipeline.completeStage('parse');

    // Transform (citations)
    if (request.bibliography) {
      pipeline.startStage('transform');
      const citationManager = new CitationManager(request.bibliography);
      const orphanedKeys = citationManager.extractKeysFromAST(tree);
      if (orphanedKeys.some((k) => !citationManager.resolve(k))) {
        logger.warn(`Orphaned citations detected: ${orphanedKeys.join(', ')}`);
      }
      pipeline.completeStage('transform');
    }

    // Render (custom or default via Pandoc)
    pipeline.startStage('render');
    const renderResult = options?.render
      ? await options.render(tree, request)
      : await new PandocWrapper(logger).convert(
          request.source, 'markdown', mapOutputFormatToRenderTarget(request.format)
        );
    pipeline.completeStage('render');

    // Compile (custom or passthrough)
    pipeline.startStage('compile');
    const compileResult = options?.compile
      ? await options.compile(renderResult, request)
      : renderResult;
    pipeline.completeStage('compile');

    // Output
    pipeline.startStage('output');
    const artifacts = artifactManager.getArtifacts();
    const response: DocumentResponse = {
      id: request.id,
      requestId: request.id,
      format: request.format,
      content: compileResult,
      artifacts,
      warnings: [],
      metadata: {
        title: request.frontmatter.title,
        author: request.frontmatter.author as string[] | undefined,
        generatedAt: new Date().toISOString(),
        citationCount: 0,
        sources: [request.frontmatter.title || 'markdown-source'],
      },
      elapsed: pipeline.elapsed(),
    };
    pipeline.completeStage('output');

    pipeline.complete(true);
    logger.info(`Document conversion successful (${pipeline.elapsed()}ms)`, {
      requestId: request.id,
      format: request.format,
    });

    return response;
  } catch (error) {
    const conversionError = errorHandler.handleError(error);
    pipeline.complete(false, undefined, conversionError);
    logger.error('Document conversion failed', {
      requestId: request.id,
      error: conversionError,
    });
    throw error;
  } finally {
    await tempFileManager.cleanup();
  }
}
```

### 3.2 ModuleManifest Interface

```typescript
/**
 * ModuleManifest: Metadata describing a ZenSci module.
 * Returned by getModuleManifest().
 */
export interface ModuleManifest {
  /** Module identifier (kebab-case, e.g., 'paper-mcp') */
  id: string;

  /** Human-readable name (e.g., 'Research Paper Generator') */
  name: string;

  /** Module version (semver) */
  version: string;

  /** Module description (1-2 sentences) */
  description: string;

  /** Supported output formats (subset of OutputFormat union) */
  outputFormats: OutputFormat[];

  /** Supported input formats (usually just 'markdown') */
  inputFormats: string[];

  /** Supported features (e.g., 'citations', 'math', 'toc') */
  features: string[];

  /** Required capabilities from packages/core */
  coreCapabilities: ('parser' | 'citations' | 'validation' | 'math')[];

  /** Documentation URL (link to README or wiki) */
  documentationUrl?: string;

  /** Author/maintainer */
  author: string;

  /** License (e.g., 'Apache-2.0', 'MIT') */
  license: string;

  /** Phase (1, 2, 3) */
  phase: 1 | 2 | 3;

  /** Shipping status */
  status: 'beta' | 'stable' | 'deprecated';

  /** Custom options schema (Zod or JSON Schema) for moduleOptions */
  optionsSchema?: Record<string, unknown>;

  /** Example input (markdown snippet) */
  exampleInput?: string;

  /** Example output (shortened) */
  exampleOutput?: string;

  /** Release notes or changelog */
  releaseNotes?: string;
}
```

### 3.3 MCPToolDefinition Interface

```typescript
/**
 * MCPToolDefinition: Definition of a single MCP tool.
 * Extends MCP ToolDefinition with input/output schema.
 */
export interface MCPToolDefinition {
  /** Tool name (e.g., 'convertPaper') */
  name: string;

  /** Short description */
  description?: string;

  /** Input JSON Schema (required parameters) */
  inputSchema: {
    type: 'object';
    properties: Record<string, { type: string; description?: string; enum?: unknown[] }>;
    required: string[];
    additionalProperties?: boolean;
  };

  /** Expected output type ('text', 'buffer', 'json') */
  outputType?: 'text' | 'buffer' | 'json';

  /** Output JSON Schema (if applicable) */
  outputSchema?: Record<string, unknown>;
}
```

---

## 4. Python Engine Integration

### 4.1 PythonEngine Class

```typescript
/**
 * PythonEngine: Orchestrate Python subprocess calls.
 * Manages SymPy validation, pandoc conversion, and custom Python jobs.
 *
 * Invokes: python3 -m zen_sci.engine
 * Input: JSON-serialized PandocJob
 * Output: JSON-serialized PandocResult
 */
export class PythonEngine {
  constructor(private logger: Logger) {}

  /**
   * Execute a job via Python engine.
   * @param job PandocJob or custom job object
   * @returns Result from Python subprocess
   */
  async executeJob(job: PandocJob): Promise<PandocResult> {
    return new Promise((resolve, reject) => {
      const { spawn } = require('child_process');

      const pythonProcess = spawn('python3', ['-m', 'zen_sci.engine'], {
        env: { ...process.env },
        timeout: 30000, // 30 second timeout
      });

      let stdout = '';
      let stderr = '';

      pythonProcess.stdin.write(JSON.stringify(job));
      pythonProcess.stdin.end();

      pythonProcess.stdout.on('data', (data: Buffer) => {
        stdout += data.toString();
      });

      pythonProcess.stderr.on('data', (data: Buffer) => {
        stderr += data.toString();
      });

      pythonProcess.on('close', (code: number) => {
        if (code !== 0) {
          this.logger.error('Python engine failed', {
            code,
            stderr: stderr.slice(0, 500),
          });
          reject(new Error(`Python engine failed (exit code ${code}): ${stderr}`));
        } else {
          try {
            const result = JSON.parse(stdout) as PandocResult;
            resolve(result);
          } catch (parseError) {
            reject(new Error(`Failed to parse Python output: ${parseError}`));
          }
        }
      });

      pythonProcess.on('error', (error: Error) => {
        reject(new Error(`Python process error: ${error.message}`));
      });
    });
  }

  /**
   * Validate LaTeX math expression.
   * @param latex LaTeX expression
   * @returns true if valid, false otherwise
   */
  async validateMath(latex: string): Promise<boolean> {
    const job: PandocJob = {
      type: 'validate-math',
      payload: { latex },
    };
    try {
      const result = await this.executeJob(job);
      return (result.success && result.payload?.valid) || false;
    } catch {
      return false;
    }
  }

  /**
   * Convert markdown to target format via Pandoc.
   * @param source Markdown source
   * @param fromFormat Input format (e.g., 'markdown')
   * @param toFormat Output format (e.g., 'latex', 'html5')
   * @returns Converted content
   */
  async convertViaPandoc(
    source: string,
    fromFormat: string,
    toFormat: string
  ): Promise<string> {
    const job: PandocJob = {
      type: 'pandoc-convert',
      payload: {
        source,
        fromFormat,
        toFormat,
      },
    };
    const result = await this.executeJob(job);
    if (!result.success) {
      throw new Error(`Pandoc conversion failed: ${result.error}`);
    }
    return result.payload?.output || '';
  }
}

/**
 * PandocJob: Input structure for Python engine.
 */
export interface PandocJob {
  type: 'pandoc-convert' | 'validate-math' | 'compile-tex' | string;
  payload: Record<string, unknown>;
}

/**
 * PandocResult: Output structure from Python engine.
 */
export interface PandocResult {
  success: boolean;
  payload?: Record<string, unknown>;
  error?: string;
}
```

### 4.2 PandocWrapper Class

```typescript
/**
 * PandocWrapper: Convenience wrapper around PythonEngine for Pandoc.
 * Encapsulates Pandoc-specific configuration and error handling.
 */
export class PandocWrapper {
  constructor(private logger: Logger, private pythonEngine?: PythonEngine) {
    if (!pythonEngine) {
      this.pythonEngine = new PythonEngine(logger);
    }
  }

  /**
   * Convert markdown to target format.
   * @param source Markdown source
   * @param fromFormat (usually 'markdown')
   * @param toFormat (e.g., 'latex', 'html5', 'docx')
   * @param options Pandoc options (toc, template, etc.)
   * @returns Converted content (string or Buffer)
   */
  async convert(
    source: string,
    fromFormat: string,
    toFormat: string,
    options?: PandocOptions
  ): Promise<string | Buffer> {
    try {
      return await this.pythonEngine!.convertViaPandoc(source, fromFormat, toFormat);
    } catch (error) {
      this.logger.error('Pandoc conversion failed', { toFormat, error });
      throw error;
    }
  }

  /**
   * Convert to LaTeX with custom preamble.
   */
  async convertToLaTeX(
    source: string,
    options?: {
      template?: string;
      includes?: { inHeader?: string };
      variables?: Record<string, string>;
    }
  ): Promise<string> {
    return this.convert(source, 'markdown', 'latex');
  }

  /**
   * Convert to PDF via LaTeX compilation.
   * (Note: actual compilation happens in Python engine)
   */
  async convertToPDF(source: string): Promise<Buffer> {
    const latex = await this.convertToLaTeX(source);
    const result = await this.pythonEngine!.executeJob({
      type: 'compile-tex',
      payload: { latex },
    });
    if (!result.success || !result.payload?.pdf) {
      throw new Error('PDF compilation failed');
    }
    return Buffer.from(result.payload.pdf, 'base64');
  }
}

export interface PandocOptions {
  template?: string;
  includes?: { inHeader?: string };
  variables?: Record<string, string>;
  toc?: boolean;
  [key: string]: unknown;
}
```

---

## 5. Error Handling

### 5.1 MCPErrorHandler Class

```typescript
/**
 * MCPErrorHandler: Convert errors to ConversionErrorData and MCP error responses.
 * Provides user-friendly error messages and structured error codes.
 */
export class MCPErrorHandler {
  /**
   * Convert thrown error to ConversionErrorData (wire-safe shape).
   * @param error JavaScript Error or ConversionError class instance
   * @returns ConversionErrorData
   */
  handleError(error: unknown): ConversionErrorData {
    if (error instanceof Error) {
      return {
        code: error.name || 'UNKNOWN_ERROR',
        message: error.message,
        stack: error.stack,
      };
    }
    return {
      code: 'UNKNOWN_ERROR',
      message: String(error),
    };
  }

  /**
   * Convert error to MCP ErrorResponse.
   * @param error ConversionError class, ConversionErrorData, or raw Error
   * @returns MCP-compatible error response
   */
  toMCPError(error: unknown): {
    content: [{ type: 'text'; text: string }];
    isError: true;
  } {
    const conversionError = error instanceof ConversionError ? error : this.handleError(error);
    const errorMessage = `[${conversionError.code}] ${conversionError.message}`;
    return {
      content: [{ type: 'text', text: errorMessage }],
      isError: true,
    };
  }

  /**
   * Map error codes to user-friendly messages.
   */
  private errorMessageMap: Record<string, string> = {
    SCHEMA_VALIDATION_FAILED: 'Request does not match expected format',
    PARSE_ERROR: 'Failed to parse markdown source',
    CITATION_ERROR: 'Citation or bibliography processing failed',
    MATH_ERROR: 'Mathematical expression validation failed',
    LINK_ERROR: 'URL validation failed',
    CONVERSION_ERROR: 'Document conversion failed',
    UNKNOWN_ERROR: 'An unknown error occurred',
  };

  getMessage(code: string): string {
    return this.errorMessageMap[code] || code;
  }
}
```

---

## 6. Logging & Observability

### 6.1 Logger Class

```typescript
/**
 * Logger: Structured logging with log levels.
 * Uses pino or bunyan for production-grade logging.
 */
export class Logger {
  private logger: any; // Actual logger from pino/bunyan

  constructor(moduleName: string) {
    // Simplified: in production use pino or bunyan
    this.logger = {
      level: process.env.LOG_LEVEL || 'info',
      module: moduleName,
    };
  }

  info(message: string, data?: Record<string, unknown>): void {
    console.log(
      JSON.stringify({
        level: 'info',
        message,
        ...data,
        timestamp: new Date().toISOString(),
      })
    );
  }

  warn(message: string, data?: Record<string, unknown>): void {
    console.warn(
      JSON.stringify({
        level: 'warn',
        message,
        ...data,
        timestamp: new Date().toISOString(),
      })
    );
  }

  error(message: string, data?: Record<string, unknown>): void {
    console.error(
      JSON.stringify({
        level: 'error',
        message,
        ...data,
        timestamp: new Date().toISOString(),
      })
    );
  }

  debug(message: string, data?: Record<string, unknown>): void {
    if (this.logger.level === 'debug') {
      console.log(
        JSON.stringify({
          level: 'debug',
          message,
          ...data,
          timestamp: new Date().toISOString(),
        })
      );
    }
  }
}
```

---

## 7. Utility Classes

### 7.1 TempFileManager

```typescript
/**
 * TempFileManager: Create and manage temporary files during conversion.
 * Auto-cleanup on process exit or manual cleanup call.
 */
export class TempFileManager {
  private tempDir: string;
  private createdFiles: Set<string> = new Set();

  constructor(moduleName: string) {
    const tmpDir = require('os').tmpdir();
    this.tempDir = `${tmpDir}/zen-sci-${moduleName}-${Date.now()}`;
    require('fs').mkdirSync(this.tempDir, { recursive: true });

    // Auto-cleanup on process exit
    process.on('exit', () => this.cleanup());
  }

  /**
   * Create a temporary file.
   * @param filename Filename or extension (e.g., 'output.pdf')
   * @returns Full path to temp file
   */
  createFile(filename: string): string {
    const path = `${this.tempDir}/${filename}`;
    this.createdFiles.add(path);
    return path;
  }

  /**
   * Create a temporary directory.
   */
  createDir(dirname: string): string {
    const path = `${this.tempDir}/${dirname}`;
    require('fs').mkdirSync(path, { recursive: true });
    this.createdFiles.add(path);
    return path;
  }

  /**
   * Clean up all temporary files.
   */
  async cleanup(): Promise<void> {
    const fs = require('fs').promises;
    for (const file of this.createdFiles) {
      try {
        await fs.rm(file, { recursive: true, force: true });
      } catch (error) {
        // Ignore cleanup errors
      }
    }
    try {
      await fs.rm(this.tempDir, { recursive: true, force: true });
    } catch (error) {
      // Ignore cleanup errors
    }
  }
}
```

### 7.2 ArtifactManager

```typescript
/**
 * ArtifactManager: Track and manage secondary output files (images, CSS, etc.).
 */
export class ArtifactManager {
  private artifacts: Artifact[] = [];

  /**
   * Register an artifact.
   */
  registerArtifact(artifact: Artifact): void {
    this.artifacts.push(artifact);
  }

  /**
   * Get all registered artifacts.
   */
  getArtifacts(): Artifact[] {
    return [...this.artifacts];
  }

  /**
   * Clear artifacts.
   */
  clear(): void {
    this.artifacts = [];
  }
}
```

---

## 8. Testing Strategy

### 8.1 Unit Tests

```typescript
// packages/sdk/tests/factory.test.ts
import { describe, it, expect } from 'vitest';
import { createZenSciServer, runConversionPipeline } from '../src';
import type { ModuleManifest, DocumentRequest } from '@zen-sci/core';
import { z } from 'zod';

const testManifest: ModuleManifest = {
  id: 'test-mcp',
  name: 'Test Module',
  version: '1.0.0',
  description: 'Test module',
  outputFormats: ['html', 'latex'],
  inputFormats: ['markdown'],
  features: ['parsing'],
  coreCapabilities: ['parser'],
  author: 'test',
  license: 'MIT',
  phase: 1,
  status: 'beta',
};

describe('createZenSciServer', () => {
  it('creates server with shared utilities', () => {
    const ctx = createZenSciServer({
      name: 'test-mcp',
      version: '1.0.0',
      manifest: testManifest,
    });
    expect(ctx.server).toBeDefined();
    expect(ctx.logger).toBeDefined();
    expect(ctx.pythonEngine).toBeDefined();
    expect(ctx.start).toBeTypeOf('function');
  });

  it('registers tools via McpServer.registerTool', () => {
    const ctx = createZenSciServer({
      name: 'test-mcp',
      version: '1.0.0',
      manifest: testManifest,
    });
    // Should not throw
    ctx.server.registerTool(
      'test_tool',
      { description: 'Test', inputSchema: z.object({ input: z.string() }) },
      async ({ input }) => ({ content: [{ type: 'text', text: input }] })
    );
  });
});
```

### 8.2 Integration Tests

Test full pipeline: request → core functions → response.

---

## 9. Dependencies

### 9.1 npm Dependencies

```json
{
  "dependencies": {
    "@modelcontextprotocol/sdk": "^2.0.0",
    "@zen-sci/core": "^1.0.0",
    "zod": "^3.22.0",
    "pino": "^8.14.0"
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "vitest": "^1.0.0"
  }
}
```

### 9.2 System Dependencies

- **Python 3.11+** — for zen_sci.engine module (SymPy, pandoc, etc.)
- **Pandoc >= 3.0** — for document conversion
- **pdflatex or xelatex** — for TeX → PDF compilation

---

## 10. Module Implementation Example (Composition Pattern)

```typescript
// servers/paper-mcp/src/index.ts
import { createZenSciServer, runConversionPipeline } from '@zen-sci/sdk';
import type { ModuleManifest, DocumentRequest } from '@zen-sci/core';
import { z } from 'zod';

// 1. Define manifest
const manifest: ModuleManifest = {
  id: 'paper-mcp',
  name: 'Research Paper Generator',
  version: '2.0.0',
  description: 'Convert markdown to publication-ready research papers',
  outputFormats: ['paper-ieee', 'paper-acm', 'paper-arxiv'],
  inputFormats: ['markdown'],
  features: ['citations', 'math', 'toc'],
  coreCapabilities: ['parser', 'citations', 'validation'],
  author: 'ZenSci',
  license: 'Apache-2.0',
  phase: 1,
  status: 'stable',
};

// 2. Create server + shared utilities via factory
const { server, logger, pythonEngine, start } = createZenSciServer({
  name: 'paper-mcp',
  version: '2.0.0',
  manifest,
});

// 3. Register module-specific tools (MCP SDK v2 pattern: registerTool + Zod)
server.registerTool(
  'convert_paper',
  {
    description: 'Convert markdown to publication-ready research paper',
    inputSchema: z.object({
      source: z.string().describe('Markdown source content'),
      style: z.enum(['ieee', 'acm', 'arxiv', 'apa']).describe('Paper format style'),
      bibliography: z.string().optional().describe('BibTeX content'),
    }),
  },
  async ({ source, style, bibliography }) => {
    const request: DocumentRequest = {
      id: crypto.randomUUID(),
      source,
      format: `paper-${style}` as any,
      frontmatter: {},
      options: { moduleOptions: { paperStyle: style } },
      bibliography,
    };
    const response = await runConversionPipeline(request, { server, logger, pythonEngine } as any, {
      supportedFormats: manifest.outputFormats,
    });
    return { content: [{ type: 'text', text: JSON.stringify(response) }] };
  }
);

server.registerTool(
  'get_module_info',
  {
    description: 'Get paper-mcp module metadata',
    inputSchema: z.object({}),
  },
  async () => {
    return { content: [{ type: 'text', text: JSON.stringify(manifest) }] };
  }
);

// 4. Start server
start();
```

---

## 11. Versioning & Compatibility

**packages/sdk** follows Semantic Versioning:

- **MAJOR (2.0.0):** Breaking changes to `createZenSciServer()` factory API, `ZenSciContext` shape, or shared utility signatures
- **MINOR (1.1.0):** New utilities, new optional methods
- **PATCH (1.0.1):** Bug fixes

All modules must specify compatible version ranges:

```json
{
  "dependencies": {
    "@zen-sci/core": "^1.0.0",
    "@zen-sci/sdk": "^2.0.0"
  }
}
```

---

## 12. Future Enhancements

1. **Plugin system:** Allow custom validation/rendering hooks
2. **Caching layer:** Cache parsed documents and rendered outputs
3. **Metrics collection:** Built-in Prometheus/OpenTelemetry integration
4. **Streaming API:** Support large document streams
5. **Multi-process pool:** Thread pool for parallel conversions
6. **Database backing:** Optional SQLite for document history/caching

---

## Audit Notes (2026-02-18)

**Auditor:** Claude (Opus-level audit)
**Status:** Reviewed — fixes applied

### Issues Found (Initial Audit)
1. **CRITICAL-2: `ZenSciServer` base class orphaned** — No module spec used it.
2. **CRITICAL-3: MCP SDK v2 pattern** — Specs used low-level `Server` class instead of `McpServer`.
3. **ADVISORY-3: Python `pandoc` import** — Should be `pypandoc`.

### Fixed in Initial Audit
- **MEDIUM-1:** Typo `convertViaP andoc` → `convertViaPandoc` (lines 750, 818).
- **HIGH-3:** Python version updated from 3.8+ to 3.11+.
- **HIGH-4/MEDIUM-5:** Pandoc version updated from 2.19+ to >= 3.0.

### Decision Propagation (Cruz, 2026-02-18)

**CRITICAL-2 resolved:** ZenSciServer abstract base class replaced with `createZenSciServer()` factory function + `ZenSciContext` interface. Composition over inheritance. Modules instantiate `McpServer` directly via factory, compose shared utilities (PythonEngine, PandocWrapper, MCPErrorHandler, Logger, TempFileManager, ArtifactManager) as standalone imports. `runConversionPipeline()` standalone function replaces the old `handleConvertDocument()` method.

**CRITICAL-3 resolved:** SDK now uses `McpServer` from `@modelcontextprotocol/sdk` v2 with `registerTool()` + Zod schemas. Dependency updated from `^0.1.0` to `^2.0.0`. `zod` added as direct dependency.

**HIGH-5 resolved (cross-ref from core):** `MCPErrorHandler` updated to reference `ConversionErrorData` (wire interface) and `ConversionError` (runtime class) per Cluster 3 decision.

**Subclass example rewritten:** `PaperServer` class replaced with flat composition pattern showing `createZenSciServer()` → `server.registerTool()` → `start()` flow.
