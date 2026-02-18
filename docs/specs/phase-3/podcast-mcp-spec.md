# podcast-mcp Spec: Podcast Transcript & Show Notes Publishing

**Author:** ZenSci Architecture (TresPiesDesign.com)
**Status:** Draft — Ship on Demand
**Created:** 2026-02-18
**Phase:** 3 (lower priority; trigger-based shipping)
**Grounded In:** Module strategy scout (2026-02-17)

---

## 1. Vision

Transform raw podcast transcripts into publication-ready show notes (markdown & HTML), episode metadata feeds (RSS/Atom), and timestamped citation indexes—enabling podcast producers to ship polished episode documentation within minutes of upload, while listeners gain searchable, citable, and accessible transcript archives.

### The Core Insight

Podcasts are a dominant research and knowledge-sharing medium, but most shows ship with minimal documentation. Listeners face friction: no searchable transcripts, no timestamped speaker identification, no citations for referenced papers or links. Meanwhile, producers lack incentive to invest in documentation given manual effort required. A ZenSci module bridges this gap: ingest raw transcript (from Rev, Otter.ai, or manual editing) → auto-structure with speaker labels and timestamps → extract citations and links → publish show notes (HTML + markdown) + RSS metadata + transcript archive.

### What Makes This Different

Unlike blog-mcp (which handles written essays), podcast-mcp owns the *audio artifact → documentation pipeline*. It parses speaker labels from transcript format (e.g., `[00:05:23] Dr. Alice: "..."`), extracts and validates external links, auto-detects academic citations, and generates time-indexed show notes for seekability. The output isn't just pretty HTML—it's *accessible and machine-readable* (RSS feeds, timestamps, BibTeX export for cited papers).

---

## 2. Goals & Success Criteria

### Primary Goals

1. **Enable rapid show notes generation** — raw transcript → structured show notes + RSS in <60 seconds
2. **Make podcasts accessible and citable** — searchable transcripts, speaker attribution, timestamp links, citation export
3. **Support multi-format output** — HTML (web-ready), markdown (blog integration), RSS feed (distribution), plain text transcript (accessibility)

### Success Criteria

✅ Can convert 85% of real podcast transcripts (80 samples from various speakers/formats) with correct speaker attribution and timestamps
✅ Auto-extracted links and citations are 90%+ accurate with <5% false positives
✅ Generated RSS feed validates against RSS 2.0 spec; timestamps link correctly to web player
✅ Show notes HTML passes WCAG 2.1 Level AA accessibility testing

### Non-Goals

❌ Real-time transcription (integrate with Rev, Otter.ai, or Descript for audio input)
❌ Audio editing or mixing (external tools own that)
❌ Video show notes (audio-first; video is future variant)
❌ Ad insertion or sponsorship block detection (out of scope)

---

## 3. Strategic Fit Assessment

**Why Phase 3 (not Phase 1 or 2):**
While podcasting is a large market, ZenSci's core value—precision document conversion, citation management, mathematical/scientific formatting—has *limited applicability* to typical podcast workflows. Podcast documentation is primarily text extraction + link validation, not mathematical rendering or complex citation management. This module is valuable only if we partner with *research-focused podcasts* (e.g., scientific interview shows, academic discussion pods) where citation extraction and transcript searchability genuinely reduce friction. Without that partner validation, this is speculative.

**Market Trigger Conditions:**

- **Trigger 1:** A research-focused podcast (e.g., "Nature Podcast," "The Ezra Klein Show," "Lex Fridman Podcast," or academic interview series) partners with ZenSci and requests podcast-mcp as part of documentation infrastructure
- **Trigger 2:** Podcast producer requests (GitHub issues or email) exceed 5+ over 3 months, all citing transcript searchability + citation export as core pain points
- **Trigger 3:** Pilot with research podcast shows >70% reduction in show-notes creation time vs. manual workflow

**Risk of NOT shipping:**
**Low.** Podcasting is well-served by existing platforms (Podbean, Transistor, Buzzsprout) and transcription services (Rev, Descript). If we don't ship, ZenSci loses a niche market but faces no existential risk. If we do ship without partner validation, the module languishes unmaintained.

---

## 4. Technical Architecture (simplified)

### 4.1 MCP Tools

**`convertPodcastTranscript`**
- **Input:** `DocumentRequest` (source transcript markdown or plaintext, output format: 'podcast-notes')
- **Output:** `DocumentResponse` (HTML show notes + markdown + RSS feed metadata)
- **Processing:** Parse speaker labels and timestamps → validate format → extract links and citations → structure markdown → render HTML + RSS

**`generateRSSFeed`**
- **Input:** episode metadata (title, description, date, duration, transcript URL) + array of show-notes documents
- **Output:** RSS 2.0 XML feed with episode entries, timestamps, and transcript links

**`extractCitationsFromTranscript`**
- **Input:** transcript text
- **Output:** array of detected citations (mentioned papers, books, URLs with speaker + timestamp)

**`validateTranscriptLinks`**
- **Input:** transcript with `[link](url)` or mentioned URLs
- **Output:** `ValidationResult` with link status, reachability, and warnings for dead links

### 4.2 Infrastructure Reuse

- **MarkdownParser** — parse transcript format (speaker labels, timestamps, section headers)
- **LinkChecker** — async validation of referenced URLs; detection of dead links
- **CitationManager** — optional: if podcast mentions academic papers, extract and validate BibTeX references
- **blog-mcp pipeline** — reuse HTML templating and RSS generation logic

### 4.3 Novel Infrastructure Required

**Podcast Transcript Schema (`PodcastTranscriptRequest`)**
```typescript
interface PodcastTranscriptRequest extends DocumentRequest {
  format: 'podcast-notes';
  moduleOptions: {
    podcastTitle: string;
    podcastDescription?: string;
    episodeNumber?: string;
    episodeTitle: string;
    episodeDate: string; // YYYY-MM-DD
    episodeDuration?: number; // seconds
    speakers: { name: string; title?: string; affiliation?: string }[];
    outputFormats: ('html' | 'markdown' | 'rss' | 'transcript')[];
    includeTimestamps: boolean;
    extractCitations: boolean;
  };
}
```

**Transcript Parser for Speaker Labels & Timestamps**
```typescript
interface TranscriptSegment {
  speaker: string;
  timestamp: number; // seconds from start
  text: string;
  citations?: CitationRecord[];
  mentions?: string[]; // detected links or people
}
```

**RSS Feed Builder** — extend blog-mcp's RSS generation to support podcast-specific fields (duration, transcript link, speaker list)

### 4.4 Key Schemas

```typescript
// Extends DocumentNode union
type PodcastNode = DocumentNode | {
  type: 'transcript-segment';
  speaker: string;
  timestamp: number;
  children: InlineNode[];
} | {
  type: 'timestamp-link';
  timestamp: number;
  text: string;
};

interface PodcastEpisodeMetadata extends ResponseMetadata {
  podcastTitle: string;
  episodeNumber?: string;
  episodeTitle: string;
  episodeDate: string;
  duration: number; // seconds
  speakers: { name: string; title?: string; affiliation?: string }[];
  citationsCount: number;
  linksCount: number;
}

interface RSSPodcastItem {
  title: string;
  description: string;
  pubDate: string; // RFC 822
  guid: string;
  link: string; // to episode page
  enclosure?: { url: string; type: string; length: number }; // for audio
  content?: { type: 'text/html' | 'text/plain'; value: string };
  duration?: string;
  speakers: string[];
}
```

---

## 5. Implementation Estimate

| Track | Duration (weeks) | Dependencies |
|-------|---------|-------------|
| Transcript parser (speaker labels, timestamps) | 1.5 | MarkdownParser |
| Citation + link extraction | 1.5 | LinkChecker; CitationManager (optional) |
| HTML show notes generator | 1 | blog-mcp HTML template reuse |
| RSS feed builder | 1 | blog-mcp RSS logic; podcast metadata |
| Markdown + plaintext output formatters | 0.5 | MarkdownParser |
| MCP tool registration & testing | 1 | packages/sdk |
| **Total** | **7** | blog-mcp (HTML + RSS templates); LinkChecker |

---

## 6. Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| Transcript parser fails on non-standard speaker formats | Medium | Medium | Test with 50+ real transcripts from various sources (Rev, Descript, manual); support common formats + fallback heuristic |
| Citation extraction produces false positives (e.g., "Smith et al." in conversation) | High | Medium | Use conservative heuristic; mark auto-detected citations as "unverified" and require editor confirmation |
| RSS feed timestamps don't align with actual audio player | Medium | Medium | Test RSS feed with common podcast players; ensure timestamp format matches player expectations |
| Podcast domain too niche; insufficient demand | Medium | Medium | This is acceptable risk for Phase 3. Monitor GitHub issues; ship only if triggers align. |
| Accessibility compliance gap in generated HTML | Low | Medium | Use blog-mcp's accessibility checklist; run generated HTML through WAVE or Axe tools |

---

## 7. Open Questions & Future Considerations

1. **Real-time transcription integration:** Should podcast-mcp accept raw audio + call Otter.ai/Rev API for transcription, or stay transcript-only (let producers use external transcription)?

2. **Video variants:** Should we support video podcasts (YouTube, Vimeo) with transcript + chapters, or stay audio-only for Phase 3?

3. **Multi-language support:** Many academic podcasts feature non-English speakers. Should podcast-mcp auto-detect language and apply localization?

4. **Timestamp clip generation:** Could we auto-generate "best moment" clips (short 30–60 second segments) for social media based on text relevance?

5. **Speaker guest directory:** Should podcast-mcp build a searchable index of all guests across episodes for easy lookup?

6. **Integration with podcast hosting:** Could we auto-publish generated show notes directly to Podbean, Transistor, or Apple Podcasts (if they support custom HTML)?

7. **Bonus feature: Audience Q&A threading:** Should podcast-mcp parse and structure listener questions from YouTube/Twitter mentions and link them to relevant transcript timestamps?

---

## Audit Notes (2026-02-18)

**Auditor:** Claude (Opus-level audit)
**Status:** Reviewed

### Issues
- **HIGH-1:** Tool names use camelCase (`convertPodcastTranscript`, `generateRSSFeed`, `extractCitationsFromTranscript`, `validateTranscriptLinks`). Should be snake_case: `convert_podcast_transcript`, `generate_rss_feed`, `extract_citations_from_transcript`, `validate_transcript_links`.
- **ADVISORY-2:** Uses `moduleOptions` field.
- Strategic Fit Assessment correctly gates behind research-focused podcast partnerships.
- Most creative Phase 3 spec — novel territory for ZenSci.
