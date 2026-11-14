# Voxel-world Revamp Plan

## 1. Context & Problem

`Mankeluvsit/Voxel-world` ("Sky Metropolis") is an isometric 3D city builder (React + Three.js + Vite + Express). Its AI layer currently depends on two live cloud providers:

- **Gemini** — `@google/genai`, model `gemini-3.5-flash` (goals, news, terrain layout, Idea Lab, procedural music, Lyria audio, performance/LOD analysis).
- **OpenAI** — `openai`, model `gpt-4o-mini`, switched via `VITE_AI_PROVIDER`.

Goal: remove both cloud providers, replace the entire AI layer with a local deterministic "fake AI" (template library + selector), and do a full refresh of the app — no API keys required, zero token cost. Additionally, add real gameplay substance (6 interlocking systems) so the city builder has depth, not just a visual toy.

## 2. Confirmed Decisions

| Decision | Choice |
|---|---|
| Cloud AI scope | Remove **both** Gemini and OpenAI entirely |
| Fake AI design | **A2** — template library + deterministic selector parameterized by city state |
| Randomness model | **Math.random** — consistent with the rest of the game; no seed plumbing |
| Idea Lab | **B** — pure offline idea board; drop the prompt textarea, drop react-markdown; "Generate Concept" → "Add Random Idea" from pool |
| Music Lab | **A** — keep the lab, drop Lyria mode, keep Tone.js procedural playback with a local template-pool composer |
| UI direction | **B** — keep indigo/sky-900 palette and component structure; strip slop only (gradient text, scanline overlays, emoji icons, stacked glass panels, lucide-only icons) |
| Gameplay substance | **All 6 systems** (road connectivity, pollution, happiness, disasters, tech/unlock tree, blueprints) |
| Tech/unlock gating | **Reputation that persists across resets** (Against the Storm style — meta-progression) |
| Disaster frequency | **Pollution-gated** — disasters only trigger when total pollution exceeds a threshold |
| Blueprint format | **Base64 of compressed JSON** — small enough for clipboard |
| Multiple settlements | **Deferred** to a future update — single city for v1 |

### UI slop to remove specifically
- Gradient text on the "SkyMetropolis" title
- CRT scanline overlay divs in `IdeaLab.tsx` and `UIOverlay.tsx`
- Emoji-as-icons in the toolbar (`⛏️`, `✕`, `⚡💧💎` badges) — replace with lucide-react icons
- Stacked translucent panels in `StartScreen.tsx` — make it a single solid surface
- `react-markdown` usage — replaced with static JSX detail blocks

### Deferred-to-my-judgment decisions
- StartScreen "AI Advisor" toggle stays, relabeled to "City Advisor (quests & news)". Gates `aiEnabled` throughout the game loop.
- `analyzePerformance` — existing `getFallbackRecommendation` heuristic in `IsoMap.tsx` wired directly; no AI call.
- `window.aistudio` global removed from `types.ts`.
- `metadata.json` description updated.

## 3. Security Bug Found

`vite.config.ts` inlines `GEMINI_API_KEY` and `OPENAI_API_KEY` into the client bundle via `define`. `openaiService.ts` uses `dangerouslyAllowBrowser: true`. Removing both providers eliminates this entirely.

## 4. Affected Files

### Delete
- `services/geminiService.ts`
- `services/openaiService.ts`

### Modify
- `package.json` — remove `@google/genai`, `openai`, `react-markdown`, `@types/express`
- `vite.config.ts` — remove `define` blocks for API keys
- `server.ts` — strip all `/api/ai/*` routes; keep `/api/health` and static serving
- `services/aiService.ts` — rewrite as sync local calls (see §6 for API contract)
- `services/localAIService.ts` — **new** template library + selector
- `types.ts` — type changes (see §5)
- `constants.tsx` — unlock thresholds, pollution values, new building configs
- `App.tsx` — remove AI key checks, integrate new game systems, reputation persistence
- `IdeaLab.tsx` — remove react-markdown, local generation, prompt textarea
- `components/MusicLab.tsx` — remove Lyria mode, keep procedural
- `components/UIOverlay.tsx`, `components/StartScreen.tsx`, `components/IsoMap.tsx` — UI cleanup + new stat displays
- `.env.example` — remove API key lines
- `metadata.json` — update description

## 5b. Constants Changes (`constants.tsx`)

```ts
// NEW: pollution values per building type (added to BUILDINGS record)
[BuildingType.Industrial]: { ..., pollution: 5 },   // high pollution
[BuildingType.Commercial]: { ..., pollution: 1 },   // low pollution
[BuildingType.Residential]: { ..., pollution: 0 },
[BuildingType.Park]: { ..., pollution: -2 },        // pollution reducer
[BuildingType.Road]: { ..., pollution: 0 },
[BuildingType.Extractor]: { ..., pollution: 0 },
[BuildingType.None]: { ..., pollution: 0 },

// NEW: unlock thresholds (reputation required)
export const UNLOCK_REP = {
  [BuildingType.Road]: 0,
  [BuildingType.Residential]: 0,
  [BuildingType.Commercial]: 0,
  [BuildingType.Park]: 0,
  [BuildingType.Industrial]: 5,
  [BuildingType.Extractor]: 10,
};

// NEW: game constants
export const POLLUTION_RADIUS = 3;           // tiles
export const DISASTER_POLLUTION_THRESHOLD = 50; // total pollution before disasters can trigger
export const DISASTER_CHANCE = 0.15;          // 15% per tick when above threshold
export const HAPPINESS_MIN = 0;
export const HAPPINESS_MAX = 100;
export const INITIAL_HAPPINESS = 50;
export const INITIAL_REPUTATION = 0;
export const REPUTATION_PER_GOAL = 3;         // reputation earned per completed goal

```ts
// TileData — add connectivity + pollution
export interface TileData {
  x: number;
  y: number;
  buildingType: BuildingType;
  variant?: number;
  resourceType?: 'Energy' | 'Water' | 'Mineral';
  isPowered?: boolean;
  hasWater?: boolean;
  isIncomeGenerating?: boolean;
  isConnected?: boolean;   // NEW: road connectivity
  pollution?: number;       // NEW: pollution level on this tile
}

// CityStats — add happiness
export interface CityStats {
  money: number;
  population: number;
  day: number;
  energyProduction?: number;
  energyDemand?: number;
  waterProduction?: number;
  waterDemand?: number;
  mineralProduction?: number;
  taxRate?: number;
  happiness?: number;      // NEW: 0-100
}

// NEW: Disaster event type
export interface DisasterEvent {
  id: string;
  type: 'fire' | 'flood' | 'crime_wave';
  x: number;
  y: number;
  radius: number;
  day: number;
  resolved: boolean;
}

// Remove from global Window interface:
//   aistudio (no longer used)
```

## 6. `aiService.ts` API Contract (after refactor)

All functions become synchronous. Callers in `App.tsx` and `IdeaLab.tsx` must drop `await` and null-checks where the return type is no longer nullable.

```ts
// Before → After
generateCityGoal(stats, grid): Promise<AIGoal | null>  →  AIGoal
generateNewsEvent(stats, action): Promise<NewsItem | null>  →  NewsItem | null
getOptimalLayout(w, h): Promise<ResourceNode[]>  →  ResourceNode[]
generateFeatureIdeas(prompt): Promise<Partial<Idea> | null>  →  Partial<Idea>   // no prompt (dropped textarea)
generateIdeaDetails(idea): Promise<string | null>  →  string                    // static template
generateProceduralMusic(prompt): Promise<MusicComposition | null>  →  MusicComposition  // keyword match or random
```

## 7. Data Flow (game loop integration)

The existing `setInterval` in `App.tsx` (TICK_RATE_MS = 2000ms) is the single integration point. New systems are calculated in this order each tick:

1. **Resource production** — existing (extractors yield energy/water/minerals)
2. **Road connectivity** — flood-fill from road tiles; mark `isConnected` on each tile
3. **Pollution** — industry tiles add pollution to 3-tile radius; accumulate per-tile pollution values
4. **Building yields** — existing logic, but gated by `isConnected` (unconnected buildings produce nothing)
5. **Happiness** — `50 + parksPerCapita*10 + (10-taxRate)*2 - avgPollution*5 - crime*3`, clamped 0–100
6. **Population growth** — existing logic, multiplied by `happiness/50`
7. **Disaster check** — if total pollution > threshold, roll for disaster (fire/flood/crime wave). On trigger: damage buildings, broadcast news, add to `disasterFeed`.
8. **Goal check** — existing logic, using local `generateCityGoal`
9. **News** — existing logic, using local `generateNewsEvent`
10. **Reputation** — when a goal completes, add reputation points (separate from money reward)
11. **Save** — existing localStorage auto-save, plus separate `sky_metropolis_reputation` key

## 8. Migration Path (existing saves)

Old saves in `localStorage` (`sky_metropolis_save`) may lack `happiness`, `pollution`, `isConnected` fields. Migration strategy:

- On load, if `stats.happiness` is undefined, default to 50.
- On load, if any tile lacks `isConnected`, recompute connectivity on first tick.
- On load, if any tile lacks `pollution`, default to 0.
- Reputation is stored separately (`sky_metropolis_reputation`), so old saves simply start at 0 reputation.
- The `aiEnabled` flag is preserved as-is (relabeled to "City Advisor" in UI only).

## 9. Failure Modes

| Failure | Handling |
|---|---|
| Corrupted `sky_metropolis_save` | Existing try/catch in useState initializers returns defaults. No change needed. |
| Corrupted `sky_metropolis_reputation` | Parse in try/catch, default to 0. |
| localStorage full | Existing behavior (silent fail). No change. |
| Template pool exhausted | Cycle deterministically (index % pool.length). |
| Disaster damages building on edge of grid | Clamp coordinates to grid bounds. |

## 10. New File: `services/localAIService.ts`

Shape (all functions synchronous, pure, parameterized by city state):

```ts
// Template pools
const GOAL_TEMPLATES = [...]   // { description template with {n} placeholders, targetType, targetValueFn, rewardFn, buildingType? }
const NEWS_TEMPLATES = [...]   // { text template, type, condition? }
const IDEA_TEMPLATES = DEFAULT_IDEAS  // seeded from existing DEFAULT_IDEAS
const MUSIC_TEMPLATES = [...] // MusicComposition objects

// Selector: pick + parameterize based on stats
export const generateCityGoal = (stats, grid) => { ... }
export const generateNewsEvent = (stats, recentAction) => { ... }
export const getOptimalLayout = (width, height) => { ... }  // existing random resource placement
export const generateFeatureIdeas = () => { ... }            // random pick from IDEA_TEMPLATES
export const generateIdeaDetails = (idea) => { ... }        // static template block
export const generateProceduralMusic = (prompt) => { ... }  // keyword match or random pick
```

Goal targets should scale with current stats (e.g., targetValue = stats.money * 2, capped) so goals are always achievable.

## 11. Gameplay Substance (curated from 45-game research)

Research basis: 45 games surveyed — Against the Storm, Frostpunk, Frostpunk 2, Cities: Skylines, Cities: Skylines 2, Sweet Transit, MEGACITY, MQ City Builder, City Architect, Turnbased City Builder, Manor Lords, Songs of Syx, Banished, Anno 1800, Surviving Mars, Tiny Glade, Flotsam, Fabledom, Airborne Kingdom, Terra Nil, Rimworld, Dwarf Fortress, They Are Billions, Northgard, New Cycle, The Wandering Village, Medieval Dynasty, Timberborn, Farthest Frontier, Foundation, Memoriapolis, Town to City, Wildwood, Settlements Rising, Synergy, CityPunk, City Defense Z, Hexfactory, Kingdoms Reborn, dotAGE, SteamWorld Build, Minami Lane, Going Medieval, Animal Town, Civilization VI.

### 11.1 Road Connectivity
- **Research basis:** Cities: Skylines (traffic-as-gameplay), Frostpunk (heat-by-proximity)
- **Design:** Buildings only produce income/population if connected to a road network. Flood-fill from any road tile to mark connected buildings. Unconnected buildings show a warning badge. Roads become a real constraint, not decoration.
- **Affected files:** `types.ts` (add `isConnected` to TileData), `App.tsx` (connectivity check in game loop), `components/IsoMap.tsx` (warning badge on unconnected buildings)

### 11.2 Pollution System
- **Research basis:** Turnbased City Builder (local vs global pollution), MQ City Builder (pollution −40% land value)
- **Design:** Industry generates pollution in a 3-tile radius. Pollution reduces happiness and land value. Visual: tiles tint brownish at high pollution. Pollution can trigger disaster events (fires).
- **Affected files:** `types.ts` (add `pollution` to TileData), `App.tsx` (pollution calc in game loop), `constants.tsx` (pollution values per building type)

### 11.3 Happiness/Welfare Metric (refined)
- **Research basis:** MQ City Builder, City Architect, MEGACITY, Frostpunk (hope/discontent)
- **Design:** New `happiness` stat (0–100). Formula: `happiness = 50 + parksPerCapita*10 + (10-taxRate)*2 - avgPollution*5 - crime*3`, clamped 0–100. Drives population growth multiplier (`growthMultiplier = happiness / 50`).
- **Crime model (simplified, no police buildings):** `crime = max(0, (population / 100) - (parksCount * 2) - (happiness / 20))`. Crime rises with population density and falls with parks and happiness. No dedicated police/fire/health buildings — keeps scope manageable.
- **Affected files:** `types.ts` (add `happiness` to CityStats), `App.tsx` (calc in game loop), `components/UIOverlay.tsx` (display happiness + crime indicators)

### 11.4 Disaster Events
- **Research basis:** Cities: Skylines (Natural Disasters DLC), Settlements Rising (raids), Rimworld (disasters shape story)
- **Design:** Pollution-gated. When total pollution exceeds threshold, roll for disaster: fires (from industry), floods (from water tiles), crime waves. Broadcast on news feed. Damaged buildings must be demolished. Goal system can include "survive 3 disasters" type goals.
- **Affected files:** `types.ts` (DisasterEvent type), `App.tsx` (disaster trigger in game loop), `components/UIOverlay.tsx` (disaster alert)

### 11.5 Tech/Unlock Tree
- **Research basis:** Farthest Frontier (142-point tree), Town to City (research tree), Wildwood (tech tree), Against the Storm (blueprints as rewards)
- **Design:** Reputation points earned from completed goals. Unlock buildings: Basic (free) → Roads/Parks (0 rep) → Residential/Commercial (0 rep) → Industrial (5 rep) → Extractor (10 rep) → Advanced (20 rep). Reputation persists across resets (stored in `sky_metropolis_reputation`).
- **Affected files:** `types.ts` (add reputation to separate storage), `constants.tsx` (unlock thresholds), `App.tsx` (unlock check), `components/UIOverlay.tsx` (locked building indicators)

### 11.6 Blueprints
- **Research basis:** Idea Lab already lists "Blueprint Sharing" as a future idea
- **Design:** Export/import district layouts as base64 of compressed JSON. Copy to clipboard / paste from clipboard. Low effort, high replay value.
- **Affected files:** `App.tsx` (export/import functions), `components/UIOverlay.tsx` (blueprint buttons)

## 12. Implementation Tasks (ordered)

1. Update `package.json` and remove cloud deps
2. Delete `geminiService.ts`, `openaiService.ts`
3. Create `localAIService.ts` with template pools for goals, news, terrain, ideas, music
4. Rewrite `aiService.ts` to call local functions (synchronous, no fetch)
5. Strip `server.ts` of all `/api/ai/*` routes
6. Clean `vite.config.ts`
7. Update `types.ts` with new type changes (§5)
8. Update `constants.tsx` with unlock thresholds, pollution values
9. Refactor `App.tsx` — remove AI key checks, integrate new game systems (§7), reputation persistence (§8), migration (§8)
10. Refactor `IdeaLab.tsx` — remove react-markdown, local generation, prompt textarea
11. Refactor `components/MusicLab.tsx` — remove Lyria mode, keep procedural
12. Refactor `components/UIOverlay.tsx`, `components/StartScreen.tsx`, `components/IsoMap.tsx` — UI cleanup + new stat displays
13. Update `.env.example` and `metadata.json`
14. Run `npm install && npm run lint`

## 13. Validation

- `npm install` succeeds
- `npm run lint` (tsc --noEmit) passes
- `npm run build` succeeds
- App runs without any env vars set
- No `@google/genai`, `openai`, or `react-markdown` imports remain anywhere in the codebase
- All 6 gameplay systems functional: road connectivity, pollution, happiness, disasters, tech tree, blueprints
- Reputation persists across game resets
- Old saves (without happiness/pollution/isConnected) load without errors

## 14. Resolved Questions (finalized)

1. **Disaster frequency** — pollution-gated. Disasters only trigger when total pollution exceeds a threshold.
2. **Blueprint format** — base64 of compressed JSON.
3. **Multiple settlements** — deferred to a future update. Single city for v1.