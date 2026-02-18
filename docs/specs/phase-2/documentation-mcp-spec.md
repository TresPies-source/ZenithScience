# documentation-mcp Spec: Code-Aware API Documentation

**Author:** ZenSci Architecture (TresPiesDesign.com)
**Status:** Draft
**Created:** 2026-02-18
**Phase:** 2 (ships after Phase 1 core is validated)
**Grounded In:** Module strategy scout (2026-02-17), ZenSci semantic clusters

---

## 1. Vision
> Automatically extract API signatures from Python and TypeScript code; generate Sphinx-compatible HTML + MkDocs markdown; integrate live examples from Claude Code notebooks.

### The Core Insight
Developers maintain two parallel documents: source code (truth) and documentation (increasingly stale). documentation-mcp unifies these: **extract function/class signatures directly from code**, generate structured Markdown/Sphinx from them, and optionally embed executable examples from Claude Code notebooks. Changes to code automatically propagate to docs via a single conversion pass.

Unlike generic doc generators (pdoc, typedoc), documentation-mcp understands **research context** (mathematical formulas in docstrings, citations, data provenance) and integrates with ZenSci's bibliography system. It also bridges Python (pandoc, SymPy) and TypeScript (MCP SDK) codebases in a single documentation build.

### What Makes This Different
1. **Dual-language extraction:** Python AST parsing + TypeScript AST parsing in one tool; unified output format.
2. **Markdown-first generation:** Output is Sphinx-compatible .rst or MkDocs-compatible Markdown (user's choice).
3. **Example-aware:** Links to executable Claude Code notebooks; optionally embeds `@example` blocks from code comments.
4. **Mathematical documentation:** Preserve LaTeX math in docstrings (e.g., `:math:\`\int_0^\infty e^{-x^2} dx\`` in Python docstrings).
5. **Citation integration:** Extract citations from docstrings; auto-generate bibliography at end of docs.
6. **Inheritance + polymorphism visualization:** Auto-diagram class hierarchies and trait implementations.

---

## 2. Goals & Success Criteria

### Primary Goals
1. Extract function, class, and method signatures from Python (via AST) and TypeScript (via ts-morph).
2. Generate Sphinx (.rst) and MkDocs (Markdown) documentation from extracted signatures + docstrings.
3. Support mathematical notation in docstrings (LaTeX, SymPy expressions).
4. Parse and integrate citations from docstrings; auto-generate bibliography.
5. Auto-generate class hierarchy diagrams and API index tables.
6. Link to executable Claude Code notebooks for examples.

### Success Criteria
✅ Extract 100+ functions/classes from zen-sci/packages/core (Python + TypeScript) in <5s.
✅ Generated Sphinx documentation renders without warnings; HTML output valid.
✅ MkDocs output (Markdown) is valid and navigable; sidebar auto-generated.
✅ Math formulas in docstrings render correctly in both HTML and Markdown.
✅ Bibliography auto-generated from docstring citations; CSL-JSON integration works.
✅ Class hierarchy diagrams are accurate and visually clear (Mermaid or similar).
✅ End-to-end test: Extract zen-sci/packages/core → Sphinx + MkDocs docs in <10s; docs build successfully.

### Non-Goals
❌ Real-time doc generation as user edits code (watch mode is out-of-scope for Phase 2).
❌ Interactive API explorer (frontend tool, separate from MCP).
❌ Automatic test coverage integration (separate build step).
❌ Multi-language support beyond Python + TypeScript.

---

## 3. Technical Architecture

### 3.1 System Overview

```
Source Code (Python AST + TypeScript AST)
    ↓
documentation-mcp MCP Server (TypeScript)
    ├─ Python AST parser (ast module)
    ├─ TypeScript AST parser (ts-morph)
    ├─ Docstring extractor (parse math, citations, examples)
    ├─ Documentation generator (Sphinx .rst or MkDocs Markdown)
    └─ Return DocumentResponse (HTML or Markdown artifacts)
    ↓
Output: Sphinx documentation + MkDocs + bibliography + diagrams
```

### 3.2 MCP Tool Definitions

#### Tool: `extract_api`
**Description:** Extract API signatures from Python and/or TypeScript code.

**Input Schema:**
```json
{
  "type": "object",
  "properties": {
    "source_paths": {
      "type": "array",
      "items": { "type": "string" },
      "description": "File paths to scan (e.g., ['/path/to/core.py', '/path/to/index.ts'])"
    },
    "language": {
      "type": "string",
      "enum": ["python", "typescript", "auto"],
      "description": "Language; auto-detect from file extension if 'auto'."
    },
    "include_private": {
      "type": "boolean",
      "description": "Include functions/classes starting with _ (private)."
    },
    "extract_examples": {
      "type": "boolean",
      "description": "Extract @example code blocks from docstrings."
    }
  },
  "required": ["source_paths"]
}
```

**Output Schema:**
```json
{
  "type": "object",
  "properties": {
    "id": { "type": "string" },
    "api_items": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "type": { "type": "string", "enum": ["function", "class", "method", "interface", "type"] },
          "name": { "type": "string" },
          "fully_qualified_name": { "type": "string" },
          "signature": { "type": "string" },
          "docstring": { "type": "string" },
          "parameters": {
            "type": "array",
            "items": {
              "type": "object",
              "properties": {
                "name": { "type": "string" },
                "type": { "type": "string" },
                "description": { "type": "string" },
                "default": { "type": ["string", "null"] }
              }
            }
          },
          "returns": {
            "type": "object",
            "properties": {
              "type": { "type": "string" },
              "description": { "type": "string" }
            }
          },
          "decorators": { "type": "array", "items": { "type": "string" } },
          "examples": { "type": "array", "items": { "type": "string" } },
          "citations": { "type": "array", "items": { "type": "string" } },
          "location": { "type": "object", "properties": { "file": { "type": "string" }, "line": { "type": "integer" } } }
        }
      }
    },
    "metadata": {
      "type": "object",
      "properties": {
        "total_items": { "type": "integer" },
        "by_type": { "type": "object" },
        "files_scanned": { "type": "integer" },
        "languages": { "type": "array", "items": { "type": "string" } }
      }
    },
    "elapsed": { "type": "number" }
  }
}
```

#### Tool: `generate_documentation`
**Description:** Generate Sphinx or MkDocs documentation from extracted API.

**Input Schema:**
```json
{
  "type": "object",
  "properties": {
    "api_items": {
      "type": "array",
      "description": "API items from extract_api tool."
    },
    "format": {
      "type": "string",
      "enum": ["sphinx", "mkdocs"],
      "description": "Output format."
    },
    "title": { "type": "string", "description": "Documentation title." },
    "version": { "type": "string", "description": "Package version." },
    "include_diagrams": { "type": "boolean", "description": "Generate class hierarchy diagrams." },
    "include_index": { "type": "boolean", "description": "Generate API index table." },
    "bibliography": { "type": "string", "description": "CSL-JSON bibliography." },
    "toc_depth": { "type": "integer", "description": "Table of contents depth (default: 2)." }
  },
  "required": ["api_items", "format", "title"]
}
```

**Output Schema:**
```json
{
  "type": "object",
  "properties": {
    "id": { "type": "string" },
    "format": { "type": "string" },
    "content": { "type": "string", "description": "Base64-encoded .tar.gz with docs files." },
    "artifacts": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "type": { "type": "string", "enum": ["index", "api_reference", "diagrams", "bibliography", "toc"] },
          "filename": { "type": "string" },
          "content": { "type": "string" }
        }
      }
    },
    "metadata": {
      "type": "object",
      "properties": {
        "documentation_files": { "type": "integer" },
        "total_pages": { "type": "integer" },
        "api_coverage": { "type": "number", "description": "% of code with docstrings." }
      }
    },
    "elapsed": { "type": "number" }
  }
}
```

#### Tool: `validate_documentation`
**Description:** Check documentation for completeness, broken links, unresolved references.

**Input Schema:**
```json
{
  "type": "object",
  "properties": {
    "documentation_path": { "type": "string", "description": "Path to built documentation (HTML or Markdown)." },
    "strict_mode": { "type": "boolean", "description": "Fail on warnings." }
  },
  "required": ["documentation_path"]
}
```

**Output Schema:**
```json
{
  "type": "object",
  "properties": {
    "valid": { "type": "boolean" },
    "errors": { "type": "array", "items": { "type": "object" } },
    "warnings": { "type": "array", "items": { "type": "object" } },
    "report": {
      "type": "object",
      "properties": {
        "total_pages": { "type": "integer" },
        "broken_links": { "type": "integer" },
        "undocumented_items": { "type": "integer" },
        "unresolved_citations": { "type": "integer" }
      }
    }
  }
}
```

### 3.3 Core Integration Points

**packages/core DocumentRequest/DocumentResponse:**
- Input: API extraction request (source paths, language, options).
- Output: DocumentResponse with documentation artifacts (HTML, Markdown, diagrams).
- Bibliography system: Integrate with packages/core's unified CSL handler for auto-generating bibliography from docstring citations.

---

### 3.4 TypeScript Implementation

**MCP Server Registration:**
```typescript
// zen-sci/packages/sdk/src/servers/documentation-mcp/index.ts

import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from '@modelcontextprotocol/sdk/types.js';
import { extractAPI } from './extractors/index.js';
import { generateDocumentation } from './generators/index.js';

const server = new Server(
  { name: 'documentation-mcp', version: '0.1.0' },
  { capabilities: { tools: {} } }
);

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: 'extract_api',
      description: 'Extract API signatures from Python and/or TypeScript code.',
      inputSchema: {
        type: 'object' as const,
        properties: {
          source_paths: {
            type: 'array',
            items: { type: 'string' },
          },
          language: {
            type: 'string',
            enum: ['python', 'typescript', 'auto'],
          },
          include_private: { type: 'boolean' },
          extract_examples: { type: 'boolean' },
        },
        required: ['source_paths'],
      },
    },
    {
      name: 'generate_documentation',
      description: 'Generate Sphinx or MkDocs documentation from API items.',
      inputSchema: {
        type: 'object' as const,
        properties: {
          api_items: { type: 'array' },
          format: { type: 'string', enum: ['sphinx', 'mkdocs'] },
          title: { type: 'string' },
          version: { type: 'string' },
          include_diagrams: { type: 'boolean' },
          include_index: { type: 'boolean' },
          bibliography: { type: 'string' },
          toc_depth: { type: 'integer' },
        },
        required: ['api_items', 'format', 'title'],
      },
    },
    {
      name: 'validate_documentation',
      description: 'Validate documentation for completeness and broken links.',
      inputSchema: {
        type: 'object' as const,
        properties: {
          documentation_path: { type: 'string' },
          strict_mode: { type: 'boolean' },
        },
        required: ['documentation_path'],
      },
    },
  ],
}));

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request;

  if (name === 'extract_api') {
    const result = await extractAPI(args as any);
    return { content: [{ type: 'text', text: JSON.stringify(result) }] };
  }

  if (name === 'generate_documentation') {
    const result = await generateDocumentation(args as any);
    return { content: [{ type: 'text', text: JSON.stringify(result) }] };
  }

  if (name === 'validate_documentation') {
    const result = await validateDocumentation(args as any);
    return { content: [{ type: 'text', text: JSON.stringify(result) }] };
  }

  throw new Error(`Unknown tool: ${name}`);
});

async function validateDocumentation(args: any) {
  return await callPythonEngine('validate_docs', args);
}

const transport = new StdioServerTransport();
server.connect(transport);
```

**Core Type Definitions:**
```typescript
// zen-sci/packages/core/src/schemas/documentation.ts

export interface APIItem {
  type: 'function' | 'class' | 'method' | 'interface' | 'type';
  name: string;
  fully_qualified_name: string;
  signature: string;
  docstring?: string;
  parameters: Parameter[];
  returns?: ReturnInfo;
  decorators?: string[];
  examples?: string[];
  citations?: string[]; // Citation keys (@cite ref)
  location: {
    file: string;
    line: number;
    endLine?: number;
  };
}

export interface Parameter {
  name: string;
  type: string;
  description?: string;
  default?: string;
  optional?: boolean;
}

export interface ReturnInfo {
  type: string;
  description?: string;
}

export interface DocumentationRequest extends DocumentRequest {
  format: 'documentation-sphinx' | 'documentation-mkdocs';
  source_paths: string[];
  language: 'python' | 'typescript' | 'auto';
  options?: DocumentationOptions;
}

export interface DocumentationOptions {
  include_private?: boolean;
  extract_examples?: boolean;
  include_diagrams?: boolean;
  include_index?: boolean;
  toc_depth?: number;
  bibliography?: string;
}

export interface DocumentationResponse extends DocumentResponse {
  format: 'documentation-sphinx' | 'documentation-mkdocs';
  content: Buffer; // .tar.gz with built docs
  artifacts: DocumentationArtifact[];
  metadata: DocumentationMetadata;
}

export interface DocumentationArtifact extends Artifact {
  type: 'index' | 'api_reference' | 'diagrams' | 'bibliography' | 'toc';
}

export interface DocumentationMetadata extends ResponseMetadata {
  documentation_files: number;
  total_pages: number;
  api_coverage: number; // % with docstrings
  languages: string[];
}
```

---

### 3.5 Python Processing Engine

**Python AST Extraction:**
```python
# zen-sci/servers/documentation-mcp/processing/python_extractor.py

import ast
import inspect
import re
from pathlib import Path
from typing import Dict, List, Any, Optional, Tuple
from dataclasses import dataclass, asdict

@dataclass
class DocstringMeta:
    """Parsed docstring metadata (numpy/Google style)."""
    summary: str
    description: str
    parameters: Dict[str, Tuple[str, str]]  # {param_name: (type, description)}
    returns: Tuple[str, str]  # (type, description)
    raises: List[Tuple[str, str]]  # [(exception_name, description)]
    examples: List[str]
    citations: List[str]  # Citation keys extracted from text

def extract_python_api(source_path: str, include_private: bool = False) -> List[Dict[str, Any]]:
    """
    Extract API items (functions, classes, methods) from Python file via AST.

    Args:
        source_path: Path to .py file.
        include_private: Include names starting with _.

    Returns:
        List of API item dicts.
    """

    tree = ast.parse(Path(source_path).read_text())
    items = []

    for node in ast.walk(tree):
        if isinstance(node, ast.FunctionDef) or isinstance(node, ast.AsyncFunctionDef):
            if not include_private and node.name.startswith('_'):
                continue
            item = extract_function(node, source_path)
            items.append(item)

        elif isinstance(node, ast.ClassDef):
            if not include_private and node.name.startswith('_'):
                continue
            item = extract_class(node, source_path)
            items.append(item)

            # Extract methods
            for inner_node in node.body:
                if isinstance(inner_node, ast.FunctionDef):
                    if not include_private and inner_node.name.startswith('_'):
                        continue
                    method_item = extract_method(inner_node, node.name, source_path)
                    items.append(method_item)

    return items


def extract_function(node: ast.FunctionDef, source_path: str) -> Dict[str, Any]:
    """Extract function signature and docstring."""

    args = node.args
    parameters = []

    # Extract parameters
    for arg in args.args:
        param = {
            'name': arg.arg,
            'type': ast.unparse(arg.annotation) if arg.annotation else 'Any',
            'description': '',
            'default': None,
        }
        parameters.append(param)

    # Extract defaults
    defaults_offset = len(args.args) - len(args.defaults)
    for i, default in enumerate(args.defaults):
        parameters[defaults_offset + i]['default'] = ast.unparse(default)

    # Extract docstring and metadata
    docstring = ast.get_docstring(node) or ''
    docstring_meta = parse_docstring(docstring)

    # Merge parameter descriptions from docstring
    for param in parameters:
        if param['name'] in docstring_meta.parameters:
            _, desc = docstring_meta.parameters[param['name']]
            param['description'] = desc

    # Return type
    return_type = ast.unparse(node.returns) if node.returns else 'Any'

    item = {
        'type': 'function',
        'name': node.name,
        'fully_qualified_name': f'{Path(source_path).stem}.{node.name}',
        'signature': f"def {node.name}({', '.join(p['name'] + ': ' + p['type'] for p in parameters)}) -> {return_type}",
        'docstring': docstring,
        'parameters': parameters,
        'returns': {
            'type': return_type,
            'description': docstring_meta.returns[1] if docstring_meta.returns else '',
        },
        'examples': docstring_meta.examples,
        'citations': docstring_meta.citations,
        'location': {
            'file': source_path,
            'line': node.lineno,
            'endLine': node.end_lineno,
        },
    }

    return item


def extract_class(node: ast.ClassDef, source_path: str) -> Dict[str, Any]:
    """Extract class signature and docstring."""

    docstring = ast.get_docstring(node) or ''
    docstring_meta = parse_docstring(docstring)

    # Extract base classes
    bases = [ast.unparse(base) for base in node.bases]

    item = {
        'type': 'class',
        'name': node.name,
        'fully_qualified_name': f'{Path(source_path).stem}.{node.name}',
        'signature': f"class {node.name}({', '.join(bases)})",
        'docstring': docstring,
        'parameters': [],  # Class init parameters extracted separately
        'bases': bases,
        'examples': docstring_meta.examples,
        'citations': docstring_meta.citations,
        'location': {
            'file': source_path,
            'line': node.lineno,
            'endLine': node.end_lineno,
        },
    }

    return item


def parse_docstring(docstring: str) -> DocstringMeta:
    """
    Parse numpy-style or Google-style docstring.

    Example (Google style):
        Args:
            x (float): Input value.

        Returns:
            float: Output value.

        Examples:
            >>> func(1.0)
            2.0
    """

    # Simplified parsing; real implementation uses docstring parsers
    summary = docstring.split('\n')[0] if docstring else ''

    parameters = {}
    examples = []
    citations = []
    returns = ('Any', '')

    # Extract examples (between ::: example ... ::: or @example blocks)
    example_pattern = r'(?:::\ example\s+|\@example\s+)(.*?)(?=(?:::|\@|\Z))'
    for match in re.finditer(example_pattern, docstring, re.DOTALL):
        examples.append(match.group(1).strip())

    # Extract citations ([@key] or [@Author2023] patterns)
    citation_pattern = r'\[@([a-zA-Z0-9_]+)\]'
    citations = re.findall(citation_pattern, docstring)

    return DocstringMeta(
        summary=summary,
        description=docstring,
        parameters=parameters,
        returns=returns,
        raises=[],
        examples=examples,
        citations=citations,
    )


def extract_method(
    node: ast.FunctionDef, class_name: str, source_path: str
) -> Dict[str, Any]:
    """Extract method (similar to function)."""
    item = extract_function(node, source_path)
    item['type'] = 'method'
    item['fully_qualified_name'] = f'{Path(source_path).stem}.{class_name}.{node.name}'
    return item
```

**TypeScript AST Extraction (ts-morph):**
```python
# zen-sci/servers/documentation-mcp/processing/typescript_extractor.py

import subprocess
import json
from pathlib import Path
from typing import Dict, List, Any

def extract_typescript_api(source_path: str, include_private: bool = False) -> List[Dict[str, Any]]:
    """
    Extract API items from TypeScript file via ts-morph.

    Args:
        source_path: Path to .ts file.
        include_private: Include private members.

    Returns:
        List of API item dicts.
    """

    # Call ts-morph via Node.js subprocess
    # (or use py-ts-morph if available)

    result = subprocess.run(
        [
            'node',
            '-e',
            f"""
const Project = require('ts-morph').Project;
const project = new Project();
project.addSourceFileAtPath('{source_path}');
const sourceFile = project.getSourceFileOrThrow('{source_path}');

// Extract functions
const functions = sourceFile.getFunctions();
const items = [];

for (const func of functions) {{
    items.push({{
        type: 'function',
        name: func.getName(),
        signature: func.getText().split('{{'')[0].trim(),
        parameters: func.getParameters().map(p => ({{
            name: p.getName(),
            type: p.getType().getText(),
            description: '',
        }})),
        docstring: (func.getJsDocs()[0]?.getText() || ''),
    }});
}}

// Extract classes and interfaces
const classes = sourceFile.getClasses();
for (const cls of classes) {{
    items.push({{
        type: 'class',
        name: cls.getName(),
        signature: cls.getText().split('{{'')[0].trim(),
        docstring: (cls.getJsDocs()[0]?.getText() || ''),
    }});
}}

const interfaces = sourceFile.getInterfaces();
for (const iface of interfaces) {{
    items.push({{
        type: 'interface',
        name: iface.getName(),
        signature: iface.getText().split('{{'')[0].trim(),
        docstring: (iface.getJsDocs()[0]?.getText() || ''),
    }});
}}

console.log(JSON.stringify(items, null, 2));
""",
        ],
        capture_output=True,
        text=True,
    )

    if result.returncode != 0:
        raise RuntimeError(f'ts-morph extraction failed: {result.stderr}')

    items = json.loads(result.stdout)
    return items
```

**Documentation Generation (Sphinx/MkDocs):**
```python
# zen-sci/servers/documentation-mcp/processing/doc_generator.py

from typing import List, Dict, Any
from pathlib import Path
import json

def generate_sphinx_documentation(
    api_items: List[Dict[str, Any]],
    title: str,
    version: str,
    include_diagrams: bool = True,
) -> Tuple[str, Dict[str, str]]:
    """
    Generate Sphinx .rst documentation from API items.

    Returns:
        Tuple[tar.gz content, artifacts dict].
    """

    rst_lines = []

    # Title
    rst_lines.append('=' * len(title))
    rst_lines.append(title)
    rst_lines.append('=' * len(title))
    rst_lines.append('')

    rst_lines.append(f'Version: {version}')
    rst_lines.append('')

    # Table of contents
    rst_lines.append('.. toctree::')
    rst_lines.append('   :maxdepth: 2')
    rst_lines.append('')
    rst_lines.append('   api/reference')
    rst_lines.append('')

    # API Reference
    rst_lines.append('API Reference')
    rst_lines.append('-' * len('API Reference'))
    rst_lines.append('')

    # Group by type
    by_type = {}
    for item in api_items:
        item_type = item['type']
        if item_type not in by_type:
            by_type[item_type] = []
        by_type[item_type].append(item)

    for item_type in ['class', 'function', 'method', 'interface']:
        if item_type not in by_type:
            continue

        rst_lines.append(item_type.capitalize() + 's')
        rst_lines.append('~' * len(item_type) + '~' * 1)
        rst_lines.append('')

        for item in by_type[item_type]:
            rst_lines.append(f".. py:{item_type}:: {item['name']}")
            rst_lines.append('')
            if item.get('docstring'):
                rst_lines.append(f"   {item['docstring']}")
            rst_lines.append('')

    # Bibliography
    rst_lines.append('Bibliography')
    rst_lines.append('-' * len('Bibliography'))
    rst_lines.append('')
    rst_lines.append('.. bibliography::')
    rst_lines.append('')

    rst_content = '\n'.join(rst_lines)

    # Package into .tar.gz
    import tarfile
    import io

    tar_buffer = io.BytesIO()
    with tarfile.open(fileobj=tar_buffer, mode='w:gz') as tar:
        # Add index.rst
        index_info = tarfile.TarInfo(name='index.rst')
        index_info.size = len(rst_content.encode('utf-8'))
        tar.addfile(index_info, io.BytesIO(rst_content.encode('utf-8')))

    tar_buffer.seek(0)
    return tar_buffer.getvalue(), {'index': rst_content}


def generate_mkdocs_documentation(
    api_items: List[Dict[str, Any]],
    title: str,
    version: str,
    include_diagrams: bool = True,
) -> Tuple[str, Dict[str, str]]:
    """Generate MkDocs Markdown documentation from API items."""

    md_lines = []
    md_lines.append(f'# {title}')
    md_lines.append(f'\n**Version:** {version}\n')

    md_lines.append('## API Reference\n')

    by_type = {}
    for item in api_items:
        item_type = item['type']
        if item_type not in by_type:
            by_type[item_type] = []
        by_type[item_type].append(item)

    for item_type in ['class', 'function', 'method', 'interface']:
        if item_type not in by_type:
            continue

        md_lines.append(f'### {item_type.capitalize()}s\n')

        for item in by_type[item_type]:
            md_lines.append(f"#### `{item['name']}`\n")
            md_lines.append(f"```{item_type.lower()}\n{item.get('signature', '')}\n```\n")
            if item.get('docstring'):
                md_lines.append(f"{item['docstring']}\n")
            md_lines.append('')

    md_content = '\n'.join(md_lines)
    return md_content, {'index': md_content}
```

---

### 3.6 Key Schemas

```typescript
// zen-sci/packages/core/src/schemas/documentation-extended.ts

export interface APIIndex {
  total_items: number;
  by_type: Record<string, number>;
  files_scanned: number;
  languages: string[];
  api_coverage: number; // % with docstrings
}

export interface ClassHierarchy {
  root_classes: string[];
  inheritance_graph: Record<string, string[]>; // {parent: [children]}
  interfaces_implemented: Record<string, string[]>; // {class: [interfaces]}
}

export interface DocumentationConfig {
  project_title: string;
  project_version: string;
  project_description?: string;
  source_dir: string;
  output_format: 'sphinx' | 'mkdocs';
  theme?: string;
  include_private: boolean;
  include_examples: boolean;
  include_diagrams: boolean;
  auto_index: boolean;
  toc_depth: number;
  highlight_language: 'python' | 'typescript' | 'auto';
}
```

---

### 3.7 Phase 2 Infrastructure Requirements

**New packages/core capabilities needed:**

1. **API extraction abstraction:** Interface for language-specific AST extraction (Python, TypeScript, future: Go, Rust).
   - *Recommendation:* Create `packages/core/api-extractors/` with pluggable extractors.

2. **Docstring parsing module:** Unified parser for multiple docstring styles (NumPy, Google, Sphinx, JSDoc).
   - *Recommendation:* Create `packages/core/docstring-parsers/` with configurable parsers.

3. **Diagram generation:** Mermaid or Graphviz integration for class hierarchies, dependency graphs.
   - *Recommendation:* Create `packages/core/diagram-generators/` for Mermaid syntax generation.

4. **Bibliography integration:** CSL-JSON bibliography from docstring citations.
   - *Dependency:* Leverage existing packages/core bibliography system; extract citation keys from docstrings.

---

### 3.8 Performance & Constraints

| Constraint | Target | Notes |
|-----------|--------|-------|
| Extract API from zen-sci/packages/core (100+ items) | <5s | AST parsing; Python ast module + ts-morph |
| Generate Sphinx docs (100+ items) | <3s | Template rendering; .rst serialization |
| Generate MkDocs docs (100+ items) | <2s | Markdown generation is faster than Sphinx |
| Validate generated docs | <5s | Link checking; unresolved reference detection |
| Memory usage | <300 MB | Single codebase; AST in memory |
| Output file size (Sphinx .tar.gz) | <2 MB | Compressed; text-heavy docs |
| Output file size (MkDocs .tar.gz) | <1 MB | Markdown is compact |

---

## 4. Implementation Plan

### 4.1 Phased Approach

| Phase | Timeline | Deliverables | Dependencies |
|-------|----------|--------------|--------------|
| **Alpha (Week 1-2)** | 2 weeks | Python AST extraction only; basic docstring parsing; MkDocs Markdown generation | packages/core v1.0 |
| **Beta (Week 3-4)** | 2 weeks | + TypeScript extraction (ts-morph); Sphinx .rst generation; class hierarchy diagrams | All Alpha |
| **Candidate (Week 5-6)** | 2 weeks | + Bibliography integration; validation tool; all docstring styles (NumPy, Google, Sphinx, JSDoc) | All Beta + funder rules infra |
| **Release (Week 7)** | 1 week | Full test suite; documentation; example generation | All Candidate |

### 4.2 Week-by-Week Breakdown

**Week 1: Python AST Extraction & MkDocs Generation**
- [ ] Set up zen-sci/servers/documentation-mcp/ directory structure
- [ ] Implement MCP server skeleton (TypeScript, registration)
- [ ] Build Python AST parser (functions, classes, methods)
- [ ] Implement docstring extractor (basic NumPy/Google style)
- [ ] Create MkDocs Markdown generator
- [ ] Implement `extract_api` tool (Python only)
- [ ] Unit tests: AST parsing; docstring extraction; Markdown generation
- [ ] Integration test: Extract zen-sci/packages/core → MkDocs Markdown

**Week 2: Docstring Parsing & Example Extraction**
- [ ] Implement comprehensive docstring parser (NumPy, Google, Sphinx styles)
- [ ] Add example block extraction (@example, ::: example)
- [ ] Add citation key extraction ([@key] patterns)
- [ ] Implement parameter description merging
- [ ] Unit tests: docstring parsing; example extraction; citation detection
- [ ] Integration test: Full extraction with examples and citations

**Week 3: TypeScript Extraction & Sphinx Generation**
- [ ] Implement TypeScript extractor via ts-morph
- [ ] Add interface and type alias extraction
- [ ] Create Sphinx .rst generator
- [ ] Add Sphinx-specific features (cross-refs, directives)
- [ ] Unit tests: TypeScript parsing; Sphinx .rst generation
- [ ] Integration test: Extract mixed Python/TypeScript codebase → Sphinx docs

**Week 4: Class Hierarchies & Diagrams**
- [ ] Implement class hierarchy extraction (inheritance, interfaces)
- [ ] Add Mermaid diagram generation (class diagrams)
- [ ] Implement inheritance relationship mapping
- [ ] Add diagram artifact generation
- [ ] Unit tests: hierarchy extraction; Mermaid syntax
- [ ] Integration test: Complex class hierarchy → generated diagrams

**Week 5: Bibliography & JSDoc Support**
- [ ] Integrate with packages/core bibliography system
- [ ] Implement CSL-JSON generation from docstring citations
- [ ] Add JSDoc parser (TypeScript/JavaScript doc standard)
- [ ] Add automatic .bib file generation
- [ ] Unit tests: JSDoc parsing; CSL generation; citation resolution
- [ ] Integration test: Docs with 20+ citations → auto-generated bibliography

**Week 6: Validation & Completeness Checking**
- [ ] Implement `validate_documentation` tool
- [ ] Add link validation (check cross-references)
- [ ] Add undocumented API detection
- [ ] Add unresolved citation detection
- [ ] Unit tests: validation logic; error reporting
- [ ] Integration test: Full docs validation; error report generation

**Week 7: Polish & Release**
- [ ] Handle edge cases (long signatures, complex type hints, async/await)
- [ ] Add API index table generation
- [ ] Write TypeScript/Python API documentation
- [ ] Create example codebases with generated docs
- [ ] Test with real zen-sci packages
- [ ] Finalize error messages
- [ ] Publish v0.1.0 to npm (@zen-sci/documentation-mcp)
- [ ] Full regression test suite (40+ unit + 10+ integration tests)

### 4.3 Dependencies & Prerequisites

**Hard Dependencies:**
- packages/core v1.0 (stable): DocumentRequest/DocumentResponse, bibliography system

**New Infrastructure (to be added to packages/core):**
- API extraction abstraction (pluggable extractors)
- Docstring parsing module (multiple styles)
- Diagram generation (Mermaid)

**External Tools:**
- Python `ast` module (built-in)
- ts-morph (TypeScript AST parser)
- pandoc (optional: for converting docstrings to different formats)
- Mermaid (diagram generation)

---

### 4.4 Testing Strategy

**Unit Tests (TypeScript):**
```typescript
// zen-sci/servers/documentation-mcp/__tests__/docs.test.ts

describe('documentation-mcp', () => {
  describe('extract_api', () => {
    it('should extract Python functions', async () => {
      const result = await extractAPI(['/path/to/test.py'], 'python');
      expect(result.api_items).toContainEqual(
        expect.objectContaining({
          type: 'function',
          name: expect.any(String),
        })
      );
    });

    it('should extract TypeScript types and interfaces', async () => {
      const result = await extractAPI(['/path/to/test.ts'], 'typescript');
      expect(result.api_items).toContainEqual(
        expect.objectContaining({ type: 'interface' })
      );
    });
  });

  describe('generate_documentation', () => {
    it('should generate valid MkDocs Markdown', async () => {
      const api_items = [/* ... */];
      const result = await generateDocumentation(api_items, 'mkdocs');
      expect(result.content).toMatch(/^# /); // Starts with Markdown H1
    });

    it('should generate valid Sphinx .rst', async () => {
      const result = await generateDocumentation(api_items, 'sphinx');
      expect(result.content).toMatch(/^=+\n/); // Sphinx .rst title format
    });
  });
});
```

**Integration Tests (Python):**
```python
# zen-sci/servers/documentation-mcp/tests/test_docs_e2e.py

def test_full_documentation_pipeline():
    """Extract API and generate docs for zen-sci/packages/core."""
    # Extract API
    # Generate Sphinx docs
    # Verify: no broken refs, all items documented
    # Generate MkDocs docs
    # Verify: valid Markdown, navigation working

def test_docstring_citation_resolution():
    """Test citation extraction and bibliography generation."""
    # Extract API with citations in docstrings
    # Generate docs with bibliography
    # Verify: all citations resolved, .bib file generated
```

---

## 5. Risk Assessment & Mitigation

| Risk | Likelihood | Impact | Mitigation |
|-------|-----------|--------|-----------|
| AST parsing edge cases (decorators, type hints) | Medium | Medium | Comprehensive unit tests; fallback to plain text if parsing fails |
| ts-morph dependency stability | Low | Medium | Pin version; test regularly; have fallback parser |
| Docstring style inconsistency | Medium | Low | Support 4 major styles; warn on unknown patterns |
| Performance on large codebases (1000+ items) | Low | Medium | Profile Week 5; implement streaming if needed |
| Circular dependencies in class hierarchies | Low | Low | Detect cycles; warn user; render cautiously |

---

## 6. Rollback & Contingency

**If TypeScript extraction is problematic (Week 3):**
- Ship Python-only in v0.1; add TypeScript in v0.1.1.

**If Sphinx generation is complex (Week 3):**
- Ship MkDocs-only in v0.1; add Sphinx in v0.1.1.

**If diagram generation is unreliable (Week 4):**
- Remove diagrams from v0.1; include in v0.1.1.

**Rollback procedure:**
- Tag npm release with `-rc` suffix if issues found.
- Publish corrected version after fixes.

---

## 7. Appendices

### 7.1 Related Work & Standards

- **Sphinx:** Industry-standard documentation generator (Python).
- **MkDocs:** Modern Markdown-based documentation (Python).
- **ts-morph:** TypeScript AST manipulation library.
- **pdoc:** Python auto-documentation tool.
- **typedoc:** TypeScript auto-documentation tool.
- **NumPy docstring standard:** Structured docstring format.
- **Google docstring style:** Alternative structured docstring format.
- **JSDoc:** JavaScript/TypeScript documentation standard.

### 7.2 Future Considerations (v1.1+)

- **Live preview in Claude Code:** Real-time doc generation as code changes.
- **Search functionality:** Full-text search over generated docs.
- **Changelog generation:** Auto-generate CHANGELOG from docstring changes.
- **Dependency graph visualization:** Interactive dependency diagrams.
- **Code coverage badges:** Integrate test coverage into docs.

### 7.3 Open Questions

1. **Should private APIs be documented?** (MVP: exclude by default; configurable.)
2. **How deeply should we auto-generate examples?** (MVP: extract from docstrings only.)
3. **Should generated docs be editable, or read-only from code?** (MVP: read-only; manual docs are separate.)

### 7.4 Schema Backlog

**Potential future additions:**

```typescript
export interface DocumentationRequest extends DocumentRequest {
  // v1.1 candidates:
  include_generated_changelog?: boolean;
  include_dependency_graph?: boolean;
  include_test_coverage?: boolean;
  custom_sidebar_config?: SidebarConfig;
  search_enabled?: boolean;
  live_preview_mode?: boolean;
}
```

---

**End of documentation-mcp Specification**

---

## Audit Notes (2026-02-18)

**Auditor:** Claude (Opus-level audit)
**Status:** Reviewed

### Issues
- **CRITICAL-2:** Instantiates `Server` directly instead of extending `ZenSciServer`.
- **CRITICAL-3:** Uses low-level `setRequestHandler` pattern.
- MCP SDK imports correct. Tool naming snake_case — consistent.
- Most complex Phase 2 spec. Dual-language AST extraction (Python + TypeScript) is ambitious but well-specified.

#### Decision Propagation (Cruz, 2026-02-18)
- **CRITICAL-2 resolved:** `ZenSciServer` abstract base class deprecated. Use `createZenSciServer()` factory + `McpServer.registerTool()` with Zod schemas at implementation time.
- **CRITICAL-3 resolved:** Low-level `setRequestHandler` replaced by `McpServer.registerTool()` with Zod input schemas per MCP SDK v2.
