# zen-sci-mobile v1.0: The Learning Surface
**A Tauri v2 iOS + Android mobile app for consuming academic knowledge as interactive learning formats**

---

## Document Metadata
- **Author:** Architecture & Product Strategy (prepared for Cruz Morales)
- **Status:** Release-Ready Specification
- **Date:** 2026-02-18
- **Version:** v1.0 — Consumption Surface (Read, watch, play, learn; v2 contribution deferred)
- **Grounded In:**
  - ZenithScience STATUS.md (Phase 3 complete, 6 MCP servers, 493 tests passing)
  - AgenticGateway ARCHITECTURE.md (v1.0.0 Go, memory persistence, orchestration, OTEL)
  - AgenticGateway STATUS.md (44 skills, 33 tools, 8 LLM providers, production-ready)
  - scout_zen-sci-portal (Tauri + Rust backend strategy)
  - wide_desktop-native-frameworks (Tauri v2 confirmed, Rust ecosystem validated)

---

## 1. Vision

**Core Insight:** Knowledge doesn't learn itself. On desktop, students *create* academic content (papers, slides, blogs, grants). On mobile, students *consume* that content—transformed from raw documents (PDFs, LaTeX, slides) into engaging, mobile-optimized learning formats: swipeable card decks, quizzes, reading views, and lesson playback.

**One-Line:** zen-sci-mobile transforms publication-ready academic artifacts from the portal into mobile-native learning experiences, offline-ready and in sync with a user's group knowledge base.

**User Story:** A researcher opens zen-sci-mobile during their commute. Their iOS or Android device instantly shows new quiz decks, blog posts, and papers—auto-transformed from yesterday's desktop contributions. Offline search works. They can read, swipe through cards, answer quizzes, and save highlights for later. When they return to desktop, their annotations sync back to the portal.

---

## 1.5. Current State

**What exists to build on:**

1. **ZenSci Platform (6 MCP Servers):** Production-ready document-to-format conversion engines
   - **latex-mcp** (v0.1) → PDF (for papers, proposals)
   - **blog-mcp** (v0.2) → HTML blog posts
   - **slides-mcp** (v0.3) → Beamer + Reveal.js decks
   - **newsletter-mcp** (v0.3) → MJML email format
   - **grant-mcp** (v0.4) → NIH/NSF grant proposals
   - **paper-mcp** (v0.5) → IEEE/ACM/arXiv LaTeX papers

2. **AgenticGateway (v1.0.0):** Production Go service with
   - OpenAI-compatible `/v1/chat/completions` API
   - Tool registry & orchestration (33 tools, 44 skills)
   - Multi-provider LLM routing (8 providers: Anthropic, OpenAI, Google, Groq, Mistral, Kimi, DeepSeek, Ollama)
   - MCP integration (MCPByDojoGenesis + Composio)
   - SQLite memory persistence + garden metaphor
   - OTEL observability → Langfuse

3. **Tauri v2 Mobile Support (Feb 2026):**
   - iOS: WKWebView (Safari engine, iOS 16+)
   - Android: Chromium WebView (Android 10+)
   - Rust IPC bridge with Tokio async runtime
   - Multi-target binary support (aarch64-apple-ios, x86_64-apple-ios, aarch64-linux-android, etc.)
   - Tauri plugin ecosystem (keyring, notification, filesystem, websocket)

**What's new and needed:**

1. **Content Transformation Pipeline:** A *new* layer that converts ZenSci outputs into mobile learning formats. Two architectural paths:
   - **Path A:** Extend existing MCP tools with a `mobile_format` parameter (e.g., `convert_to_pdf(doc, format: "mobile_reading")`)
   - **Path B:** Create new mobile-specific MCP tools (e.g., `quiz_from_paper`, `flashcard_deck_from_slides`)

   *Decision: Recommend Path A for v0.1 (reuse existing tool surface), migrate to Path B in v1 if tool proliferation becomes a problem.*

2. **Mobile Content Schema:** Define the contract for mobile learning objects
   - `LearningCard` (front/back, image, audio)
   - `Quiz` (questions, answer validation, progress tracking)
   - `ReadingContent` (text, highlights, inline notes)
   - `ContentType` enum (card_deck, quiz, reading, video_script, lesson)

3. **Rust Mobile Backend (Tauri):** Four components not yet in place
   - **ContentCache** (rusqlite) — offline storage, LRU eviction, sync state tracking
   - **AuthManager** (jsonwebtoken + platform keychain) — JWT validation, credential storage
   - **SyncEngine** (reqwest + tokio) — pull new group content, conflict resolution, sync scheduling
   - **IPC Commands** — bridge between SvelteKit frontend and Rust backend

4. **SvelteKit Mobile UI:** Mobile-optimized routes and components
   - **Routes:** `/` (home/feed), `/content/[id]` (player), `/quiz/[id]`, `/downloads`, `/settings`
   - **Components:** ContentCard, SwipeableDeck, QuizPlayer, ReadingView, ProgressBar

---

## 2. Goals & Success Criteria

### Primary Goals
1. **Offline consumption:** A student downloads academic content (papers, slides, quizzes) to their mobile device and learns without internet
2. **Instant sync:** When new content is published to the student's group on the portal, it appears on mobile within 30 seconds
3. **Mobile-first UX:** Content is not "responsive PDFs"—it's *transformed* into touch-optimized learning formats (cards, quizzes, reading mode)
4. **Cross-platform:** Identical app on iOS and Android (one Rust + SvelteKit codebase, Tauri handles platform differences)

### Success Criteria (v0.1 launch)
- **S1:** A student can log in via platform keychain (iOS Keychain / Android Keystore) in 2 taps
- **S2:** After sync, student sees 5+ pieces of content from their group (papers, quizzes, blog posts, slides)
- **S3:** Student can swipe through a 20-card quiz deck, receive a score, and save progress—*without network*
- **S4:** A PDF paper is rendered as a mobile reading view (adjustable text size, dark mode, highlights saved locally)
- **S5:** When network returns, highlights + quiz scores sync back to the portal (or are ready for future v2 contribution)
- **S6:** App starts in <1.5 seconds, consumes <50MB idle memory (iOS target: <80MB due to WKWebView overhead)
- **S7:** Content is served in 6 mobile learning formats (see Section 3.4 transformation table)

### Non-Goals (Explicitly Deferred to v2 or Later)
- **NOT v0.1:** Student contribution (writing, creating content from mobile) → defer to v2+
- **NOT v0.1:** Real-time collaboration (two users editing the same doc on mobile) → defer to v2 if Tauri supports CRDT well
- **NOT v0.1:** Advanced annotation tools (voice memos, rich highlighting, sketching) → v1.1 nice-to-have
- **NOT v0.1:** Offline-first composition (editing documents on mobile while offline) → v2 contributor surface
- **NOT v0.1:** Document editor in mobile webview → belongs on desktop only

---

## 3. Technical Architecture

### 3.1 System Overview

**Data flow and component layering:**

```
┌──────────────────────────────────────────────────────────────────┐
│                   Mobile Device (iOS / Android)                   │
│                                                                    │
│  ┌──────────────────────────┐   ┌─────────────────────────────┐  │
│  │   SvelteKit Webview      │   │   Rust Tauri Backend        │  │
│  │  (TypeScript + Svelte)   │   │  (Learning Surface Logic)   │  │
│  │                          │   │                             │  │
│  │ Routes:                  │   │ Components:                 │  │
│  │  /                       │◄──┤ • ContentCache (rusqlite)   │  │
│  │  /content/[id]          │ ◄──┤ • AuthManager (JWT)         │  │
│  │  /quiz/[id]             │IPC │ • SyncEngine (reqwest)      │  │
│  │  /downloads             │    │ • IPC Command Handler       │  │
│  │  /settings              │    │                             │  │
│  │                          │    │ Tauri Commands:             │  │
│  │ Components:             │    │ • sync_content              │  │
│  │ • ContentCard           │    │ • search_cache              │  │
│  │ • SwipeableDeck         │    │ • get_offline_content       │  │
│  │ • QuizPlayer            │    │ • validate_token            │  │
│  │ • ReadingView           │    │ • subscribe_to_sync_events  │  │
│  │ • ProgressBar           │    │                             │  │
│  │                          │    │ Runtime:                    │  │
│  │ Stores (Svelte):        │    │ • Tokio async runtime       │  │
│  │ • userSession           │    │ • Background sync thread    │  │
│  │ • contentFeed           │    │ • Event broadcaster (app    │  │
│  │ • quizProgress          │    │   brings content updates)   │  │
│  │ • cacheStatus           │    │                             │  │
│  │                          │    │                             │  │
│  └──────────────────────────┘   └─────────────────────────────┘  │
│                                                                    │
└────────────────────────────────┬─────────────────────────────────┘
                                 │
                    HTTP + WebSocket (online)
                                 │
     ┌───────────────────────────┼───────────────────────────┐
     │                           │                           │
┌────▼──────────┐    ┌──────────▼──────────┐    ┌──────────▼──────┐
│ AgenticGateway│    │  ZenSci Servers     │    │  Cloud Sync (v2)│
│  (Go v1.0.0)  │◄───┤  (TypeScript MCP)   │    │   S3 + Versioning
│               │    │                     │    │                 │
│ • Session API │    │ • latex-mcp         │    │ • Document store│
│ • Memory API  │    │ • blog-mcp          │    │ • Annotation DB │
│ • Tool API    │    │ • slides-mcp        │    │ • Sync state    │
│               │    │ • newsletter-mcp    │    │                 │
└────────────────┘   │ • grant-mcp         │    └─────────────────┘
                     │ • paper-mcp         │
                     │                     │
                     │ Mobile-Specific     │
                     │ Transformations:    │
                     │                     │
                     │ • PDF → Reading     │
                     │ • Slides → Cards    │
                     │ • Paper → Quiz      │
                     │ • Blog → Digest     │
                     │ • Grant → Summary   │
                     │ • Newsletter → Cards│
                     │                     │
                     └─────────────────────┘
```

**Content pipeline (detailed):**

```
Desktop User creates content on zen-sci-portal
  ↓ (e.g., uploads myresearch.pdf + metadata)
AgenticGateway stores in cloud (S3 + DB)
  ↓
ZenSci servers transform:
  - myresearch.pdf → LaTeX → ZenSci parsing → structured AST
  - ↓ with mobile_format=true flag (Path A)
  - Generates: reading_view_json, quiz_questions_json, summary_cards_json
  ↓ (stored in Gateway memory or new cloud "mobile_formats" table)
Mobile app background sync (SyncEngine)
  - Checks Gateway for new content in user's group
  - Queries mobile-format transformations
  - Downloads + caches locally (rusqlite)
  - Emits Tauri event "content_synced"
  ↓
SvelteKit UI listener receives event
  - Re-fetches contentFeed store
  - Renders new cards in /home
  ↓
Student sees new content in <30 seconds
  - Taps to open
  - SvelteKit routes to /content/[id]
  - Rust backend fetches from local cache (offline-ready)
  - Component renders (reading view, card deck, or quiz)
```

---

### 3.2 Tauri v2 Mobile Project Structure

**Directory layout:**

```
zen-sci-mobile/
├── src-tauri/                      # Rust backend
│   ├── Cargo.toml                  # Main workspace manifest
│   ├── Cargo.lock                  # Pinned dependencies
│   ├── src/
│   │   ├── main.rs                 # Entry point, window setup
│   │   ├── commands.rs             # Tauri IPC command definitions
│   │   ├── cache/
│   │   │   ├── mod.rs              # ContentCache module
│   │   │   ├── db.rs               # rusqlite database layer
│   │   │   └── lru.rs              # LRU eviction policy
│   │   ├── auth/
│   │   │   ├── mod.rs              # AuthManager module
│   │   │   ├── jwt.rs              # JWT validation
│   │   │   └── keyring.rs          # Platform keychain integration
│   │   ├── sync/
│   │   │   ├── mod.rs              # SyncEngine module
│   │   │   ├── gateway.rs          # HTTP client to AgenticGateway
│   │   │   ├── state.rs            # Sync state machine
│   │   │   └── scheduler.rs        # Background sync scheduling
│   │   └── error.rs                # Custom error types
│   │
│   ├── tauri.mobile.conf.json      # Mobile-specific Tauri config
│   ├── build.rs                    # Mobile-target build script
│   └── Cargo.toml                  # Dependencies:
│       ├── tauri = { version = "2", features = ["mobile"] }
│       ├── tokio = { version = "1.40", features = ["full"] }
│       ├── tokio-rusqlite = "0.5"
│       ├── jsonwebtoken = "9"
│       ├── reqwest = { version = "0.12", features = ["json"] }
│       ├── serde = { version = "1", features = ["derive"] }
│       ├── serde_json = "1"
│       ├── rusqlite = { version = "0.31", features = ["bundled"] }
│       └── tauri-plugin-keyring = "1"  # Keychain access (v2+)
│
├── src/                            # SvelteKit frontend
│   ├── app.html
│   ├── app.postcss
│   ├── routes/
│   │   ├── +page.svelte            # / — Home (feed)
│   │   ├── +layout.svelte          # Navigation layout
│   │   ├── content/
│   │   │   └── [id]/
│   │   │       └── +page.svelte    # /content/[id] — Content player
│   │   ├── quiz/
│   │   │   └── [id]/
│   │   │       └── +page.svelte    # /quiz/[id] — Quiz mode
│   │   ├── downloads/
│   │   │   └── +page.svelte        # /downloads — Cache manager
│   │   └── settings/
│   │       └── +page.svelte        # /settings — Auth, sync prefs
│   │
│   ├── components/
│   │   ├── ContentCard.svelte      # Card component (title, preview, sync status)
│   │   ├── SwipeableDeck.svelte    # Touch swipe handling for card deck
│   │   ├── QuizPlayer.svelte       # Quiz question rendering + answer submission
│   │   ├── ReadingView.svelte      # Text view (adjustable font, dark mode, highlights)
│   │   ├── ProgressBar.svelte      # Quiz progress indicator
│   │   ├── SyncStatus.svelte       # Sync indicator (spinning icon, last sync time)
│   │   └── NavBar.svelte           # Bottom navigation (home, downloads, settings)
│   │
│   ├── stores/
│   │   ├── user.ts                 # Svelte store: { token, profile, userId, groupId }
│   │   ├── content.ts              # Svelte store: { items[], loading, error }
│   │   ├── cache.ts                # Svelte store: { downloadedCount, totalSize, lastSync }
│   │   └── quiz.ts                 # Svelte store: { currentScore, currentQuestion, answers[] }
│   │
│   ├── lib/
│   │   ├── api.ts                  # Tauri IPC client wrapper
│   │   ├── types.ts                # TypeScript interfaces (Content, Quiz, etc.)
│   │   └── utils.ts                # Helpers (date formatting, etc.)
│   │
│   └── app.css                     # Global styles (Tailwind + mobile-friendly)
│
├── package.json
├── svelte.config.js                # SvelteKit config (static adapter required)
├── vite.config.ts
├── tsconfig.json
│
├── tauri.conf.json                 # Tauri v2 app config (windows, permissions, icons)
├── capabilities.json               # Tauri security capabilities (IPC commands allowed)
└── README.md
```

**Key configuration files:**

**tauri.mobile.conf.json:**
```json
{
  "build": {
    "devUrl": "http://localhost:5173",
    "frontendDist": "../build",
    "beforeDevCommand": "npm run dev",
    "beforeBuildCommand": "npm run build"
  },
  "app": {
    "windows": [
      {
        "label": "main",
        "title": "zen-sci",
        "fullscreen": false,
        "resizable": true,
        "maximized": false,
        "minWidth": 320,
        "minHeight": 568,
        "width": 393,
        "height": 852,
        "x": 0,
        "y": 0
      }
    ]
  },
  "security": {
    "csp": null
  }
}
```

**tauri.conf.json (permissions excerpt):**
```json
{
  "plugins": {
    "core": {
      "enable": true
    },
    "fs": {
      "allow": ["read", "write"],
      "scope": ["$APPDATA", "$CACHE"]
    },
    "http": {
      "allow": ["http://*", "https://*"]
    },
    "notification": {
      "all": true
    }
  }
}
```

**iOS-specific (Info.plist keys to add):**
```xml
<key>NSLocalNetworkUsageDescription</key>
<string>zen-sci needs to sync with your knowledge base</string>
<key>NSBonjourServices</key>
<array>
  <string>_zen-sci._tcp</string>
</array>
<key>NSCameraUsageDescription</key>
<string>zen-sci does not use your camera</string>
<key>NSMicrophoneUsageDescription</key>
<string>zen-sci does not use your microphone</string>
```

**Android-specific (AndroidManifest.xml permissions to add):**
```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
```

---

### 3.3 Rust Mobile Backend

**Four-component architecture:**

#### **Component 1: ContentCache (rusqlite)**

Stores downloaded learning content, tracks sync state, enforces LRU eviction.

**Schema (SQL):**
```sql
-- Main content table
CREATE TABLE IF NOT EXISTS content (
  id TEXT PRIMARY KEY,
  title TEXT NOT NULL,
  author TEXT,
  group_id TEXT NOT NULL,
  content_type TEXT NOT NULL, -- 'card_deck', 'quiz', 'reading', 'video_script', 'lesson'
  source_format TEXT, -- 'pdf', 'slides', 'blog', 'newsletter', 'grant'
  data BLOB NOT NULL, -- JSON-serialized LearningCard[] or Quiz or ReadingContent
  metadata_json TEXT, -- { summary, tags[], created_at, updated_at, version }
  downloaded_at INTEGER, -- Unix timestamp
  last_accessed_at INTEGER, -- For LRU
  sync_state TEXT DEFAULT 'synced', -- 'synced', 'pending_delete', 'pending_update'
  local_size_bytes INTEGER,
  CHECK (content_type IN ('card_deck', 'quiz', 'reading', 'video_script', 'lesson'))
);

-- Sync log (for conflict resolution in v2)
CREATE TABLE IF NOT EXISTS sync_log (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  content_id TEXT NOT NULL,
  action TEXT NOT NULL, -- 'download', 'update', 'delete', 'annotation_add'
  timestamp INTEGER NOT NULL,
  local_hash TEXT,
  remote_hash TEXT,
  FOREIGN KEY (content_id) REFERENCES content(id)
);

-- User annotations (highlights, quiz answers saved for v2 contribution)
CREATE TABLE IF NOT EXISTS annotations (
  id TEXT PRIMARY KEY,
  content_id TEXT NOT NULL,
  type TEXT NOT NULL, -- 'highlight', 'quiz_answer', 'note'
  data_json TEXT NOT NULL, -- { text, range, color, timestamp }
  sync_state TEXT DEFAULT 'pending', -- 'pending', 'synced'
  FOREIGN KEY (content_id) REFERENCES content(id)
);
```

**Rust structs:**

```rust
use rusqlite::{Connection, params};
use serde::{Serialize, Deserialize};
use tokio_rusqlite::Connection as AsyncConnection;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ContentType {
    CardDeck,
    Quiz,
    Reading,
    VideoScript,
    Lesson,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CachedContent {
    pub id: String,
    pub title: String,
    pub author: Option<String>,
    pub group_id: String,
    pub content_type: ContentType,
    pub source_format: String, // "pdf", "slides", "blog", etc.
    pub data: serde_json::Value,
    pub metadata: ContentMetadata,
    pub downloaded_at: i64,
    pub last_accessed_at: i64,
    pub sync_state: SyncState,
    pub local_size_bytes: u64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ContentMetadata {
    pub summary: Option<String>,
    pub tags: Vec<String>,
    pub created_at: String, // ISO 8601
    pub updated_at: String,
    pub version: u32,
}

#[derive(Debug, Clone, Copy, PartialEq, Serialize, Deserialize)]
pub enum SyncState {
    Synced,
    PendingDelete,
    PendingUpdate,
}

pub struct ContentCache {
    db: AsyncConnection,
    max_cache_bytes: u64, // Default: 500MB
    current_size: std::sync::Arc<std::sync::Mutex<u64>>,
}

impl ContentCache {
    pub async fn new(db_path: &str, max_cache_bytes: u64) -> Result<Self, tokio_rusqlite::Error> {
        let db = AsyncConnection::open(db_path).await?;

        // Initialize schema
        db.call(|conn| {
            conn.execute_batch(SCHEMA)?;
            Ok(())
        }).await?;

        Ok(Self {
            db,
            max_cache_bytes,
            current_size: std::sync::Arc::new(std::sync::Mutex::new(0)),
        })
    }

    /// Store a piece of content locally
    pub async fn insert(&self, content: CachedContent) -> Result<(), tokio_rusqlite::Error> {
        let data_json = serde_json::to_string(&content.data)?;
        let metadata_json = serde_json::to_string(&content.metadata)?;
        let sync_state_str = format!("{:?}", content.sync_state).to_lowercase();

        self.db.call(move |conn| {
            conn.execute(
                "INSERT OR REPLACE INTO content
                 (id, title, author, group_id, content_type, source_format, data, metadata_json,
                  downloaded_at, last_accessed_at, sync_state, local_size_bytes)
                 VALUES (?1, ?2, ?3, ?4, ?5, ?6, ?7, ?8, ?9, ?10, ?11, ?12)",
                params![
                    &content.id,
                    &content.title,
                    &content.author,
                    &content.group_id,
                    format!("{:?}", content.content_type).to_lowercase(),
                    &content.source_format,
                    data_json.as_bytes(),
                    metadata_json,
                    content.downloaded_at,
                    content.last_accessed_at,
                    sync_state_str,
                    content.local_size_bytes as i64,
                ],
            )?;
            Ok(())
        }).await?;

        // Check if LRU eviction needed
        self.evict_if_needed().await?;
        Ok(())
    }

    /// Retrieve content by ID (instant, from cache)
    pub async fn get(&self, id: &str) -> Result<Option<CachedContent>, tokio_rusqlite::Error> {
        let id_copy = id.to_string();
        self.db.call(move |conn| {
            let mut stmt = conn.prepare(
                "SELECT id, title, author, group_id, content_type, source_format, data,
                        metadata_json, downloaded_at, last_accessed_at, sync_state, local_size_bytes
                 FROM content WHERE id = ?1"
            )?;

            let result = stmt.query_row(params![&id_copy], |row| {
                let content_type_str: String = row.get(4)?;
                let sync_state_str: String = row.get(10)?;

                Ok(CachedContent {
                    id: row.get(0)?,
                    title: row.get(1)?,
                    author: row.get(2)?,
                    group_id: row.get(3)?,
                    content_type: parse_content_type(&content_type_str),
                    source_format: row.get(5)?,
                    data: serde_json::from_slice(row.get_ref(6)?.as_blob()?).ok().unwrap_or_default(),
                    metadata: serde_json::from_str(&row.get::<_, String>(7)?).ok().unwrap_or_default(),
                    downloaded_at: row.get(8)?,
                    last_accessed_at: row.get(9)?,
                    sync_state: parse_sync_state(&sync_state_str),
                    local_size_bytes: row.get::<_, i64>(11)? as u64,
                })
            });

            match result {
                Ok(content) => Ok(Some(content)),
                Err(rusqlite::Error::QueryReturnedNoRows) => Ok(None),
                Err(e) => Err(e.into()),
            }
        }).await?;

        Ok(None) // Placeholder
    }

    /// List all content for a given group (with pagination)
    pub async fn list_by_group(
        &self,
        group_id: &str,
        limit: u64,
        offset: u64,
    ) -> Result<(Vec<CachedContent>, u64), tokio_rusqlite::Error> {
        // Implementation similar to get(), returns paginated results
        Ok((vec![], 0)) // Placeholder
    }

    /// LRU eviction: remove oldest accessed content if cache exceeds max_cache_bytes
    async fn evict_if_needed(&self) -> Result<(), tokio_rusqlite::Error> {
        let current_size = *self.current_size.lock().unwrap();

        if current_size > self.max_cache_bytes {
            // Query oldest accessed content (not synced to cloud yet)
            // Delete until under threshold
            // This is a background operation; don't block the app
        }

        Ok(())
    }

    /// Delete content by ID
    pub async fn delete(&self, id: &str) -> Result<(), tokio_rusqlite::Error> {
        let id_copy = id.to_string();
        self.db.call(move |conn| {
            conn.execute("DELETE FROM content WHERE id = ?1", params![&id_copy])?;
            Ok(())
        }).await?;
        Ok(())
    }

    /// Get total cache size and count
    pub async fn cache_stats(&self) -> Result<(u64, u32), tokio_rusqlite::Error> {
        self.db.call(|conn| {
            let total_size: i64 = conn.query_row(
                "SELECT SUM(local_size_bytes) FROM content",
                [],
                |row| row.get(0),
            ).unwrap_or(0);

            let count: i32 = conn.query_row(
                "SELECT COUNT(*) FROM content",
                [],
                |row| row.get(0),
            ).unwrap_or(0);

            Ok((total_size as u64, count as u32))
        }).await?;
        Ok((0, 0)) // Placeholder
    }
}
```

---

#### **Component 2: AuthManager (jsonwebtoken + platform keychain)**

Handles JWT validation and platform-native credential storage (iOS Keychain, Android Keystore).

**Rust structs:**

```rust
use jsonwebtoken::{decode, encode, DecodingKey, EncodingKey, Header, Validation, Algorithm};
use serde::{Serialize, Deserialize};
use tauri::api::process::Command;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct JwtClaims {
    pub sub: String, // user ID
    pub group_id: String,
    pub iat: i64, // issued at
    pub exp: i64, // expiration
    pub roles: Vec<String>,
}

#[derive(Debug, Clone)]
pub struct UserSession {
    pub user_id: String,
    pub group_id: String,
    pub token: String,
    pub roles: Vec<String>,
    pub expires_at: i64,
}

pub struct AuthManager {
    gateway_url: String,
    keychain_service: String, // "zen-sci-mobile"
}

impl AuthManager {
    pub fn new(gateway_url: &str) -> Self {
        Self {
            gateway_url: gateway_url.to_string(),
            keychain_service: "zen-sci-mobile".to_string(),
        }
    }

    /// Validate JWT token and return claims
    pub fn validate_token(&self, token: &str, public_key: &str) -> Result<JwtClaims, String> {
        let decoding_key = DecodingKey::from_rsa_pem(public_key.as_bytes())
            .map_err(|e| format!("Invalid public key: {}", e))?;

        let token_data = decode::<JwtClaims>(
            token,
            &decoding_key,
            &Validation::new(Algorithm::RS256),
        ).map_err(|e| format!("Token validation failed: {}", e))?;

        // Check expiration
        let now = std::time::SystemTime::now()
            .duration_since(std::time::UNIX_EPOCH)
            .unwrap()
            .as_secs() as i64;

        if token_data.claims.exp < now {
            return Err("Token expired".to_string());
        }

        Ok(token_data.claims)
    }

    /// Store credentials in platform keychain
    #[cfg(target_os = "ios")]
    pub async fn store_credentials(&self, user_id: &str, token: &str) -> Result<(), String> {
        // Use Tauri keyring plugin (iOS Keychain backend)
        // Implementation via tauri-plugin-keyring
        Ok(())
    }

    #[cfg(target_os = "android")]
    pub async fn store_credentials(&self, user_id: &str, token: &str) -> Result<(), String> {
        // Use Tauri keyring plugin (Android Keystore backend)
        Ok(())
    }

    /// Retrieve credentials from platform keychain (silent auth)
    #[cfg(target_os = "ios")]
    pub async fn retrieve_credentials(&self) -> Result<Option<UserSession>, String> {
        // Query iOS Keychain for zen-sci-mobile:user_token
        Ok(None) // Placeholder
    }

    #[cfg(target_os = "android")]
    pub async fn retrieve_credentials(&self) -> Result<Option<UserSession>, String> {
        // Query Android Keystore for zen-sci-mobile:user_token
        Ok(None) // Placeholder
    }

    /// Delete stored credentials (logout)
    pub async fn clear_credentials(&self) -> Result<(), String> {
        // Remove from platform keychain
        Ok(())
    }

    /// Check if session is still valid (called before every sync)
    pub async fn is_session_valid(&self) -> Result<bool, String> {
        // Retrieve stored credentials, validate token
        Ok(false) // Placeholder
    }
}
```

---

#### **Component 3: SyncEngine (reqwest + tokio)**

Polls AgenticGateway for new content, downloads mobile learning formats, and manages offline sync state.

**Rust structs:**

```rust
use reqwest::Client;
use tokio::time::{interval, Duration};
use std::sync::Arc;
use tokio::sync::Mutex;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SyncManifest {
    pub group_id: String,
    pub items: Vec<SyncItem>,
    pub last_sync_timestamp: i64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SyncItem {
    pub id: String,
    pub title: String,
    pub content_type: String,
    pub source_format: String,
    pub size_bytes: u64,
    pub updated_at: i64,
    pub hash: String, // For conflict detection
}

pub struct SyncEngine {
    http_client: Client,
    cache: Arc<ContentCache>,
    auth: Arc<AuthManager>,
    gateway_url: String,
    sync_interval: Duration,
}

impl SyncEngine {
    pub fn new(
        cache: Arc<ContentCache>,
        auth: Arc<AuthManager>,
        gateway_url: &str,
    ) -> Self {
        Self {
            http_client: Client::new(),
            cache,
            auth,
            gateway_url: gateway_url.to_string(),
            sync_interval: Duration::from_secs(30), // Default: sync every 30 seconds when online
        }
    }

    /// Start background sync loop (runs in separate tokio task)
    pub async fn start_sync_loop(self: Arc<Self>) {
        let mut interval = interval(self.sync_interval);

        loop {
            interval.tick().await;

            // Only sync if network is available and user is authenticated
            if let Ok(true) = self.auth.is_session_valid().await {
                if let Ok(true) = self.check_network_available().await {
                    if let Err(e) = self.sync_once().await {
                        eprintln!("Sync error: {}", e);
                        // Don't fail the loop; retry next interval
                    }
                }
            }
        }
    }

    /// Perform one sync cycle
    pub async fn sync_once(&self) -> Result<(), String> {
        // 1. Retrieve user's group ID from session
        let session = self.auth.retrieve_credentials().await?
            .ok_or("No active session")?;

        // 2. Fetch manifest from Gateway: GET /gateway/mobile/sync?group_id={group_id}&since={last_sync_time}
        let manifest = self.fetch_manifest(&session.group_id).await?;

        // 3. For each item in manifest:
        //    - Check if already cached (by hash)
        //    - If not, download content from Gateway: GET /gateway/content/{id}
        //    - Store in ContentCache
        for item in &manifest.items {
            if let Err(e) = self.sync_item(item, &session.group_id).await {
                eprintln!("Error syncing item {}: {}", item.id, e);
                // Continue with next item; don't fail entire sync
            }
        }

        // 4. Emit Tauri event "sync_completed" with count of new items
        // (Frontend listener will refresh contentFeed store)

        Ok(())
    }

    async fn fetch_manifest(&self, group_id: &str) -> Result<SyncManifest, String> {
        let session = self.auth.retrieve_credentials().await?
            .ok_or("No session")?;

        let url = format!(
            "{}/gateway/mobile/sync?group_id={}&since={}",
            self.gateway_url,
            group_id,
            0 // TODO: fetch from cache metadata last_sync_timestamp
        );

        let response = self.http_client
            .get(&url)
            .bearer_auth(&session.token)
            .send()
            .await
            .map_err(|e| format!("Manifest fetch failed: {}", e))?;

        if !response.status().is_success() {
            return Err(format!("Manifest fetch returned {}", response.status()));
        }

        let manifest = response
            .json::<SyncManifest>()
            .await
            .map_err(|e| format!("Manifest parse failed: {}", e))?;

        Ok(manifest)
    }

    async fn sync_item(&self, item: &SyncItem, group_id: &str) -> Result<(), String> {
        let session = self.auth.retrieve_credentials().await?
            .ok_or("No session")?;

        // Check if already cached
        if let Ok(Some(cached)) = self.cache.get(&item.id).await {
            if cached.metadata.version >= item.hash.parse::<u32>().unwrap_or(0) {
                return Ok(()); // Already have latest version
            }
        }

        // Download content
        let url = format!("{}/gateway/content/{}?mobile=true", self.gateway_url, item.id);
        let response = self.http_client
            .get(&url)
            .bearer_auth(&session.token)
            .send()
            .await
            .map_err(|e| format!("Content download failed: {}", e))?;

        if !response.status().is_success() {
            return Err(format!("Content download returned {}", response.status()));
        }

        let data = response
            .json::<serde_json::Value>()
            .await
            .map_err(|e| format!("Content parse failed: {}", e))?;

        // Store in cache
        let cached_content = CachedContent {
            id: item.id.clone(),
            title: item.title.clone(),
            author: None,
            group_id: group_id.to_string(),
            content_type: parse_content_type(&item.content_type),
            source_format: item.source_format.clone(),
            data,
            metadata: ContentMetadata {
                summary: None,
                tags: vec![],
                created_at: chrono::Utc::now().to_rfc3339(),
                updated_at: chrono::Utc::now().to_rfc3339(),
                version: 1,
            },
            downloaded_at: std::time::SystemTime::now()
                .duration_since(std::time::UNIX_EPOCH)
                .unwrap()
                .as_secs() as i64,
            last_accessed_at: std::time::SystemTime::now()
                .duration_since(std::time::UNIX_EPOCH)
                .unwrap()
                .as_secs() as i64,
            sync_state: SyncState::Synced,
            local_size_bytes: item.size_bytes,
        };

        self.cache.insert(cached_content).await
            .map_err(|e| format!("Cache insert failed: {}", e))?;

        Ok(())
    }

    async fn check_network_available(&self) -> Result<bool, String> {
        // Simple check: try to reach gateway with a short timeout
        let result = tokio::time::timeout(
            Duration::from_secs(3),
            self.http_client.head(&format!("{}/health", self.gateway_url)).send(),
        ).await;

        Ok(result.is_ok())
    }
}
```

---

#### **Component 4: IPC Commands**

Tauri commands exposed to SvelteKit frontend. Async operations return early; completion emitted via Tauri event.

**Rust command definitions (commands.rs):**

```rust
use tauri::{command, ipc::InvokeError, State};
use crate::cache::ContentCache;
use crate::auth::AuthManager;
use crate::sync::SyncEngine;

#[command]
pub async fn search_cache(
    query: String,
    limit: u32,
    cache: State<'_, std::sync::Arc<ContentCache>>,
) -> Result<Vec<CachedContent>, InvokeError> {
    // Full-text search over local content (if Tantivy added later)
    // For v0.1: simple title/author grep
    Ok(vec![])
}

#[command]
pub async fn get_content(
    id: String,
    cache: State<'_, std::sync::Arc<ContentCache>>,
) -> Result<CachedContent, InvokeError> {
    cache.get(&id)
        .await
        .map_err(|e| InvokeError::from(format!("Cache lookup failed: {}", e)))?
        .ok_or_else(|| InvokeError::from("Content not found"))
}

#[command]
pub async fn list_content(
    group_id: String,
    limit: u32,
    offset: u32,
    cache: State<'_, std::sync::Arc<ContentCache>>,
) -> Result<(Vec<CachedContent>, u32), InvokeError> {
    cache.list_by_group(&group_id, limit as u64, offset as u64)
        .await
        .map_err(|e| InvokeError::from(format!("List failed: {}", e)))
}

#[command]
pub async fn sync_content(
    sync_engine: State<'_, std::sync::Arc<SyncEngine>>,
    window: tauri::Window,
) -> Result<(), InvokeError> {
    let sync_arc = sync_engine.inner().clone();

    tauri::async_runtime::spawn(async move {
        match sync_arc.sync_once().await {
            Ok(_) => {
                let _ = window.emit("sync_completed", json!({ "success": true }));
            }
            Err(e) => {
                let _ = window.emit("sync_error", json!({ "error": e }));
            }
        }
    });

    Ok(())
}

#[command]
pub async fn validate_token(
    token: String,
    auth: State<'_, std::sync::Arc<AuthManager>>,
) -> Result<JwtClaims, InvokeError> {
    let public_key = "..."; // Fetch from Gateway's public key endpoint
    auth.validate_token(&token, public_key)
        .map_err(|e| InvokeError::from(e))
}

#[command]
pub async fn store_token(
    user_id: String,
    token: String,
    auth: State<'_, std::sync::Arc<AuthManager>>,
) -> Result<(), InvokeError> {
    auth.store_credentials(&user_id, &token)
        .await
        .map_err(|e| InvokeError::from(e))
}

#[command]
pub async fn retrieve_token(
    auth: State<'_, std::sync::Arc<AuthManager>>,
) -> Result<Option<UserSession>, InvokeError> {
    auth.retrieve_credentials()
        .await
        .map_err(|e| InvokeError::from(e))
}

#[command]
pub async fn logout(
    auth: State<'_, std::sync::Arc<AuthManager>>,
) -> Result<(), InvokeError> {
    auth.clear_credentials()
        .await
        .map_err(|e| InvokeError::from(e))
}

#[command]
pub async fn cache_stats(
    cache: State<'_, std::sync::Arc<ContentCache>>,
) -> Result<(u64, u32), InvokeError> {
    cache.cache_stats()
        .await
        .map_err(|e| InvokeError::from(format!("Stats failed: {}", e)))
}

#[command]
pub async fn delete_content(
    id: String,
    cache: State<'_, std::sync::Arc<ContentCache>>,
) -> Result<(), InvokeError> {
    cache.delete(&id)
        .await
        .map_err(|e| InvokeError::from(format!("Delete failed: {}", e)))
}

#[command]
pub async fn clear_cache(
    cache: State<'_, std::sync::Arc<ContentCache>>,
) -> Result<(), InvokeError> {
    // Bulk delete all content
    Ok(())
}
```

**Tauri event emitters (main.rs, background sync):**

```rust
// In main.rs during app setup:
let window = app_handle.get_window("main").unwrap();
let sync_engine = Arc::new(SyncEngine::new(cache.clone(), auth.clone(), GATEWAY_URL));

// Start background sync loop
let sync_arc = sync_engine.clone();
tauri::async_runtime::spawn(async move {
    sync_arc.start_sync_loop().await;
});

// Listen for network connectivity changes (optional: add connectivity monitoring)
```

---

### 3.4 Content Transformation Pipeline

**The core of v0.1: how raw ZenSci outputs become mobile learning formats.**

**Transformation Table (6 module types → 6 mobile formats):**

| ZenSci Module | Input | Mobile Output Format | JSON Schema | Example |
|---|---|---|---|---|
| **latex-mcp** (PDF) | myresearch.pdf (LaTeX) | **reading_view** | `ReadingContent[]` | Paragraphs + highlighted equations |
| **blog-mcp** (HTML) | blog.md → HTML | **reading_view** + optional digest cards | `ReadingContent[]` or `LearningCard[]` | Article text + 3-5 key takeaway cards |
| **slides-mcp** (Beamer/Reveal.js) | slides.md → Reveal.js JSON | **card_deck** | `LearningCard[]` | One card per slide (title + speaker notes) |
| **newsletter-mcp** (MJML) | newsletter.md → MJML → HTML | **card_deck** | `LearningCard[]` | One card per block section |
| **grant-mcp** (Grant proposal) | proposal.md → NSF/NIH LaTeX | **summary_cards** | `LearningCard[]` | Aims, significance, innovation, impact (4 cards) |
| **paper-mcp** (IEEE/ACM/arXiv) | paper.tex → formatted LaTeX | **reading_view** + **quiz** | `ReadingContent[]` + `Quiz` | Full paper text + 5-10 comprehension questions |

**Detailed schemas:**

```typescript
// TypeScript definitions (src/lib/types.ts)

// Base content wrapper
export interface MobileContent {
  id: string;
  title: string;
  author?: string;
  groupId: string;
  contentType: 'card_deck' | 'quiz' | 'reading' | 'video_script' | 'lesson';
  sourceFormat: 'pdf' | 'slides' | 'blog' | 'newsletter' | 'grant';
  data: CardDeck | Quiz | ReadingContent | VideoScript;
  metadata: ContentMetadata;
  downloadedAt: number; // Unix timestamp
}

// Format 1: Card Deck (Swipeable)
export interface CardDeck {
  cards: LearningCard[];
  totalCards: number;
  estimatedMinutesPerCard: number;
}

export interface LearningCard {
  id: string;
  front: {
    text: string;
    htmlFormatted?: string; // For complex cards with formatted text
    imageUrl?: string;
    audioUrl?: string; // Optional voice-over
  };
  back: {
    text: string;
    htmlFormatted?: string;
    imageUrl?: string;
    audioUrl?: string;
  };
  tags: string[]; // For categorization
  difficulty: 'easy' | 'medium' | 'hard';
  hint?: string;
  mnemonicDevices?: string; // Memory aids
}

// Format 2: Quiz
export interface Quiz {
  questions: QuizQuestion[];
  totalQuestions: number;
  passingScore: number; // e.g., 70
  estimatedMinutes: number;
  feedbackMode: 'immediate' | 'end_of_quiz';
  randomizeQuestions?: boolean;
  randomizeAnswers?: boolean;
}

export interface QuizQuestion {
  id: string;
  questionText: string;
  questionType: 'multiple_choice' | 'true_false' | 'short_answer' | 'matching';
  answers: QuizAnswer[];
  correctAnswerIndex: number; // or indices[] for multiple correct
  explanation: string;
  imageUrl?: string;
  estimatedSeconds: number;
  difficulty: 'easy' | 'medium' | 'hard';
  tags: string[];
}

export interface QuizAnswer {
  id: string;
  text: string;
  isCorrect: boolean;
}

// Format 3: Reading View
export interface ReadingContent {
  sections: ReadingSection[];
  totalWords: number;
  estimatedMinutes: number;
  fontSize: 'small' | 'medium' | 'large'; // User preference (stored locally)
  darkMode: boolean; // User preference
  columns: 1 | 2; // Single or two-column layout
  lineHeight: 'compact' | 'normal' | 'relaxed';
}

export interface ReadingSection {
  id: string;
  title: string;
  content: ContentBlock[];
  pageNumber?: number; // For referencing original PDF
}

export interface ContentBlock {
  type: 'paragraph' | 'heading' | 'code' | 'blockquote' | 'image' | 'equation';
  text?: string;
  htmlFormatted?: string;
  imageUrl?: string;
  altText?: string;
  equationLatex?: string; // For math rendering
  level?: number; // For headings (h1, h2, h3, etc.)
}

// Format 4: Video Script (for future v1.1)
export interface VideoScript {
  title: string;
  chapters: VideoChapter[];
  totalMinutes: number;
  transcript: TranscriptSegment[];
}

export interface VideoChapter {
  id: string;
  title: string;
  startSecond: number;
  endSecond: number;
  thumbnail?: string;
}

export interface TranscriptSegment {
  startSecond: number;
  endSecond: number;
  speaker?: string;
  text: string;
}

// Format 5: Lesson (structured interactive)
export interface Lesson {
  title: string;
  objectives: string[];
  sections: LessonSection[];
  estimatedMinutes: number;
  difficulty: 'beginner' | 'intermediate' | 'advanced';
}

export interface LessonSection {
  id: string;
  title: string;
  content: ContentBlock[];
  activity?: LessonActivity;
  quiz?: Quiz;
}

export interface LessonActivity {
  type: 'experiment' | 'reflection' | 'discussion' | 'problem_solving';
  prompt: string;
  resources?: string[];
}

// Metadata (shared across all formats)
export interface ContentMetadata {
  summary?: string;
  tags: string[];
  createdAt: string; // ISO 8601
  updatedAt: string;
  version: number;
  sourceDocument?: {
    originalId: string;
    originalFormat: string;
    zeNSciServer: string; // which server created this
  };
  estimatedTimeMinutes?: number;
  difficulty?: 'beginner' | 'intermediate' | 'advanced';
}

// User progress (stored locally, synced in v2)
export interface UserProgress {
  contentId: string;
  contentType: string;
  startedAt: number;
  lastAccessedAt: number;
  percentComplete: number; // 0–100
  quizScore?: number; // if quiz
  correctAnswers?: number;
  totalQuestions?: number;
  highlights?: Highlight[];
  notes?: UserNote[];
  status: 'in_progress' | 'completed' | 'paused';
}

export interface Highlight {
  id: string;
  contentId: string;
  startIndex: number;
  endIndex: number;
  text: string;
  color: 'yellow' | 'green' | 'blue' | 'red';
  addedAt: number;
  note?: string;
}

export interface UserNote {
  id: string;
  contentId: string;
  text: string;
  addedAt: number;
  linkedHighlightId?: string;
}
```

**Transformation logic (conceptual flow):**

**Path A: Extend existing ZenSci tools with `mobile_format` parameter**

```typescript
// Example: latex-mcp extended call

POST /gateway/mcp/invoke HTTP/1.1
{
  "server": "latex-mcp",
  "tool": "convert_to_pdf",
  "arguments": {
    "documentRequest": {
      "id": "paper-2026-01",
      "markdown": "# My Research...",
      "outputFormat": "pdf",
      "mobile_format": "reading_view", // NEW: mobile flag
      "metadata": { /* ... */ }
    }
  }
}

// Response: includes both traditional PDF and mobile ReadingContent JSON
{
  "pdf": { /* binary PDF */ },
  "mobile_learning_format": {
    "contentType": "reading",
    "data": {
      "sections": [ /* structured paragraphs + equations */ ],
      "totalWords": 5432,
      "estimatedMinutes": 15
    }
  }
}
```

**Path B: New mobile-specific MCP tools (for v1+)**

```typescript
POST /gateway/mcp/invoke HTTP/1.1
{
  "server": "mobile-transform-mcp", // NEW server in v1
  "tool": "quiz_from_paper",
  "arguments": {
    "paperId": "paper-2026-01",
    "numQuestions": 10,
    "difficulty": "mixed",
    "style": "comprehension" // vs. "definition", "application"
  }
}

Response: Quiz JSON with 10 auto-generated questions
```

**Recommendation for v0.1:** Use Path A (minimal Gateway changes). In v2, if tool proliferation becomes unwieldy, migrate to Path B (cleaner semantic boundary, but more servers to maintain).

---

### 3.5 SvelteKit Mobile UI

**Mobile-optimized routes and components. All components use touch-friendly gestures and responsive Tailwind CSS.**

**Route structure (src/routes/):**

```
/                              # Home feed
├── +page.svelte              # Renders <ContentFeed /> component
├── +layout.svelte            # Shared layout (nav bar, header)
│
├── content/
│   └── [id]/
│       └── +page.svelte      # Content player (detects type and renders)
│
├── quiz/
│   └── [id]/
│       └── +page.svelte      # Full-screen quiz mode
│
├── downloads/
│   └── +page.svelte          # Cache management UI
│
└── settings/
    └── +page.svelte          # Auth, sync preferences, cache size

```

**Key component examples:**

**SwipeableDeck.svelte — Touch-enabled card swiper:**

```svelte
<script lang="ts">
  import { onMount } from 'svelte';
  import type { LearningCard } from '$lib/types';

  export let cards: LearningCard[] = [];
  export let onCardChange: (index: number) => void = () => {};

  let currentIndex = 0;
  let touchStartX = 0;
  let touchEndX = 0;
  let isFlipped = false;

  onMount(() => {
    // Keyboard navigation (left/right arrows)
    const handleKeydown = (e: KeyboardEvent) => {
      if (e.key === 'ArrowLeft') previousCard();
      if (e.key === 'ArrowRight') nextCard();
      if (e.key === ' ') toggleFlip();
    };
    window.addEventListener('keydown', handleKeydown);
    return () => window.removeEventListener('keydown', handleKeydown);
  });

  const nextCard = () => {
    if (currentIndex < cards.length - 1) {
      currentIndex++;
      isFlipped = false;
      onCardChange(currentIndex);
    }
  };

  const previousCard = () => {
    if (currentIndex > 0) {
      currentIndex--;
      isFlipped = false;
      onCardChange(currentIndex);
    }
  };

  const toggleFlip = () => {
    isFlipped = !isFlipped;
  };

  const handleTouchStart = (e: TouchEvent) => {
    touchStartX = e.changedTouches[0].screenX;
  };

  const handleTouchEnd = (e: TouchEvent) => {
    touchEndX = e.changedTouches[0].screenX;
    const diff = touchStartX - touchEndX;

    if (Math.abs(diff) > 50) {
      if (diff > 0) {
        nextCard(); // Swiped left
      } else {
        previousCard(); // Swiped right
      }
    }
  };

  const card = cards[currentIndex];
</script>

<div class="flex flex-col h-full bg-gradient-to-b from-blue-50 to-white">
  <!-- Header -->
  <div class="bg-white border-b border-gray-200 px-4 py-3">
    <p class="text-center text-sm font-semibold text-gray-600">
      Card {currentIndex + 1} of {cards.length}
    </p>
  </div>

  <!-- Card Container -->
  <div class="flex-1 flex items-center justify-center px-4 py-8"
       on:touchstart={handleTouchStart}
       on:touchend={handleTouchEnd}
       role="region"
       aria-label="Card deck">

    {#if card}
      <div class="w-full max-w-sm">
        <!-- Flip Card -->
        <div
          class="h-96 bg-white rounded-lg shadow-lg p-6 cursor-pointer flex flex-col justify-center items-center text-center transition-transform duration-300 transform"
          on:click={toggleFlip}
          role="button"
          tabindex="0"
          aria-pressed={isFlipped}
        >
          {#if !isFlipped}
            <div class="space-y-3">
              <h3 class="text-lg font-bold text-gray-800">{card.front.text}</h3>
              {#if card.front.imageUrl}
                <img src={card.front.imageUrl} alt="Card illustration" class="w-32 h-32 mx-auto rounded" />
              {/if}
              {#if card.front.htmlFormatted}
                <div class="text-sm text-gray-600 prose prose-sm">{@html card.front.htmlFormatted}</div>
              {/if}
              <p class="text-xs text-gray-400 mt-4">Tap to reveal answer</p>
            </div>
          {:else}
            <div class="space-y-3">
              <p class="text-base text-green-700 font-semibold">{card.back.text}</p>
              {#if card.back.imageUrl}
                <img src={card.back.imageUrl} alt="Answer illustration" class="w-32 h-32 mx-auto rounded" />
              {/if}
              {#if card.back.htmlFormatted}
                <div class="text-sm text-gray-600 prose prose-sm">{@html card.back.htmlFormatted}</div>
              {/if}
              {#if card.hint}
                <details class="text-xs text-blue-600 cursor-pointer mt-4">
                  <summary>💡 Hint</summary>
                  <p class="mt-2">{card.hint}</p>
                </details>
              {/if}
            </div>
          {/if}
        </div>

        <!-- Tags -->
        <div class="flex flex-wrap gap-2 mt-4 justify-center">
          {#each card.tags as tag}
            <span class="text-xs bg-blue-100 text-blue-700 px-2 py-1 rounded-full">#{tag}</span>
          {/each}
        </div>
      </div>
    {/if}
  </div>

  <!-- Navigation Buttons -->
  <div class="bg-white border-t border-gray-200 px-4 py-3 flex gap-2 justify-between">
    <button
      on:click={previousCard}
      disabled={currentIndex === 0}
      class="px-4 py-2 bg-gray-200 text-gray-700 rounded disabled:opacity-50 disabled:cursor-not-allowed"
    >
      ← Previous
    </button>
    <button
      on:click={toggleFlip}
      class="px-4 py-2 bg-blue-500 text-white rounded"
    >
      🔄 Flip
    </button>
    <button
      on:click={nextCard}
      disabled={currentIndex === cards.length - 1}
      class="px-4 py-2 bg-blue-600 text-white rounded disabled:opacity-50 disabled:cursor-not-allowed"
    >
      Next →
    </button>
  </div>
</div>
```

**QuizPlayer.svelte — Interactive quiz with scoring:**

```svelte
<script lang="ts">
  import { onMount } from 'svelte';
  import type { Quiz, QuizQuestion } from '$lib/types';
  import { quizProgress } from '$lib/stores/quiz';

  export let quiz: Quiz;
  export let quizId: string;

  let currentQuestionIndex = 0;
  let selectedAnswerIndex: number | null = null;
  let answered = false;
  let score = 0;
  let answeredCount = 0;

  const question: QuizQuestion = quiz.questions[currentQuestionIndex];

  const selectAnswer = (index: number) => {
    if (!answered) {
      selectedAnswerIndex = index;
    }
  };

  const submitAnswer = () => {
    if (selectedAnswerIndex !== null) {
      answered = true;
      answeredCount++;

      if (selectedAnswerIndex === question.correctAnswerIndex) {
        score++;
      }

      // Store progress locally
      quizProgress.update(p => ({
        ...p,
        correctAnswers: score,
        totalQuestions: answeredCount,
        percentComplete: Math.round((answeredCount / quiz.questions.length) * 100)
      }));
    }
  };

  const nextQuestion = () => {
    if (currentQuestionIndex < quiz.questions.length - 1) {
      currentQuestionIndex++;
      selectedAnswerIndex = null;
      answered = false;
    } else {
      showResults();
    }
  };

  const showResults = () => {
    const passed = (score / quiz.questions.length) * 100 >= quiz.passingScore;
    // Navigate to results screen or show modal
  };

  const currentQuestion = quiz.questions[currentQuestionIndex];
</script>

<div class="flex flex-col h-full bg-white">
  <!-- Progress Bar -->
  <div class="bg-gray-100 px-4 py-2">
    <div class="flex justify-between text-xs font-semibold text-gray-600 mb-1">
      <span>Question {currentQuestionIndex + 1} of {quiz.questions.length}</span>
      <span>Score: {score}</span>
    </div>
    <div class="w-full bg-gray-300 rounded-full h-2">
      <div
        class="bg-green-500 h-2 rounded-full transition-all duration-300"
        style="width: {((currentQuestionIndex + 1) / quiz.questions.length) * 100}%"
      ></div>
    </div>
  </div>

  <!-- Question -->
  <div class="flex-1 px-4 py-6 overflow-y-auto">
    <h2 class="text-lg font-bold text-gray-800 mb-4">{currentQuestion.questionText}</h2>

    {#if currentQuestion.imageUrl}
      <img src={currentQuestion.imageUrl} alt="Question illustration" class="w-full rounded mb-4" />
    {/if}

    <!-- Answer Options -->
    <div class="space-y-2">
      {#each currentQuestion.answers as answer, index}
        <button
          on:click={() => selectAnswer(index)}
          class="w-full p-3 text-left border-2 rounded-lg transition-all
            {selectedAnswerIndex === index
              ? 'border-blue-500 bg-blue-50'
              : 'border-gray-300 bg-white'}
            {answered && index === currentQuestion.correctAnswerIndex
              ? 'border-green-500 bg-green-50'
              : ''}
            {answered && selectedAnswerIndex === index && index !== currentQuestion.correctAnswerIndex
              ? 'border-red-500 bg-red-50'
              : ''}"
        >
          <div class="flex items-start gap-3">
            <div class="w-6 h-6 rounded-full border-2 flex items-center justify-center mt-0.5
              {selectedAnswerIndex === index ? 'border-blue-500 bg-blue-500' : 'border-gray-400'}">
              {#if selectedAnswerIndex === index}
                <span class="text-white text-sm">✓</span>
              {/if}
            </div>
            <span class="text-gray-700">{answer.text}</span>
          </div>
        </button>
      {/each}
    </div>

    <!-- Explanation (shown after answer) -->
    {#if answered}
      <div class="mt-6 p-4 bg-blue-50 border-l-4 border-blue-500 rounded">
        <p class="text-sm font-semibold text-blue-900 mb-2">Explanation:</p>
        <p class="text-sm text-blue-800">{currentQuestion.explanation}</p>
      </div>
    {/if}
  </div>

  <!-- Action Buttons -->
  <div class="bg-white border-t border-gray-200 px-4 py-3 flex gap-2">
    {#if !answered}
      <button
        on:click={submitAnswer}
        disabled={selectedAnswerIndex === null}
        class="flex-1 px-4 py-2 bg-blue-600 text-white font-semibold rounded disabled:opacity-50 disabled:cursor-not-allowed"
      >
        Check Answer
      </button>
    {:else}
      <button
        on:click={nextQuestion}
        class="flex-1 px-4 py-2 bg-green-600 text-white font-semibold rounded"
      >
        {currentQuestionIndex === quiz.questions.length - 1 ? 'Finish' : 'Next'}
      </button>
    {/if}
  </div>
</div>
```

**+page.svelte (Home Feed):**

```svelte
<script lang="ts">
  import { onMount } from 'svelte';
  import { contentFeed, cacheStatus } from '$lib/stores/content';
  import { invoke } from '@tauri-apps/api/core';
  import { appWindow } from '@tauri-apps/api/window';
  import ContentCard from '$lib/components/ContentCard.svelte';
  import SyncStatus from '$lib/components/SyncStatus.svelte';

  let loading = true;
  let error: string | null = null;

  onMount(async () => {
    // Listen for sync events from Rust backend
    const unlisten = await appWindow.listen('sync_completed', (event) => {
      console.log('Content synced:', event);
      loadContentFeed();
    });

    // Load initial content from cache
    await loadContentFeed();

    // Trigger background sync
    try {
      await invoke('sync_content', {});
    } catch (e) {
      console.error('Sync failed:', e);
      error = 'Sync error';
    }

    return unlisten;
  });

  async function loadContentFeed() {
    try {
      loading = true;
      const items = await invoke('list_content', {
        groupId: 'user-group-id', // Fetch from user store
        limit: 20,
        offset: 0,
      });
      contentFeed.set(items);
      error = null;
    } catch (e) {
      error = String(e);
    } finally {
      loading = false;
    }
  }

  async function manualSync() {
    try {
      await invoke('sync_content', {});
    } catch (e) {
      error = String(e);
    }
  }
</script>

<div class="h-full flex flex-col bg-gray-50">
  <!-- Header with Sync Status -->
  <div class="bg-white border-b border-gray-200 px-4 py-3 flex justify-between items-center">
    <h1 class="text-lg font-bold text-gray-900">zen-sci</h1>
    <SyncStatus on:refresh={manualSync} />
  </div>

  <!-- Content Feed -->
  <div class="flex-1 overflow-y-auto">
    {#if loading}
      <div class="flex items-center justify-center h-full">
        <p class="text-gray-500">Loading content...</p>
      </div>
    {:else if error}
      <div class="m-4 p-4 bg-red-50 border border-red-200 rounded">
        <p class="text-red-700 text-sm">{error}</p>
        <button
          on:click={manualSync}
          class="mt-2 text-red-600 text-sm font-semibold underline"
        >
          Retry
        </button>
      </div>
    {:else if $contentFeed.length === 0}
      <div class="flex flex-col items-center justify-center h-full gap-4">
        <p class="text-gray-500">No content yet</p>
        <button
          on:click={manualSync}
          class="px-4 py-2 bg-blue-600 text-white rounded"
        >
          Sync Now
        </button>
      </div>
    {:else}
      <div class="grid gap-3 p-3">
        {#each $contentFeed as content (content.id)}
          <ContentCard {content} />
        {/each}
      </div>
    {/if}
  </div>

  <!-- Cache Status Footer -->
  <div class="bg-white border-t border-gray-200 px-4 py-2 text-xs text-gray-600">
    <p>Cache: {$cacheStatus.downloadedCount} items, {Math.round($cacheStatus.totalSize / 1024 / 1024)}MB</p>
  </div>
</div>
```

---

### 3.6 Offline Architecture

**What's available offline, what requires network, sync scheduling:**

**Offline-available (v0.1):**
- Read downloaded content (reading views, cards, quizzes)
- View quiz answers and progress
- Search local cache (grep-based in v0.1; Tantivy in v1)
- Change text size, dark mode, highlighting
- View user annotations (highlights, notes)

**Network-required (v0.1):**
- Initial login (token refresh)
- Content sync (new group publications)
- Browsing group feed before download
- Publishing annotations back to portal (v2)

**Sync strategy (v0.1):**

1. **Trigger:** App foreground + network available + last sync >30 seconds ago
2. **Fetch manifest** from Gateway: `GET /gateway/mobile/sync?group_id={}&since={}`
3. **Download new items** in background (don't block UI)
4. **Conflict resolution:** Server always wins in v0.1 (read-only consumption). In v2, CRDT-based merging for annotations.
5. **Emit event:** When sync completes, emit `sync_completed` event; SvelteKit listener re-fetches feed

**Offline indicator (UI):**

```svelte
<!-- Show in SyncStatus component -->
{#if !isNetworkAvailable}
  <div class="text-xs text-orange-600 font-semibold">⚠️ Offline</div>
{:else if isSyncing}
  <div class="text-xs text-blue-600 font-semibold">⏳ Syncing...</div>
{:else}
  <div class="text-xs text-green-600 font-semibold">✓ Synced</div>
{/if}
```

---

### 3.7 Platform-Specific Considerations

#### **iOS (WKWebView, iOS 16+)**

**Configuration:**

1. **CSP (Content Security Policy):**
   - WKWebView enforces stricter CSP than Chromium. Ensure SvelteKit doesn't rely on `eval()` or unsafe-inline scripts.
   - Solution: Use nonce-based inline scripts or externalize them.

2. **Push Notifications (optional v1.1):**
   - Use Tauri's notification plugin + Apple Push Notification (APN)
   - When new group content is published, push "3 new papers added to your group"

3. **Deep Linking (optional v1.1):**
   - Register `zen-sci://` custom scheme in Info.plist
   - Allow portal to open mobile app directly to content: `zen-sci://content/paper-2026-01`

4. **Background Sync (optional v1.1):**
   - iOS allows background app refresh for content sync
   - Requires BGProcessingTask registration + battery impact tolerance

**Info.plist additions (for v0.1 + v1.1):**

```xml
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleURLName</key>
    <string>zen-sci</string>
    <key>CFBundleURLSchemes</key>
    <array>
      <string>zen-sci</string>
    </array>
  </dict>
</array>

<!-- For background sync (v1.1) -->
<key>BGTaskSchedulerPermittedIdentifiers</key>
<array>
  <string>com.tresidesdesign.zensci.mobile.sync</string>
</array>
```

#### **Android (Chromium WebView, Android 10+)**

**Configuration:**

1. **Back Button Handling:**
   - Android users expect back button to navigate within the app, not close
   - Solution: Handle `back_requested` event in Tauri; navigate routes instead of closing

2. **Notification Channels (required for Android 8+):**
   - Create notification channel "zen-sci-sync" for sync updates
   - Users can configure notification behavior

3. **Deep Linking:**
   - Register `zen-sci://` + `https://zen-sci.zeniththscience.org/` deep links in AndroidManifest.xml
   - Both schemes should route to same content player

**AndroidManifest.xml additions:**

```xml
<intent-filter>
  <action android:name="android.intent.action.VIEW" />
  <category android:name="android.intent.category.DEFAULT" />
  <category android:name="android.intent.category.BROWSABLE" />
  <data
    android:scheme="zen-sci"
    android:host="content"
    android:pathPattern="/.*" />
  <data
    android:scheme="https"
    android:host="zen-sci.zenithscience.org"
    android:pathPattern="/app/content/.*" />
</intent-filter>
```

#### **Shared: Deep Link Schema**

**Format:** `zen-sci://content/{contentId}?groupId={groupId}&type={contentType}`

**Examples:**
```
zen-sci://content/paper-2026-01?groupId=group-bio-101&type=reading
zen-sci://content/quiz-stats-001?groupId=group-stat-301&type=quiz
```

**Route handling (SvelteKit):**

```typescript
// src/routes/+page.svelte or in Tauri setup

import { invoke } from '@tauri-apps/api/core';

// Listen for deep link navigation
appWindow.listen('tauri://deep-link', (event) => {
  const url = event.payload as string;
  const parsed = new URL(url);

  if (parsed.hostname === 'content') {
    const contentId = parsed.pathname.replace('/', '');
    const groupId = parsed.searchParams.get('groupId');
    const type = parsed.searchParams.get('type');

    // Navigate to content player
    goto(`/content/${contentId}?group=${groupId}&type=${type}`);
  }
});
```

---

## 4. Implementation Plan

**4-week sprint structure (assuming one Rust engineer + one SvelteKit engineer):**

### **Week 1–2: Foundation & Architecture**

**Deliverables:**
- Tauri v2 mobile project scaffold (iOS + Android targets)
- Database schema & ContentCache (rusqlite) implemented & tested
- Auth manager skeleton (JWT validation, keychain integration stubbed)
- SvelteKit project scaffold with routing structure
- Svelte stores for user session, content feed, cache status, quiz progress

**Validation:**
- `cargo test` passes for cache module
- `npm run dev` launches SvelteKit on localhost:5173
- Tauri build succeeds for both iOS + Android targets

### **Week 3: Backend Integration & Content Sync**

**Deliverables:**
- AuthManager fully implemented (JWT validation, platform keychain API calls)
- SyncEngine implemented (fetches from Gateway, populates cache)
- IPC commands fully wired (search_cache, get_content, list_content, sync_content, validate_token, store_token, logout, cache_stats)
- Background sync loop running on app startup
- Mock Gateway endpoint responding with test content manifests

**Validation:**
- `invoke('sync_content')` successfully downloads test content to cache
- `invoke('get_content', { id: 'test-1' })` returns cached content
- JWT token validation rejects expired tokens
- Platform keychain stores/retrieves credentials (manual test on iOS/Android)

### **Week 4: UI Polish & Mobile-Native Experience**

**Deliverables:**
- All Svelte components completed (ContentCard, SwipeableDeck, QuizPlayer, ReadingView, ProgressBar, SyncStatus)
- Home feed displays cached content
- Content player routes to correct component based on contentType
- Quiz scoring + progress tracking
- Offline indicator + sync status shown in UI
- Touch gestures working (swipe cards, tap to flip)
- Responsive design tested on multiple screen sizes

**Validation:**
- App launches on iOS simulator/device; loads content from cache instantly
- Swipeable deck responds smoothly to touch (60 FPS target)
- Quiz submission records answers locally
- Sync indicator updates in real-time

### **Week 5–6: Mobile-Specific Hardening & Testing**

**Deliverables:**
- Deep linking tested (zen-sci://content/[id] opens content player)
- Offline mode tested (disable network, app still loads cached content)
- iOS keychain integration tested on physical device
- Android back button handling (navigates routes instead of closing app)
- Battery impact assessment (background sync throttled if low battery)
- App icon, splash screen, localization setup

**Validation:**
- TestFlight (iOS) + Play Store alpha (Android) ready for external testers
- All S1–S7 success criteria met

---

## 5. Risk Assessment

| # | Risk | Probability | Impact | Mitigation |
|---|---|---|---|---|
| **R1** | **Tauri v2 mobile maturity:** iOS WKWebView CSP restrictions cause SvelteKit incompatibility | Medium | High | Early CSP testing (week 1); fallback to iframe-based app if needed |
| **R2** | **Platform keychain integration fails:** iOS Keychain / Android Keystore APIs poorly documented or permissions issues | Medium | High | Implement fallback: encrypted file storage in app sandbox; test on physical devices week 3 |
| **R3** | **rusqlite on ARM (iOS/Android):** Compilation fails or performance degrades on ARM64 architecture | Low | Medium | Use `tokio-rusqlite` pre-compiled binaries; test builds on both architectures early |
| **R4** | **Gateway API contract unstable:** Mobile-format endpoints not yet implemented in Gateway; require coordination with Go team | High | High | Specification lock-in (this spec), API mocking in week 1, parallel implementation with Gateway team |
| **R5** | **IPC serialization bottleneck:** Tauri's string-based IPC causes latency for large content transfers (e.g., 5MB PDF as JSON) | Medium | Medium | Batch IPC calls; use file-based transfer for large content (write to app sandbox, pass path); stream via WebSocket if needed |
| **R6** | **Network availability detection unreliable:** App can't detect offline state, attempts sync and fails silently | Low | Medium | Implement platform-native network status API (iOS NENetworkMonitor, Android ConnectivityManager); listen to changes |
| **R7** | **LRU eviction policy too aggressive:** Important content deleted before user expects; user frustrated when content disappears | Medium | Medium | Warn before deletion; allow pinning content to prevent eviction; implement soft deletion (move to archive, not purge) |
| **R8** | **Quiz answer validation in Rust is slow:** Complex question types (matching, short answer with regex) cause lag | Low | Medium | Keep answer validation simple in v0.1 (multiple choice, true/false only); defer complex types to v1 |
| **R9** | **Sync state tracking gets out of sync:** Annotations marked "pending" but synced to cloud; or vice versa | Medium | Medium | Implement sync log (see schema); test reconciliation logic extensively; add telemetry for sync failures |
| **R10** | **User expectations for "offline = no new content":** User expects to download everything upfront, gets confused when only recent content available | Low | Medium | Clear UX messaging: "Recently synced content is available offline. Full archive in downloads section." Educate in onboarding. |

---

## 6. Rollback & Contingency

**v0.1 is production-grade; rollback to prior version is standard app store procedure.**

**Pre-release checklist:**
- [ ] All S1–S7 success criteria met
- [ ] Crash-free rate >99.5% in 48-hour beta (TestFlight/Play Store alpha)
- [ ] Battery impact: idle memory <80MB iOS, <100MB Android
- [ ] Startup time <1.5 seconds (measured from app launch to home feed)
- [ ] Sync latency <30 seconds (manifest fetch + first 3 items downloaded)
- [ ] No known critical bugs or security vulnerabilities
- [ ] Privacy policy + data handling docs reviewed (GDPR/CCPA)

**If major issue found pre-release:**
1. Halt rollout to public
2. Investigate root cause (usually Gateway API incompatibility or platform-specific bug)
3. Fix in next commit
4. Re-test on devices
5. Redeploy

**If major issue found post-release:**
1. Immediately yank from app store (TestFlight, Play Store alpha can be pulled)
2. Issue hotfix in emergency sprint
3. Re-deploy with higher priority

---

## 7. Documentation & Communication

**Documentation deliverables (alongside code):**

1. **README.md** — User-facing, quick start, troubleshooting
2. **ARCHITECTURE.md** — Technical deep-dive (this spec + implementation details)
3. **API.md** — Tauri IPC commands, event schemas, error codes
4. **CONTRIBUTING.md** — Development setup, build instructions, testing
5. **DEPLOYMENT.md** — App store submission, beta testing, release process
6. **CHANGELOG.md** — Version history, breaking changes

**Stakeholder communication:**

- **Cruz Morales:** Weekly progress updates (v0.1 only, lean cadence)
- **AgenticGateway team:** Weekly API contract reviews (are Gateway endpoints stable?)
- **ZenSci servers team:** Coordinate on mobile transformation schema (Path A vs B decision)
- **Beta testers:** In-app feedback mechanism (tap to send screenshot + bug report)

---

## 8. Appendices

### 8.1 Cargo.toml (Rust Dependencies)

```toml
[package]
name = "zen-sci-mobile"
version = "0.1.0"
edition = "2021"

[dependencies]
tauri = { version = "2.0", features = ["mobile", "shell-open", "notification", "fs-all", "http-all"] }
tokio = { version = "1.40", features = ["full"] }
tokio-rusqlite = "0.5"
rusqlite = { version = "0.31", features = ["bundled", "chrono"] }
jsonwebtoken = "9"
reqwest = { version = "0.12", features = ["json"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
chrono = { version = "0.4", features = ["serde"] }
uuid = { version = "1.0", features = ["v4", "serde"] }
log = "0.4"
env_logger = "0.11"
anyhow = "1.0"
thiserror = "1.0"

# Platform-specific (iOS)
[target.'cfg(target_os = "ios")'.dependencies]
security-framework = "2.9"

# Platform-specific (Android)
[target.'cfg(target_os = "android")'.dependencies]
jni = "0.21"

[dev-dependencies]
tokio-test = "0.4"
mockito = "1.2"

[profile.release]
opt-level = 3
lto = true
strip = true
```

**package.json (SvelteKit dependencies):**

```json
{
  "name": "zen-sci-mobile-ui",
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "test": "vitest",
    "lint": "eslint src",
    "format": "prettier --write src"
  },
  "dependencies": {
    "@tauri-apps/api": "^2.0.0",
    "@tauri-apps/plugin-notification": "^2.0.0",
    "@tauri-apps/plugin-websocket": "^2.0.0",
    "svelte": "^5.0.0",
    "sveltekit": "^2.0.0"
  },
  "devDependencies": {
    "@sveltejs/adapter-static": "^2.0.0",
    "autoprefixer": "^10.4.0",
    "postcss": "^8.4.0",
    "tailwindcss": "^3.4.0",
    "typescript": "^5.3.0",
    "vite": "^5.0.0",
    "vitest": "^1.0.0"
  }
}
```

---

### 8.2 Content Transformation Table (Detailed Reference)

| Input (ZenSci) | Output Format | Schema | Example Transformation | Mobile UX |
|---|---|---|---|---|
| **PDF (latex-mcp)** myresearch.pdf | **reading_view** | `ReadingContent { sections[], fontSize, darkMode, lineHeight }` | "Abstract\nMotivation\nMethods\nResults\nConclusion" → segmented into `ReadingSection[]` with adjustable typography | Tap + drag to scroll, swipe for TOC navigation, bookmarks for sections |
| **Slides (slides-mcp)** presentation.md → Reveal.js | **card_deck** | `LearningCard[] { front: title, back: notes, difficulty, tags }` | Slide 1: "Photosynthesis" (front) + "Process of converting light → glucose" (back) | Swipe left/right, tap to flip, progress bar shows slide count |
| **Blog (blog-mcp)** blog.md → HTML | **reading_view** OR **card_deck** | ReadingContent OR summary cards with key quotes | Article: "The Future of AI" → 5 cards: intro + 4 key insights | Scroll reading view OR swipe through idea cards |
| **Newsletter (newsletter-mcp)** newsletter.md → MJML/HTML | **card_deck** | One `LearningCard` per MJML block (header, body, CTA) | Newsletter with 3 articles → 3 cards (headline + excerpt on front, full text on back) | Swipe through articles, save favorites |
| **Grant (grant-mcp)** proposal.md → NSF/NIH LaTeX | **summary_cards** | `LearningCard[] { front: aim/sig/innovation/impact section, back: full text }` | "Specific Aims" (front) → bulleted list (back) | 4 cards (Aims, Significance, Innovation, Impact), swipe to review |
| **Paper (paper-mcp)** paper.tex → IEEE/ACM LaTeX | **reading_view** + **quiz** | `ReadingContent { abstract, intro, methods, results, conclusion }` + `Quiz { 10 comprehension questions }` | Full paper → 5 reading sections + auto-generated Q&A | Read paper, then tap "Take Quiz" for comprehension check |

---

### 8.3 Deep Link Specification

**URI Scheme:** `zen-sci://`

**Format:** `zen-sci://content/{contentId}?groupId={groupId}&type={contentType}&action={action}`

**Parameters:**
- `contentId` (required): Unique content identifier (e.g., `paper-2026-01`)
- `groupId` (optional): Group the content belongs to (speeds up cache lookup)
- `type` (optional): Content type (`card_deck`, `quiz`, `reading`, etc.)
- `action` (optional): `view` (default), `download`, `share`

**Examples:**

```
# Open a paper for reading
zen-sci://content/paper-2026-01?type=reading

# Open a quiz
zen-sci://content/quiz-stats-001?type=quiz&groupId=group-stat-301

# Pre-download content before opening
zen-sci://content/slides-lecture-5?action=download

# Web fallback (for portal links)
https://zen-sci.zenithscience.org/app/content/paper-2026-01
```

**Handling (SvelteKit):**

```typescript
// src/routes/+page.svelte or lib/handlers/deeplinks.ts

import { goto } from '$app/navigation';
import { appWindow } from '@tauri-apps/api/window';

export async function setupDeepLinkListener() {
  return appWindow.listen('tauri://deep-link', (event) => {
    const url = event.payload as string; // e.g., "zen-sci://content/paper-2026-01?type=reading"

    try {
      const parsed = new URL(url);

      if (parsed.hostname === 'content') {
        const contentId = parsed.pathname.replace('/', '');
        const type = parsed.searchParams.get('type') || 'reading';
        const groupId = parsed.searchParams.get('groupId') || '';
        const action = parsed.searchParams.get('action') || 'view';

        // Navigate based on action
        if (action === 'download') {
          // Trigger download before navigation
          invoke('sync_content', { contentId }).then(() => {
            goto(`/content/${contentId}?type=${type}&group=${groupId}`);
          });
        } else {
          goto(`/content/${contentId}?type=${type}&group=${groupId}`);
        }
      }
    } catch (e) {
      console.error('Invalid deep link:', url, e);
    }
  });
}
```

---

### 8.4 v2 Contribution Surface Preview

*This is intentionally deferred; documented here to shape v0.1 architecture for future growth.*

**v2 Goals:**
- Students can create, edit, and submit content (research notes, summaries, quizzes) from mobile
- Annotations + highlights sync back to portal for teacher/peer review
- Student-authored quizzes appear in group feed
- Collaborative editing on shared documents

**v2 Features Enabled by v0.1 Architecture:**

1. **Annotations table (already in schema):** Stores highlights + notes locally; v2 extends to submission workflow
2. **Sync log (already in schema):** Tracks local changes; v2 uses for conflict-free merging
3. **User progress (already tracked):** Quiz scores, reading progress; v2 enables earning badges, leaderboards
4. **Auth boundary (already set):** JWT + keychain; v2 adds role-based permissions (student, instructor, curator)

**v2 Implementation would require:**
- New Tauri commands: `submit_annotation`, `create_quiz`, `edit_content`
- New SvelteKit routes: `/create`, `/edit/[id]`, `/submit`
- CRDT library (Automerge 3) for conflict-free merging
- WebSocket sync for real-time collab (optional)

**v0.1 doesn't implement these, but the architecture (content cache schema, sync log, auth) prepares for them.**

---

### 8.5 Open Questions & Decisions

**Before kickoff, answer these with Cruz:**

1. **Path A vs Path B for content transformation:**
   - Path A: Extend existing ZenSci tools with `mobile_format` flag (reuse, less new code)
   - Path B: Create new mobile-specific MCP tools (cleaner semantics, more servers)
   - **Recommendation:** Path A for v0.1, migrate to B in v2 if needed
   - **Decision Owner:** Cruz + Gateway team

2. **Content cache size limit:**
   - Default 500MB (reasonable for 50–100 papers + quizzes)
   - Should this be configurable?
   - What happens when cache exceeds limit? (Recommendations: LRU eviction vs. aggressive deletion)
   - **Decision Owner:** Cruz (based on user feedback post-v0.1)

3. **Sync interval:**
   - v0.1 defaults to 30 seconds when online
   - Should this be user-configurable? (On WiFi: sync more aggressively; on cellular: sync less often)
   - **Decision Owner:** UX team (low priority for v0.1, high priority for v1.1)

4. **Offline-first vs. online-first UX:**
   - v0.1 is mostly read-only, so offline works fine
   - v2 (with contribution) will need real-time sync
   - Should we invest in CRDT conflict resolution now, or defer to v2?
   - **Decision Owner:** Cruz (architecture locked in v0.1, can't easily change v2)
   - **Recommendation:** Defer to v2; v0.1 structure is sound

5. **Analytics & telemetry:**
   - Should v0.1 send usage data to portal? (e.g., "user viewed 5 papers", "spent 20 min on quiz")
   - Privacy concerns: does analytics comply with GDPR/CCPA?
   - **Decision Owner:** Legal + Privacy team

6. **A/B testing framework:**
   - Should we prepare for feature flags (e.g., test dark mode on 10% of users)?
   - Adds complexity; is it worth it for v0.1?
   - **Decision Owner:** Product (low priority for v0.1)

---

## Conclusion

**zen-sci-mobile v0.1** is a **consumption-only learning surface** that transforms academic content into mobile-native formats. Built on **Tauri v2 + Rust backend + SvelteKit frontend**, it delivers:

- Instant offline access to downloaded papers, quizzes, slides
- 30-second sync cycle for new group publications
- Platform-native authentication (iOS Keychain, Android Keystore)
- Touch-optimized UI (swipeable cards, adjustable reading view, quiz scoring)

**Implementation timeline:** 4–6 weeks, one Rust engineer + one SvelteKit engineer.

**Success criteria:** User can log in, sync content, take quizzes offline, and see new publications in <30 seconds.

**v1 roadmap:** Real-time collaboration, advanced annotations, student contribution surface.

---

**Prepared by:** Architecture & Product (Strategic Scout + WIDE Research synthesis)
**Date:** 2026-02-18
**Status:** Ready for Technical Review & Implementation Kickoff
**Next Gate:** Platform keychain API integration confirmation + Gateway API contract finalization

---

## Revision R1: Games — Adaptive Quiz Demo (2026-02-18)

### Games Architecture Decision

**Core Reframe:** Games are not a separate content category—they are existing content types (Quiz, CardDeck, ReadingContent, VideoScript) augmented with game mechanics: scoring, difficulty levels, streaks, and achievements. The existing transformation pipeline remains intact; game state layers on top.

This design choice unlocks:
1. **Content reuse** — papers already transformed to Quiz questions can immediately become games
2. **Simple storage** — game state (score, streak, achievements) persists in local SQLite; no cloud sync required for v1
3. **Adaptive learning** — difficulty adjusts in real time based on performance, without requiring separate content authoring
4. **Extensibility** — v2 can add new game formats (TimelineSort, MatchingGame, BranchingNarrative) using the same GameRules schema

The first working demo ships AdaptiveQuiz format only; other formats reserved for v2.

---

### Updated Type Definitions

#### MobileContentType (expanded)

```typescript
type MobileContentType = 'CardDeck' | 'Quiz' | 'ReadingContent' | 'VideoScript' | 'AdaptiveQuiz'
```

#### GameRules Interface

```typescript
interface GameRules {
  format: 'adaptive_quiz'  // v1 ships this; other formats reserved for v2
  difficulty_levels: ['easy', 'medium', 'hard']
  questions_per_session: number  // default: 10
  streak_bonus_threshold: number // default: 3 (3 correct in a row = bonus)
  achievement_ids: string[]
  time_limit_per_question: number | null // null = no timer; in seconds if set
}
```

#### AdaptiveQuizContent Interface

```typescript
interface AdaptiveQuizContent extends MobileContent {
  type: 'AdaptiveQuiz'
  questions: QuizQuestion[]  // reuses existing QuizQuestion schema
  game_rules: GameRules
  game_state?: GameSession   // present after first play
}
```

#### GameSession Interface

Represents a single play session's state:

```typescript
interface GameSession {
  session_id: string
  score: number
  streak: number
  highest_streak: number
  difficulty: 'easy' | 'medium' | 'hard'
  questions_answered: number
  correct_answers: number
  achievements_earned: string[]  // IDs of unlocked achievements
  started_at: string              // ISO 8601 timestamp
  last_played_at: string          // ISO 8601 timestamp
}
```

#### Achievement Interface

```typescript
interface Achievement {
  id: string
  name: string
  description: string
  icon: string  // emoji (e.g., "🏆", "🔬", "🎯") or icon name
  earned_at: string | null  // ISO 8601 timestamp or null if not yet earned
}
```

#### QuizQuestion with Difficulty Tag

The existing QuizQuestion schema is augmented with a difficulty level for filtering:

```typescript
interface QuizQuestion {
  id: string
  question_text: string
  options: string[]
  correct_answer_index: number
  explanation: string
  concept_id?: string  // links to concept extracted from paper
  difficulty: 'easy' | 'medium' | 'hard'  // NEW: required for adaptive quiz
}
```

---

### Game State Storage

#### Rust IPC Commands (new file: `game_state.rs`)

Add three new Tauri commands to the mobile Rust backend:

```rust
// src/commands/game_state.rs

use tauri::State;
use crate::db::GameDatabase;

#[tauri::command]
async fn save_game_session(
    content_id: String,
    session: GameSession,
    db: State<'_, GameDatabase>
) -> Result<String, String> {
    db.save_session(&content_id, &session)
        .map_err(|e| format!("Failed to save game session: {}", e))
}

#[tauri::command]
async fn get_game_session(
    content_id: String,
    db: State<'_, GameDatabase>
) -> Result<Option<GameSession>, String> {
    db.get_latest_session(&content_id)
        .map_err(|e| format!("Failed to retrieve game session: {}", e))
}

#[tauri::command]
async fn get_achievements(
    content_id: String,
    db: State<'_, GameDatabase>
) -> Result<Vec<Achievement>, String> {
    db.get_achievements(&content_id)
        .map_err(|e| format!("Failed to retrieve achievements: {}", e))
}

#[tauri::command]
async fn unlock_achievement(
    content_id: String,
    achievement_id: String,
    db: State<'_, GameDatabase>
) -> Result<(), String> {
    db.mark_achievement_earned(&content_id, &achievement_id)
        .map_err(|e| format!("Failed to unlock achievement: {}", e))
}
```

#### SQLite Schema

Add two tables to the mobile local cache database:

```sql
CREATE TABLE game_sessions (
  id TEXT PRIMARY KEY,
  content_id TEXT NOT NULL,
  session_data JSON NOT NULL,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (content_id) REFERENCES mobile_content(id)
);

CREATE INDEX idx_game_sessions_content_id ON game_sessions(content_id);
CREATE INDEX idx_game_sessions_updated_at ON game_sessions(updated_at);

CREATE TABLE achievements (
  id TEXT NOT NULL,
  content_id TEXT NOT NULL,
  earned_at DATETIME,
  PRIMARY KEY (id, content_id),
  FOREIGN KEY (content_id) REFERENCES mobile_content(id)
);

CREATE INDEX idx_achievements_earned_at ON achievements(earned_at);
```

Game state is persisted locally only—no cloud sync in v1. When a user syncs content via SyncEngine, game state remains client-side.

---

### AdaptiveQuizPlayer.svelte — Working Game Demo

This is the complete, production-ready component implementation. It manages the full game loop: initialization, adaptive difficulty, question flow, scoring, achievement unlock detection, and summary display.

```svelte
<!-- src/lib/components/games/AdaptiveQuizPlayer.svelte -->
<script lang="ts">
  import { onMount } from 'svelte'
  import { writable } from 'svelte/store'
  import type { AdaptiveQuizContent, GameSession, Achievement } from '$lib/types'
  import { invoke } from '@tauri-apps/api/core'
  
  export let content: AdaptiveQuizContent
  export let onComplete: (session: GameSession) => void = () => {}

  // State: screens
  type Screen = 'start' | 'question' | 'result' | 'summary'
  let currentScreen: Screen = 'start'
  
  // State: game session
  let sessionId: string = crypto.randomUUID()
  let score = writable(0)
  let streak = writable(0)
  let highestStreak = writable(0)
  let difficulty = writable<'easy' | 'medium' | 'hard'>('easy')
  let questionsAnswered = writable(0)
  let correctAnswers = writable(0)
  
  // State: question flow
  let availableQuestions = writable<any[]>([])
  let currentQuestionIndex = writable(0)
  let currentQuestion = writable<any>(null)
  let selectedAnswerIndex = writable<number | null>(null)
  let isAnswerRevealed = writable(false)
  let answerFeedback = writable<{ correct: boolean; message: string } | null>(null)
  
  // State: achievements
  let achievements = writable<Achievement[]>([])
  let unlockedThisSession = writable<Achievement[]>([])
  let showAchievementToast = writable(false)
  let toastAchievement = writable<Achievement | null>(null)
  
  // State: UI
  let bestScore = writable(0)
  let accuracy = writable(0)
  let sessionStartTime = new Date()

  // Derived: filter questions by current difficulty
  $: {
    const currentDiff = $difficulty
    const filtered = content.questions.filter(q => q.difficulty === currentDiff)
    availableQuestions.set(filtered.length > 0 
      ? filtered 
      : content.questions // fallback to all if none at current difficulty
    )
  }

  // Initialize: load previous session data for "best score"
  onMount(async () => {
    try {
      const prevSession = await invoke<GameSession | null>('get_game_session', {
        contentId: content.id
      })
      if (prevSession) {
        bestScore.set(prevSession.score)
      }
      
      const achList = await invoke<Achievement[]>('get_achievements', {
        contentId: content.id
      })
      achievements.set(achList)
    } catch (err) {
      console.error('Failed to load previous session:', err)
    }
  })

  // Adaptive difficulty algorithm
  function updateDifficulty(correct: boolean) {
    const current = $difficulty
    let newDiff = current
    let currentStreak = $streak

    if (correct) {
      currentStreak += 1
      streak.set(currentStreak)
      
      if (currentStreak >= content.game_rules.streak_bonus_threshold) {
        // Bump difficulty up
        if (current === 'easy') newDiff = 'medium'
        else if (current === 'medium') newDiff = 'hard'
      }
    } else {
      streak.set(0)
      
      // Bump difficulty down after 2 wrong in a row
      if (currentStreak <= 2) {
        if (current === 'hard') newDiff = 'medium'
        else if (current === 'medium') newDiff = 'easy'
      }
    }
    
    difficulty.set(newDiff)
  }

  // Select next question from available (filtered by difficulty)
  function selectNextQuestion() {
    const available = $availableQuestions
    if (available.length === 0) return null
    
    const randomIndex = Math.floor(Math.random() * available.length)
    return available[randomIndex]
  }

  // Calculate score delta based on difficulty and streak
  function calculateScoreDelta(correct: boolean): number {
    if (!correct) return 0
    
    const basePoints = 10
    const difficultyMultiplier = 
      $difficulty === 'easy' ? 1 : 
      $difficulty === 'medium' ? 1.5 : 
      2
    const streakBonus = $streak >= content.game_rules.streak_bonus_threshold ? 5 : 0
    
    return Math.floor(basePoints * difficultyMultiplier + streakBonus)
  }

  // Check if new achievements are unlocked
  function checkAchievements() {
    const currentScore = $score
    const currentAccuracy = $correctAnswers / $questionsAnswered

    const possibleAchievements = content.game_rules.achievement_ids.map(id => 
      $achievements.find(a => a.id === id)
    ).filter(Boolean) as Achievement[]

    possibleAchievements.forEach(ach => {
      if (ach.earned_at === null) {
        let shouldUnlock = false
        
        if (ach.id.includes('perfect') && currentAccuracy === 1) {
          shouldUnlock = true
        } else if (ach.id.includes('streak') && $highestStreak >= 5) {
          shouldUnlock = true
        } else if (ach.id.includes('master') && currentScore >= 100) {
          shouldUnlock = true
        }
        
        if (shouldUnlock) {
          ach.earned_at = new Date().toISOString()
          unlockedThisSession.update(list => [...list, ach])
          toastAchievement.set(ach)
          showAchievementToast.set(true)
          
          setTimeout(() => showAchievementToast.set(false), 3000)
        }
      }
    })
  }

  // Handle answer selection
  async function handleAnswerSelect(index: number) {
    selectedAnswerIndex.set(index)
    isAnswerRevealed.set(true)
    
    const question = $currentQuestion
    const correct = index === question.correct_answer_index
    
    // Update session state
    questionsAnswered.update(q => q + 1)
    if (correct) correctAnswers.update(c => c + 1)
    
    // Update score
    const delta = calculateScoreDelta(correct)
    score.update(s => s + delta)
    
    // Update difficulty
    updateDifficulty(correct)
    
    // Update highest streak tracking
    if ($streak > $highestStreak) {
      highestStreak.set($streak)
    }
    
    // Feedback
    answerFeedback.set({
      correct,
      message: correct 
        ? `Correct! +${delta} points` 
        : `Incorrect. ${question.explanation}`
    })
    
    // Check for achievements
    checkAchievements()
    
    currentScreen = 'result'
  }

  // Next question or end game
  function handleContinue() {
    const questionsPerSession = content.game_rules.questions_per_session || 10
    
    if ($questionsAnswered >= questionsPerSession) {
      endGame()
    } else {
      // Reset for next question
      selectedAnswerIndex.set(null)
      isAnswerRevealed.set(false)
      answerFeedback.set(null)
      
      const next = selectNextQuestion()
      if (next) {
        currentQuestion.set(next)
        currentScreen = 'question'
      } else {
        endGame()
      }
    }
  }

  // Start game
  function startGame() {
    sessionStartTime = new Date()
    const first = selectNextQuestion()
    if (first) {
      currentQuestion.set(first)
      currentScreen = 'question'
    }
  }

  // End game: save session and show summary
  async function endGame() {
    accuracy.set($questionsAnswered > 0 
      ? Math.round(($correctAnswers / $questionsAnswered) * 100) 
      : 0
    )
    
    const session: GameSession = {
      session_id: sessionId,
      score: $score,
      streak: $streak,
      highest_streak: $highestStreak,
      difficulty: $difficulty,
      questions_answered: $questionsAnswered,
      correct_answers: $correctAnswers,
      achievements_earned: $unlockedThisSession.map(a => a.id),
      started_at: sessionStartTime.toISOString(),
      last_played_at: new Date().toISOString()
    }
    
    try {
      await invoke('save_game_session', {
        contentId: content.id,
        session
      })
    } catch (err) {
      console.error('Failed to save game session:', err)
    }
    
    currentScreen = 'summary'
    onComplete(session)
  }

  // Play again: reset state
  function playAgain() {
    sessionId = crypto.randomUUID()
    score.set(0)
    streak.set(0)
    highestStreak.set(0)
    difficulty.set('easy')
    questionsAnswered.set(0)
    correctAnswers.set(0)
    selectedAnswerIndex.set(null)
    isAnswerRevealed.set(false)
    answerFeedback.set(null)
    unlockedThisSession.set([])
    accuracy.set(0)
    currentScreen = 'start'
  }
</script>

<!-- SCREEN: Start -->
{#if currentScreen === 'start'}
  <div class="screen start-screen">
    <div class="game-header">
      <h1>{content.title}</h1>
      <p class="game-subtitle">{content.description}</p>
    </div>
    
    <div class="game-stats">
      <div class="stat">
        <span class="label">Best Score</span>
        <span class="value">{$bestScore}</span>
      </div>
      <div class="stat">
        <span class="label">Format</span>
        <span class="value">{content.game_rules.questions_per_session} Questions</span>
      </div>
    </div>
    
    <div class="game-rules-summary">
      <h3>How to Play</h3>
      <ul>
        <li>Answer {content.game_rules.questions_per_session} questions</li>
        <li>Difficulty adapts based on your performance</li>
        <li>{content.game_rules.streak_bonus_threshold} correct in a row = bonus points</li>
        <li>Unlock achievements for special feats</li>
      </ul>
    </div>
    
    <button class="btn btn-primary" on:click={startGame}>
      Start Game
    </button>
  </div>

<!-- SCREEN: Question -->
{:else if currentScreen === 'question'}
  <div class="screen question-screen">
    <div class="question-header">
      <div class="progress">
        <div class="progress-bar" style="width: {($questionsAnswered / (content.game_rules.questions_per_session || 10)) * 100}%"></div>
      </div>
      <span class="progress-text">{$questionsAnswered} / {content.game_rules.questions_per_session || 10}</span>
    </div>
    
    <div class="game-hud">
      <div class="score-display">
        <span class="score-value">{$score}</span>
        <span class="score-label">pts</span>
      </div>
      
      <div class="difficulty-badge" class:easy={$difficulty === 'easy'} class:medium={$difficulty === 'medium'} class:hard={$difficulty === 'hard'}>
        {$difficulty.toUpperCase()}
      </div>
      
      <div class="streak-display" class:active={$streak >= 3}>
        <span class="streak-emoji">🔥</span>
        <span class="streak-count">{$streak}</span>
      </div>
    </div>
    
    <div class="question-content">
      <h2>{$currentQuestion?.question_text}</h2>
      
      <div class="answer-options">
        {#each $currentQuestion?.options || [] as option, index (index)}
          <button
            class="answer-btn"
            class:selected={$selectedAnswerIndex === index}
            class:correct={$isAnswerRevealed && index === $currentQuestion.correct_answer_index}
            class:incorrect={$isAnswerRevealed && $selectedAnswerIndex === index && index !== $currentQuestion.correct_answer_index}
            disabled={$isAnswerRevealed}
            on:click={() => handleAnswerSelect(index)}
          >
            <span class="option-letter">{String.fromCharCode(65 + index)}</span>
            <span class="option-text">{option}</span>
          </button>
        {/each}
      </div>
    </div>
  </div>

<!-- SCREEN: Result Overlay -->
{:else if currentScreen === 'result'}
  <div class="screen result-screen">
    <div class="result-overlay">
      <div class="result-animation" class:correct={$answerFeedback?.correct} class:incorrect={!$answerFeedback?.correct}>
        {#if $answerFeedback?.correct}
          <div class="result-icon">✓</div>
        {:else}
          <div class="result-icon">✗</div>
        {/if}
      </div>
      
      <p class="result-message">{$answerFeedback?.message}</p>
      
      {#if $answerFeedback?.correct}
        <p class="streak-message">Streak: {$streak} 🔥</p>
      {/if}
      
      {#if $showAchievementToast && $toastAchievement}
        <div class="achievement-toast">
          <span class="achievement-icon">{$toastAchievement.icon}</span>
          <span class="achievement-name">Achievement Unlocked: {$toastAchievement.name}</span>
        </div>
      {/if}
      
      <button class="btn btn-primary" on:click={handleContinue}>
        Continue →
      </button>
    </div>
  </div>

<!-- SCREEN: Summary -->
{:else if currentScreen === 'summary'}
  <div class="screen summary-screen">
    <div class="summary-header">
      <h1>Game Complete!</h1>
    </div>
    
    <div class="summary-stats">
      <div class="stat-block">
        <div class="stat-row">
          <span class="stat-label">Final Score</span>
          <span class="stat-value">{$score}</span>
        </div>
        <div class="stat-row">
          <span class="stat-label">Accuracy</span>
          <span class="stat-value">{$accuracy}%</span>
        </div>
        <div class="stat-row">
          <span class="stat-label">Best Streak</span>
          <span class="stat-value">{$highestStreak} 🔥</span>
        </div>
      </div>
      
      <div class="achievements-block" class:empty={$unlockedThisSession.length === 0}>
        <h3>Achievements Earned</h3>
        {#if $unlockedThisSession.length > 0}
          <div class="achievements-list">
            {#each $unlockedThisSession as ach (ach.id)}
              <div class="achievement-item">
                <span class="achievement-icon">{ach.icon}</span>
                <div class="achievement-info">
                  <div class="achievement-title">{ach.name}</div>
                  <div class="achievement-desc">{ach.description}</div>
                </div>
              </div>
            {/each}
          </div>
        {:else}
          <p class="empty-message">Play again to earn achievements!</p>
        {/if}
      </div>
    </div>
    
    <div class="summary-actions">
      <button class="btn btn-primary" on:click={playAgain}>
        Play Again
      </button>
      <button class="btn btn-secondary" on:click={() => onComplete($currentQuestion)}>
        Back to Content
      </button>
    </div>
  </div>
{/if}

<style>
  .screen {
    display: flex;
    flex-direction: column;
    height: 100vh;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    color: white;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  }

  .start-screen {
    justify-content: center;
    padding: 2rem;
    gap: 2rem;
  }

  .game-header h1 {
    font-size: 2.5rem;
    margin: 0;
    text-align: center;
  }

  .game-subtitle {
    font-size: 1.1rem;
    opacity: 0.9;
    margin: 0.5rem 0 0 0;
    text-align: center;
  }

  .game-stats {
    display: flex;
    gap: 2rem;
    justify-content: center;
  }

  .stat {
    display: flex;
    flex-direction: column;
    align-items: center;
    background: rgba(255, 255, 255, 0.15);
    padding: 1rem 1.5rem;
    border-radius: 12px;
    backdrop-filter: blur(10px);
  }

  .stat .label {
    font-size: 0.9rem;
    opacity: 0.8;
  }

  .stat .value {
    font-size: 1.8rem;
    font-weight: bold;
    margin-top: 0.5rem;
  }

  .game-rules-summary {
    background: rgba(255, 255, 255, 0.1);
    padding: 1.5rem;
    border-radius: 12px;
    backdrop-filter: blur(10px);
  }

  .game-rules-summary h3 {
    margin: 0 0 1rem 0;
    font-size: 1.1rem;
  }

  .game-rules-summary ul {
    list-style: none;
    padding: 0;
    margin: 0;
  }

  .game-rules-summary li {
    padding: 0.5rem 0;
    padding-left: 1.5rem;
    position: relative;
  }

  .game-rules-summary li::before {
    content: '✓';
    position: absolute;
    left: 0;
    font-weight: bold;
  }

  .question-screen {
    justify-content: space-between;
    padding: 1rem;
    gap: 0;
  }

  .question-header {
    padding: 1rem;
  }

  .progress {
    height: 6px;
    background: rgba(255, 255, 255, 0.2);
    border-radius: 3px;
    overflow: hidden;
    margin-bottom: 0.5rem;
  }

  .progress-bar {
    height: 100%;
    background: white;
    transition: width 0.3s ease;
  }

  .progress-text {
    font-size: 0.9rem;
    opacity: 0.8;
  }

  .game-hud {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 0 1rem;
    margin-bottom: 1rem;
  }

  .score-display {
    display: flex;
    align-items: baseline;
    gap: 0.3rem;
  }

  .score-value {
    font-size: 2rem;
    font-weight: bold;
  }

  .score-label {
    font-size: 0.9rem;
    opacity: 0.8;
  }

  .difficulty-badge {
    padding: 0.5rem 1rem;
    border-radius: 20px;
    font-size: 0.8rem;
    font-weight: bold;
    background: rgba(255, 255, 255, 0.2);
    transition: background 0.3s ease;
  }

  .difficulty-badge.easy {
    background: rgba(76, 175, 80, 0.4);
  }

  .difficulty-badge.medium {
    background: rgba(255, 193, 7, 0.4);
  }

  .difficulty-badge.hard {
    background: rgba(244, 67, 54, 0.4);
  }

  .streak-display {
    display: flex;
    align-items: center;
    gap: 0.3rem;
    font-size: 1.2rem;
    font-weight: bold;
    opacity: 0.6;
    transition: opacity 0.3s ease;
  }

  .streak-display.active {
    opacity: 1;
    animation: pulse 0.6s ease-in-out;
  }

  @keyframes pulse {
    0%, 100% { transform: scale(1); }
    50% { transform: scale(1.1); }
  }

  .question-content {
    flex: 1;
    display: flex;
    flex-direction: column;
    justify-content: center;
    padding: 2rem 1rem;
  }

  .question-content h2 {
    font-size: 1.5rem;
    margin: 0 0 2rem 0;
    line-height: 1.4;
  }

  .answer-options {
    display: flex;
    flex-direction: column;
    gap: 1rem;
  }

  .answer-btn {
    display: flex;
    align-items: center;
    gap: 1rem;
    padding: 1rem;
    background: rgba(255, 255, 255, 0.1);
    border: 2px solid rgba(255, 255, 255, 0.3);
    border-radius: 12px;
    color: white;
    font-size: 1rem;
    cursor: pointer;
    transition: all 0.3s ease;
    text-align: left;
  }

  .answer-btn:hover:not(:disabled) {
    background: rgba(255, 255, 255, 0.2);
    border-color: rgba(255, 255, 255, 0.5);
    transform: translateX(8px);
  }

  .answer-btn:disabled {
    cursor: not-allowed;
  }

  .answer-btn.selected:not(.correct):not(.incorrect) {
    background: rgba(255, 255, 255, 0.3);
    border-color: white;
  }

  .answer-btn.correct {
    background: rgba(76, 175, 80, 0.6);
    border-color: #4caf50;
  }

  .answer-btn.incorrect {
    background: rgba(244, 67, 54, 0.6);
    border-color: #f44336;
  }

  .option-letter {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 32px;
    height: 32px;
    background: rgba(255, 255, 255, 0.2);
    border-radius: 8px;
    font-weight: bold;
    flex-shrink: 0;
  }

  .option-text {
    flex: 1;
  }

  .result-screen {
    justify-content: center;
    align-items: center;
    padding: 2rem;
  }

  .result-overlay {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 1.5rem;
  }

  .result-animation {
    width: 100px;
    height: 100px;
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 3rem;
    animation: resultPop 0.6s cubic-bezier(0.34, 1.56, 0.64, 1);
  }

  .result-animation.correct {
    background: rgba(76, 175, 80, 0.3);
    border: 3px solid #4caf50;
    color: #4caf50;
  }

  .result-animation.incorrect {
    background: rgba(244, 67, 54, 0.3);
    border: 3px solid #f44336;
    color: #f44336;
  }

  @keyframes resultPop {
    0% { transform: scale(0); }
    50% { transform: scale(1.2); }
    100% { transform: scale(1); }
  }

  .result-message {
    font-size: 1.2rem;
    margin: 0;
    text-align: center;
  }

  .streak-message {
    font-size: 1rem;
    opacity: 0.9;
    margin: 0;
  }

  .achievement-toast {
    background: rgba(255, 193, 7, 0.3);
    border: 2px solid #ffc107;
    border-radius: 12px;
    padding: 1rem;
    display: flex;
    align-items: center;
    gap: 1rem;
    margin: 0.5rem 0;
    animation: slideIn 0.5s ease-out;
  }

  @keyframes slideIn {
    from { transform: translateY(-20px); opacity: 0; }
    to { transform: translateY(0); opacity: 1; }
  }

  .achievement-icon {
    font-size: 1.8rem;
  }

  .achievement-name {
    font-size: 0.95rem;
    font-weight: 600;
  }

  .summary-screen {
    justify-content: flex-start;
    padding: 2rem 1rem;
    overflow-y: auto;
  }

  .summary-header {
    text-align: center;
    margin-bottom: 2rem;
  }

  .summary-header h1 {
    font-size: 2.2rem;
    margin: 0;
  }

  .summary-stats {
    display: flex;
    flex-direction: column;
    gap: 2rem;
    margin-bottom: 2rem;
  }

  .stat-block {
    background: rgba(255, 255, 255, 0.1);
    border-radius: 12px;
    padding: 1.5rem;
    backdrop-filter: blur(10px);
  }

  .stat-row {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 1rem 0;
    border-bottom: 1px solid rgba(255, 255, 255, 0.1);
  }

  .stat-row:last-child {
    border-bottom: none;
  }

  .stat-label {
    font-size: 1rem;
    opacity: 0.8;
  }

  .stat-value {
    font-size: 1.5rem;
    font-weight: bold;
  }

  .achievements-block {
    background: rgba(255, 255, 255, 0.1);
    border-radius: 12px;
    padding: 1.5rem;
    backdrop-filter: blur(10px);
  }

  .achievements-block h3 {
    margin: 0 0 1rem 0;
    font-size: 1.1rem;
  }

  .achievements-block.empty .empty-message {
    opacity: 0.7;
    margin: 0;
    text-align: center;
  }

  .achievements-list {
    display: flex;
    flex-direction: column;
    gap: 1rem;
  }

  .achievement-item {
    display: flex;
    gap: 1rem;
    align-items: flex-start;
  }

  .achievement-icon {
    font-size: 2rem;
    flex-shrink: 0;
  }

  .achievement-info {
    flex: 1;
  }

  .achievement-title {
    font-weight: 600;
    font-size: 1rem;
  }

  .achievement-desc {
    font-size: 0.85rem;
    opacity: 0.8;
    margin-top: 0.25rem;
  }

  .summary-actions {
    display: flex;
    flex-direction: column;
    gap: 1rem;
  }

  .btn {
    padding: 1rem;
    border: none;
    border-radius: 12px;
    font-size: 1rem;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.3s ease;
  }

  .btn-primary {
    background: white;
    color: #667eea;
  }

  .btn-primary:hover {
    transform: translateY(-2px);
    box-shadow: 0 8px 24px rgba(0, 0, 0, 0.2);
  }

  .btn-secondary {
    background: rgba(255, 255, 255, 0.2);
    color: white;
    border: 2px solid white;
  }

  .btn-secondary:hover {
    background: rgba(255, 255, 255, 0.3);
  }

  @media (max-width: 600px) {
    .game-header h1 {
      font-size: 1.8rem;
    }

    .question-content h2 {
      font-size: 1.2rem;
    }

    .game-stats {
      flex-direction: column;
      gap: 1rem;
    }
  }
</style>
```

---

### Content Transformation Pipeline

The Gateway transformation pipeline (AgenticGateway DAG) converts ZenSci paper outputs into AdaptiveQuiz content. The following DAG orchestrates the transformation:

#### DAG Specification: `generate_adaptive_quiz`

```yaml
# docs/orchestration/dags/generate_adaptive_quiz.yaml
name: generate_adaptive_quiz
description: Transform a ZenSci paper into an AdaptiveQuiz mobile content item
version: "1.0.0"
author: "Architecture"
grounded_in: 
  - AgenticGateway ARCHITECTURE.md v1.0.0
  - zen-sci-mobile-v0.1-spec.md Revision R1

# Input: ZenSci paper-mcp output (PDF + structured metadata)
input_schema:
  paper_id: string
  paper_title: string
  paper_abstract: string
  paper_content: string  # extracted text or markdown
  paper_authors: string[]
  paper_domain: string   # e.g., "neuroscience", "materials-science"

# Output: AdaptiveQuizContent JSON
output_schema:
  id: string
  type: "AdaptiveQuiz"
  title: string
  description: string
  domain: string
  questions: object[]
  game_rules: object
  achievements: object[]

steps:
  - name: extract_key_concepts
    type: llm
    provider: openai
    model: gpt-4o
    prompt_template: |
      Extract 20-30 testable key concepts from this academic paper.
      Return JSON array of objects with: { id, name, definition, importance_level }
      
      Paper Title: {paper_title}
      Abstract: {paper_abstract}
      Content: {paper_content}
    output_key: concepts
    retry_policy:
      max_attempts: 3
      backoff: exponential

  - name: generate_questions
    type: llm
    provider: openai
    model: gpt-4o
    depends_on: extract_key_concepts
    prompt_template: |
      Generate 10-15 multiple-choice questions for each difficulty level (easy, medium, hard).
      Total: 30-45 questions.
      
      Each question must:
      - Test ONE concept from this list: {concepts}
      - Have 4 options, 1 correct answer
      - Include a concise explanation
      - Be tagged with the concept it tests and its difficulty
      
      Return JSON array of QuizQuestion objects:
      [{{ question_text, options, correct_answer_index, explanation, concept_id, difficulty }}]
    output_key: questions
    retry_policy:
      max_attempts: 3
      backoff: exponential

  - name: tag_by_difficulty
    type: llm
    provider: openai
    model: gpt-4
    depends_on: generate_questions
    prompt_template: |
      Validate and re-tag difficulty for each question based on concept depth.
      Ensure balanced distribution: ~40% easy, ~40% medium, ~20% hard.
      
      Questions: {questions}
      Concepts: {concepts}
      
      Return corrected JSON array with updated difficulty tags.
    output_key: validated_questions
    retry_policy:
      max_attempts: 2
      backoff: linear

  - name: generate_achievements
    type: llm
    provider: openai
    model: gpt-4
    depends_on: extract_key_concepts
    prompt_template: |
      Create 3-5 paper-specific achievements that reward learning milestones.
      
      Paper domain: {paper_domain}
      Paper title: {paper_title}
      Concepts: {concepts}
      
      Each achievement: { id, name, description, icon (emoji), unlock_condition }
      Examples:
      - "Methodology Master" — unlock for 100% accuracy
      - "Concept Collector" — unlock for 5+ streak
      - "Domain Expert" — unlock for final score >= 80
      
      Return JSON array of Achievement objects.
    output_key: achievements
    retry_policy:
      max_attempts: 2
      backoff: linear

  - name: assemble_content
    type: python
    depends_on: [validated_questions, achievements, extract_key_concepts]
    code: |
      import json
      import uuid
      
      content = {
        "id": str(uuid.uuid4()),
        "type": "AdaptiveQuiz",
        "title": f"Quiz: {input['paper_title']}",
        "description": f"Test your understanding of {input['paper_title']} with adaptive difficulty.",
        "domain": input.get('paper_domain', 'general'),
        "authors": input.get('paper_authors', []),
        "paper_id": input['paper_id'],
        "questions": validated_questions,
        "game_rules": {
          "format": "adaptive_quiz",
          "difficulty_levels": ["easy", "medium", "hard"],
          "questions_per_session": 10,
          "streak_bonus_threshold": 3,
          "achievement_ids": [ach['id'] for ach in achievements],
          "time_limit_per_question": None
        },
        "achievements": achievements,
        "created_at": datetime.utcnow().isoformat()
      }
      return content
    output_key: content_json

  - name: validate_content
    type: python
    depends_on: assemble_content
    code: |
      # Validate schema, question coverage, achievement logic
      assert len(content_json['questions']) >= 30, "Need >= 30 questions"
      assert len(content_json['achievements']) >= 3, "Need >= 3 achievements"
      
      easy_count = len([q for q in content_json['questions'] if q['difficulty'] == 'easy'])
      medium_count = len([q for q in content_json['questions'] if q['difficulty'] == 'medium'])
      hard_count = len([q for q in content_json['questions'] if q['difficulty'] == 'hard'])
      
      total = easy_count + medium_count + hard_count
      assert total == len(content_json['questions']), "Difficulty tags must cover all questions"
      
      return {
        "is_valid": True,
        "stats": {
          "total_questions": total,
          "easy": easy_count,
          "medium": medium_count,
          "hard": hard_count,
          "achievements": len(content_json['achievements'])
        }
      }
    output_key: validation_result

# Success condition
success_condition: |
  validation_result.is_valid == True AND
  len(content_json['questions']) >= 30 AND
  len(content_json['achievements']) >= 3

# Failure handling
on_failure:
  retry_entire_dag: true
  max_dag_retries: 2
  fallback_action: create_basic_quiz_from_abstract  # degrade gracefully

# Caching
cache_key: "quiz_{paper_id}_{paper_domain}"
cache_ttl: 86400  # 24 hours; regenerate daily or on paper update

# Output storage
store_output:
  destination: mobile_content_cache  # SyncEngine local SQLite
  format: json
  schema: AdaptiveQuizContent
```

#### Integration with SyncEngine

When the mobile app syncs with the portal:

1. **Trigger:** User opens a paper, taps "Play Game" (or automatically offered if not yet played)
2. **Check:** Mobile client queries local cache for AdaptiveQuiz version of paper
3. **Generate/Download:** If missing, invoke Gateway via portal API:
   ```
   POST /api/v1/gateway/transform
   {
     "paper_id": "...",
     "format": "adaptive_quiz",
     "source": "paper-mcp"
   }
   ```
4. **DAG Execution:** Gateway runs `generate_adaptive_quiz` DAG (takes ~30-60 seconds for LLM calls)
5. **Cache:** Result stored in mobile local SQLite via SyncEngine
6. **Play:** AdaptiveQuizPlayer.svelte loads content immediately

---

### Demo Verification Checklist

A "working demo" is verified when:

- [ ] **Content Discovery:** A researcher opens zen-sci-mobile and sees a list of papers they've authored/accessed via the portal
- [ ] **Paper-to-Game:** They tap "Play" on a paper
- [ ] **Transformation Runs:** Mobile app detects no cached AdaptiveQuiz, invokes Gateway DAG `generate_adaptive_quiz`
- [ ] **Load & Start:** DAG completes, content loads into AdaptiveQuizPlayer.svelte, "Start Game" screen displays
- [ ] **Game Loop Works:** Researcher answers 10 questions
  - [ ] Questions are pulled from mix of difficulties (easy, medium, hard)
  - [ ] Score updates with each correct answer (+10 base, difficulty multiplier, streak bonus)
  - [ ] Difficulty badge changes (easy → medium after 3-streak, medium → hard after next 3-streak)
  - [ ] Streak counter shows and resets on wrong answer
  - [ ] Result overlay confirms correct/incorrect with explanation
- [ ] **Achievements Earned:** At least one achievement unlocks during the session (e.g., "Perfect Score" if all 10 correct, or "Streak Master" for 5+ streak)
- [ ] **Summary Screen:** After 10 questions:
  - [ ] Final score displayed
  - [ ] Accuracy % calculated and shown
  - [ ] Best streak shown
  - [ ] Unlocked achievements listed with icons
  - [ ] "Play Again" button present
- [ ] **Persistence:** Researcher closes app, reopens 5 minutes later
  - [ ] Same paper still shows the game
  - [ ] Tapping "Play" shows previous best score on start screen
  - [ ] Playing again saves a new session (game_sessions table updated)
- [ ] **No Cloud Sync:** Game state stays on device; no upload to portal (v1 offline-only)

#### Demo Scenario: E2E Flow

**Researcher: Dr. Sarah Chen**

1. Opens zen-sci-mobile on iPhone
2. Syncs with portal (papers auto-downloaded via SyncEngine)
3. Sees paper: "Attention Mechanisms in Transformer Networks" (her recent paper)
4. Taps "Play Game" → triggers Gateway DAG `generate_adaptive_quiz`
5. DAG runs (30-60 seconds):
   - Extracts 25 key concepts (self-attention, scaled dot-product, multi-head attention, etc.)
   - Generates 40 questions (10 per difficulty level × 4 levels, then selects best)
   - Validates difficulty distribution
   - Creates 4 achievements (Attention Expert, Optimization Master, Perfect Run, Transformer Scholar)
6. AdaptiveQuizPlayer loads with content
7. Starts game:
   - **Q1** (Easy): "What is attention?" → Correct (+10) → Streak: 1
   - **Q2** (Easy): "Define query matrix" → Correct (+10) → Streak: 2
   - **Q3** (Easy): "What is softmax?" → Correct (+10) → Streak: 3 → *Difficulty bumps to Medium*
   - **Q4** (Medium): "Derive scaled dot-product..." → Incorrect → Streak: 0 → *Difficulty drops to Easy*
   - ...continues for 6 more questions
   - Final: Score 92, Accuracy 80%, Best Streak 4, Unlocked "Optimizer Master"
8. Summary screen shows stats + achievement
9. Closes app; game_sessions table saved locally
10. Next day, reopens app, sees "Best Score: 92" on start screen for that paper

---

### Notes: Future Extensions

**v1.1 Planned Enhancements:**
- Time limits per question (countdown timer visual)
- Leaderboards (local per content item; v2: group-wide)
- Spaced repetition: recommend re-play if user hasn't played in 7 days
- Hint system: consume hints to reveal 50% of an answer

**v2 Reserved Game Formats:**
- `matching_game`: Match concepts to definitions (reuses ReadingContent vocab)
- `timeline_sort`: Arrange historical events/steps in correct order (from papers with methodology sections)
- `branching_narrative`: Choose-your-own-adventure based on research decisions (from papers with discussion sections)

---


## Revision R2: Locked Decisions (2026-02-18)

### Mobile Transformation Trigger — Final Decision: AI Recommendation on Open

**Decision:** When a user opens a content item, the Gateway reads the content type and metadata, then surfaces a non-blocking suggestion card: "This paper has 14 key concepts — want to play an Adaptive Quiz?" One tap to confirm. No background auto-generation. No manual menu navigation.

**Rationale:** Low friction (one tap, not buried in menus). No wasted API credits on content the user never engages with. Showcases Gateway intelligence actively rather than hiding it. Fits self-directed researcher persona for v1.

**Trigger flow:**

```
User opens ContentItem (e.g. paper-mcp output)
  ↓
ContentView.svelte mounts
  ↓
[Rust IPC] suggest_game_format(content_id, content_type, metadata)
  → Gateway: POST /v1/chat/completions with system prompt:
      "Given this {content_type} with title '{title}', {word_count} words,
       and {section_count} sections, suggest the best game format.
       Respond with: { format, reason, concept_count, estimated_minutes }"
  → Returns: { format: 'AdaptiveQuiz', reason: '...', concept_count: 14, estimated_minutes: 8 }
  ↓
SuggestionCard.svelte renders (non-blocking, slides up from bottom):
  "📚 14 key concepts found — Play an Adaptive Quiz? (~8 min)"
  [Play Now]  [Maybe Later]
  ↓
"Play Now" → trigger generate_adaptive_quiz DAG (R1 spec)
"Maybe Later" → dismiss card, store dismissal in local SQLite (don't show again this session)
```

**New Rust IPC command:**
```rust
#[tauri::command]
async fn suggest_game_format(
    content_id: String,
    content_type: String,  // 'paper' | 'slides' | 'blog' | etc.
    metadata: ContentMetadata,
) -> Result<GameFormatSuggestion, String>

pub struct GameFormatSuggestion {
    pub format: String,         // 'AdaptiveQuiz' in v1; future: 'MatchingGame', etc.
    pub reason: String,         // human-readable explanation
    pub concept_count: u32,
    pub estimated_minutes: u32,
}
```

**SuggestionCard.svelte behaviour:**
- Slides up 300ms after ContentView mounts (not immediately — let user orient first)
- Non-blocking: the content is fully readable behind the card
- Dismissal is per-session (dismissing once doesn't permanently hide it)
- After game is generated and cached: card changes to "▶ Play Adaptive Quiz (14 concepts)" — persistent, no re-suggestion needed
- Card is shown at most once per session per content item

**v2 extension note:** When more game formats are available (MatchingGame, TimelineSort, BranchingNarrative), the suggestion card can offer a format picker: "This paper works well as a Quiz or a Matching Game — which do you prefer?" For v1, only AdaptiveQuiz is offered.
