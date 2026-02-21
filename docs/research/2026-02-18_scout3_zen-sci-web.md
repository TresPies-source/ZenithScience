# Scout 3: zen-sci-web v0.1 — Thin Client vs. Stateful Workspace

**Date:** 2026-02-18
**Scout:** Claude (Strategic Reconnaissance)
**Status:** Architecture Gap Analysis
**Audience:** Product Owner (Cruz), Engineering Leadership

---

## Core Finding

**The spec positions zen-sci-web as a thin client that proxies to the desktop app's state. This contradicts the product vision: the portal should be a full, independent stateful workspace where researchers work directly in canonical source.**

The product owner's correction is explicit:
> **Document catalog: Cloud (AgenticGateway backend) owns the canonical catalog. Users on the web portal are working directly in the canonical source, not a clone.**

The current spec fails on this axis. Most routes treat the web portal as a *display layer* for data the desktop app owns or syncs from local storage.

---

## Part 1: Where the Current Spec is Thin

### 1.1 Data Flow: Portal as Read-Only View

**Current spec language (Section 3.1):**
```
Browser → SvelteKit Server (proxy) → AgenticGateway → ZenSci servers
```

**Reality check:** This implies:
- Desktop app creates/edits documents locally
- Desktop app converts via ZenSci servers
- Conversion result appears on web portal
- Portal is a viewer, not an editor

**Evidence in spec:**
- Section 2 (Goals): "Document sharing & discovery — Share ZenSci output links... Recipients see rendered document"
- Section 3.8 (Document Viewer): "Readonly: true" — portal documents are non-editable
- Section 4 (Implementation Plan, Phase 3): "display document list... Filter by module type" — implies listing pre-converted outputs, not creating
- Non-Goals (Section 2): "❌ Document editor. Users edit markdown locally (desktop app) or via external editors. Web portal is viewer/curator only."

**The thin-client logic:** If the desktop app owns document creation/editing, and the portal only views converted outputs, then:
- Portal is a consumption layer
- Desktop is the authoritative editor
- Sync conflicts are possible (user edits on desktop, sees different state on portal)
- Portal's social features (groups, projects) are decorative metadata, not workspace constructs

---

### 1.2 Social Data Model: Assumed to Live in Gateway, Not Specified

**Spec Section 3.3 (Data Model):**
```typescript
export interface Group {
  id: string;
  name: string;
  description?: string;
  owner_id: string;
  members: Array<{ user_id, role, joined_at }>;
  project_count: number;
  // ...
}
```

**Reality check:** Where does this live?

The spec says: *"For v0.1, the Portal assumes Gateway memory endpoints exist. If not, portal uses basic file-based storage (JSON files in Gateway's data directory) as fallback."*

**This is not a detail.** The Gateway spec (ARCHITECTURE.md) shows the Gateway is a **tool orchestrator**, not a social platform. The Gateway's SQLite memory is designed for:
- Chat history (garden metaphor: seeds, snapshots)
- Memory context (agent state)
- Tool invocation logs

**Not for:**
- Groups (hierarchical collections)
- Roles and access control
- Invitation tokens (48-hour TTL tracking)
- Activity event streams
- Document metadata (word count, page count, citations)

**Implication:** Either:
1. The Gateway gets a major schema expansion to support social entities, OR
2. The portal becomes a **stateful application** with its own PostgreSQL database

The spec glosses this over. It doesn't clarify ownership of the social data model.

---

### 1.3 Project/Document Ownership: Ambiguous

**Spec Section 3.7 (Onboarding):**
```
1. Existing user (Alice in biology group) generates invite link
   - Portal: POST /api/groups/[group_id]/invites
   - Returns: { token: "abc123def456...", expires_at: "..." }

2. Bob clicks link → /invite/[token]
   - Portal shows public preview of group
   - CTA: "Sign up to join"

3. Bob signs up, claims invite
   - POST /api/invites/[token]/claim
   - Portal adds Bob to group (role: 'member')
```

**Question: Now that Bob is in the group, what can Bob do?**

The spec doesn't answer:
- Can Bob create projects directly in the portal, or only on desktop?
- If Bob creates a project on the web, where is the markdown stored?
- If Bob creates a project on desktop, how does it appear on the web?
- Can multiple Bobs edit the same project simultaneously on web + desktop?

**Current non-goal:** *"❌ Document editor. Users edit markdown locally (desktop app) or via external editors."*

**This means:** If Bob wants to collaborate with the group on the portal, he can't—he must use the desktop app. The portal is a *viewing theater*, not a *workspace*.

---

### 1.4 Activity Feed: Event Source Unspecified

**Spec Section 3.6 (Activity Feed):**
```typescript
export interface ActivityEvent {
  id: string;
  group_id: string;
  type: 'user_joined' | 'project_created' | 'document_converted' | ...;
  actor: User;
  timestamp: string;
  data: Record<string, unknown>;
}
```

**Question: Who generates these events?**

The spec shows polling: `fetch(/api/groups/[id]/activity?limit=50&offset=0)` every 30s.

**But where do the events come from?**
- Desktop app creates a document → does it fire an event to the portal?
- Portal user joins the group → does it fire an event that desktop users see?
- Or are these two separate event streams (desktop events, portal events) that never sync?

**The thin-client interpretation:** Desktop app is the event source. Portal polls to see what the desktop app did.

**The stateful workspace interpretation:** Portal is a peer to the desktop app. Both generate events. Both consume the same canonical event stream.

The spec doesn't clarify which.

---

### 1.5 MCP App Rendering: Viewer-Only, No Collaboration

**Spec Section 3.5 (MCP App Browser Rendering):**
```typescript
const iframeRef = ref<HTMLIFrameElement>();

onMount(() => {
  window.addEventListener('message', (e) => {
    const { type, payload } = e.data;
    switch (type) {
      case 'app:ready':
        iframeRef.current?.contentWindow?.postMessage({
          type: 'portal:document',
          payload: { document, editable: false }  // ← read-only
        }, '*');
        break;
      case 'app:share':
        showShareDialog(payload);
        break;
      case 'app:download':
        triggerDownload(payload);
        break;
    }
  });
});
```

**The key:** `editable: false` — documents are read-only in the portal's MCP Apps.

**Implication:** Users can view rendered PDFs, HTML, slides, but can't annotate, comment, or provide feedback inside the portal. They must:
1. View on portal
2. Download the file
3. Open on desktop
4. Edit locally
5. Re-upload or sync

**This is not collaboration.** This is consumption.

---

## Part 2: What "Notion for Academic Work" Actually Requires

### 2.1 Document Model (Not Just Outputs)

**Current spec:** Documents are *converted outputs* (PDF, HTML, email, slides).

```typescript
export interface Document {
  id: string;
  output_format: 'pdf' | 'html' | 'slides' | 'email' | 'latex' | 'arxiv';
  module: 'latex-mcp' | 'blog-mcp' | 'slides-mcp' | ...;
  status: 'pending' | 'converting' | 'success' | 'error';
  preview_url?: string;
  source_url?: string;
}
```

**What's missing:** The document *itself* — the editable source, not just the output.

**Notion-like workspace:** Users write and edit documents directly in the portal:
- **Research Notes:** Markdown + LaTeX math, inline citations, linked backlinks
- **Paper Drafts:** Sections (Abstract, Introduction, Methods, Results, Discussion), collaboratively editable
- **Lab Notebooks:** Time-stamped entries, figure attachments, reproducibility metadata
- **Datasets:** Tables, metadata, version history
- **Figures:** Embedded images, captions, figure references

Each of these is **primary source material**, not just a conversion output.

**Required data model:**
```typescript
export interface Document {
  id: string;
  type: 'research-note' | 'paper' | 'lab-notebook-entry' | 'dataset' | 'figure' | 'citation-collection';
  title: string;
  content: string;  // Editable markdown + math
  owner_id: string;
  created_at: string;
  updated_at: string;
  version: number;  // Collaboration version
  access_level: 'private' | 'group' | 'public';

  // Collaboration metadata
  collaborators: Array<{ user_id, role, added_at }>;
  edit_status: 'editing' | 'reviewed' | 'locked';

  // Conversion outputs (derived, not primary)
  outputs: Array<{
    module: 'latex-mcp' | 'blog-mcp' | ...;
    format: 'pdf' | 'html' | ...;
    status: 'pending' | 'success' | 'error';
    url?: string;
  }>;

  // Related resources
  citations: Citation[];
  figures: Figure[];
  linked_documents: string[];  // Backlinks
}
```

---

### 2.2 Editor Capabilities

**Current spec:** Read-only viewers + external editor disclaimer.

**Notion-like workspace:** Rich inline editor:
- **Block editor** (Notion-style, not monolithic text area)
  - Headings, paragraphs, lists, quotes, code blocks
  - LaTeX math blocks (`$$...$$`) and inline math (`$...$`)
  - Citation insertion (drag-and-drop citations into text)
  - Figure embedding and captioning
  - Table insertion
  - Linked document references (wiki-style: `[[Paper Title]]`)

- **Real-time collaboration** (v1.1+)
  - CRDT backend (Automerge, Yjs)
  - Live cursors showing who's editing where
  - Comments and resolved discussions

- **Version history**
  - Diff view (compare v1 vs. v2)
  - Restore previous version
  - Track who made which changes

---

### 2.3 Organization Model

**Current spec:** Groups > Projects > Documents (one level of nesting).

**Notion-like workspace:** Flexible organization:
- **Folders** (hierarchical tree)
- **Tags** (non-hierarchical labels, cross-cutting)
- **Collections** (linked database: "All papers by author X")
- **Custom views** ("Papers published in 2025", "Lab notebook entries from January")

**Required data model:**
```typescript
export interface Folder {
  id: string;
  parent_id?: string;  // For nesting
  name: string;
  group_id: string;
  documents: Document[];  // Inline for flat list, or separate query
}

export interface Tag {
  id: string;
  group_id: string;
  name: string;
  color?: string;
}

export interface DocumentTag {
  document_id: string;
  tag_id: string;
}

export interface LinkedDatabase {
  id: string;
  group_id: string;
  name: string;
  filter: {
    type?: string;  // paper, note, dataset
    tags?: string[];
    author_id?: string;
    date_range?: { start: string; end: string };
  };
  sort?: { field: string; order: 'asc' | 'desc' };
  view_type: 'list' | 'grid' | 'timeline' | 'kanban';
}
```

---

### 2.4 Minimum Viable "Notion for Academic Work" (v0.1)

**Phase 0 (v0.1): Stateful workspace, local editing, group browsing**
- Editable documents (markdown + math)
- Folders + tags for organization
- Comment threads on documents (async)
- Public/private/group-only access control
- Version history (basic: store snapshots, show who edited what)

**Phase 1 (v1.1): Real-time collaboration**
- Live cursors and typing indicators
- CRDT-based conflict-free editing
- Inline comments with real-time notifications

**Phase 2 (v2.0): Advanced discovery**
- Full-text search
- Custom linked databases and views
- Citation graph visualization

---

## Part 3: What "GitHub Pages for Group Outputs" Requires

### 3.1 The Group Site: What Is It?

**Current spec:** Landing page + group preview, but no mention of "group site" as a distinct entity.

**Product vision:** Every research group gets a **public-facing URL** (e.g., `portal.zenithscience.org/groups/biology-101`) where their work is published.

**Required:** Group sites must be:
- **Discoverable** — Listed on the portal's group directory, searchable
- **Public by default** — Papers, datasets, preprints, blog posts automatically published
- **Curated** — Group owner controls what appears on the site (can hide documents)
- **Attributed** — Clear authorship, institutional affiliation, funder acknowledgments
- **Citable** — DOI support (if integrating with arXiv, Zenodo, or publishing partners)

---

### 3.2 What Gets Published?

**Current spec:** Documents (PDFs, blogs, slides) + metadata.

**Reality check:** What should appear on a group's public site?

**Candidates:**
1. **Papers** (arXiv preprints, published PDFs)
2. **Datasets** (with README, metadata, download link)
3. **Notebooks** (Jupyter, Observable, RMarkdown)
4. **Blog posts** (blog-mcp outputs)
5. **Preprints** (lab-notebook-mcp thinking captures)
6. **Code repositories** (links to GitHub, snippets)
7. **Researcher profiles** (who works in the group)
8. **Project descriptions** (what is the group working on)
9. **News / Publications** (highlights, recent outputs)

**Required data model:**
```typescript
export interface PublishedItem {
  id: string;
  group_id: string;
  document_id: string;
  type: 'paper' | 'dataset' | 'notebook' | 'blog' | 'profile' | 'project-page';
  title: string;
  description?: string;
  published_at: string;
  authors: string[];
  keywords: string[];
  public_url: string;  // e.g., /groups/bio-101/papers/photosynthesis-2025
  metadata: {
    doi?: string;
    arxiv_id?: string;
    zenodo_id?: string;
  };
  visibility: 'public' | 'internal' | 'private';
}
```

---

### 3.3 How Does Publishing Work?

**Current spec:** Silent assumption — documents appear on portal, therefore on group site.

**Required:** Explicit publishing flow:
```
1. User creates document on portal
2. Document has metadata: title, authors, keywords
3. User clicks "Publish to Group Site" (or auto-publish if configured)
4. Document appears on group's public URL
5. Document gets a DOI (if group site integrates with Crossref/Zenodo)
6. Document is discoverable via search/directory
```

**Required endpoints:**
```
POST /api/groups/[id]/publish
  Request: { document_id, visibility: 'public' | 'internal', metadata: { doi?, arxiv_id? } }
  Response: { public_url: '/groups/bio-101/papers/...' }

GET /api/groups/[id]/published  # List of published items
PATCH /api/groups/[id]/published/[item_id]  # Update visibility, metadata
DELETE /api/groups/[id]/published/[item_id]  # Unpublish
```

---

### 3.4 Static Site Generation vs. Server-Rendered vs. Hybrid

**Current spec:** No mention of static site generation (SSG).

**Reality check:** What's the right architecture for group sites?

**Option A: Server-rendered (hybrid, current approach)**
- Portal serves `/groups/[id]` dynamically
- Pro: Easy to iterate, real-time updates, personalization
- Con: Harder to scale to 10K groups, requires active server

**Option B: Static site generation (SSG, GitHub Pages-like)**
- Group site generated as static HTML on publish
- Portal triggers build: markdown → HTML → static files
- Static files served from CDN or GitHub Pages
- Pro: Unlimited scale, fast, no server load
- Con: Slower to iterate, stale content until rebuild

**Option C: Hybrid (recommended)**
- Portal shows live group site (server-rendered)
- Export group site as static HTML on demand (for GitHub Pages integration)
- Group owner can self-host static site if desired

**Implementation:** Portal needs:
```typescript
export async function generateStaticGroupSite(groupId: string) {
  const group = await getGroup(groupId);
  const items = await getPublishedItems(groupId);

  const html = renderGroupTemplate(group, items);

  return {
    index_html: html,
    static_files: [
      '/assets/style.css',
      '/assets/js/interactive.js',
      ...itemAssets
    ],
    output_format: 'zip' | 'github-pages-ready'
  };
}
```

---

## Part 4: Social Layer Architecture

### 4.1 What Social Data Lives Where?

**Current spec:** Activity feed assumed to live in Gateway memory, but no schema defined.

**Reality check:** Social data is complex. What lives where?

**Option A: Gateway-only (current assumption, problematic)**
- Gateway stores Groups, Projects, Documents, ActivityEvents, Comments
- Problem: Gateway is not designed for this; needs major schema extension

**Option B: Portal + Gateway separation**
- Portal owns: Groups, Projects, Documents, Comments, Permissions, Collaborators
- Gateway owns: Tool invocations, chat history, MCP server status
- Sync: Portal syncs document outputs *to* Gateway after conversion (read-only)

**Option C: Dedicated social service (best, but more complex)**
- Separate **Social Service** (e.g., another Go service, or PostgreSQL + API)
- Social Service owns: All of the above
- Portal: Client to Social Service
- Gateway: Remains tool orchestrator
- Desktop: Also client to Social Service (syncs from local + social via gateway)

**Recommendation:** Option B for v0.1, Option C for v1.0+

---

### 4.2 Activity Feed Events

**Current spec:** Limited event types (`user_joined`, `project_created`, `document_converted`, `member_added`, `settings_updated`).

**What's missing:** For a collaborative workspace, add:

```typescript
export type ActivityEventType =
  // Current
  | 'user_joined'
  | 'project_created'
  | 'document_converted'
  | 'member_added'
  | 'member_removed'
  | 'member_role_changed'
  | 'settings_updated'
  | 'group_created'

  // NEW for stateful workspace
  | 'document_created'    // User created a new editable document on portal
  | 'document_edited'     // User edited a document (fires frequently)
  | 'document_published'  // User published document to group site
  | 'comment_added'       // User commented on a document
  | 'comment_resolved'    // Discussion thread resolved
  | 'document_shared'     // User shared document with specific users
  | 'collaboration_started' // User started collaborating on a document
  | 'version_created'     // New version of document (for history tracking)
  | 'document_deleted'    // User deleted a document
  | 'citation_added'      // User added citation to document
  | 'figure_added'        // User embedded a figure
  | 'tag_added'           // User tagged a document
;
```

**Impact:** Portal needs to store and query these events efficiently:
```
GET /api/groups/[id]/activity?type=document_edited&limit=50&offset=0
GET /api/groups/[id]/activity?actor_id=[user_id]&limit=50
GET /api/documents/[id]/activity  # Version history for a single doc
```

---

### 4.3 Discovery: How Do Users Find Groups?

**Current spec:** Group directory (`/groups`), but no discovery algorithm mentioned.

**Required:** For v0.1+:

```typescript
export interface GroupDiscovery {
  // Explicit discovery
  group_directory: Array<{
    name: string;
    description: string;
    members: number;
    public_documents: number;
    tags: string[];
  }>;

  // Filtered search
  GET /api/discover/groups?tags=biology&sort=newest  # By tag
  GET /api/discover/groups?search=photosynthesis    # Full-text search (v1.0+)
  GET /api/discover/groups?member_count=gte:10      # By size

  // Trending
  GET /api/discover/trending/groups?period=week     # Most active
  GET /api/discover/trending/documents?period=week  # Most viewed/cited

  // Following
  GET /api/user/following  # Groups user follows
  POST /api/groups/[id]/follow  # Subscribe to group activity
}
```

---

### 4.4 Ghost Town Prevention

**Critical question:** How do you prevent the portal from being a ghost town on day one?

**Answer: Bootstrap with content and collaboration incentives**

**For v0.1:**
- Invite links (✓ spec has this) — seed with known groups
- Educators create class cohorts — pre-populate with students
- Showcase papers/projects — curated list of high-quality outputs
- Activity feed defaults to "all groups" if user new — see what others are doing
- Email digest — weekly summary of group activity (for members)

**For v1.0+:**
- Follow/discover interface — personalized feed based on tags/interests
- Trending groups/papers — algorithmic discovery
- Collaborative features — comment threads attract engagement
- Integration with social (Twitter, Slack, email) — content sharing

---

## Part 5: The Web Portal as Independent Workspace

### 5.1 What the Portal Should Own

**Reframe:** The web portal is not a thin client. It is a **first-class workspace** where researchers conduct the full workflow:

1. **Write** — Create/edit documents directly on portal
2. **Organize** — Use folders, tags, collections
3. **Collaborate** — Comment, share, iterate with team
4. **Convert** — Export to PDF, HTML, slides, email (via ZenSci servers)
5. **Publish** — Publish papers to group site, share with outside researchers
6. **Discover** — Browse other groups' work, follow trends

**Data portal owns:**
- Documents (markdown source + editable state)
- Folders, tags, collections
- Comments and discussion threads
- Collaborators and access control
- Version history
- Published items and group site metadata

**Data portal orchestrates (via Gateway):**
- ZenSci conversions (markdown → PDF, HTML, etc.)
- LLM-powered tools (writing assistance, citations, abstract generation)
- Chat/reasoning (if integrated)

---

### 5.2 Canonical Workflow: Desktop + Portal Sync

**Corrected model (replaces thin-client model):**

```
Desktop App (Tauri)               Web Portal (SvelteKit)
─────────────────                 ──────────────────

User writes paper.md locally
  ↓
Desktop stores locally (SQLite)
  ↓
User clicks "Convert to PDF"
  → Calls ZenSci via Gateway
  ← Receives PDF
  ↓
User clicks "Upload to Portal"
  ↓
Portal stores paper.md as NEW document (or syncs version)
  ← Portal shows paper in group workspace

Meanwhile, group member (Bob) is on Portal:
  ├─ Views the paper (read-only rendering)
  ├─ Adds comment thread: "Add section on experimental validation"
  └─ Shares with Alice: "Please review my comments"

Alice sees comment notification:
  ├─ Reviews comment on Portal
  ├─ Decides to edit locally (desktop has rich editor too)
  ├─ Downloads paper.md from Portal
  ├─ Edits locally
  ├─ Uploads revised version
  ├─ Portal shows new version (v2) with diff
  └─ Comments carry over (resolved: Alice addressed feedback)

Portal publishes:
  ├─ User clicks "Publish to Group Site"
  ├─ Paper appears at `/groups/biology-101/papers/paper-title`
  ├─ Gets DOI
  └─ Shared with world
```

**Key difference:** Documents exist on the Portal as *editable source* (not just outputs). Desktop and Portal are sync'd, not separate.

---

### 5.3 Sync Strategy: Conflict Resolution

**Question: What if Alice edits on desktop and Bob edits on portal simultaneously?**

**Strategies:**

**Strategy A: Last-write-wins (simple, v0.1)**
- Desktop: User edits locally, clicks "Save to Portal"
- Portal: User edits in browser, clicks "Save"
- Conflict: Last save wins (shows warning: "This version is newer on desktop/portal")
- Drawback: Can lose work

**Strategy B: CRDT (Conflict-free, v1.1+)**
- Use Automerge or Yjs CRDT library
- Desktop syncs changes to Portal (not full document, just ops)
- Portal syncs changes to Desktop (same)
- Conflicts automatically resolved (both users' changes preserved)
- Drawback: Requires sophisticated backend

**Recommendation:** Start with Strategy A (v0.1), migrate to Strategy B (v1.1) if collaboration proves essential.

**Implementation (v0.1):**
```typescript
export interface DocumentVersion {
  id: string;
  document_id: string;
  version: number;
  content: string;
  author_id: string;
  edited_on: 'desktop' | 'portal';  // Where the edit happened
  created_at: string;
  conflict_with?: string;  // If conflict, reference previous version
}

// On upload from desktop:
POST /api/documents/[id]/sync
  Request: { content: string, version: number, base_version: number }
  Response: {
    merged: boolean,
    conflict?: {
      portal_version: number,
      portal_edited_at: string,
      desktop_version: number
    },
    new_version: number
  }
```

---

## Part 6: v0.1 Full Architecture (Revised)

### 6.1 Data Model Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│ Portal Database (PostgreSQL or SQLite, portal-owned)             │
├─────────────────────────────────────────────────────────────────┤
│ DOCUMENTS                                                         │
│  ├─ Document (id, title, content, owner_id, type, status)       │
│  ├─ DocumentVersion (id, doc_id, version, content, author, ts)  │
│  ├─ DocumentTag (doc_id, tag_id)                                │
│  ├─ Collaborator (doc_id, user_id, role)                        │
│  └─ Comment (id, doc_id, text, author_id, timestamp, resolved)  │
│                                                                   │
│ GROUPS & ORGANIZATION                                            │
│  ├─ Group (id, name, description, owner_id)                     │
│  ├─ Folder (id, parent_id, name, group_id)                      │
│  ├─ GroupMember (group_id, user_id, role, joined_at)            │
│  ├─ Tag (id, group_id, name, color)                             │
│  └─ LinkedDatabase (id, group_id, filter, sort, view_type)      │
│                                                                   │
│ PUBLISHING                                                        │
│  ├─ PublishedItem (id, doc_id, group_id, type, public_url)      │
│  └─ PublicationMetadata (item_id, doi, arxiv_id, zenodo_id)     │
│                                                                   │
│ ACTIVITY & SOCIAL                                                │
│  ├─ ActivityEvent (id, group_id, type, actor_id, data)          │
│  ├─ InviteToken (token, group_id, created_at, expires_at)       │
│  └─ UserFollow (user_id, group_id, started_at)                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ AgenticGateway (Existing)                                         │
├─────────────────────────────────────────────────────────────────┤
│ TOOL ORCHESTRATION (unchanged)                                   │
│  ├─ Chat/completions API                                        │
│  ├─ Tool registry                                               │
│  ├─ MCP server management                                       │
│  └─ Skill system                                                │
│                                                                   │
│ CONVERSIONS (Interface to Portal)                                │
│  ├─ POST /v1/convert { markdown, module: 'latex-mcp', ... }     │
│  └─ Returns: { pdf_url, html_url, latex_url, ... }              │
│                                                                   │
│ AUTH (New or inherited from external provider)                   │
│  └─ JWT token validation                                        │
└─────────────────────────────────────────────────────────────────┘
```

---

### 6.2 Key Routes (Stateful Workspace)

**Editable document routes:**
```
GET  /groups/[id]/documents                # List documents in group
POST /groups/[id]/documents                # Create new document
GET  /groups/[id]/documents/[doc_id]       # View/edit document
PATCH /groups/[id]/documents/[doc_id]      # Save document
GET  /groups/[id]/documents/[doc_id]/versions  # Version history
POST /groups/[id]/documents/[doc_id]/publish   # Publish to group site
```

**Collaboration routes:**
```
POST /groups/[id]/documents/[doc_id]/comments     # Add comment
GET  /groups/[id]/documents/[doc_id]/comments     # List comments
PATCH /groups/[id]/documents/[doc_id]/comments/[cid]  # Resolve thread
POST /groups/[id]/documents/[doc_id]/share        # Share with users
POST /groups/[id]/documents/[doc_id]/collaborate  # Add collaborator
```

**Organization routes:**
```
POST /groups/[id]/folders                  # Create folder
POST /groups/[id]/tags                     # Create tag
POST /groups/[id]/linked-databases         # Create linked view
```

**Discovery routes:**
```
GET  /discover/groups                      # Browse all groups
GET  /discover/groups/[id/papers           # Public papers
GET  /user/following                       # Groups user follows
POST /groups/[id]/follow                   # Subscribe to group
```

---

### 6.3 Why This Fixes the Thin-Client Problem

| Aspect | Thin-Client (Current) | Stateful Workspace (Revised) |
|--------|----------------------|------------------------------|
| **Data ownership** | Desktop app owns documents; portal displays | Portal owns canonical source; desktop syncs |
| **Editing** | Only on desktop | On portal + desktop, conflict resolution |
| **Collaboration** | Activity feed only | Comments, version history, shared editing |
| **Publishing** | Undefined | Explicit publish flow → group site |
| **Sync** | One-way (desktop → portal) | Two-way (desktop ↔ portal) |
| **Database** | None (assumes Gateway) | Portal owns PostgreSQL (or SQLite) |
| **Portal role** | Viewer | Workspace |
| **Ghost town risk** | High (no way to create content) | Low (native editing + collaboration) |

---

## Part 7: Implementation Path (Revised)

### 7.1 Phase 1–2 (Weeks 1–6): Foundational Workspace

**Week 1–2: Portal database + authentication**
- Set up PostgreSQL (or SQLite v3 with WAL)
- Define schema: Document, Group, User, etc.
- Implement auth (JWT from Gateway, or separate auth service)
- Test database persistence

**Week 3–4: Basic CRUD + editing**
- Implement document create/read/update/delete
- Build rich text editor (Svelte component, e.g., Tiptap)
- Support markdown + LaTeX math
- Implement version history (store snapshots)

**Week 5–6: Groups + collaboration framework**
- Implement Group management (create, invite, member roles)
- Implement Comment threads
- Implement DocumentShare (permissions)
- Build activity feed

---

### 7.2 Phase 3–4 (Weeks 7–10): Conversion + Publishing

**Week 7–8: ZenSci integration**
- Portal calls Gateway to convert markdown → PDF, HTML, etc.
- Store converted outputs as linked to Document
- Build document viewer (MCP App iframe)

**Week 9–10: Publishing & discovery**
- Implement "Publish to Group Site" flow
- Generate public URLs for published documents
- Build group site (list of published items)
- Implement group discovery/directory

---

### 7.3 Phase 5 (Weeks 11–13): Polish + testing

- Mobile responsiveness
- Comprehensive testing (unit, integration, E2E)
- Performance optimization
- Deployment (Docker, staging, production)

---

## Part 8: Key Decisions Required

**Before spec revision, Cruz must decide:**

1. **Portal database ownership:** Does the portal own a PostgreSQL instance, or does the Gateway expand to include social schema?

2. **Sync strategy:** Last-write-wins (v0.1) or CRDT (v1.1)? This determines backend complexity.

3. **Publishing model:** Is publishing explicit ("Publish to Group Site" button), or automatic (all documents public by default, unless marked private)?

4. **Collaboration scope:** Can users comment on documents? Can they share with specific users? Or is all collaboration async via activity feed only?

5. **Desktop integration:** How tightly should portal and desktop sync? Full two-way, or portal reads Desktop documents (via Gateway)?

6. **Auth architecture:** Is the Gateway an identity provider, or is there a separate auth service?

---

## Part 9: Summary & Recommendation

### The Core Reframe

**From:** "Web portal is a thin client that displays what the desktop app creates."

**To:** "Web portal is a full stateful workspace where researchers work directly in canonical source, independently of the desktop."

---

### What Changes

1. **Portal owns its database** (PostgreSQL or SQLite), not just a proxy
2. **Documents are editable on the portal** (not just viewing converted outputs)
3. **Collaboration happens on the portal** (comments, version history, shared editing)
4. **Publishing is explicit** (portal → group site, not automatic)
5. **Desktop and portal sync** (two-way, with conflict resolution)
6. **Discovery is built in** (group directory, trending, following)

---

### What Stays

- SvelteKit architecture ✓
- SSR for discovery routes ✓
- MCP App rendering (with `editable: true` for some apps) ✓
- Graceful degradation ✓
- Responsive design ✓

---

### Implementation Effort

- **v0.1 (thin client):** 8–10 weeks
- **v0.1 (stateful workspace):** 12–16 weeks

**The investment is worth it.** A stateful workspace is a defensible product. A thin client is a feature, not a product.

---

**Scout prepared by:** Claude (Strategic)
**Date:** 2026-02-18
**Status:** Ready for decision session with Cruz
**Next step:** Schedule 1-hour architecture review to lock decisions on database ownership, sync strategy, and collaboration scope. Then proceed to Phase 1 implementation.
