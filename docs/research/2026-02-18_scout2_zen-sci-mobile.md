# Strategic Scout: zen-sci-mobile v0.1 — Tensions, Gaps, and Reframe Opportunities

**Date:** 2026-02-18
**Status:** Strategic Scouting (No Code)
**Audience:** Cruz Morales, Architecture Decision Makers
**Grounded In:** zen-sci-mobile-v0.1-spec.md, zen-sci-web-v0.1-spec.md, 2026-02-18_scout_zen-sci-portal-rust-web.md, STATUS.md

---

## Spec Summary

zen-sci-mobile v0.1 is a **Tauri v2 iOS + Android app** that transforms academic content (PDFs, slides, blog posts) from the ZenSci platform into mobile-optimized learning formats: swipeable card decks, quizzes, reading views, and video scripts. The app operates in consumption-only mode (students read, quiz, learn; contributions deferred to v2). Content syncs from AgenticGateway when online; remains available offline once cached locally in rusqlite. The architecture layers SvelteKit UI atop a Rust backend (ContentCache, AuthManager, SyncEngine) coordinated via Tauri IPC. The spec proposes two transformation paths (Path A: extend existing MCP tools with `mobile_format` flag; Path B: new mobile-specific MCP tools) without fully committing to either.

---

## Tensions Found

### Tension 1: The Transformation Pipeline Ownership — Who Decides Content Shape?

**What the spec says:**
Section 3.4 defines two paths for converting ZenSci outputs into mobile formats. Path A extends existing tools (latex-mcp, blog-mcp, etc.) with a `mobile_format` parameter; Path B creates new mobile-specific MCP tools (e.g., `quiz_from_paper`, `flashcard_deck_from_slides`). The spec recommends Path A for v0.1 "to reuse existing tool surface." But the spec never answers the deeper question: **Who decides whether a PDF becomes a reading view, a quiz, or a card deck?**

**The tension:**
The spec treats content transformation as deterministic (latex → PDF → reading_view_json → mobile). But learning outcomes are not deterministic. A paper on photosynthesis could generate:
- A reading view (for deep study)
- A 10-card flashcard deck (for quick memorization)
- A 15-question quiz (for comprehension assessment)
- A narrative lesson (if you want interactive pacing)
- An animated explainer video (if video production is in scope)

The spec glosses over this choice. Is it:
1. **Student choice** (UI dropdown: "View this paper as card deck / quiz / reading")? → Requires runtime format selection; each format must be pre-generated.
2. **Instructor choice** (instructor tags the paper: "view_as: quiz")? → Requires metadata model; conflicts with multi-view expectations.
3. **AI-recommended** (LLM suggests best format based on content semantics)? → Implies new MCP server; adds latency; ties consumption surface to LLM quality.
4. **Hardcoded per module** (blog posts always become digests, papers always become quizzes)? → Simple, but brittle and non-generalizable.

**The "question behind the question":**
How does zen-sci-mobile respect student agency and learning preferences without bloating the app with 5 parallel views of the same content? If Path A extends existing tools, the Gateway must generate *all* formats preemptively, wasting storage. If Path B creates specialized tools, when do they run—on publish (slow, expensive), on demand (slow, bad UX), or on cache miss (unpredictable)?

---

### Tension 2: Tauri v2 Mobile Maturity — Is iOS/Android Ready for Production?

**What the spec says:**
Section 1.5 lists Tauri v2 mobile features (WKWebView on iOS 16+, Chromium WebView on Android 10+, Rust IPC bridge, plugin ecosystem). It acknowledges Tauri v2 is "newer than desktop" and lists platform-specific risks (CSP strictness on iOS, back button handling on Android). But it never quantifies the risk. The word "newer" hides real uncertainties.

**The tension:**
Tauri v2 desktop is production-ready (released Oct 2024, used by Zed, Lapce). But Tauri v2 *mobile* entered beta in mid-2024 and only stabilized in early 2025. For an app targeting iOS App Store and Google Play Store:

1. **Build toolchain complexity** — Must cross-compile to aarch64-apple-ios, x86_64-apple-ios, aarch64-linux-android. Each target has different linker flags, NDK versions, signing certificates. One broken symlink breaks all targets.
2. **WebView differences** — iOS uses WKWebView (WebKit, Safari engine); Android uses Chromium. They diverge on:
   - CSS support (iOS lags: no CSS subgrid, limited `backdrop-filter`)
   - JavaScript APIs (iOS blocks some DOM manipulation; Android permits it)
   - Scroll physics (iOS elastic; Android has momentum)
   - Local storage (different quota limits)
3. **App Store approval** — Apple reviews every iOS submission; may reject if Tauri's IPC pattern looks like app-in-app vulnerability. Google is more permissive but has automated scanning.
4. **Distribution overhead** — Each iOS release requires developer account ($99/year), TestFlight builds, UDID registration, provisioning profiles. Android is easier but still requires signed APK + Google Play account.

**The spec's risk mitigation ("monitor")** is passive. It doesn't ask:
- What's the fallback if Tauri v2 mobile has a critical bug post-launch?
- Are we locked into a specific Tauri version by App Store approval timelines?
- What's the escape hatch if build times exceed 30 minutes (common complaint on Rust mobile)?

**The "question behind the question":**
Should v0.1 ship as iOS + Android simultaneously, or iOS-only first (simpler approval, smaller surface area)? The spec assumes simultaneous; no mention of phased rollout risk.

---

### Tension 3: The "Games" Gap — What Does a Game Built from LaTeX Actually Look Like?

**What the spec says:**
Cruz's original vision (Section 1): mobile is for "games, learning, content, and videos." The spec handles learning (quizzes, card decks), content (reading views), and videos (VideoScript format). **Games are absent.** A quiz is not a game. A card deck is drill, not play.

**The tension:**
Educational games have distinct characteristics:
- **Spatial reasoning** (geometry puzzles, physics simulations)
- **Narrative progression** (branching stories that teach)
- **Competitive/cooperative loops** (leaderboards, multiplayer challenges)
- **Procedural generation** (infinite problem sets, adaptive difficulty)
- **Time pressure** (speed-based challenges that reward quick recall)

None of these map cleanly to "card_deck" or "quiz" formats. A LaTeX paper on topology could become:
- A game: "fold and rotate 3D shapes to match target topology"
- A quiz: "identify which knot invariants match these properties"
- A reading view: the paper as-is

**The spec's silence is deafening.** No ContentType enum value for "game". No schema for game state, scoring, progression, or multiplayer sync. Section 3.4 lists ContentType values (card_deck, quiz, reading, video_script, lesson) but no game.

**The "question behind the question":**
Is "games" a v2 feature that should shape v1 architecture now? Or a post-v0.1 nice-to-have that the spec can safely ignore? If the latter, explicitly say so. If the former, design one game format end-to-end (e.g., a "GrammarDrill" type that generates quiz-like challenges procedurally). The absence of this decision creates ambiguity about whether v1 can ever support games without rearchitecting.

---

### Tension 4: Content Freshness and Permission Model — Who Controls What Syncs to Whom?

**What the spec says:**
Section 3.6 (Offline Architecture) says: "Content syncs from the Gateway when online." SyncEngine fetches a manifest `GET /gateway/mobile/sync?group_id={}&since={}` and downloads items. But the spec never defines **permission boundaries**. Is all group content pushed to all members' devices? Can an instructor mark content as "read-only for mobile" vs. "full access on web"? Can a student selectively ignore certain topics?

**The tension:**
Content discovery and cache management are entangled:

1. **Push vs. Pull** — The spec uses pull (device checks for updates every 30 seconds). But does the student *want* every paper the instructor posts? Or can they filter by topic/difficulty/deadline?
2. **Permissions at cache time** — Who decides *what* is cached locally? Student (opt-in per paper), instructor (auto-push required papers), or system (all group papers)?
3. **Space constraints** — The spec limits cache to 500MB. On a 32GB phone, that's 1.5% of storage. But on a older phone (16GB), that's 3%. If a student is in 5 groups, each pushing 100MB, conflicts arise.
4. **Multi-tenant issues** — If two students share a device (family iPad), they see each other's cached content by default (privacy issue).

**The spec's SyncEngine.fetch_manifest()** returns a flat list of items; no per-item permission check. The ContentCache stores group_id but doesn't enforce access control; it trusts the backend.

**The "question behind the question":**
Should v0.1 include a content **discovery UI** (search, filter by topic/difficulty), or assume instructors pre-select what syncs? The spec's SyncStatus component suggests passive sync ("Syncing..." spinner) but no active user control. This is a UX question that affects architecture (do we need permission filtering in the cache layer?).

---

### Tension 5: SvelteKit on Mobile — Is Mobile-Optimized Web Framework Enough?

**What the spec says:**
Section 3.5 defines SvelteKit routes (/home, /content/[id], /quiz/[id], /downloads, /settings) and components (SwipeableDeck, QuizPlayer, ReadingView). All components use Tailwind CSS and handle touch events via `on:touchstart` / `on:touchend` handlers. The spec includes one detailed example: SwipeableDeck.svelte, which implements card swiping via touch coordinate deltas.

**The tension:**
SvelteKit is designed for **web** (desktop-first, SSR, hyperlinks, form submission). Running it in a **Tauri mobile webview** introduces friction:

1. **Safe area insets** — iOS has notch/Dynamic Island; Android has system nav bars. SvelteKit must respect viewport-fit=cover + env(safe-area-inset-*) CSS. The spec's Tailwind classes don't mention this.
2. **Scroll physics** — iOS users expect elastic overscroll; Android expects momentum. Browser-level physics don't match native expectations. SwipeableDeck's touch event handling is custom, but reading views and quizzes use standard scrolling—is that enough?
3. **Status bar and keyboard** — WKWebView on iOS renders *behind* the status bar by default. Keyboard appearance resizes the webview; SvelteKit must handle height changes. The spec's layout doesn't address bottom-sheet keyboards or IME handling.
4. **Native platform patterns** — iOS uses bottom tabs (tab bar); Android uses bottom nav + hamburger. The spec shows a `NavBar.svelte` component but doesn't specify whether it's iOS-style (translucent, in safe area) or Android-style (opaque, above nav bar).
5. **Accessibility** — Mobile a11y is different from web. iOS VoiceOver expects specific ARIA patterns; Android TalkBack expects different ones. The spec's component code includes some ARIA labels (`aria-label="Card deck"`) but not consistently.

**The "question behind the question":**
Is SvelteKit + Tailwind sufficient for a native-feeling mobile app, or should v0.1 accept "web app in webview" UX and defer native iOS/Android UI libraries to v1? The spec seems to assume the former without acknowledging the latter's cost.

---

### Tension 6: The v1/v2 Boundary — What v1 Decisions Get Unmade for v2?

**What the spec says:**
Section 8.4 (Appendix: v2 Contribution Surface) sketches v2 loosely: "Students can add highlights, notes, quizzes, and card decks for re-export to portal." It mentions annotations stored in the `annotations` table (sync_state: pending/synced) and a v2 sync conflict resolution strategy (CRDT). But v0.1 doesn't implement contributions; it's read-only.

**The tension:**
The offline-first architecture *assumes* contributions will exist in v2, but v1 is not designed for them:

1. **ContentCache is read-only** — The `insert()` method stores downloaded content; the `annotations` table stores highlights + notes. But there's no API for writing *new content* (e.g., "I created a quiz for this paper; push it to the portal"). v2 would need a whole new sync direction (device → cloud).
2. **Conflict resolution is naive** — Section 3.6: "Server always wins in v0.1." This is fine for read-only consumption but breaks for contributions. If a student highlights text offline, then the instructor updates the paper, which version wins? v1's SQL schema (`annotations` with `sync_state`) hints at CRDT, but there's no CRDT implementation.
3. **MobileContent is immutable** — The `data` field is a JSON blob (serde_json::Value). Once cached, it's treated as atomic. v2 contributions (edits to card decks, quiz answers) require a diff/patch model, not atomic replacement.
4. **Tauri IPC is one-directional** — Commands are request-reply (frontend → Rust → frontend). v2 would need push notifications (Rust → frontend) when a student's contribution is approved. The spec includes `subscribe_to_sync_events` but doesn't define what events exist.

**The spec's Appendix 8.4** says v1 "shapes v1 architecture," but the concrete implications are vague. v1 will be shipped as-is for months; making contributions work in v2 will require patching or rearchitecting.

**The "question behind the question":**
Should v0.1 pre-implement a contribution data model (even if disabled in UI) to avoid rework in v2? Or accept that v2 will need schema migrations + data model cleanup? The spec doesn't address migration cost.

---

## The Reframe

**If you could resolve one architectural tension, which would unlock the most value for v0.1 and v2?**

The answer is **Tension 1: Transform pipeline ownership.**

Currently, the spec treats content transformation as a backend concern (MCP tools generate formats; mobile app consumes them). But the *real* decision is **at the learning layer**: What format best serves *this student's learning goal with this content right now*?

**Reframe as a question:**

> **"Should content transformation be a compile-time decision (formats generated once, at publish) or a runtime decision (formats generated on-demand, when student opens content)?"**

This single question cascades through architecture:

- **Compile-time** (Instructor publishes paper → Gateway generates reading + quiz + cards → cached on all devices): Fast UX (instant format switching), high storage cost, poor flexibility (locked-in formats).
- **Runtime** (Student opens paper → requests specific format → LLM/MCP generates it live → caches result): Slow first load (3-5 seconds), lower storage (one format at a time), flexible (adapts to student preference).

**Why this reframe wins:**

1. It **forces a product decision** (UX tradeoff) instead of hiding it in architecture
2. It **clarifies permission boundaries** (If compile-time, permissions checked at publish; if runtime, checked at request)
3. It **explains the v1/v2 boundary** (v1 is compile-time; v2 adds runtime, enabling contribution feedback loops)
4. It **addresses the "games" gap** (Games are runtime-generated from quiz/lesson content; not a separate content type)

---

## Gaps

The spec does not adequately address:

1. **User preference model for content format** — No mention of "preferred learning style" (visual vs. textual vs. kinesthetic), difficulty level, language, or pacing. How does a student signal "show me quizzes, not reading views"? Is this per-content, per-course, or global?

2. **Sync conflict resolution and versioning** — Section 3.6 says "server always wins," but what happens when a student's highlights conflict with a paper update? No version history, no three-way merge, no "accept/reject" flow. v1 can ignore this, but it's a time bomb for v2.

3. **Content discovery and search before download** — The spec includes `search_cache()` for offline search but no API for browsing group content before it's cached. How does a student discover what's available to download? Is there a "browseable feed" that lists available content without downloading it?

4. **Authentication and session refresh strategy** — Section 3.2 mentions "JWT + platform keychain," but when does the token refresh? If a student is offline for a week, opens the app, and tries to sync, is the token expired? No refresh flow specified.

5. **Offline detection and UX fallback** — The spec shows a SyncStatus indicator (offline/syncing/synced) but no behavior when network returns mid-operation. If a student starts a quiz offline, then loses network, what happens to their answers? Are they auto-saved locally?

---

## What to Keep

The spec gets these things right and should survive revision:

1. **Clear separation of concerns (Rust + SvelteKit)** — Rust owns local state, security, platform integration; SvelteKit owns UI. This boundary is clean and testable. Don't blur it by adding backend business logic to SvelteKit.

2. **Offline-first caching model** — rusqlite + LRU eviction is the right choice. It's lightweight, doesn't require server infrastructure, and respects device constraints. The ContentCache abstraction is solid.

3. **SyncEngine polling strategy (30-second intervals, adaptive)** — Polling is simpler than WebSocket/push and works offline. The 30-second default balances battery drain vs. freshness. Good pragmatism.

4. **Tauri v2 mobile for iOS + Android parity** — One codebase, native webview on both platforms. Better than React Native (JS in GC overhead) or Flutter (no TypeScript story). Tauri's choice is defensible.

5. **Detailed schema for mobile learning formats (LearningCard, Quiz, ReadingContent)** — The TypeScript definitions in Section 3.4 are thorough, field-by-field, with proper enums and optionals. This is spec-quality work; don't second-guess it.

---

## Recommended Questions for Cruz

Ask Cruz these three questions before locking the spec:

### Q1: Format Generation — Compile Time vs. Runtime?

> "When a student opens a paper on mobile, should the app show a pre-generated quiz (compiled at publish time), or ask the LLM to generate a quiz on-demand?"

- **If compile-time:** Spec is correct as-is. Path A (extend MCP tools) is optimal.
- **If runtime:** Spec needs a new "on-demand format" request pattern, new Gateway endpoint, and caching for frequently-requested formats.

**Decision impact:** Changes Section 3.4 entirely. Sets scope for Path B (new mobile-specific MCP tools).

---

### Q2: Game Content — In Scope for v0.1 or Defer to v1/v2?

> "Should v0.1 support structured games (e.g., topology puzzle, chemistry simulation), or is 'game' just marketing language for 'interactive quizzes'?"

- **If in scope:** Define at least one game schema (GameActivity? InteractiveExperiment?) and show one end-to-end example (e.g., "convert this physics paper into a pendulum-motion simulator").
- **If deferred:** Explicitly state in Section 1 (Non-Goals): "Game formats deferred to v1.1+."

**Decision impact:** If in scope, adds complexity to transformation pipeline. If deferred, clarifies scope and increases shipping velocity.

---

### Q3: Mobile-First UX or Web-in-Webview?

> "Should v0.1 optimize for native mobile UX (notch-safe layouts, platform-specific gestures, iOS/Android design systems), or is 'Tauri webview' enough and we accept 'web app' feel?"

- **If native-first:** SvelteKit + Tailwind is insufficient. Need ios/android specific styles, viewport handling, platform-aware components.
- **If web-acceptable:** Spec as-is. Accept that the app feels like "responsive web" on mobile.

**Decision impact:** If native-first, add 2-3 weeks to v0.1 timeline and increase platform-specific testing load. If web-acceptable, launch faster but risk lower app store reviews ("doesn't feel native").

---

## Summary: Tensions vs. Spec Quality

The spec is *architecturally sound* but *strategically underspecified*. The Rust backend, SvelteKit UI, and offline-first caching are well-designed. The problem is not execution; it's decision clarity:

- **Tension 1** (format ownership) hides a product decision (student agency vs. pre-computation tradeoff)
- **Tension 2** (Tauri maturity) hides a risk appetite decision (iOS+Android day-1 vs. phased rollout)
- **Tension 3** (games) hides a scope decision (entertainment-first vs. learning-focused)
- **Tension 4** (permissions) hides a UX decision (active sync management vs. passive)
- **Tension 5** (SvelteKit on mobile) hides a quality decision (native feel vs. responsive web)
- **Tension 6** (v1/v2 boundary) hides a tech debt decision (pre-migrate for v2 vs. refactor on demand)

**Resolve these six decisions, and the spec becomes a shipping spec. Ignore them, and they become production incidents in 6 months.**

---

**Document prepared by:** Strategic Scout
**Date:** 2026-02-18
**Status:** Ready for decision & spec refinement
**Approval gate:** Resolve Tensions 1, 2, 3 before committing to implementation
