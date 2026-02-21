# Strategic Scout: zen-sci-portal v0.1 Spec Deep Read
**Interrogating the Tensions, Gaps, and Hidden Questions**

**Date:** 2026-02-18
**Prepared By:** Strategic Scout (Claude)
**For:** Cruz Morales, TresPiesDesign.com
**Scope:** Spec deep-read interrogation; NO code generation, NO rewrites, NO synthesis

---

## Spec Summary

zen-sci-portal is a **self-contained Tauri v2 desktop application** (SvelteKit frontend + Rust backend) designed as the "Create" surface in a 3-surface knowledge platform strategy. Each user installs a local instance pointing to a shared cloud AgenticGateway. The portal bundles AgenticGateway as a sidecar, provides local full-text search via Tantivy, manages JWTs via platform keychain, and embeds MCP Apps (Phase 4) as iframes. All ZenSci servers (6 MCP servers for document conversion) run server-side in the cloud Gateway.

**The core claim:** Local-first persistence + search + auth, powered by Rust, yields a lightweight, responsive knowledge-work forge for academics and researchers.

---

## Tensions Found

### Tension 1: The Sidecar Bundling Question

**What the spec says:**
- Section 1.5 lists AgenticGateway as "Production-Ready, Go" with "Status: Ready to bundle as Tauri sidecar"
- Section 3.2 defines `externalBin` configuration for cross-platform sidecar bundling (Windows MSVC, Linux GNU, macOS universal)
- Section 3.3.4 defines GatewaySidecar module: spawn, health check, restart logic with exponential backoff
- Section 3.5 says "All Gateway requests go through Rust backend via Tauri commands" — no direct frontend-to-Gateway communication

**The tension:**
AgenticGateway is a **production Go binary** running a full orchestration engine (188 files, ~39K LOC, multi-provider routing, MCP host manager, OTEL observability, SQLite memory). Bundling it as a sidecar means:

1. **Size & startup overhead:** The spec claims "10MB bundle, 30–50MB idle memory" but this likely excludes the bundled Gateway binary. Gateway's Go binary (even stripped) is ~50–150MB depending on platform. Total footprint may be 60–150MB, closer to Electron territory.

2. **Platform-specific complexity:** Tauri's sidecar naming convention per platform (gnu, msvc, darwin) requires separate Gateway builds per platform. The spec mentions this briefly but doesn't address: What if Gateway's build pipeline can't keep pace with Tauri releases? Who maintains cross-platform parity?

3. **Local vs. cloud tension:** The spec frames the portal as "cloud-connected social platform" (Section 1.5: "shared cloud AgenticGateway"). But bundling the Gateway locally creates **two Gateway instances per user** — one embedded in the desktop app, one in the cloud. This is fine if they're truly isolated (local for compute, cloud for persistence), but the spec never clarifies:
   - Are local and cloud Gateways in sync?
   - Can a user edit locally then sync to cloud?
   - What's the conflict resolution strategy?

4. **The real question hidden here:** Is bundling the full Gateway actually necessary? Could the portal instead:
   - Bundle a lightweight **sidecar that only runs ZenSci servers** (the 6 TypeScript MCP servers)?
   - Let those servers talk to a **cloud Gateway** for orchestration?
   - This would eliminate the "two Gateway" problem, reduce footprint, and simplify updates.

**The question behind the question:** *Does every user really need their own full copy of AgenticGateway? Or is the sidecar strategy optimized for a different use case (e.g., offline-first, fully disconnected operation) that the spec doesn't explicitly name?*

---

### Tension 2: The Document Catalog Ownership Question

**What the spec says:**
- Section 3.3.1 (SearchIndexer): "Index all user document outputs locally. Provide sub-50ms search."
- The data flow (Section 3, "User Creates a Document") shows: User edits markdown → SvelteKit → Rust → Gateway → latex-mcp → PDF → Rust → local disk + Tantivy index
- Section 3.3.3 (FileWatcher): "Monitor `~/.zen-sci/` folder for document changes. Trigger conversions on file save."

**The tension:**
The spec treats the **local filesystem** as the source of truth. But ZenSci servers run server-side (in the cloud Gateway), and they may have access to documents the user created *through other surfaces* (web portal, mobile, etc.). So the indexing question becomes:

1. **What exactly is being indexed?**
   - Only documents created locally via the desktop app?
   - Or also documents created on the web portal, synced to the local desktop?
   - The file watcher watches `~/.zen-sci/`, but what populates that folder if the user creates a document on the web portal?

2. **Index staleness:** The spec doesn't address: What if a user creates a document on the web portal, then opens the desktop app hours later? The Tantivy index has no knowledge of that remote document until it's... what? Manually synced? Auto-synced on app launch?

3. **The Rust layer ownership question:** Section 3.3.1 describes the SearchIndexer as owning document indexing. But if documents are created server-side (by ZenSci servers), then the authoritative catalog lives in the Gateway's SQLite, not the local Tantivy index. Is Tantivy a **read-only replica** of the Gateway's document catalog? Or the source of truth?

4. **Practical consequence:** If Tantivy is the local replica, it must be kept in sync. The spec mentions "reindex on app startup if metadata changed" (Section 5, Risk Assessment) but doesn't specify:
   - How does the app know if the remote catalog changed?
   - Does it call Gateway on startup to check?
   - What if the user is offline?

**The question behind the question:** *Is the document catalog fundamentally local-first (Tantivy as primary) or cloud-first (Tantivy as cache)? The spec assumes both without reconciling which is authoritative. This affects everything from sync strategy to conflict resolution to offline capability.*

---

### Tension 3: The React-in-Svelte-in-Tauri Iframe Conflict

**What the spec says:**
- Section 1 (Vision): "The frontend (SvelteKit) is consistent with MCP Apps (Phase 4)"
- Section 3.2: "/app/* (MCP App iframes)" routes for embedding Phase 4 apps
- Section 3.3.4: "MCP App embedding: Phase 4 apps (PDF.js viewers, SEO dashboards, compliance panels) render in webview iframes"
- Section 3.4 (AppBridge component): "iframe container for Phase 4 MCP Apps. Handles messaging + CSP sandboxing"
- STATUS.md (Phase 4): "All 6 server index.ts files updated with registerAppTool() + registerAppResource()... Apps are Vite + vite-plugin-singlefile single-file HTML bundles"

**The tension:**
The earlier scout (2026-02-18_scout_zen-sci-portal-rust-web.md) noted that MCP Apps are **React + Vite** single-file bundles. But the portal frontend is **SvelteKit**. The stack becomes:

```
Host: SvelteKit (Svelte 5 fine-grained reactivity)
  ↓ embeds
Iframe: MCP App (React 18 via Vite single-file)
  ↓ both inside
Tauri Webview (native OS webview: WebKit/Edge)
```

**The architectural awkwardness:**
1. **Framework mismatch:** React's virtual DOM inside a Svelte store-driven architecture creates impedance mismatch. If the host needs to communicate state to the MCP App iframe, it must cross the iframe boundary (postMessage API), lose type safety, and manually serialize/deserialize.

2. **CSP complexity:** Section 3.2 defines CSP:
   ```json
   "default-src": ["'self'", "ui://", "data:", "https:"],
   "script-src": ["'self'", "'wasm-unsafe-eval'"]
   ```
   Allowing `ui://` for bundled HTML (Phase 4 apps) is correct, but React inside an iframe has its own JSX compilation, event delegation, and ref management. Testing this interaction is non-trivial.

3. **The hidden assumption:** The spec says "Phase 4 apps render in webview iframes" but **never tests this integration**. What if:
   - The React iframe can't efficiently sync state with the SvelteKit host?
   - The CSP sandbox prevents React from accessing needed browser APIs?
   - Hot Module Replacement (HMR) doesn't work across iframe boundaries?

**Why this matters:** The spec promises "Zero-friction onboarding" (Goal 4) but embeds a React app inside a Svelte shell inside a Tauri webview. Each layer has different event handling semantics. If a user clicks a button in the React PDF.js viewer and expects the SvelteKit host to respond, the message flow is: React → postMessage → window listener → SvelteKit store → UI update. This is not zero-friction.

**The question behind the question:** *Is the MCP App embedding strategy proven on the SvelteKit + Tauri stack? Or is it a theoretical design that may surface integration pain during implementation?*

---

### Tension 4: The Offline-First Assumption Without Offline Specification

**What the spec says:**
- Goal 4: "Zero-friction onboarding: First launch → login (optional, can use locally)"
- Section 3.3.1: "Offline-first knowledge discovery" (Tantivy search)
- Success criteria: "Offline search works when Gateway unavailable" (Section 2)
- Risk Assessment: "If Gateway crashes → show 'Gateway offline' error. User can still browse local documents + search."

**The tension:**
The spec claims offline support but doesn't specify **what "offline" means**:

1. **No cloud sync in v1.0:** Section 3 explicitly notes "❌ Web-based portal... Web is v2.0+ scope" and "❌ Real-time collaboration in v1.0". This means v1.0 has **no cloud sync mechanism**. So if a user creates a document on the desktop, it stays on the desktop unless they manually move it to cloud.

2. **Incomplete offline scenario:** What happens if:
   - User opens portal offline (no Gateway, no network)
   - User clicks "search" → works (local Tantivy)
   - User clicks "convert document to PDF" → fails (requires Gateway + ZenSci servers)
   - User tries to login → fails (needs Gateway auth endpoint)

3. **The missing offline affordance:** The spec says "optional login" but if the user is offline and hasn't logged in yet, they can't authenticate to the Gateway later. The keychain can cache a token, but what if the token expires while offline?

4. **Practical consequence:** The offline claim is technically true (Tantivy search works offline) but misleading. A knowledge-work tool that can't convert documents or sync to the cloud is read-only offline. For an academic researcher, this is **not usable offline** in any meaningful sense.

**Why this matters for the spec:** Section 5 (Risk Assessment) lists "User expectations for 'cloud' not met" as a medium-likelihood, high-impact risk. The mitigation is "Clear UX messaging: local folder icon vs. cloud icon." But the messaging can't hide the fact that v1.0 is **local-first in design, but online-required in practice**.

**The question behind the question:** *Is the portal truly "offline-first," or is it "offline-capable for read-only access"? The spec conflates the two. Which one is the actual design goal?*

---

### Tension 5: The Tauri IPC Granularity Question

**What the spec says:**
- Section 3.6 lists 11 Tauri IPC commands: `search`, `convert_document`, `login`, `logout`, `check_session`, `list_documents`, `create_document`, `get_gateway_status`, `get_mcp_app`, `watch_folder`, `stop_watcher`
- Each command has specific input/output types (e.g., `search` → SearchResults)
- Section 3.5 (Data Flow): "All Gateway requests go through Rust backend"

**The tension:**
The command set seems reasonable, but there's no guidance on **granularity**:

1. **Is search a Tauri IPC call or a local HTTP server?**
   - The spec shows search as a command (`invoke("search", {query, limit})`)
   - But if we're running a Tantivy index in the Rust backend AND a full AgenticGateway sidecar, why not expose the Gateway's OpenAI-compatible chat API directly to the frontend?
   - Instead, we're forcing all queries through Rust IPC, which means Rust must deserialize, call Tantivy, serialize results back

2. **Is the Rust layer too thin or too thick?**
   - Thin interpretation: Rust is just glue (spawn Gateway, watch files, manage auth tokens)
   - Thick interpretation: Rust owns search, file metadata, session state
   - The spec describes a "thick" Rust layer, but there's tension: If the Gateway has its own search API, why duplicate search in Tantivy?

3. **Token size and serialization overhead:** Every `invoke()` call serializes the request to JSON, crosses the Tauri boundary, Rust deserializes, executes, serializes response back, frontend deserializes. For a search query returning 50 results, this is acceptable. But what about **high-frequency operations**?
   - File watcher events: Every file save triggers a FileEvent. Does this come through IPC or through a background tokio task?
   - Presence updates (v1.1): Real-time presence would be too slow over IPC. The spec defers this to WebSocket (Section 3.4, Item 3), but doesn't show the WebSocket server architecture.

4. **Practical question:** Is the IPC surface designed for v1.0 scope or will it need rework for v1.1?

**The question behind the question:** *Is the IPC command set optimized for latency and usability, or is it a reasonable-but-naive first pass that assumes "control plane" operations are acceptable over Tauri IPC?*

---

### Tension 6: The 3-Surface Handoff Layer Gap

**What the spec says:**
- Vision (intro): "local-first, knowledge-work forge... Create surface in a 3-surface strategy (Create / Connect / Consume+Contribute)"
- Section 1.5: "The cloud layer (AgenticGateway + ZenSci) is unchanged — the portal is a new surface, not a platform overhaul"
- But the spec focuses entirely on the desktop portal. It doesn't mention:
  - Web portal (v2.0+)
  - Mobile (out of scope)
  - How a document created on desktop appears on the web portal
  - How cloud sync works, if at all

**The tension:**
The 3-surface strategy assumes **seamless handoffs** between desktop, web, and mobile. But the portal spec describes the desktop in isolation. Critical gaps:

1. **No sync specification:** If I create a document locally (`~/zen-sci/myresearch.md`), convert it to PDF, where does that PDF live?
   - Local disk only? Then how does it appear on the web portal?
   - Synced to cloud? Which cloud service? S3? Database? When?
   - The spec mentions "Cloud sync (optional v1.1)" but v1.0 has no story.

2. **No document sharing specification:** Section 2 lists Goal 3: "Multi-user device support." But what about **document sharing across users** (e.g., collaborative editing)?
   - Section 2 explicitly marks "❌ Real-time collaboration in v1.0"
   - So v1.0 has no collaboration story at all
   - This limits the "social platform" claim to single-user desktop use

3. **No handoff affordance:** How does a user move a document from desktop to web?
   - Manual export/import?
   - Click a "share to cloud" button?
   - The spec doesn't define this UX

4. **The unspoken assumption:** The spec assumes the cloud Gateway is the **single source of truth**. But if the desktop app can work offline with local-only documents, the system now has **two sources of truth**: local Tantivy/filesystem + cloud Gateway/S3.

**Why this matters:** The spec promises "Prove Tauri viability for knowledge platforms" (Primary Goal 5). But without the handoff layer, it's only proving Tauri viability for a **single-user, single-device knowledge tool**. The "social platform" and "knowledge platform" claims require cross-surface handoffs.

**The question behind the question:** *Is the portal v1.0 designed as a standalone single-user app, or as the Create surface of a larger 3-surface platform? The spec claims both but specifies only the standalone case.*

---

## The Reframe

**If I had to identify the single reframe that would improve this spec most significantly, it is this:**

**What the spec should explicitly answer:**

> **"The zen-sci-portal is not a cloud-connected platform in v1.0. It is a standalone offline-capable single-user knowledge forge. The 3-surface strategy (Create/Connect/Consume) is the long-term vision, but this spec describes only the Create surface, running in isolation, with the assumption that cloud sync happens in v1.1 or later. To avoid overpromising, the spec should remove references to 'social,' 'multi-user device support,' and 'platform' until the handoff and sync layers are specified."**

**Stated as a question:**

*Is zen-sci-portal v1.0 a self-contained knowledge-work tool (desktop-only, single-user, local-first), or the Create component of a cloud-connected platform? The spec claims both. Which is true?*

If the answer is **"self-contained tool for v1.0"**, then:
- Remove cloud sync promises
- Reframe "offline-first" as "works without Gateway after initial setup"
- Drop the "social" and "3-surface" language
- Focus on demonstrating that Tauri + local search + local auth is a compelling UX for solo researchers

If the answer is **"Create surface of a platform"**, then:
- Specify the sync layer (S3 + metadata store)
- Define document ownership and sharing
- Specify the handoff UX (how does a local doc move to cloud?)
- Clarify whether local and cloud Gateways sync state

This reframe is not a fix to the spec—it's a clarification of what the spec is actually trying to accomplish. The implementation will be very different depending on which vision is correct.

---

## Gaps

**5 critical things the spec doesn't address:**

1. **Gateway Sidecar Update Strategy**
   - *What's specified:* AgenticGateway is bundled as a sidecar, health-checked every 10 seconds, auto-restarted on crash
   - *What's missing:* How does the user update AgenticGateway when a new version is released? Does the entire app need to be reinstalled? Can the sidecar be updated independently? What's the version mismatch risk?
   - *Why it matters:* In production, Gateway will be updated (security patches, new MCP servers, bug fixes). The spec has no story for this.

2. **Tantivy Index Consistency in Multi-Device / Multi-Process Scenarios**
   - *What's specified:* Tantivy index lives in `~/.zen-sci/search-index/` with WAL durability. "Reindex on app startup if metadata changed."
   - *What's missing:* What if the user opens the portal on two different machines with the same home directory (e.g., shared NFS mount, or cloud-synced home folder)? Two instances of Tantivy will try to write to the same index. Tantivy is not designed for concurrent writes across processes.
   - *Why it matters:* Modern knowledge workers use multiple machines. This scenario is not unrealistic.

3. **Conflict Resolution Policy for Offline Edits**
   - *What's specified:* "Offline search works when Gateway unavailable." "File sync queue for ZenSci processing."
   - *What's missing:* What if the user edits a document offline, then later the document is updated on another device (or the web portal)? When the desktop app comes back online, which version wins? The spec mentions "Last write wins" in the earlier scout, but the spec itself doesn't say.
   - *Why it matters:* This is a data loss scenario. It needs explicit specification and clear UX messaging.

4. **Keychain Fallback & Failure Modes**
   - *What's specified:* JWT stored in platform keychain (macOS Keychain, Windows DPAPI, Linux Secret Service). Risk assessment mentions "Keychain integration platform-specific failures."
   - *What's missing:* What if the keychain is unavailable or corrupted? The spec mentions "encrypted file storage fallback" but doesn't design it. Also: What if the user has multiple devices and wants to share their JWT across them? Keychains are device-specific.
   - *Why it matters:* On Linux, the Secret Service may not be running in a headless environment, or it may be unavailable in certain desktop configurations. The app needs graceful degradation.

5. **MCP App Security & Message Passing Protocol**
   - *What's specified:* MCP App iframes embedded with CSP, `ui://` protocol for bundled HTML, "iframe sandboxing, CSP, message passing."
   - *What's missing:* What's the actual message protocol between the host (SvelteKit) and the embedded MCP App (React iframe)? How does the app request data from Tantivy, or trigger a document conversion? The spec says "AppBridge component: iframe sandboxing, CSP, message passing" but provides zero implementation detail.
   - *Why it matters:* This is the critical integration point for Phase 4 apps. Without specifying the protocol, teams building MCP Apps won't know how to communicate with the host.

---

## What to Keep

**5 things the spec gets right and should survive revision:**

1. **Sidecar Bundling as a Force Multiplier for Adoption**
   - The insight that bundling AgenticGateway eliminates the "manual setup hell" is correct. One-click launch is a massive UX win over requiring users to `docker-compose up`.
   - Even if the implementation details need refinement, this principle should survive.

2. **Tantivy as the Local Search Engine**
   - The decision to use Tantivy (not just rely on Gateway search) is strategically sound. Local search at <50ms is a compelling UX that offline-first tools (Obsidian, Roam) have proven users love.
   - Tantivy is production-grade and well-suited to this role.

3. **Platform Keychain Integration for Auth**
   - Storing JWTs in the platform keychain (not localStorage) is the right security call. Multi-user device support without shipping unencrypted secrets is a win.
   - This design will likely carry forward unchanged.

4. **File Watcher + Auto-Indexing for Low-Friction UX**
   - The user saves a markdown file, the desktop app detects it (via notify crate), queues it for ZenSci conversion, and indexes the output. This is elegant and requires zero user action.
   - This pattern works well in other knowledge tools (Obsidian, Dendron) and should survive.

5. **Clear Separation of Concerns: Rust Backend Owns Persistence & Security**
   - The architectural choice to have Rust own local search, auth, and file watching (while SvelteKit owns UI) is clean. Each layer has a clear responsibility.
   - This separation of concerns will scale better than a JavaScript-only app.

---

## Recommended Questions for Cruz

**Before the next revision, answer these 3–4 questions to unlock the next iteration:**

### Q1: v1.0 Scope Clarification — "Self-Contained Tool" vs. "Platform Create Surface"

**Context:** The spec claims both "local-first knowledge-work forge" (standalone) and "Create surface in a 3-surface strategy" (platform). These are different products.

**The question:**
> Is zen-sci-portal v1.0 explicitly designed as a **self-contained tool that can be used entirely offline and locally** (like Obsidian Desktop), or as a **component of a larger cloud-connected platform** (like Notion Desktop)?

**Why it matters:**
- If standalone: Drop cloud sync promises, reframe "offline" claims, simplify the spec.
- If platform component: Specify the cloud layer (S3, metadata store, handoff UX) now, even if implementation is v1.1+.

**Suggested answer format:**
"v1.0 is a standalone offline-capable tool. Cloud sync and the 3-surface handoff are v1.1 scope. This means [specify what is and isn't possible in v1.0]."

---

### Q2: The Gateway Duplication — Why Bundle the Full Gateway Instead of Just ZenSci Servers?

**Context:** The spec bundles the entire AgenticGateway (188 files, ~39K LOC) as a sidecar. But the portal's primary need is **document conversion** (run ZenSci servers locally). Why not bundle only the TypeScript MCP servers and point them to a cloud Gateway?

**The question:**
> What capability does bundling the **full Go Gateway** unlock that bundling only the **6 ZenSci MCP servers** would not?

**Specific sub-questions:**
- Is the goal to provide **offline document conversion** (no cloud call required)? If so, say so explicitly.
- Or is the goal to provide **local orchestration** (plan multi-step workflows locally)?
- Or is the goal to provide a **self-contained demo** that doesn't depend on cloud infrastructure for development?

**Why it matters:**
- Full Gateway bundle: Higher footprint, simpler setup, fully offline capable
- ZenSci-only: Lower footprint, cleaner separation of concerns, but requires cloud Gateway for orchestration

**Suggested answer format:**
"The sidecar bundles the full Gateway because [specific offline or local-compute requirement]. This is worth the [size/complexity] tradeoff because [value proposition]."

---

### Q3: The Document Catalog Ownership — Local-First or Cloud-First?

**Context:** The spec assumes documents are created locally and optionally synced to the cloud. But ZenSci servers run on the cloud, so they're creating documents on the cloud side. The indexing strategy assumes Tantivy is the source of truth, but the authoritatively stored documents live in the cloud Gateway.

**The question:**
> Is the **local filesystem + Tantivy index** the source of truth for documents, or is the **cloud Gateway + S3 storage** the source of truth?

**Specific sub-questions:**
- If a user creates a document on the web portal (v2.0+), and then opens the desktop portal, does that document automatically appear in the local Tantivy index? If so, how?
- If a user edits a document locally and loses network connectivity, what happens when they come back online?
- Is Tantivy a read-only cache or the primary index?

**Why it matters:**
- Local-first: Simpler sync (local → cloud), but requires cloud polling on startup
- Cloud-first: Cleaner authoritative source, but requires cloud connection for completeness

**Suggested answer format:**
"Documents are authored locally and synced to the cloud on [condition]. Tantivy is a [read-only cache / primary index] because [architectural rationale]. When a user opens the desktop portal, it [pulls from cloud / checks local state] to ensure [consistency property]."

---

### Q4: The MCP App Embedding Protocol — How Do React Iframes Talk to the SvelteKit Host?

**Context:** Phase 4 MCP Apps are React single-file bundles embedded in SvelteKit iframes. The spec mentions "AppBridge component: iframe sandboxing, CSP, message passing" but provides zero implementation detail.

**The question:**
> What is the **exact message protocol** between the SvelteKit host and an embedded MCP App iframe?

**Specific sub-questions:**
- How does the React app inside the iframe request the user's current document?
- How does the React app trigger a Tauri IPC command (e.g., "convert this to PDF")?
- How does the SvelteKit host notify the React app of state changes (e.g., "the user selected a different document")?
- Is there a shared context/store, or only postMessage?

**Why it matters:**
- This is the critical integration point for v1.0 implementation and Phase 4 adoption.
- Teams building MCP Apps need to know this protocol before they can build reliably.
- CSP and iframe boundaries create significant constraints that need explicit design.

**Suggested answer format:**
"MCP Apps communicate with the host via [postMessage / custom protocol]. The host sends [list of command types] and the app responds with [list of result types]. Apps can request data by [mechanism], and the host pushes updates via [mechanism]."

---

## Conclusion

This spec is **implementation-ready on the surface** but rests on three unspoken assumptions:
1. The portal is v1.0 scope only (no cloud sync, no collaboration)
2. The bundled Gateway is worth the footprint tradeoff (offline-capable operation)
3. The local filesystem + Tantivy are the authoritative document store (until v1.1 sync)

None of these are wrong, but **they're not articulated in the spec**. As a result, the spec overpromises on features (offline-first, social, 3-surface) that aren't actually specified, and underpromises on the offline capability (can't convert documents without the bundled Gateway).

**Recommended path forward:**
1. Answer Q1–Q4 above to clarify scope and assumptions
2. Add a "v1.0 Scope & Limitations" section that explicitly states what is and isn't in v1.0
3. Rewrite Sections 1 (Vision) and 2 (Goals) to match the clarified scope
4. Add a "Phase 1.1 Roadmap" section that details cloud sync, handoff, and collaboration
5. Specify the MCP App message protocol and create a companion "MCP App Developer Guide"

---

**Scout prepared by:** Claude (Strategic Scout)
**Status:** Ready for decision and architecture lockdown
**Next gate:** Answer Q1–Q4, then commission implementation prompts

