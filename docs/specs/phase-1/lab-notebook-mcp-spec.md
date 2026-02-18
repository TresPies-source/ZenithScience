# Lab Notebook MCP v0.4: Structured Scientific Reasoning & Reproducible Notebooks

**Author:** ZenSci Architecture (TresPiesDesign.com)
**Status:** Draft
**Created:** 2026-02-18
**Grounded In:** ZenSci semantic clusters analysis, module strategy scout (2026-02-17)

---

## 1. Vision

> Capture raw scientific thinking as structured lab entries with hypothesis, methodology, observations, and reasoning chains—then export as PDF notebooks, markdown archives, or queryable JSON databases for reproducible science.

### The Core Insight

Lab Notebook MCP is ZenSci's **thinking partner differentiator**. While other modules convert polished markdown to publication formats (PDF, HTML, LaTeX), Lab Notebook MCP works upstream—capturing the messiness of science before it's publishable.

Scientists maintain lab notebooks to record experiments: what did I try? What happened? What do I think it means? Traditional digital lab notebooks (LabArchives, Benchling, Notion) couple capture with infrastructure; ZenSci's approach is format-agnostic. A scientist writes entries in plain Markdown (or a simple structured YAML), and Lab Notebook MCP:

1. **Parses entries** into semantic units: hypothesis, methodology, observations, reasoning chain.
2. **Validates** that entries follow scientific rigor (cite prior work, state assumptions, interpret data).
3. **Exports** entries as PDF notebooks (beautiful prints for archival), markdown archives (for git version control), or JSON (for analysis and cross-referencing).
4. **Enables reproducibility** by timestamping entries, recording metadata (equipment used, reagents, environmental conditions), and linking to datasets and citations.

This is where ZenSci becomes a "thinking partner"—not just a formatter. The lab notebook is the scientist's thinking externalized.

### What Makes This Different

Existing lab notebook tools are either too rigid (ELN systems force you into spreadsheets) or too loose (wikis lack structure). ZenSci's approach is **structured but flexible**:

1. **Optional structure.** Entries can be free-form markdown or follow a schema (hypothesis, methodology, observations, conclusion). Both are valid.

2. **Scientific rigor checks.** If you cite something, verify the citation exists. If you use equipment, record serial numbers. If you make a claim, note whether it's speculative or experimental.

3. **Searchable, linkable notebook.** Export JSON with full-text search, cross-references between entries, links to datasets and papers. Your lab notebook becomes a queryable knowledge base.

4. **Reproducibility first.** Every entry is timestamped, attributed, and version-controlled. Running the same experiment months later? Your notebook shows exactly what you did before.

---

## 2. Goals & Success Criteria

### Primary Goals

1. **Accept structured lab entries** (YAML + Markdown) with hypothesis, methodology, observations, reasoning chain, and optional metadata (equipment, reagents, environmental conditions).

2. **Validate scientific rigor** (citations present, assumptions stated, data interpretation supported) and flag missing context with warnings.

3. **Export in multiple formats:**
   - **PDF notebook:** Beautiful, printable lab notebook (like traditional bound notebook) with entry dates, metadata, cross-references.
   - **Markdown archive:** Git-friendly markdown files with YAML frontmatter, one file per entry or one per week.
   - **JSON database:** Fully queryable JSON with cross-references, citations, full-text search metadata.

4. **Enable reproducibility** by timestamping entries, recording equipment/reagent metadata, and auto-generating reproducibility checklists.

5. **Support entry linking** (cross-reference between entries, link to external datasets, bibliography).

6. **Generate summary reports** (daily, weekly, monthly summaries of what was tried and what was learned).

### Success Criteria

- ✅ CLI tool operational: `zen-sci-notebook add entry.yaml` adds entry; `zen-sci-notebook export --format pdf` exports notebook PDF.
- ✅ Entries can be free-form markdown or structured YAML (hypothesis, methodology, observations).
- ✅ Validation warnings for missing context (uncited claims, unspecified equipment, etc.).
- ✅ PDF export produces a beautiful, paginated lab notebook with dates, metadata, cross-references, bibliography.
- ✅ Markdown export produces git-friendly `.md` files with YAML frontmatter and optional index.
- ✅ JSON export is fully queryable with full-text search metadata, cross-reference graph, citation map.
- ✅ Entries are timestamped automatically; can be manually backdated if needed.
- ✅ Equipment metadata tracked (equipment name, serial number, calibration date).
- ✅ Reagent/sample metadata tracked (batch, lot number, expiration date).
- ✅ Environmental metadata (temperature, humidity, location, operator) optional but encouraged.
- ✅ Processing latency < 5 seconds for exporting 100-entry notebook.
- ✅ Unit and integration tests for all export formats.

### Non-Goals (Out of Scope for this version)

- ❌ Real-time collaboration. Lab notebooks are single-author for now; git is the sharing mechanism.
- ❌ Image/attachment embedding. Link to external files (datasets, microscope images) via URLs.
- ❌ Statistical analysis or data visualization. We capture data; analysis is downstream.
- ❌ Instrument integration (directly reading from lab equipment). Out of scope; manual entry only.
- ❌ Regulatory compliance (FDA 21 CFR Part 11, GLP). We provide the tools; compliance is user's responsibility.
- ❌ ELN-like features (workflow templates, approval chains, audit logs). We're a notebook, not an ELN.

---

## 3. Technical Architecture

### 3.1 System Overview

Lab Notebook MCP manages a collection of **entries**—each entry is an experiment, observation, or reasoning session. Entries are stored as individual YAML/Markdown files in a notebook directory. Lab Notebook MCP:

1. **Reads entries** from the filesystem (one file per entry).
2. **Parses and validates** entry structure and content.
3. **Builds a notebook index** (timestamps, citations, cross-references, equipment used).
4. **Exports in multiple formats** (PDF, Markdown, JSON).

```
Notebook directory (entries/)
    entry-2025-01-15.yaml
    entry-2025-01-16.yaml
    entry-2025-01-20.yaml
           ↓
  MCP Tool Request (list entries, add entry, export)
           ↓
  TypeScript Router (servers/lab-notebook-mcp/src/index.ts)
           ├─ Parse entries (frontmatter + body)
           ├─ Validate structure
           ├─ Build index (timestamps, citations, cross-refs)
           └─ Export (PDF, Markdown, JSON)
           ↓
  Python Engine (servers/lab-notebook-mcp/engine)
           ├─ Parse YAML/Markdown entries
           ├─ Validate citations and references
           ├─ Generate notebook PDF (reuses latex-mcp pipeline)
           ├─ Export Markdown archive
           └─ Export JSON database
           ↓
  Output artifacts (notebook.pdf, notebook.json, notebook.zip)
```

### 3.2 MCP Tool Definitions

```typescript
// servers/lab-notebook-mcp/src/index.ts
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: 'add_entry',
      description: 'Add a new lab notebook entry with hypothesis, methodology, observations, and reasoning chain.',
      inputSchema: {
        type: 'object',
        properties: {
          title: {
            type: 'string',
            description: 'Entry title (e.g., "Attempted protein extraction from bacterial culture")'
          },
          date: {
            type: 'string',
            format: 'date',
            description: 'Entry date. Auto-set to today if omitted.'
          },
          hypothesis: {
            type: 'string',
            description: 'What you expected to happen and why'
          },
          methodology: {
            type: 'string',
            description: 'Experimental procedure: materials, steps, conditions'
          },
          observations: {
            type: 'string',
            description: 'What actually happened: measurements, data, outcomes'
          },
          reasoning: {
            type: 'string',
            description: 'Interpretation: what the results mean, limitations, next steps'
          },
          tags: {
            type: 'array',
            items: { type: 'string' },
            description: 'Tags for categorization (e.g., ["protein-extraction", "bacterial-culture"])'
          },
          equipment: {
            type: 'array',
            items: {
              type: 'object',
              properties: {
                name: { type: 'string' },
                serial_number: { type: 'string' },
                calibration_date: { type: 'string', format: 'date' }
              }
            },
            description: 'Equipment used'
          },
          reagents: {
            type: 'array',
            items: {
              type: 'object',
              properties: {
                name: { type: 'string' },
                batch: { type: 'string' },
                expiration_date: { type: 'string', format: 'date' }
              }
            },
            description: 'Reagents and samples used'
          },
          environment: {
            type: 'object',
            properties: {
              temperature: { type: 'string' },
              humidity: { type: 'string' },
              location: { type: 'string' },
              operator: { type: 'string' }
            },
            description: 'Environmental conditions'
          },
          attachments: {
            type: 'array',
            items: { type: 'string' },
            description: 'URLs or file paths to datasets, images, or supplementary data'
          },
          citations: {
            type: 'array',
            items: { type: 'string' },
            description: 'BibTeX citation keys (e.g., ["Smith2020", "Jones2019"])'
          },
          references: {
            type: 'array',
            items: { type: 'string' },
            description: 'Cross-references to other notebook entries (e.g., ["entry-2025-01-10", "entry-2025-01-12"])'
          },
          notebook_path: {
            type: 'string',
            description: 'Path to notebook directory (will be created if not exists)'
          }
        },
        required: ['title', 'notebook_path']
      },
      outputSchema: {
        type: 'object',
        properties: {
          entry_id: { type: 'string', description: 'Unique entry ID (e.g., "entry-2025-01-15")' },
          entry_path: { type: 'string', description: 'Path to saved entry file' },
          warnings: { type: 'array', items: { type: 'string' } },
          success: { type: 'boolean' }
        }
      }
    },
    {
      name: 'list_entries',
      description: 'List all entries in a notebook.',
      inputSchema: {
        type: 'object',
        properties: {
          notebook_path: { type: 'string', description: 'Path to notebook directory' },
          tags: { type: 'array', items: { type: 'string' }, description: 'Filter by tags' },
          date_from: { type: 'string', format: 'date', description: 'Filter by date range' },
          date_to: { type: 'string', format: 'date' }
        },
        required: ['notebook_path']
      },
      outputSchema: {
        type: 'object',
        properties: {
          entries: {
            type: 'array',
            items: {
              type: 'object',
              properties: {
                id: { type: 'string' },
                title: { type: 'string' },
                date: { type: 'string' },
                tags: { type: 'array', items: { type: 'string' } },
                summary: { type: 'string' }
              }
            }
          },
          total: { type: 'integer' }
        }
      }
    },
    {
      name: 'export_notebook',
      description: 'Export notebook to PDF, Markdown, or JSON format.',
      inputSchema: {
        type: 'object',
        properties: {
          notebook_path: { type: 'string', description: 'Path to notebook directory' },
          output_format: {
            type: 'string',
            enum: ['pdf', 'markdown', 'json'],
            description: 'Output format'
          },
          output_path: {
            type: 'string',
            description: 'Path for output file(s)'
          },
          include_metadata: {
            type: 'boolean',
            default: true,
            description: 'Include equipment, reagents, environment metadata in output'
          },
          include_citations: {
            type: 'boolean',
            default: true,
            description: 'Include bibliography in output'
          },
          bibliography: {
            type: 'string',
            description: 'BibTeX file content for citation resolution'
          },
          date_from: {
            type: 'string',
            format: 'date',
            description: 'Filter entries by date range'
          },
          date_to: { type: 'string', format: 'date' },
          tags: {
            type: 'array',
            items: { type: 'string' },
            description: 'Filter entries by tags'
          }
        },
        required: ['notebook_path', 'output_format', 'output_path']
      },
      outputSchema: {
        type: 'object',
        properties: {
          output_path: { type: 'string' },
          format: { type: 'string' },
          entry_count: { type: 'integer' },
          file_size: { type: 'integer' },
          warnings: { type: 'array', items: { type: 'string' } },
          elapsed_ms: { type: 'number' },
          success: { type: 'boolean' }
        }
      }
    },
    {
      name: 'validate_entry',
      description: 'Validate entry structure and scientific rigor.',
      inputSchema: {
        type: 'object',
        properties: {
          title: { type: 'string' },
          hypothesis: { type: 'string' },
          methodology: { type: 'string' },
          observations: { type: 'string' },
          reasoning: { type: 'string' },
          citations: { type: 'array', items: { type: 'string' } },
          equipment: { type: 'array', items: { type: 'object' } }
        },
        required: ['title', 'methodology', 'observations']
      },
      outputSchema: {
        type: 'object',
        properties: {
          valid: { type: 'boolean' },
          rigor_score: { type: 'number', description: 'Scientific rigor score (0-100)' },
          warnings: { type: 'array', items: { type: 'string' } },
          suggestions: { type: 'array', items: { type: 'string' } }
        }
      }
    },
    {
      name: 'generate_summary',
      description: 'Generate summary report of notebook entries (daily, weekly, or monthly).',
      inputSchema: {
        type: 'object',
        properties: {
          notebook_path: { type: 'string' },
          period: { type: 'string', enum: ['day', 'week', 'month'] },
          date: { type: 'string', format: 'date', description: 'Date within the period' }
        },
        required: ['notebook_path', 'period', 'date']
      },
      outputSchema: {
        type: 'object',
        properties: {
          period: { type: 'string' },
          entry_count: { type: 'integer' },
          tags: { type: 'array', items: { type: 'string' } },
          summary: { type: 'string' },
          key_findings: { type: 'array', items: { type: 'string' } },
          next_steps: { type: 'array', items: { type: 'string' } }
        }
      }
    }
  ]
}));
```

### 3.3 Core Integration Points

Lab Notebook MCP extends the ZenSci stack by adding **entry management** and **multiple export formats**:

1. **Reuse from LaTeX MCP:** PDF generation (notebook.pdf uses LaTeX MCP pipeline).
2. **Reuse from packages/core:** Markdown parsing, citation resolution, cross-reference checking.
3. **New abstractions:**
   - `NotebookEntry` interface (structured entry format)
   - `NotebookIndex` interface (index of all entries with cross-references)
   - Export plugins (PDF, Markdown, JSON exporters)

### 3.4 TypeScript Implementation

```typescript
// servers/lab-notebook-mcp/src/index.ts
import { MCPServer } from '@zen-sci/sdk';
import * as fs from 'fs';
import * as path from 'path';
import { v4 as uuidv4 } from 'uuid';

interface NotebookEntry {
  id: string;
  title: string;
  date: string; // YYYY-MM-DD
  hypothesis?: string;
  methodology: string;
  observations: string;
  reasoning?: string;
  tags: string[];
  equipment: Array<{ name: string; serial_number?: string; calibration_date?: string }>;
  reagents: Array<{ name: string; batch?: string; expiration_date?: string }>;
  environment?: { temperature?: string; humidity?: string; location?: string; operator?: string };
  attachments: string[];
  citations: string[];
  references: string[]; // Links to other entry IDs
  created_at: string; // ISO timestamp
  updated_at?: string;
}

interface NotebookIndex {
  entries: NotebookEntry[];
  citations_map: Map<string, string[]>; // citation key → entry IDs that cite it
  references_map: Map<string, string[]>; // entry ID → entry IDs that reference it
  tags: Set<string>;
  date_range: { earliest: string; latest: string };
}

export class LabNotebookMCPServer extends MCPServer {
  constructor() {
    super('lab-notebook-mcp', {
      version: '0.4.0',
      homepage: 'https://github.com/TresPies-source/ZenithScience'
    });
  }

  async initialize() {
    await super.initialize();
    this.registerTools();
  }

  private registerTools() {
    this.server.setRequestHandler(ListToolsRequestSchema, async () => ({
      tools: [
        {
          name: 'add_entry',
          description: 'Add a new lab notebook entry...',
          inputSchema: { /* from 3.2 */ },
          handler: this.handleAddEntry.bind(this)
        },
        {
          name: 'list_entries',
          description: 'List all entries in a notebook...',
          inputSchema: { /* from 3.2 */ },
          handler: this.handleListEntries.bind(this)
        },
        {
          name: 'export_notebook',
          description: 'Export notebook to PDF, Markdown, or JSON...',
          inputSchema: { /* from 3.2 */ },
          handler: this.handleExportNotebook.bind(this)
        },
        {
          name: 'validate_entry',
          description: 'Validate entry structure and scientific rigor...',
          inputSchema: { /* from 3.2 */ },
          handler: this.handleValidateEntry.bind(this)
        },
        {
          name: 'generate_summary',
          description: 'Generate summary report...',
          inputSchema: { /* from 3.2 */ },
          handler: this.handleGenerateSummary.bind(this)
        }
      ]
    }));
  }

  private async handleAddEntry(input: Record<string, any>): Promise<any> {
    try {
      const notebookPath = input.notebook_path;
      const entriesDir = path.join(notebookPath, 'entries');

      // Ensure directories exist
      if (!fs.existsSync(entriesDir)) {
        fs.mkdirSync(entriesDir, { recursive: true });
      }

      // Create entry object
      const entry: NotebookEntry = {
        id: `entry-${input.date || new Date().toISOString().split('T')[0]}`,
        title: input.title,
        date: input.date || new Date().toISOString().split('T')[0],
        hypothesis: input.hypothesis,
        methodology: input.methodology,
        observations: input.observations,
        reasoning: input.reasoning,
        tags: input.tags || [],
        equipment: input.equipment || [],
        reagents: input.reagents || [],
        environment: input.environment,
        attachments: input.attachments || [],
        citations: input.citations || [],
        references: input.references || [],
        created_at: new Date().toISOString(),
        updated_at: new Date().toISOString()
      };

      // Validate entry
      const validation = this.validateEntry(entry);
      const warnings: string[] = [];

      if (!validation.valid) {
        warnings.push(...validation.warnings);
      }

      // Save entry as YAML file
      const entryPath = path.join(entriesDir, `${entry.id}.yaml`);
      const yaml = this._entryToYAML(entry);
      fs.writeFileSync(entryPath, yaml, 'utf-8');

      return {
        entry_id: entry.id,
        entry_path: entryPath,
        warnings,
        success: true
      };
    } catch (error) {
      return {
        error: error instanceof Error ? error.message : String(error),
        success: false
      };
    }
  }

  private async handleListEntries(input: Record<string, any>): Promise<any> {
    try {
      const index = this.buildNotebookIndex(input.notebook_path);
      let entries = index.entries;

      // Filter by tags
      if (input.tags && input.tags.length > 0) {
        entries = entries.filter(e => input.tags.some(t => e.tags.includes(t)));
      }

      // Filter by date range
      if (input.date_from) {
        entries = entries.filter(e => e.date >= input.date_from);
      }
      if (input.date_to) {
        entries = entries.filter(e => e.date <= input.date_to);
      }

      // Sort by date descending
      entries.sort((a, b) => b.date.localeCompare(a.date));

      return {
        entries: entries.map(e => ({
          id: e.id,
          title: e.title,
          date: e.date,
          tags: e.tags,
          summary: (e.observations || '').substring(0, 100) + '...'
        })),
        total: entries.length
      };
    } catch (error) {
      return {
        error: error instanceof Error ? error.message : String(error)
      };
    }
  }

  private async handleExportNotebook(input: Record<string, any>): Promise<any> {
    const startTime = Date.now();
    try {
      const format = input.output_format;
      const outputPath = input.output_path;
      const index = this.buildNotebookIndex(input.notebook_path);

      // Filter entries
      let entries = index.entries;
      if (input.date_from) entries = entries.filter(e => e.date >= input.date_from);
      if (input.date_to) entries = entries.filter(e => e.date <= input.date_to);
      if (input.tags) entries = entries.filter(e => input.tags.some(t => e.tags.includes(t)));

      let output: Buffer | string;
      let actualPath = outputPath;

      if (format === 'pdf') {
        output = await this.exportToPDF(entries, input.bibliography, input.include_metadata);
        fs.writeFileSync(actualPath, output);
      } else if (format === 'markdown') {
        output = this.exportToMarkdown(entries, input.include_metadata);
        // Create directory and write multiple files
        fs.mkdirSync(actualPath, { recursive: true });
        const indexMd = this.generateMarkdownIndex(entries);
        fs.writeFileSync(path.join(actualPath, 'README.md'), indexMd);
        for (const entry of entries) {
          const entryMd = this._entryToMarkdown(entry);
          fs.writeFileSync(path.join(actualPath, `${entry.id}.md`), entryMd);
        }
      } else if (format === 'json') {
        output = JSON.stringify(this.exportToJSON(entries), null, 2);
        fs.writeFileSync(actualPath, output);
      }

      return {
        output_path: actualPath,
        format,
        entry_count: entries.length,
        file_size: typeof output === 'string' ? output.length : output.length,
        warnings: [],
        elapsed_ms: Date.now() - startTime,
        success: true
      };
    } catch (error) {
      return {
        error: error instanceof Error ? error.message : String(error),
        success: false
      };
    }
  }

  private handleValidateEntry(input: Record<string, any>): any {
    const entry: Partial<NotebookEntry> = {
      title: input.title,
      methodology: input.methodology,
      observations: input.observations,
      hypothesis: input.hypothesis,
      reasoning: input.reasoning,
      citations: input.citations || [],
      equipment: input.equipment || []
    };

    const validation = this.validateEntry(entry as NotebookEntry);
    return {
      valid: validation.valid,
      rigor_score: validation.rigor_score,
      warnings: validation.warnings,
      suggestions: validation.suggestions
    };
  }

  private async handleGenerateSummary(input: Record<string, any>): Promise<any> {
    try {
      const index = this.buildNotebookIndex(input.notebook_path);
      const period = input.period; // 'day', 'week', 'month'
      const date = new Date(input.date);

      let startDate: Date, endDate: Date;

      if (period === 'day') {
        startDate = new Date(date);
        endDate = new Date(date);
        endDate.setDate(endDate.getDate() + 1);
      } else if (period === 'week') {
        startDate = new Date(date);
        startDate.setDate(startDate.getDate() - startDate.getDay());
        endDate = new Date(startDate);
        endDate.setDate(endDate.getDate() + 7);
      } else if (period === 'month') {
        startDate = new Date(date.getFullYear(), date.getMonth(), 1);
        endDate = new Date(date.getFullYear(), date.getMonth() + 1, 1);
      }

      const entries = index.entries.filter(
        e => new Date(e.date) >= startDate && new Date(e.date) < endDate
      );

      const tags = new Set<string>();
      const key_findings: string[] = [];
      const next_steps: string[] = [];

      for (const entry of entries) {
        entry.tags.forEach(t => tags.add(t));
        if (entry.reasoning) {
          key_findings.push(`[${entry.date}] ${entry.title}: ${(entry.observations || '').substring(0, 50)}`);
        }
        // Extract next steps from reasoning
        if (entry.reasoning && entry.reasoning.toLowerCase().includes('next')) {
          next_steps.push(entry.reasoning.substring(0, 100));
        }
      }

      return {
        period: `${period} of ${input.date}`,
        entry_count: entries.length,
        tags: Array.from(tags),
        summary: `Completed ${entries.length} entries during this ${period}. Key areas: ${Array.from(tags).join(', ')}.`,
        key_findings,
        next_steps
      };
    } catch (error) {
      return {
        error: error instanceof Error ? error.message : String(error)
      };
    }
  }

  private buildNotebookIndex(notebookPath: string): NotebookIndex {
    const entriesDir = path.join(notebookPath, 'entries');
    const entries: NotebookEntry[] = [];
    const citations_map = new Map<string, string[]>();
    const references_map = new Map<string, string[]>();
    const tags = new Set<string>();
    let earliest = '9999-12-31';
    let latest = '0000-01-01';

    if (!fs.existsSync(entriesDir)) {
      return { entries, citations_map, references_map, tags, date_range: { earliest, latest } };
    }

    const files = fs.readdirSync(entriesDir).filter(f => f.endsWith('.yaml'));

    for (const file of files) {
      const filePath = path.join(entriesDir, file);
      const content = fs.readFileSync(filePath, 'utf-8');
      const entry = this._yamlToEntry(content);

      entries.push(entry);
      entry.tags.forEach(t => tags.add(t));

      // Track citations and references
      entry.citations.forEach(c => {
        if (!citations_map.has(c)) citations_map.set(c, []);
        citations_map.get(c)!.push(entry.id);
      });

      entry.references.forEach(ref => {
        if (!references_map.has(entry.id)) references_map.set(entry.id, []);
        references_map.get(entry.id)!.push(ref);
      });

      if (entry.date < earliest) earliest = entry.date;
      if (entry.date > latest) latest = entry.date;
    }

    return { entries, citations_map, references_map, tags, date_range: { earliest, latest } };
  }

  private validateEntry(entry: NotebookEntry): {
    valid: boolean;
    rigor_score: number;
    warnings: string[];
    suggestions: string[];
  } {
    const warnings: string[] = [];
    let rigor_score = 100;

    // Check required fields
    if (!entry.methodology) {
      warnings.push('Missing methodology: describe what you did');
      rigor_score -= 20;
    }

    if (!entry.observations) {
      warnings.push('Missing observations: describe what happened');
      rigor_score -= 20;
    }

    // Check scientific rigor
    if (!entry.hypothesis) {
      warnings.push('No hypothesis stated: what did you expect to happen?');
      rigor_score -= 10;
    }

    if (!entry.reasoning) {
      warnings.push('No reasoning section: interpret your results');
      rigor_score -= 10;
    }

    if (entry.equipment.length === 0) {
      warnings.push('No equipment recorded: note what instruments were used');
      rigor_score -= 5;
    }

    if (entry.citations.length === 0) {
      warnings.push('No citations: reference prior work to contextualize your experiment');
      rigor_score -= 5;
    }

    const suggestions: string[] = [];
    if (entry.environment && !entry.environment.temperature) {
      suggestions.push('Consider recording temperature and environmental conditions');
    }
    if (entry.reagents.length === 0) {
      suggestions.push('Record reagent batch and expiration dates for reproducibility');
    }

    return {
      valid: warnings.length === 0,
      rigor_score: Math.max(0, rigor_score),
      warnings,
      suggestions
    };
  }

  private async exportToPDF(
    entries: NotebookEntry[],
    bibliography?: string,
    includeMetadata?: boolean
  ): Promise<Buffer> {
    // Generate LaTeX notebook using LaTeX MCP
    const latexContent = this._generateNotebookLaTeX(entries, includeMetadata);

    // Call LaTeX MCP to compile to PDF
    // (delegated to subprocess; return Buffer)
    return Buffer.from('PDF content');
  }

  private exportToMarkdown(entries: NotebookEntry[], includeMetadata?: boolean): string {
    return entries.map(e => this._entryToMarkdown(e)).join('\n---\n');
  }

  private exportToJSON(entries: NotebookEntry[]): any {
    return {
      notebook: {
        entries,
        metadata: {
          total_entries: entries.length,
          date_range: [entries.at(0)?.date, entries.at(-1)?.date],
          tags: Array.from(new Set(entries.flatMap(e => e.tags)))
        }
      }
    };
  }

  private _entryToYAML(entry: NotebookEntry): string {
    return `---
id: ${entry.id}
title: ${entry.title}
date: ${entry.date}
hypothesis: |
  ${entry.hypothesis || ''}
methodology: |
  ${entry.methodology || ''}
observations: |
  ${entry.observations || ''}
reasoning: |
  ${entry.reasoning || ''}
tags: ${JSON.stringify(entry.tags)}
equipment: ${JSON.stringify(entry.equipment, null, 2)}
reagents: ${JSON.stringify(entry.reagents, null, 2)}
citations: ${JSON.stringify(entry.citations)}
references: ${JSON.stringify(entry.references)}
created_at: ${entry.created_at}
---
`;
  }

  private _yamlToEntry(yaml: string): NotebookEntry {
    // Parse YAML frontmatter (simplified; use yaml library in production)
    const match = yaml.match(/^---\n([\s\S]*?)\n---/);
    if (match) {
      const frontmatter = match[1];
      // Parse key: value pairs (simplified)
      const data: Record<string, any> = {};
      for (const line of frontmatter.split('\n')) {
        const [key, ...valueParts] = line.split(':');
        const value = valueParts.join(':').trim();
        data[key.trim()] = value;
      }

      return {
        id: data.id || '',
        title: data.title || '',
        date: data.date || '',
        hypothesis: data.hypothesis,
        methodology: data.methodology,
        observations: data.observations,
        reasoning: data.reasoning,
        tags: data.tags ? JSON.parse(data.tags) : [],
        equipment: data.equipment ? JSON.parse(data.equipment) : [],
        reagents: data.reagents ? JSON.parse(data.reagents) : [],
        attachments: [],
        citations: data.citations ? JSON.parse(data.citations) : [],
        references: data.references ? JSON.parse(data.references) : [],
        created_at: data.created_at || new Date().toISOString()
      };
    }

    return {
      id: '',
      title: '',
      date: '',
      methodology: '',
      observations: '',
      tags: [],
      equipment: [],
      reagents: [],
      attachments: [],
      citations: [],
      references: [],
      created_at: new Date().toISOString()
    };
  }

  private _entryToMarkdown(entry: NotebookEntry): string {
    return `# ${entry.title}

**Date:** ${entry.date}
**Tags:** ${entry.tags.join(', ')}

## Hypothesis
${entry.hypothesis || '(Not stated)'}

## Methodology
${entry.methodology}

## Observations
${entry.observations}

## Reasoning
${entry.reasoning || '(Not provided)'}

## Equipment Used
${entry.equipment.map(e => `- ${e.name}${e.serial_number ? ` (SN: ${e.serial_number})` : ''}`).join('\n')}

## Reagents
${entry.reagents.map(r => `- ${r.name}${r.batch ? ` (Batch: ${r.batch})` : ''}`).join('\n')}

## Citations
${entry.citations.length > 0 ? entry.citations.map(c => `- \\cite{${c}}`).join('\n') : '(None)'}

## Cross-References
${entry.references.length > 0 ? entry.references.map(r => `- [${r}](#${r})`).join('\n') : '(None)'}
`;
  }

  private _generateNotebookLaTeX(entries: NotebookEntry[], includeMetadata?: boolean): string {
    // Generate LaTeX notebook from entries
    return `\\documentclass{book}
\\usepackage[a4paper, margin=1in]{geometry}
\\usepackage{hyperref}
\\usepackage{fancyhdr}
\\title{Lab Notebook}
\\author{ZenSci}
\\begin{document}
\\maketitle
\\tableofcontents
${entries.map(e => `\\chapter{${e.title}}\n\\section*{${e.date}}\n${e.observations}`).join('\n')}
\\end{document}`;
  }

  private generateMarkdownIndex(entries: NotebookEntry[]): string {
    return `# Lab Notebook Index

Total entries: ${entries.length}

## By Date
${entries.sort((a, b) => b.date.localeCompare(a.date)).map(e => `- [${e.date}: ${e.title}](./${e.id}.md)`).join('\n')}

## By Tag
${Array.from(new Set(entries.flatMap(e => e.tags))).map(tag => {
      const tagEntries = entries.filter(e => e.tags.includes(tag));
      return `### ${tag}\n${tagEntries.map(e => `- [${e.title}](./${e.id}.md)`).join('\n')}`;
    }).join('\n\n')}
`;
  }
}

const server = new LabNotebookMCPServer();
server.initialize().then(() => {
  console.log('Lab Notebook MCP server listening...');
});
```

### 3.5 Python Processing Engine

```python
# servers/lab-notebook-mcp/engine/notebook_engine.py
import json
import sys
import yaml
from pathlib import Path
from typing import Dict, Any, List

def parse_entry(yaml_content: str) -> Dict[str, Any]:
    """Parse YAML entry."""
    try:
        return yaml.safe_load(yaml_content)
    except Exception as e:
        raise ValueError(f'Invalid YAML: {e}')

def export_entries_to_pdf(entries: List[Dict[str, Any]], output_path: str) -> None:
    """Generate PDF notebook from entries."""
    # Delegate to LaTeX MCP
    pass

def export_entries_to_json(entries: List[Dict[str, Any]]) -> Dict[str, Any]:
    """Export entries as JSON."""
    return {
        'entries': entries,
        'metadata': {
            'total': len(entries),
            'date_range': [
                min(e['date'] for e in entries) if entries else None,
                max(e['date'] for e in entries) if entries else None
            ]
        }
    }

def main():
    """Entry point for subprocess invocation."""
    if len(sys.argv) < 2:
        print(json.dumps({'error': 'No action specified'}))
        sys.exit(1)

    action = sys.argv[1]
    input_data = json.loads(sys.stdin.read())

    try:
        if action == 'parse_entry':
            result = parse_entry(input_data['yaml_content'])
            print(json.dumps({'entry': result}))
        elif action == 'export_to_json':
            result = export_entries_to_json(input_data['entries'])
            print(json.dumps(result))
        else:
            print(json.dumps({'error': f'Unknown action: {action}'}))
    except Exception as e:
        print(json.dumps({'error': str(e)}))
        sys.exit(1)

if __name__ == '__main__':
    main()
```

### 3.6 Key Schemas

```typescript
// servers/lab-notebook-mcp/src/types.ts
export interface NotebookEntry {
  id: string;
  title: string;
  date: string; // YYYY-MM-DD
  hypothesis?: string;
  methodology: string;
  observations: string;
  reasoning?: string;
  tags: string[];
  equipment: Array<{
    name: string;
    serial_number?: string;
    calibration_date?: string;
  }>;
  reagents: Array<{
    name: string;
    batch?: string;
    expiration_date?: string;
  }>;
  environment?: {
    temperature?: string;
    humidity?: string;
    location?: string;
    operator?: string;
  };
  attachments: string[]; // URLs to external data
  citations: string[]; // BibTeX keys
  references: string[]; // Cross-references to other entries
  created_at: string; // ISO timestamp
  updated_at?: string;
}

export interface NotebookIndex {
  entries: NotebookEntry[];
  citations_map: Map<string, string[]>;
  references_map: Map<string, string[]>;
  tags: Set<string>;
  date_range: { earliest: string; latest: string };
}

export interface EntryValidation {
  valid: boolean;
  rigor_score: number; // 0-100
  warnings: string[];
  suggestions: string[];
}
```

### 3.7 Performance & Constraints

- **Processing latency:** < 5 seconds for exporting 100-entry notebook to PDF, Markdown, or JSON.
- **File size limits:** Notebook directory can contain up to 10,000 entries (total ~1GB).
- **Concurrency:** Multiple servers for horizontal scaling (notebook operations are single-threaded but independent).
- **Memory:** Peak memory ~200MB during PDF generation of large notebooks.

---

## 4. Implementation Plan

### 4.1 Phased Approach

| Phase | Duration | Focus | Deliverables |
|-------|----------|-------|--------------|
| 1 (Setup) | Week 1 | Entry structure, YAML parsing, filesystem | Entry management infrastructure |
| 2 (Export) | Week 1-2 | JSON and Markdown export, indexing | Working exports to JSON and Markdown |
| 3 (PDF & Validation) | Week 2-3 | PDF export (reuse LaTeX MCP), entry validation | Full PDF export and rigor scoring |
| 4 (Polish & Test) | Week 3 | Summary generation, comprehensive testing | Production release |

### 4.2 Week-by-Week Breakdown

**Week 1: Entry Management & Storage**

- [ ] Define `NotebookEntry` interface and YAML schema
- [ ] Implement YAML parsing and serialization (`_entryToYAML`, `_yamlToEntry`)
- [ ] Implement filesystem operations (create notebook directory, save/load entries)
- [ ] Implement `add_entry` MCP tool
- [ ] Implement `list_entries` MCP tool with filtering (by tags, date range)
- [ ] Write unit tests for YAML parsing and entry management

**Week 2: Export & Indexing**

- [ ] Implement `NotebookIndex` builder (cross-references, citations map)
- [ ] Implement `exportToJSON` with full structure
- [ ] Implement `exportToMarkdown` with per-entry files
- [ ] Generate markdown index (by date, by tag)
- [ ] Test JSON export is valid and queryable
- [ ] Test Markdown export is readable and git-friendly

**Week 3: PDF, Validation & Summary**

- [ ] Implement entry validation (`validateEntry`) with rigor scoring
- [ ] Implement `validate_entry` MCP tool
- [ ] Integrate LaTeX MCP for PDF generation (`exportToPDF`)
- [ ] Implement summary generation (daily, weekly, monthly)
- [ ] Implement `generate_summary` MCP tool
- [ ] Write comprehensive test suite (unit, integration, real notebooks)

**Week 4: CLI & Release**

- [ ] Implement CLI tool: `zen-sci-notebook add entry.yaml`
- [ ] Implement CLI tool: `zen-sci-notebook list`
- [ ] Implement CLI tool: `zen-sci-notebook export --format pdf`
- [ ] Test all workflows end-to-end
- [ ] Write user guide and examples
- [ ] Release v0.4 to npm

### 4.3 Dependencies & Prerequisites

**System dependencies:**
- Node.js >= 18, TypeScript >= 5.0
- Python >= 3.11
- LaTeX installation (for PDF export)

**npm dependencies:**
```json
{
  "@modelcontextprotocol/sdk": "^2.0.0",
  "@zen-sci/core": "workspace:*",
  "@zen-sci/sdk": "workspace:*",
  "@zen-sci/latex-mcp": "workspace:*",
  "uuid": "^9.0.0",
  "yaml": "^2.3.0"
}
```

**Python dependencies:**
```txt
pyyaml>=6.0
```

**Internal dependencies:**
- `servers/latex-mcp/` — PDF generation
- `packages/core/` — markdown parsing, citation resolution

### 4.4 Testing Strategy

**Unit tests:**
- YAML serialization/deserialization
- Entry validation and rigor scoring
- Notebook index building (citations, cross-references)
- Date/tag filtering
- Summary generation logic

**Integration tests:**
- Add entry → list entries workflow
- Export to JSON, Markdown, PDF → verify format correctness
- Filter by tags and date → verify results
- Cross-reference resolution

**Real-world tests:**
- Create 10-entry notebook over time
- Export to all formats
- Verify PDF renders correctly
- Verify Markdown is git-friendly

---

## 5. Risk Assessment & Mitigation

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| YAML parsing fails on complex entries | Medium | Medium | Use robust yaml library; provide clear error messages; support plain markdown as fallback. |
| Cross-reference links become stale | Medium | Low | Validate references when building index; warn if unresolved. |
| PDF generation too slow for large notebooks | Low | Medium | Lazy-load entries; batch compilation; cache intermediate results. |
| Filesystem permissions block entry saving | Low | Medium | Check permissions on startup; provide clear error; suggest solutions. |
| Notebook grows too large (10,000+ entries) | Low | Medium | Implement entry archiving; split notebook by date range. |

---

## 6. Rollback & Contingency

**If YAML parsing fails:**
- Fall back to plain markdown with structured comments
- Return error with suggestion to check YAML syntax

**If PDF generation fails:**
- Skip PDF; return Markdown and JSON exports
- Suggest user use LaTeX MCP directly

**If cross-reference resolution fails:**
- Log warnings but don't block export
- List unresolved references in report

---

## 7. Appendices

### 7.1 Related Work

- **Traditional lab notebooks:** Paper-based, good for archival but not searchable.
- **Digital ELNs** (LabArchives, Benchling): Full infrastructure but overkill for individual scientists; high cost.
- **Notion, Obsidian:** General note-taking; not structured for science.
- **GitHub + Markdown:** Version control but no validation, no export.

Lab Notebook MCP combines git-friendliness (Markdown storage) with scientific structure (validation, metadata).

### 7.2 Future Considerations

- **Instrument integration:** Read data directly from lab equipment (balance, spectrophotometer, etc.)
- **Statistical summaries:** Auto-generate charts, statistics from data entries
- **Collaboration:** Multi-author notebooks with approval workflows
- **GLP compliance:** Audit logs, approval chains, regulatory signatures
- **Data visualization:** Plot datasets referenced in entries
- **Automated calculations:** Extract data from entries, compute statistics
- **Templates:** Entry templates for repeated experiments (e.g., "protein extraction protocol")
- **Mobile app:** Capture entries on phone during wet lab work

### 7.3 Open Questions

1. Should entries be stored as individual files or in a single database? Individual files (more git-friendly) for v0.4.
2. Should we enforce a specific entry structure (hypothesis/methodology/observations) or allow free-form? Support both (structured optional).
3. How do we handle very large notebooks (1,000+ entries)? Lazy-load; archive old entries.
4. Should PDF export group entries by time period (daily, weekly, monthly)? Group by week for v0.4.

### 7.4 Schema Backlog

Schemas defined in `servers/lab-notebook-mcp/src/types.ts`:

- `NotebookEntry` (module-specific)
- `NotebookIndex` (module-specific)
- `EntryValidation` (module-specific)

---

## Audit Notes (2026-02-18)

**Auditor:** Claude (Opus-level audit)
**Status:** Reviewed — fixes applied

### Fixed in This Audit
- **CRITICAL-1:** Package.json dependency corrected from `@anthropic-ai/sdk` to `@modelcontextprotocol/sdk`.
- **HIGH-3:** Python version standardized to >= 3.11.

### Remaining Issues
- Imports `MCPServer` from `@zen-sci/sdk` — should be renamed to `ZenSciServer` when SDK spec finalized.
- **Strategic note:** This is the flagship "thinking partner" module. Its reasoning traces, hypothesis tracking, and proof structures are unique differentiators. Ensure these features are reflected in `packages-core-spec.md` as a `ThinkingPartnership` extension point.

#### Decision Propagation (Cruz, 2026-02-18)
- **CRITICAL-2 resolved:** `ZenSciServer` abstract base class deprecated. lab-notebook-mcp's `MCPServer` extends pattern should be replaced with `createZenSciServer()` factory + `McpServer.registerTool()` with Zod schemas at implementation time. No rename needed — composition replaces inheritance.
