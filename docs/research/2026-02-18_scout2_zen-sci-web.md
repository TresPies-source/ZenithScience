# Strategic Scout: zen-sci-web v0.1 — Finding the Questions Behind the Questions

**Date:** 2026-02-18
**Scout:** Claude (Opus-level interrogation)
**Status:** Pre-Implementation Strategic Analysis
**Audience:** Cruz Morales, Architecture Decision Makers

---

## Spec Summary

**zen-sci-web v0.1** is a SvelteKit-based hosted web portal that serves as the **"Connect" surface** in a 3-surface strategy (portal / desktop / mobile). It enables:
- Group discovery and onboarding via invitation flows
- Async group collaboration (activity feeds, project browsing, member management)
- Rendering of Phase 4 MCP App companions in browser iframes
- Seamless handoff from web discovery to desktop app installation

The spec positions the portal as a **stateless SvelteKit server** proxying to the shared AgenticGateway, with all user data stored in Gateway's SQLite memory. Success criteria emphasize fast page loads (<3s), responsive design (320px–1920px), and comprehensive test coverage.

---

## Tensions Found

### Tension 1: The "Same Gateway" Assumption — Is the Gateway Actually a Social Platform?

**What the spec says:**
Both the web portal and desktop app connect to the same AgenticGateway v1.0.0. The Gateway is described as production-ready with OpenAI-compatible chat API, tool registry, MCP integration, and SQLite memory management.

**What the tension is:**
The Gateway's architecture, as documented in ARCHITECTURE.md, is designed as a **tool orchestration platform** — DAG-based execution, LLM routing, MCP tool calling, skill invocation. It's purpose-built for agentic reasoning, not social features. Yet zen-sci-web requires:
- Groups (hierarchical collections with members + roles)
- Projects (collections within groups)
- Activity feeds (time-ordered events with typed metadata)
- Invitation tokens (48-hour TTL, claim flow, email-to-portal onboarding)
- Profile pages (user document history, membership roster)
- Metadata storage for documents (title, author, module, status, word count, math expressions)

The spec assumes these entities "live in Gateway memory" — but the Gateway specs don't show a social data model. There's no `POST /v1/gateway/memory/upsert` for "create a group," no concept of roles, no invitation management. The ARCHITECTURE.md shows SQLite-based memory for Garden metaphor (seeds, snapshots), not social entities.

**The question behind the question:**
*Is the social layer (Groups, Projects, Invitations) a first-class citizen in the Gateway's data model, or is the web portal expected to implement its own social storage layer and use the Gateway only for tool orchestration and auth?*

If it's the former, Gateway needs significant schema expansion. If it's the latter, the web portal becomes a **stateful application** (not stateless), requiring either:
- Its own database (PostgreSQL, MongoDB)
- Filesystem-based storage (JSON files) with sync-on-startup
- A separate microservice (auth + social) alongside the Gateway

The spec glosses this over with: *"For v0.1, the Portal assumes Gateway memory endpoints exist. If not, portal uses basic file-based storage (JSON files in Gateway's data directory) as fallback."*

This is not a detail — it's the architectural boundary between the two systems.

---

### Tension 2: The Auth Identity Question — JWT in Browser vs. Keychain

**What the spec says:**
Auth flow: User submits email/password → SvelteKit calls Gateway `/v1/chat/completions` with auth intent → returns JWT → stored in secure, HttpOnly cookie + localStorage → all subsequent requests include `Authorization: Bearer {JWT}`.

The desktop app (zen-sci-portal spec) uses JWT + platform keychain (macOS Keychain, Windows DPAPI, Linux Secret Service).

**What the tension is:**
The two specs describe fundamentally different auth models:
- **Web (zen-sci-web):** JWT in browser (secure cookie + localStorage). Browser-based auth, typical SPA pattern.
- **Desktop (zen-sci-portal):** JWT in platform keychain. Secure credential storage, multi-user device support.

The web spec never clarifies who the identity provider is. It calls Gateway with an "auth intent," but Gateway is a tool orchestrator, not an identity provider. There's no mention of:
- Password hashing (bcrypt? argon2?)
- Session tables (user_id, email, created_at, expires_at)
- Token rotation (refresh tokens?)
- Multi-factor auth
- Account creation flow

Is the Gateway expected to become an identity provider? Or is there a separate auth service (e.g., Keycloak, Auth0, a custom Flask service)?

**The question behind the question:**
*Where is the source of truth for user credentials and session state? Is it the Gateway (which would require adding an auth module), or a separate service that the portal and Gateway both trust?*

This affects:
- Portal architecture (self-contained SvelteKit vs. dependent on external auth service)
- Deployment complexity (single Gateway service vs. multi-service auth infrastructure)
- Desktop-to-web credential handoff (can the desktop app's keychain-stored JWT be used on the web, or are they separate auth realms?)

---

### Tension 3: MCP Apps in Browser — Where Does the AppBridge Protocol Live?

**What the spec says:**
The web portal is supposed to render Phase 4 MCP App companions (React iframes). The spec shows:
```
Portal: POST /api/mcp-apps/[module]/manifest
Server returns: { module: 'latex-mcp', resource_uri: 'ui://latex-mcp/preview.html', ... }
Portal renders: <iframe srcDoc={appHtml} sandbox="allow-scripts allow-same-origin" />
```

And describes a postMessage bridge:
```typescript
window.addEventListener('message', (e) => {
  if (e.origin !== window.location.origin) return;
  const { type, payload } = e.data;
  switch (type) {
    case 'app:ready':
      iframeRef.current?.contentWindow?.postMessage({
        type: 'portal:document',
        payload: { document, editable: false }
      }, '*');
    // ...
  }
});
```

**What the tension is:**
MCP Apps were designed to run inside Claude and the AgenticGateway's AppBridge infrastructure. Phase 4 specs define an `@modelcontextprotocol/ext-apps` SDK with a specific resource URI pattern (`ui://`).

But the web portal is a different host — a SvelteKit app running on Node.js, not the AgenticGateway. The spec assumes:
1. MCP App HTML bundles are served by SvelteKit at static paths or fetched from somewhere
2. The portal implements its own postMessage bridge (not shown in detail)
3. The portal's AppBridge is compatible with the AgenticGateway's AppBridge protocol

**Critical gap:** The spec doesn't specify:
- Where MCP App bundles live (bundled in portal? fetched from Gateway? served from a CDN?)
- The exact postMessage protocol (MCP App spec defines it, but portal integration is vague)
- Whether the portal's AppBridge can call back to Gateway tools or is limited to document rendering
- CSP (Content Security Policy) rules needed to allow `ui://` iframes without compromising security

**The question behind the question:**
*Is the portal's AppBridge a full implementation of the MCP App protocol, or a simplified read-only viewer? If the MCP App needs to call Gateway tools (e.g., "export to PDF" from a PDF viewer), how does that message flow back through the portal to the Gateway?*

This could mean the portal is either a **simple iframe host** (minimal integration) or a **full AppBridge implementation** (portal becomes a second host for MCP Apps, requiring protocol-level sync with Gateway).

---

### Tension 4: The "Social" Definition Gap — What Does "Social" Actually Mean Here?

**What the spec says:**
"Async group collaboration, discovery, and sharing." Features include:
- Group discovery & onboarding
- Activity feeds (who created what, what outputs were produced)
- Group & project management
- Document sharing & discovery
- Invitation flows
- Profile pages

**What the tension is:**
The spec is ambiguous about what students (or users) actually **do** on the web portal. Consider three different product models:

1. **GitHub-like async collaboration:**
   - Users fork projects
   - Submit pull requests (or submit documents for feedback)
   - Comment on artifacts
   - Merge changes
   - Full version control integration

2. **Course LMS (Learning Management System):**
   - Instructor uploads assignments
   - Students submit work
   - Instructor grades + provides feedback
   - Grade book visible to students
   - Activity feed shows submissions, feedback

3. **Slack-like communication:**
   - Users post in group channels
   - Chat in real-time (or async)
   - Search message history
   - Share documents
   - Typing indicators, reactions

The zen-sci-web spec doesn't clarify which model it's following. The routes suggest GitHub-like (project browsing, document viewing), but there's no PR flow. The activity feed suggests LMS-like (track what was created), but no assignment/submission model. There's no mention of comments, feedback, or async discussions.

**The question behind the question:**
*Is the web portal a discovery/viewing surface only (read-only, no contribution), or is it a collaboration platform where students can comment, submit work, and iterate?*

If it's read-only, the phrase "async group collaboration" is misleading. If it's collaborative, the spec needs:
- Comment threads on documents
- Submission/approval workflows
- Feedback annotation UI
- Version history (compare v1 vs. v2)
- Access control (who can comment? who can edit?)

This determines whether the portal is a **thin client** (just renders data from Gateway) or a **stateful app** (tracks discussion threads, permissions, collaboration workflows).

---

### Tension 5: Content Ownership & Sync Model — Local vs. Cloud, Who Owns What?

**What the spec says:**
When a student creates a document on the desktop app, it converts to PDF/HTML/email. The spec mentions "appears on the web portal" as `/groups/bio-101/projects/photosynthesis/document/draft-1`.

**What the tension is:**
The spec never clarifies the ownership and sync model:

1. **Where is the document stored?**
   - On the student's local machine (desktop app)?
   - On the Gateway's SQLite database?
   - On cloud storage (S3)?
   - All three (with sync)?

2. **What does "appears on the web portal" mean?**
   - A live link to the same document (one copy, two views)?
   - A copy synced from desktop to cloud (eventual consistency)?
   - A web-only version created after desktop conversion (two separate artifacts)?

3. **Who owns the document?**
   - The student (created it)?
   - The group (created within group context)?
   - The instructor (if it's a course assignment)?

4. **Sync semantics:**
   - Does editing the same document on desktop and web cause conflicts?
   - Is there a "publish to portal" button, or automatic sync?
   - Can a desktop user see changes made on the web, and vice versa?

The spec says: *"When a student creates a document on the desktop, who owns it? Is it stored on the Gateway, on the student's machine, or both? When it 'appears' on the web portal, is that a link, a copy, or a live view?"*

This is not answered. The implementation plan (Phase 3) says "Project listing: Groups display list of projects (markdown documents + ZenSci outputs)" — but doesn't specify whether these are fetched from Gateway or synced from user machines.

**The question behind the question:**
*Is the web portal a read-only view into cloud-stored documents, or does it support collaborative editing with sync semantics? This determines whether conflicts can occur, who wins on conflict, and whether the portal needs CRDT (Conflict-free Replicated Data Type) or simpler last-write-wins semantics.*

---

### Tension 6: The Mobile Handoff Pipeline — Publish to Mobile, or Automatic Derivation?

**What the spec says:**
"Seamless desktop handoff: Invitation flow: email → web portal → desktop app deep link. User discovers on web, then installs desktop to contribute locally."

The zen-sci spec mentions a Phase 4 (mobile consumption surface) where documents become "learning cards."

**What the tension is:**
The spec mentions desktop → mobile but doesn't address:
- Who creates mobile-optimized content?
- Is the web portal the bridge, or is mobile content auto-generated from desktop conversions?
- What format is a "learning card"? (SRS flashcard? Interactive slide? Excerpt?)
- Who controls the mobile → card transformation? (Portal admin? User?)

Example flow missing from the spec:
```
Student creates: "Photosynthesis mechanism" (markdown on desktop)
  ↓
Desktop converts to: PDF + HTML + email (3 outputs)
  ↓
Web portal shows: all 3 outputs + activity feed
  ↓
??? Now what?
Mobile sees: Learning cards for "Photosynthesis mechanism"
  ↓ But where did these cards come from?
- Extracted from PDF summary?
- Hand-authored by instructor?
- Auto-generated from HTML headings?
- Derived from email outline?
```

The "handoff" direction is also unclear: Does the portal initiate the mobile transformation ("Publish to Mobile" button), or does the mobile app pull from a feed?

**The question behind the question:**
*Is the web portal responsible for creating mobile-ready content, or is mobile content auto-generated from the desktop conversions? Does the portal expose a mobile API endpoint, or is mobile a separate consumer of the same Gateway data?*

This affects:
- Portal's scope (view-only vs. content curation)
- Mobile architecture (fed by portal, or independently consumes Gateway?)
- Content ownership (who decides what becomes a learning card?)

---

## The Reframe

If I had to identify **one key reframe** that would improve the spec most significantly, it would be:

### **"Clarify the boundary between the portal as a thin client and the portal as a stateful social platform."**

**The question that unlocks clarity:**
*Is zen-sci-web primarily a **discovery and consumption surface** (read-only access to documents and projects stored in the Gateway), or is it a **collaborative workspace** where groups can discuss, provide feedback, and iterate on shared documents?*

**Why this matters:**
- If discovery/consumption: Portal is thin, stateless, proxies to Gateway. Scope is ~10 weeks, tech risk is low.
- If collaborative: Portal needs its own database for discussions, access control, workflows. Scope is 16–20 weeks, tech risk is medium (sync, conflict resolution, permissions).

This single reframe cascades through all other tensions:
- Auth identity: Discovery portals can use simple JWT; collaborative workspaces need richer user/role models
- Social definition: Clarifies whether "async collaboration" means comments/feedback or just async viewing
- Content ownership: Read-only = no conflicts; collaborative = need sync/CRDT semantics
- MCP App integration: Simple iframes for rendering; or full AppBridge for interactive feedback/submission
- Mobile handoff: Portal feeds mobile; or mobile is independent consumer

---

## Gaps

**The spec doesn't address:**

1. **Database Architecture for Social Entities** — Groups, invitations, activity events, permissions. Currently assumed to live in Gateway memory, but Gateway specs don't define the schema. Recommend: either extend Gateway with social schema, or add a PostgreSQL service alongside Gateway.

2. **Comment / Feedback Thread Data Model** — If the portal supports collaboration (even async), need: comment text, author, timestamp, threading, access control. Not mentioned.

3. **Conflict Resolution Strategy** — If same document edited on desktop and web, how are conflicts resolved? Last-write-wins? CRDT? Manual merge? Not specified.

4. **Mobile Content Transformation** — How markdown → learning cards? Who decides? How is mobile data related to desktop data? Not addressed.

5. **Role-Based Access Control (RBAC) Spec** — Groups have roles (owner/member/viewer), but the spec doesn't detail what each role can actually do. Can viewers comment? Can members delete projects? Not specified.

6. **Offline Support Strategy** — What happens if the Gateway goes down? Can users still view cached documents? Can they create new ones? Portal can't work offline (it's web), but should graceful degradation be specified?

7. **Email Infrastructure for Invitations** — Spec says "invitation email sent to Bob," but doesn't specify: SendGrid? Postmark? Custom SMTP? Email template? Retry logic? Spam handling?

8. **Activity Feed Retention & Pagination** — Spec says "50-item paginated feed," but doesn't specify: how many days of activity? What triggers older activity deletion? Pagination token format?

9. **Search & Filtering Strategy** — Spec mentions "filter by module type" and "sort by date/name," but no full-text search over document content. Is that out of scope for v0.1? (Probably yes, but should be explicit.)

10. **MCP App Security & CSP** — Spec defines CSP for iframes, but doesn't specify: how are CSP violations handled? Can users request `unsafe-eval`? What if an MCP App needs external CDN resources?

---

## What to Keep

**The spec gets these things right:**

1. **Invitation flow is well-designed** — 48-hour tokens, email → public preview → signup → group join is a natural UX. Keep this.

2. **Stateless SvelteKit architecture** — Proxying all data requests to Gateway (rather than maintaining local copies) keeps the portal simple and deployable. Good choice if Gateway social schema is figured out.

3. **SSR for discovery routes** — Landing page, public group directory, and group previews benefit from server-side rendering (SEO, fast initial load). Activity and projects can be SPA. Hybrid approach is pragmatic.

4. **Responsive design emphasis** — 320px–1920px coverage + mobile-first components show the team understands accessibility. Keep this.

5. **Comprehensive test coverage** — Spec calls for unit tests (components), integration tests (auth flows), and E2E tests (critical paths). Test strategy is sound.

6. **Graceful degradation** — "If Gateway unavailable, show maintenance mode" is the right UX pattern. Prevents blank screens.

7. **Phase 4 MCP App design** — Idea of rendering interactive app companions in iframes is good. The postMessage bridge pattern is standard. Keep this; just need to clarify protocol detail.

8. **Phase handoff clarity** — Spec is explicit about v0.1 scope: no real-time collab, no full-text search, no file uploads, no custom branding. Deferred features are listed. Good boundary-setting.

---

## Recommended Questions for Cruz

**To unlock the next spec revision, ask Cruz:**

1. **Social Layer Ownership:** Does the Gateway become a social platform (with Groups, Invitations, Activity tables), or does the portal implement its own social database? If portal owns it, are we adding PostgreSQL or using JSON files?

2. **Portal as Workspace or View?** Can users comment/provide feedback on documents in the portal, or is it read-only discovery? This determines scope and architecture.

3. **Mobile Content Pipeline:** Who/what creates "learning cards" for mobile? Is the portal responsible for curating desktop conversions into mobile format, or is mobile an independent consumer of the same Gateway data?

4. **Auth Boundaries:** Is the Gateway responsible for password hashing and session management (making it an identity provider), or is there a separate auth service? Can a desktop app's keychain JWT be used for web auth, or are they separate realms?

5. **Collaboration & Sync Priorities:** For Phase 1.1 (real-time collab), should the portal support live presence/typing indicators, or start with async-only and add real-time later? This affects whether we need Tokio WebSocket or just polling.

---

## Summary

**zen-sci-web v0.1 is a well-scoped spec** with clear deliverables and success criteria. But it glosses over six architectural tensions that will surface during implementation:

1. Is the Gateway becoming a social platform, or is the portal getting its own database?
2. Who provides auth (Gateway, separate service, or built into portal)?
3. How do MCP Apps integrate when the portal is a different host than AgenticGateway?
4. What does "social" and "async collaboration" actually mean for users?
5. What's the content ownership and sync model across desktop/web/mobile?
6. How are documents transformed for mobile consumption?

**The key reframe:** Clarify whether the portal is a **thin read-only client** or a **stateful collaborative workspace**. This single decision cascades through all other design choices.

**Recommendation:** Before Phase 2 spec revision, schedule a 1-hour architecture session with Cruz to answer the 5 questions above. Once those are locked, the next scout can validate the spec against those decisions and identify any remaining gaps.

---

**Document prepared by:** Strategic Scout (Claude, Opus)
**Date:** 2026-02-18
**Status:** Ready for Architecture Review
**Next Step:** Schedule decision session with Cruz on questions 1–5 above; then commission Phase 2 spec revision
