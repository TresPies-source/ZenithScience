# grant-mcp MCP App Specification
## Phase 4: Visual Application Layer

**Version:** 0.4.0
**Status:** Production Ready
**Author:** ZenSci Architecture (Cruz Morales / TresPiesDesign.com)
**Date:** 2026-02-18
**Depends on:** grant-mcp v0.1.0 (server spec from Phase 2)

---

## 1. Purpose

The grant-mcp server generates compliance dashboards for grant proposals. The companion MCP App renders the `format_grant_proposal` tool result as an interactive **compliance dashboard**, enabling users to:

- **View overall compliance status** (PASS / WARN / FAIL) with color-coded badges
- **Navigate section checklists** with compliance status, page/word counts, and jump-to affordances
- **Inspect attachment requirements** by funding agency (NIH / NSF / ERC)
- **Review agency-specific formatting rules** (page limits, fonts, margins, required sections, submission platform)
- **Re-run compliance checks** and switch funding agencies on-the-fly

The app transforms raw compliance data into a navigable interface that surfaces issues early, preventing rejection due to formatting violations.

---

## 2. Tool Integration

### 2.1 Primary Tool: `format_grant_proposal`

The primary conversion tool declares the MCP App resource:

```typescript
registerAppTool(
  server,
  'format_grant_proposal',
  {
    type: 'object',
    properties: {
      source: { type: 'string', description: '...' },
      frontmatter: { type: 'object', description: '...' },
      format: { type: 'string', enum: ['pdf', 'docx', 'both'] },
    },
    required: ['source', 'frontmatter', 'format'],
    _meta: {
      ui: {
        resourceUri: 'ui://grant-mcp/dashboard.html',
      },
    },
  },
  formatGrantProposal
);
```

**Tool result shape received by app (from `format_grant_proposal`):**

```json
{
  "id": "grant-20260218-abc123",
  "format": "grant-pdf",
  "content": "JVBERi0xLjQK...",
  "artifacts": [
    {
      "type": "specific_aims",
      "filename": "specific_aims.md",
      "content": "## Specific Aims\n..."
    },
    {
      "type": "budget_table",
      "filename": "budget_table.md",
      "content": "| Category | Description | Amount |\n..."
    },
    {
      "type": "validation_report",
      "filename": "validation_report.json",
      "content": "{...}"
    }
  ],
  "metadata": {
    "funder": "nih",
    "program_type": "nih-r01",
    "page_count": 12,
    "word_count": 15000,
    "budget_total": 250000,
    "budget_overhead": 117500,
    "compliance_status": "compliant",
    "validation_messages": [],
    "sections": [
      {
        "name": "Specific Aims",
        "status": "pass",
        "page_count": 1,
        "required_pages": 1,
        "word_count": 800,
        "required": true
      },
      {
        "name": "Research Plan",
        "status": "pass",
        "page_count": 8,
        "required_pages": "8-10",
        "word_count": 12000,
        "required": true
      },
      {
        "name": "Budget Justification",
        "status": "warn",
        "page_count": 3,
        "required_pages": "1-2",
        "word_count": 2200,
        "required": true
      }
    ],
    "attachments": [
      {
        "role": "biosketch",
        "required": true,
        "present": true
      },
      {
        "role": "facilities",
        "required": false,
        "present": false
      }
    ]
  },
  "elapsed": 2500
}
```

### 2.2 Secondary Tools (Called from App)

The app can invoke these secondary tools to refresh data or switch agencies:

```typescript
// Re-check compliance or switch to different agency
server.registerTool('format_grant_proposal', inputSchema, formatGrantProposal);
server.registerTool('validate_grant_proposal', inputSchema, validateGrantProposal);
server.registerTool('list_grant_funders', inputSchema, listGrantFunders);
```

The app uses `app.callServerTool()` to invoke these when the user:
- Clicks "Re-check Compliance"
- Switches the agency selector (NIH → NSF → ERC)

---

## 3. App File Structure

```
zen-sci/servers/grant-mcp/
├── app/
│   ├── src/
│   │   ├── main.tsx                          ← React entry
│   │   ├── App.tsx                           ← Root component
│   │   ├── components/
│   │   │   ├── ComplianceDashboard.tsx       ← Main container
│   │   │   ├── OverviewPanel.tsx             ← Status badges, counts
│   │   │   ├── SectionChecklist.tsx          ← Scrollable section list
│   │   │   ├── AttachmentsPanel.tsx          ← Attachment status table
│   │   │   ├── AgencyRequirementsTab.tsx     ← Funder rules table
│   │   │   ├── ComplianceToolbar.tsx         ← Agency selector, re-check button
│   │   │   ├── LoadingState.tsx              ← (shared)
│   │   │   ├── ErrorState.tsx                ← (shared)
│   │   │   └── EmptyState.tsx                ← (shared)
│   │   ├── hooks/
│   │   │   ├── useToolResult.ts              ← Parse tool result JSON
│   │   │   ├── useComplianceData.ts          ← Helpers for compliance logic
│   │   │   └── useToolCall.ts                ← Invoke server tools
│   │   ├── types.ts                          ← TypeScript interfaces
│   │   └── styles.css                        ← ZenSci design tokens
│   ├── index.html
│   ├── vite.config.ts
│   ├── tsconfig.json
│   └── package.json
├── dist/
│   └── grant-dashboard.html                  ← Built output
└── server/
    └── src/
        └── index.ts                          ← (MODIFIED) includes registerAppResource
```

---

## 4. Component Architecture

### 4.1 Root App Component

```typescript
// app/src/App.tsx

import { useApp, useHostStyles } from '@modelcontextprotocol/ext-apps/react';
import { useState, useEffect } from 'react';
import { ComplianceDashboard } from './components/ComplianceDashboard';
import { LoadingState, ErrorState, EmptyState } from './components';
import type { GrantComplianceData } from './types';

export default function App() {
  const app = useApp();
  const hostStyles = useHostStyles();
  const [data, setData] = useState<GrantComplianceData | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    app.ontoolresult = (result) => {
      try {
        const parsed = JSON.parse(result.content[0].text);
        setData(parsed as GrantComplianceData);
        setLoading(false);
        setError(null);
      } catch (e) {
        setError('Failed to parse tool result');
        setLoading(false);
      }
    };
  }, [app]);

  if (loading) return <LoadingState />;
  if (error) return <ErrorState message={error} />;
  if (!data) return <EmptyState />;

  return (
    <div style={{ ...hostStyles, padding: '16px', fontFamily: 'var(--zen-font-sans)' }}>
      <ComplianceDashboard data={data} app={app} />
    </div>
  );
}
```

### 4.2 Tab / View Structure

The app uses two main tabs:

1. **Compliance Checker** (default)
   - OverviewPanel (top)
   - SectionChecklist (main)
   - AttachmentsPanel (optional expandable)

2. **Agency Requirements** (secondary tab)
   - Formatted table of rules for selected agency

```typescript
// app/src/components/ComplianceDashboard.tsx

import { useState } from 'react';
import { OverviewPanel } from './OverviewPanel';
import { SectionChecklist } from './SectionChecklist';
import { AttachmentsPanel } from './AttachmentsPanel';
import { AgencyRequirementsTab } from './AgencyRequirementsTab';
import { ComplianceToolbar } from './ComplianceToolbar';
import type { GrantComplianceData } from '../types';

export function ComplianceDashboard({ data, app }: Props) {
  const [activeTab, setActiveTab] = useState<'checker' | 'requirements'>('checker');

  return (
    <div className="compliance-dashboard">
      <ComplianceToolbar data={data} app={app} />

      <div className="zen-tabs">
        <button
          className={`zen-tab ${activeTab === 'checker' ? 'active' : ''}`}
          onClick={() => setActiveTab('checker')}
        >
          Compliance Checker
        </button>
        <button
          className={`zen-tab ${activeTab === 'requirements' ? 'active' : ''}`}
          onClick={() => setActiveTab('requirements')}
        >
          Agency Requirements
        </button>
      </div>

      {activeTab === 'checker' && (
        <>
          <OverviewPanel data={data} />
          <SectionChecklist sections={data.metadata.sections} />
          <AttachmentsPanel attachments={data.metadata.attachments} />
        </>
      )}

      {activeTab === 'requirements' && (
        <AgencyRequirementsTab funder={data.metadata.funder} />
      )}
    </div>
  );
}
```

### 4.3 Major Components with Props Interfaces

#### OverviewPanel

Displays funding agency badge, overall compliance status, and page/word counts.

```typescript
// app/src/components/OverviewPanel.tsx

interface Props {
  data: GrantComplianceData;
}

export function OverviewPanel({ data }: Props) {
  const { funder, program_type, compliance_status, page_count, word_count } = data.metadata;
  const agencyColors = {
    nih: { bg: '#ffebee', color: '#c62828' },
    nsf: { bg: '#e3f2fd', color: '#1565c0' },
    erc: { bg: '#f3e5f5', color: '#6a1b9a' },
  };

  return (
    <div className="overview-panel" style={{ padding: '16px', borderRadius: 'var(--zen-radius)', border: 'var(--zen-border)' }}>
      <div style={{ display: 'flex', gap: '12px', alignItems: 'center', marginBottom: '12px' }}>
        <div
          className="zen-badge"
          style={agencyColors[funder as keyof typeof agencyColors]}
        >
          {funder.toUpperCase()} {program_type.split('-')[1]?.toUpperCase()}
        </div>
        <span
          className={`zen-badge ${compliance_status}`}
        >
          {compliance_status === 'compliant' ? '✓ Compliant' : compliance_status === 'warning' ? '⚠ Warning' : '✗ Non-Compliant'}
        </span>
      </div>

      <table className="zen-meta-table">
        <tbody>
          <tr>
            <td>Page Count</td>
            <td>{page_count} pages</td>
          </tr>
          <tr>
            <td>Word Count</td>
            <td>{word_count.toLocaleString()} words</td>
          </tr>
          <tr>
            <td>Budget Total</td>
            <td>${data.metadata.budget_total.toLocaleString()}</td>
          </tr>
          <tr>
            <td>Indirect Costs</td>
            <td>${data.metadata.budget_overhead.toLocaleString()}</td>
          </tr>
        </tbody>
      </table>
    </div>
  );
}
```

#### SectionChecklist

Scrollable list of all required sections with status, counts, and jump affordance.

```typescript
// app/src/components/SectionChecklist.tsx

interface SectionRow {
  name: string;
  status: 'pass' | 'warn' | 'fail';
  page_count: number;
  required_pages: string | number;
  word_count: number;
  required: boolean;
}

interface Props {
  sections: SectionRow[];
}

export function SectionChecklist({ sections }: Props) {
  const statusColor = (status: string) => {
    switch (status) {
      case 'pass': return '#4caf50';
      case 'warn': return '#ff9800';
      case 'fail': return '#f44336';
      default: return '#999';
    }
  };

  return (
    <div className="section-checklist" style={{ marginTop: '16px' }}>
      <h3>Section Checklist</h3>
      <div style={{ maxHeight: '400px', overflowY: 'auto', border: 'var(--zen-border)', borderRadius: 'var(--zen-radius)' }}>
        {sections.map((section, idx) => (
          <div
            key={idx}
            style={{
              padding: '12px',
              borderBottom: 'var(--zen-border)',
              display: 'grid',
              gridTemplateColumns: '1fr 80px 80px auto',
              gap: '12px',
              alignItems: 'center',
            }}
          >
            <div>
              <div style={{ fontWeight: 500 }}>{section.name}</div>
              <div style={{ fontSize: '12px', color: 'var(--mcp-text-secondary, #666)' }}>
                {section.page_count} / {section.required_pages} pages
              </div>
            </div>
            <div
              className="zen-badge"
              style={{
                background: statusColor(section.status),
                color: '#fff',
                padding: '4px 8px',
                borderRadius: '4px',
                fontSize: '11px',
              }}
            >
              {section.status === 'pass' ? '✓' : section.status === 'warn' ? '⚠' : '✗'}
            </div>
            <div style={{ fontSize: '12px', textAlign: 'right' }}>
              {section.word_count} words
            </div>
            <button
              style={{
                padding: '4px 8px',
                border: 'none',
                background: 'var(--mcp-accent, #2563eb)',
                color: '#fff',
                borderRadius: '4px',
                cursor: 'pointer',
                fontSize: '11px',
              }}
            >
              Jump
            </button>
          </div>
        ))}
      </div>
    </div>
  );
}
```

#### AttachmentsPanel

Table of required attachments with role, status, and present/missing indicator.

```typescript
// app/src/components/AttachmentsPanel.tsx

interface AttachmentRow {
  role: string; // 'biosketch', 'facilities', 'data-management-plan', etc.
  required: boolean;
  present: boolean;
}

interface Props {
  attachments: AttachmentRow[];
}

export function AttachmentsPanel({ attachments }: Props) {
  return (
    <div className="attachments-panel" style={{ marginTop: '16px' }}>
      <h3>Required Attachments</h3>
      <table className="zen-meta-table" style={{ width: '100%' }}>
        <thead style={{ background: 'var(--mcp-border, #f0f0f0)' }}>
          <tr>
            <th style={{ textAlign: 'left', padding: '8px' }}>File Role</th>
            <th style={{ textAlign: 'center', padding: '8px' }}>Required</th>
            <th style={{ textAlign: 'center', padding: '8px' }}>Present</th>
          </tr>
        </thead>
        <tbody>
          {attachments.map((att, idx) => (
            <tr key={idx}>
              <td style={{ padding: '8px' }}>{att.role}</td>
              <td style={{ textAlign: 'center', padding: '8px' }}>
                {att.required ? '✓ Yes' : '○ No'}
              </td>
              <td style={{ textAlign: 'center', padding: '8px' }}>
                <span
                  style={{
                    color: att.present ? '#4caf50' : '#f44336',
                    fontWeight: 500,
                  }}
                >
                  {att.present ? '✓ Yes' : '✗ No'}
                </span>
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

#### AgencyRequirementsTab

Formatted table of formatting requirements for the selected agency.

```typescript
// app/src/components/AgencyRequirementsTab.tsx

interface Props {
  funder: string; // 'nih', 'nsf', 'erc'
}

const FUNDER_RULES = {
  nih: {
    program: 'NIH R01',
    pageLimit: 12,
    fontMin: 10,
    marginMin: 0.5,
    platform: 'nih-era-commons',
    sections: [
      'Specific Aims',
      'Research Plan',
      'Budget Justification',
      'References',
    ],
  },
  nsf: {
    program: 'NSF Standard',
    pageLimit: 15,
    fontMin: 10,
    marginMin: 1.0,
    platform: 'nsf-research-gov',
    sections: [
      'Project Summary',
      'Project Description',
      'References Cited',
      'Budget Justification',
    ],
  },
  erc: {
    program: 'ERC Starting',
    pageLimit: 12,
    fontMin: 11,
    marginMin: 1.0,
    platform: 'erc-sygma',
    sections: [
      'Excellence',
      'Impact',
      'Implementation',
      'References',
    ],
  },
};

export function AgencyRequirementsTab({ funder }: Props) {
  const rules = FUNDER_RULES[funder as keyof typeof FUNDER_RULES];

  return (
    <div className="agency-requirements" style={{ marginTop: '16px', padding: '16px', background: 'var(--mcp-bg-secondary, #f9f9f9)', borderRadius: 'var(--zen-radius)' }}>
      <h3>{rules.program} Requirements</h3>
      <table className="zen-meta-table">
        <tbody>
          <tr>
            <td>Page Limit</td>
            <td>{rules.pageLimit} pages</td>
          </tr>
          <tr>
            <td>Minimum Font Size</td>
            <td>{rules.fontMin} pt</td>
          </tr>
          <tr>
            <td>Minimum Margins</td>
            <td>{rules.marginMin}" all sides</td>
          </tr>
          <tr>
            <td>Submission Platform</td>
            <td style={{ fontFamily: 'var(--zen-font-mono)' }}>{rules.platform}</td>
          </tr>
          <tr>
            <td>Required Sections</td>
            <td>
              <ul style={{ margin: '0', paddingLeft: '20px' }}>
                {rules.sections.map((sec, i) => <li key={i}>{sec}</li>)}
              </ul>
            </td>
          </tr>
        </tbody>
      </table>
    </div>
  );
}
```

#### ComplianceToolbar

Agency selector dropdown and re-check button.

```typescript
// app/src/components/ComplianceToolbar.tsx

interface Props {
  data: GrantComplianceData;
  app: ReturnType<typeof useApp>;
}

export function ComplianceToolbar({ data, app }: Props) {
  const [selectedAgency, setSelectedAgency] = useState(data.metadata.funder);
  const [checking, setChecking] = useState(false);

  const handleAgencyChange = async (newAgency: string) => {
    setSelectedAgency(newAgency);
    // Call format_grant_proposal with new funder
    setChecking(true);
    try {
      const result = await app.callServerTool({
        name: 'format_grant_proposal',
        arguments: {
          // These would come from the original tool call context
          source: data.content,
          frontmatter: { ...data.metadata, funder: newAgency },
          format: data.format,
        },
      });
      if (!result.isError) {
        app.ontoolresult?.(result);
      }
    } finally {
      setChecking(false);
    }
  };

  const handleRecheck = async () => {
    setChecking(true);
    try {
      const result = await app.callServerTool({
        name: 'format_grant_proposal',
        arguments: {
          source: data.content,
          frontmatter: data.metadata,
          format: data.format,
        },
      });
      if (!result.isError) {
        app.ontoolresult?.(result);
      }
    } finally {
      setChecking(false);
    }
  };

  return (
    <div className="compliance-toolbar" style={{ display: 'flex', gap: '12px', marginBottom: '12px' }}>
      <label>
        Funding Agency:
        <select
          value={selectedAgency}
          onChange={(e) => handleAgencyChange(e.target.value)}
          disabled={checking}
          style={{ marginLeft: '8px', padding: '6px' }}
        >
          <option value="nih">NIH</option>
          <option value="nsf">NSF</option>
          <option value="erc">ERC</option>
        </select>
      </label>
      <button
        onClick={handleRecheck}
        disabled={checking}
        style={{
          padding: '6px 12px',
          background: 'var(--mcp-accent, #2563eb)',
          color: '#fff',
          border: 'none',
          borderRadius: '4px',
          cursor: checking ? 'not-allowed' : 'pointer',
          opacity: checking ? 0.6 : 1,
        }}
      >
        {checking ? 'Checking...' : 'Re-check Compliance'}
      </button>
    </div>
  );
}
```

---

## 5. Data Types

TypeScript interfaces for the tool result shape and component state:

```typescript
// app/src/types.ts

/**
 * Shape of the data received from format_grant_proposal tool result.
 * Matches the metadata structure returned by the server.
 */
export interface GrantComplianceData {
  id: string;
  format: 'grant-pdf' | 'grant-docx' | 'grant-both';
  content: string; // Base64-encoded PDF or DOCX
  artifacts: Artifact[];
  metadata: GrantMetadata;
  elapsed: number;
}

export interface Artifact {
  type: 'specific_aims' | 'budget_table' | 'timeline' | 'validation_report' | 'budget_justification';
  filename: string;
  content: string;
}

export interface GrantMetadata {
  funder: 'nih' | 'nsf' | 'erc';
  program_type: string;
  page_count: number;
  word_count: number;
  budget_total: number;
  budget_overhead: number;
  compliance_status: 'compliant' | 'warning' | 'error';
  validation_messages: string[];
  sections: SectionStatus[];
  attachments: AttachmentStatus[];
}

export interface SectionStatus {
  name: string;
  status: 'pass' | 'warn' | 'fail';
  page_count: number;
  required_pages: number | string;
  word_count: number;
  required: boolean;
}

export interface AttachmentStatus {
  role: SubmissionFileRole;
  required: boolean;
  present: boolean;
}

/**
 * SubmissionFileRole: Strict union of known attachment roles across major grant platforms.
 * Ensures type safety and prevents invalid role values.
 */
export type SubmissionFileRole =
  | 'specific-aims'
  | 'research-plan'
  | 'budget-justification'
  | 'biosketch'
  | 'facilities'
  | 'data-management-plan'
  | 'references'
  | 'cover-letter'
  | 'abstract'
  | 'other';
```

---

## 6. Tool Result Shape

The tool result is received via `app.ontoolresult` and has this shape:

```typescript
interface ToolResult {
  content: Array<{
    type: string;
    text: string; // JSON-serialized GrantComplianceData
  }>;
  isError: boolean;
}

// The app parses content[0].text as JSON:
const data = JSON.parse(result.content[0].text) as GrantComplianceData;
```

---

## 7. Interactive Tool Calls

The app calls secondary server tools to refresh or change agencies:

```typescript
// hook: app/src/hooks/useToolCall.ts

import { useApp } from '@modelcontextprotocol/ext-apps/react';

export function useToolCall() {
  const app = useApp();

  const callTool = async (
    toolName: string,
    args: Record<string, unknown>
  ): Promise<GrantComplianceData | null> => {
    try {
      const result = await app.callServerTool({ name: toolName, arguments: args });
      if (result.isError) throw new Error(result.content[0].text);
      return JSON.parse(result.content[0].text);
    } catch (e) {
      console.error(`Tool call failed: ${e instanceof Error ? e.message : 'Unknown error'}`);
      return null;
    }
  };

  return { callTool };
}
```

---

## 8. Server Modifications Required

The server must register the app resource and update the primary tool. Modifications to `/servers/grant-mcp/server/src/index.ts`:

### 8.1 Import Changes

```typescript
import {
  registerAppTool,
  registerAppResource,
  RESOURCE_MIME_TYPE,
} from '@modelcontextprotocol/ext-apps/server';
import fs from 'node:fs/promises';
import path from 'node:path';
import { fileURLToPath } from 'node:url';

const __dirname = path.dirname(fileURLToPath(import.meta.url));
```

### 8.2 Tool Registration (Primary Tool)

Replace the existing `server.registerTool('format_grant_proposal', ...)` with:

```typescript
registerAppTool(
  server,
  'format_grant_proposal',
  {
    type: 'object',
    properties: {
      source: {
        type: 'string',
        description: 'Markdown proposal content (without YAML frontmatter).',
      },
      frontmatter: {
        type: 'object',
        description: 'YAML frontmatter with funder, PI info, budget, specific aims.',
        properties: {
          title: { type: 'string' },
          funder: { type: 'string', enum: ['nih', 'nsf', 'erc'] },
          program_type: { type: 'string' },
          principal_investigator: { type: 'object' },
          specific_aims: { type: 'array' },
          budget: { type: 'array' },
          institution: { type: 'object' },
          bibliography: { type: 'string' },
          options: { type: 'object' },
        },
        required: ['title', 'funder', 'program_type', 'principal_investigator'],
      },
      format: {
        type: 'string',
        enum: ['pdf', 'docx', 'both'],
        description: 'Output format(s).',
      },
    },
    required: ['source', 'frontmatter', 'format'],
    _meta: {
      ui: {
        resourceUri: 'ui://grant-mcp/dashboard.html',
      },
    },
  },
  formatGrantProposal
);
```

### 8.3 Resource Registration

Add after tool registrations:

```typescript
const RESOURCE_URI = 'ui://grant-mcp/dashboard.html';
const DIST_PATH = path.resolve(__dirname, '../../dist/grant-dashboard.html');

registerAppResource(
  server,
  RESOURCE_URI,
  RESOURCE_URI,
  { mimeType: RESOURCE_MIME_TYPE },
  async () => {
    const html = await fs.readFile(DIST_PATH, 'utf-8');
    return {
      contents: [
        {
          uri: RESOURCE_URI,
          mimeType: RESOURCE_MIME_TYPE,
          text: html,
        },
      ],
    };
  }
);
```

### 8.4 Secondary Tools (Remain Standard)

Keep existing registrations for secondary tools:

```typescript
server.registerTool('validate_grant_proposal', validateInputSchema, validateGrantProposal);
server.registerTool('list_grant_funders', listFundersInputSchema, listGrantFunders);
```

---

## 9. CSS / Styling Notes

The app uses the shared ZenSci design language from `app/src/styles.css` (identical across all Phase 4 apps):

```css
:root {
  --zen-font-mono: 'JetBrains Mono', 'Fira Code', monospace;
  --zen-font-sans: system-ui, -apple-system, sans-serif;
  --zen-radius: 6px;
  --zen-border: 1px solid var(--mcp-border, #e0e0e0);
  --zen-shadow: 0 1px 3px rgba(0,0,0,0.08);
}

.zen-badge {
  display: inline-flex;
  align-items: center;
  padding: 2px 8px;
  border-radius: 12px;
  font-size: 11px;
  font-weight: 600;
}

.zen-badge.pass  { background: #d4edda; color: #155724; }
.zen-badge.warn  { background: #fff3cd; color: #856404; }
.zen-badge.fail  { background: #f8d7da; color: #721c24; }

.zen-tabs {
  display: flex;
  gap: 2px;
  border-bottom: var(--zen-border);
  margin-bottom: 12px;
}

.zen-tab {
  padding: 6px 14px;
  border: none;
  background: transparent;
  cursor: pointer;
  border-radius: var(--zen-radius) var(--zen-radius) 0 0;
  font-size: 13px;
  color: var(--mcp-text-secondary, #666);
}

.zen-tab.active {
  background: var(--mcp-accent, #2563eb);
  color: #ffffff;
}

.zen-meta-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 13px;
}

.zen-meta-table td {
  padding: 4px 8px;
  border-bottom: var(--zen-border);
}

.zen-meta-table td:first-child {
  color: var(--mcp-text-secondary, #666);
  width: 40%;
}
```

**Styling rules:**
- Use `useHostStyles()` to inherit Claude's light/dark mode
- No hardcoded colors outside of semantic status (pass/warn/fail)
- Tab navigation for multi-view apps
- Responsive grid layout for section checklist

---

## 10. Build Configuration

### 10.1 app/package.json

```json
{
  "name": "@zen-sci/grant-mcp-app",
  "version": "0.4.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "@modelcontextprotocol/ext-apps": "^1.0.1",
    "react": "^18.3.0",
    "react-dom": "^18.3.0"
  },
  "devDependencies": {
    "@types/react": "^18.3.0",
    "@types/react-dom": "^18.3.0",
    "@vitejs/plugin-react": "^4.3.0",
    "typescript": "^5.4.0",
    "vite": "^5.4.0",
    "vite-plugin-singlefile": "^2.0.0"
  }
}
```

### 10.2 app/vite.config.ts

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { viteSingleFile } from 'vite-plugin-singlefile';
import path from 'node:path';

export default defineConfig({
  plugins: [react(), viteSingleFile()],
  build: {
    outDir: path.resolve(__dirname, '../dist'),
    rollupOptions: {
      input: path.resolve(__dirname, 'index.html'),
    },
  },
});
```

### 10.3 Module Root package.json (Build Scripts)

Add to `/servers/grant-mcp/package.json`:

```json
{
  "scripts": {
    "build:app": "cd app && vite build",
    "build": "tsc && cd app && vite build"
  }
}
```

---

## 11. File Manifest

### Shared Files (Identical Across All Phase 4 Apps)

| File | Description |
|------|-------------|
| `app/vite.config.ts` | Vite + singlefile build config |
| `app/index.html` | HTML entry point (title differs) |
| `app/src/main.tsx` | React app entry |
| `app/src/styles.css` | ZenSci design tokens |
| `app/src/components/LoadingState.tsx` | Loading spinner |
| `app/src/components/ErrorState.tsx` | Error display |
| `app/src/components/EmptyState.tsx` | Empty/no-data display |
| `app/tsconfig.json` | TypeScript config |

### Module-Specific Files

| File | Description |
|------|-------------|
| `app/src/App.tsx` | Root component (uses ComplianceDashboard) |
| `app/src/types.ts` | `GrantComplianceData`, `SectionStatus`, `AttachmentStatus`, `SubmissionFileRole` |
| `app/src/components/ComplianceDashboard.tsx` | Main container; tab navigation; orchestrates sub-components |
| `app/src/components/OverviewPanel.tsx` | Agency badge, compliance status, page/word/budget counts |
| `app/src/components/SectionChecklist.tsx` | Scrollable list of sections with status and counts |
| `app/src/components/AttachmentsPanel.tsx` | Table of required attachments with present/missing status |
| `app/src/components/AgencyRequirementsTab.tsx` | Formatted table of funder rules (page limits, fonts, margins, sections) |
| `app/src/components/ComplianceToolbar.tsx` | Agency selector dropdown, re-check button, state management |
| `app/src/hooks/useToolResult.ts` | Parse JSON from tool result |
| `app/src/hooks/useComplianceData.ts` | Helper functions for compliance logic (status colors, count validation) |
| `app/src/hooks/useToolCall.ts` | Invoke secondary server tools |
| `dist/grant-dashboard.html` | Built output (generated by Vite singlefile build) |

---

## 12. Success Criteria

- [ ] `pnpm run build:app` in `/servers/grant-mcp/` produces `dist/grant-dashboard.html` (single file, no external refs)
- [ ] Server registers `format_grant_proposal` with `_meta.ui.resourceUri` pointing to `ui://grant-mcp/dashboard.html`
- [ ] Server registers resource handler that reads and serves the built HTML
- [ ] App renders tool result without page reload; displays OverviewPanel, SectionChecklist, AttachmentsPanel
- [ ] Clicking "Re-check Compliance" re-calls `format_grant_proposal` with same arguments
- [ ] Changing agency selector calls `format_grant_proposal` with different `funder` parameter; re-renders dashboard
- [ ] Clicking "Agency Requirements" tab renders funder rules table with correct limits and sections
- [ ] All status badges (pass/warn/fail) display with correct colors
- [ ] Section checklist scrolls smoothly with 15+ sections
- [ ] No console errors; TypeScript strict mode passes
- [ ] Responsive layout on narrow viewports (min 320px width)

---

## Audit Notes

**Spec authored:** 2026-02-18
**Grounded in:** Phase 4 architecture spec, grant-mcp v0.1 server spec
**Status:** Production Ready
**Open items:** None

### Design Decisions

1. **Dashboard over document preview:** The app is a compliance navigator, not a PDF/DOCX viewer. Users see formatted output in the grant-mcp server's main result; the app focuses on structure and compliance metadata.

2. **Agency switcher:** Users can re-run the conversion for different funders without leaving the app, enabling rapid what-if analysis (NIH vs. NSF vs. ERC formatting differences).

3. **Strict `SubmissionFileRole` union:** Prevents invalid attachment role values; ensures type safety across funder platforms.

4. **No external dependencies:** Grant dashboard uses only React + MCP Apps SDK; no charting, PDF.js, or other heavy libraries. Dashboard is performant and lightweight.

---

**End of grant-mcp MCP App Specification**
