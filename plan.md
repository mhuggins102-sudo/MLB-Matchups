# Top 10 Mode - Implementation Plan

## Overview
A new "Top 10" game mode where players are presented a question (e.g., "Top 10 HR hitters in the 1990s") and must type player names to guess who's on the list. Correct guesses flip to reveal the player's rank and stat value; misses show a red X with the player's actual ranking.

---

## 1. Categories

### Batter Categories (9 total)
**Decade-computable (from season data):**
1. `HR` — Home Runs
2. `H` — Hits
3. `RBI` — Runs Batted In
4. `SB` — Stolen Bases
5. `BB` — Walks
6. `BA` — Batting Average (min 1500 PA over the decade)
7. `OPS` — On-Base Plus Slugging (min 1500 PA over the decade)
8. `XBH%` — Extra-Base Hit Rate (min 1500 PA over the decade)

**Career-only (for letter-filtered questions only):**
9. `WAR` — Career WAR (batter)

### Pitcher Categories (5 total)
**Decade-computable (from season data):**
1. `W` — Wins
2. `K` — Strikeouts
3. `ERA` — Earned Run Average (min 500 IP over the decade; lower is better)
4. `IP` — Innings Pitched

**Career-only (for letter-filtered questions only):**
5. `WAR` — Career WAR (pitcher)

---

## 2. Question Types

Each question is one of three filter types (no plain career/unfiltered):

### A. Decade Only
- "Top 10 batters in HR in the 1990s"
- Stats aggregated across all seasons within the decade (e.g., 1990–1999)
- WAR excluded (career-only data)

### B. Letter Only
- "Top 10 batters with last name starting 'H' in career HR"
- "Top 10 pitchers with first name starting 'M' in career K"
- Uses career stats (all categories including WAR)
- Filter: first letter of first name OR last name (randomly chosen)

### C. Decade + Letter Combined
- "Top 10 batters with last name starting 'H' in HR in the 1990s"
- Uses decade-aggregated stats (WAR excluded)
- Filter: decade + first letter of first or last name

### Question Generation Logic
- Weighted random: ~50% decade-only, ~25% letter-only, ~25% combined
- Must validate that at least 10 eligible players exist for the question
- If global era setting is a specific decade, decade questions use that decade
- If global era is "All Eras" or "Modern", decade is randomly chosen from [1930–2020]
- Player pool setting (batters/pitchers/all) constrains which category types appear
- Eligibility (Standard/All-Star/HOF) filters the player pool
- For letter questions, only use letters that yield 10+ eligible players
- Combined questions may need multiple attempts to find valid letter+decade+stat combos

---

## 3. Decade Stat Computation

### New Function: `computeDecadeStats(decade)`
Aggregates per-season data for all players active in the given decade (years `decade` through `decade + 9`).

**Batting:** Sum AB, H, BB, HBP, SF, HR, 2B, 3B, SB across matching seasons per player. Compute BA, OPS, XBH% from aggregates.

**Pitching:** Sum IPouts, ER, SO, W, HR, BB, H across matching seasons per player. Compute ERA from aggregates.

**Fix needed:** 2025 pitching season objects (`pitSeasons`) are missing `BB` and `H` fields. Must add these during the `_pit25` processing loop (line ~4697) so decade aggregation works for future WHIP/K-BB if needed.

### Caching
Cache decade stats in a Map keyed by decade number to avoid recomputation. Invalidate on data reload.

---

## 4. Player Search / Autocomplete

### Input Component
- Text input field below the question, above the 10 cells
- As user types (min 2 characters), show dropdown of matching players
- **Prefix match**: Match start of first name OR start of last name
- Max 8 results in dropdown to keep it manageable
- Keyboard navigation: arrow keys to highlight, Enter to select
- Mobile-friendly: large touch targets in dropdown

### Disambiguation
- Display format: `"Ken Griffey Jr. (1989–2010)"`
- Years derived from player's `years` Set (min year – max year)
- If two players have identical name AND overlapping years, append team: `"Alex Gonzalez (1994–2006, TOR)"`

### Player Pool for Search
- Search filters to only players who could plausibly be in the answer set:
  - Correct type (hitter/pitcher matching the question)
  - Active in the relevant decade (for decade questions)
  - Name matches letter constraint (for letter questions)
- This prevents confusion but doesn't reveal the answer (many eligible players won't be top 10)

### Implementation
- Build a sorted name index at startup (after `buildAgg()`)
- On keyup, filter index with prefix match, limit to 8 results
- Render dropdown absolutely positioned below input
- Click or Enter selects a player, triggering the guess evaluation

---

## 5. UI Layout

### Game Screen (when gameMode === 'top10')
```
┌─────────────────────────────────────┐
│  Round 3/10          Score: 1,240   │
│                                     │
│  "Top 10 HR hitters in the 1990s"   │
│                                     │
│  ┌─────────────────────────────┐    │
│  │  🔍 Type a player name...   │    │
│  │  ┌─ dropdown ─────────────┐ │    │
│  │  │ Ken Griffey Jr (89-10) │ │    │
│  │  │ Kevin Mitchell (84-98) │ │    │
│  │  └────────────────────────┘ │    │
│  └─────────────────────────────┘    │
│                                     │
│  Guesses: 3/10  Hits: 2            │
│                                     │
│  ┌──┬─────────────────────┬─────┐   │
│  │ 1│  ??? (unrevealed)   │     │   │
│  ├──┼─────────────────────┼─────┤   │
│  │ 2│  Mark McGwire       │ 405 │   │
│  ├──┼─────────────────────┼─────┤   │
│  │ 3│  Ken Griffey Jr.    │ 382 │   │
│  ├──┼─────────────────────┼─────┤   │
│  │ 4│  ??? (unrevealed)   │     │   │
│  ├──┼─────────────────────┼─────┤   │
│  │ ..│  ...               │     │   │
│  ├──┼─────────────────────┼─────┤   │
│  │10│  ??? (unrevealed)   │     │   │
│  └──┴─────────────────────┴─────┘   │
│                                     │
│  [Bomb] [Reroll]     [Give Up]      │
└─────────────────────────────────────┘
```

### Cell States
1. **Unrevealed**: Dark/muted background, rank number visible, "???" placeholder
2. **Revealed (correct guess)**: Flip animation → green/bright background, player name + stat value
3. **End-of-round reveal**: Cascade reveal of remaining unrevealed cells (dimmer color than player-guessed ones)

### Miss Feedback
- Large red X overlay appears center-screen (or above the board) for ~1.5 seconds
- Text below X: "Kevin Mitchell was #14" or "Kevin Mitchell was not active in the 1990s"
- Fades out after 1.5s

---

## 6. Animations

### Correct Guess — Cell Flip
- CSS 3D transform: rotateX(90deg) → swap content → rotateX(0deg)
- Duration: ~400ms
- Green glow/pulse on completion

### Miss — Red X
- Scale-in from 0 → 1 with slight overshoot
- Hold for 1.5s, then fade out
- Duration: ~2s total

### End-of-Round Cascade
- Same pattern as Showdown mode: 300ms step delay between each unrevealed cell
- Cells flip to reveal with slightly muted styling (to differentiate from player guesses)
- After all revealed: show round score, update total, show Next Round button

### All animations skip when `useAnimations === false`

---

## 7. Scoring

### Per-Round (max 1000 points)
Points awarded for each correctly guessed position:
| Position | Points |
|----------|--------|
| #1       | 200    |
| #2       | 160    |
| #3       | 130    |
| #4       | 110    |
| #5       | 90     |
| #6       | 80     |
| #7       | 70     |
| #8       | 60     |
| #9       | 50     |
| #10      | 50     |
| **Total**| **1000**|

Score is based on which positions were correctly guessed, not guess order.

### Round Results Emoji
- 10/10 correct: 💎
- 8-9 correct: 🟩
- 6-7 correct: 🟧
- 4-5 correct: 🟨
- 0-3 correct: 🟥

---

## 8. State Management

### New Variables
```javascript
var top10State = {
    answer: [],          // Array of 10 {id, name, value, rank} objects, sorted by rank
    guessesLeft: 10,
    guessedPositions: [],// Set of rank indices (0-9) that have been revealed
    missCount: 0,
    hitCount: 0,
    question: {},        // {stat, decade, letter, letterType, playerType, displayText}
    roundOver: false
};
```

### Decade Stats Cache
```javascript
var decadeStatsCache = new Map(); // decade → Map(playerID → {statKey: value, ...})
```

---

## 9. Integration with Existing Game Flow

### Game Mode Registration
- Add `'top10'` to the modes array alongside `'howMuch'`, `'sideBySide'`, `'pickEm'`
- Add a 4th game mode button in the HTML
- Add to `generateDailySettings()` mode pool

### Pregeneration
- New function `pregenTop10(rng, usedStats)` that:
  1. Picks question type (decade/letter/combined) using weighted random
  2. Picks stat category based on player pool
  3. Picks decade (from global era setting or random)
  4. Picks letter (random, validated to have 10+ players)
  5. Computes the top 10 answer
  6. Returns: `{question, answer}` — pre-computed for instant round start

### Round Flow
1. `startRound()` → loads pregenerated question+answer into `top10State`
2. Renders question text, empty board, search input
3. Player types and selects guesses
4. Each guess → evaluate → flip cell or show miss
5. After 10 guesses (or Give Up) → cascade reveal remaining → show score
6. "Next Round" button → proceed

### Bomb / Reroll
- **Bomb**: Reveals a random unrevealed cell (costs a guess? or free reveal?)
- **Reroll**: Generates a new question for this round (resets guesses)

---

## 10. Implementation Steps (ordered)

### Phase 1: Data Layer
1. Fix 2025 pitching season objects to include BB and H fields
2. Build `computeDecadeStats(decade, type)` function
3. Build decade stats cache with lazy computation
4. Build player name index for search

### Phase 2: Question Generation
5. Define Top 10 category configs (stat key, label, type, lowerBetter, minThreshold)
6. Build question generator (picks type/stat/decade/letter)
7. Build answer computation (top 10 ranking for given question params)
8. Build `pregenTop10()` for round pre-generation
9. Validate questions have 10+ eligible players

### Phase 3: UI
10. Add "Top 10" game mode button to HTML
11. Build Top 10 game screen HTML (question, search input, 10-cell board, controls)
12. Build player search/autocomplete component
13. Style cells for unrevealed/revealed/cascade states
14. Add guess counter and hit counter display

### Phase 4: Game Logic
15. Implement guess evaluation (lookup in answer, determine hit/miss)
16. Implement cell flip on correct guess
17. Implement miss feedback (red X + ranking text)
18. Implement end-of-round cascade reveal
19. Implement scoring calculation
20. Wire into startRound/nextRound flow

### Phase 5: Polish
21. Add flip animation CSS
22. Add miss X animation CSS
23. Add cascade reveal timing
24. Handle edge cases (give up early, all 10 found before 10 guesses)
25. Add to daily challenge mode pool
26. Add round result emoji tracking
27. Dark mode styling for all new components
