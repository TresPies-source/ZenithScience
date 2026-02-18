# packages/core Spec: Shared Foundation Library

**Author:** ZenSci Architecture (TresPiesDesign.com)
**Status:** Production Ready
**Created:** 2026-02-18
**Version:** 1.0.0
**Audience:** All module developers; implemented by architecture team

---

## 1. Overview

**packages/core** is the foundational shared library imported by all 5+ ZenSci module servers. It provides:

- **Parsing cluster:** Markdown parsing (remark/unified), frontmatter extraction (gray-matter), AST node definitions
- **Citation cluster:** BibTeX parsing, CSL bibliography rendering, citation resolution
- **Validation cluster:** Schema validation (Zod), math expression validation (SymPy subprocess), link checking
- **Type definitions:** Complete TypeScript interfaces for `DocumentRequest`, `DocumentResponse`, `OutputFormat`, and all AST node types
- **Error handling patterns:** `ConversionError`, `ConversionWarning`, `ValidationResult` types

**Key design principle:** packages/core is *transport-agnostic* — it doesn't know about MCP, stdio, or network protocols. It operates on pure data structures. Module servers (in packages/sdk) wrap these functions for MCP transport.

---

## 2. Architecture

### 2.1 Public API Surface

All modules import from `@zen-sci/core`:

```typescript
// packages/core/src/index.ts
export { MarkdownParser } from './parsing/markdown-parser.js';
export { FrontmatterExtractor } from './parsing/frontmatter-extractor.js';
export { CitationManager } from './citations/citation-manager.js';
export { BibTeXParser } from './citations/bibtex-parser.js';
export { CSLRenderer } from './citations/csl-renderer.js';
export { SchemaValidator } from './validation/schema-validator.js';
export { MathValidator } from './validation/math-validator.js';
export { LinkChecker } from './validation/link-checker.js';
export { ConversionPipeline } from './pipeline/conversion-pipeline.js';
export type * from './types.js';
```

### 2.2 Directory Structure

```
packages/core/
├── src/
│   ├── index.ts                          # Public API exports
│   ├── types.ts                          # All TypeScript interfaces
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

---

## 3. Complete Type Definitions

### 3.1 Core Data Contracts

```typescript
// packages/core/src/types.ts

/**
 * DocumentRequest: Input structure for all module conversion operations.
 * Defines the source content, target format, and processing options.
 */
export interface DocumentRequest {
  /** Unique identifier for this request (UUID or user-provided) */
  id: string;

  /** Source content: raw markdown or plaintext */
  source: string;

  /** Target output format */
  format: OutputFormat;

  /** YAML frontmatter metadata (parsed from source or provided separately) */
  frontmatter: FrontmatterMetadata;

  /** Optional: BibTeX bibliography file content or path */
  bibliography?: string;

  /** Processing options (varies by module) */
  options: DocumentOptions;

  /** Optional: Thinking session context (for research-aware formatting) */
  thinkingSession?: ThinkingSession;
}

/**
 * OutputFormat: Union of all supported output formats across all ZenSci modules.
 * Modules implement a subset of these; SchemaValidator ensures format is valid for the module.
 */
export type OutputFormat =
  // LaTeX-based (beamer, paper, thesis, patent, grant)
  | 'latex'
  | 'beamer'
  | 'grant-latex'
  | 'paper-ieee'
  | 'paper-acm'
  | 'paper-arxiv'
  | 'thesis'
  | 'patent'

  // Presentation-based
  | 'revealjs'
  | 'pptx'

  // Web/markup
  | 'html'
  | 'email'
  | 'epub'
  | 'mobi'

  // Document formats
  | 'docs'
  | 'docx'

  // Lab/research
  | 'lab-notebook'

  // Specialized (Phase 3)
  | 'policy-brief'
  | 'proposal'
  | 'podcast-notes'
  | 'resume'
  | 'whitepaper';

/**
 * DocumentOptions: Configurable conversion settings.
 * Modules extend this with moduleOptions: Record<string, unknown> for format-specific settings.
 */
export interface DocumentOptions {
  /** Override title from frontmatter */
  title?: string;

  /** Override author(s) from frontmatter */
  author?: string[];

  /** Override date from frontmatter (YYYY-MM-DD) */
  date?: string;

  /** Generate table of contents */
  toc?: boolean;

  /** Bibliography and citation options */
  bibliography?: BibliographyOptions;

  /** Math rendering options */
  math?: MathOptions;

  /** Format-specific options (e.g., { theme: 'dark', fontSize: 14 }) */
  moduleOptions?: Record<string, unknown>;
}

/**
 * FrontmatterMetadata: YAML frontmatter parsed from markdown source.
 * Supports arbitrary keys; common keys listed below.
 */
export interface FrontmatterMetadata {
  title?: string;
  author?: string | string[];
  date?: string; // YYYY-MM-DD or free-form
  tags?: string[];
  description?: string;
  keywords?: string[];
  lang?: string; // ISO 639-1 code (en, fr, etc.)
  [key: string]: unknown; // Supports custom keys
}

/**
 * BibliographyOptions: Configuration for citation and bibliography rendering.
 */
export interface BibliographyOptions {
  /** CSL style: 'apa', 'ieee', 'chicago', 'mla', 'harvard', etc. */
  style?: BibliographyStyle;

  /** Include full bibliography at end of document */
  include?: boolean;

  /** Sort bibliography: 'citation-order' or 'alphabetical' */
  sortBy?: 'citation-order' | 'alphabetical';

  /** Citation linking (PDF only): 'href' links to bibliography entries */
  link?: boolean;
}

export type BibliographyStyle =
  | 'apa'
  | 'ieee'
  | 'chicago'
  | 'mla'
  | 'harvard'
  | 'vancouver'
  | 'nature'
  | 'arxiv'
  | 'custom';

/**
 * MathOptions: Configuration for math expression rendering.
 */
export interface MathOptions {
  /** Validate math syntax before rendering */
  validate?: boolean;

  /** Render engine: 'mathjax', 'katex', 'unicode' */
  engine?: 'mathjax' | 'katex' | 'unicode';

  /** Number equations (LaTeX only) */
  numberEquations?: boolean;

  /** Inline math delimiter (e.g., '$...$' or '\\(...\\)') */
  inlineDelimiter?: string;

  /** Display math delimiter (e.g., '$$...$$' or '\\[...\\]') */
  displayDelimiter?: string;
}

/**
 * DocumentResponse: Output structure returned by all module conversion operations.
 * Contains primary content + artifacts + metadata + warnings.
 */
export interface DocumentResponse {
  /** Echo request ID for tracing */
  id: string;

  /** Request ID (if different from id) */
  requestId: string;

  /** Confirmed output format */
  format: OutputFormat;

  /** Primary content: string (text/HTML/markdown/XML) or Buffer (binary PDF/DOCX) */
  content: string | Buffer;

  /** Secondary files (embedded images, CSS files, footnotes.bib, etc.) */
  artifacts: Artifact[];

  /** Non-blocking warnings (missing images, unsupported syntax, etc.) */
  warnings: ConversionWarning[];

  /** Response metadata (author, generated date, word count, etc.) */
  metadata: ResponseMetadata;

  /** Total conversion time in milliseconds */
  elapsed: number;
}

/**
 * Artifact: Secondary file generated during conversion (not primary content).
 */
export interface Artifact {
  /** Filename (e.g., 'image-1.png', 'bibliography.bib') */
  name: string;

  /** Relative path for embedding in output (e.g., 'assets/image-1.png') */
  path: string;

  /** MIME type (e.g., 'image/png', 'text/plain') */
  mimeType: string;

  /** File size in bytes */
  size: number;

  /** Optional: binary content (if small enough to include in response) */
  content?: Buffer;
}

/**
 * ConversionWarning: Non-blocking issue detected during conversion.
 * Conversion succeeds but output may be incomplete or degraded.
 */
export interface ConversionWarning {
  /** Machine-readable code: 'missing-image', 'orphaned-citation', 'unsupported-syntax' */
  code: string;

  /** Human-readable message */
  message: string;

  /** Location in source (if applicable) */
  location?: DocumentLocation;

  /** Suggested fix or workaround */
  suggestion?: string;
}

/**
 * ConversionErrorData: Serializable error shape for wire transport (JSON-safe).
 * Use this for error data in API responses, pipeline results, and MCP messages.
 * See ConversionError class (Section 6) for the throwable runtime version.
 */
export interface ConversionErrorData {
  /** Machine-readable error code */
  code: string;

  /** Human-readable error message */
  message: string;

  /** Location in source (line, column, path) */
  location?: DocumentLocation;

  /** Suggested user actions to resolve */
  suggestions?: string[];

  /** Stack trace (development only) */
  stack?: string;
}

/**
 * DocumentLocation: Precise location in source document.
 */
export interface DocumentLocation {
  line?: number; // 1-indexed
  column?: number; // 1-indexed
  path?: string; // File path (for multi-file inputs)
}

/**
 * ResponseMetadata: Metadata about conversion output.
 */
export interface ResponseMetadata {
  /** Final title (from frontmatter or override) */
  title?: string;

  /** Final author(s) */
  author?: string[];

  /** Generation timestamp (ISO 8601) */
  generatedAt: string;

  /** Word count in primary content */
  wordCount?: number;

  /** Page count (if applicable to format) */
  pageCount?: number;

  /** Bibliography style applied */
  bibliographyStyle?: BibliographyStyle;

  /** Number of citations in output */
  citationCount?: number;

  /** List of sources/dependencies (for reproducibility) */
  sources?: string[];
}

/**
 * ConversionPipeline: Execution trace for multi-stage conversion.
 * Used for monitoring, debugging, and progress reporting.
 */
export interface ConversionPipeline {
  /** Unique pipeline ID (UUID) */
  id: string;

  /** Associated request ID for tracing */
  requestId: string;

  /** Ordered list of pipeline stages */
  stages: PipelineStage[];

  /** Pipeline start timestamp */
  startedAt: Date;

  /** Pipeline completion timestamp (null if in progress) */
  completedAt?: Date;

  /** Overall status */
  status: 'pending' | 'running' | 'completed' | 'failed';

  /** Overall result */
  result?: { success: boolean; error?: ConversionErrorData };
}

/**
 * PipelineStage: Single stage in conversion pipeline.
 * Standard stages: parse → validate → transform → render → compile → output
 */
export interface PipelineStage {
  /** Stage name (canonical, known set) */
  name: 'parse' | 'validate' | 'transform' | 'render' | 'compile' | 'output';

  /** Current status */
  status: 'pending' | 'running' | 'complete' | 'failed';

  /** Start timestamp */
  startedAt?: Date;

  /** Duration in milliseconds */
  elapsed?: number;

  /** Error (if status === 'failed') */
  error?: ConversionErrorData;

  /** Progress (if applicable): 0-100 */
  progress?: number;
}

/**
 * ThinkingSession: Optional context from user's thinking/reasoning process.
 * Allows modules to make format decisions based on user's stated intent.
 */
export interface ThinkingSession {
  /** Session ID */
  id: string;

  /** Original prompt/question that spawned the document */
  prompt: string;

  /** Chain of reasoning steps */
  reasoningChain: ReasoningStep[];

  /** Key decisions made during thinking */
  decisions: SessionDecision[];

  /** User's stated intent for output (e.g., 'publish to journal' vs 'internal memo') */
  outputIntent: string;
}

export interface ReasoningStep {
  step: number;
  thought: string;
  conclusion: string;
}

export interface SessionDecision {
  decision: string;
  rationale: string;
}

/**
 * ValidationResult: Output of schema or content validation.
 */
export interface ValidationResult {
  /** Overall validation passed */
  valid: boolean;

  /** Blocking errors */
  errors: ConversionErrorData[];

  /** Non-blocking warnings */
  warnings: ConversionWarning[];

  /** Timestamp of validation */
  validatedAt: Date;
}
```

### 3.2 Document Tree & AST Nodes

```typescript
/**
 * DocumentTree: Complete parsed document structure.
 * Root node for all AST operations.
 */
export interface DocumentTree {
  type: 'document';
  children: DocumentNode[];
  frontmatter: FrontmatterMetadata;
  bibliography: CitationRecord[];
}

/**
 * DocumentNode: Union of all possible document AST node types.
 * Recursive structure supporting arbitrary nesting (sections contain sections, etc.).
 */
export type DocumentNode =
  // Structural
  | SectionNode
  | ParagraphNode

  // Math
  | MathNode

  // Code
  | CodeBlockNode

  // Media
  | FigureNode
  | TableNode

  // Citations & metadata
  | CitationNode;

export interface SectionNode {
  type: 'section';
  level: 1 | 2 | 3 | 4 | 5 | 6;
  title: string;
  id?: string; // For cross-references
  children: DocumentNode[];
}

export interface ParagraphNode {
  type: 'paragraph';
  children: InlineNode[];
}

export interface MathNode {
  type: 'math';
  mode: 'inline' | 'display';
  latex: string;
  /** True if validated by SymPy */
  validated: boolean;
  /** ASCII alternative for accessibility */
  alt?: string;
}

export interface CodeBlockNode {
  type: 'code';
  language: string; // e.g., 'python', 'rust', 'json'
  content: string;
  /** Line number markers (if source has them) */
  lineNumbers?: boolean;
  /** Highlighted line ranges (for emphasis) */
  highlight?: [number, number][]; // [[1, 3], [10, 12]]
  /** Filename or label */
  label?: string;
}

export interface FigureNode {
  type: 'figure';
  src: string; // URL or relative path
  caption?: string;
  label?: string; // For cross-references
  width?: string; // CSS width
  height?: string; // CSS height
  alt: string; // Accessibility
}

export interface TableNode {
  type: 'table';
  headers: string[];
  /** Each row is array of cell contents (can contain nested InlineNode) */
  rows: (InlineNode[] | string)[][];
  /** Table caption/label */
  caption?: string;
  label?: string; // For cross-references
}

export interface CitationNode {
  type: 'citation';
  key: string; // BibTeX key: @key
  resolved?: CitationRecord; // If citation was resolved from .bib
}

/**
 * InlineNode: Inline content within paragraphs and cells.
 */
export type InlineNode =
  | TextNode
  | EmphasisNode
  | StrongNode
  | CodeNode
  | LinkNode
  | CitationReferenceNode;

export interface TextNode {
  type: 'text';
  text: string;
}

export interface EmphasisNode {
  type: 'emphasis';
  children: InlineNode[];
}

export interface StrongNode {
  type: 'strong';
  children: InlineNode[];
}

export interface CodeNode {
  type: 'code';
  code: string;
}

export interface LinkNode {
  type: 'link';
  url: string;
  title?: string;
  children: InlineNode[];
}

export interface CitationReferenceNode {
  type: 'citation-reference';
  key: string;
  prefix?: string; // e.g., "see "
  suffix?: string; // e.g., " for details"
}
```

### 3.3 Citation Types

```typescript
/**
 * CitationRecord: Structured citation metadata parsed from BibTeX.
 */
export interface CitationRecord {
  /** BibTeX key (unique identifier within bibliography) */
  key: string;

  /** Citation type (BibTeX entry type) */
  type: 'article' | 'book' | 'inproceedings' | 'techreport' | 'thesis' | 'misc' | 'other';

  /** Title */
  title: string;

  /** Author(s) */
  author: string[];

  /** Publication year */
  year: number;

  /** Journal name (for articles) */
  journal?: string;

  /** Book title or conference name (for chapters/proceedings) */
  booktitle?: string;

  /** Publisher */
  publisher?: string;

  /** Volume */
  volume?: string;

  /** Issue */
  issue?: string;

  /** Pages or page range */
  pages?: string;

  /** Digital Object Identifier */
  doi?: string;

  /** URL */
  url?: string;

  /** Date accessed (ISO 8601) */
  accessed?: string;

  /** Raw BibTeX entry (for fallback rendering) */
  raw: Record<string, string>;
}
```

### 3.4 Extension & Processing Types

```typescript
/**
 * ConversionRule: Mapping from a markdown/AST construct to its target format representation.
 * Registered in each module's rule registry; applied by the transform stage.
 */
export interface ConversionRule {
  /** Source node type (e.g., 'math', 'table', 'figure', 'code') */
  sourceType: string;

  /** Target format this rule applies to */
  targetFormat: OutputFormat;

  /** Transform function identifier (looked up in module's rule registry) */
  transform: string;

  /** Priority (higher overrides lower when rules conflict) */
  priority: number;

  /** Whether conversion fails without this rule (true) or falls back to default (false) */
  required: boolean;

  /** Human-readable description */
  description?: string;

  /** Options forwarded to the transform function */
  options?: Record<string, unknown>;
}

/**
 * FormatConstraint: Structural and sizing limits declared per output format.
 * Used by the validate stage to enforce format-specific requirements before rendering.
 */
export interface FormatConstraint {
  /** Format these constraints apply to */
  format: OutputFormat;

  /** Max width in pixels (email, HTML) */
  maxWidth?: number;

  /** Slide aspect ratio (strict enum — add new values to this union as needed) */
  aspectRatio?: '16:9' | '4:3' | '1:1' | '3:2' | '21:9';

  /** Page margins in inches (PDF formats) */
  margins?: { top: number; right: number; bottom: number; left: number };

  /** Minimum font size in points (print formats) */
  minFontSize?: number;

  /** Maximum page count */
  maxPages?: number;

  /** Maximum word count */
  maxWords?: number;

  /** Maximum output file size in bytes */
  maxFileSizeBytes?: number;

  /** Whether embedded images are supported in this format */
  supportsImages?: boolean;
}

/**
 * ImageProcessingOptions: Configuration for image resize, compress, and format convert.
 * Used when figure nodes need to be adapted for a target format.
 */
export interface ImageProcessingOptions {
  /** Target width in pixels (null = derive from height) */
  width?: number | null;

  /** Target height in pixels (null = derive from width) */
  height?: number | null;

  /** Output format */
  format?: 'png' | 'jpg' | 'webp' | 'svg' | 'original';

  /** Compression quality (1–100; default: 85) */
  quality?: number;

  /** Resize fit mode */
  fit?: 'cover' | 'contain' | 'fill' | 'inside' | 'outside';

  /** Explicitly set alt text */
  alt?: string;

  /** Auto-generate alt text when missing (default: true) */
  generateAlt?: boolean;
}

/**
 * ImageProcessingResult: Output of a single image processing operation.
 */
export interface ImageProcessingResult {
  /** Original source path or URL */
  source: string;

  /** Processed image buffer */
  buffer: Buffer;

  /** MIME type of processed output */
  mimeType: string;

  /** Final width in pixels */
  width: number;

  /** Final height in pixels */
  height: number;

  /** Processed file size in bytes */
  sizeBytes: number;

  /** Final alt text (generated or from options.alt) */
  alt: string;

  /** Non-blocking issues (e.g., low resolution, missing source alt text) */
  warnings: ConversionWarning[];
}

/**
 * AccessibilityReport: WCAG 2.1 compliance results for HTML, email, and slides output.
 * Produced by the validate stage when the target format supports accessibility checking.
 */
export interface AccessibilityReport {
  /** Highest WCAG level fully achieved */
  level: 'AAA' | 'AA' | 'A' | 'none';

  /** Format that was checked */
  format: OutputFormat;

  /** Whether output meets the minimum AA threshold */
  isCompliant: boolean;

  /** Blocking WCAG violations */
  violations: AccessibilityViolation[];

  /** Non-blocking advisory warnings */
  warnings: AccessibilityWarning[];

  /** Composite score 0–100 (higher = more accessible) */
  score: number;

  /** Timestamp of validation run (ISO 8601) */
  validatedAt: string;
}

export interface AccessibilityViolation {
  /** WCAG success criterion reference (e.g., '1.1.1', '1.4.3') */
  criterion: string;

  /** Human-readable description */
  description: string;

  /** Severity level */
  severity: 'critical' | 'serious' | 'moderate' | 'minor';

  /** Element selector or location hint in output */
  location?: string;

  /** How to fix this violation */
  remediation: string;
}

export interface AccessibilityWarning {
  /** Check category (e.g., 'color-contrast', 'missing-landmark', 'skip-link') */
  check: string;

  /** Human-readable description */
  description: string;

  /** Recommended action */
  recommendation: string;
}

/**
 * WebMetadataSchema: OpenGraph, Twitter Card, and Schema.org structured data for web output.
 * Used by blog-mcp and newsletter-mcp; blog-mcp's SEOMetadata extends this base type.
 */
export interface WebMetadataSchema {
  /** OpenGraph tags for social sharing */
  og?: OpenGraphMetadata;

  /** Twitter Card tags */
  twitter?: TwitterCardMetadata;

  /** Schema.org JSON-LD structured data */
  structuredData?: SchemaOrgData;

  /** Canonical URL */
  canonical?: string;

  /** Language (BCP 47, e.g., 'en-US') */
  lang?: string;

  /** Meta description (max 160 chars) */
  description?: string;

  /** Robots directive */
  robots?: 'index,follow' | 'noindex,follow' | 'index,nofollow' | 'noindex,nofollow';
}

export interface OpenGraphMetadata {
  title: string;
  description?: string;
  image?: string;       // Absolute URL
  imageAlt?: string;
  type: 'article' | 'website' | 'profile';
  url?: string;
  siteName?: string;
  locale?: string;
  publishedTime?: string;  // ISO 8601
  modifiedTime?: string;   // ISO 8601
  author?: string;
  section?: string;
  tags?: string[];
}

export interface TwitterCardMetadata {
  card: 'summary' | 'summary_large_image' | 'app' | 'player';
  site?: string;     // @handle
  creator?: string;  // @handle
  title?: string;
  description?: string;
  image?: string;    // Absolute URL
  imageAlt?: string;
}

export interface SchemaOrgData {
  '@context': 'https://schema.org';
  '@type': 'Article' | 'BlogPosting' | 'NewsArticle' | 'ScholarlyArticle' | 'WebPage';
  headline?: string;
  description?: string;
  author?: { '@type': 'Person' | 'Organization'; name: string };
  datePublished?: string;
  dateModified?: string;
  image?: string;
  url?: string;
  keywords?: string;
  publisher?: {
    '@type': 'Organization';
    name: string;
    logo?: { '@type': 'ImageObject'; url: string };
  };
}

/**
 * DocumentVersion: A single named version of a document (draft, reviewed, final).
 * Version history is tracked in DocumentVersionHistory.
 * Not required for v0.1 single-user mode; designed for collaborative review workflows.
 */
export interface DocumentVersion {
  /** Semantic version string (e.g., '0.1.0', '1.0.0') */
  version: string;

  /** Author of this version */
  author: string;

  /** Creation timestamp (ISO 8601) */
  createdAt: string;

  /** Brief summary of what changed */
  summary: string;

  /** Detailed change list */
  changes: DocumentChange[];

  /** Workflow tag */
  tag?: 'draft' | 'under-review' | 'final' | 'submitted' | 'archived';

  /** SHA-256 hash of content at this version (for integrity checking) */
  contentHash?: string;

  /** Reference to the prior version string, if any */
  parentVersion?: string;
}

export interface DocumentChange {
  /** Type of change */
  type: 'addition' | 'deletion' | 'modification' | 'restructure';

  /** Section or location where change occurred */
  section?: string;

  /** Human-readable description */
  description: string;

  /** Reviewer or collaborator note */
  comment?: string;
}

export interface DocumentVersionHistory {
  /** Document identifier */
  documentId: string;

  /** All versions in chronological order (oldest first) */
  versions: DocumentVersion[];

  /** Version string of the current/latest version */
  currentVersion: string;
}
```

---

## 4. Public Classes & Interfaces

### 4.1 MarkdownParser

```typescript
/**
 * MarkdownParser: Parse markdown source into DocumentTree AST.
 * Built on remark/unified ecosystem.
 */
export class MarkdownParser {
  /**
   * Parse markdown source into DocumentTree with AST nodes.
   * @param source Raw markdown string
   * @param options Parsing options
   * @returns Parsed DocumentTree
   */
  parse(source: string, options?: MarkdownParseOptions): DocumentTree;

  /**
   * Extract frontmatter from markdown source without full parsing.
   * @param source Raw markdown string
   * @returns Parsed frontmatter object
   */
  extractFrontmatter(source: string): FrontmatterMetadata;

  /**
   * Parse and return both tree and frontmatter.
   * @param source Raw markdown string
   * @returns { tree, frontmatter }
   */
  parseComplete(source: string): { tree: DocumentTree; frontmatter: FrontmatterMetadata };

  /**
   * Validate markdown syntax without full parsing.
   * @param source Raw markdown string
   * @returns ValidationResult
   */
  validate(source: string): ValidationResult;
}

export interface MarkdownParseOptions {
  /** Strict mode: throw on syntax errors (default: false, warnings only) */
  strict?: boolean;

  /** Include position data (line/column) in AST (default: true) */
  includePositions?: boolean;

  /** Detected language for highlighting code blocks (default: 'en') */
  lang?: string;
}
```

### 4.2 FrontmatterExtractor

```typescript
/**
 * FrontmatterExtractor: YAML frontmatter parsing (gray-matter based).
 */
export class FrontmatterExtractor {
  /**
   * Extract YAML frontmatter from markdown source.
   * @param source Raw markdown string
   * @returns Parsed FrontmatterMetadata and remaining content
   */
  extract(source: string): { frontmatter: FrontmatterMetadata; content: string };

  /**
   * Inject frontmatter into markdown source (prepend as YAML block).
   * @param content Markdown content
   * @param frontmatter Metadata object
   * @returns Markdown with prepended frontmatter
   */
  inject(content: string, frontmatter: FrontmatterMetadata): string;

  /**
   * Validate frontmatter object against schema.
   * @param frontmatter Metadata object
   * @returns ValidationResult
   */
  validate(frontmatter: FrontmatterMetadata): ValidationResult;

  /**
   * Merge two frontmatter objects (second overrides first).
   * @param base First frontmatter
   * @param override Second frontmatter
   * @returns Merged result
   */
  merge(base: FrontmatterMetadata, override: FrontmatterMetadata): FrontmatterMetadata;
}
```

### 4.3 CitationManager

```typescript
/**
 * CitationManager: Resolve and render citations.
 * Coordinates BibTeXParser and CSLRenderer.
 */
export class CitationManager {
  constructor(bibliography: string); // BibTeX content or path

  /**
   * Parse bibliography and return all citation records.
   * @returns Array of CitationRecord
   */
  getAllRecords(): CitationRecord[];

  /**
   * Resolve a single citation by key.
   * @param key BibTeX key (e.g., 'smith2020')
   * @returns CitationRecord or null if not found
   */
  resolve(key: string): CitationRecord | null;

  /**
   * Resolve multiple citations by key.
   * @param keys Array of BibTeX keys
   * @returns Array of CitationRecord, maintaining order
   */
  resolveMultiple(keys: string[]): CitationRecord[];

  /**
   * Find citations matching a query (title, author, year).
   * @param query Search string
   * @returns Array of matching CitationRecord
   */
  search(query: string): CitationRecord[];

  /**
   * Render citation in given CSL style.
   * @param key BibTeX key
   * @param style CSL style name
   * @returns Formatted citation string
   */
  formatCitation(key: string, style: BibliographyStyle): string;

  /**
   * Render entire bibliography in given style.
   * @param keys Array of BibTeX keys (if null, includes all)
   * @param style CSL style name
   * @returns Formatted bibliography (multiline string)
   */
  formatBibliography(keys?: string[], style?: BibliographyStyle): string;

  /**
   * Extract all citation keys from markdown AST.
   * @param tree DocumentTree
   * @returns Array of citation keys found
   */
  extractKeysFromAST(tree: DocumentTree): string[];

  /**
   * Validate all citations in AST (orphaned reference check).
   * @param tree DocumentTree
   * @returns ValidationResult
   */
  validateAST(tree: DocumentTree): ValidationResult;
}
```

### 4.4 BibTeXParser

```typescript
/**
 * BibTeXParser: Parse BibTeX files into structured CitationRecord array.
 */
export class BibTeXParser {
  /**
   * Parse BibTeX content.
   * @param content BibTeX text
   * @returns Array of CitationRecord
   */
  parse(content: string): CitationRecord[];

  /**
   * Validate BibTeX content without full parsing.
   * @param content BibTeX text
   * @returns ValidationResult
   */
  validate(content: string): ValidationResult;

  /**
   * Generate BibTeX from CitationRecord array.
   * @param records Array of CitationRecord
   * @returns BibTeX text
   */
  generate(records: CitationRecord[]): string;
}
```

### 4.5 CSLRenderer

```typescript
/**
 * CSLRenderer: Format citations using CSL (Citation Style Language).
 * Uses embedded CSL style definitions or fetches from Zotero CSL repository.
 */
export class CSLRenderer {
  /**
   * Format single citation.
   * @param record CitationRecord
   * @param style CSL style name
   * @returns Formatted citation string
   */
  formatCitation(record: CitationRecord, style: BibliographyStyle): string;

  /**
   * Format bibliography.
   * @param records Array of CitationRecord
   * @param style CSL style name
   * @returns Formatted bibliography (multiline)
   */
  formatBibliography(records: CitationRecord[], style: BibliographyStyle): string;

  /**
   * List available built-in CSL styles.
   * @returns Array of style names
   */
  listStyles(): BibliographyStyle[];

  /**
   * Load custom CSL style definition.
   * @param styleName Style identifier
   * @param styleXML CSL XML content
   */
  registerCustomStyle(styleName: string, styleXML: string): void;
}
```

### 4.6 SchemaValidator

```typescript
/**
 * SchemaValidator: Validate DocumentRequest against format-specific schema.
 * Uses Zod for runtime type checking.
 */
export class SchemaValidator {
  /**
   * Validate DocumentRequest for a specific module format.
   * @param request DocumentRequest
   * @param supportedFormats Array of OutputFormat values the module supports
   * @returns ValidationResult
   */
  static validate(request: DocumentRequest, supportedFormats: OutputFormat[]): ValidationResult;

  /**
   * Validate frontmatter metadata.
   * @param frontmatter FrontmatterMetadata
   * @returns ValidationResult
   */
  static validateFrontmatter(frontmatter: FrontmatterMetadata): ValidationResult;

  /**
   * Validate bibliography options.
   * @param options BibliographyOptions
   * @returns ValidationResult
   */
  static validateBibliographyOptions(options: BibliographyOptions): ValidationResult;

  /**
   * Validate output format.
   * @param format OutputFormat value
   * @param supportedFormats Array of supported formats
   * @returns true if format is supported
   */
  static isFormatSupported(format: OutputFormat, supportedFormats: OutputFormat[]): boolean;
}
```

### 4.7 MathValidator

```typescript
/**
 * MathValidator: Validate LaTeX math expressions.
 * Calls Python SymPy subprocess for syntax checking.
 */
export class MathValidator {
  /**
   * Validate single math expression.
   * @param latex LaTeX string (without delimiters)
   * @param mode 'inline' or 'display'
   * @returns ValidationResult
   */
  async validate(latex: string, mode: 'inline' | 'display'): Promise<ValidationResult>;

  /**
   * Validate all math in DocumentTree.
   * @param tree DocumentTree
   * @returns ValidationResult with all math errors
   */
  async validateTree(tree: DocumentTree): Promise<ValidationResult>;

  /**
   * Attempt to parse and simplify expression via SymPy.
   * @param latex LaTeX expression
   * @returns Simplified expression or error
   */
  async simplify(latex: string): Promise<string>;

  /**
   * Convert LaTeX to ASCII representation for accessibility.
   * @param latex LaTeX expression
   * @returns ASCII fallback
   */
  async toAscii(latex: string): Promise<string>;
}
```

### 4.8 LinkChecker

```typescript
/**
 * LinkChecker: Validate URLs and internal document references.
 * Async HTTP checking for external links; structural validation for internal refs.
 */
export class LinkChecker {
  /**
   * Check single URL (HTTP GET with timeout).
   * @param url URL to check
   * @param timeout Timeout in milliseconds (default: 5000)
   * @returns { valid: boolean, statusCode?: number, error?: string }
   */
  async checkUrl(url: string, timeout?: number): Promise<LinkCheckResult>;

  /**
   * Check all links in DocumentTree.
   * @param tree DocumentTree
   * @param options Check options
   * @returns ValidationResult with link errors
   */
  async checkTree(tree: DocumentTree, options?: LinkCheckOptions): Promise<ValidationResult>;

  /**
   * Validate internal references (cross-references to sections, figures).
   * @param tree DocumentTree
   * @returns ValidationResult with unresolved reference errors
   */
  validateInternalReferences(tree: DocumentTree): ValidationResult;

  /**
   * Extract all links from DocumentTree.
   * @param tree DocumentTree
   * @returns Array of { type, url, location }
   */
  extractLinks(tree: DocumentTree): Link[];
}

export interface LinkCheckResult {
  valid: boolean;
  url: string;
  statusCode?: number;
  error?: string;
  checkedAt: Date;
}

export interface LinkCheckOptions {
  /** Skip external URL checking (check internal refs only) */
  skipExternal?: boolean;

  /** Skip internal reference validation */
  skipInternal?: boolean;

  /** Timeout per URL in milliseconds */
  timeout?: number;

  /** Retry count for failed requests */
  retries?: number;
}

export interface Link {
  type: 'external' | 'internal' | 'anchor';
  url: string;
  location?: DocumentLocation;
}
```

---

## 5. ConversionPipeline Class

```typescript
/**
 * ConversionPipeline: Execute and monitor multi-stage document conversion.
 * Used by modules to track progress and handle errors.
 */
export class ConversionPipeline {
  constructor(requestId: string);

  /**
   * Start the pipeline and initialize stages.
   */
  start(): void;

  /**
   * Mark a stage as running.
   * @param stageName Stage name
   */
  startStage(stageName: PipelineStage['name']): void;

  /**
   * Complete a stage successfully.
   * @param stageName Stage name
   * @param progress Optional progress percentage (0-100)
   */
  completeStage(stageName: PipelineStage['name'], progress?: number): void;

  /**
   * Fail a stage with error.
   * @param stageName Stage name
   * @param error ConversionErrorData
   */
  failStage(stageName: PipelineStage['name'], error: ConversionErrorData): void;

  /**
   * Finish pipeline (successful or failed).
   * @param success Whether conversion succeeded overall
   * @param result Optional result or final error
   */
  complete(success: boolean, result?: unknown, error?: ConversionErrorData): void;

  /**
   * Get current pipeline state.
   * @returns ConversionPipeline object
   */
  getState(): ConversionPipeline;

  /**
   * Get duration so far in milliseconds.
   * @returns Milliseconds since start
   */
  elapsed(): number;
}
```

---

## 6. Error Handling Patterns

### 6.1 Error Classes (Optional: extend Error for type safety)

```typescript
/**
 * Base class for ZenSci errors (optional, for instanceof checks).
 */
export class ZenSciError extends Error {
  constructor(
    public code: string,
    message: string,
    public location?: DocumentLocation,
    public suggestions?: string[]
  ) {
    super(message);
    this.name = 'ZenSciError';
  }
}

/**
 * Parsing errors (malformed markdown, invalid YAML).
 */
export class ParseError extends ZenSciError {
  constructor(message: string, location?: DocumentLocation, suggestions?: string[]) {
    super('PARSE_ERROR', message, location, suggestions);
    this.name = 'ParseError';
  }
}

/**
 * Validation errors (schema mismatch, missing citations).
 */
export class ValidationError extends ZenSciError {
  constructor(message: string, location?: DocumentLocation, suggestions?: string[]) {
    super('VALIDATION_ERROR', message, location, suggestions);
    this.name = 'ValidationError';
  }
}

/**
 * ConversionError: Runtime error class for throw/catch.
 * Use this in code; use ConversionErrorData (Section 3.1) for wire transport.
 */
export class ConversionError extends ZenSciError {
  constructor(message: string, location?: DocumentLocation, suggestions?: string[]) {
    super('CONVERSION_ERROR', message, location, suggestions);
    this.name = 'ConversionError';
  }

  /**
   * Construct a ConversionError from a ConversionErrorData object.
   * Use when deserializing errors from JSON/wire format.
   */
  static fromData(data: ConversionErrorData): ConversionError {
    const error = new ConversionError(data.message, data.location, data.suggestions);
    if (data.stack) error.stack = data.stack;
    return error;
  }

  /**
   * Serialize to ConversionErrorData (wire-safe).
   */
  toData(): ConversionErrorData {
    return {
      code: this.code,
      message: this.message,
      location: this.location,
      suggestions: this.suggestions,
      stack: this.stack,
    };
  }
}
```

---

## 7. Dependencies

### 7.1 npm Dependencies

```json
{
  "dependencies": {
    "unified": "^10.1.0",
    "remark-parse": "^10.0.0",
    "remark-frontmatter": "^5.0.0",
    "gray-matter": "^4.0.3",
    "bibtex-parser": "^0.2.0",
    "zod": "^3.22.0",
    "citeproc": "^2.4.0",
    "@types/node": "^20.0.0"
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "vitest": "^1.0.0",
    "@types/jest": "^29.0.0"
  }
}
```

### 7.2 External Services

- **Zotero CSL Repository** (optional): Fetch CSL style definitions for rare bibliography styles
- **Python subprocess** (for MathValidator): SymPy must be installed on system running ZenSci

---

## 8. Testing Strategy

### 8.1 Unit Tests

```typescript
// packages/core/tests/parsing.test.ts
import { describe, it, expect } from 'vitest';
import { MarkdownParser } from '../src/parsing/markdown-parser';

describe('MarkdownParser', () => {
  it('parses simple markdown into DocumentTree', () => {
    const parser = new MarkdownParser();
    const tree = parser.parse('# Title\nParagraph text.');
    expect(tree.type).toBe('document');
    expect(tree.children[0].type).toBe('section');
    expect(tree.children[0].level).toBe(1);
  });

  it('extracts frontmatter correctly', () => {
    const parser = new MarkdownParser();
    const source = `---\ntitle: Test\nauthor: John\n---\n# Content`;
    const { tree, frontmatter } = parser.parseComplete(source);
    expect(frontmatter.title).toBe('Test');
    expect(frontmatter.author).toBe('John');
  });
});

// packages/core/tests/citations.test.ts
describe('CitationManager', () => {
  it('resolves citation by key', () => {
    const bib = '@article{smith2020,\ntitle={Test},\nauthor={Smith},\nyear={2020}\n}';
    const manager = new CitationManager(bib);
    const record = manager.resolve('smith2020');
    expect(record?.title).toBe('Test');
  });

  it('validates bibliography syntax', () => {
    const manager = new CitationManager('@article{broken,\ntitle={No closing brace');
    const result = manager.getAllRecords();
    // Should handle gracefully
  });
});

// packages/core/tests/validation.test.ts
describe('SchemaValidator', () => {
  it('validates DocumentRequest for supported formats', () => {
    const request: DocumentRequest = {
      id: 'test-1',
      source: '# Title',
      format: 'latex',
      frontmatter: { title: 'Test' },
      options: {},
    };
    const result = SchemaValidator.validate(request, ['latex', 'pdf']);
    expect(result.valid).toBe(true);
  });

  it('rejects unsupported format', () => {
    const request = { ...validRequest, format: 'unknown' as OutputFormat };
    const result = SchemaValidator.validate(request, ['latex']);
    expect(result.valid).toBe(false);
  });
});
```

### 8.2 Integration Tests

Test complete pipelines: parse markdown → extract citations → validate → render output.

---

## 9. Versioning Strategy

### 9.1 Semantic Versioning

**packages/core** follows Semantic Versioning strictly:

- **MAJOR (1.0.0 → 2.0.0):** Breaking API changes (removed classes, changed interface signatures)
- **MINOR (1.0.0 → 1.1.0):** New features, backward-compatible
- **PATCH (1.0.0 → 1.0.1):** Bug fixes

### 9.2 Monorepo Version Coordination

All ZenSci packages use **locked versioning** within the monorepo:

```json
{
  "workspaces": [
    "packages/core",
    "packages/sdk",
    "servers/*"
  ]
}
```

When packages/core reaches 1.1.0, all dependent modules maintain compatible lock (import from "@zen-sci/core": "^1.1.0").

### 9.3 Changelog

Maintain CHANGELOG.md in packages/core with entry for each release:

```
## [1.0.0] - 2026-02-18

### Added
- Initial public release
- MarkdownParser, FrontmatterExtractor
- CitationManager, BibTeXParser, CSLRenderer
- SchemaValidator, MathValidator, LinkChecker
- Full TypeScript type definitions

### Fixed
- ...
```

---

## 10. Documentation & Development

### 10.1 README

packages/core/README.md must include:

- Installation instructions
- Quick start example (parse markdown → extract citations)
- API reference (links to JSDoc comments in code)
- Example integration in a module server
- Testing and contributing guidelines

### 10.2 JSDoc Comments

All public methods and interfaces must have JSDoc comments with `@param`, `@returns`, `@throws` tags.

### 10.3 Performance Expectations

- **MarkdownParser:** Parse 10,000 words in <100ms
- **CitationManager:** Resolve 100 citations in <50ms
- **MathValidator:** Validate single expression in <200ms
- **LinkChecker:** Check 50 URLs concurrently in <2s (with network latency)

---

## 11. Future Enhancements

1. **Caching layer:** LRU cache for frequently parsed documents
2. **Plugin system:** Allow custom AST node types
3. **Streaming API:** For large documents (>10MB)
4. **Observability:** Built-in telemetry and logging hooks

---

## Audit Notes (2026-02-18)

**Auditor:** Claude (Opus-level audit)
**Status:** Reviewed — issues identified, fixes recommended

### Issues Found
1. **HIGH-5: `ConversionError` naming conflict** — `ConversionError` is defined as both an interface (error data shape) and a class (throwable). Rename the interface to `ConversionErrorData` to disambiguate.
2. **Completeness: `OutputFormat` union incomplete** — The `OutputFormat` type must include all format strings from all 19 module specs. Complete list: `'latex-pdf' | 'html-blog' | 'beamer-pdf' | 'revealjs-html' | 'newsletter-html' | 'newsletter-mjml' | 'grant-pdf' | 'grant-docx' | 'thesis-pdf' | 'whitepaper-pdf' | 'patent-pdf' | 'epub' | 'mobi' | 'ebook-pdf' | 'documentation-sphinx' | 'documentation-mkdocs' | 'policy-brief' | 'proposal' | 'podcast-notes' | 'pptx' | 'google-slides' | 'resume' | 'cv' | 'paper-pdf'`.
3. **MEDIUM-2: `BibliographyStyle` enum incomplete** — Add `'nature'` and `'arxiv'` styles used by latex-mcp.
4. **ADVISORY-4: Thinking partnership extension point** — Add a `ThinkingPartnership` namespace or interface stub for modules to optionally implement reasoning trace, hypothesis tracking features. Not needed for v0.1 but should be planned.

### Fixed in This Audit
- No direct fixes needed (issues are design decisions requiring Cruz's input).

### Section 3.4 Audit (2026-02-18, follow-up)

**New types reviewed:** ConversionRule, FormatConstraint, ImageProcessingOptions/Result, AccessibilityReport/Violation/Warning, WebMetadataSchema/OG/Twitter/SchemaOrg, DocumentVersion/Change/VersionHistory.

#### Fixed
1. **OpenGraphMetadata.type:** Removed `'blog'` — not a valid Open Graph type. Blog content should use `'article'`.
2. **SchemaOrgData.author:** Changed from `{ '@type': 'Person'; name: string }` to `{ '@type': 'Person' | 'Organization'; name: string }` — Schema.org authors can be organizations.
3. **AccessibilityReport.validatedAt:** Changed from `Date` to `string` (ISO 8601) for consistency with `DocumentVersion.createdAt` and `ResponseMetadata.generatedAt` (both string). All timestamps in packages/core should use ISO 8601 strings, not `Date` objects, for JSON serialization safety.

#### Noted (deferred)
5. **MEDIUM: `ConversionRule.transform` is stringly-typed** — The `transform: string` field references a function by name in a registry. Acceptable for v0.1; document the registry lookup contract explicitly at implementation time.
7. **ADVISORY: `DocumentVersion` / `DocumentVersionHistory` are post-v0.1** — No action needed now.

#### Decision Propagation (Cruz, 2026-02-18)

**HIGH-5 resolved (Cluster 3):** `ConversionError` interface renamed to `ConversionErrorData` (wire-safe serializable shape). `ConversionError` class retained for runtime throw/catch. Added `ConversionError.fromData(data)` static factory and `toData()` instance method to bridge between the two. All type references in `ConversionPipeline.result.error`, `PipelineStage.error`, and `ValidationResult.errors` updated to `ConversionErrorData`.

**MEDIUM-2 resolved (Cluster 2):** `BibliographyStyle` union gained `'nature'` and `'arxiv'` values used by latex-mcp.

**FormatConstraint.aspectRatio resolved (Cluster 2):** Dropped `| string` escape hatch. Now strict enum: `'16:9' | '4:3' | '1:1' | '3:2' | '21:9'`. Add new ratios to this union as needed.

**blog-mcp.SEOMetadata resolved (Cluster 4):** blog-mcp's `SEOMetadata` refactored to extend `WebMetadataSchema`. JSDoc on `WebMetadataSchema` confirmed accurate.
