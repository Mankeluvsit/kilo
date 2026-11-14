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
| Randomness model | **Math.random** — consistent with the rest of the game (terrain, traffic, citizens already use it); no seed plumbing |
| Idea Lab | **B** — pure offline idea board; drop the prompt textarea, drop react-markdown; "Generate Concept" → "Add Random Idea" from pool |
| Music Lab | **A** — keep the lab, drop Lyria mode, keep Tone.js procedural playback with a local template-pool composer |
| UI direction | **B** — keep indigo/sky-900 palette and component structure; strip slop only (gradient text, scanline overlays, emoji icons, stacked glass panels, lucide-only icons) |
| Gameplay substance | **All 6 systems** (road connectivity, pollution, happiness, disasters, tech/unlock tree, blueprints) |
| Tech/unlock gating | **Reputation that persists across resets** (Against the Storm style — meta-progression) |

### UI slop to remove specifically
- Gradient text on the "SkyMetropolis" title (`bg-gradient-to-br ... bg-clip-text text-transparent`)
- CRT scanline overlay divs in `IdeaLab.tsx` and `UIOverlay.tsx` (the `bg-[linear-gradient...]` fixed divs)
- Emoji-as-icons in the toolbar (`⛏️`, `✕`, and the `⚡💧💎` status badges) — replace with lucide-react icons
- Stacked translucent panels: `StartScreen.tsx` sits over a glass panel over the canvas — make it a single solid surface
- `react-markdown` usage — replaced with static JSX detail blocks

### Deferred-to-my-judgment decisions (recorded, not asked)
- StartScreen "AI Advisor" toggle stays, relabeled to "City Advisor (quests & news)". It gates `aiEnabled` throughout the game loop, so removing it would require plumbing. Keeping it preserves a real gameplay choice at zero cost.
- `analyzePerformance` — the existing `getFallbackRecommendation` heuristic in `IsoMap.tsx` already exists. Wire it directly; no AI call needed.
- `window.aistudio` global removed from `types.ts` (only used by the now-deleted Lyria key flow).
- `metadata.json` description updated from "fulfill tasks provided by Gemini" to "fulfill tasks provided by the City Advisor".

## 3. Security Bug Found

`vite.config.ts` inlines `GEMINI_API_KEY` and `OPENAI_API_KEY` into the client bundle via `define`. `openaiService.ts` uses `dangerouslyAllowBrowser: true`. Both keys ship to the browser. Removing both providers eliminates this entirely.

## 4. Affected Files

### Delete
- `services/geminiService.ts`
- `services/openaiService.ts`

### Modify
- `package.json` — remove `@google/genai`, `openai`, `react-markdown`, `@types/express` (keep `express` for static serving)
- `vite.config.ts` — remove the `define` blocks for API keys
- `server.ts` — strip all `/api/ai/*` routes; keep `/api/health` and static serving
- `services/aiService.ts` — replace fetch calls with local functions
- `services/localAIService.ts` — **new** template library + selector
- `App.tsx` — remove AI key checks, simplify, add reputation stat + persistence
- `IdeaLab.tsx` — remove react-markdown, local generation
- `components/MusicLab.tsx` — remove Lyria mode, keep procedural
- `components/UIOverlay.tsx`, `components/StartScreen.tsx`, `components/IsoMap.tsx` — UI cleanup
- `.env.example` — remove API key lines
- `metadata.json` — update description

## 5. New File: `services/localAIService.ts`

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
export const generateFeatureIdeas = (prompt) => { ... }     // keyword match or random pick
export const generateIdeaDetails = (idea) => { ... }        // static template block
export const generateProceduralMusic = (prompt) => { ... }  // keyword match or random pick
```

Goal targets should scale with current stats (e.g., targetValue = stats.money * 2, capped) so goals are always achievable.

## 6. Implementation Tasks (AI + UI layer)

1. Update `package.json` and remove cloud deps
2. Delete `geminiService.ts`, `openaiService.ts`
3. Create `localAIService.ts` with template pools for goals, news, terrain, ideas, music
4. Rewrite `aiService.ts` to call local functions (synchronous, no fetch)
5. Strip `server.ts` of all `/api/ai/*` routes
6. Clean `vite.config.ts`
7. Refactor UI components (anti-slop)
8. Update `.env.example` and `metadata.json`
9. Run `npm install && npm run lint`

## 7. Validation (AI + UI layer)

- `npm install` succeeds
- `npm run lint` (tsc --noEmit) passes
- `npm run build` succeeds
- App runs without any env vars set
- No `@google/genai`, `openai`, or `react-markdown` imports remain anywhere in the codebase

## 8. Gameplay Substance (curated from 45-game research)

Research basis: 45 games surveyed — Against the Storm, Frostpunk, Frostpunk 2, Cities: Skylines, Cities: Skylines 2, Sweet Transit, MEGACITY, MQ City Builder, City Architect, Turnbased City Builder, Manor Lords, Songs of Syx, Banished, Anno 1800, Surviving Mars, Tiny Glade, Flotsam, Fabledom, Airborne Kingdom, Terra Nil, Rimworld, Dwarf Fortress, They Are Billions, Northgard, New Cycle, The Wandering Village, Medieval Dynasty, Timberborn, Farthest Frontier, Foundation, Memoriapolis, Town to City, Wildwood, Settlements Rising, Synergy, CityPunk, City Defense Z, Hexfactory, Kingdoms Reborn, dotAGE, SteamWorld Build, Minami Lane, Going Medieval, Animal Town, Civilization VI.

### 8.1 Road Connectivity
- **Research basis:** Cities: Skylines (traffic-as-gameplay), Frostpunk (heat-by-proximity)
- **Design:** Buildings only produce income/population if connected to a road network. Flood-fill from any road tile to mark connected buildings. Unconnected buildings show a warning badge. Roads become a real constraint, not decoration.
- **Affected files:** `types.ts` (add `isConnected` to TileData), `App.tsx` (connectivity check in game loop), `components/IsoMap.tsx` (warning badge on unconnected buildings)

### 8.2 Pollution System
- **Research basis:** Turnbased City Builder (local vs global pollution), MQ City Builder (pollution −40% land value)
- **Design:** Industry generates pollution in a 3-tile radius. Pollution reduces happiness and land value. Visual: tiles tint brownish at high pollution. Pollution can trigger disaster events (fires).
- **Affected files:** `types.ts` (add `pollution` to TileData), `App.tsx` (pollution calc in game loop), `constants.tsx` (pollution values per building type)

### 8.3 Happiness/Welfare Metric
- **Research basis:** MQ City Builder, City Architect, MEGACITY, Frostpunk (hope/discontent)
- **Design:** New `happiness` stat (0–100). Formula: `happiness = 50 + parksPerCapita*10 + (10-taxRate)*2 - pollution*5 - crime*3`. Drives population growth multiplier. Displayed in UI.
- **Affected files:** `types.ts` (add `happiness` to CityStats), `App.tsx` (calc in game loop), `components/UIOverlay.tsx` (display)

### 8.4 Disaster Events
- **Research basis:** Cities: Skylines (Natural Disasters DLC), Settlements Rising (raids), Rimworld (disasters shape story)
- **Design:** Periodic events: fires (from industry, random), floods (from water tiles), crime waves. Broadcast on news feed. Damaged buildings must be demolished. Goal system can include "survive 3 disasters" type goals.
- **Affected files:** `types.ts` (DisasterEvent type), `App.tsx` (disaster trigger in game loop), `components/UIOverlay.tsx` (disaster alert)

### 8.5 Tech/Unlock Tree
- **Research basis:** Farthest Frontier (142-point tree), Town to City (research tree), Wildwood (tech tree), Against the Storm (blueprints as rewards)
- **Design:** Reputation points earned from completed goals. Unlock buildings: Basic (free) → Roads/Parks (0 rep) → Residential/Commercial (0 rep) → Industrial (5 rep) → Extractor (10 rep) → Advanced (20 rep). Reputation persists across resets (stored separately from save).
- **Affected files:** `types.ts` (add `reputation` to separate storage), `constants.tsx` (unlock thresholds), `App.tsx` (unlock check), `components/UIOverlay.tsx` (locked building indicators)

### 8.6 Blueprints
- **Research basis:** Idea Lab already lists "Blueprint Sharing" as a future idea
- **Design:** Export/import district layouts as base64 text strings. Low effort, high replay value. Copy to clipboard / paste from clipboard.
- **Affected files:** `App.tsx` (export/import functions), `components/UIOverlay.tsx` (blueprint buttons)

## 9. Implementation Tasks (full scope)

1. Update `package.json` and remove cloud deps
2. Delete `geminiService.ts`, `openaiService.ts`
3. Create `localAIService.ts` with template pools for goals, news, terrain, ideas, music
4. Rewrite `aiService.ts` to call local functions (synchronous, no fetch)
5. Strip `server.ts` of all `/api/ai/*` routes
6. Clean `vite.config.ts`
7. Refactor UI components (anti-slop)
8. Update `.env.example` and `metadata.json`
9. **Add reputation stat + persistence** (separate localStorage key `sky_metropolis_reputation`)
10. **Add road connectivity check** (flood-fill, warning badges)
11. **Add pollution system** (radius calc, tile tinting, happiness penalty)
12. **Add happiness metric** (formula, UI display, growth multiplier)
13. **Add disaster events** (trigger logic, news broadcast, damage, goal types)
14. **Add tech/unlock tree** (unlock thresholds, locked indicators)
15. **Add blueprints** (export/import, clipboard)
16. Run `npm install && npm run lint`

## 10. Validation (full scope)

- `npm install` succeeds
- `npm run lint` (tsc --noEmit) passes
- `npm run build` succeeds
- App runs without any env vars set
- No `@google/genai`, `openai`, or `react-markdown` imports remain anywhere in the codebase
- All 6 gameplay systems functional: road connectivity, pollution, happiness, disasters, tech tree, blueprints
- Reputation persists across game resets

## 11. Open Questions (still unresolved)

1. **Disaster frequency** — how often should disasters trigger? (e.g., 5% chance per tick, or only when pollution is high?) Recommendation: pollution-gated — disasters only trigger when total pollution exceeds a threshold, making industry placement a real strategic choice.
2. **Blueprint format** — base64 of the grid JSON, or a compact custom encoding? Recommendation: base64 of compressed JSON (small enough for clipboard, human-readable-ish).
3. **Multiple settlements** — should the player be able to build multiple towns (Against the Storm style)? Recommendation: defer to a future update — single city is enough for v1 of this revamp.