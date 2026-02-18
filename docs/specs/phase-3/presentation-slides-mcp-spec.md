# presentation-slides-mcp Spec: Business Presentation & Slide Deck Generation

**Author:** ZenSci Architecture (TresPiesDesign.com)
**Status:** Draft — Ship on Demand
**Created:** 2026-02-18
**Phase:** 3 (lower priority; trigger-based shipping)
**Grounded In:** Module strategy scout (2026-02-17)

---

## 1. Vision

Transform structured markdown outlines into polished, presenter-ready slide decks (.pptx and Google Slides JSON) with consistent theme application, speaker notes, animation hints, and build-out sequences—enabling non-designers to author professional presentations in minutes without touching PowerPoint.

### The Core Insight

Presentation design is one of the most time-consuming yet commoditized tasks in business: outlines exist, but converting them to visually coherent slides requires design knowledge. Beamer (covered by beamer-mcp, Phase 1) targets academic audiences. Business presenters and product managers use PowerPoint or Google Slides, but lack a markdown-first authoring pipeline. A ZenSci module bridges this gap: structure talks as markdown → apply professional theme → export .pptx or Google Slides JSON → presenter with speaker notes and animations.

### What Makes This Different

Unlike **beamer-mcp** (which uses LaTeX and targets academics), presentation-slides-mcp generates *business-grade* PPTX and Google Slides, with built-in theming, speaker notes per slide, and animation sequences. The module treats slides as data (slide type + content structure) rather than free-form HTML/LaTeX, enabling consistent layout and professional polish without manual tweaking.

---

## 2. Goals & Success Criteria

### Primary Goals

1. **Enable rapid deck authoring** — markdown outline → polished .pptx or Google Slides in <120 seconds
2. **Professional-grade output without design overhead** — consistent theme, typography, color hierarchy, animation
3. **Support multiple slide types** — title slide, content + bullet points, two-column layouts, image + caption, data tables, quote callouts

### Success Criteria

✅ Can convert 80% of real business presentations (20 sample decks) into .pptx without manual editing
✅ Speaker notes integrate correctly into PowerPoint; presenter view works as expected
✅ Slide animations and build sequences work in PowerPoint 2019+ and Google Slides
✅ Theme consistency scores 9/10 across diverse presentation types (product pitch, investor update, all-hands meeting)

### Non-Goals

❌ Design algorithm generation (humans design; ZenSci applies designs)
❌ Animated GIF or video embedding in PPTX (static slide focus)
❌ Speaker voice integration or AI voiceover (separate domain)
❌ Real-time collaborative editing (Google Slides + Docs already own this)

---

## 3. Strategic Fit Assessment

**Why Phase 3 (not Phase 1 or 2):**
Presentations are a massive market, but ZenSci's core strengths—markdown parsing, citation management, mathematical rendering—are *secondary* in business presentations. Unlike research papers or lab notebooks, slides prioritize visual design and persuasion over citation rigor or formula correctness. Additionally, generating PPTX requires python-pptx (new dependency, different from Pandoc) and rendering to Google Slides requires their API (external service dependency), adding implementation complexity without proven user demand.

**Market Trigger Conditions:**

- **Trigger 1:** A SaaS startup or corporate product team explicitly requests presentation-slides-mcp and commits to using it for 10+ internal presentations
- **Trigger 2:** GitHub issues/email requests for slide generation exceed 10+ over 3 months from non-academic users
- **Trigger 3:** A pilot with corporate partner shows ≥60% time savings vs. manual PowerPoint creation, with acceptable design quality

**Risk of NOT shipping:**
**Low-to-Medium.** Presentations are well-served by existing tools (Slidebean, Beautiful.ai, Gamma.app). However, integrating with ZenSci's research + presentation story (e.g., "research paper → slide deck") could be valuable for academic and corporate research teams. If we don't ship, we lose a potential adjacent market.

---

## 4. Technical Architecture (simplified)

### 4.1 MCP Tools

**`convertToSlides`**
- **Input:** `DocumentRequest` (source markdown, output format: 'pptx' | 'google-slides')
- **Output:** `DocumentResponse` (PPTX binary or Google Slides JSON representation + metadata)
- **Processing:** Parse slide structure → detect slide types → apply theme → render speaker notes → export PPTX/JSON

**`validateSlideDeck`**
- **Input:** markdown source
- **Output:** `ValidationResult` with slide count, missing title checks, image path validation, speaker notes length

**`applyTheme`**
- **Input:** raw slide deck + theme name
- **Output:** deck with applied color scheme, font stack, master slide layouts
- **Processing:** Validates theme exists; applies colors, fonts, and layout masters

**`generateSpeakerNotes`**
- **Input:** slide content + optional extended notes
- **Output:** formatted speaker notes (one per slide, with timing suggestions)

### 4.2 Infrastructure Reuse

- **MarkdownParser** — parse slide structure and metadata
- **SchemaValidator** — validate `DocumentRequest` against slide schema
- **Pandoc Python engine** — optional: Pandoc's native PPTX support (if better than python-pptx)

### 4.3 Novel Infrastructure Required

**Slide Deck Schema (`SlideDeckRequest`)**
```typescript
interface SlideDeckRequest extends DocumentRequest {
  format: 'pptx' | 'google-slides';
  moduleOptions: {
    theme: 'default' | 'dark' | 'vibrant' | 'minimal' | 'corporate';
    aspectRatio: '16:9' | '4:3';
    includeSpeakerNotes: boolean;
    animations: boolean;
    transitionDuration?: number; // milliseconds
    customFonts?: { heading: string; body: string };
    logoUrl?: string; // for slide master
  };
}
```

**Slide Type Definitions** — markdown syntax for each slide type
```
# Title Slide
## Subtitle Slide (auto-layout: title + subtitle)

---

## Content Slide
- Bullet 1
- Bullet 2
- Bullet 3

---

## Two-Column Layout
| **Left Column**     | **Right Column**     |
|--------------------|-------------------|
| Content here       | Content here       |

---

## Image + Caption
![](image.png)
Caption text

---

## Quote Callout
> "Quote text with attribution"
— Speaker Name

---

## Data Table
| Header 1 | Header 2 | Header 3 |
|----------|----------|----------|
| Data     | Data     | Data     |
```

**PPTX Renderer** — python-pptx adapter for PPTX generation

**Google Slides JSON Adapter** — output Google Slides compatible JSON (presentation.json format)

### 4.4 Key Schemas

```typescript
type SlideType =
  | 'title'
  | 'content'
  | 'two-column'
  | 'image'
  | 'quote'
  | 'table'
  | 'blank'
  | 'section-header';

interface Slide {
  type: SlideType;
  title?: string;
  content: SlideContent;
  layout: 'default' | 'no-title' | 'custom';
  speakerNotes?: string;
  animations?: SlideAnimation[];
  transition?: {
    type: 'fade' | 'slide' | 'push' | 'wipe';
    duration: number; // milliseconds
  };
}

type SlideContent =
  | { type: 'text'; bullets: string[] }
  | { type: 'two-column'; left: SlideContent; right: SlideContent }
  | { type: 'image'; src: string; caption?: string }
  | { type: 'quote'; text: string; attribution: string }
  | { type: 'table'; headers: string[]; rows: string[][] }
  | { type: 'blank' };

interface SlideAnimation {
  element: string; // CSS selector or element ID
  type: 'fade' | 'slide' | 'scale' | 'appear';
  duration: number; // milliseconds
  delay?: number; // milliseconds
}

interface PresentationTheme {
  name: string;
  colors: {
    primary: string;
    secondary: string;
    accent: string;
    text: string;
    background: string;
  };
  fonts: {
    heading: string;
    body: string;
  };
  slide: {
    width: number;
    height: number;
    marginTop: number;
    marginBottom: number;
    marginLeft: number;
    marginRight: number;
  };
}
```

---

## 5. Implementation Estimate

| Track | Duration (weeks) | Dependencies |
|-------|---------|-------------|
| Slide syntax + parser | 1 | MarkdownParser |
| Slide type detection & layout | 1.5 | Parser; schema design |
| PPTX renderer (python-pptx) | 2 | python-pptx library; theme system |
| Google Slides JSON adapter | 1.5 | Google Slides API; theme system |
| Theme system & built-in themes (5x) | 1.5 | Design review; color + font specifications |
| Speaker notes + animations | 1 | PPTX/Slides API integration |
| MCP tool registration & testing | 1 | packages/sdk |
| **Total** | **10** | New: python-pptx, Google Slides API client; New design work |

---

## 6. Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| PPTX output quality/compatibility issues with Microsoft Office versions | Medium | High | Test early with Office 2019, Office 365, PowerPoint Online; validate with 10+ corporate machines |
| Google Slides JSON format changes or API limitations | Medium | High | Investigate Google Slides JSON export format early; assess API constraints before committing |
| Slide animation rendering inconsistent across PowerPoint vs. Google Slides | Medium | Medium | Test animations on both platforms; document limitations; provide fallback (static slides) |
| Design theme quality too generic for professional use | Medium | Medium | Hire or partner with designer for 2–3 professional themes before shipping; conduct design review with 3 external presenters |
| Implementation timeline exceeds 10 weeks due to design iteration | High | Medium | Allocate week 0 for design specification + theme mockups; lock designs by end of week 2 |
| Presentation market too commoditized; low adoption | Medium | Low | This is acceptable risk for Phase 3. Monitor demand; deprioritize if triggers don't align. |

---

## 7. Open Questions & Future Considerations

1. **Animated transitions & reveals:** Should we support more sophisticated animations (e.g., staggered bullet point reveals, animated data chart build-outs)?

2. **Custom theme editor:** Should users be able to customize colors/fonts in the markdown frontmatter, or force theme selection from pre-defined options?

3. **Speaker timer integration:** Could we embed a speaker timer in PowerPoint presenter notes that counts down to slide time limits?

4. **Collaborative variants:** Should presentation-slides-mcp auto-export to shared Google Drive folder, enabling team review before presentation?

5. **Video & GIF embedding:** Should we support embedding short videos or GIFs in slides (requires PPTX 2016+ or Google Slides)?

6. **PDF handout generation:** Should the module auto-generate PDF handouts (multiple slides per page, speaker notes) for distribution to audience?

7. **Presenter view optimization:** Should we auto-generate presenter view with notes, next slide preview, and timer?

8. **Localization & multi-language slides:** Should presentation-slides-mcp support multi-language decks with language-specific fonts and RTL support?

---

## Audit Notes (2026-02-18)

**Auditor:** Claude (Opus-level audit)
**Status:** Reviewed

### Issues
- **HIGH-1:** Tool names use camelCase (`convertToSlides`, `validateSlideDeck`, `applyTheme`, `generateSpeakerNotes`). Should be snake_case: `convert_to_slides`, `validate_slide_deck`, `apply_theme`, `generate_speaker_notes`.
- **ADVISORY-1:** Naming overlap with Phase 2 `slides-mcp`. This module does PPTX/Google Slides (business); `slides-mcp` does Beamer/Reveal.js (academic). Consider renaming to `pptx-mcp` for clarity.
- **ADVISORY-2:** Uses `moduleOptions` field.
- Depends on python-pptx (new dependency) and Google Slides API (external service).
