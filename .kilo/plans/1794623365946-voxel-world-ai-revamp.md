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
| UI | Anti-slop refactor: clean, consistent, no gradients-on-text, no glassmorphism-on-glass, no emoji-as-icons, no scanline overlays |

## 3. Security Bug Found

`vite.config.ts` inlines `GEMINI_API_KEY` and `OPENAI_API_KEY` into the client bundle via `define`. `openaiService.ts` uses `dangerouslyAllowBrowser: true`. Both keys ship to the browser. Removing both providers eliminates this entirely.

## 4. Affected Files

- `package.json` — remove `@google/genai`, `openai`, `react-markdown`, `@types/express` (keep express for static serving)
- `vite.config.ts` — remove the `define` blocks for API keys
- `server.ts` — strip all `/api/ai/*` routes
- `services/aiService.ts` — replace fetch calls with local functions
- `services/geminiService.ts` — delete
- `services/openaiService.ts` — delete
- `services/localAIService.ts` — **new** template library + selector
- `App.tsx` — remove AI key checks, simplify
- `IdeaLab.tsx` — remove react-markdown, local generation
- `components/MusicLab.tsx` — remove Lyria mode, keep procedural
- `components/UIOverlay.tsx`, `components/StartScreen.tsx`, `components/IsoMap.tsx` — UI cleanup
- `.env.example` — remove API key lines

## 5. Implementation Tasks

1. Update `package.json` and remove cloud deps
2. Delete `geminiService.ts`, `openaiService.ts`
3. Create `localAIService.ts` with template pools for goals, news, terrain, ideas, music, performance
4. Rewrite `aiService.ts` to call local functions
5. Strip `server.ts` of all `/api/ai/*` routes
6. Clean `vite.config.ts`
7. Refactor UI components (anti-slop)
8. Update `.env.example`
9. Run `npm install && npm run lint`

## 6. Validation

- `npm install` succeeds
- `npm run lint` (tsc --noEmit) passes
- `npm run build` succeeds
- App runs without any env vars set