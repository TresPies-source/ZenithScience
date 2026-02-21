# Strategic Scout: zen-sci-mobile v0.1 — Games Specification

**Date:** 2026-02-18
**Status:** Strategic Scouting (No Code)
**Audience:** Cruz Morales, Product Strategy, Architecture
**Grounded In:** zen-sci-mobile-v0.1-spec.md, 2026-02-18_scout2_zen-sci-mobile.md, STATUS.md

---

## Executive Summary

The prior scout identified **Tension 3: The "Games" Gap** — games are mentioned in Cruz's original vision but absent from the spec. This scout clarifies what "games" means for academic knowledge work on mobile, identifies 4 distinct game format families that work with ZenSci content, and defines what a "working game demo" requires concretely.

**Key finding:** Games are not a separate content type; they are a *runtime-generated transformation* of existing content types (quiz, lesson, reading). A working demo can be built in v0.1 using the existing QuizQuestion schema augmented with game mechanics (scoring, progression, time pressure, adaptive difficulty). The demo is NOT just a quiz—it's a quiz with game loop properties (challenge, feedback, reward).

**Core reframe:** "Should games be pre-rendered at publish time (like quizzes today) or generated at runtime when a student opens content?" Answer: **Start with compile-time games (simpler), evolve to runtime games in v1 if AI-driven personalization justifies the cost.**

---

## Part 1: Game Formats for Academic Knowledge Work

### What Makes a Game Different from a Quiz?

**Quiz (current spec):**
- Linear question sequence
- One attempt per question or cumulative scoring
- Static difficulty
- Goal: assess comprehension
- Feedback: right/wrong + explanation

**Game (proposed):**
- Interactive challenge with a *loop* (attempt → feedback → improvement opportunity)
- Dynamic difficulty (gets harder/easier based on performance)
- Time pressure optional but often present
- Goal: *engage* while assessing
- Feedback: immediate reward (points, streak, achievement)
- State: persistent (score, level, progress saved across sessions)

---

### Scout 1: Four Game Format Families for Academic Content

#### Format A: **Concept Matching Games** (Drag-Drop, Sort, Connect)

*Best for:* Classification, taxonomy, relationship mapping
*Academic contexts:* Biology (taxonomy trees), chemistry (molecular structures), history (timeline events), linguistics (grammar classification)

**Example from ZenSci:**
- **Source:** Paper on phylogenetic taxonomy
- **Game:** Drag-and-drop organism cards into the correct kingdom/phylum/class hierarchy
- **Mechanics:** 10 organisms, 8 possible slots, 3 lives, score = time_bonus + accuracy_bonus
- **Feedback loop:** Miss one → explanation of the relationship → retry

**What it requires:**
- Source content must have *structured data* (term-definition pairs, parent-child relationships)
- Schema: `ConceptMatchingGame { concepts: Concept[], answerSlots: Slot[], timeLimit: number, livesPerRound: number }`
- Generation: AI extracts concepts from paper sections + relationship graph → generates random shuffles of correct/plausible answers
- Rendering: DOM-based drag-drop in SvelteKit (standard Svelte Kit events)

**Difficulty:** Medium. Requires NLP to extract concepts and relationships. Can be AI-generated at publish time.

**Feasibility in Tauri v2 mobile:** High. Touch drag-drop is well-supported on iOS/Android. Performance cost: ~50MB rendering a few hundred DOM elements.

---

#### Format B: **Adaptive Quiz Games** (Branching, Leveling, Streaks)

*Best for:* Knowledge testing with dynamic difficulty
*Academic contexts:* Any comprehension assessment (paper reading, grant understanding, concept recall)

**Example from ZenSci:**
- **Source:** Paper with 20 extracted quiz questions (easy/medium/hard)
- **Game:** Start on easy questions. Correct answer → progress to harder question (branching). Wrong answer → repeat on medium difficulty + hint. Streak counter: how many correct in a row?
- **Mechanics:** Cumulative score, level progression (Level 1→10), streaks, achievement badges (e.g., "Perfect 5-Streak")
- **Feedback loop:** Answer → immediate score + visual feedback (green checkmark, +10 points) → next question auto-advances or waits for tap

**What it requires:**
- Quiz questions must be pre-tagged with difficulty (existing schema: `QuizQuestion.difficulty`)
- Schema: `AdaptiveQuizGame { questions: QuizQuestion[], adaptiveMode: boolean, streakCounter: number, currentLevel: number, pointsPerQuestion: number }`
- Generation: Existing quiz schema extended with adaptive branching logic
- Rendering: QuizPlayer.svelte extended with streaks/levels/achievements UI

**Difficulty:** Low-Medium. Adaptive branching logic is well-understood (similar to CAT—Computerized Adaptive Testing). Requires scoring + state management.

**Feasibility in Tauri v2 mobile:** Very high. This is almost identical to QuizPlayer already in spec, with added game state (score, streak, level).

---

#### Format C: **Simulation/Exploration Games** (Hypothesis Builder, Lab Simulator)

*Best for:* Process understanding, experimental design, iterative problem-solving
*Academic contexts:* Physics (pendulum simulator), chemistry (reaction simulator), biology (population dynamics), medicine (diagnostic reasoning)

**Example from ZenSci:**
- **Source:** Physics paper on projectile motion
- **Game:** "Design an experiment to maximize distance. Adjust angle, initial velocity, launch height. Predict outcome, then run simulation. Compare prediction to result. Adjust and retry."
- **Mechanics:** 5 experimental attempts, accuracy score = (distance_error / correct_distance) * 100
- **Feedback loop:** Choose parameters → predict → run simulation → compare → learn → adjust

**What it requires:**
- Source content must include *mathematical models* (equations, parameters, physics)
- Schema: `SimulationGame { model: { equations: string[], parameters: Parameter[] }, simulationSteps: SimulationStep[], maxAttempts: number, scoringFunction: string }`
- Generation: Extract parameters/equations from paper (via SymPy or LLM) → generate interactive simulator
- Rendering: Canvas-based visualization (graph, 3D model) + controls (sliders, buttons)

**Difficulty:** High. Requires domain-specific physics/math knowledge, custom rendering per simulation. Highest lift per game instance.

**Feasibility in Tauri v2 mobile:** Medium-to-High. Rendering can use Canvas or WebGL (both supported in WKWebView + Chromium). Performance: tight loop for physics simulation; test on lower-end devices. Offline: pre-compiled simulation engine (wasm? SymPy.js?).

---

#### Format D: **Narrative/Branching Games** (Choose-Your-Own-Adventure, Hypothesis Testing)

*Best for:* Reasoning, decision-making, case studies
*Academic contexts:* Medicine (diagnostic reasoning), law (case law analysis), policy (decision trees), research ethics (scenario-based learning)

**Example from ZenSci:**
- **Source:** Medical paper on diagnostic algorithms for chest pain
- **Game:** "You are presented with patient symptoms. Choose next diagnostic step (EKG, troponin, chest X-ray). Each choice branches to a new scenario. Reach correct diagnosis with minimum tests = high score. Wrong diagnosis = game over."
- **Mechanics:** 10-15 scenarios in a branching tree, score based on diagnostic accuracy + efficiency (fewer tests = higher score)
- **Feedback loop:** Choose action → see outcome → feedback on correctness → next scenario or game over

**What it requires:**
- Source content must have *decision logic* (if-then rules, branching pathways)
- Schema: `BranchingGame { scenarios: Scenario[], decisions: Decision[], scoreFunction: string, endConditions: EndCondition[] }`
- Generation: AI extracts decision points from case studies/algorithms → generates branching tree
- Rendering: Scenario text + button options (standard SvelteKit components)

**Difficulty:** Medium. Branching logic is straightforward; the challenge is extracting decision points from academic text. Requires LLM to parse "if condition, then outcome."

**Feasibility in Tauri v2 mobile:** Very high. No special rendering or performance requirements. Text-based, button-based interaction.

---

### Summary of Game Format Families

| Format | Source Content | Generation Complexity | Rendering Complexity | Performance Cost | Best for v0.1? |
|--------|---|---|---|---|---|
| **A: Concept Matching** | Taxonomy, relationships | Medium (NLP) | Medium (DOM drag-drop) | ~50MB | Yes (if NLP pre-computed) |
| **B: Adaptive Quiz** | Quiz questions | Low (branching logic) | Low (extends QuizPlayer) | <5MB | Yes (RECOMMENDED for demo) |
| **C: Simulation** | Equations, parameters | High (domain knowledge) | High (canvas/3D) | 100-200MB | No (defer to v1) |
| **D: Branching** | Case studies, scenarios | Medium (decision extraction) | Low (text + buttons) | <5MB | Yes (good second) |

---

## Part 2: What a "Working Game Demo" Means Concretely

The prior scout flagged games as absent. Cruz's direction: "We definitely need a working game demo."

A **working game demo** is NOT:
- A mockup or prototype (static screenshots)
- A quiz with a spinner (content consumption, not a game loop)
- A concept doc ("here's what we could build")
- A design system component library

A **working game demo** IS:
1. **Playable end-to-end** — A real game loop a user can complete in <3 minutes
2. **Generated from real ZenSci content** — Uses output from one of the 6 MCP servers (not hardcoded fixtures)
3. **Demonstrable in 2 minutes** — Clear enough that a non-technical stakeholder understands "this is a game, not a quiz"
4. **Built with v0.1 architecture** — Uses Rust backend + SvelteKit frontend + Tauri IPC, so it becomes the template for v1

---

### Scout 2: Which Game Format Should Be the Demo?

#### Route A: Concept Matching Game (Format A)

**Candidate:** "Classify organisms into phylogenetic taxonomy"

**Pros:**
- Visually obvious it's a game (drag-drop is interactive and satisfying)
- Works with paper-mcp output (papers often have taxonomies, classifications)
- Demonstrates the gap between "quiz" and "game" (UI clearly shows matching/sorting, not just questions)
- Reuses existing LearningCard + custom game data model

**Cons:**
- Requires NLP to extract concept relationships from papers (added complexity)
- Concept matching is niche (not all papers have taxonomies)
- Need custom rendering (DOM drag-drop, not standard Svelte Kit component)

**Demo complexity:** Medium. NLP pipeline adds 1-2 weeks if building from scratch; 3-4 days if using LLM to extract relationships.

**Demo sustainability:** Good—becomes a template for similar "matching" games in v1.

---

#### Route B: Adaptive Quiz Game (Format B) — RECOMMENDED

**Candidate:** "Level-up Quiz: Answer questions to advance through difficulty levels"

**Pros:**
- **Minimal new infrastructure** — Uses existing `QuizQuestion` schema (almost no new data model)
- **Easy to generate** — Paper-mcp already produces quiz questions with difficulty tags; just reorder by difficulty
- **Clearly a game** — Visual feedback (level counter, score, streak, progress bar) makes it feel like a game, not a quiz
- **Very fast to build** — Extend QuizPlayer.svelte with game state (level, score, streak, achievements)
- **Becomes the template** — All future quiz-like games (Adaptive Quiz, Branches, etc.) use this as base

**Cons:**
- Less visually novel (still looks like a quiz, with game elements overlaid)
- Doesn't showcase "this is different from a quiz" as dramatically as concept matching

**Demo complexity:** Low. 2-3 weeks to add game loop to QuizPlayer (state management, scoring, level progression, visual feedback).

**Demo sustainability:** Excellent—becomes v1 foundation for all quiz-based games.

---

#### Route C: Branching Game (Format D)

**Candidate:** "Medical case simulator: Choose diagnostic steps to reach diagnosis"

**Pros:**
- Very clear it's a game (branching scenarios are obviously interactive)
- Works with any case study content (medical, legal, policy)
- Demonstrates decision-making, not just recall
- Easy to generate (LLM extracts if-then rules from case text)

**Cons:**
- Requires curated case study content (not all papers are case studies)
- Branching logic requires careful testing (wrong logic breaks the game)
- Medium complexity to implement (scenario state machine)

**Demo complexity:** Medium. 2-3 weeks if you have source content; 4+ weeks if you need to create demo content.

**Demo sustainability:** Good—becomes template for scenario-based learning.

---

#### Route D: Simulation Game (Format C)

**Candidate:** "Physics simulator: Adjust parameters, predict outcome, see result"

**Pros:**
- Most impressive visually (animated visualization)
- Demonstrates technical sophistication
- Very clear it's a game

**Cons:**
- **Highest complexity** — Requires physics engine, custom rendering, domain knowledge
- Takes 4-6 weeks to build even a simple pendulum simulator
- Ties v0.1 launch date to specialized physics rendering
- Risk: If simulator UX is poor, demo undermines, not showcases, the platform

**Demo complexity:** High. 4-6 weeks minimum.

**Demo sustainability:** Risky—specialized to physics/STEM; less generalizable to other domains.

---

### Recommendation: Route B (Adaptive Quiz Game) + Route A (Concept Matching) Sequence

**Phase 1 (v0.1 — Week 1-3):** Build and ship Adaptive Quiz Game demo
- Use paper-mcp output (existing quiz questions with difficulty tags)
- Extend QuizPlayer.svelte with level progression, score, streaks, achievements
- New schema: `GameState { currentLevel, totalScore, currentStreak, achievementsList }`
- Becomes proof that games are feasible, not just quizzes

**Phase 2 (v0.1 RC or v1 — Week 4-6):** Add Concept Matching Game
- Extract relationships from paper sections (via LLM or heuristic parsing)
- Build drag-drop UI component (reusable for future matching games)
- Demonstrates variety of game formats

**Why this sequence?**
1. **Adaptive Quiz ships faster** — unblocks v0.1 launch; proves games work
2. **Concept Matching adds visual novelty** — shows variety, builds confidence for future formats
3. **Both reuse existing schemas** — no rearchitecting of transformation pipeline
4. **Sets foundation for v1** — each format becomes a template for similar games

---

## Part 3: How Games Integrate with ZenSci Content Transformation Pipeline

### Current Pipeline (v0.1 spec)

```
ZenSci MCP servers (latex-mcp, paper-mcp, etc.)
  ↓
Generate mobile formats: reading, quiz, card_deck, video_script
  ↓
Store in AgenticGateway memory / cloud
  ↓
Mobile app syncs, caches locally
  ↓
SvelteKit renders: ReadingView, QuizPlayer, SwipeableDeck
```

### Updated Pipeline with Games (v0.1+v1 evolution)

```
ZenSci MCP servers
  ↓
Generate mobile formats: reading, quiz, card_deck, lesson, video_script
  ↓
(NEW) Extract game metadata: content_type + game_format tag
  ↓
(NEW) Generate game-specific data: GameState, AdaptiveLogic, BranchingTree, ConceptGraph
  ↓
Store in Gateway + mobile_formats table (adds game_data BLOB column)
  ↓
Mobile app syncs: includes game metadata alongside content
  ↓
SvelteKit router: if game_format == "adaptive_quiz" → render GameQuizPlayer
                  if game_format == "branching" → render BranchingGamePlayer
                  if game_format == "matching" → render MatchingGamePlayer
```

### Data Model Changes Required

**v0.1 (current):**
```typescript
interface MobileContent {
  id: string;
  contentType: 'card_deck' | 'quiz' | 'reading' | 'video_script' | 'lesson';
  data: CardDeck | Quiz | ReadingContent | VideoScript | Lesson;
}
```

**v0.1+v1 (add game support):**
```typescript
interface MobileContent {
  id: string;
  contentType: 'card_deck' | 'quiz' | 'reading' | 'video_script' | 'lesson' | 'game';
  gameFormat?: 'adaptive_quiz' | 'concept_matching' | 'branching' | 'simulation'; // Optional
  data: CardDeck | Quiz | ReadingContent | VideoScript | Lesson | GameContent;
}

interface GameContent {
  gameFormat: 'adaptive_quiz' | 'concept_matching' | 'branching' | 'simulation';
  sourceQuiz?: Quiz; // For adaptive_quiz games
  concepts?: Concept[]; // For concept_matching games
  scenarios?: Scenario[]; // For branching games
  simulationModel?: SimulationModel; // For simulation games
  gameRules: GameRules;
}

interface GameRules {
  scoringFunction: string; // e.g., "points_per_question * accuracy_multiplier"
  difficultyProgression: 'linear' | 'exponential' | 'adaptive';
  timeLimit?: number; // seconds
  livesPerRound?: number;
  achievements: Achievement[];
}
```

### Content Transformation Decision: Compile-Time vs. Runtime

**Reframing the prior scout's Tension 1:**

**Compile-time game generation (Recommended for v0.1):**
- MCP servers generate game data at publish time
- Example: `paper-mcp` takes a paper + `game_format="adaptive_quiz"` → emits Quiz + GameRules
- Pros: Fast UX (no wait); predictable; low server load; offline-friendly
- Cons: Fixed formats per content; can't personalize to student preference

**Runtime game generation (v1+):**
- Student opens content → app requests format → LLM generates game on-demand → caches result
- Example: `paper-mcp.generate_game(paper_id, format="concept_matching", student_level="advanced")`
- Pros: Adaptive to student level; formats generated based on current preferences; personalized
- Cons: Higher latency (3-5s first load); server cost; requires re-cache if preferences change

**Recommendation for v0.1:** Use **compile-time** game generation. Start with instructor/instructor-curated game formats. Move to runtime in v1 if personalization ROI is clear.

---

## Part 4: Technical Feasibility in Tauri v2 Mobile

### Tauri v2 Mobile Status (February 2026)

**iOS:**
- WKWebView (WebKit/Safari engine) on iOS 16+
- Supports all modern CSS, JavaScript
- Limitation: No direct access to device GPU acceleration (canvas works, WebGL supported but limited by WebKit)
- App Store approval: 1-3 weeks typical, can be picky about IPC patterns
- Status: Production-ready since Jan 2025

**Android:**
- Chromium WebView on Android 10+
- Full support for canvas, WebGL, modern CSS
- Drag-drop works well (better than iOS)
- Google Play: 1-2 hours automated, no manual review usually
- Status: Stable since mid-2025

**Tauri v2 mobile overall:** Production-ready for consumption apps. Known issues: build times (30-45 min), occasional linker issues on Android NDK version mismatches.

### Game Rendering Performance by Format

| Format | Rendering Method | Mobile Performance | iOS Concerns | Android Concerns | Notes |
|--------|---|---|---|---|---|
| **Adaptive Quiz** | DOM (divs, buttons) | Excellent (<1 frame drop) | None | None | Standard web tech |
| **Concept Matching** | DOM drag-drop + CSS | Very Good (~2-3 fps) | Elastic scroll may interfere with dragging | None | Test on lower-end devices |
| **Branching** | DOM (text + buttons) | Excellent | None | None | Trivial performance |
| **Simulation** | Canvas 2D | Good (60fps possible, test dependent) | WebKit canvas is slower than Chromium; test on iPhone 12+ | Excellent | Avoid intensive loops in physics; use requestAnimationFrame |
| **Simulation** | WebGL 3D | Medium (30fps realistic) | WebKit WebGL is limited; test early | Good | Risky for v0.1; recommend defer to v1 |

### Gesture & Touch Interaction

**What works well in Tauri v2 mobile:**
- Standard touch events (`touchstart`, `touchmove`, `touchend`) → works identically on iOS/Android
- Drag-drop (using Svelte `on:dragstart` or custom touch handlers) → works but needs tuning for overscroll
- Long-press detection → need custom timer + touch handler
- Swipe detection → already in spec (SwipeableDeck.svelte)
- Pinch-to-zoom → not in spec; WebView handles natively

**Gotchas:**
- iOS has elastic overscroll (feels different from Android momentum); games shouldn't fight this
- iOS may disable passive event listeners by default; affects drag-drop performance
- Android back button default behavior (browser back) can interfere with game state if not managed

**Recommendation:** Test drag-drop games on real iOS + Android devices early (by Week 2 of Phase 1).

### Offline Support for Games

All game formats proposed (Adaptive Quiz, Concept Matching, Branching) are **fully offline-capable**:
- Game logic is deterministic (no server calls during play)
- Pre-computed game data cached in rusqlite at sync time
- Player state (score, answers, progress) stored locally; sync on network return

**Storage cost per game:**
- Adaptive Quiz: ~50-100KB (includes source quiz + game rules)
- Concept Matching: ~30-50KB (includes concept graph + images)
- Branching: ~20-40KB (includes scenario tree)

500MB cache (v0.1 spec) can store ~5,000-10,000 games without compression.

---

## Part 5: The Reframe

**From:** "Should games be a separate content type in the mobile app?"

**To:** "Should games be compile-time-generated transformations of existing content types (quiz → adaptive quiz, paper sections → concept matching), or runtime-generated based on student preferences?"

**Implication:** Games are NOT a new module in the transformation pipeline. They are *game mechanics applied to existing formats*. This reframe cascades:

1. **Schema:** No `Game` content type. Instead: `contentType: "quiz"` + `gameFormat: "adaptive_quiz"` + `gameRules: {}`
2. **Generation:** MCP servers don't need new endpoints. Existing tools (paper-mcp, grant-mcp) just add an optional `--game-format` flag
3. **Rendering:** Instead of 5 different players (ReadingView, QuizPlayer, CardDeck, etc.), build 5 game players (GameQuizPlayer, GameConceptPlayer, GameBranchingPlayer, etc.) that *wrap* existing players with game state
4. **v1 evolution:** Runtime generation (LLM generates game on-demand) becomes an opt-in feature, not a requirement

---

## Part 6: Architectural Questions for Cruz

### Q1: Game Scope for v0.1

> "Should v0.1 ship *one* working game format (Adaptive Quiz), or *two* (Adaptive Quiz + Concept Matching)?"

- **One format:** Faster ship (2-3 weeks); proves games are feasible; leaves room for v1 expansion
- **Two formats:** More impressive demo; shows variety; sets template for multiple game types; adds 2-3 weeks

**Recommended answer for v0.1:** One format (Adaptive Quiz). Ship in first 3 weeks; validate with users. Add Concept Matching in v1 if demand signals are strong.

---

### Q2: Game Generation Strategy

> "Should game data be generated at content-publish time (today's decision), or should students request game formats on-demand?"

- **Compile-time (recommended for v0.1):** Instructor/instructor publishes paper → MCP server auto-generates quiz + adaptive_quiz format → both cached on all devices
- **Runtime (defer to v1+):** Student opens paper → chooses "Play adaptive quiz" → app calls `generate_game(paper_id, format="adaptive_quiz")` → LLM generates → caches

**Recommended answer for v0.1:** Compile-time. Simpler infrastructure. Runtime (with LLM) comes in v1 if personalization ROI is clear.

---

### Q3: Should v0.1 Demo Include a Custom Game From Scratch?

> "Should we build a demo with fresh game data generated specifically for the demo, or use existing paper-mcp output + game transform?"

- **Fresh demo data:** More polished, showcases full potential; requires curating source content
- **Existing paper-mcp data:** Faster, reuses existing infrastructure; might feel less curated

**Recommended answer:** Use existing paper-mcp output (a paper already published to the portal). Extend it with game transforms. Faster, more authentic to real usage.

---

## Part 7: Gaps and Open Questions

### Gap 1: Achievement/Reward Design

The proposed game schema includes `achievements`, but no definition of what achievements are, how they're earned, or how they're displayed. Need:
- Achievement taxonomy (e.g., "5-Streak", "Perfect 10", "Speed Demon: answered 5 questions <10s each")
- Achievement UI (badges, unlocks, notifications)
- Reward mechanism (do achievements affect score, or are they cosmetic?)

### Gap 2: Difficulty Adaptation Logic

Adaptive Quiz format has `difficultyProgression` (linear, exponential, adaptive), but no algorithm defined. Need:
- IRT (Item Response Theory) or simpler heuristic?
- How many questions before difficulty changes?
- How to handle edge cases (all correct → jump to hardest? All wrong → repeat at current level?)

### Gap 3: Leaderboards and Multiplayer

This scout does not explore multiplayer games or leaderboards. Single-player games are in scope for v0.1. Multiplayer (competitive or cooperative) should be a separate scout.

### Gap 4: Accessibility in Games

Games require careful attention to a11y (accessibility). Adaptive Quiz is mostly text-based and screen-reader compatible. Concept Matching (drag-drop) is harder for VoiceOver/TalkBack. Need:
- ARIA labels for game elements
- Keyboard alternatives to drag-drop (tab + arrow keys)
- Color contrast in game UI

### Gap 5: Analytics and Player Behavior

No mention of how to track game engagement, drop-off rates, or which game formats are most effective. Need:
- What metrics matter? (completion rate, time-to-completion, score distribution, feature adoption)
- When to collect analytics (offline vs. online sync)
- Privacy implications (student data, FERPA compliance for educational content)

---

## Part 8: Files and Artifacts

### New Schema to Add to `SCHEMAS.md`

```typescript
// Game Formats (v0.1+v1)

// Adaptive Quiz Game
interface AdaptiveQuizGame {
  gameFormat: "adaptive_quiz";
  sourceQuiz: Quiz; // Base quiz questions
  gameRules: GameRules;
  difficultyLevels: number; // e.g., 5 levels (1=easiest, 5=hardest)
  levelProgression: "linear" | "exponential" | "adaptive";
  questionsPerLevel: number;
}

// Concept Matching Game
interface ConceptMatchingGame {
  gameFormat: "concept_matching";
  concepts: Concept[]; // Term-definition pairs
  conceptGraph: ConceptRelationship[]; // parent-child, related-to, etc.
  matchingSlots: string[]; // Available categories/bins
  timeLimit?: number; // seconds
  livesPerRound?: number;
  gameRules: GameRules;
}

interface Concept {
  id: string;
  term: string;
  definition: string;
  imageUrl?: string;
  difficulty: "easy" | "medium" | "hard";
}

interface ConceptRelationship {
  sourceConceptId: string;
  targetConceptId: string;
  relationshipType: "parent_of" | "related_to" | "synonym" | "opposite";
}

// Branching Scenario Game
interface BranchingGame {
  gameFormat: "branching";
  scenarios: Scenario[];
  decisionTree: DecisionNode[];
  gameRules: GameRules;
}

interface Scenario {
  id: string;
  title: string;
  narrative: string; // Story text
  imageUrl?: string;
  decisions: Decision[]; // Available choices
}

interface Decision {
  id: string;
  scenarioId: string;
  choiceText: string;
  nextScenarioId?: string; // Branching target
  outcome: string; // What happened as a result
  scoreImpact: number; // +/- points
}

interface GameRules {
  scoringFunction: string; // e.g., "points_per_question * (1 + streak_bonus)"
  pointsPerQuestion: number;
  streakBonus: number; // e.g., 1.5 means 50% bonus for streaks
  achievements: Achievement[]; // Unlock conditions
  maxScore: number;
  passingScore: number;
}

interface Achievement {
  id: string;
  name: string;
  description: string;
  unlockCondition: string; // e.g., "streak_count >= 5"
  badgeUrl?: string;
  points: number; // Bonus points for unlocking
}

// Player State (stored in annotations table for v2 sync)
interface GamePlayerState {
  gameId: string;
  playerId: string;
  currentScore: number;
  currentLevel?: number; // For adaptive games
  currentStreak: number;
  totalAttempts: number;
  correctAnswers: number;
  timeSpentSeconds: number;
  achievementsUnlocked: string[]; // achievement IDs
  isComplete: boolean;
  completedAt?: number; // Unix timestamp
  lastUpdated: number;
}
```

### Recommended Reading

- **Paper:** Prensky, M. (2007). "Digital Game-Based Learning." *Computers in Education*, 9(1).
- **Framework:** Bloom's taxonomy + game mechanics (challenge, feedback, reward loops)
- **Reference:** Duolingo's adaptive algorithm (public papers available)

---

## Summary: Tensions Resolved

| Tension | Prior Scout Finding | This Scout Resolution |
|---------|---|---|
| **Tension 3: Games Gap** | "Games are absent; no schema" | Games are runtime-generated transformations of quiz/lesson content. Start with compile-time generation (v0.1). Demo: Adaptive Quiz format. |
| **Format ownership** | "Who decides content shape?" | Game format is instructor/curated choice (compile-time). Personalization (student preference) deferred to v1. |
| **Content transformation pipeline** | "Path A vs. Path B?" | Path A: Extend existing tools with `--game-format` flag. Simpler. Games add optional `gameFormat` to `MobileContent` schema. |
| **Working demo requirement** | "What is a game demo?" | 1 playable game, generated from real paper-mcp output, built in 2-3 weeks using existing Rust + SvelteKit architecture. Adaptive Quiz is the target. |

---

## Recommendation: Next Steps

### For Product (Cruz)

1. **Decide:** One game format (Adaptive Quiz) or two (Adaptive + Concept Matching) for v0.1?
2. **Scope:** Confirm compile-time game generation is acceptable for v0.1 (runtime/personalization → v1)
3. **Design:** Define 3-5 achievement types for Adaptive Quiz demo

### For Architecture

1. Add `gameFormat` and `gameRules` fields to `MobileContent` schema
2. Extend `paper-mcp` to emit game metadata when publishing quiz-bearing papers
3. Plan SvelteKit game player components (GameQuizPlayer as primary; others in v1)

### For Implementation (v0.1 Phase 1)

1. Week 1-2: Schema + data model (Adaptive Quiz game rules, achievements)
2. Week 2-3: Extend QuizPlayer.svelte → GameQuizPlayer (level progression, score, streaks, achievements)
3. Week 3: Integration test with real paper-mcp output
4. Week 3-4: Polish UX, add game sounds/animations (optional but delightful)

---

**Document prepared by:** Strategic Scout
**Date:** 2026-02-18
**Status:** Ready for product decision gate
**Next gate:** Clarify Q1-Q3 above before committing implementation
