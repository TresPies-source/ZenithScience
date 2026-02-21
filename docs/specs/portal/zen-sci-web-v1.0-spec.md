# zen-sci-web v1.0: The Common Room
## Hosted Web Portal (Social/Discovery Layer)

**Author:** Architecture Planning (TresPiesDesign.com)
**Status:** Specification (Pre-Implementation)
**Created:** 2026-02-18
**Grounded In:** ZenSci Phase 0–3 complete, AgenticGateway v1.0.0 production-ready, strategic portal scout (2026-02-18)

---

## 1. Vision

> **The entry point for knowledge workers**: Browse projects, join research groups, see your cohort's outputs, and share discoveries—before installing the desktop app.

zen-sci-web is the **web/cloud face** of ZenSci, positioned as the "Connect" surface. It serves three core audiences:
1. **Students & researchers** discovering projects and research groups
2. **Educators** managing class cohorts and viewing student outputs
3. **New users** onboarding before installing the Tauri desktop app

Unlike the desktop app (local-first, full conversion pipeline), zen-sci-web focuses on **async group collaboration, discovery, and sharing**. Users can browse group projects, search for outputs by module type (LaTeX, blog, grant), and receive invitation links to join groups. The app renders MCP Apps (Phase 4) in browser iframes—same visual experience as the desktop, but running against the shared cloud Gateway.

---

## 1.5 Current State: What We Build On

### Existing Infrastructure
- **AgenticGateway v1.0.0** (Go, production-ready)
  - OpenAI-compatible `/v1/chat/completions` API
  - Tool registry + orchestration (DAG-based execution)
  - MCP integration (14 Dojo tools via stdio, extensible to Composio/custom)
  - SQLite-based memory management
  - OTEL observability + Langfuse integration
  - 8 model providers (Anthropic, OpenAI, Google, Groq, Mistral, Kimi, DeepSeek, Ollama)
  - JWT auth + API key support
  - 45 API endpoints (health, chat, admin, tools, agents, memory)

- **ZenSci v0.1–0.4** (TypeScript MCP servers, 493 tests passing)
  - **6 MCP servers** (all Phase 0–3 implemented):
    - `latex-mcp` (v0.1) — Markdown → PDF with math/citations
    - `blog-mcp` (v0.2) — Markdown → HTML with SEO/RSS
    - `slides-mcp` (v0.3) — Markdown → Beamer/Reveal.js
    - `newsletter-mcp` (v0.3) — Markdown → MJML email
    - `grant-mcp` (v0.4) — Markdown → NIH/NSF proposals
    - `paper-mcp` (v0.5) — Markdown → IEEE/ACM/arXiv LaTeX
  - **Phase 4 (MCP Apps)** specced but not yet implemented—6 companion apps (PDF viewers, HTML previews, compliance dashboards)
  - **packages/core** — Shared parsing, citation, validation, math handling
  - **packages/sdk** — MCP server factory, pipeline utilities

### Key Gateway API Endpoints Used by zen-sci-web
| Endpoint | Method | Purpose | Auth |
|----------|--------|---------|------|
| `/health` | GET | Health check | None |
| `/v1/chat/completions` | POST | Chat/reasoning (for discovery) | JWT |
| `/v1/gateway/tools` | GET | List available tools | JWT |
| `/admin/mcp/status` | GET | Check if 6 ZenSci servers online | JWT |
| (Future: `/v1/gateway/memory/*`) | GET/POST | Group/project memory | JWT |

### SvelteKit Foundation
- **SvelteKit 2.0+** — SSR + SPA hybrid
- **Vite 5.0+** — Build tool
- **TypeScript** — Strict mode
- **TailwindCSS** — Styling (optional; semantic HTML preferred)
- **@sveltejs/adapter-node** — Node.js server deployment

---

## 2. Goals & Success Criteria

### Primary Goals

1. **Enable group discovery & onboarding** — New users receive invite links, see group projects, understand ZenSci's value before desktop install.

2. **Async group collaboration** — Groups see activity feeds (who created what, what outputs were produced), not real-time presence. Foundation for future real-time collab in v1.1.

3. **MCP App rendering in browser** — Phase 4 MCP Apps (PDF viewer, HTML preview, compliance dashboard) load in iframes, same UX as desktop Tauri webview. Users render LaTeX PDFs, blog posts, grant proposals from web.

4. **Group & project management** — Create groups, invite members, assign roles (owner/member/viewer), manage projects within groups. Admin views for educators.

5. **Document sharing & discovery** — Share ZenSci output links (e.g., `/groups/biology-101/projects/photosynthesis-paper/document/draft-1`). Recipients see rendered document + metadata, prompted to join group or download desktop app.

6. **Seamless desktop handoff** — Invitation flow: email → web portal → desktop app deep link. User discovers on web, then installs desktop to contribute locally.

7. **Gateway health visibility** — Dashboard shows status of 6 ZenSci MCP servers. If a server is offline, users see "LaTeX conversion unavailable" gracefully.

### Success Criteria

- ✅ **Auth flow:** Users log in via Gateway JWT (email/password or SSO). Session persists in localStorage + secure cookies. Logout clears both.
- ✅ **Group CRUD:** Create/read/update/delete groups. Add members, assign roles, remove users. Admin-only edit settings. Member/viewer can read only.
- ✅ **Project listing:** Groups display list of projects (markdown documents + ZenSci outputs). Filter by module type (LaTeX, blog, grant, etc.). Sort by date, author, name.
- ✅ **Activity feed:** Group shows recent events (user X created project Y, user Z converted to PDF, etc.). 50-item paginated feed. No real-time updates; polling or SSE optional for v1.1.
- ✅ **MCP App rendering:** Phase 4 PDF viewer, HTML preview, compliance dashboard load in iframes. Users can interact (pan/zoom PDFs, click links in HTML, toggle compliance sections).
- ✅ **Invitation flow:** Generate shareable link (48-hour token). Non-user receives email → clicks link → lands on `/invite/[token]` → sees group preview → signup form → email verified → joins group. Desktop app detects group context on launch (deep link).
- ✅ **Document sharing:** User shares `/groups/bio-101/projects/photosynthesis/document/draft-1` → recipient sees rendered output + metadata + "Join group" button.
- ✅ **Profile pages:** User profile shows document history, group memberships, roles, account settings.
- ✅ **Gateway status dashboard:** Shows health of 6 ZenSci servers + AgenticGateway itself. Red/yellow/green indicators. Error messages if conversion fails.
- ✅ **Mobile responsive:** Works on 320px–1920px screens. Touch-friendly buttons, readable text. Collapse navigation on mobile.
- ✅ **SSR for discovery routes:** `/` (landing), `/groups` (directory), `/groups/[id]` (group public page) all SSR for SEO + fast initial load. `/groups/[id]/projects` onwards is SPA (authenticated).
- ✅ **Fast page load:** Initial HTML delivery < 3s on 4G. Interactive within < 5s. Lazy-load MCP Apps on demand.
- ✅ **No errors on missing Gateway:** If Gateway is unavailable, show "Maintenance mode" gracefully. Core UI still loads; data fetch fails with user-friendly message.
- ✅ **Comprehensive test suite:** Unit tests for components, integration tests for auth/group flows, E2E tests for critical paths (invite → join → view document).

### Non-Goals (Out of Scope for v0.1)

- ❌ **Real-time collaboration.** Async only. Real-time editing deferred to v1.1 (Rust backend + Automerge CRDT).
- ❌ **Document editor.** Users edit markdown locally (desktop app) or via external editors. Web portal is viewer/curator only.
- ❌ **Full-text search.** Simple filter by name/tag/date. Full-text search (Tantivy) is Rust backend feature, v1.0+ portal.
- ❌ **PDF upload/attachment API.** Users convert via desktop app or request conversion via chat. Attachments are out of scope.
- ❌ **Mobile apps (iOS/Android).** Web portal only (responsive web). Native apps post-v1.
- ❌ **Offline mode.** Requires active Gateway connection (cloud-first). Desktop app handles offline-first.
- ❌ **Custom branding/white-label.** One design. Custom logos/colors future release.
- ❌ **Email notifications.** Invitations only. Activity digest/real-time notifications deferred.
- ❌ **Analytics/usage tracking.** Monitoring is Gateway/backend concern. Portal logs access.
- ❌ **Rate limiting per user.** Gateway enforces budgets; portal trusts Gateway's enforcement.

---

## 3. Technical Architecture

### 3.1 System Overview

**Browser** (user's device) ↔ **SvelteKit app** (Node.js server) ↔ **AgenticGateway** (Go, cloud) ↔ **ZenSci servers** (TypeScript MCP, cloud)

```
┌─────────────────────────────────────────────────────────────────┐
│ Browser (User Device)                                            │
│  - React/Svelte UI                                               │
│  - localStorage (JWT, user prefs)                                │
│  - MCP App iframes (ui://latex-mcp/preview.html, etc.)           │
└───────────┬─────────────────────────────────────────────────────┘
            │ HTTPS (TLS 1.3+)
            │ JSON payloads
            │ SSE streams (optional)
            ▼
┌─────────────────────────────────────────────────────────────────┐
│ SvelteKit Server (Node.js)                                       │
│ ├─ SSR routes (/, /groups, /groups/[id], etc.)                  │
│ ├─ SPA routes (/groups/[id]/projects, /profile, etc.)           │
│ ├─ API proxy layer (`/api/*`)                                   │
│ │  ├─ `/api/auth/login` → Gateway JWT                           │
│ │  ├─ `/api/auth/logout` → clear session                        │
│ │  ├─ `/api/groups` → fetch groups from Gateway memory          │
│ │  ├─ `/api/groups/[id]/projects` → list projects              │
│ │  ├─ `/api/groups/[id]/activity` → activity feed              │
│ │  ├─ `/api/invites` → generate/validate invite tokens         │
│ │  └─ `/api/mcp-apps` → Phase 4 app manifest                   │
│ ├─ Session middleware (JWT validation)                          │
│ ├─ CSRF protection (SvelteKit form actions)                     │
│ └─ Static file serving (MCP App bundles)                        │
└───────────┬─────────────────────────────────────────────────────┘
            │ HTTP to Gateway
            │ Authorization: Bearer {JWT}
            │
            ▼
┌─────────────────────────────────────────────────────────────────┐
│ AgenticGateway v1.0.0 (Go, Production)                           │
│  /health, /v1/chat/completions, /admin/mcp/status               │
│  ├─ Memory Manager (SQLite)                                      │
│ ├─ MCP Host Manager (14 Dojo tools)                              │
│  ├─ Tool Registry (33 built-in tools)                            │
│  ├─ Provider Layer (8 model providers)                           │
│  ├─ Orchestration Engine (DAG execution)                         │
│  └─ Skill System (44 skills)                                     │
└───────────┬─────────────────────────────────────────────────────┘
            │ Stdio/SSE
            │
            ▼
┌─────────────────────────────────────────────────────────────────┐
│ ZenSci MCP Servers (TypeScript, 6 servers)                       │
│  - latex-mcp, blog-mcp, slides-mcp                               │
│  - newsletter-mcp, grant-mcp, paper-mcp                          │
│  ├─ Each server: MarkdownParser → Validation → Conversion       │
│  └─ Phase 4: companion apps (PDF viewer, HTML preview, etc.)    │
└─────────────────────────────────────────────────────────────────┘
```

**Key flows:**
1. **Authentication:** Browser POST `/api/auth/login` → SvelteKit calls Gateway `/v1/chat/completions` with auth intent → returns JWT → stored in secure cookie + localStorage → all subsequent requests include `Authorization: Bearer {JWT}`
2. **Group discovery:** Browser GET `/groups` → SvelteKit calls Gateway memory API → returns group list → renders SSR landing page + SPA filters
3. **MCP App rendering:** Browser fetches `/api/mcp-apps/[module]/manifest` → SvelteKit returns app HTML URL → browser loads `<iframe src="ui://[module]/[app].html" />` → MCP App runs in iframe, communicates with parent via `postMessage`

### 3.2 Application Structure

**SvelteKit Route Layout:**

```
src/
├── routes/
│   ├── +layout.svelte              # Root layout (nav, auth guard)
│   ├── +layout.server.ts           # Load user session, Gateway health
│   ├── +page.svelte                # Landing page (public, SSR)
│   ├── +page.server.ts             # Load featured groups, public projects
│   │
│   ├── login/
│   │   ├── +page.svelte            # Login form (public)
│   │   └── +page.server.ts         # POST login → Gateway JWT
│   │
│   ├── register/
│   │   ├── +page.svelte            # Signup form (public)
│   │   └── +page.server.ts         # POST register → Gateway create user
│   │
│   ├── invite/
│   │   ├── [token]/
│   │   │   ├── +page.svelte        # Invite preview (public, show group)
│   │   │   └── +page.server.ts     # Validate token, load group preview
│   │
│   ├── (auth)/                     # Group: requires JWT auth
│   │   ├── +layout.svelte          # Auth guard layout
│   │   ├── +layout.server.ts       # Verify JWT, redirect to login if invalid
│   │   │
│   │   ├── groups/
│   │   │   ├── +page.svelte        # List all groups user is member of
│   │   │   ├── +page.server.ts     # GET /api/groups
│   │   │   ├── +page.svelte        # New group form
│   │   │   │
│   │   │   └── [id]/
│   │   │       ├── +page.svelte    # Group workspace (activity feed, members)
│   │   │       ├── +page.server.ts # GET /api/groups/[id], activity
│   │   │       ├── +layout.svelte  # Group nav (Projects, Members, Settings)
│   │   │       │
│   │   │       ├── projects/
│   │   │       │   ├── +page.svelte        # Project list (filter by module)
│   │   │       │   ├── +page.server.ts     # GET /api/groups/[id]/projects
│   │   │       │   │
│   │   │       │   └── [project_id]/
│   │   │       │       ├── +page.svelte    # Project detail (docs, outputs)
│   │   │       │       ├── +page.server.ts # GET project metadata
│   │   │       │       │
│   │   │       │       └── documents/
│   │   │       │           └── [doc_id]/
│   │   │       │               ├── +page.svelte    # Document viewer (MCP App iframe)
│   │   │       │               └── +page.server.ts # GET doc metadata + app URL
│   │   │       │
│   │   │       ├── members/
│   │   │       │   ├── +page.svelte        # Members list, invite form
│   │   │       │   └── +page.server.ts     # POST /api/groups/[id]/invite
│   │   │       │
│   │   │       └── settings/
│   │   │           └── +page.svelte        # Group settings (owner only)
│   │   │
│   │   ├── profile/
│   │   │   └── [user_id]/
│   │   │       ├── +page.svelte            # User profile (document history, memberships)
│   │   │       └── +page.server.ts         # GET user profile from Gateway
│   │   │
│   │   └── (shared)/
│   │       ├── settings/
│   │       │   └── +page.svelte            # Account settings, logout
│   │       │
│   │       └── notifications/
│   │           └── +page.svelte            # Invite notifications, activity digest
│   │
│   └── api/
│       ├── auth/
│       │   ├── login/+server.ts            # POST email/password → Gateway
│       │   ├── logout/+server.ts           # POST → clear cookies
│       │   └── me/+server.ts               # GET current user from Gateway
│       │
│       ├── groups/
│       │   ├── +server.ts                  # GET list, POST create
│       │   └── [id]/
│       │       ├── +server.ts              # GET group, PATCH update (owner only)
│       │       ├── projects/+server.ts     # GET projects, POST create (member+)
│       │       ├── activity/+server.ts     # GET activity feed
│       │       ├── members/+server.ts      # GET members, POST add, DELETE remove
│       │       └── invites/+server.ts      # POST generate invite, GET pending
│       │
│       ├── invites/
│       │   └── [token]/
│       │       ├── +server.ts              # GET validate, POST claim
│       │       └── preview/+server.ts      # GET preview (no auth)
│       │
│       ├── documents/
│       │   └── [doc_id]/+server.ts         # GET metadata, render status
│       │
│       ├── mcp-apps/
│       │   ├── [module]/
│       │   │   ├── manifest/+server.ts     # GET app manifest (resourceUri, appName)
│       │   │   └── health/+server.ts       # GET server health (is module online?)
│       │   │
│       │   └── status/+server.ts           # GET health of all 6 ZenSci servers
│       │
│       └── gateway/
│           └── health/+server.ts           # GET AgenticGateway health
│
├── lib/
│   ├── server/
│   │   ├── gateway.ts                      # Gateway API client
│   │   ├── auth.ts                         # JWT validation, session helpers
│   │   ├── groups.ts                       # Group CRUD helpers
│   │   ├── memory.ts                       # Memory API helpers (activity, metadata)
│   │   └── mcp-apps.ts                     # MCP App manifest resolver
│   │
│   ├── client/
│   │   ├── stores.ts                       # Svelte stores (auth, groups, user)
│   │   ├── api.ts                          # Client API helpers
│   │   └── mcp-app-bridge.ts               # postMessage bridge for iframe apps
│   │
│   └── types/
│       ├── gateway.ts                      # Gateway API types (responses)
│       ├── portal.ts                       # Portal types (User, Group, Project, Document, ActivityEvent, InviteToken)
│       └── mcp-apps.ts                     # MCP App types (AppManifest, ResourceUri)
│
├── components/
│   ├── Nav.svelte                          # Main navigation bar
│   ├── Auth/
│   │   ├── LoginForm.svelte                # Email/password login
│   │   └── RegisterForm.svelte             # Signup form
│   │
│   ├── Groups/
│   │   ├── GroupCard.svelte                # Group summary card
│   │   ├── GroupList.svelte                # List of groups
│   │   ├── GroupSettings.svelte            # Group admin panel
│   │   └── InviteMemberForm.svelte         # Invite form
│   │
│   ├── Projects/
│   │   ├── ProjectCard.svelte              # Project summary card
│   │   ├── ProjectList.svelte              # List of projects (filterable)
│   │   └── ProjectDetail.svelte            # Project view with docs
│   │
│   ├── Documents/
│   │   ├── DocumentViewer.svelte           # MCP App iframe wrapper
│   │   ├── DocumentMetadata.svelte         # Title, author, module, date
│   │   └── DocumentList.svelte             # List of outputs in project
│   │
│   ├── Activity/
│   │   ├── ActivityFeed.svelte             # Activity stream (paginated)
│   │   ├── ActivityItem.svelte             # Single activity event (user X created Y)
│   │   └── ActivityFilter.svelte           # Filter by event type
│   │
│   ├── Profile/
│   │   ├── UserCard.svelte                 # User summary (avatar, name, role)
│   │   ├── UserProfile.svelte              # Full profile (history, memberships)
│   │   └── DocumentHistory.svelte          # User's documents across groups
│   │
│   ├── Gateway/
│   │   ├── ServerStatus.svelte             # 6 ZenSci servers + Gateway health
│   │   ├── ServerHealthItem.svelte         # Single server (name, status, error)
│   │   └── DowntimeNotice.svelte           # "Maintenance mode" banner
│   │
│   └── Shared/
│       ├── Modal.svelte                    # Reusable modal dialog
│       ├── Toast.svelte                    # Toast notifications
│       ├── Pagination.svelte               # Paginated lists
│       ├── Loading.svelte                  # Spinner
│       └── ErrorBoundary.svelte            # Error fallback UI
│
├── hooks.server.ts                         # JWT validation, session loading
├── hooks.client.ts                         # Client-side auth guards
├── app.css                                 # Global styles (Tailwind or vanilla CSS)
└── app.html                                # HTML template
```

**Key structural choices:**
- **`(auth)` route group:** All authenticated routes live here. `+layout.server.ts` validates JWT; invalid sessions redirect to `/login`.
- **`+page.server.ts` vs `+page.svelte`:** Server loads data via Gateway API before rendering (SSR). Page component displays loaded data (reactive, interactive).
- **`/api/*` endpoints:** Server-side proxies. SvelteKit handles auth context (JWT in `Authorization` header). Browser calls `/api/groups` (SvelteKit handles Gateway auth), not direct Gateway API calls.

### 3.3 Group & Project Data Model

**TypeScript Interfaces** (source of truth for all types):

```typescript
// lib/types/portal.ts

export interface User {
  id: string;                       // UUID from Gateway
  email: string;
  username: string;
  display_name?: string;
  avatar_url?: string;
  roles: string[];                  // ['viewer', 'member', 'owner', 'admin']
  created_at: string;               // ISO 8601
  last_login_at?: string;
}

export interface GroupRole {
  role: 'owner' | 'member' | 'viewer';
  can_edit_settings: boolean;       // owner only
  can_add_members: boolean;         // owner + member
  can_create_projects: boolean;     // owner + member
  can_delete_group: boolean;        // owner only
}

export interface Group {
  id: string;                       // UUID
  name: string;
  description?: string;
  owner_id: string;                 // User ID
  members: Array<{
    user_id: string;
    role: 'owner' | 'member' | 'viewer';
    joined_at: string;
  }>;
  project_count: number;
  created_at: string;
  updated_at: string;
  public: boolean;                  // discoverable?
  tags?: string[];                  // ['biology', '101', 'spring-2026']
}

export interface Project {
  id: string;                       // UUID
  group_id: string;
  name: string;
  description?: string;
  created_by: User;
  created_at: string;
  updated_at: string;
  documents: Document[];            // embedded for project detail view
  status: 'draft' | 'in-progress' | 'completed';
}

export interface Document {
  id: string;                       // UUID
  project_id: string;
  title: string;
  source_format: 'markdown';        // future: 'docx', 'latex', etc.
  output_format: 'pdf' | 'html' | 'slides' | 'email' | 'latex' | 'arxiv';
  module: 'latex-mcp' | 'blog-mcp' | 'slides-mcp' | 'newsletter-mcp' | 'grant-mcp' | 'paper-mcp';
  status: 'pending' | 'converting' | 'success' | 'error';
  error_message?: string;
  created_at: string;
  updated_at: string;
  created_by: User;
  preview_url?: string;             // Where to fetch converted output
  source_url?: string;              // If markdown file is accessible
  metadata: {
    word_count?: number;
    page_count?: number;
    math_expressions?: number;
    citations?: number;
  };
}

export interface ActivityEvent {
  id: string;
  group_id: string;
  type: 'user_joined' | 'project_created' | 'document_converted' | 'member_added' | 'settings_updated';
  actor: User;
  timestamp: string;               // ISO 8601
  data: Record<string, unknown>;   // event-specific data
  // Examples:
  // { type: 'project_created', data: { project_id, project_name } }
  // { type: 'document_converted', data: { doc_id, module, status } }
  // { type: 'member_added', data: { member_id, member_name, role } }
}

export interface InviteToken {
  token: string;                   // Random 32-char token
  group_id: string;
  invited_by: User;
  created_at: string;
  expires_at: string;              // 48 hours from creation
  claimed: boolean;
  claimed_by?: string;             // User ID who claimed
  claimed_at?: string;
}

export interface MCPAppManifest {
  module: string;                  // 'latex-mcp', 'blog-mcp', etc.
  app_name: string;                // 'PDF Viewer', 'HTML Preview'
  resource_uri: string;            // 'ui://latex-mcp/preview.html'
  description?: string;
  supported_formats: string[];     // ['pdf'], ['html'], etc.
  dependencies?: string[];         // External CDNs, if any
}
```

**Alignment with AgenticGateway Memory:**
- **Groups** stored in Gateway's SQLite memory (garden metaphor). Portal queries via memory API endpoints (future: `/v1/gateway/memory/get` pattern).
- **Projects & Documents** stored as sub-documents in group memory (hierarchical structure).
- **ActivityEvents** stored in separate memory table (retention policy: 90 days, then archive).
- **InviteTokens** stored in a dedicated table with TTL index (auto-delete after 48 hours).

For v0.1, the Portal assumes Gateway memory endpoints exist. If not, portal uses basic file-based storage (JSON files in Gateway's data directory) as fallback.

### 3.4 AgenticGateway Integration

**Which Gateway APIs zen-sci-web calls:**

| Endpoint | Method | Purpose | Frequency |
|----------|--------|---------|-----------|
| `/health` | GET | Check Gateway online | Page load |
| `/v1/chat/completions` | POST | Auth intent (login) | On user login |
| `/v1/gateway/memory/query` | GET | Fetch groups, projects, activity | On group/project view |
| `/v1/gateway/memory/upsert` | POST | Create/update group, project | On form submit |
| `/admin/mcp/status` | GET | Health of 6 ZenSci servers | Page load + periodic |

**JWT Auth Strategy:**
```
1. User submits email/password via POST /api/auth/login
2. SvelteKit server calls Gateway's chat API with auth intent
3. Gateway returns JWT (with claims: user_id, email, roles, exp: 24h)
4. SvelteKit stores JWT in:
   - Secure, HttpOnly cookie (httpOnly: true, secure: true, sameSite: 'strict')
   - localStorage (for XHR, with token refresh logic)
5. All subsequent requests include: Authorization: Bearer {JWT}
6. SvelteKit server validates JWT before serving authenticated routes
```

**Fallback & Graceful Degradation:**
- If Gateway is offline: Show "Maintenance mode" banner. Core UI loads (no data fetch). Redirect to login if accessing protected route.
- If individual MCP server is offline: Show "Module unavailable" in document viewer. Allow viewing other modules.
- If memory API not available: Fall back to basic in-memory store (requires portal restart to persist).

### 3.5 MCP App Browser Rendering

**Phase 4 Integration:** Each ZenSci module ships a companion app (PDF viewer, HTML preview, compliance dashboard, etc.). Portal loads these apps in browser iframes.

**Resource Loading Model:**
```typescript
// MCP App bundles are single-file HTML (generated by Vite singlefile plugin)
// They are served by SvelteKit at static paths or from MCP server's ui:// URIs

// Portal flow:
1. Document viewer route: GET /groups/[id]/projects/[project_id]/documents/[doc_id]
2. Portal calls GET /api/mcp-apps/[module]/manifest
3. Server returns: { module: 'latex-mcp', resource_uri: 'ui://latex-mcp/preview.html', ... }
4. Portal renders:
   <DocumentViewer
     module="latex-mcp"
     resourceUri="ui://latex-mcp/preview.html"
     documentId={doc_id}
     documentData={document}
   />
5. DocumentViewer component:
   - Fetches app HTML from resourceUri (via /api/mcp-apps/[module]/fetch endpoint if needed)
   - Renders: <iframe srcDoc={appHtml} title="PDF Viewer" sandbox="allow-scripts allow-same-origin" />
   - Establishes postMessage bridge for app ↔ portal communication
```

**CSP (Content Security Policy) for Iframes:**
- Browser CSP: `default-src 'self'; script-src 'self' 'wasm-unsafe-eval'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self' data:`
- Iframe sandbox: `allow-scripts allow-same-origin allow-popups allow-forms`
- Prevents XSS while allowing app functionality

**postMessage Bridge:**
```typescript
// In document viewer component:
const iframeRef = ref<HTMLIFrameElement>();

onMount(() => {
  window.addEventListener('message', (e) => {
    if (e.origin !== window.location.origin) return; // Security: verify origin
    const { type, payload } = e.data;

    switch (type) {
      case 'app:ready':
        // App loaded, send document data
        iframeRef.current?.contentWindow?.postMessage({
          type: 'portal:document',
          payload: { document, editable: false }
        }, '*');
        break;
      case 'app:share':
        // User clicked "share" in app
        showShareDialog(payload);
        break;
      case 'app:download':
        // User clicked "download" in app
        triggerDownload(payload);
        break;
    }
  });
});
```

### 3.6 Group Activity Feed

**Activity Event Schema:**
```typescript
// lib/types/portal.ts
export type ActivityEventType =
  | 'user_joined'
  | 'user_left'
  | 'project_created'
  | 'document_converted'
  | 'document_shared'
  | 'member_added'
  | 'member_removed'
  | 'member_role_changed'
  | 'settings_updated'
  | 'group_created';

export interface ActivityEvent {
  id: string;
  group_id: string;
  type: ActivityEventType;
  actor_id: string;
  actor_name: string;
  timestamp: string;
  description: string;  // Human-readable: "Alice converted photosynthesis.md to PDF"
  data: {
    project_id?: string;
    project_name?: string;
    document_id?: string;
    document_title?: string;
    module?: string;  // 'latex-mcp', etc.
    status?: 'success' | 'error';
    error?: string;
    [key: string]: unknown;
  };
}
```

**Feed Polling (v0.1) / SSE (v1.1):**
- **v0.1 (polling):** Portal fetches `/api/groups/[id]/activity?limit=50&offset=0` every 30 seconds. UX: slight delay (30s), acceptable for async.
- **v1.1 (SSE):** Server pushes events via `EventSource`. Real-time updates, no polling. Requires Gateway support.

**Feed Components:**
```svelte
<!-- ActivityFeed.svelte -->
<script lang="ts">
  export let groupId: string;
  export let events: ActivityEvent[];

  let page = 0;
  const pageSize = 50;

  onMount(() => {
    // Polling interval
    const interval = setInterval(() => {
      fetchActivity();
    }, 30000);
    return () => clearInterval(interval);
  });

  async function fetchActivity() {
    const res = await fetch(`/api/groups/${groupId}/activity?limit=${pageSize}&offset=${page * pageSize}`);
    events = await res.json();
  }
</script>

<div class="activity-feed">
  {#each events as event (event.id)}
    <ActivityItem {event} />
  {/each}
  <Pagination bind:page on:change={fetchActivity} />
</div>
```

### 3.7 Onboarding & Invitation Flow

**Step-by-step flow for new user:**

```
1. Existing user (Alice in biology group) generates invite link
   - Portal: POST /api/groups/[group_id]/invites
   - Returns: { token: "abc123def456...", expires_at: "2026-02-20T10:00:00Z" }
   - Copy link: https://zen-sci-web.com/invite/abc123def456

2. Invitation email sent to new user (Bob)
   Subject: "Alice invited you to Biology 101 on ZenSci"
   Body: "Click here to join: [link]"

3. Bob clicks link → https://zen-sci-web.com/invite/abc123def456
   - Portal route: /invite/[token]
   - No auth required; show public preview of group
   - Page displays: Group name, description, member count, sample projects
   - CTA: "Sign up to join"

4. Bob signs up (email, password)
   - POST /api/auth/register → Gateway creates user
   - Portal redirects to /invite/[token] with session

5. Bob claims invite
   - POST /api/invites/[token]/claim
   - Portal adds Bob to group (role: 'member' by default)
   - Redirects to /groups/[group_id] (now Bob sees full group)

6. Onboarding nudge: Download desktop app
   - Banner: "Install ZenSci desktop to create projects locally"
   - Link: https://zen-sci-desktop.com/download?group=[group_id]
   - Desktop app deep link: Opens with group pre-loaded

7. Bob installs desktop, launches with group context
   - Desktop detects group ID from URL → pre-fetches group metadata
   - Seamless transition: Bob can now edit locally + sync to web portal
```

**Invitation Validation:**
```typescript
// /api/invites/[token]/+server.ts
export async function GET({ params, locals }) {
  const { token } = params;

  // Query Gateway memory for invite
  const invite = await gateway.getInvite(token);

  if (!invite) {
    return new Response(JSON.stringify({ error: 'Invalid or expired token' }), { status: 404 });
  }

  if (new Date(invite.expires_at) < new Date()) {
    return new Response(JSON.stringify({ error: 'Token expired' }), { status: 410 });
  }

  // Return public preview (no auth required)
  const group = await gateway.getGroup(invite.group_id, { publicOnly: true });
  return json({ group, token: invite.token });
}

export async function POST({ params, locals, request }) {
  const { token } = params;
  const user = locals.user; // Validated from JWT

  if (!user) {
    return new Response(JSON.stringify({ error: 'Not authenticated' }), { status: 401 });
  }

  const invite = await gateway.getInvite(token);
  if (!invite || new Date(invite.expires_at) < new Date()) {
    return new Response(JSON.stringify({ error: 'Invalid or expired token' }), { status: 410 });
  }

  // Add user to group
  await gateway.addMemberToGroup(invite.group_id, user.id, 'member');

  // Mark invite as claimed
  await gateway.markInviteClaimed(token, user.id);

  return json({ success: true, groupId: invite.group_id });
}
```

### 3.8 Key Components

| Component | Route(s) | Purpose | Key Props/State |
|-----------|----------|---------|-----------------|
| **Nav** | All | Top navigation bar (logo, search, user menu, notifications) | `currentUser`, `unreadNotifications` |
| **LoginForm** | `/login` | Email/password login | `onLogin` callback, error state |
| **GroupCard** | `/groups`, `/` | Compact group summary (name, members, projects) | `group: Group`, `onClick` handler |
| **GroupList** | `/groups` | Paginated list of user's groups | `groups: Group[]`, filters (type, role) |
| **GroupSettings** | `/groups/[id]/settings` | Admin panel (name, desc, member roles, delete) | `group: Group`, `onSave` callback, owner-only |
| **ProjectCard** | Project list views | Summary card (name, doc count, status) | `project: Project`, `module?: string` (filter) |
| **ProjectList** | `/groups/[id]/projects` | Filterable list (module type, date range) | `projects: Project[]`, `filters`, sort options |
| **DocumentViewer** | `/groups/[id]/projects/[pid]/documents/[did]` | MCP App iframe + metadata sidebar | `module: string`, `document: Document`, `readonly: true` |
| **ActivityFeed** | `/groups/[id]` (home tab) | Paginated event stream | `groupId: string`, polling state |
| **ActivityItem** | Inside ActivityFeed | Single event row | `event: ActivityEvent` |
| **UserCard** | Profile pages, member lists | Avatar, name, role badge | `user: User`, `role?: GroupRole` |
| **UserProfile** | `/profile/[user_id]` | Full profile (history, memberships, settings) | `userId: string`, fetch state |
| **ServerStatus** | Dashboard, Admin panel | Grid of 6 ZenSci servers + Gateway | `servers: ServerHealth[]`, refresh interval |
| **Modal** | Throughout (forms, confirm dialogs) | Reusable modal wrapper | `title`, `open: boolean`, `onClose` |
| **Toast** | Throughout | Toast notifications (success, error, info) | Auto-dismiss after 5s, dismiss button |
| **Pagination** | Activity feed, project lists, member lists | Page controls | `page: number`, `pageSize: number`, `total: number` |

---

## 4. Implementation Plan

**Timeline: 8–10 weeks (with parallel work on Rust backend if doing full portal v1.0)**

### Phase 1 (Week 1–2): Scaffold + Auth
**Deliverable:** SvelteKit app boots, login/register flows work, JWT tokens persist.

- Initialize SvelteKit project with TailwindCSS (or vanilla CSS)
- Implement auth routes (`/login`, `/register`, `/api/auth/*`)
- Build LoginForm, RegisterForm components
- Gateway JWT integration: POST `/v1/chat/completions` with auth intent
- Session management: JWT in cookie + localStorage
- Auth guard on `(auth)` route group
- Basic layout + navigation bar

**Acceptance:**
- `POST /api/auth/login` with email/password succeeds, returns JWT
- `GET /groups` without JWT redirects to `/login`
- Session persists on refresh

---

### Phase 2 (Week 3–4): Groups CRUD + Data Model
**Deliverable:** Users can create groups, view group list, see members, invite new members.

- Implement `/api/groups` endpoints (GET list, POST create, PATCH update)
- Build Group, GroupList, GroupCard, GroupSettings components
- Implement invite token generation: `POST /api/groups/[id]/invites`
- Build InviteForm component
- Display members list with roles (owner, member, viewer)
- Role-based UI (owner sees settings, delete; member sees read-only view)

**Data model setup:**
- Define TypeScript interfaces (User, Group, GroupRole, Project, Document, etc.)
- Map to Gateway memory structure
- Implement Gateway API client (`lib/server/gateway.ts`)

**Acceptance:**
- User creates group, sees it in `/groups` list
- User invites another user via email
- Invite token can be claimed via `/invite/[token]`
- Member joins group

---

### Phase 3 (Week 5–6): Projects & Documents
**Deliverable:** Projects display documents, documents show metadata, basic filtering by module.

- Implement `/groups/[id]/projects` route
- Build ProjectCard, ProjectList components
- Implement `/groups/[id]/projects/[project_id]` detail view
- Display document list (title, module, status, created date)
- Filter by module type (LaTeX, blog, grant, etc.)
- Sort by date, name, status

**Activity Feed (basic):**
- Implement `/api/groups/[id]/activity` endpoint
- Build ActivityFeed, ActivityItem components
- Polling interval: 30s fetch from `/api/groups/[id]/activity`
- Display human-readable event descriptions

**Acceptance:**
- User clicks project → sees list of documents
- User filters by module → sees only PDFs, or only blog posts, etc.
- Activity feed shows recent events (user created project, etc.)

---

### Phase 4 (Week 7): Document Viewer + MCP App Rendering
**Deliverable:** Clicking a document shows MCP App in iframe. Metadata sidebar shows title, author, status.

- Implement `/groups/[id]/projects/[project_id]/documents/[doc_id]` route
- Build DocumentViewer component (iframe wrapper)
- Implement `/api/mcp-apps/[module]/manifest` endpoint
- Fetch Phase 4 app HTML (from MCP server or static bundles)
- Render iframe with sandbox restrictions
- Build DocumentMetadata component (sidebar)
- postMessage bridge (basic: app can signal "ready", portal sends document data)

**Gateway Status Dashboard:**
- Implement `/api/mcp-apps/status` (poll `/admin/mcp/status`)
- Build ServerStatus component (6 ZenSci servers + Gateway health)
- Green/yellow/red indicators
- Show error message if conversion failed

**Acceptance:**
- Click document → iframe loads and displays app
- Sidebar shows title, author, module, status
- If module is offline, show "Unavailable" message

---

### Phase 5 (Week 8): Onboarding + Invitation Landing Page
**Deliverable:** Public `/invite/[token]` page shows group preview, signup flow, claim invite.

- Implement public invite preview route (`/invite/[token]`)
- Build group preview card (name, description, member count, sample projects)
- Redirect unsigned users to signup form
- Implement invite claim flow: `POST /api/invites/[token]/claim`
- Add "Download desktop app" nudge (with deep link)
- Landing page (`/`) with featured groups, call-to-action

**Acceptance:**
- Unauthenticated user receives invite link
- Opens link → sees group preview
- Signs up → joins group
- Sees "Download desktop app" prompt with deep link

---

### Phase 6 (Week 9): Profile Pages + Account Settings
**Deliverable:** Users can view/edit profiles, see document history, manage account.

- Implement `/profile/[user_id]` route
- Build UserProfile, UserCard, DocumentHistory components
- Implement account settings (`/(auth)/settings`)
- Password change, avatar upload, email update
- Document history: list of all documents user created (across groups)

**Acceptance:**
- User views own profile
- User sees list of documents they created
- User can change password

---

### Phase 7 (Week 10): Polish + Testing + Deployment
**Deliverable:** Fast page loads, responsive design, comprehensive tests, deployed to Node.js server.

- **Performance:**
  - SSR landing pages (/, /groups, /groups/[id]) for SEO + speed
  - Lazy-load MCP App iframes
  - Optimize images, defer non-critical CSS
  - Target: LCP < 2.5s, FID < 100ms, CLS < 0.1

- **Responsive design:**
  - Test on 320px (mobile), 768px (tablet), 1024px+ (desktop)
  - Touch-friendly buttons, readable text
  - Collapse navigation on mobile

- **Accessibility:**
  - Semantic HTML, ARIA labels
  - Color contrast (4.5:1 for text)
  - Keyboard navigation (Tab, Enter, Escape)

- **Testing:**
  - Unit tests: components (Vitest + Svelte Testing Library)
  - Integration tests: auth flows, group CRUD (Playwright)
  - E2E tests: critical paths (invite → join → view document)

- **Deployment:**
  - Build: `npm run build`
  - Deploy to Node.js server (pm2, systemd, or Docker)
  - Configure env vars (GATEWAY_URL, JWT_SECRET, SESSION_SECRET)
  - SSL/TLS (Let's Encrypt)
  - DNS, subdomain (zen-sci-web.com or portal.zenithscience.org)

---

## 5. Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| **Gateway API changes** | Medium | High | Lock Gateway to v1.0.0 during portal dev. Communicate breaking changes via ADR. Test against mock Gateway API. |
| **Phase 4 MCP Apps not ready on time** | Medium | Medium | Implement static document preview (image + metadata) as fallback. Defer fancy iframe rendering to v1.1. |
| **JWT token expiration logic breaks** | Low | High | Implement refresh token flow early (token rotation every 1h). Handle 401 responses with auto-logout. Test extensively. |
| **MCP Server goes offline during demo** | Medium | Low | Implement graceful degradation (show "Server offline" message). Cache last-known server health. No hard failure. |
| **Invite token collision (hash not unique)** | Very Low | High | Use cryptographically secure random token (crypto.randomBytes(32).toString('hex')). Verify uniqueness in DB. Set TTL index on tokens table. |
| **Tauri deep link to portal conflict** | Low | Medium | Coordinate protocol schemes with Rust backend. Use different domains: `zen-sci-web.com` (web) vs. `zen-sci://` (desktop). |
| **Browser CSP blocks MCP App iframe** | Low | High | Test CSP extensively. Warn if app requires `unsafe-eval`. Provide CSP override docs. |
| **Real-time activity feed too slow with polling** | Medium | Medium | Start with 30s polling. Move to SSE/WebSocket in v1.1. Optimize query (index on group_id + timestamp). |
| **Group memory queries are slow** | Medium | High | Denormalize project/document counts in group record. Add DB index on (group_id, created_at). Profile with Gateway team. |
| **Invite email delivery fails** | Low | Low | Use email service (SendGrid, Postmark). Test email templates. Retry with exponential backoff. |
| **Mobile viewport too small for MCP App iframe** | Low | Medium | Detect mobile, show "Open on desktop for best experience" fallback. Implement responsive app wrapper. |

---

## 6. Rollback & Contingency

**Deployment strategy:**
1. **Staging:** Test all changes on staging environment (same schema, mock data).
2. **Blue-green:** Deploy v0.1 to new Node.js instance. Switch traffic via reverse proxy.
3. **Rollback:** Keep previous version running. If issues, revert traffic in < 5 minutes.

**Data persistence:**
- All user data (groups, projects, documents) stored in **Gateway's SQLite database** (portal is stateless).
- If portal crashes, restart and data is intact.
- If portal database (session store) is lost, users re-login (no data loss).

**Graceful degradation:**
- If Gateway is unavailable: show "Maintenance mode" banner. Core UI still renders.
- If MCP server is offline: show "Module unavailable" per document. Other modules work.
- If email service fails: queue invite emails, retry async. Don't block user invite creation.

**Monitoring:**
- Gateway health endpoint: `/health` (checked every 10s)
- MCP server health: `/admin/mcp/status` (checked every 30s)
- Error tracking: Sentry (all uncaught exceptions logged)
- Database backups: Gateway-managed (not portal's concern)

---

## 7. Documentation & Communication

**User-facing docs (post-launch):**
- **Getting Started Guide:** Create account → join group → explore projects → download desktop app
- **Invitation Guide:** How to generate & share invite links
- **FAQ:** "What's the difference between web and desktop?" "Can I edit on web?" "How do I sync changes?"

**Developer docs:**
- **Architecture diagram:** Browser ↔ SvelteKit ↔ Gateway ↔ ZenSci
- **API reference:** All `/api/*` endpoints, request/response schemas
- **Deployment guide:** Env vars, Docker Compose, SSL setup
- **Contributing:** Code style (Prettier, ESLint), testing standards, PR process

**Internal communication:**
- **Status updates:** Weekly check-in (progress, blockers, decisions)
- **Architecture decisions:** Document in ADR format (if diverging from spec)
- **Gateway API contract:** Keep in sync with Gateway STATUS.md
- **Phase 4 coordination:** Daily standup during MCP App integration

---

## 8. Appendices

### 8.1 Route Table (All Routes with Auth Requirements)

| Route | Method | Auth | Purpose | SSR |
|-------|--------|------|---------|-----|
| `/` | GET | None | Landing page, featured groups | ✅ Yes |
| `/login` | GET/POST | None | Login form | ✅ Yes |
| `/register` | GET/POST | None | Signup form | ✅ Yes |
| `/invite/[token]` | GET | None | Invite preview, public group details | ✅ Yes |
| `/groups` | GET | JWT | User's groups list | ❌ SPA |
| `/groups/new` | GET | JWT | Create group form | ❌ SPA |
| `/groups/[id]` | GET | JWT | Group workspace (activity feed, home) | ❌ SPA |
| `/groups/[id]/projects` | GET | JWT | Project list (filterable) | ❌ SPA |
| `/groups/[id]/projects/new` | GET | JWT | Create project form | ❌ SPA |
| `/groups/[id]/projects/[pid]` | GET | JWT | Project detail | ❌ SPA |
| `/groups/[id]/projects/[pid]/documents/[did]` | GET | JWT | Document viewer (MCP App iframe) | ❌ SPA |
| `/groups/[id]/members` | GET | JWT | Members list, invite form | ❌ SPA |
| `/groups/[id]/settings` | GET | JWT | Group settings (owner only) | ❌ SPA |
| `/profile/[user_id]` | GET | JWT | User profile, document history | ❌ SPA |
| `/settings` | GET | JWT | Account settings, logout | ❌ SPA |
| `/api/auth/login` | POST | None | Authenticate user, return JWT | — |
| `/api/auth/logout` | POST | JWT | Clear session | — |
| `/api/auth/register` | POST | None | Create user account | — |
| `/api/auth/me` | GET | JWT | Get current user | — |
| `/api/groups` | GET/POST | JWT | List/create groups | — |
| `/api/groups/[id]` | GET/PATCH/DELETE | JWT | Read/update/delete group | — |
| `/api/groups/[id]/projects` | GET/POST | JWT | List/create projects | — |
| `/api/groups/[id]/projects/[pid]` | GET/PATCH/DELETE | JWT | Read/update/delete project | — |
| `/api/groups/[id]/projects/[pid]/documents` | GET | JWT | List documents in project | — |
| `/api/groups/[id]/members` | GET/POST/DELETE | JWT | List/add/remove members | — |
| `/api/groups/[id]/activity` | GET | JWT | Activity feed (paginated) | — |
| `/api/groups/[id]/invites` | POST | JWT | Generate invite token | — |
| `/api/invites/[token]` | GET | None | Get invite preview | — |
| `/api/invites/[token]/claim` | POST | JWT | Claim invite, join group | — |
| `/api/documents/[id]` | GET | JWT | Get document metadata | — |
| `/api/mcp-apps/[module]/manifest` | GET | JWT | Get MCP App manifest (resourceUri) | — |
| `/api/mcp-apps/[module]/health` | GET | JWT | Check if MCP server online | — |
| `/api/mcp-apps/status` | GET | JWT | Health of all 6 ZenSci servers | — |
| `/api/gateway/health` | GET | None | Check Gateway online | — |

---

### 8.2 Gateway API Endpoints Used (Detailed Contracts)

**Note:** These endpoints are called by SvelteKit server-side code, not directly from browser.

#### Chat API (for auth)
```
POST /v1/chat/completions
Authorization: Bearer {optional-api-key}

Request:
{
  "model": "claude-3-5-sonnet-20241022",
  "messages": [
    {
      "role": "user",
      "content": "User login: email=alice@example.com, password=..."
    }
  ]
}

Response:
{
  "id": "chatcmpl-...",
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "User authenticated. JWT: eyJhbGc..."
      }
    }
  ]
}
```

**Note:** For v0.1, auth is simplified. Gateway may provide a dedicated `/v1/auth` endpoint (if implemented). Portal should check Gateway STATUS.md for latest auth contract.

#### Memory API (future endpoints, if available in v1.1+)
```
GET /v1/gateway/memory/query?type=group&filter=owner_id:{user_id}
Authorization: Bearer {jwt}

Response:
{
  "records": [
    {
      "id": "group-uuid",
      "type": "group",
      "content": {
        "name": "Biology 101",
        "description": "...",
        "members": [...],
        ...
      }
    }
  ]
}

POST /v1/gateway/memory/upsert
Authorization: Bearer {jwt}
Content-Type: application/json

Request:
{
  "type": "group",
  "content": {
    "id": "group-uuid",
    "name": "Biology 101",
    ...
  }
}

Response:
{
  "id": "group-uuid",
  "created": false,
  "updated": true
}
```

#### MCP Status
```
GET /admin/mcp/status
Authorization: Bearer {admin-jwt}

Response:
{
  "servers": {
    "mcp_by_dojo": {
      "server_id": "mcp_by_dojo",
      "display_name": "MCP By Dojo Genesis",
      "state": "connected",
      "tool_count": 14,
      "last_health_check": "2026-02-18T12:00:00Z",
      "last_error": ""
    }
  },
  "total_servers": 1,
  "total_tools": 14,
  "healthy": true
}
```

---

### 8.3 Desktop App Deep Link Spec

**Protocol:** `zen-sci://` (Tauri custom scheme)

**Format:**
```
zen-sci://group?id={group_id}&user={email}&token={invite_token?}

Examples:
zen-sci://group?id=550e8400-e29b-41d4-a716-446655440000&user=alice@example.com
zen-sci://group?id=550e8400-e29b-41d4-a716-446655440000&user=alice@example.com&token=invite-token-from-web
```

**Desktop App Handling:**
1. Tauri app registers `zen-sci://` protocol handler (OS-specific)
2. On launch with deep link:
   - Parse `group_id`, `user`, `token` from URL
   - Pre-fetch group metadata from Gateway
   - If `token` provided, auto-join group (if not already member)
   - Pre-populate `user` field (or auto-login if token valid)
   - Load group workspace directly (skip home screen)

**Portal Flow:**
- After user joins group on web, show: "Install desktop app: [download-link?group={group_id}]"
- Download link redirects to installer, appends `group_id` to installer config
- On first run, desktop app launches with group pre-loaded

---

### 8.4 Future Features (v1.1+): Real-Time & More

**Deferred to v1.1 (not blocking v0.1):**

#### Real-Time Collaboration (v1.1)
- Rust backend (Tokio WebSocket + Automerge CRDT)
- Live presence (who's editing which document)
- Collaborative editing with conflict resolution
- Estimated effort: 12–16 weeks (Rust backend) + 4–6 weeks (portal integration)

#### Advanced Activity Feed (v1.1)
- SSE event streaming (real-time updates)
- Personalized digest emails
- Notification preferences (per-group, per-event-type)
- Estimated effort: 3–4 weeks

#### Local Search Index (v1.0)
- Rust backend (Tantivy full-text search)
- Portal integrates Rust search backend (if co-deployed)
- Estimated effort: 6–8 weeks (Rust) + 2–3 weeks (portal)

#### Mobile Apps (v2)
- React Native or Flutter wrapper around web portal
- Touch-optimized UI
- Offline sync (future)
- Estimated effort: 8–12 weeks

#### Payment & Team Licensing (v2)
- Stripe integration for premium tiers
- Team management (seats, billing)
- Admin dashboard
- Estimated effort: 6–8 weeks

---

### 8.5 Open Questions

**For Cruz / Product:**

1. **Invite expiry:** 48 hours from creation? Configurable per group? Never expires?

2. **Role model:** Is `viewer` (read-only) necessary for v0.1? Or just `member` + `owner`?

3. **Project ownership:** Can only project creator delete it, or any group `owner`?

4. **Activity retention:** How long to keep activity events in Gateway memory? (Recommendation: 90 days)

5. **Email integration:** Should web portal send invites, or just generate token + let Gateway/admin send?

6. **Desktop app URL scheme:** Is `zen-sci://` the right choice, or use different protocol (e.g., `zenithscience://`)?

7. **Analytics scope:** Should portal log page views / feature usage? (Recommendation: defer to v1.1)

8. **Multi-group view:** Should a project belong to multiple groups, or just one?

9. **Document versioning:** Should portal show version history (v1, v2, etc.), or just latest?

10. **Export/download:** Can users download converted documents (PDFs, HTML) from portal? Or view-only?

---

## Summary

**zen-sci-web v0.1** is a **hosted web portal** delivering the following value:

1. **New user onboarding** — Invite links → group discovery → desktop app download
2. **Async group collaboration** — Activity feeds, project browsing, member management
3. **MCP App rendering** — Phase 4 apps (PDF viewer, HTML preview, compliance dashboard) run in browser
4. **Gateway-backed** — Stateless SvelteKit server proxying to production AgenticGateway

**Deliverables:**
- SvelteKit 2.0+ application (SSR + SPA hybrid)
- 40+ TypeScript components (Nav, Auth, Groups, Projects, Documents, Activity, etc.)
- 45+ API endpoints (`/api/*` routes)
- JWT auth (secure cookies, token refresh)
- Responsive design (320px–1920px)
- Comprehensive tests (unit, integration, E2E)
- Deployed to Node.js server (pm2, Docker, or systemd)

**Timeline:** 8–10 weeks

**Quality bar:** Production-ready. Fast page loads (LCP < 2.5s). Accessible (WCAG 2.1 AA). Comprehensive tests.

---

**Document prepared by:** Architecture Planning (Scout & Design)
**Date:** 2026-02-18
**Status:** Ready for implementation
**Next step:** Assign engineering resources, lock decisions in Section 8.5, begin Phase 1 (Week 1 scaffold + auth)

---

## Revision R1: Architectural Corrections (2026-02-18)

This revision corrects the initial framing of the web portal from "display layer for desktop outputs" to "full peer workspace," and establishes the canonical architecture for collaboration, versioning, and publication.

### 1. Portal Architecture Reframe: Peer Workspace Model

#### From Display Layer to Workspace
The initial spec treated zen-sci-web as a "thin client" displaying pre-created desktop outputs. This is incorrect. The portal is a **full stateful workspace** where researchers work directly in the canonical document source.

**Corrected Mental Model:**
- **"Notion for academic work"** — researchers write, edit, and structure documents directly in the browser
- **"GitHub Pages for group outputs"** — groups publish polished output pages from their documents (one-click publish from any document)
- **Social layer** — groups, discovery, activity feeds, group pages
- **Portal and desktop are PEERS** that sync to the cloud catalog. Neither is primary.
  - Desktop = deep offline work with local Ollama and full MCP pipeline
  - Web portal = always-on cloud workspace, always working with canonical source

#### Document Ownership & Canonical Source
- All documents are stored in the portal's canonical database
- Desktop and web portal are **sync clients** to this source:
  - Desktop can pull documents to local cache, work offline, push changes back
  - Web portal always works live against the canonical database
  - Sync mechanism: Git-like version hashes + AI-assisted merge (see Section 5)
- Groups can collaborate in portal, desktop users see updated documents on next sync

#### Access Model
- Documents are private to their project by default
- Project members (via group membership) can edit/comment in real-time or async
- Published documents (see Section 6) have public read-only URLs

---

### 2. Portal Database Schema

The portal maintains its own PostgreSQL (or SQLite for self-hosted) database, separate from the Gateway's agent memory SQLite.

#### Schema Overview
```sql
-- Users (auth via Gateway JWT)
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT NOT NULL UNIQUE,
  username TEXT NOT NULL UNIQUE,
  avatar_url TEXT,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Research groups, labs, classrooms
CREATE TABLE groups (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  slug TEXT NOT NULL UNIQUE,
  owner_id UUID NOT NULL REFERENCES users(id),
  description TEXT,
  visibility TEXT NOT NULL DEFAULT 'private', -- 'private' | 'internal' | 'public'
  avatar_url TEXT,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Group membership
CREATE TABLE group_members (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  group_id UUID NOT NULL REFERENCES groups(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  role TEXT NOT NULL DEFAULT 'member', -- 'owner' | 'co-author' | 'viewer'
  joined_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(group_id, user_id)
);

-- Projects within groups
CREATE TABLE projects (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  group_id UUID NOT NULL REFERENCES groups(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  slug TEXT NOT NULL,
  description TEXT,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(group_id, slug)
);

-- Documents (canonical source of truth)
CREATE TABLE documents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  title TEXT NOT NULL,
  version_hash TEXT NOT NULL, -- git-like content hash (SHA-256)
  mode TEXT NOT NULL DEFAULT 'draft', -- 'draft' | 'structured'
  owner_id UUID NOT NULL REFERENCES users(id),
  published_at TIMESTAMP, -- NULL if not published
  publish_url TEXT, -- e.g., /groups/{slug}/outputs/{doc-slug}
  crdt_state BYTEA, -- NULL in v1, populated when WebSocket/CRDT enabled in v2
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Sections (section ownership model, used in structured mode)
CREATE TABLE sections (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  document_id UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
  title TEXT NOT NULL,
  content TEXT NOT NULL,
  owner_id UUID REFERENCES users(id), -- NULL = orphaned
  section_order INTEGER NOT NULL,
  locked BOOLEAN NOT NULL DEFAULT false,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Version history (git-like)
CREATE TABLE document_versions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  document_id UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
  version_hash TEXT NOT NULL UNIQUE,
  content_snapshot JSONB NOT NULL, -- full document state at this version
  author_id UUID NOT NULL REFERENCES users(id),
  commit_message TEXT,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Comments (inline collaboration)
CREATE TABLE comments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  section_id UUID NOT NULL REFERENCES sections(id) ON DELETE CASCADE,
  author_id UUID NOT NULL REFERENCES users(id),
  content TEXT NOT NULL,
  resolved BOOLEAN NOT NULL DEFAULT false,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Suggestions (proposed changes)
CREATE TABLE suggestions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  section_id UUID NOT NULL REFERENCES sections(id) ON DELETE CASCADE,
  author_id UUID NOT NULL REFERENCES users(id),
  proposed_content TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending', -- 'pending' | 'accepted' | 'rejected'
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Activity feed
CREATE TABLE activity_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  group_id UUID NOT NULL REFERENCES groups(id) ON DELETE CASCADE,
  actor_id UUID NOT NULL REFERENCES users(id),
  verb TEXT NOT NULL, -- 'created', 'edited', 'commented', 'published', etc.
  object_type TEXT NOT NULL, -- 'document', 'section', 'comment', 'suggestion'
  object_id UUID NOT NULL,
  metadata JSONB, -- extra context (e.g., old_value, new_value for diffs)
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Indexes for common queries
CREATE INDEX idx_documents_project_id ON documents(project_id);
CREATE INDEX idx_documents_owner_id ON documents(owner_id);
CREATE INDEX idx_documents_published_at ON documents(published_at);
CREATE INDEX idx_sections_document_id ON sections(document_id);
CREATE INDEX idx_sections_owner_id ON sections(owner_id);
CREATE INDEX idx_document_versions_document_id ON document_versions(document_id);
CREATE INDEX idx_document_versions_author_id ON document_versions(author_id);
CREATE INDEX idx_comments_section_id ON comments(section_id);
CREATE INDEX idx_suggestions_section_id ON suggestions(section_id);
CREATE INDEX idx_activity_events_group_id ON activity_events(group_id);
CREATE INDEX idx_activity_events_created_at ON activity_events(created_at DESC);
CREATE INDEX idx_group_members_user_id ON group_members(user_id);
```

#### Schema Notes
- **users** table is synced from Gateway JWT claims; portal stores minimal user profile
- **group_members.role** controls document editing permissions
- **documents.mode** controls whether sections have enforced ownership
- **sections.owner_id = NULL** indicates an orphaned section (see Section 4)
- **document_versions.content_snapshot** is a full JSONB snapshot of the document at that commit
- **documents.crdt_state** is reserved for WebSocket/CRDT in v2+; remains NULL in v1

---

### 3. Auth Integration: Gateway as Identity Provider

The Gateway already has JWT authentication (auth.go). The portal uses Gateway as its identity provider.

#### New Gateway Auth Endpoints (to be added to Gateway v1.1+)
```
POST /auth/register
  Body: { email: string, username: string, password: string }
  Response: { token: string, user_id: string, user: { id, email, username, avatar_url } }
  Status: 201 on success, 409 on email/username conflict

POST /auth/login
  Body: { email: string, password: string }
  Response: { token: string, user_id: string, user: { id, email, username, avatar_url } }
  Status: 200 on success, 401 on invalid credentials

POST /auth/refresh
  Body: { token: string }
  Response: { token: string }
  Status: 200 on success, 401 on invalid/expired token

GET /auth/me
  Headers: Authorization: Bearer {token}
  Response: { user: { id, email, username, avatar_url } }
  Status: 200 on success, 401 on missing/invalid token
```

#### Portal Client Authentication
**Web Client:**
- On successful login, Gateway returns JWT token
- Portal stores token in **httpOnly cookie** (secure, same-site=strict, not accessible to JavaScript)
- Fallback: localStorage for development (not recommended for production)
- Browser automatically sends cookie on all same-origin requests

**Desktop Client:**
- Stores JWT in secure storage (Tauri SecureStorage or OS keychain)
- Sends token via Bearer header on all API requests to portal

**Token Lifecycle:**
- Default expiry: 24 hours
- Refresh endpoint allows issuing new token without re-login
- Portal can request new token before expiry (e.g., 1 hour before expiry)

#### Portal API Authentication
All portal endpoints (except public published documents) require:
```
GET /api/documents/:id
  Headers: Authorization: Bearer {token}
  OR
  Cookie: session={jwt}
```

Portal middleware validates JWT signature using Gateway's public key (distributed at `/auth/public-key`).

---

### 4. Collaboration Data Model: Section Ownership & Modes

#### Draft Mode (Default)
When a document is created, it starts in `mode: draft`:
- **Free-form writing:** document owner and all co-authors can edit any part of the document
- **No section boundaries:** content is a continuous text blob
- **Collaborative editing:**
  - Multiple users can work simultaneously (async in v1; real-time WebSocket in v2)
  - Comments and suggestions work on text selections
  - All changes create a version (see Section 5)
- **Permissions:** group members with 'co-author' role can edit; 'viewer' role can only comment/suggest

#### Structured Mode
When the document owner clicks "Organize sections," the document transitions to `mode: structured`:
- **Section ownership:** document is divided into sections; each section has an assigned owner
- **Ownership assignment:**
  - Document owner sees a modal listing all sections (auto-extracted from headings or manual breaks)
  - Owner drags sections from "Unassigned" pool to co-author names
  - OR uses dropdown to select owner for each section
  - UI shows color-coded sections (each author gets a color)
- **Edit permissions in structured mode:**
  - **Section owner** can directly edit their own section
  - **Co-authors (non-owner)** cannot directly edit another's section
  - Co-authors see "Request edit access" button → opens comment form with context request
  - Document owner can force-edit any section (with notification to section owner)
- **Transition UI:** document header shows "Draft" badge in draft mode, "Structured (5 sections, 3 authors)" in structured mode
- **Reversibility:** document owner can revert to draft mode (all sections merge back to continuous content)

#### Orphaned Sections
When a co-author is removed from a group or project:
- Their sections become `owner_id = NULL` (orphaned)
- Portal displays a **warning banner** in document: "⚠️ 2 sections are unowned—click to reassign"
- Document owner can:
  - **Reassign section** to another active group member via `PATCH /api/sections/:id/owner { new_owner_id }`
  - **Claim section** (document owner takes ownership)
  - **Lock section** (readable, no editing; useful for archival)
- Orphaned sections cannot be directly edited by anyone (locked by system)
- Activity event logged: "Section 'Introduction' orphaned due to user removal from group"

#### API Endpoints for Section Ownership
```
PATCH /api/documents/:id/mode
  Body: { mode: 'draft' | 'structured' }
  Response: { document: { id, title, mode, ... } }
  Requires: bearer token, document owner

GET /api/documents/:id/sections
  Response: { sections: [ { id, title, owner_id, owner: { username, avatar_url }, locked, ... } ] }

PATCH /api/sections/:id/owner
  Body: { new_owner_id: UUID }
  Response: { section: { id, owner_id, ... } }
  Requires: bearer token, document owner

PATCH /api/sections/:id/lock
  Body: { locked: boolean }
  Response: { section: { id, locked } }
  Requires: bearer token, document owner
```

---

### 5. Document Version History: Git-like Versioning & AI-Assisted Merge

#### Versioning Model
Every "save" action by a user creates a new entry in `document_versions`:
- **Autosave** (background, no explicit user action): creates a draft version (no `commit_message`, not returned in history)
- **Manual save** (user clicks "Save"): creates a committed version with optional commit message
- **Version hash:** computed as SHA-256 of document content (deterministic, like git)

#### Version Structure
```json
{
  "id": "uuid",
  "document_id": "uuid",
  "version_hash": "a3f7d2e...",
  "author_id": "uuid",
  "commit_message": "Fixed typo in abstract",
  "content_snapshot": {
    "title": "Research Paper",
    "sections": [
      { "id": "sec1", "title": "Abstract", "content": "..." },
      { "id": "sec2", "title": "Methods", "content": "..." }
    ],
    "metadata": { "mode": "structured", ... }
  },
  "created_at": "2026-02-18T10:30:00Z"
}
```

#### Desktop ↔ Portal Sync
**Desktop pulls document:**
1. Desktop sends request: `GET /api/documents/:id/versions?since=<last-known-hash>`
2. Portal returns: `{ current_version_hash, versions: [ ... ], canonical_content }`
3. Desktop caches locally and works offline

**Desktop pushes changes:**
1. Desktop computes local version hash from edited content
2. Desktop sends: `POST /api/documents/:id/sync { local_version_hash, content, commit_message, parent_hash }`
3. Portal checks if `parent_hash == current_version_hash`:
   - **If match:** Accept push, create new version, update `documents.version_hash`
   - **If diverged:** Return 409 Conflict with both versions + common ancestor

**AI-Assisted Merge (on divergence):**
1. Portal returns: `{ conflict: true, portal_version, desktop_version, common_ancestor, merge_endpoint }`
2. Desktop triggers merge UI: "Remote changes detected. Review and merge?"
3. User selects "Suggest AI merge" → Desktop sends to Gateway DAG:
   ```
   POST /gateway/v1/chat/completions
     model: 'claude-opus'
     tools: ['merge_documents']
     prompt: "Merge these two versions with common ancestor..."
   ```
4. Gateway invokes `merge_documents` tool (returns merged content as JSON)
5. Desktop receives merged result, user reviews, clicks "Accept merge"
6. Desktop creates new version with `parent_hash: [portal_hash, desktop_hash]` (multi-parent, like git merge commits)
7. Desktop pushes merged version: `POST /api/documents/:id/sync { merged_content, commit_message: "Merge desktop and portal changes" }`
8. Portal stores new version, all clients see merged result

#### Version History UI
**Portal's Version History panel** (accessible from document settings):
- List of all committed versions (most recent first)
- Each shows: author avatar, commit message, timestamp, version hash
- Click version → view document at that snapshot
- Right-click two versions → "Compare" → diff view (visual or text)
- "Revert to this version" button → creates new version with cherry-picked snapshot as content

#### API Endpoints for Versioning
```
GET /api/documents/:id/versions
  Query: ?limit=50&offset=0
  Response: { versions: [ { id, author, commit_message, created_at, version_hash } ] }

GET /api/documents/:id/versions/:hash
  Response: { version: { ..., content_snapshot } }

POST /api/documents/:id/sync
  Body: {
    local_version_hash: string,
    content: object,
    commit_message?: string,
    parent_hash: string
  }
  Response (success): { version: { id, version_hash, ... } }
  Response (conflict): { conflict: true, portal_version, desktop_version, common_ancestor }

POST /api/documents/:id/versions/:hash/revert
  Response: { version: { id, version_hash, ... } }
  Effect: Creates new version with content from :hash as snapshot
```

---

### 6. Publish Model: One-Click Public Output Pages

#### Publishing Flow
**Document is private by default:**
- Only project members can view in portal
- Not indexed, not discoverable
- Can only be shared via explicit invite links

**Publishing (Explicit Action):**
1. Document owner clicks **"Publish"** button
2. Portal shows confirmation dialog:
   - Preview of public URL (e.g., `/groups/neuroscience-lab/outputs/neural-plasticity`)
   - Checkbox: "Allow comments on published page"
   - Confirm
3. Portal sets:
   - `documents.published_at = now()`
   - `documents.publish_url = '/groups/{group_slug}/outputs/{doc_slug}'`
   - Creates activity event: "Document published"
4. Portal generates static HTML view of document (styled as academic output)

**Published View (Public, Read-Only):**
- Accessible at: `portal.zensci.io/groups/{group_slug}/outputs/{doc_slug}`
- No authentication required
- Renders document in polished, print-friendly layout
- Author attribution and group metadata visible
- Publication timestamp visible
- If "Allow comments" enabled: read-only comment section (no login required to view)
- Meta tags for social sharing (Open Graph, Twitter Card)

**Unpublishing:**
- Document owner clicks "Unpublish" → confirmation → sets `published_at = NULL`, removes `publish_url`
- Published URL returns 404
- Activity event logged: "Document unpublished"

#### API Endpoints for Publishing
```
PATCH /api/documents/:id/publish
  Body: { published: true, allow_comments?: boolean }
  Response: { document: { id, published_at, publish_url, ... } }
  Requires: bearer token, document owner

PATCH /api/documents/:id/unpublish
  Response: { document: { id, published_at: null, publish_url: null, ... } }
  Requires: bearer token, document owner

GET /public/groups/:slug/outputs/:doc-slug
  Response: HTML page (no auth required)
  Status: 200 on success, 404 if not published
```

#### Published Document Storage & Caching
- Published HTML is generated on-demand (not cached in v1)
- In v2+: add CDN cache headers, generate static HTML file on publish
- URL slug is derived from document title (slugify), with uniqueness enforcement
- SEO: published pages include canonical URL meta tag pointing to publication URL

---

### 7. Forward-Looking CRDT Field for Real-Time Collaboration

#### Current State (v1): Async-Only
The portal operates in async mode in v1:
- Multiple users can edit, but changes are versioned sequentially
- No real-time cursor/awareness
- Comments and suggestions enable async feedback

#### Future State (v2+): WebSocket & CRDT
The `documents.crdt_state` column is introduced in v1 schema but remains NULL during v1 operation. This prepares for v2+ real-time collaboration without requiring table migration.

**When WebSocket/CRDT is added:**
1. Portal adds WebSocket endpoint: `WS /v1/docs/:id/sync`
2. CRDT library (Yjs recommended) manages `documents.crdt_state`:
   - `crdt_state` stores serialized Yjs document state (BYTEA)
   - On each client edit, Yjs emits update → sent over WebSocket to other clients
   - Portal receives update → applies to CRDT state → broadcasts to other clients
   - Periodically: save snapshot to `crdt_state` column (checkpoint)
3. Version history still works:
   - Periodic "commits" from CRDT snapshot create document_versions
   - Manual save: user clicks "Commit" → creates document_version from current CRDT state
4. Migration path from v1 to v2:
   - v2 migration script: `UPDATE documents SET crdt_state = <snapshot from latest document_version>`
   - Existing v1 documents instantly gain CRDT support

#### CRDT Benefits (Future)
- **Real-time co-editing:** multiple users see each other's edits live
- **Conflict-free:** CRDT automatically merges concurrent edits (no manual merge needed)
- **Offline support:** desktop can cache CRDT state, sync changes when reconnected
- **Rich presence:** see who else is editing, cursor positions (with additional extension)

#### Placeholder Implementation (v1)
In v1, the `crdt_state` column exists but is not used:
- Portal writes NULL on document creation
- Portal ignores the column on save
- Schema is forward-compatible; no migration on v2 release

---

### 8. Activity Feed & Notifications (Social Layer)

#### Activity Events
Portal logs all significant actions in `activity_events`:

```sql
INSERT INTO activity_events (
  group_id, actor_id, verb, object_type, object_id, metadata
) VALUES (
  'group-uuid', 'user-uuid', 'edited', 'document', 'doc-uuid',
  '{"section_id": "sec-uuid", "changed_fields": ["content"]}'
);
```

**Common events:**
- `verb: 'created'` + `object_type: 'document'` → "User created document"
- `verb: 'edited'` + `object_type: 'document'` → "User edited document" (payload includes which section)
- `verb: 'commented'` + `object_type: 'comment'` → "User commented on section X"
- `verb: 'suggested'` + `object_type: 'suggestion'` → "User suggested change to section X"
- `verb: 'published'` + `object_type: 'document'` → "User published document"
- `verb: 'joined'` + `object_type: 'group'` → "User joined group"

#### Group Activity Feed
**GET /api/groups/:id/activity**
```json
{
  "events": [
    {
      "id": "event-uuid",
      "verb": "edited",
      "object_type": "document",
      "actor": { "id", "username", "avatar_url" },
      "target": { "id", "title" },
      "created_at": "2026-02-18T10:30:00Z",
      "metadata": { "changed_fields": [...] }
    }
  ]
}
```

#### Portal Home Feed
- Shows activity from all groups user is a member of
- Sorted by recency
- Grouped by date or document
- Actionable: "View document", "View comment thread", etc.

---

### 9. Implementation Checklist for v1

**Portal Backend (Go, new service):**
- [ ] Set up PostgreSQL schema (run migrations)
- [ ] Implement auth middleware (validate Gateway JWT)
- [ ] Implement CRUD endpoints for documents, sections, groups
- [ ] Implement versioning & sync endpoints
- [ ] Implement publish/unpublish
- [ ] Implement comment & suggestion endpoints
- [ ] Implement activity logging
- [ ] Add integration tests against Gateway auth

**Portal Frontend (React/TypeScript):**
- [ ] User registration & login (call Gateway auth endpoints)
- [ ] Document editor (simple markdown or rich text)
- [ ] Section ownership UI (drag-and-drop assignment)
- [ ] Version history panel (list, diff, revert)
- [ ] Publish dialog & published view
- [ ] Comments & suggestions UI
- [ ] Activity feed
- [ ] Group management pages
- [ ] Public profile pages

**Gateway (Go, update existing):**
- [ ] Add `POST /auth/register` endpoint
- [ ] Add `POST /auth/login` endpoint
- [ ] Add `POST /auth/refresh` endpoint
- [ ] Add `GET /auth/me` endpoint
- [ ] Add `GET /auth/public-key` endpoint (for portal JWT verification)
- [ ] (Optional) Add `merge_documents` tool for AI-assisted merge

**Desktop Integration:**
- [ ] Update Tauri frontend to support sync with portal
- [ ] Implement `sync` flow (pull from portal, push changes, handle conflicts)
- [ ] Display "synced ↔ portal" indicator in document header

---

### 10. Glossary of Key Terms

| Term | Definition |
|------|-----------|
| **Peer workspace** | Portal and desktop are co-equal sync clients; neither is primary. Both read/write to canonical cloud source. |
| **Canonical source** | Portal's PostgreSQL database (documents, versions, sections). Desktop has local cache. |
| **Draft mode** | Document state where all co-authors can edit any part freely; no section ownership. |
| **Structured mode** | Document state where sections have assigned owners; non-owner co-authors must request edit access. |
| **Orphaned section** | Section whose owner has been removed from the group (owner_id = NULL); locked from editing. |
| **Version hash** | SHA-256 hash of document content (like git commit hash); uniquely identifies a snapshot. |
| **CRDT state** | Conflict-free replicated data type state (Yjs format); stored in `documents.crdt_state` column; NULL in v1, used in v2+ for real-time sync. |
| **Publish** | Make document publicly readable at a generated URL; document owner action. |
| **Activity event** | Log entry of a user action (create, edit, comment, publish); feeds the group activity feed. |

---

### 11. Backwards Compatibility & Gotchas

#### Desktop App (existing Tauri app) + Portal (new)
- **Initial state:** Desktop app has no knowledge of portal; users create documents locally
- **On-boarding:** when user logs into portal, portal provides UI to "Import from desktop" or "Start fresh in cloud"
- **Sync is optional:** users can use desktop-only, portal-only, or both (with sync)
- **No forced migration:** existing desktop documents remain local until explicitly synced

#### Public URLs are immutable (v1)
- Once a document is published, the public URL should not change
- If document title changes after publication, slug remains same (requires manual update if desired)
- In v2+, add "slug" field to documents for explicit control

#### Group visibility & published documents
- A document can be published (public) even if its group is private
- Published document contains group name (for attribution) but does not require group membership to view
- Published page includes "Back to group" link (but link returns 404 if viewer is not group member)

---

End of Revision R1.

## Revision R2: Locked Decisions (2026-02-18)

### Portal Hosting — Final Decision: Self-Hosted First (Docker Compose)

**Decision:** v1 ships as a Docker Compose stack. Managed cloud is a v1.1 addition.

**Rationale:** Academic data sovereignty. Self-hosted Gateway means Ollama is available to web users (co-located, not just desktop). Near-zero hosting cost during validation. Academic IT departments can handle Docker.

**Docker Compose stack (`docker-compose.yml` schema):**
```yaml
version: '3.9'
services:
  gateway:
    image: trespies/agentic-gateway:latest
    environment:
      - JWT_SECRET=${JWT_SECRET}
      - ENVIRONMENT=production
      - DATABASE_URL=sqlite:///data/gateway.db
    volumes:
      - gateway-data:/data
      - ./gateway-config.yaml:/app/gateway-config.yaml
    ports:
      - "8080:8080"

  portal:
    image: trespies/zen-sci-portal:latest
    environment:
      - DATABASE_URL=postgresql://postgres:${POSTGRES_PASSWORD}@db:5432/zensci
      - GATEWAY_URL=http://gateway:8080
      - JWT_SECRET=${JWT_SECRET}
      - PUBLIC_BASE_URL=${PUBLIC_BASE_URL}
    depends_on:
      - db
      - gateway
    ports:
      - "3000:3000"

  db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_DB=zensci
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data

  ollama:
    image: ollama/ollama:latest
    volumes:
      - ollama-data:/root/.ollama
    profiles: ["ollama"]  # opt-in: docker compose --profile ollama up

volumes:
  gateway-data:
  postgres-data:
  ollama-data:
```

**Key deployment notes:**
- `JWT_SECRET` is shared between gateway and portal — single source of truth for token validation
- Ollama is opt-in via Docker Compose profile (not required for institutions using cloud LLMs only)
- `PUBLIC_BASE_URL` sets the canonical URL for published document links
- SQLite for self-hosted is acceptable for small groups (<20 users); PostgreSQL is the production default and is always included in the stack
- v1.1 managed cloud path: same images, deployed to Fly.io/Railway with managed PostgreSQL

### Section Boundaries in Draft Mode — Final Decision

Same as desktop spec R2: any group member creates sections in draft mode; owner approves structure at draft → structured transition.

**Portal API endpoints:**
```
POST /api/documents/:id/sections
  Body: { title, position? }
  Auth: any co-author with document access (draft mode only; owner-only in structured mode)
  Returns: Section

POST /api/documents/:id/sections/merge
  Body: { section_ids: string[], merged_title: string }
  Auth: document owner only
  Returns: Section (merged)
```

**Transition UI note:** The "Organize sections" modal shows a drag-and-drop list of all existing draft sections. Owner can: drag to reorder, click to rename, select multiple and merge, assign to co-authors via dropdown. Confirming the modal sets `documents.mode = 'structured'`.
