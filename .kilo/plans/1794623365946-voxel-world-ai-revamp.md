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
4. **Implementation-level details** — all resolved with recommended choices (template pools, performance heuristic, music lab, idea lab, UI overlay). See §10, §12, §15.

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
export const SCREENSHOT_RESOLUTION = 4;  // 4x render scale
```