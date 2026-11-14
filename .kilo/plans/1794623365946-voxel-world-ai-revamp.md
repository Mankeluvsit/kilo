# Voxel-world Revamp Plan

## 1. Context & Problem

`Mankeluvsit/Voxel-world` ("Sky Metropolis") is an isometric 3D city builder (React + Three.js + Vite + Express). Its AI layer currently depends on two live cloud providers:

- **Gemini** — `@google/genai`, model `gemini-3.5-flash` (goals, news, terrain layout, Idea Lab, procedural music, Lyria audio, performance/LOD analysis).
- **OpenAI** — `openai`, model `gpt-4o-mini`, switched via `VITE_AI_PROVIDER`.

Goal: remove both cloud providers, replace the entire AI layer with a local deterministic "fake AI" (template library + selector), and do a full refresh of the app — no API keys required, zero token cost. Additionally, add real gameplay substance (6 interlocking systems + new mechanics) so the city builder has depth, not just a visual toy.

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
- `constants.tsx` — unlock thresholds, pollution values, new building configs (see §5b)
- `App.tsx` — remove AI key checks, integrate new game systems, reputation persistence
- `IdeaLab.tsx` — remove react-markdown, local generation, prompt textarea
- `components/MusicLab.tsx` — remove Lyria mode, keep procedural
- `components/UIOverlay.tsx`, `components/StartScreen.tsx`, `components/IsoMap.tsx` — UI cleanup + new stat displays
- `.env.example` — remove API key lines
- `metadata.json` — update description

## 5c. Full BUILDINGS Record Update (`constants.tsx`)

The `BUILDINGS` record gains `pollution` and `upkeep` fields. `unlockRep` is a separate `UNLOCK_REP` map (see §5b).

```ts
export const BUILDINGS: Record<BuildingType, BuildingConfig> = {
  [BuildingType.None]: {
    type: BuildingType.None, cost: 0, name: 'Bulldoze', description: 'Clear a tile',
    color: '#ef4444', popGen: 0, incomeGen: 0, pollution: 0, upkeep: 0,
  },
  [BuildingType.Road]: {
    type: BuildingType.Road, cost: 10, name: 'Road', description: 'Connects buildings.',
    color: '#374151', popGen: 0, incomeGen: 0, pollution: 0, upkeep: 1,
  },
  [BuildingType.Residential]: {
    type: BuildingType.Residential, cost: 100, name: 'House', description: '+5 Pop/day',
    color: '#f87171', popGen: 5, incomeGen: 0, pollution: 0, upkeep: 2,
  },
  [BuildingType.Commercial]: {
    type: BuildingType.Commercial, cost: 200, name: 'Shop', description: '+$15/day',
    color: '#60a5fa', popGen: 0, incomeGen: 15, pollution: 1, upkeep: 3,
  },
  [BuildingType.Industrial]: {
    type: BuildingType.Industrial, cost: 400, name: 'Factory', description: '+$40/day',
    color: '#facc15', popGen: 0, incomeGen: 40, pollution: 5, upkeep: 5,
  },
  [BuildingType.Park]: {
    type: BuildingType.Park, cost: 50, name: 'Park', description: 'Looks nice.',
    color: '#4ade80', popGen: 1, incomeGen: 0, pollution: -2, upkeep: 1,
  },
  [BuildingType.Extractor]: {
    type: BuildingType.Extractor, cost: 300, name: 'Extractor', description: 'Build on resource node.',
    color: '#fb923c', popGen: 0, incomeGen: 0, pollution: 0, upkeep: 4,
  },
};

// BuildingConfig interface update (types.ts):
export interface BuildingConfig {
  type: BuildingType;
  cost: number;
  name: string;
  description: string;
  color: string;
  popGen: number;
  incomeGen: number;
  pollution: number;   // NEW
  upkeep: number;      // NEW
}
```

## 5d. Helper Functions (App.tsx)

```ts
const countBuildings = (grid: Grid, type?: BuildingType): number => {
  if (type) return grid.flat().filter(t => t.buildingType === type).length;
  return grid.flat().filter(t => t.buildingType !== BuildingType.None).length;
};

const avgPollution = (grid: Grid): number => {
  const tiles = grid.flat();
  const total = tiles.reduce((sum, t) => sum + (t.pollution || 0), 0);
  return total / tiles.length;
};

const totalPollution = (grid: Grid): number => {
  return grid.flat().reduce((sum, t) => sum + (t.pollution || 0), 0);
};

const isConnectedFloodFill = (grid: Grid, gridSize: number): boolean[][] => {
  const connected = Array.from({ length: gridSize }, () => Array(gridSize).fill(false));
  const queue: [number, number][] = [];
  // Seed: all road tiles adjacent to any building
  grid.flat().forEach(tile => {
    if (tile.buildingType === BuildingType.Road) {
      // Check if any neighbor is a non-road building
      const neighbors = [[-1,0],[1,0],[0,-1],[0,1]];
      for (const [dx, dy] of neighbors) {
        const nx = tile.x + dx, ny = tile.y + dy;
        if (nx >= 0 && nx < gridSize && ny >= 0 && ny < gridSize) {
          const neighbor = grid[ny][nx];
          if (neighbor.buildingType !== BuildingType.None && neighbor.buildingType !== BuildingType.Road) {
            queue.push([tile.x, tile.y]);
            break;
          }
        }
      }
    }
  });
  // BFS flood-fill through roads
  while (queue.length > 0) {
    const [x, y] = queue.shift()!;
    if (connected[y][x]) continue;
    connected[y][x] = true;
    const neighbors = [[-1,0],[1,0],[0,-1],[0,1]];
    for (const [dx, dy] of neighbors) {
      const nx = x + dx, ny = y + dy;
      if (nx >= 0 && nx < gridSize && ny >= 0 && ny < gridSize && !connected[ny][nx]) {
        if (grid[ny][nx].buildingType === BuildingType.Road) {
          queue.push([nx, ny]);
        }
      }
    }
  }
  // Mark buildings as connected if adjacent to a connected road
  grid.flat().forEach(tile => {
    if (tile.buildingType !== BuildingType.None && tile.buildingType !== BuildingType.Road) {
      const neighbors = [[-1,0],[1,0],[0,-1],[0,1]];
      for (const [dx, dy] of neighbors) {
        const nx = tile.x + dx, ny = tile.y + dy;
        if (nx >= 0 && nx < gridSize && ny >= 0 && ny < gridSize && connected[ny][nx]) {
          // Mark this tile's connected status via the grid
          break;
        }
      }
    }
  });
  return connected;
};
```

## 5e. `generateIdeaDetails` Static Template

```ts
export const generateIdeaDetails = (idea: Idea): string => {
  return `## ${idea.title}\n\n**Category:** ${idea.category} | **Impact:** ${idea.impact}/10 | **Difficulty:** ${idea.difficulty}\n\n### Technical Approach\n- Implement using React + Three.js components\n- Add state management in App.tsx\n- Update types.ts and constants.tsx\n\n### Key Components\n- New component or modification to existing\n- Service layer if needed\n\n### Potential Challenges\n- Performance impact on 3D rendering\n- State synchronization with game loop\n\n### Quick Start\n\`\`\`tsx\n// Add to App.tsx state\nconst [newFeature, setNewFeature] = useState(false);\n\`\`\``;
};
```

This replaces the AI-generated markdown with a consistent 4-section template. No react-markdown needed.

```ts
export interface TileData {
  x: number;
  y: number;
  buildingType: BuildingType;
  variant?: number;
  resourceType?: 'Energy' | 'Water' | 'Mineral';
  isPowered?: boolean;
  hasWater?: boolean;
  isIncomeGenerating?: boolean;
  isConnected?: boolean;   // NEW
  pollution?: number;       // NEW
}

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

export interface DisasterEvent {
  id: string;
  type: 'fire' | 'flood' | 'crime_wave';
  x: number;
  y: number;
  radius: number;
  day: number;
  resolved: boolean;
}

// Remove from global Window: aistudio (no longer used)
```

## 5b. Constants Changes (`constants.tsx`)

```ts
// Pollution values added to BUILDINGS record
[BuildingType.Industrial]: { ..., pollution: 5 },
[BuildingType.Commercial]: { ..., pollution: 1 },
[BuildingType.Residential]: { ..., pollution: 0 },
[BuildingType.Park]: { ..., pollution: -2 },
[BuildingType.Road]: { ..., pollution: 0 },
[BuildingType.Extractor]: { ..., pollution: 0 },
[BuildingType.None]: { ..., pollution: 0 },

export const UNLOCK_REP = {
  [BuildingType.Road]: 0,
  [BuildingType.Residential]: 0,
  [BuildingType.Commercial]: 0,
  [BuildingType.Park]: 0,
  [BuildingType.Industrial]: 5,
  [BuildingType.Extractor]: 10,
};

export const POLLUTION_RADIUS = 3;
export const DISASTER_POLLUTION_THRESHOLD = 50;
export const DISASTER_CHANCE = 0.15;
export const HAPPINESS_MIN = 0;
export const HAPPINESS_MAX = 100;
export const INITIAL_HAPPINESS = 50;
export const INITIAL_REPUTATION = 0;
export const REPUTATION_PER_GOAL = 3;
```

## 6. `aiService.ts` API Contract (after refactor)

All functions become synchronous. Callers in `App.tsx` and `IdeaLab.tsx` must drop `await` and null-checks where the return type is no longer nullable.

```ts
generateCityGoal(stats, grid): Promise<AIGoal | null>  →  AIGoal
generateNewsEvent(stats, action): Promise<NewsItem | null>  →  NewsItem | null
getOptimalLayout(w, h): Promise<ResourceNode[]>  →  ResourceNode[]
generateFeatureIdeas(): Promise<Partial<Idea> | null>  →  Partial<Idea>
generateIdeaDetails(idea): Promise<string | null>  →  string
generateProceduralMusic(prompt): Promise<MusicComposition | null>  →  MusicComposition
```

## 7. Data Flow (game loop integration)

The existing `setInterval` in `App.tsx` (TICK_RATE_MS = 2000ms) is the single integration point. New systems calculated in this order each tick:

1. **Resource production** — existing (extractors yield energy/water/minerals)
2. **Road connectivity** — flood-fill from road tiles; mark `isConnected` on each tile
3. **Pollution** — industry tiles add pollution to 3-tile radius; accumulate per-tile pollution values
4. **Building yields** — existing logic, but gated by `isConnected` (unconnected buildings produce nothing)
5. **Happiness** — `50 + parksPerCapita*10 + (10-taxRate)*2 - avgPollution*5 - crime*3`, clamped 0–100
6. **Crime** — `max(0, (population/100) - (parksCount*2) - (happiness/20))`
7. **Population growth** — existing logic, multiplied by `happiness/50`
8. **Disaster check** — if total pollution > threshold, roll for disaster (fire/flood/crime wave). On trigger: damage buildings, broadcast news, add to disaster feed.
9. **Goal check** — existing logic, using local `generateCityGoal`
10. **News** — existing logic, using local `generateNewsEvent`
11. **Reputation** — when a goal completes, add reputation points (separate from money reward)
12. **Save** — existing localStorage auto-save, plus separate `sky_metropolis_reputation` key

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

## 10b. New File: `services/soundService.ts` (detailed)

Procedural SFX using Web Audio API (no asset files). Each function creates an oscillator + gain node, plays a short sound, and cleans up.

```ts
// Sound types and their parameters
const SOUNDS = {
  place: { freq: 440, duration: 0.1, type: 'sine', gain: 0.3 },
  bulldoze: { freq: 220, duration: 0.2, type: 'sawtooth', gain: 0.2, slide: -100 },
  disaster: { freq: 150, duration: 0.5, type: 'square', gain: 0.4, slide: -50 },
  goalComplete: { freq: 523, duration: 0.3, type: 'sine', gain: 0.4, arpeggio: [523, 659, 784] },
  tick: { freq: 800, duration: 0.05, type: 'sine', gain: 0.1 },
  upgrade: { freq: 660, duration: 0.2, type: 'sine', gain: 0.3, arpeggio: [660, 880] },
  achievement: { freq: 880, duration: 0.4, type: 'sine', gain: 0.4, arpeggio: [880, 1100, 1320] },
};

export const playSound = (sound: keyof typeof SOUNDS) => { ... }
export const setMuted = (muted: boolean) => { ... }
export const isMuted = () => boolean;
```

- All sounds respect a global mute toggle (stored in localStorage `sky_metropolis_muted`).
- Mute toggle button in UIOverlay toolbar.
- Sounds are triggered from App.tsx (place, bulldoze, disaster, goal complete, tick) and IsoMap.tsx (upgrade).

## 10c. New File: `services/achievementService.ts` (detailed)

localStorage-based milestone tracking. Achievements are checked after each game tick and on specific events.

```ts
const ACHIEVEMENTS = [
  { id: 'first_pop', title: 'First Citizens', description: 'Reach 10 population', check: (stats) => stats.population >= 10 },
  { id: 'first_100_pop', title: 'Growing Town', description: 'Reach 100 population', check: (stats) => stats.population >= 100 },
  { id: 'first_1k_money', title: 'Treasury Full', description: 'Accumulate $1,000', check: (stats) => stats.money >= 1000 },
  { id: 'first_10k_money', title: 'Wealthy City', description: 'Accumulate $10,000', check: (stats) => stats.money >= 10000 },
  { id: 'first_50_buildings', title: 'Urban Sprawl', description: 'Build 50 buildings', check: (stats, grid) => countBuildings(grid) >= 50 },
  { id: 'first_road', title: 'Connected', description: 'Build your first road', check: (stats, grid) => countBuildings(grid, BuildingType.Road) >= 1 },
  { id: 'first_disaster', title: 'Trial by Fire', description: 'Survive your first disaster', check: (stats, grid, disasters) => disasters.length >= 1 },
  { id: 'survive_3_disasters', title: 'Resilient', description: 'Survive 3 disasters', check: (stats, grid, disasters) => disasters.filter(d => d.resolved).length >= 3 },
  { id: 'first_goal', title: 'Goal Oriented', description: 'Complete your first goal', check: (stats, grid, disasters, goals) => goals >= 1 },
  { id: 'first_5_goals', title: 'Overachiever', description: 'Complete 5 goals', check: (stats, grid, disasters, goals) => goals >= 5 },
  { id: 'first_upgrade', title: 'Up and Coming', description: 'Upgrade your first building', check: (stats, grid, disasters, goals, upgrades) => upgrades >= 1 },
  { id: 'happiness_80', title: 'Happy City', description: 'Reach 80% happiness', check: (stats) => stats.happiness >= 80 },
  { id: 'first_blueprint', title: 'Architect', description: 'Export your first blueprint', check: (stats, grid, disasters, goals, upgrades, blueprints) => blueprints >= 1 },
  { id: 'day_100', title: 'Centennial', description: 'Reach day 100', check: (stats) => stats.day >= 100 },
];

export const checkAchievements = (stats, grid, disasters, goalsCompleted, upgrades, blueprints) => Achievement[];
export const getUnlockedAchievements = () => string[];
export const isAchievementUnlocked = (id: string) => boolean;
```

- Achievements stored in `localStorage` (`sky_metropolis_achievements`).
- Achievement popup shown in UIOverlay for 3 seconds on unlock.
- Achievement panel accessible from toolbar.

Shape (all functions synchronous, pure, parameterized by city state). Template pools must contain at least the following:

### Goal Templates (minimum 8)
```ts
const GOAL_TEMPLATES = [
  { description: "Expand the residential district to house {n} citizens.", targetType: 'population', targetValueFn: (stats) => stats.population + 50, rewardFn: (stats) => 300 + stats.day * 10, rewardRep: 1 },
  { description: "Accumulate ${n} in the city treasury.", targetType: 'money', targetValueFn: (stats) => stats.money + 2000, rewardFn: () => 500, rewardRep: 1 },
  { description: "Build {n} commercial zones to boost the economy.", targetType: 'building_count', buildingType: BuildingType.Commercial, targetValueFn: (stats, grid) => countBuildings(grid, BuildingType.Commercial) + 3, rewardFn: () => 400, rewardRep: 1 },
  { description: "Create a green city with {n} parks.", targetType: 'building_count', buildingType: BuildingType.Park, targetValueFn: (stats, grid) => countBuildings(grid, BuildingType.Park) + 3, rewardFn: () => 250, rewardRep: 1 },
  { description: "Reach a population of {n} citizens.", targetType: 'population', targetValueFn: (stats) => stats.population + 100, rewardFn: () => 600, rewardRep: 2 },
  { description: "Build {n} roads to connect the city.", targetType: 'building_count', buildingType: BuildingType.Road, targetValueFn: (stats, grid) => countBuildings(grid, BuildingType.Road) + 5, rewardFn: () => 200, rewardRep: 1 },
  { description: "Survive 3 disaster events.", targetType: 'disaster_count', targetValue: 3, rewardFn: () => 800, rewardRep: 2 },
  { description: "Reach happiness level {n}%.", targetType: 'happiness', targetValueFn: (stats) => Math.min(100, stats.happiness + 20), rewardFn: () => 400, rewardRep: 1 },
];
```

### News Templates (minimum 12)
```ts
const NEWS_TEMPLATES = [
  { text: "Citizens report seeing strange lights in the sky.", type: 'neutral' },
  { text: "Local park voted 'Best place to sit' by local pigeon.", type: 'positive' },
  { text: "Traffic congestion reported near the city center.", type: 'negative', condition: (stats) => stats.population > 50 },
  { text: "New mayor promises more clouds, less rain.", type: 'neutral' },
  { text: "Economic forecast: Sunny with a chance of taxes.", type: 'neutral' },
  { text: "Factory workers demand better working conditions.", type: 'negative', condition: (stats, grid) => countBuildings(grid, BuildingType.Industrial) > 0 },
  { text: "City population reaches new milestone!", type: 'positive', condition: (stats) => stats.population > 100 },
  { text: "Pollution levels rising in industrial district.", type: 'negative', condition: (stats, grid) => avgPollution(grid) > 2 },
  { text: "New park opened to great fanfare.", type: 'positive', condition: (stats, grid) => countBuildings(grid, BuildingType.Park) > 0 },
  { text: "Roads in poor condition, citizens complain.", type: 'negative', condition: (stats, grid) => countBuildings(grid, BuildingType.Road) < stats.population / 20 },
  { text: "City treasury overflowing with surplus funds.", type: 'positive', condition: (stats) => stats.money > 5000 },
  { text: "Citizens celebrate another beautiful day.", type: 'positive' },
];
```

### Music Templates (minimum 4)
```ts
const MUSIC_TEMPLATES = [
  { tempo: 120, scale: "C major", synthType: "amsynth", notes: [ /* 4-bar loop */ ] },
  { tempo: 90, scale: "A minor", synthType: "plucksynth", notes: [ /* 4-bar loop */ ] },
  { tempo: 140, scale: "G major", synthType: "fmsynth", notes: [ /* 4-bar loop */ ] },
  { tempo: 100, scale: "D minor", synthType: "duosynth", notes: [ /* 4-bar loop */ ] },
];
```

### Selector Logic
- `generateCityGoal(stats, grid)`: filter GOAL_TEMPLATES by condition (if any), pick random, parameterize {n} with targetValueFn, return AIGoal.
- `generateNewsEvent(stats, recentAction)`: filter NEWS_TEMPLATES by condition (if any), pick random, return NewsItem. 2% chance per tick (existing behavior).
- `getOptimalLayout(width, height)`: existing random resource placement (Energy/Water/Mineral, ~40% of tiles).
- `generateFeatureIdeas()`: random pick from IDEA_TEMPLATES (existing DEFAULT_IDEAS).
- `generateIdeaDetails(idea)`: return a static template string with the idea's title, category, and a 4-bullet implementation outline.
- `generateProceduralMusic(prompt)`: keyword match against MUSIC_TEMPLATES by synthType/scale keywords, or random pick if no match.

Goal targets should scale with current stats so goals are always achievable.

## 11. Gameplay Substance (curated from 45-game research)

Research basis: 45 games surveyed — Against the Storm, Frostpunk, Frostpunk 2, Cities: Skylines, Cities: Skylines 2, Sweet Transit, MEGACITY, MQ City Builder, City Architect, Turnbased City Builder, Manor Lords, Songs of Syx, Banished, Anno 1800, Surviving Mars, Tiny Glade, Flotsam, Fabledom, Airborne Kingdom, Terra Nil, Rimworld, Dwarf Fortress, They Are Billions, Northgard, New Cycle, The Wandering Village, Medieval Dynasty, Timberborn, Farthest Frontier, Foundation, Memoriapolis, Town to City, Wildwood, Settlements Rising, Synergy, CityPunk, City Defense Z, Hexfactory, Kingdoms Reborn, dotAGE, SteamWorld Build, Minami Lane, Going Medieval, Animal Town, Civilization VI.

### 11.1 Road Connectivity
- **Research basis:** Cities: Skylines (traffic-as-gameplay), Frostpunk (heat-by-proximity)
- **Design:** Buildings only produce income/population if connected to a road network. Flood-fill from any road tile. Unconnected buildings show a warning badge.
- **Affected files:** `types.ts`, `App.tsx`, `components/IsoMap.tsx`

### 11.2 Pollution System
- **Research basis:** Turnbased City Builder (local vs global pollution), MQ City Builder (pollution −40% land value)
- **Design:** Industry generates pollution in a 3-tile radius. Pollution reduces happiness and land value. Tiles tint brownish at high pollution. Pollution can trigger disaster events.
- **Affected files:** `types.ts`, `App.tsx`, `constants.tsx`

### 11.3 Happiness/Welfare Metric (refined)
- **Research basis:** MQ City Builder, City Architect, MEGACITY, Frostpunk (hope/discontent)
- **Design:** New `happiness` stat (0–100). Formula: `happiness = 50 + parksPerCapita*10 + (10-taxRate)*2 - avgPollution*5 - crime*3`, clamped 0–100. Drives population growth multiplier (`growthMultiplier = happiness / 50`).
- **Crime model (simplified, no police buildings):** `crime = max(0, (population / 100) - (parksCount * 2) - (happiness / 20))`. Crime rises with population density and falls with parks and happiness.
- **Affected files:** `types.ts`, `App.tsx`, `components/UIOverlay.tsx`

### 11.4 Disaster Events
- **Research basis:** Cities: Skylines (Natural Disasters DLC), Settlements Rising (raids), Rimworld (disasters shape story)
- **Design:** Pollution-gated. When total pollution exceeds threshold (50), roll 15% per tick for disaster: fires (from industry), floods (from water tiles), crime waves. Broadcast on news feed. Damaged buildings must be demolished. Goal system can include "survive 3 disasters" type goals.
- **Affected files:** `types.ts`, `App.tsx`, `components/UIOverlay.tsx`

### 11.5 Tech/Unlock Tree
- **Research basis:** Farthest Frontier (142-point tree), Town to City (research tree), Wildwood (tech tree), Against the Storm (blueprints as rewards)
- **Design:** Reputation points earned from completed goals. Unlock buildings: Basic (free) → Roads/Parks (0 rep) → Residential/Commercial (0 rep) → Industrial (5 rep) → Extractor (10 rep) → Advanced (20 rep). Reputation persists across resets (stored in `sky_metropolis_reputation`).
- **Affected files:** `types.ts`, `constants.tsx`, `App.tsx`, `components/UIOverlay.tsx`

### 11.6 Blueprints
- **Research basis:** Idea Lab already lists "Blueprint Sharing" as a future idea
- **Design:** Export/import district layouts as base64 of compressed JSON. Copy to clipboard / paste from clipboard. Low effort, high replay value.
- **Affected files:** `App.tsx`, `components/UIOverlay.tsx`

## 11.7 New Gameplay & Mechanics (beyond requested scope)

Curated from gap analysis of the current codebase — systems that are missing or underdeveloped:

### A. Sound Effects (Web Audio API, no new deps)
- **What's missing:** Zero audio feedback. Building placement, bulldoze, disaster, goal complete, tick — all silent.
- **Design:** Procedural SFX using Web Audio API (oscillator + gain nodes). Building place (short blip), bulldoze (descending tone), disaster (alarm), goal complete (fanfare), tick (subtle click). No asset files needed.
- **Affected files:** `App.tsx` (trigger calls), new `services/soundService.ts`

### B. Undo/Redo (grid snapshot stack)
- **What's missing:** No undo. Bulldoze is the only reversal. Accidental placement is permanent.
- **Design:** Stack of grid snapshots (max 20). Ctrl+Z / Ctrl+Y keyboard shortcuts. Undo button in toolbar. Each placement pushes a snapshot.
- **Affected files:** `App.tsx` (undo stack), `components/UIOverlay.tsx` (undo/redo buttons)

### C. Statistics/History (population, money, happiness per day)
- **What's missing:** No charts or history. City builders love data visualization.
- **Design:** Track `population`, `money`, `happiness` per day in an array (max 100 entries). Display as a simple line chart in a collapsible panel. Use HTML canvas or SVG (no new deps).
- **Affected files:** `App.tsx` (tracking), `components/UIOverlay.tsx` (chart panel)

### D. Day/Night Cycle (visual lighting)
- **What's missing:** Day counter exists but lighting is static. No visual time progression.
- **Design:** `day % 24` maps to hour. Modify ambient light intensity and directional light color in IsoMap. Dawn (warm), day (bright), dusk (orange), night (dark blue). Subtle — doesn't affect gameplay.
- **Affected files:** `components/IsoMap.tsx` (lighting based on day prop)

### E. Building Upgrades (tier system)
- **What's missing:** Buildings are static. No upgrade path. Residential is always a house.
- **Design:** Tier system: House → Apartment → Tower (cost + reputation). Shop → Mall. Factory → Plant. Upgrade button on selected building. Costs money + reputation.
- **Affected files:** `types.ts` (add `tier` to TileData), `constants.tsx` (upgrade costs), `App.tsx` (upgrade logic), `components/UIOverlay.tsx` (upgrade button)

### F. Achievements (milestone tracking)
- **What's missing:** No milestone tracking beyond goals. No long-term objectives.
- **Design:** localStorage-based achievement list (first 100 pop, first $10k, 50 buildings, survive 3 disasters, etc.). Achievement popup on unlock. Achievement panel in UI.
- **Affected files:** new `services/achievementService.ts`, `App.tsx` (trigger checks), `components/UIOverlay.tsx` (achievement panel + popup)

### G. Minimap (2D canvas overlay)
- **What's missing:** Large grids (60x60) are hard to navigate. No overview.
- **Design:** Small 120x120 canvas in corner showing building types as colored dots. Click to pan camera. Updates on grid change.
- **Affected files:** `components/IsoMap.tsx` (minimap canvas), `App.tsx` (pass grid)

### H. Maintenance Costs (upkeep system)
- **What's missing:** Buildings cost nothing to maintain. No upkeep. Economy is one-directional (income only).
- **Design:** Each building type has a daily upkeep cost deducted from treasury. If treasury can't cover upkeep, buildings become unpowered (existing `isPowered` flag). Adds economic pressure.
- **Affected files:** `constants.tsx` (upkeep values), `App.tsx` (deduct in game loop)

### I. Tutorial Overlay (first-run guidance)
- **What's missing:** New players have no guidance. StartScreen is minimal.
- **Design:** Step-by-step overlay on first game: "Place a road", "Build a house", "Connect to road", "Watch citizens move in". Dismissible. Stored in localStorage (`sky_metropolis_tutorial_done`).
- **Affected files:** `components/StartScreen.tsx` or new `components/Tutorial.tsx`, `App.tsx` (trigger)

### J. Keyboard Shortcuts
- **What's missing:** No keyboard shortcuts. Everything is mouse-driven.
- **Design:** Number keys 1-7 select tools. B = bulldoze. U = undo. R = rotate. Space = pause. Escape = deselect. Display shortcut hints in toolbar.
- **Affected files:** `App.tsx` (keydown listener), `components/UIOverlay.tsx` (shortcut hints)

## 11.8 Shortcomings Analysis (current plan)

1. **Crime is undefined** — happiness formula references `crime*3` but crime has no definition. **Fixed:** added simplified crime model (§11.3).
2. **No failure recovery for disasters** — disasters damage buildings but there's no fire spread, evacuation, or repair mechanic. Just damage + demolish. **Acceptable for v1** — keep it simple.
3. **No resource scarcity** — resources are infinite once placed. No depletion. **Acceptable for v1** — depletion adds complexity without clear benefit.
4. **No population cap per building type** — residential has implicit 50/pop but no explicit cap UI. **Fixed:** add population cap display in building tooltip.
5. **No maintenance costs** — buildings cost nothing to maintain. **Fixed:** added upkeep system (§11.7.H).
6. **No loan/debt system** — can't borrow money when broke. **Deferred** — adds complexity, not essential for v1.
7. **No tutorial** — new players lost. **Fixed:** added tutorial overlay (§11.7.I).
8. **No accessibility** — no keyboard shortcuts, no screen reader. **Partially fixed:** keyboard shortcuts (§11.7.J). Screen reader deferred.
9. **Performance on large grids** — 60x60 = 3600 tiles. Could be slow. **Mitigated:** existing LOD system + instanced rendering already in place. No change needed.
10. **No mobile support** — UI is desktop-focused. **Deferred** — not in scope for this revamp.

## 12. Implementation Tasks (ordered)

1. Update `package.json` and remove cloud deps
2. Delete `geminiService.ts`, `openaiService.ts`
3. Create `localAIService.ts` with template pools for goals, news, terrain, ideas, music
4. Create `services/soundService.ts` (Web Audio API SFX)
5. Create `services/achievementService.ts` (milestone tracking)
6. Rewrite `aiService.ts` to call local functions (synchronous, no fetch)
7. Strip `server.ts` of all `/api/ai/*` routes
8. Clean `vite.config.ts`
9. Update `types.ts` with new type changes (§5)
10. Update `constants.tsx` with unlock thresholds, pollution values, upkeep values (§5b)
11. Refactor `App.tsx` — remove AI key checks, integrate new game systems (§7), reputation persistence (§8), migration (§8), undo/redo stack, statistics tracking, keyboard shortcuts
12. Refactor `IdeaLab.tsx` — remove react-markdown, local generation, prompt textarea
13. Refactor `components/MusicLab.tsx` — remove Lyria mode, keep procedural
14. Refactor `components/UIOverlay.tsx` — UI cleanup + new stat displays (happiness, crime, pollution, reputation, blueprint buttons, locked indicators, undo/redo, statistics panel, achievement panel, minimap, tutorial overlay)
15. Refactor `components/StartScreen.tsx` — relabel AI toggle, remove glassmorphism
16. Refactor `components/IsoMap.tsx` — remove `analyzePerformance` fetch (use heuristic), add day/night lighting, add minimap canvas
17. Update `.env.example` and `metadata.json`
18. Run `npm install && npm run lint`

## 13. Validation

- `npm install` succeeds
- `npm run lint` (tsc --noEmit) passes
- `npm run build` succeeds
- App runs without any env vars set
- No `@google/genai`, `openai`, or `react-markdown` imports remain anywhere in the codebase
- All 6 gameplay systems functional: road connectivity, pollution, happiness, disasters, tech tree, blueprints
- Reputation persists across game resets
- Old saves (without happiness/pollution/isConnected) load without errors
- Sound effects play on building placement, bulldoze, disaster, goal complete
- Undo/redo works via Ctrl+Z/Ctrl+Y and toolbar buttons
- Statistics panel shows population/money/happiness history
- Day/night cycle visibly changes lighting
- Building upgrades work (house → apartment → tower)
- Achievements unlock and display
- Minimap shows building types as colored dots
- Maintenance costs deducted each tick
- Tutorial overlay shows on first game
- Keyboard shortcuts work (1-7 tools, B bulldoze, U undo, Space pause, Esc deselect)

## 15. UI Refactor Details (concrete changes per component)

### `components/UIOverlay.tsx`
**Remove:**
- AI Goal Panel's provider badge (Gemini/OpenAI logo + name) → replace with "City Advisor" label only
- Rate limit status display (red warning box, cooldown timer) → remove entirely
- CRT scanline overlay div (`bg-[linear-gradient...]` fixed div)
- `getAiStatus` polling interval (2-second setInterval) → remove
- `aiStatus` state → remove

**Modify:**
- Stats Bar: add Happiness display (0-100, color-coded: green >70, yellow 40-70, red <40)
- Stats Bar: add Crime display (only when crime > 0, red badge)
- Stats Bar: add Pollution display (only when avgPollution > 0, brown badge)
- AI Goal Panel → rename to "City Advisor" panel, keep goal display + claim reward
- Add reputation display in top bar (⭐ icon + number)
- Add undo/redo buttons in toolbar (Undo icon, Redo icon)
- Add mute toggle button (Volume2/VolumeX icon)
- Add statistics panel toggle (BarChart3 icon → collapsible chart)
- Add achievement panel toggle (Trophy icon → collapsible list)
- Add blueprint export/import buttons (Download/Upload icons)
- Add minimap toggle (Map icon)
- Toolbar: replace `⛏️` emoji with `Pickaxe` lucide icon, `✕` with `X` lucide icon
- Status badges: replace `⚡` with `Zap` icon, `💧` with `Droplets` icon, `💎` with `Gem` icon, `$` stays as text

**Add:**
- Happiness trend indicator (↑/↓ arrow based on last 3 ticks)
- Disaster alert banner (red, dismissible, shows disaster type + location)
- Achievement popup (toast notification, 3-second auto-dismiss)
- Locked building tooltip ("Requires {n} reputation" on locked tools)

### `components/StartScreen.tsx`
**Remove:**
- Glassmorphism: `bg-slate-900/90 ... backdrop-blur-xl` → solid `bg-slate-900`
- Gradient text on "SkyMetropolis" title → solid white text
- "AI City Advisor" label → "City Advisor (quests & news)"
- `window.aistudio` API key check logic

**Modify:**
- Keep the AI toggle (relabeled), keep map size selector
- Remove the decorative background glow divs

### `components/IsoMap.tsx`
**Remove:**
- `analyzePerformance` import and call → replace with `getFallbackRecommendation` (already exists, called directly)
- `PerformanceMonitor`'s `analyzePerformance` fetch → call `getFallbackRecommendation(stats)` synchronously
- AI reason display text (the italic blue text under perf overlay) → keep but show heuristic reason

**Modify:**
- Add `day` prop to `IsoMapProps` (currently not passed) → use for day/night lighting
- Lighting: ambient intensity + directional light color based on `day % 24`:
  - Hours 6-8 (dawn): warm orange, intensity 0.6
  - Hours 8-18 (day): bright white, intensity 1.0
  - Hours 18-20 (dusk): orange-red, intensity 0.7
  - Hours 20-6 (night): dark blue, intensity 0.3
- Add minimap: 120x120 canvas in bottom-right corner showing building types as colored dots
- Add unconnected building warning: red outline on buildings where `isConnected === false`

**Add:**
- `minimapRef` canvas ref
- `drawMinimap()` function called on grid change
- Minimap click handler → pan camera to clicked location

### `components/MusicLab.tsx`
**Remove:**
- Lyria mode entirely: mode toggle, image upload, API key check, `handleConnectKey`, `handleImageUpload`, `generateMusic` import
- `hasApiKey` state
- `generationMode` state (only procedural remains)
- `imageBase64` state

**Modify:**
- Remove mode selection grid (only "Synth Composer" remains, always active)
- Remove Lyria-specific UI (image preview, API key warning)
- Keep procedural music generation (calls `generateProceduralMusic` locally)
- Keep Tone.js playback (unchanged)
- Keep track list (unchanged)
- Rename header from "AI Music Lab" to "Music Lab"

### `IdeaLab.tsx`
**Remove:**
- `react-markdown` import and usage → replace with static JSX detail cards
- `generateFeatureIdeas` and `generateIdeaDetails` async calls → call local sync versions
- Prompt textarea and "Generate Concept" button → replace with "Add Random Idea" button
- `isGenerating` state (no async generation)
- `error` state (no async errors)
- Scanline overlay div

**Modify:**
- "Generate Concept" button → "Add Random Idea" button (calls `generateFeatureIdeas()` synchronously)
- "Expand Details" modal → shows static JSX detail card (from `generateIdeaDetails`)
- "Build Combined Plan" → concatenates static detail cards for selected approved ideas
- Remove `Markdown` component usage, replace with styled divs

### `App.tsx`
**Remove:**
- `generateCityGoal`, `generateNewsEvent`, `getOptimalLayout` async imports → sync local calls
- `isGeneratingGoal` state (goals are instant now)
- AI key checks in `fetchNewGoal` and `fetchNews`
- `aiEnabledRef` sync (keep `aiEnabled` state but remove ref since no async needed)

**Modify:**
- `fetchNewGoal` → sync: `const newGoal = generateCityGoal(statsRef.current, gridRef.current); setCurrentGoal(newGoal);`
- `fetchNews` → sync: `const news = generateNewsEvent(statsRef.current, null); if (news) addNewsItem(news);`
- `handleStart` → remove `getOptimalLayout` async call, use local sync version
- Add reputation state: `const [reputation, setReputation] = useState(() => loadReputation())`
- Add undo stack: `const [undoStack, setUndoStack] = useState<Grid[]>([])`
- Add statistics tracking: `const [statsHistory, setStatsHistory] = useState<StatsSnapshot[]>([])`
- Add keyboard shortcut listener (useEffect with keydown)
- Add sound triggers on place, bulldoze, disaster, goal complete
- Add achievement checks after each tick
- Add maintenance cost deduction in game loop
- Add road connectivity flood-fill in game loop
- Add pollution calculation in game loop
- Add happiness calculation in game loop
- Add crime calculation in game loop
- Add disaster check in game loop
- Add reputation award on goal complete
- Add statistics snapshot on each tick

**Add:**
- `loadReputation()` / `saveReputation()` functions
- `pushUndo(grid)` function
- `undo()` / `redo()` functions
- `handleExportBlueprint()` / `handleImportBlueprint()` functions
- `triggerDisaster()` function
- `checkAchievements()` call in game loop

### `StartScreen.tsx`
**Remove:**
- Glassmorphism panel → solid surface
- Gradient text → solid text
- Decorative glow divs
- `window.aistudio` references

**Modify:**
- "AI City Advisor" → "City Advisor (quests & news)"
- Keep toggle and map size selector
- Remove the `aiEnabled` pulse indicator (or replace with simple dot)

## 16. Resolved Questions (finalized)

1. **Disaster frequency** — pollution-gated. Disasters only trigger when total pollution exceeds a threshold.
2. **Blueprint format** — base64 of compressed JSON.
3. **Multiple settlements** — deferred to a future update. Single city for v1.
## 19. Living World — Autonomous Civilization Simulation (vertical slice)

### 19.1 Scope Lock (first vertical slice)

One isolated settlement, 100% offline, no LLMs/APIs. Reference configuration:

| Entity | Initial quantity |
|---|---:|
| Citizens | 24 |
| Households | 8 |
| Residential buildings | 8 |
| Farm | 1 |
| Bakery | 1 |
| Market | 1 |
| Government / town hall | 1 |
| Currency | 1 |
| Goods | Grain and bread |
| Tax type | Income tax (10%) |
| Road network | 1 connected network |
| Simulation region | 1 settlement (20×20 tile test map) |

These are acceptance-test fixtures, not permanent engine limitations.

### 19.2 Required Citizen Behaviors

Every citizen has: unique persistent ID, household assignment, current location, home location, needs (hunger, fatigue, financial security), employment status, employer reference, wage rate, personal/household financial balance, skills, personality weights, current goal, current action, bounded memories of meaningful events.

Every citizen must be capable of: traveling to work, performing assigned work, receiving wages, seeking employment, purchasing bread when hungry, eating purchased food, returning home and resting, changing priorities on important events.

Use utility scoring + GOAP (goal-oriented action planning). Randomness may influence preferences but must not determine behavior without causal justification. Citizens retain goals across ticks (hysteresis, commitment intervals, replanning triggers).

### 19.3 Economic Loop

Farm → Grain → Bakery → Bread → Market → Households

- Farm produces grain using land, labor, production time.
- Bakery purchases grain, converts to bread via recipe (labor + processing time).
- Market purchases bread, sells to households.
- Citizens work → receive wages → pay income tax → purchase food.
- Government receives taxes into treasury, spends on public expenses (road maintenance).
- Businesses pay for inventories, employees, operating expenses.
- Every purchase transfers money between real accounts. Every produced item has recorded source. Every consumed item removed from inventory. No money/goods/resources appear without defined origin.

### 19.4 Government Loop

Income tax initially 10%. When wages paid: employer debited gross wage, worker receives net wage, government receives tax amount. Transaction references worker, employer, tax policy, simulation time. Government maintains actual treasury. Public spending debits treasury, credits recipient. If treasury can't afford, expenditure rejected or deferred. No additional tax types, elections, or bureaucracy in first slice.

### 19.5 State Architecture

Separated state domains: WorldState (time, seed, RNG state, map entities, roads, buildings, resources, weather, events), CitizenState (identity, needs, location, knowledge, skills, goals, actions, employment, memories, relationships), HouseholdState (members, home, ownership, shared finances), BusinessState (identity, location, employees, openings, cash, inventory, orders, recipes, prices, production status), GovernmentState (treasury, tax rates, expenditures, infrastructure), EconomyState (accounts, transactions, inventories, orders, prices, production records), ChaosState (config, incidents, history, cooldowns, exposure, consequences), SimulationState (entity refs, scheduled actions, deterministic ordering, pending commands, persistence metadata).

Single authoritative owner per mutable state. Stable entity IDs (not array indices). No duplicated mutable financial balances.

### 19.6 Non-Negotiable State Invariants (INV-01 through INV-14)

- **INV-01 Currency Conservation:** sum(all account balances) = initial total currency. Every transaction has equal debit/credit. Money cannot be created/destroyed.
- **INV-02 Nonnegative Finances:** Account cannot spend more than owned. No credit/overdraft. Failed transaction is atomic (no partial state).
- **INV-03 Inventory Conservation:** `current stock = initial stock + production + imports − consumption − exports − destruction`. Inventory never negative. Bread recipe consumes grain before producing bread.
- **INV-04 Unique Entity Ownership:** Every citizen belongs to exactly one household. Every employed citizen has at most one employer. Every job has at most one occupant. Entity refs valid after save/load.
- **INV-05 Spatial Validity:** Citizens occupy valid positions. Movement respects walkability and route connectivity. No teleportation. Must be physically present at workplace for location-dependent work.
- **INV-06 Atomic Transactions:** Economic ops succeed completely or leave state unchanged. No money deduction without goods transfer. No employer debit without worker credit + tax.
- **INV-07 Causal Event Integrity:** Every state change has originating command/action/event. Record event ID, tick, actor, affected entities, preconditions, state changes, outcome, parent event ID.
- **INV-08 Valid State Transitions:** `IDLE → PLANNED → TRAVELING → EXECUTING → COMPLETED` (plus FAILED, CANCELLED, INTERRUPTED). Failed actions release reservations, no ghost assignments.
- **INV-09 Deterministic Replay:** Identical initial state, seed, config, inputs, version → identical state + event history. Fixed timestep. Persist RNG state, pending events, action queues. Stable ordering for simultaneous events.
- **INV-10 Persistence Integrity:** Save/load preserves citizens, households, money, inventories, employment, goals, actions, treasury, clock, RNG state, scheduled events, chaos config. Saving must not mutate state.
- **INV-11 Need Bounds:** `0 ≤ hunger ≤ 100`, `0 ≤ fatigue ≤ 100`. Rate-limited by simulation time and activity costs.
- **INV-12 No Omniscient NPCs:** Decisions depend on obtainable information only (observation, communication, published jobs, memories, public market data). No hidden future events.
- **INV-13 Deterministic Conflict Resolution:** Two citizens claiming one job/item/resource — only one succeeds. Authoritative reservations, deterministic resolution.
- **INV-14 Invariant Safety Under Chaos:** Chaos events use same validated state transitions and accounting. Even supernatural events account for money/resources/entities created or destroyed. Rules of reality may change; rules of state consistency may not.

### 19.7 Enforcement Strategy

Enforce invariants at system boundaries. Validated commands for employment, transactions, production, movement, construction, event application. Development-mode invariant audits at every committed tick. Production: critical validations always, full-world audits less frequently. On violation: reject operation, preserve/restore previous state, record failure, continue safely, expose via diagnostics. Never silently repair invalid financial state by inventing money/resources.

### 19.8 Chaos Engine

Two independent controls:

**Chaos Intensity (0-100):**
| Setting | Name | Behavior |
|---|---|---|
| 0 | Peaceful | No injected incidents |
| 1-20 | Stable | Occasional minor incidents |
| 21-40 | Everyday Life | Typical setbacks |
| 41-60 | Turbulent | Frequent meaningful disruptions |
| 61-80 | Crisis-Prone | Severe incidents, substantial ripple effects |
| 81-100 | Catastrophic | Repeated crises, possible systemic collapse |

**Reality Mode:** REALISTIC (plausible causes/effects), HEIGHTENED (unusual coincidences, exaggerated consequences), SURREAL (impossible/absurd events). Default: intensity 30, REALISTIC. Independently configurable. Surreal must be explicitly enabled.

### 19.9 Realistic Chaos Categories

Economic: demand changes, bankruptcies, recessions, labor shortages, price spikes, supplier failures, unexpected opportunities. Environmental: storms, flooding, drought, fires, crop failures, heat waves, infrastructure damage. Social: workplace disputes, community disagreements, strikes, localized crime, boycotts, unexpected cooperation, migration pressure. Infrastructure: road closures, service interruptions, transport bottlenecks, building damage, delayed construction, equipment failures. Do not enable an event until its required underlying mechanics exist.

### 19.10 Surreal Chaos Catalogue (extensible, future)

Examples: The Great Goose Uprising (geese occupy roads, disrupt commuting), Bread Rain (bread falls from sky, prices collapse), Mysterious Gold Rush (strange resource appears, labor shortages), Collective Workplace Madness (mass resignations), Time Distortion (activity durations change, tick ordering preserved), Sentient Architecture (building refuses occupants via access restrictions), The Potato Cult (charismatic citizen, followers alter spending/employment). Do not implement all in first slice — treat as extensible future catalogue.

### 19.11 Chaos Must Be Causal

Every event must have: eligibility rules, trigger conditions, target selection, severity, duration, affected systems, immediate consequences, potential secondary effects, recovery conditions, cooldown, recorded provenance. Realistic event risk depends on actual exposure. In surreal mode, impossible triggers permitted but consequences still interact with normal systems. A magical event may create bread; it may not silently alter accounting balances.

### 19.12 Cascading Consequences

Avoid artificial chains. Instead: disaster → damaged infrastructure → reduced accessibility → delayed deliveries → inventory shortage → lost sales → reduced cash reserves → missed payroll → unemployment → reduced demand. Each link depends on actual state and may or may not happen. Same event produces different outcomes in different settlements.

### 19.13 Recovery and Adaptation

Every recoverable incident has defined recovery mechanics: repairs, replacement purchases, reemployment, emergency spending, route changes, resource substitution, business restructuring, household relocation. Avoid permanent freezes unless recovery genuinely impossible. Recovery is not guaranteed — some settlements should fail.

### 19.14 First Chaos Implementation (3 events only)

**Event 1 — Equipment Breakdown** (all reality modes): Target farm or bakery. Reduces/stops production. Causes inventory shortages. Reduces revenue. Affects employment/wages via normal business policies. Recovers via funded repair or documented maintenance.

**Event 2 — Road Closure** (all reality modes): Target one road segment. Blocks route. Forces alternative routes. Increases travel/delivery time. Delays workers/supplies. Affects production/retail inventory. Restores via repair/reopening. Reference map must have primary road + alternative route.

**Event 3 — Bread Rain** (Surreal only): Spawn bounded quantity of bread at valid locations. Record as anomalous resource creation. Available via explicit collection rules. Affects market demand/prices via existing mechanics. Must never trigger in Realistic/Heightened modes. Do not implement additional chaos events until these three work and pass tests.

### 19.15 Time and Scheduling

Fixed simulation timestep = 1 simulated minute. Explicit schedules for: citizen needs, movement, work, production, shopping, wage payments, taxes, business decisions, chaos events, maintenance/recovery. Use event queues and scheduled updates. Don't recompute every decision every render frame. Simulation must run headlessly.

### 19.16 Deterministic Randomness

Persisted, seeded random generator. Separate random streams for: citizen initialization, routine decision variation, economic uncertainty, realistic chaos, surreal chaos. Persist stream state. Log externally introduced events and config changes.

### 19.17 Chaos Configuration

```json
{
  "chaos": {
    "enabled": true,
    "intensity": 30,
    "reality_mode": "REALISTIC",
    "seed": 12345,
    "max_concurrent_incidents": 2,
    "cooldown_ticks": 1440,
    "recovery_enabled": true,
    "allow_cascades": true
  }
}
```

Validate ranges and enum values. Don't expose controls that do nothing. Define how intensity maps to incident eligibility/severity. Changing settings affects future behavior without retroactively corrupting active incidents.

### 19.18 No-Model Decision System (GOAP)

Utility scores + goal-oriented action planning. Example: hungry citizen evaluates eat owned food, buy bread, work for money, seek assistance, rest. Each candidate has preconditions, expected costs, expected benefits, required resources, location requirements, risk/failure conditions. Agent selects feasible action or plan that makes desired action feasible. No action bypasses authoritative state rules. Citizens retain goals across ticks (hysteresis, commitment intervals, replanning triggers).

### 19.19 Implementation Steps (from spec §27)

1. **Inspect** — examine existing project, identify relevant code and dependencies.
2. **Plan** — define smallest compatible integration architecture, authoritative state structures, exact files to modify.
3. **Test foundation** — implement invariant tests and deterministic reference fixtures.
4. **Functional economy** — implement citizen behavior, work, production, transactions, taxation, shopping, needs.
5. **World integration** — connect simulation state to existing tiles, buildings, pathfinding.
6. **Chaos engine** — implement eligibility, incident state, recovery, three required events.
7. **Observability** — expose real citizen, business, government, chaos state.
8. **Verification** — run unit, integration, headless, persistence, deterministic replay, invariant stress tests.

Do not begin broad second development phase until first vertical slice passes.

### 19.20 Implementation Boundaries (explicitly out of scope for v1)

Full political simulation, complex relationships/marriage, generational inheritance, advanced healthcare, large-scale military conflict, sophisticated crime systems, global trade, dozens of professions, hundreds of resource types, complex technology progression, hundreds of supernatural event types. Design extension points but don't build unrequested subsystems. Preserve unrelated project files. Don't remove existing functionality. Document migrations and provide rollback path for save format changes.

### 19.21 Required Final Report

Report: files added/modified, implemented subsystems, implemented chaos incidents, invariants enforced, tests passed/failed, reference scenario results, deterministic replay outcome, performance measurements, known limitations, features deliberately deferred, rollback instructions. Use factual statuses: PASS, FAIL, BLOCKED, or NOT IMPLEMENTED. Never claim work completed without actual verification.

### 19.22 Test Scenarios (from spec §22)

- **Test A — Peaceful:** chaos intensity 0, no incidents, economy operates.
- **Test B — Realistic:** realistic incidents at fixed intensity, verify eligible incidents produce bounded effects.
- **Test C — Road Disruption:** close a road, verify pathfinding responds and deliveries change.
- **Test D — Surreal Lockout:** max intensity, REALISTIC mode, run sufficient schedule, verify no surreal event eligible/executed.
- **Test E — Bread Rain:** SURREAL mode, trigger Bread Rain, verify anomalous bread creation, collection, market effects, conservation accounting.
- **Test F — Replay:** identical seeds/configs/inputs/ticks twice, compare state hashes and event histories.
- **Test G — Save During Incident:** save while road closure or equipment breakdown active, reload, verify incident duration/effects/pending actions/recovery continue.
- **Test H — Invariant Stress:** repeated purchases, job competition, insufficient funds, shortages, overlapping incidents. Verify all invariants satisfied.

### 19.23 Acceptance Criteria (from spec §23)

Vertical slice complete only when: all reference entities exist with valid state; citizens perform useful work and make context-dependent decisions; complete grain-to-bread supply chain operates; financial transactions and taxes reconcile exactly; inventory accounting reconciles exactly; at least one complete employment-to-purchase lifecycle succeeds; both realistic incidents function and recover; Bread Rain functions only under Surreal mode; an incident produces measurable causal economic consequence; save/load preserves active civilization; deterministic replay passes; automated invariant checks pass; reference scenario runs headlessly; citizen and business inspectors display actual state; existing world-builder functionality remains intact. Test over at least 30 simulated days with additional targeted crisis tests. Use integer currency units. Measure and report performance. If a test fails, report failure and cause. Do not mark complete.

### 19.24 UI and Observability

- **Citizen inspector:** current location, household, employment, income, savings, needs, current action, current goal, why action selected, important memories, recent consequences.
- **Business inspector:** employees, account balance, inventory, production, revenue, expenses, supply problems, operational status.
- **Government inspector:** treasury balance, taxes collected, expenditures, pending maintenance.
- **Chaos control panel:** enable/disable, intensity 0-100, reality mode, active incident count, incident history, event seed, recovery setting, manual incident trigger (deterministic when recorded as external input). Show why incident occurred, who affected, resulting changes.
- **Causality inspector:** make consequences inspectable (e.g., bakery equipment breakdown → production stopped → bread inventory fell → market stock decreased → purchases unfulfilled → revenue declined → wages delayed → unemployment → reduced demand). Each explanation references real events and state transitions.

### 19.25 Key Implementation Notes

- Use integer currency units to avoid floating-point monetary drift.
- Measure and report performance rather than claiming scalability without benchmarks.
- Provide reproducible reference settlement fixture (20×20 tile test map if compatible world map doesn't exist).
- Citizens and businesses respond via same authoritative simulation systems — no hardcoded reactions.
- Citizens retain goals across ticks (hysteresis, commitment intervals, replanning triggers).
- No action bypasses authoritative state rules.
- Design extension points but don't build unrequested subsystems.
- Preserve unrelated project files. Don't remove existing functionality.
- Document migrations and provide rollback path for save format changes.

## 18. Foliage, Vegetation, Environmental Decoration (refinements)

### 18.1 Foliage System
- **Current:** No foliage exists. Terrain is flat with buildings. Water plane at y=-0.6. No trees, bushes, or grass.
- **Refinements:** Trees (3 types: deciduous, coniferous, palm — cost 25-30, +1 happiness within 3-tile radius, reduces pollution by 0.5, 3 variants per type, instanced rendering). Bushes/Shrubs (2 types — cost 10, +0.5 happiness, 2 variants). Flowers (3 types: red/yellow/purple — cost 5, decorative only, 3 variants). Grass (auto-generated on empty tiles, 3 types, visual only, replaced by buildings). Fallen leaves (autumn particle effect from deciduous trees).
- **Affected files:** `types.ts` (add `FoliageType` enum), `constants.tsx` (foliage configs), `App.tsx` (placement logic, happiness/pollution integration), `components/IsoMap.tsx` (instanced foliage rendering)

### 18.2 Environmental Decoration
- **Current:** No environmental props. Only buildings, roads, resources, cars, citizens, clouds, birds.
- **Refinements:** Street lamps (cost 15, light at night, auto-placed on roads). Benches (cost 8, +0.5 happiness within 2-tile radius). Fountains (cost 50, +2 happiness within 3-tile radius, water particles). Statues (cost 100, +1 happiness within 4-tile radius). Street signs (auto-placed on intersections, decorative). Trash cans (auto-placed near commercial). Fire hydrants (auto-placed on road-adjacent tiles).
- **Affected files:** `types.ts` (add `DecorationType` enum), `constants.tsx` (decoration configs), `App.tsx` (placement logic), `components/IsoMap.tsx` (instanced decoration rendering)

### 18.3 Seasonal Changes
- **Current:** No seasonal system. Day counter exists but no visual season changes.
- **Refinements:** 4 seasons, 90 days each (360-day year). Spring (days 1-90): grass grows, flowers bloom, trees green, rain more frequent, +2 happiness. Summer (91-180): bright lighting, no rain, trees full green, +1 happiness, water demand +20%. Autumn (181-270): leaves change color, leaf particles, grass browns, 0 happiness bonus. Winter (271-360): snow on ground/rooftops, bare deciduous trees, -2 happiness, water demand -20%, snow particles. Season indicator in top bar (🌸☀️🍂❄️).
- **Affected files:** `types.ts` (add `Season` enum), `constants.tsx` (season day ranges, bonuses), `App.tsx` (season calculation), `components/IsoMap.tsx` (season-based colors, snow overlay, leaf particles), `components/UIOverlay.tsx` (season indicator)

### 18.4 Weather System (expanded)
- **Current:** No weather system. Clouds exist but are static decoration.
- **Refinements:** Rain (particles, -1 happiness, +20% water production, washes 1 pollution per tick, 20% chance in spring/autumn). Snow (particles, visual only, 30% chance in winter). Fog (volumetric FogExp2, reduces visibility, 10% chance any season). Thunderstorm (heavy rain + lightning + thunder, -3 happiness, +40% water, washes 2 pollution, 5% chance in spring/summer). Weather indicator in top bar (☀️🌧️❄️🌫️⛈️).
- **Affected files:** `types.ts` (add `Weather` enum), `constants.tsx` (weather chances, effects), `App.tsx` (weather roll in game loop), `components/IsoMap.tsx` (particle systems, fog), `components/UIOverlay.tsx` (weather indicator)

### 18.5 Wildlife
- **Current:** Birds (instanced meshes, flying in circles). No other wildlife.
- **Refinements:** Enhanced birds (seagulls near water, pigeons near commercial, perch on buildings, varied flight). Butterflies (near flowers/parks, 3 colors, spring/summer only). Squirrels (near trees, 2 colors, year-round). Fish (in water tiles, jump occasionally, visual only).
- **Affected files:** `types.ts` (add `WildlifeType` enum), `components/IsoMap.tsx` (instanced wildlife rendering, behavior)

### 18.6 Foliage & Decoration Constants
```ts
export const FOLIAGE_COSTS = { tree_deciduous: 25, tree_coniferous: 25, tree_palm: 30, bush_round: 10, bush_tall: 10, flower_red: 5, flower_yellow: 5, flower_purple: 5 };
export const FOLIAGE_HAPPINESS = { tree_deciduous: 1, tree_coniferous: 1, tree_palm: 1, bush_round: 0.5, bush_tall: 0.5 };
export const FOLIAGE_POLLUTION_REDUCTION = { tree_deciduous: 0.5, tree_coniferous: 0.5, tree_palm: 0.5 };
export const DECORATION_COSTS = { street_lamp: 15, bench: 8, fountain: 50, statue: 100 };
export const DECORATION_HAPPINESS = { street_lamp: 0, bench: 0.5, fountain: 2, statue: 1 };
export const SEASON_DAYS = { spring: [1, 90], summer: [91, 180], autumn: [181, 270], winter: [271, 360] };
export const SEASON_HAPPINESS_BONUS = { spring: 2, summer: 1, autumn: 0, winter: -2 };
export const SEASON_WATER_DEMAND_MULT = { spring: 1.0, summer: 1.2, autumn: 1.0, winter: 0.8 };
export const WEATHER_CHANCES = { rain: 0.2, snow: 0.3, fog: 0.1, thunderstorm: 0.05 };
export const WEATHER_HAPPINESS_EFFECT = { rain: -1, snow: 0, fog: 0, thunderstorm: -3, sunny: 0 };
export const WEATHER_WATER_BOOST = { rain: 1.2, snow: 1.0, fog: 1.0, thunderstorm: 1.4, sunny: 1.0 };
export const WEATHER_POLLUTION_WASH = { rain: 1, snow: 0, fog: 0, thunderstorm: 2, sunny: 0 };
```

### 18.7 Foliage Rendering
- Instanced rendering for trees, bushes, flowers, grass (same pattern as buildings/cars/citizens).
- Season-based color changes: deciduous trees green→yellow→orange→bare, coniferous stays green, grass green→brown→snow-covered.
- Foliage LOD: high=full mesh, medium=simplified, low=colored dot.
- Grass auto-generated on empty tiles: 4 grass blades per tile, random rotation, slight color variation.
- Leaf particles: instanced small quads falling from deciduous trees in autumn.

### 18.8 Foliage BuildingType Additions
```ts
export enum BuildingType {
  // ... existing ...
  Bridge = 10,
  TreeDeciduous = 11, TreeConiferous = 12, TreePalm = 13,
  BushRound = 14, BushTall = 15,
  FlowerRed = 16, FlowerYellow = 17, FlowerPurple = 18,
  StreetLamp = 19, Bench = 20, Fountain = 21, Statue = 22,
}
```

### 18.9 Type Changes (additions)
```ts
// Add to TileData:
rotation?: number;   // 0, 90, 180, 270
tier?: number;       // 0 = base, 1 = upgraded, 2 = max
damaged?: boolean;   // disaster damage state

// New enums:
export enum Season { Spring, Summer, Autumn, Winter }
export enum Weather { Sunny, Rain, Snow, Fog, Thunderstorm }
export enum WildlifeType { Bird, Seagull, Pigeon, Butterfly, Squirrel, Fish }
```

### 18.10 Implementation Tasks (foliage/season/weather)
19. Add foliage types to BuildingType enum and BUILDINGS record
20. Add Season and Weather enums to types.ts
21. Add foliage placement logic to App.tsx (happiness + pollution integration)
22. Add decoration placement logic to App.tsx (happiness integration)
23. Add season calculation to game loop
24. Add weather roll to game loop (with effects on happiness, water, pollution)
25. Add foliage instanced rendering to IsoMap.tsx (season-based colors)
26. Add decoration instanced rendering to IsoMap.tsx
27. Add grass auto-generation to IsoMap.tsx
28. Add leaf particles (autumn) to IsoMap.tsx
29. Add weather particle systems (rain, snow, fog, thunderstorm) to IsoMap.tsx
30. Add season/weather indicators to UIOverlay.tsx
31. Add wildlife enhancement (seagulls, pigeons, butterflies, squirrels, fish) to IsoMap.tsx
32. Run `npm install && npm run lint`

1. **Disaster frequency** — pollution-gated. Disasters only trigger when total pollution exceeds a threshold.
2. **Blueprint format** — base64 of compressed JSON.
3. **Multiple settlements** — deferred to a future update. Single city for v1.
4. **Implementation-level details** — all resolved with recommended choices (template pools, performance heuristic, music lab, idea lab, UI overlay). See §10, §12.

## 17. Engine, Terrain, Buildings, People, Cars (refinements)

### 17.1 Terrain
- **Current:** Flat grid with slight height variation (noise-based). Water plane at y=-0.6. No bridges, no terrain editing.
- **Refinements:** Terrain height editor (raise/lower land, costs money, creates hills/valleys/cliffs). Water bodies (placeable lakes/rivers, affect land value and pollution). Bridges (new building type, road over water, cost 50). Terrain smoothing (auto-smooth edges between height differences to prevent floating buildings).
- **Affected files:** `types.ts` (add `height` to TileData, `BuildingType.Bridge`), `constants.tsx` (bridge config), `App.tsx` (terrain edit logic), `components/IsoMap.tsx` (height-based rendering)

### 17.2 Buildings
- **Current:** Procedural box geometries with 3 variants per type. LOD system exists. Smoke stacks on industrial. Window blocks on residential/commercial.
- **Refinements:** Building damage states (visual damage on disaster: cracked texture, tilted mesh, smoke; damaged buildings produce 50% output until repaired). Building upgrades (tier system: House→Apartment→Tower, Shop→Mall, Factory→Plant; costs money + reputation; taller/more detailed mesh per tier). Building rotation (R key rotates 90° before placement, stores `rotation` on TileData). Building variety (add 2 more variants per type, target 5 total). Glowing windows at night (emissive material on residential/commercial windows when day/night cycle active).
- **Affected files:** `types.ts` (add `rotation`, `tier`, `damaged` to TileData), `constants.tsx` (tier costs, upgrade rep, repair cost), `components/IsoMap.tsx` (damage visuals, tier meshes, rotation, night glow)

### 17.3 People (Citizens)
- **Current:** Instanced box meshes walking randomly on walkable tiles. Agent count = `min(population/2, 300)`. Citizens have no needs, homes, or jobs.
- **Refinements:** Citizen needs (hunger, shelter, employment — unmet needs reduce happiness; hungry citizens walk slower, unemployed idle). Home assignment (citizens assigned to residential buildings; no homes = population decreases). Job assignment (citizens work at commercial/industrial; employment rate affects happiness and income). Daily routines (home → work → shop → home along roads, not random). Citizen models (replace boxes with low-poly head+body, 2-3 color variants, keep instanced rendering).
- **Affected files:** `types.ts` (Citizen interface), `App.tsx` (citizen simulation in game loop), `components/IsoMap.tsx` (citizen rendering, pathfinding)

### 17.4 Cars (Traffic)
- **Current:** Instanced box meshes driving on roads. Car count = `min(roadTiles, 30)`. Simple neighbor-based pathfinding. Cars have no destination logic.
- **Refinements:** Destination-based traffic (spawn at residential, drive to commercial/industrial, return; reduces random wandering). Traffic lights (4-way junctions get lights; cars stop at red; reduces congestion). Parking (commercial/industrial have parking spots; cars park and disappear while "shopping/working"). Vehicle variety (sedan, truck, bus — different sizes/speeds). Traffic congestion metric (track cars per road tile; if > threshold, traffic jam → happiness penalty + news event).
- **Affected files:** `types.ts` (VehicleType, TrafficLight), `App.tsx` (traffic simulation), `components/IsoMap.tsx` (vehicle rendering, traffic lights)

### 17.5 Engine/Rendering
- **Current:** Three.js with R3F. Orthographic camera. Soft shadows. Environment preset "city". Performance monitor with LOD. Instanced rendering for ground, buildings, resources, cars, citizens.
- **Refinements:** Post-processing (bloom for glowing windows, SSAO for depth, color grading — add `@react-three/postprocessing`). Weather particles (rain reduces happiness slightly but boosts water production; snow visual only; fog reduces visibility). Dynamic shadows (shadow quality scales with LOD: high=2048px, medium=1024px, low=512px). Camera improvements (smooth zoom lerp, camera shake on disaster, cinematic auto-rotate mode). Screenshot mode (pause game, hide UI, render high-res PNG download).
- **Affected files:** `components/IsoMap.tsx` (post-processing, weather, camera, screenshot), `package.json` (add `@react-three/postprocessing`)

### 17.6 Engine Constants Update
```ts
export const TERRAIN_MAX_HEIGHT = 3;
export const TERRAIN_MIN_HEIGHT = -1;
export const BRIDGE_COST = 50;
export const UPGRADE_COST_MULTIPLIER = 1.5;
export const REPAIR_COST_RATIO = 0.5;
export const TRAFFIC_JAM_THRESHOLD = 3;
export const CONGESTION_HAPPINESS_PENALTY = -5;
export const NIGHT_GLOW_INTENSITY = 0.5;
## 23. Independent Discovery & Design Expansion

### 23.1 Architectural Weaknesses Identified

1. **God component (App.tsx):** Single component manages game state, simulation loop, rendering, UI, and persistence. At ~1000+ lines, this violates separation of concerns. **Recommended:** Split into simulation engine (pure logic, testable), state management (React context or store), and rendering (R3F scene). Priority: High. Complexity: Medium. Phase: 1.

2. **No spatial indexing:** Finding neighbors, buildings within radius, or connected roads requires O(n) scans over all tiles. With 3600 tiles + hundreds of citizens, this is a bottleneck. **Recommended:** Implement a spatial hash or quadtree for O(1) neighbor queries. Priority: High. Complexity: Medium. Phase: 1.

3. **No event sourcing:** State changes are direct mutations, not recorded as events. This makes debugging, replay, and undo impossible at the system level. **Recommended:** Implement command pattern — every state change is a command object with execute/undo. Enables undo/redo, replay, and audit trail. Priority: Medium. Complexity: High. Phase: 2.

4. **No entity component system:** Using object-oriented approach with inheritance. Adding new entity types requires modifying base classes. **Recommended:** Consider ECS (entity-component-system) for citizen/building/business entities. Priority: Low. Complexity: High. Phase: 3+.

5. **No separation of simulation and rendering:** Simulation logic embedded in React components. Cannot run headlessly or test independently. **Recommended:** Extract simulation into pure TypeScript module with no React dependencies. React components become thin views. Priority: High. Complexity: Medium. Phase: 1.

6. **No configuration management:** Settings scattered across constants.tsx, localStorage, and component state. **Recommended:** Centralized config store with schema validation, defaults, and persistence. Priority: Medium. Complexity: Low. Phase: 1.

7. **No logging framework:** console.log everywhere. No log levels, filtering, or persistence. **Recommended:** Implement structured logging with levels (debug/info/warn/error), component tags, and optional file output. Priority: Low. Complexity: Low. Phase: 2.

8. **No error boundaries per system:** Only top-level error boundary. A rendering error crashes the entire app. **Recommended:** Per-system error boundaries (simulation, rendering, UI) with graceful degradation. Priority: Medium. Complexity: Low. Phase: 1.

### 23.2 Missing Features (not in current plan)

9. **Citizen lifecycle (birth, aging, death):** Citizens are immortal. No population dynamics. **Recommended:** Add birth (requires housing + happiness), aging (skills improve with experience), death (natural/old age, disaster). Generational succession. Priority: Medium. Complexity: Medium. Phase: 2.

10. **Skill progression:** Citizens don't improve at jobs. **Recommended:** Skills increase with work experience. Higher skill = higher productivity = higher wages. Creates career paths. Priority: Medium. Complexity: Low. Phase: 2.

11. **Technology/research tree:** No unlocking new buildings through research. **Recommended:** Research points from university/library buildings. Unlock new building types, production recipes, and upgrades. Priority: Medium. Complexity: Medium. Phase: 2.

12. **Diplomacy/relations:** No interaction with other settlements. **Recommended:** Add neighboring settlements (abstract, not rendered). Trade routes, alliances, conflicts. Affects economy and happiness. Priority: Low. Complexity: High. Phase: 3+.

13. **Citizen interactions:** Citizens don't talk to each other. **Recommended:** Social interactions (greeting, conversation, conflict, friendship). Affects happiness and productivity. Visual: citizens stop and face each other briefly. Priority: Low. Complexity: Medium. Phase: 3+.

14. **Quest system:** Beyond goals, no multi-step quests. **Recommended:** Quest chains with prerequisites, rewards, and branching outcomes. E.g., "Build 3 farms → Establish trade route → Unlock market upgrade." Priority: Medium. Complexity: Medium. Phase: 2.

15. **Reputation/fame system:** No tracking of settlement reputation. **Recommended:** Settlement reputation affects migration (more citizens want to move in), trade prices, and tourism. Priority: Low. Complexity: Low. Phase: 3+.

16. **Automated testing infrastructure:** No test framework specified. **Recommended:** Vitest + React Testing Library. Unit tests for simulation logic, integration tests for game loop, E2E tests for critical flows. Priority: High. Complexity: Medium. Phase: 1.

17. **Save format versioning:** No versioned save format. **Recommended:** Add `version` field to save data. Migration functions for each version. Prevents data loss on schema changes. Priority: High. Complexity: Low. Phase: 1.

18. **Automatic backups:** No backup system. **Recommended:** Auto-save to separate slot every N minutes. Keep last 3 backups. Restore from backup on corruption. Priority: Medium. Complexity: Low. Phase: 1.

19. **Crash recovery:** No crash detection or recovery. **Recommended:** Detect unclean shutdown (flag in localStorage). On next load, offer to restore from last backup. Priority: Medium. Complexity: Low. Phase: 1.

20. **Memory management:** No pooling or cleanup. **Recommended:** Object pooling for frequently created/destroyed entities (particles, projectiles). Periodic cleanup of old event history, completed goals, resolved disasters. Priority: Medium. Complexity: Medium. Phase: 2.

21. **Hot-reload for development:** Vite HMR exists but game state is lost on reload. **Recommended:** Persist game state to sessionStorage on HMR, restore on reload. Dev-only feature. Priority: Low. Complexity: Low. Phase: 1.

22. **Modding/extensibility:** No way for users to add content. **Recommended:** Data-driven content (JSON config for buildings, recipes, events). Mod folder with override support. Priority: Low. Complexity: High. Phase: 3+.

23. **Localization/i18n:** All text hardcoded English. **Recommended:** Extract all strings to locale files. Support at least English and one other language. Priority: Low. Complexity: Medium. Phase: 3+.

24. **Accessibility:** No screen reader support, high contrast, colorblind modes. **Recommended:** ARIA labels on UI elements, high contrast theme, colorblind-friendly color palette option, keyboard navigation for all UI. Priority: Medium. Complexity: Medium. Phase: 2.

25. **Performance profiling integration:** No built-in performance monitoring. **Recommended:** Dev-mode performance overlay showing FPS, tick duration, entity counts, memory usage. Toggle with F3. Priority: Medium. Complexity: Low. Phase: 1.

### 23.3 Technical Risks

26. **localStorage size limits (5-10MB):** Large grids (60×60) + hundreds of citizens + event history could exceed limit. **Mitigation:** Compress save data (lz-string), split into multiple localStorage keys, or use IndexedDB for large saves. Priority: High. Complexity: Medium. Phase: 1.

27. **Single-threaded simulation:** All simulation runs on main thread, blocking rendering. **Mitigation:** Move simulation to Web Worker. React components communicate via postMessage. Priority: Medium. Complexity: High. Phase: 2.

28. **State mutation in game loop:** Direct mutation of state objects in setInterval. Could cause React re-render issues. **Mitigation:** Use immutable state updates or a state management library (Zustand/Redux). Priority: Medium. Complexity: Medium. Phase: 1.

29. **No input validation:** User inputs (blueprint import, config values) not validated. **Mitigation:** Schema validation (zod) on all external inputs. Priority: Medium. Complexity: Low. Phase: 1.

30. **Race conditions:** Async operations (fetch, setTimeout) without proper synchronization. **Mitigation:** Use AbortController for fetches, clear timeouts on unmount. Priority: Medium. Complexity: Low. Phase: 1.

### 23.4 Gameplay Opportunities

31. **Citizen lifecycle (birth, aging, death):** Citizens are immortal. No population dynamics. **Recommended:** Birth (requires housing + happiness + food surplus), aging (skills improve with experience, needs change), death (natural/old age, disaster, starvation). Generational succession. Priority: Medium. Complexity: Medium. Phase: 2.

32. **Skill progression:** Citizens don't improve at jobs. **Recommended:** Skills increase with work experience. Higher skill = higher productivity = higher wages. Creates career paths and specialist citizens. Priority: Medium. Complexity: Low. Phase: 2.

33. **Technology/research tree:** No unlocking new buildings through research. **Recommended:** Research points from university/library buildings. Unlock new building types, production recipes, and upgrades. Priority: Medium. Complexity: Medium. Phase: 2.

34. **Quest system:** Beyond goals, no multi-step quests. **Recommended:** Quest chains with prerequisites, rewards, and branching outcomes. E.g., "Build 3 farms → Establish trade route → Unlock market upgrade." Priority: Medium. Complexity: Medium. Phase: 2.

35. **Citizen interactions:** Citizens don't talk to each other. **Recommended:** Social interactions (greeting, conversation, conflict, friendship). Affects happiness and productivity. Visual: citizens stop and face each other briefly. Priority: Low. Complexity: Medium. Phase: 3+.

36. **Reputation/fame system:** No tracking of settlement reputation. **Recommended:** Settlement reputation affects migration (more citizens want to move in), trade prices, and tourism. Priority: Low. Complexity: Low. Phase: 3+.

37. **Diplomacy/relations:** No interaction with other settlements. **Recommended:** Add neighboring settlements (abstract, not rendered). Trade routes, alliances, conflicts. Affects economy and happiness. Priority: Low. Complexity: High. Phase: 3+.

38. **Event system for citizen interactions:** Citizens don't have personal events. **Recommended:** Personal events (promotion, marriage, illness, accident) that affect individual citizens and their households. Priority: Medium. Complexity: Medium. Phase: 2.

### 23.5 Quality-of-Life Improvements

39. **Auto-save indicator:** No visual feedback when auto-save occurs. **Recommended:** Small "Saving..." indicator in corner, auto-dismisses after 1 second. Priority: Low. Complexity: Low. Phase: 1.

40. **Pause on focus loss:** Game continues running when tab loses focus. **Recommended:** Auto-pause when tab loses focus (configurable). Priority: Low. Complexity: Low. Phase: 1.

41. **Screenshot gallery:** Screenshot mode exists but no gallery. **Recommended:** Save screenshots to localStorage or download. Gallery view in admin console. Priority: Low. Complexity: Low. Phase: 2.

42. **Undo for admin actions:** Only grid placement has undo. **Recommended:** Undo for admin commands (with true inverse operations). Priority: Medium. Complexity: Medium. Phase: 2.

43. **Notification center:** No centralized notification system. **Recommended:** Bell icon with badge count. Click to see list of recent notifications (goal complete, disaster, achievement, etc.). Priority: Medium. Complexity: Low. Phase: 1.

44. **Tooltips on all UI elements:** Many UI elements lack tooltips. **Recommended:** Tooltip on hover for all buttons, stats, and indicators. Priority: Low. Complexity: Low. Phase: 1.

45. **Keyboard shortcut customization:** Shortcuts are hardcoded. **Recommended:** Allow remapping keyboard shortcuts in settings. Priority: Low. Complexity: Low. Phase: 3+.

46. **Game speed presets:** Only single speed (2s tick). **Recommended:** Speed presets (1x, 2x, 4x, 8x) with hotkeys. Priority: Medium. Complexity: Low. Phase: 1.

47. **Auto-pause on disaster:** Game doesn't pause when disaster occurs. **Recommended:** Optional auto-pause when disaster triggers (configurable). Priority: Low. Complexity: Low. Phase: 1.

48. **Minimap click-to-pan:** Minimap exists but no click-to-pan. **Recommended:** Click on minimap to pan camera to that location. Priority: Low. Complexity: Low. Phase: 1.

### 23.6 Scalability Challenges

49. **Grid size limits:** Current max 60×60. **Recommended:** Support up to 200×200 with chunked rendering and LOD. Priority: Low. Complexity: High. Phase: 3+.

50. **Citizen count limits:** Current max 300 citizens (instanced). **Recommended:** Support up to 10,000 citizens with aggressive LOD (dots at distance, no animation at far range). Priority: Low. Complexity: High. Phase: 3+.

51. **Save file size:** No compression. **Recommended:** Use lz-string compression for save data. Reduces localStorage usage by 60-80%. Priority: Medium. Complexity: Low. Phase: 1.

52. **Rendering performance:** No frustum culling. **Recommended:** Only render tiles within camera viewport + margin. Priority: Medium. Complexity: Medium. Phase: 1.

### 23.7 Priority Summary (Phase 1 — must-have for first vertical slice)

| # | Feature | Priority | Complexity |
|---|---|---|---|
| 1 | Split App.tsx into simulation engine + state + rendering | High | Medium |
| 2 | Spatial indexing (quadtree/spatial hash) | High | Medium |
| 3 | Automated testing infrastructure (Vitest) | High | Medium |
| 4 | Save format versioning | High | Low |
| 5 | localStorage size mitigation (compression + IndexedDB) | High | Medium |
| 6 | Per-system error boundaries | Medium | Low |
| 7 | Configuration management (centralized config store) | Medium | Low |
| 8 | Input validation (zod) | Medium | Low |
| 9 | Performance profiling overlay | Medium | Low |
| 10 | Auto-save indicator | Low | Low |
| 11 | Pause on focus loss | Low | Low |
| 12 | Notification center | Medium | Low |
| 13 | Tooltips on all UI elements | Low | Low |
| 14 | Game speed presets | Medium | Low |
| 15 | Auto-pause on disaster | Low | Low |
| 16 | Minimap click-to-pan | Low | Low |
| 17 | Hot-reload for development | Low | Low |
| 18 | Crash recovery (backup restore) | Medium | Low |
| 19 | Automatic backups | Medium | Low |
| 20 | Frustum culling | Medium | Medium |

### 23.8 Dependency-Ordered Roadmap

**Phase 1 (Foundation):** Split App.tsx, spatial indexing, automated testing, save format versioning, localStorage compression, per-system error boundaries, centralized config, input validation, performance profiling, auto-save indicator, pause on focus loss, notification center, tooltips, game speed presets, auto-pause on disaster, minimap click-to-pan, hot-reload, crash recovery, automatic backups, frustum culling.

**Phase 2 (Civilization Core):** Citizen lifecycle (birth/aging/death), skill progression, technology/research tree, quest system, citizen interactions, reputation/fame, event system for citizen interactions, citizen needs (hunger/shelter/employment), home/job assignment, daily routines, citizen models, event sourcing/command pattern, memory management, accessibility (screen reader, high contrast, colorblind), undo for admin actions, screenshot gallery, skill progression, technology tree.

**Phase 3 (Advanced Features):** Diplomacy/relations, modding/extensibility, localization/i18n, keyboard shortcut customization, citizen lifecycle, grid size limits (200×200), citizen count limits (10,000), ECS architecture, Web Worker simulation, multiplayer groundwork.

**Phase 4 (Polish & Scale):** Performance optimization, advanced rendering, full asset library, comprehensive testing, documentation, rollback instructions.

### 23.9 Key Recommendations (challenging assumptions)

1. **Challenge: "No AI models"** — The spec says 100% offline with no LLMs. This is correct for the civilization simulation. But the original app had AI-generated goals/news. The fake AI (template library) is the right replacement. No change needed.

2. **Challenge: "Single settlement"** — The spec focuses on one settlement. But the game already has a 60×60 grid which could support multiple settlements. **Recommendation:** Design the state architecture to support multiple settlements from the start (even if only one is active in v1). This avoids a painful migration later.

3. **Challenge: "Citizens are boxes"** — The spec says citizens are instanced box meshes. The PS1 visual spec says modular sprite system. **Recommendation:** Start with simple low-poly models (head + body) rather than boxes. The instanced rendering already supports this. Priority: Medium.

4. **Challenge: "No save versioning"** — The spec doesn't mention save format versioning. **Recommendation:** Add `version` field to save data from day one. Migration functions for each version. This is critical for long-term maintenance.

5. **Challenge: "Monolithic App.tsx"** — The spec doesn't address code architecture. **Recommendation:** Split App.tsx into simulation engine (pure logic), state management (React context/store), and rendering (R3F scene). This is essential for testability and maintainability.

6. **Challenge: "No automated testing"** — The spec mentions tests but no framework. **Recommendation:** Use Vitest + React Testing Library. Unit tests for simulation logic, integration tests for game loop, E2E tests for critical flows. Target 80% coverage for simulation logic.

7. **Challenge: "localStorage only"** — The spec uses localStorage for saves. **Recommendation:** Use IndexedDB for large saves (citizens, event history, statistics). localStorage for small settings. This avoids the 5-10MB limit.

8. **Challenge: "No spatial indexing"** — The spec doesn't mention spatial data structures. **Recommendation:** Implement a spatial hash or quadtree for O(1) neighbor queries. Critical for performance with 3600+ tiles and hundreds of citizens.

9. **Challenge: "No event sourcing"** — The spec mentions causal event integrity but no event sourcing. **Recommendation:** Implement command pattern for all state changes. Enables undo/redo, replay, and audit trail. Essential for the admin console's "undo" feature.

10. **Challenge: "No configuration management"** — Settings are scattered. **Recommendation:** Centralized config store with schema validation, defaults, and persistence. Single source of truth for all game settings.

## 18. Foliage, Vegetation, Environmental Decoration (refinements)

### 18.1 Foliage System
- Trees (3 types: deciduous, coniferous, palm — cost 25-30, +1 happiness within 3-tile radius, reduces pollution by 0.5, 3 variants per type, instanced rendering). Bushes/Shrubs (2 types — cost 10, +0.5 happiness, 2 variants). Flowers (3 types: red/yellow/purple — cost 5, decorative only, 3 variants). Grass (auto-generated on empty tiles, 3 types, visual only, replaced by buildings). Fallen leaves (autumn particle effect from deciduous trees).
- **Affected files:** `types.ts` (add `FoliageType` enum), `constants.tsx` (foliage configs), `App.tsx` (placement logic, happiness/pollution integration), `components/IsoMap.tsx` (instanced foliage rendering)

### 18.2 Environmental Decoration
- Street lamps (cost 15, light at night, auto-placed on roads). Benches (cost 8, +0.5 happiness within 2-tile radius). Fountains (cost 50, +2 happiness within 3-tile radius, water particles). Statues (cost 100, +1 happiness within 4-tile radius). Street signs (auto-placed on intersections, decorative). Trash cans (auto-placed near commercial). Fire hydrants (auto-placed on road-adjacent tiles).
- **Affected files:** `types.ts` (add `DecorationType` enum), `constants.tsx` (decoration configs), `App.tsx` (placement logic), `components/IsoMap.tsx` (instanced decoration rendering)

### 18.3 Seasonal Changes
- 4 seasons, 90 days each (360-day year). Spring (days 1-90): grass grows, flowers bloom, trees green, rain more frequent, +2 happiness. Summer (91-180): bright lighting, no rain, trees full green, +1 happiness, water demand +20%. Autumn (181-270): leaves change color, leaf particles, grass browns, 0 happiness bonus. Winter (271-360): snow on ground/rooftops, bare deciduous trees, -2 happiness, water demand -20%, snow particles. Season indicator in top bar.
- **Affected files:** `types.ts` (add `Season` enum), `constants.tsx` (season day ranges, bonuses), `App.tsx` (season calculation), `components/IsoMap.tsx` (season-based colors, snow overlay, leaf particles), `components/UIOverlay.tsx` (season indicator)

### 18.4 Weather System (expanded)
- Rain (particles, -1 happiness, +20% water production, washes 1 pollution per tick, 20% chance in spring/autumn). Snow (particles, visual only, 30% chance in winter). Fog (volumetric FogExp2, reduces visibility, 10% chance any season). Thunderstorm (heavy rain + lightning + thunder, -3 happiness, +40% water, washes 2 pollution, 5% chance in spring/summer). Weather indicator in top bar.
- **Affected files:** `types.ts` (add `Weather` enum), `constants.tsx` (weather chances, effects), `App.tsx` (weather roll in game loop), `components/IsoMap.tsx` (particle systems, fog), `components/UIOverlay.tsx` (weather indicator)

### 18.5 Wildlife
- Enhanced birds (seagulls near water, pigeons near commercial, perch on buildings, varied flight). Butterflies (near flowers/parks, 3 colors, spring/summer only). Squirrels (near trees, 2 colors, year-round). Fish (in water tiles, jump occasionally, visual only).
- **Affected files:** `types.ts` (add `WildlifeType` enum), `components/IsoMap.tsx` (instanced wildlife rendering, behavior)

### 18.6 Foliage & Decoration Constants
```ts
export const FOLIAGE_COSTS = { tree_deciduous: 25, tree_coniferous: 25, tree_palm: 30, bush_round: 10, bush_tall: 10, flower_red: 5, flower_yellow: 5, flower_purple: 5 };
export const FOLIAGE_HAPPINESS = { tree_deciduous: 1, tree_coniferous: 1, tree_palm: 1, bush_round: 0.5, bush_tall: 0.5 };
export const FOLIAGE_POLLUTION_REDUCTION = { tree_deciduous: 0.5, tree_coniferous: 0.5, tree_palm: 0.5 };
export const DECORATION_COSTS = { street_lamp: 15, bench: 8, fountain: 50, statue: 100 };
export const DECORATION_HAPPINESS = { street_lamp: 0, bench: 0.5, fountain: 2, statue: 1 };
export const SEASON_DAYS = { spring: [1, 90], summer: [91, 180], autumn: [181, 270], winter: [271, 360] };
export const SEASON_HAPPINESS_BONUS = { spring: 2, summer: 1, autumn: 0, winter: -2 };
export const SEASON_WATER_DEMAND_MULT = { spring: 1.0, summer: 1.2, autumn: 1.0, winter: 0.8 };
export const WEATHER_CHANCES = { rain: 0.2, snow: 0.3, fog: 0.1, thunderstorm: 0.05 };
export const WEATHER_HAPPINESS_EFFECT = { rain: -1, snow: 0, fog: 0, thunderstorm: -3, sunny: 0 };
export const WEATHER_WATER_BOOST = { rain: 1.2, snow: 1.0, fog: 1.0, thunderstorm: 1.4, sunny: 1.0 };
export const WEATHER_POLLUTION_WASH = { rain: 1, snow: 0, fog: 0, thunderstorm: 2, sunny: 0 };
```

### 18.7 Foliage Rendering
- Instanced rendering for trees, bushes, flowers, grass (same pattern as buildings/cars/citizens).
- Season-based color changes: deciduous trees green→yellow→orange→bare, coniferous stays green, grass green→brown→snow-covered.
- Foliage LOD: high=full mesh, medium=simplified, low=colored dot.
- Grass auto-generated on empty tiles: 4 grass blades per tile, random rotation, slight color variation.
- Leaf particles: instanced small quads falling from deciduous trees in autumn.

### 18.8 Foliage BuildingType Additions
```ts
export enum BuildingType {
  // ... existing ...
  Bridge = 10,
  TreeDeciduous = 11, TreeConiferous = 12, TreePalm = 13,
  BushRound = 14, BushTall = 15,
  FlowerRed = 16, FlowerYellow = 17, FlowerPurple = 18,
  StreetLamp = 19, Bench = 20, Fountain = 21, Statue = 22,
}
```

### 18.9 Type Changes (additions)
```ts
// Add to TileData:
rotation?: number;   // 0, 90, 180, 270
tier?: number;       // 0 = base, 1 = upgraded, 2 = max
damaged?: boolean;   // disaster damage state

// New enums:
export enum Season { Spring, Summer, Autumn, Winter }
export enum Weather { Sunny, Rain, Snow, Fog, Thunderstorm }
export enum WildlifeType { Bird, Seagull, Pigeon, Butterfly, Squirrel, Fish }
```

### 18.10 Implementation Tasks (foliage/season/weather)
19. Add foliage types to BuildingType enum and BUILDINGS record
20. Add Season and Weather enums to types.ts
21. Add foliage placement logic to App.tsx (happiness + pollution integration)
22. Add decoration placement logic to App.tsx (happiness integration)
23. Add season calculation to game loop
24. Add weather roll to game loop (with effects on happiness, water, pollution)
25. Add foliage instanced rendering to IsoMap.tsx (season-based colors)
26. Add decoration instanced rendering to IsoMap.tsx
27. Add grass auto-generation to IsoMap.tsx
28. Add leaf particles (autumn) to IsoMap.tsx
29. Add weather particle systems (rain, snow, fog, thunderstorm) to IsoMap.tsx
30. Add season/weather indicators to UIOverlay.tsx
31. Add wildlife enhancement (seagulls, pigeons, butterflies, squirrels, fish) to IsoMap.tsx
32. Run `npm install && npm run lint`

## 20. Integrated Administration, Configuration & Developer Console

### 20.1 Three Operational Areas
Administration (direct control over running civilization), Configuration (control over simulation rules), Developer Tools (inspection, profiling, debugging, experiments).

### 20.2 Access Levels
Player (normal gameplay only), Administrator (manage entities, modify settings, trigger events), Developer (debugging, profiling, state manipulation, replay, testing). World owner has admin+dev by default. Record which administrator initiated each privileged command.

### 20.3 In-Game Control Center (10 sections)
Overview, Citizens, Economy, Government, World, Simulation, Chaos, Developer, History, Configuration. Functional tabs, searchable entity lists, filters, contextual actions, detailed inspectors. Selecting entity on map opens corresponding inspector.

### 20.4 Citizen Administration
View complete authoritative citizen state. Search/filter by name, household, occupation, needs, status. Inspect employment, finances, needs, skills, goals, memories, relationships. Follow movement and decision-making. Inspect action plans and prerequisites. Change employment via validated hiring/termination. Adjust attributes within documented ranges. Transfer citizens between valid households/residences. Spawn new citizens using valid initialization rules. Retire, relocate, or remove citizens with explicit cleanup. Apply controlled interventions to selected populations. Bulk operations with previews, affected-entity counts, confirmation for destructive changes. Never leave orphaned households, invalid jobs, or unresolved ownership references.

### 20.5 Economic Administration
Inspect business accounts, financial health, employment, unemployment, wages, vacancies, supply chains, production, inventories, shortages, pricing, market demand, household spending, public revenue/expenditure, transaction histories. Adjust documented economic policies and production parameters. Controlled financial interventions (fund struggling business via explicit ledger transaction with defined source). Dedicated administrative issuance account or explicitly modeled source/sink for sandbox currency. No silent balance increases. All financial intervention auditable.

### 20.6 Government Administration
Set income tax rates. Inspect public revenue. View and authorize expenditure. Configure public service priorities. Adjust maintenance budgets. Monitor infrastructure condition. Trigger government-funded projects where implemented. Review policy consequences. Policy changes via versioned configuration events. Show when change takes effect. Do not retrospectively alter finalized transactions.

### 20.7 World Administration
Inspect buildings and occupants, roads and accessibility, production facilities, resource deposits, construction projects, ownership/operational status, active incidents. Controlled world edits: place supported buildings, remove buildings with dependency validation, open/close roads, repair damaged infrastructure, adjust resources via recorded interventions, modify building operating parameters, trigger construction/demolition workflows. Separate normal gameplay construction from administrative world editing. Administrative construction may bypass gameplay costs only when sandbox privileges explicitly allow. Bypass still creates valid entity state and provenance records.

### 20.8 Simulation Configuration
Editable controls for: Time (speed, pause, resume, single-tick, day length, scheduling), Citizen Behavior (needs decay, decision intervals, goal priorities, personality influence, memory retention, variability), Economy (wages, tax rates, production coefficients, pricing limits, market responsiveness, operating expenses), Population (initialization, migration eligibility, limits, household rules), Environment (resource regeneration, environmental risk, infrastructure deterioration, construction constraints), Performance (detail levels, update frequencies, scheduling budgets, diagnostic overhead). Every parameter: documented type and valid range, default value, explanation of effect, validation before application, defined effective time, persistence rule, reset-to-default. Categorize as: Live (safe immediately), Tick-boundary (next tick), Paused-only (requires paused), Restart-required (cannot safely affect running simulation).

### 20.9 Chaos Administration (Chaos Laboratory)
Controls: enable/disable chaos, intensity 0-100, reality mode (REALISTIC/HEIGHTENED/SURREAL), incident frequency and severity, max concurrent incidents, event cooldowns, cascade propagation, recovery behavior, incident eligibility, random seed, manual event injection. Incident catalogue: each event displays prerequisites, possible targets, estimated severity, expected affected systems, recovery mechanics. Preview → Trigger → Observe workflow. Administrators can select specific building, citizen, road, or district as target. Manual triggers bypass ordinary probability/cooldown only through explicit override. Never bypass invariant validation. Optional Chaos Director mode: chooses incidents according to world's changing stability. Must obey selected realism mode and use actual world conditions. Presets: Peaceful Settlement, Realistic Civilization, Economic Turbulence, Disaster-Prone World, Unstable Reality, Total Absurdity. Presets editable and exportable.

### 20.10 Developer Inspection Tools
State Inspector (browse entities, components, references, inventories, accounts, action states; show current/previous values; validated edit controls). Decision Inspector (candidate goals, utility scores, chosen goal, preconditions, planned actions, rejected alternatives, failure reasons, replanning triggers). Economic Inspector (monetary flows, inventory movements, supply chains, transaction failures, conservation reconciliations). Spatial Debugging (overlay walkable/blocked tiles, navigation paths, citizen destinations, road connectivity, business service areas, resource locations, movement reservations). Event Inspector (trace events from originating actions to downstream consequences, parent/child relationships). Performance Profiler (measured values for simulation update duration, rendering duration, active citizens, decisions evaluated, queued events, pathfinding work, entity counts, memory usage, slow systems).

### 20.11 Simulation Experiments
Sandbox experimentation workflow: (1) save/snapshot current world, (2) create separate experimental branch, (3) adjust parameters, (4) inject selected events, (5) advance specified ticks, (6) inspect outcomes and metrics, (7) compare with baseline, (8) discard or adopt resulting state. Keep experiments isolated from main save until explicitly committed. Comparisons use actual completed simulation runs. Support reproducible experiment identifiers, seeds, configuration, ordered intervention logs.

### 20.12 Administrative Safety
All privileged modifications use validated commands. Each command defines: required permission, target entities, expected state version, preconditions, proposed changes, invariant checks, resulting events, audit metadata, reversal/recovery strategy. Preview and confirmation for high-impact actions. Atomic application for multi-entity commands. Append-only administrative audit history. Support undo only when true inverse operation is safe; otherwise require checkpoint restoration or compensating transaction. Restoring checkpoint must also restore/branch relevant RNG state, event queue, simulation history. Never modify authoritative world state directly through a UI component.

### 20.13 Configuration Management
In-game configuration manager with: current settings, documented defaults, named presets, preset comparison, changed-value indicators, search/category filters, import/export, schema validation, save-specific overrides, reset options. Separate configuration layers (deterministic precedence): (1) engine defaults, (2) selected simulation preset, (3) current world overrides, (4) temporary experiment overrides. Include configuration schema version and migrations. Prevent incompatible settings from being silently applied.

### 20.14 First Vertical Slice (10 items)
1. Overview: population, employment, businesses, treasury, simulation time, active incidents.
2. Citizen Inspector: select any citizen, inspect needs, job, finances, location, goal, current action.
3. Economy Inspector: inspect farm, bakery, market, inventories, account balances, transaction history.
4. Simulation Controls: pause, resume, speed adjustment, single-tick execution.
5. Configuration: edit income tax rate and at least one documented citizen decision parameter.
6. Chaos Laboratory: edit intensity and reality mode, manually trigger Equipment Breakdown, Road Closure, Bread Rain, inspect consequences.
7. Developer Inspector: inspect authoritative state, decision reasoning, failed commands, invariant status.
8. Audit History: review administrator actions (actor, target, tick, parameters, outcome, resulting events).
9. Persistence: save and reload all administrative settings and active world state.
10. Everything listed must have real functionality. No placeholder buttons, fake controls, or simulated diagnostic values.

### 20.15 Acceptance Tests (15 items)
Verify that: (1) Normal players cannot execute privileged commands. (2) Administrators can access required in-game controls. (3) Configuration changes obey declared application timing. (4) Invalid configuration values rejected. (5) Financial interventions preserve auditable accounting. (6) Citizen modifications cannot create invalid references. (7) Chaos controls affect actual incident eligibility. (8) Surreal incidents cannot run under REALISTIC mode. (9) Manual incident injection recorded and reproducible. (10) Pausing prevents simulation progression. (11) Single-step executes exactly one authoritative simulation tick. (12) Snapshot experiments do not mutate original world. (13) Administrative actions survive save/load. (14) Identical administrative input sequences replay deterministically. (15) State and accounting invariants remain valid. Developer diagnostics accurately reflect actual simulation.

### 20.16 Design Requirements
Dense, practical developer interface. Dark, low-distraction appearance consistent with existing game. Clear information hierarchy, compact controls, readable metrics, searchable tables, informative status indicators. Avoid unnecessary animations, oversized decorative panels, glass effects, gradients. Responsive at supported resolutions. Don't interrupt normal gameplay unless administrator opens modal operation requiring confirmation.

### 20.17 Key Implementation Notes
- Every game system that can be configured should eventually have appropriate in-game administration interface.
- Engine remains authoritative. In-game console becomes primary control surface for administrators.
- Terminal commands and raw config editing may remain optional dev interfaces, but ordinary administration must not depend on them.
- Preserve strict first vertical slice and its invariants.
- Build, validate, deliver essential administration controls before expanding to advanced experimentation or full civilization management.

### 20.18 Implementation Tasks (admin console)
33. Create admin control center component (tabbed interface)
34. Implement Overview tab (population, employment, businesses, treasury, time, incidents)
35. Implement Citizen inspector (select, inspect, follow, modify, spawn, remove)
36. Implement Economy inspector (businesses, inventories, transactions, finances)
37. Implement Government inspector (tax, treasury, spending, policies)
38. Implement World inspector (buildings, roads, resources, construction)
39. Implement Simulation controls (pause, resume, speed, single-tick)
40. Implement Configuration panel (time, behavior, economy, population, environment, performance)
41. Implement Chaos Laboratory (intensity, reality mode, manual triggers, incident catalogue, presets)
42. Implement Developer inspector (state, decisions, economics, spatial, events, performance)
43. Implement Simulation experiments (snapshot, branch, adjust, advance, compare, adopt/discard)
44. Implement administrative safety (validated commands, preview/confirm, atomic application, audit history)
45. Implement configuration management (current settings, defaults, presets, import/export, schema validation)
46. Run `npm install && npm run lint`

## 21. World Forking (prioritized feature)

### 21.1 Concept
Clone the current civilization into two separate saves, introduce a different disaster or policy in each, and compare how their histories diverge. Turns the entire world builder into an experimentation environment.

### 21.2 Implementation
- **Fork button** in admin console: creates a deep copy of current WorldState, CitizenState, HouseholdState, BusinessState, GovernmentState, EconomyState, ChaosState, SimulationState.
- Fork gets a new save slot ID (e.g., `sky_metropolis_fork_1`, `sky_metropolis_fork_2`).
- Each fork runs independently with its own simulation clock, RNG state, and event history.
- Forks can be renamed, deleted, or merged back (adopt fork state into main save).
- Comparison view: side-by-side metrics (population, treasury, happiness, pollution, buildings, disasters) with divergence chart.
- Fork lineage tracked: parent save ID, fork tick, fork reason, fork parameters.

### 21.3 State Forking Rules
- Deep copy all state domains (no shared references).
- RNG state branched with new seed derived from parent seed + fork index.
- Event history copied up to fork tick, then diverges.
- Chaos configuration copied, but incidents are fork-specific after fork point.
- Administrative audit history copied up to fork tick.

### 21.4 UI
- Fork button in admin console header (with confirmation dialog).
- Fork list in save/load menu (shows parent, fork tick, divergence metrics).
- Comparison view: toggle between fork and main save, side-by-side stats, divergence chart.
- Merge button: adopt fork state into main save (with confirmation, creates backup of main save first).

### 21.5 Implementation Tasks
47. Implement deep copy of all state domains (WorldState, CitizenState, etc.)
48. Implement fork creation (new save slot, RNG seed derivation, event history copy)
49. Implement fork management UI (list, rename, delete, compare)
50. Implement comparison view (side-by-side metrics, divergence chart)
51. Implement merge (adopt fork state into main save with backup)
52. Run `npm install && npm run lint`

### 21.6 Validation
- Fork creates independent save that runs without affecting original.
- Fork at tick N, advance both by M ticks with different policies → divergent states.
- Deterministic replay: same fork inputs → identical fork state.
- Save/load preserves fork lineage and divergence data.
- Merge creates backup of main save before overwriting.

## 22. PlayStation-Era Visual Identity & Rendering System

### 22.1 Visual Identity
PS1-era isometric RPG aesthetics with optional enhanced PS1/early PS2 fidelity. Isometric 2D environments with convincing 3D depth. Pixel-art textures and low-poly-inspired architecture. Rich but controlled color palettes. Distinctive building silhouettes and environmental details. Small expressive animated citizens. Soft atmospheric depth and directional shadows. Visually interesting neighborhoods that evolve over time. Carefully controlled retro rendering artifacts (dithering, texture quantization, low-res shadows). Avoid generic mobile-game graphics, overly polished vector artwork, smooth plastic surfaces, arbitrary pixelation. No copyrighted sprites/textures/designs.

### 22.2 Rendering Architecture
Maintain existing isometric 2D architecture (R3F + Three.js). Support consistent isometric projection, correct depth sorting via entity anchor positions, terrain elevation, building/character occlusion, camera pan/zoom, pixel snapping at appropriate zoom levels, crisp texture scaling, render-layer separation (terrain/buildings/characters/effects/interface), efficient sprite atlases and batching. Use configurable low-resolution internal world-rendering target scaled to display. Preserve sharp readable UI independently of retro world-rendering resolution. Optional PlayStation-inspired texture jitter only where technically meaningful. Avoid unnecessary blur, extreme filtering, aggressive screen-space effects.

### 22.3 Terrain and Environments
Unified terrain asset system: grass and varied ground cover, dirt and footpaths, stone and paved roads, farmland and cultivated soil, water and shorelines, rocks and resource deposits, trees/bushes/vegetation, foundations and construction sites, environmental wear and damage. Tiles must connect naturally (no seams, repetitive patterns, incompatible lighting). Procedural tile variation with deterministic seeded selection. Small environmental details communicate land use and history. Terrain appearance reflects actual simulation data.

### 22.4 Architecture
Recognizable architectural language. Buildings feel like small carefully modeled 3D objects represented through pixel-art sprites or compatible low-poly-inspired rendering. Visual variation in: roof shape/material, wall textures, windows/doors, chimneys, signs/storefronts, building height, decorative details, maintenance condition, construction progress. Initially prioritize: residential homes, farm, bakery, market, town hall, roads and infrastructure. Consistent perspective, proportions, lighting, tile footprints. Coherent asset pipeline for future variations. No colored placeholder rectangles as final implementation.

### 22.5 Citizen Visual Design
Modular sprite system supporting variations in: clothing, hair, body proportions, occupation, age presentation, accessories, color combinations, walking/working animations. Appearance generation deterministic from persistent citizen attributes or saved appearance state. Citizen retains appearance after save/load. Initially support: idle, walk, work, eat, rest, interact, react to danger. Appropriate directional animations for camera/movement system. Animations correspond to real activity states (no working animations when unemployed).

### 22.6 A World That Looks Alive
Environment visibly reflects simulation activity. Morning: citizens leave homes, travel to workplaces, businesses begin operating. Working hours: farmers tend crops, bakery workers produce, customers visit market. Evening: citizens return home, some buildings close, residential activity increases. Economic downturn: businesses may close, storefronts inactive, buildings show neglect if maintained condition declines. Prosperity: construction increases, businesses expand where supported, neighborhoods more developed. All changes correspond to authoritative simulation state.

### 22.7 Day, Night, Weather and Atmosphere
Coherent atmospheric rendering system. Day/night: warm morning lighting, clear daytime illumination, long evening shadows, darker nighttime palettes, building/street illumination. Environmental effects: clouds and atmospheric shadows, rain, wind-driven vegetation, fog, dust, storm conditions, fire and smoke, water/surface effects. Weather effects reflect actual simulated weather when gameplay system exists. Visual-only ambient effects explicitly cosmetic. Restrained lighting and particles. Maintain readability of citizens, road connectivity, important buildings.

### 22.8 Visual Consequences
Environment communicates consequences of civilization simulation. Damaged building looks damaged. Closed business looks inactive. Repaired road visibly returns to usable condition. Construction project advances through visible stages. Resource depletion produces appropriate environmental changes. Citizens exhibit recognizable reactions to significant events. Avoid visual changes contradicting authoritative simulation.

### 22.9 Chaos Visualization
Integrate chaos engine with rendering system. Realistic chaos: equipment breakdowns → stopped machinery, repair activity; road closures → physical obstructions, altered movement; disasters → damage and recovery visuals. Surreal chaos: Bread Rain (pixel-art bread falls from sky, accumulates at valid locations, collectible per simulation rules), Goose Uprising (animated geese occupy roads, disrupt traffic, visible disorder), Time Distortion (localized visual distortion where temporal anomaly active), Sentient Architecture (buildings animate/exhibit unusual behavior while maintaining valid footprints/collision). Surreal visuals may exaggerate reality but must not corrupt world state or rendering. Only implement visual effects for chaos incidents supported by actual engine. For first slice, prioritize Equipment Breakdown, Road Closure, Bread Rain.

### 22.10 Visual Style Presets
Configurable rendering presets in admin/developer console:
- **Authentic PS1:** low internal rendering resolution, nearest-neighbor scaling, restricted color precision, pronounced pixel textures, dithering, simplified lighting, conservative animation complexity.
- **Enhanced PS1 (default):** PS1-inspired art direction, sharper environment detail, more varied character animation, improved atmospheric lighting, better shadow quality, comfortable modern display scaling.
- **Early PS2:** higher-resolution textures/sprites, more environmental detail, smoother animations, more sophisticated lighting, still consistent with established isometric visual identity.
Presets are stylization configurations, not hardware emulators. Must use implemented rendering capabilities.

### 22.11 In-Game Graphics Configuration
Functional controls for: visual preset, internal rendering resolution, pixel scaling mode, camera zoom, texture filtering, dithering intensity, color quantization, lighting quality, shadow quality, particle density, weather visual intensity, day/night brightness, animation quality, retro texture effects, optional CRT display effect, reduced motion, performance statistics. CRT effects and excessive screen distortion disabled by default. Clearly identify options requiring renderer restart or asset reload. Persist settings across sessions.

### 22.12 Administrative Visual Debugging
Developer console visualization tools. Overlays for: tile coordinates, collision boundaries, navigation paths, entity footprints, depth-sorting anchors, occlusion regions, building ownership, citizen destinations, resource locations, active incident areas, rendering performance. Toggle each overlay individually. Allow inspecting visual state of citizen/building alongside authoritative simulation state. Keep technical overlays separate from normal gameplay.

### 22.13 Performance Requirements
Graphical improvements must not undermine simulation performance. Use: sprite atlases, batched rendering, camera culling, reused animation assets, pooled particles, cached terrain layers, controlled lighting updates, efficient depth sorting, LOD techniques. Do not require expensive dynamic lighting or high-poly 3D geometry. Preserve headless simulation support. Benchmark rendering and simulation performance separately.

### 22.14 Strict First Visual Slice
Before creating large art library, completely polish one small playable settlement:
1. Cohesive isometric terrain palette.
2. Original terrain and road assets.
3. Eight visually distinct residential buildings or documented variants.
4. Farm with visible production activity.
5. Bakery with operational and inactive states.
6. Market with inventory/activity indicators.
7. Visually identifiable town hall.
8. Twenty-four persistent citizen appearances.
9. At least idle and walking animation, plus supported work animations.
10. Correct depth sorting and movement.
11. Day/night presentation.
12. Visual effects for three initial chaos incidents.
13. Functional graphics settings.
14. Polished interface consistent with game's art direction.
All visuals must reflect underlying simulation where relevant. Don't build entire world's asset catalogue before reference settlement is functional.

### 22.15 Art and Engineering Validation
Verify: tile alignment and seamless terrain transitions, stable depth sorting during movement, correct occlusion around buildings, consistent sprite scale and perspective, persistent citizen appearance after save/load, animated activity matching real citizen states, road closures matching actual pathfinding restrictions, Bread Rain visuals corresponding to actual generated inventories, day/night and weather presentation responding to real world state, settings changing actual rendering behavior, no unintended regressions in simulation, no unsupported external dependencies, acceptable rendering performance measured on available hardware. Capture comparison screenshots for daytime, nighttime, busy settlement activity, damaged infrastructure, surreal chaos.

### 22.16 Final Art Direction
The world should feel handmade, atmospheric, inhabited, and memorable. A player should be able to zoom into a small settlement and see individual lives unfolding. Buildings should have personality. Neighborhoods should communicate their history. Workplaces should visibly operate. The environment should change as the civilization evolves. Realistic disasters should look consequential. Surreal chaos should look gloriously ridiculous while remaining stylistically coherent. The final aesthetic is a PlayStation-era isometric living-world simulation—not a generic pixel-art city builder and not a modern 3D game disguised with a retro filter.

### 22.17 Suggested Default Preset
Enhanced PS1 (default). Gives old-school PlayStation atmosphere while keeping population, buildings, and economic activity readable as settlement grows.

### 22.18 Implementation Tasks (visual identity)
53. Establish visual specification document (color palette, sprite sizes, perspective angles, lighting direction)
54. Create terrain tile asset set (grass, dirt, stone, paved, farmland, water, shorelines, rocks)
55. Create road asset set (straight, corner, T-junction, cross, bridge)
56. Create building sprite set (8 residential variants, farm, bakery, market, town hall) with operational/inactive/damaged states
57. Create citizen sprite system (modular: clothing, hair, body, accessories, 24 unique combinations)
58. Create citizen animation set (idle, walk 4-dir, work, eat, rest, interact, danger reaction)
59. Implement depth sorting by entity anchor position
60. Implement camera pan/zoom with pixel snapping
61. Implement day/night lighting system (warm morning, clear day, long evening shadows, dark night, building/street lights)
62. Implement weather visual effects (rain particles, snow particles, fog, thunderstorm with lightning)
63. Implement chaos visual effects (equipment breakdown stopped machinery, road closure obstructions, Bread Rain falling bread)
64. Implement visual style presets (Authentic PS1, Enhanced PS1, Early PS2)
65. Implement in-game graphics configuration controls (resolution, scaling, filtering, dithering, color quantization, lighting, shadows, particles, weather intensity, brightness, animation quality, CRT effect, reduced motion, performance stats)
66. Implement administrative visual debugging overlays (tile coords, collision, paths, footprints, depth anchors, occlusion, ownership, destinations, resources, incidents, performance)
67. Implement sprite atlases and batched rendering for performance
68. Implement LOD system for foliage, buildings, citizens
69. Capture comparison screenshots (daytime, nighttime, busy activity, damaged infrastructure, surreal chaos)
70. Run `npm install && npm run lint`
```