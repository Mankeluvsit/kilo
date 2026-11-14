# Voxel-world Revamp Plan

## 1. Context & Problem

`Mankeluvsit/Voxel-world` ("Sky Metropolis") is an isometric 3D city builder (React + Three.js + Vite + Express). Its AI layer currently depends on two live cloud providers:

- **Gemini** — `@google/genai`, model `gemini-3.5-flash` (goals, news, terrain layout, Idea Lab, procedural music, Lyria audio, performance/LOD analysis).
- **OpenAI** — `openai`, model `gpt-4o-mini`, switched via `VITE_AI_PROVIDER`.

Goal: remove both cloud providers, replace the entire AI layer with a local deterministic "fake AI" (template library + selector), and do a full refresh of the app — no API keys required, zero token cost.

## 2. Confirmed Decisions

| Decision | Choice |
|---|---|
| Cloud AI scope | Remove **both** Gemini and OpenAI entirely |
| Fake AI design | **A2** — template library + deterministic selector parameterized by city state |
| Randomness model | **Math.random** — consistent with the rest of the game (terrain, traffic, citizens already use it); no seed plumbing |
| Idea Lab | **B** — pure offline idea board; drop the prompt textarea, drop react-markdown; "Generate Concept" → "Add Random Idea" from pool |
| Music Lab | **A** — keep the lab, drop Lyria mode, keep Tone.js procedural playback with a local template-pool composer |
| UI direction | **B** — keep indigo/sky-900 palette and component structure; strip slop only (gradient text, scanline overlays, emoji icons, stacked glass panels, lucide-only icons) |

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
- `App.tsx` — remove AI key checks, simplify
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

## 6. Implementation Tasks

1. Update `package.json` and remove cloud deps
2. Delete `geminiService.ts`, `openaiService.ts`
3. Create `localAIService.ts` with template pools for goals, news, terrain, ideas, music
4. Rewrite `aiService.ts` to call local functions (synchronous, no fetch)
5. Strip `server.ts` of all `/api/ai/*` routes
6. Clean `vite.config.ts`
7. Refactor UI components (anti-slop)
8. Update `.env.example` and `metadata.json`
9. Run `npm install && npm run lint`

## 7. Validation

- `npm install` succeeds
- `npm run lint` (tsc --noEmit) passes
- `npm run build` succeeds
- App runs without any env vars set
- No `@google/genai`, `openai`, or `react-markdown` imports remain anywhere in the codebase