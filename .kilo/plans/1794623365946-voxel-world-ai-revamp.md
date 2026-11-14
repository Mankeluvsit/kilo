# LIVING WORLD — Master Implementation Plan

## A. Readiness Verdict & Contradiction Register

### Readiness Verdict: NOT IMPLEMENTATION-READY

The plan contains 23 sections and 70+ tasks but has 10 blocking contradictions and 8 missing implementation contracts. These must be resolved before implementation begins.

### Contradiction Register

| # | Old Clause | Chosen Replacement | Rationale | Affected Requirements |
|---|---|---|---|---|
| 1 | §2 "Math.random — consistent with rest of game; no seed plumbing" | **Seeded persisted RNG** with separate streams (citizen init, routine decisions, economic uncertainty, realistic chaos, surreal chaos). Cosmetic randomness (particles, UI) may use Math.random but must not affect simulation state. | Deterministic replay (INV-09) requires reproducible randomness. Math.random breaks replay. | INV-09, §19.16 |
| 2 | §§7,12,15 expand React game loop first; §23.8 defers engine extraction to Phase 2 | **Establish pure authoritative simulation engine first** (pure TS, no React). UI/rendering consume read-only views and submit validated commands. | Commands are required for invariants (INV-06, INV-07, INV-08). Cannot add systems before the engine boundary exists. | INV-06–08, §19.5, §23.1 |
| 3 | §5c passive building income (`incomeGen`), §10 goal rewards (`rewardFn`), §11.7.H treasury-only upkeep deductions, §20.5 sandbox issuance | **Single conserved-accounting model.** Every money flow has a defined source and destination account. Passive income requires a counterparty (e.g., "city economy" account). Goal rewards debit a "reward pool" account. Upkeep debits building owner, credits maintenance fund. Sandbox issuance uses explicit issuance account with audit trail. | INV-01 (currency conservation) and INV-02 (nonnegative finances) require every flow to have equal debit/credit. | INV-01, INV-02, §19.3, §19.4 |
| 4 | §8-9 "defaulting corrupt saves to new world, silently failing on quota exhaustion" | **Versioned migration with validated staged writes.** Save to temp key → validate → swap. On corruption: retain original, attempt migration, report failure. On quota: report error, offer backup export. Never silently lose data. | INV-10 (persistence integrity) requires explicit failure reporting. | INV-10, §19.10 |
| 5 | §5d flood-fill "seeds every road adjacent to any building" | **Reachability-based connectivity.** A building is connected if there exists a path along roads from that building to a defined destination (e.g., town hall, market, or any road tile on the map boundary). Disconnected road islands do not qualify. Test: disconnected island, complete isolation, alternate routes. | Current implementation marks all roads adjacent to any building as "connected" regardless of actual connectivity. | §11.1, §17.4 |
| 6 | §11.4 pollution-only disasters with mandatory demolition | **Replace with §19.8-14 causal incident model.** Equipment Breakdown (production stop, recover via funded repair), Road Closure (route block, recover via repair/reopening), Bread Rain (Surreal only, anomalous bread creation). No mandatory demolition. Eligibility based on exposure, intensity, cooldowns. | Pollution-only disasters conflict with exposure-based incidents and funded repair. | §11.4, §19.14 |
| 7 | §11.7.B grid-only undo (snapshot stack) | **Validated inverses only when safe; otherwise checkpoint branching or compensation.** Grid undo can restore building placement. Money/inventory/employment changes require checkpoint branching (World Forking) or compensating transactions. Never claim arbitrary undo. | Grid-only undo cannot restore money, inventories, occupants, jobs, reservations, RNG, or scheduled events. | §11.7.B, §20.12 |
| 8 | §1/§17 3D meshes vs §22 modular sprite/depth-sorting spec | **Hybrid: orthographic 3D rendering with sprite-based assets.** Keep R3F/Three.js (existing). Buildings/citizens rendered as textured quads (billboards) with depth sorting by anchor position. Terrain as 3D mesh with pixel-art textures. Post-processing for retro aesthetic. This preserves existing rendering while achieving PS1 look. | Existing codebase is R3F. Migration to 2D sprite engine would lose instanced rendering, LOD, and post-processing. | §1, §17, §22 |
| 9 | §21.3 reseed forks (new seed from parent + fork index) | **Controlled comparison mode:** preserve initial RNG states for identical no-intervention branches. Offer reseeded exploration as separate "experimental" mode, labeled distinctly. "Merge" = adopt-as-replacement with backup, not a true merge. | Reseeding makes policy comparisons differ in both intervention and randomness. Cannot isolate policy effects. | §21.3, §21.4 |
| 10 | §§12,18.10,20.18,21.5,22.18,23.8 competing roadmaps | **Single gated backlog** (see §B). Admin mutation UI follows its safety task. Foliage/progression deferred until first economy passes. One dependency-ordered task graph. | Competing roadmaps cause admin mutation UI to precede safety task. | §12, §18.10, §20.18, §21.5, §22.18, §23.8 |

### Security Completeness

- **Client-bundled API keys (§3):** Removing providers prevents future inclusion but does not establish whether keys were previously exposed. Add read-only exposure assessment: check git history for committed keys, check built assets/source maps, check runtime network activity. Owner action: revoke/rotate confirmed exposed credentials without displaying secrets.
- **Trust model:** In offline browser-owned world, roles enforce application workflows but cannot protect against device owner. If remote/shared authority is intended, authentication/authorization needs trusted enforcement boundary. Do not add multiplayer merely for role labels. Local append-only history is useful provenance, not tamper-proof security.
- **Blueprint/save/config imports:** Need limits on compressed/decompressed size, entity count, dimensions, nesting. Schema/version checks. Safe field handling. Reference validation. Atomic preview/apply path. Base64/compression provide neither trust nor validation. Reject invalid input without overwriting current world.

---

## B. Consolidated Specification

### B.1 Clock & Time

- **Timestep:** 1 simulated minute = 1 tick. 1440 ticks/day. 30 days = 43,200 ticks.
- **Tick phases (deterministic order):** (1) needs decay, (2) movement, (3) work/production, (4) shopping/purchases, (5) wage payments, (6) tax collection, (7) business decisions, (8) chaos events, (9) maintenance/recovery, (10) statistics snapshot, (11) save.
- **Pause:** stops tick advancement. **Step:** executes exactly one tick. **Catch-up:** on tab focus, execute at most 10 ticks then resume normal rate.
- **Testing:** 43,200 ticks headlessly. No real-time waits. Each tick is synchronous.
- **Needs rates:** hunger +0.5/tick, fatigue +0.3/tick (working), -1.0/tick (resting). Financial security based on savings ratio.
- **Work/production periods:** work from tick 480 (08:00) to tick 1080 (18:00). Production completes at end of work period.

### B.2 Economy Bootstrapping

**Initial balances (reference settlement):**
- Each household: $500 starting money
- Farm business: $2000 cash, 0 grain stock
- Bakery business: $1500 cash, 0 grain, 0 bread
- Market business: $1000 cash, 0 bread
- Government treasury: $0
- Total initial currency: 8×500 + 2000 + 1500 + 1000 = $8000

**Prices:** Grain $5/unit, Bread $12/unit. Farm sells grain to bakery at $5. Bakery sells bread to market at $12. Market sells bread to households at $15.

**Recipes:** Bakery: 2 grain → 1 bread, 5 ticks processing time, 1 worker.

**Wages:** Farm worker $8/tick, Bakery worker $10/tick, Market worker $8/tick. Paid at end of work period (tick 1080).

**Tax:** 10% income tax. Employer debited gross wage. Worker receives net (90%). Government receives 10%.

**Hiring thresholds:** Farm hires 1 worker when grain stock < 10 and cash > $100. Bakery hires 1 worker when bread stock < 5 and cash > $150. Market hires 1 worker when bread stock < 3 and cash > $100.

**Payment timing:** Wages paid at end of work period. Tax collected simultaneously. Business expenses (upkeep) deducted at start of day (tick 0). Production costs (grain purchase) deducted when production starts.

**Liquidity bootstrapping:** Farm starts with $2000 cash to pay first worker before first sale. First grain sale to bakery provides revenue. If farm can't pay worker, worker goes unpaid (employment continues, wage debt accumulates? No — INV-02 prevents overdraft. If farm can't pay, worker is not paid, employment may be terminated per business policy).

**Tax recirculation:** Government treasury spends on road maintenance ($50/road/tick). If treasury insufficient, maintenance deferred. No other spending in v1.

**Rounding:** All currency in integer units. No floating-point. Division uses floor. Tax = floor(grossWage * 0.1).

**Insolvency:** If business cash < 0 after a transaction, transaction fails atomically (INV-02). Business may terminate employees if cash insufficient for next wage cycle.

**Repair recipients:** Road maintenance credits "city infrastructure" account (not a citizen account). Building repair credits the business that owns the building.

### B.3 Physical Logistics

**Goods movement:** Goods are transferred between inventories at the moment of purchase. No physical transport simulation in v1. Ownership transfers immediately. Reservation system prevents overselling.

**Reservation rules:** When a buyer initiates purchase, stock is reserved (decremented from available, added to reserved). On successful payment, reserved stock moves to buyer's inventory. On failed payment, reserved stock returns to available. Reservation expires after 5 ticks if not completed.

**Capacities:** No storage capacity limits in v1. Unlimited inventory.

**Delivery delays:** None in v1. Instant transfer.

**Cancellation:** Purchase can be cancelled before payment. Reservation released.

**Blocked routes:** Road closure does not affect goods transfer in v1 (no physical transport). Road closure affects citizen movement only (travel time increases via alternate routes).

### B.4 Citizen Planning (GOAP)

**Action catalogue:** IDLE, TRAVEL_TO_WORK, WORK, TRAVEL_TO_MARKET, BUY_BREAD, EAT, TRAVEL_HOME, REST, SEEK_EMPLOYMENT, APPLY_FOR_JOB.

**Bounded search budget:** Max 50 nodes per planning cycle. Max 5 actions per plan. Max 1 plan per citizen per tick (replan only if current plan fails or priority changes significantly).

**Knowledge boundaries:** Citizen knows: own needs, own location, own employment, own money, publicly listed job openings, market prices (observed), road network (learned through travel). Does not know: other citizens' finances, business internal state, future events, hidden simulation state.

**Utility ties:** If two actions have equal utility, choose the one with lower cost (time). If still tied, choose by entity ID (deterministic).

**Reservation expiry:** Job application reservation expires after 10 ticks. Market purchase reservation expires after 5 ticks.

**Failure/replanning rules:** If an action fails (e.g., market out of stock), citizen replans immediately. If travel is blocked (road closure), citizen replans with alternate route. If no feasible plan exists, citizen goes IDLE and retries next tick.

**Memory:** Bounded to 20 entries. Each entry: event type, tick, affected entity, outcome. Oldest entries evicted when full.

### B.5 Time & Scheduling (detailed)

- **Canonical clock:** `simulationTick: number` (monotonically increasing). `day = floor(simulationTick / 1440)`. `hour = floor((simulationTick % 1440) / 60)`. `minute = simulationTick % 60`.
- **Needs decay rates:** hunger +0.5/tick, fatigue +0.3/tick (working), -1.0/tick (resting at home), -0.5/tick (idle at home).
- **Work period:** ticks 480-1080 (08:00-18:00). Production completes at tick 1080.
- **Wage payment:** tick 1080 (end of work period).
- **Tax collection:** tick 1080 (simultaneous with wage payment).
- **Business decisions:** tick 0 (start of day) — hire/fire, adjust prices, schedule production.
- **Chaos events:** tick 0 (start of day) — eligibility check, trigger if conditions met.
- **Maintenance:** tick 0 (start of day) — deduct upkeep, repair damaged buildings.
- **Statistics snapshot:** tick 1439 (end of day) — record population, money, happiness.
- **Save:** every 60 ticks (1 hour) — auto-save to localStorage/IndexedDB.
- **Pause:** stops tick advancement. Simulation state frozen.
- **Step:** executes exactly one tick (all 11 phases).
- **Catch-up:** on tab focus, execute at most 10 ticks then resume normal rate (2000ms/tick).

### B.6 State Ownership & Replay

**Canonical owners:**
- `WorldState`: owns map, tiles, buildings, roads, resources, weather, time, seed, RNG state
- `CitizenState`: owns citizen personal data (needs, location, employment, money, memories, goals, actions)
- `HouseholdState`: owns household membership, home, shared finances
- `BusinessState`: owns business cash, inventory, employees, orders, recipes, prices, production status
- `GovernmentState`: owns treasury, tax rates, expenditures, infrastructure
- `EconomyState`: owns accounts, transactions, inventories, orders, prices, production records (derived views, not duplicated)
- `ChaosState`: owns chaos config, active incidents, event history, cooldowns, exposure, consequences
- `SimulationState`: owns entity references, scheduled actions, deterministic ordering, pending commands, persistence metadata

**Command ordering:** Commands processed in FIFO order within a tick. If multiple commands target the same entity, later commands see the updated state.

**RNG algorithm:** `mulberry32` (deterministic, fast, good distribution). Seed: 32-bit integer. State: single 32-bit integer. Separate streams via seed derivation: `streamSeed = baseSeed + streamIndex * 0x9e3779b9`.

**State hashing:** SHA-256 of serialized state (JSON.stringify with sorted keys). Used for replay verification.

**Event retention:** Keep last 10,000 events. Older events archived (compressed) but not deleted.

**Replay compatibility:** Replay requires same simulation version. Version stored in save data. If version mismatch, replay is not guaranteed (but migration may update version).

### B.7 Gameplay Formulas (resolved)

**Happiness:** `happiness = 50 + parksPerCapita*10 + (10-taxRate)*2 - avgPollution*5 - crime*3`, clamped 0-100. Uses previous tick's crime value (no circular dependency).

**Crime:** `crime = max(0, (population/100) - (parksCount*2) - (happiness_prev/20))`. Uses previous tick's happiness (no circular dependency).

**Pollution:** Industry adds pollution to 3-tile radius. Pollution accumulates per tick. Pollution decays by 0.1 per tick (natural dissipation). Rain washes 1 pollution per tick. Thunderstorm washes 2.

**Population growth:** `growthMultiplier = happiness/50`. Population increases by `popGen * growthMultiplier` per residential building per tick. Population decreases if happiness < 20 (citizens leave).

**Reputation vs research unlocks:** Reputation is earned from completed goals (3 rep per goal). Research points are earned from library/university buildings. Reputation unlocks buildings (Industrial=5, Extractor=10, Advanced=20). Research unlocks production recipes and upgrades. Separate systems, separate currencies.

**Repair vs demolition:** Damaged buildings can be repaired (cost = 50% of build cost, deducts from building owner's account, credits maintenance fund). If repair not funded within 10 ticks, building is auto-demolished (refund = 0, no credits). Demolish tool allows manual demolition (refund = 50% of build cost).

**Tax units:** Tax = floor(grossWage * 0.1). Integer arithmetic. No floating point.

### B.8 Migration & Regressions

**Legacy save mapping:**
- Old saves (v0, no version field) → migrate to v1: add `happiness: 50`, `pollution: 0` to tiles, add `reputation: 0`, add `version: 1`.
- Old buildings without `pollution`/`upkeep` → default to 0.
- Old citizens without `needs`/`skills`/`memories` → default to initial values.
- Old goals without `rewardRep` → default to 1.

**Intentional removals (not regressions):** Cloud AI providers (Gemini, OpenAI), Lyria audio mode, react-markdown, AI key checks. These are explicitly removed, not broken.

**Unmatched entities:** If a legacy save contains entities not in the new schema, log a warning and skip. Do not crash.

**Preservation:** Always retain original save data before migration. If migration fails, restore original.

### B.9 Security Boundary

- **Local owner-controlled roles:** Application-level enforcement only. Device owner can bypass. This is acceptable for offline single-player.
- **Permission enforcement:** At command boundary. Every privileged command checks `accessLevel >= requiredLevel`. If insufficient, command rejected with audit log entry.
- **Client-key exposure assessment:** Read-only check of git history, built assets, source maps, runtime network activity. If keys found, owner must revoke/rotate. Do not display secrets.
- **Import validation:** Limits on compressed/decompressed size (max 5MB), entity count (max 10,000), dimensions (max 200×200), nesting depth (max 10). Schema/version checks. Safe field handling (no eval, no prototype pollution). Reference validation (all entity IDs must resolve). Atomic preview/apply (validate first, apply only if valid).

### B.10 Rendering Architecture

**Chosen approach: Hybrid orthographic 3D with sprite-based assets.**

- **Terrain:** 3D mesh with pixel-art textures. Height variation via vertex displacement. Water plane with animated texture.
- **Buildings:** Textured quads (billboards) with depth sorting by anchor position. 3D mesh for shadows. Sprite-based for PS1 aesthetic.
- **Citizens/Cars:** Instanced textured quads with directional animation (4 directions). Depth sorting by anchor.
- **Foliage:** Instanced meshes (trees, bushes, flowers, grass) with season-based color changes.
- **Post-processing:** Bloom (glowing windows), SSAO (depth), color grading (retro palette). Via `@react-three/postprocessing`.
- **Camera:** Orthographic, pan/zoom with pixel snapping at low zoom levels. Smooth zoom (lerp). Camera shake on disaster.
- **Pixel scaling:** Configurable internal resolution (1x, 2x, 3x, 4x) with nearest-neighbor scaling. UI rendered at full resolution.
- **Lighting:** Directional light (sun) with shadow map. Ambient light. Point lights for street lamps/buildings at night. Day/night cycle modifies light color/intensity.
- **Asset pipeline:** PNG spritesheets (texture atlases). Deterministic seeded selection for variants. Asset manifest JSON for loading.
- **First-slice assets:** 8 residential variants, farm, bakery, market, town hall, terrain tiles (grass, dirt, stone, paved, farmland, water, shorelines, rocks), road set (straight, corner, T-junction, cross, bridge), citizen sprites (24 unique combinations with 4-dir idle/walk/work), car sprites (3 types), foliage (3 tree types, 2 bush types, 3 flower types, grass), weather particles (rain, snow, fog), UI icons (lucide-react).

---

## C. Dependency-Ordered Task Graph

### Phase 0: Baseline & Security
| ID | Task | Dependencies | Files | Acceptance Test | Rollback |
|---|---|---|---|---|---|
| P0-01 | Read-only source inventory | None | All source files | Document all existing functionality, architecture, dependencies | N/A (read-only) |
| P0-02 | Baseline regression fixtures | P0-01 | New test files | Capture current behavior as test expectations | Delete test files |
| P0-03 | Conflict/decision register | P0-01 | Plan file | Document all contradictions and chosen resolutions | Revert plan |
| P0-04 | Cloud exposure assessment | P0-01 | Git history, built assets | Report whether keys were exposed (read-only, no secrets displayed) | N/A (read-only) |
| P0-05 | Owner credential rotation | P0-04 | N/A (owner action) | If keys exposed, owner revokes/rotates. Verify no cloud calls in built assets. | N/A |

### Phase 1: Offline Replacement
| ID | Task | Dependencies | Files | Acceptance Test | Rollback |
|---|---|---|---|---|---|
| P1-01 | Remove cloud providers | P0-05 | package.json, vite.config.ts, server.ts, services/geminiService.ts, services/openaiService.ts, services/aiService.ts | `npm install` succeeds, no `@google/genai`/`openai` imports, app runs without env vars | Restore deleted files, reinstall deps |
| P1-02 | Create localAIService.ts | P1-01 | services/localAIService.ts (new) | All 6 functions return valid data, no fetch calls, no async | Delete file |
| P1-03 | Rewrite aiService.ts | P1-02 | services/aiService.ts | All callers work synchronously, no async/await | Restore original |
| P1-04 | Strip server.ts AI routes | P1-01 | server.ts | `/api/health` works, no `/api/ai/*` routes | Restore original |
| P1-05 | Clean vite.config.ts | P1-01 | vite.config.ts | No `define` blocks for API keys | Restore original |
| P1-06 | Update App.tsx/IdeaLab.tsx | P1-03 | App.tsx, IdeaLab.tsx | No await on AI calls, no key checks | Restore original |
| P1-07 | Update MusicLab.tsx | P1-03 | components/MusicLab.tsx | Lyria mode removed, procedural works | Restore original |
| P1-08 | Update .env.example/metadata.json | P1-01 | .env.example, metadata.json | No API key references | Restore original |
| P1-09 | Verify no cloud calls | P1-01 | Built assets | Network tab shows no external calls, source maps clean | N/A |

### Phase 2: Deterministic State Engine
| ID | Task | Dependencies | Files | Acceptance Test | Rollback |
|---|---|---|---|---|---|
| P2-01 | Extract simulation engine | P1-09 | simulation/engine.ts, simulation/state.ts (new) | Engine runs headlessly, no React imports, produces identical state on replay | Delete files, restore App.tsx |
| P2-02 | Implement command pattern | P2-01 | simulation/commands.ts (new) | All state changes go through commands, each has execute/undo | Delete file |
| P2-03 | Implement seeded RNG | P2-01 | simulation/rng.ts (new) | Same seed → same sequence, separate streams | Delete file |
| P2-04 | Implement invariants | P2-01 | simulation/invariants.ts (new) | All 14 invariants checked per tick, violations logged | Delete file |
| P2-05 | Implement spatial indexing | P2-01 | simulation/spatial.ts (new) | O(1) neighbor lookup, 3600-tile grid benchmarks | Delete file |
| P2-06 | Implement clock/scheduling | P2-01 | simulation/engine.ts | Fixed timestep, tick phases in order, pause/step/catch-up | Revert |
| P2-07 | Implement save format v1 | P2-01 | simulation/save.ts (new) | Versioned save, migration from v0, atomic write | Delete file |
| P2-08 | Implement configuration store | P2-01 | simulation/config.ts (new) | Centralized config, schema validation, defaults | Delete file |
| P2-09 | Implement error boundaries | P2-01 | components/ErrorBoundary.tsx (new) | Per-system error boundaries, graceful degradation | Delete file |
| P2-10 | Implement structured logging | P2-01 | services/logService.ts (new) | Log levels, component tags, optional file output | Delete file |
| P2-11 | Implement automated testing | P2-01 | vitest.config.ts, test files (new) | Vitest runs, 80% coverage for simulation logic | Delete test files |
| P2-12 | Implement performance profiling | P2-01 | components/PerfOverlay.tsx (new) | Dev-mode overlay showing FPS, tick duration, entity counts | Delete file |

### Phase 3: Complete Economic Loop
| ID | Task | Dependencies | Files | Acceptance Test | Rollback |
|---|---|---|---|---|---|
| P3-01 | Implement economy module | P2-01 | simulation/economy.ts (new) | Farm produces grain, bakery converts to bread, market sells, currency conserved | Delete file |
| P3-02 | Implement citizen module | P2-01 | simulation/citizens.ts (new) | Citizens travel, work, earn wages, pay tax, buy bread, eat, rest | Delete file |
| P3-03 | Implement building module | P2-01 | simulation/buildings.ts (new) | Building yields, upgrades, damage, repair, demolition | Delete file |
| P3-04 | Implement GOAP planner | P3-02 | simulation/goap.ts (new) | Citizens select actions via utility + GOAP, bounded search, replanning | Delete file |
| P3-05 | Implement road connectivity | P3-01 | simulation/spatial.ts (update) | Flood-fill connectivity, disconnected islands don't qualify, buildings marked connected | Revert |
| P3-06 | Implement pollution system | P3-01 | simulation/pollution.ts (new) | Industry adds pollution to 3-tile radius, happiness penalty, decay | Delete file |
| P3-07 | Implement happiness/crime | P3-01, P3-06 | simulation/happiness.ts (new) | Happiness formula with previous-tick crime, crime formula with previous-tick happiness | Delete file |
| P3-08 | Implement population growth | P3-02, P3-07 | simulation/population.ts (new) | Population grows based on happiness, decreases if unhappy | Delete file |
| P3-09 | Implement maintenance costs | P3-01, P3-03 | simulation/maintenance.ts (new) | Upkeep deducted per tick, buildings unpowered if can't pay | Delete file |
| P3-10 | Implement tech/unlock tree | P3-01 | simulation/tech.ts (new) | Reputation unlocks buildings, research unlocks recipes | Delete file |
| P3-11 | Implement blueprints | P3-01 | simulation/blueprints.ts (new) | Export/import grid as base64 compressed JSON | Delete file |
| P3-12 | Implement undo/redo | P2-02 | simulation/undo.ts (new) | Grid placement undo via command pattern, checkpoint for money/inventory | Delete file |
| P3-13 | Integration test: 30-day run | P3-01 to P3-12 | test files | 43,200 ticks headless, all invariants pass, economy reconciles | N/A |

### Phase 4: Chaos Engine
| ID | Task | Dependencies | Files | Acceptance Test | Rollback |
|---|---|---|---|---|---|
| P4-01 | Implement chaos engine | P3-13 | simulation/chaos.ts (new) | Eligibility, incident state, recovery, 3 events | Delete file |
| P4-02 | Equipment Breakdown event | P4-01 | simulation/chaos.ts (update) | Production stops, inventory shortages, recover via funded repair | Revert |
| P4-03 | Road Closure event | P4-01 | simulation/chaos.ts (update) | Route blocked, alternate routes, travel time increases, recover via repair | Revert |
| P4-04 | Bread Rain event (Surreal) | P4-01 | simulation/chaos.ts (update) | Anomalous bread creation, collection rules, market effects, only in Surreal | Revert |
| P4-05 | Chaos config | P4-01 | simulation/chaos.ts (update) | Intensity 0-100, reality mode, presets, validation | Revert |
| P4-06 | Chaos Director | P4-01 | simulation/chaos.ts (update) | Chooses incidents based on stability, obeys realism mode | Revert |
| P4-07 | Invariant stress tests | P4-01 | test files | Repeated purchases, job competition, insufficient funds, shortages, overlapping incidents | N/A |

### Phase 5: Admin Console
| ID | Task | Dependencies | Files | Acceptance Test | Rollback |
|---|---|---|---|---|---|
| P5-01 | Admin control center | P3-13 | components/AdminConsole.tsx (new) | Tabbed interface, 10 sections | Delete file |
| P5-02 | Overview tab | P5-01 | components/admin/Overview.tsx (new) | Population, employment, businesses, treasury, time, incidents | Delete file |
| P5-03 | Citizen inspector | P5-01 | components/admin/CitizenInspector.tsx (new) | Select, inspect, follow, modify, spawn, remove | Delete file |
| P5-04 | Economy inspector | P5-01 | components/admin/EconomyInspector.tsx (new) | Businesses, inventories, transactions, finances | Delete file |
| P5-05 | Government inspector | P5-01 | components/admin/GovernmentInspector.tsx (new) | Tax, treasury, spending, policies | Delete file |
| P5-06 | World inspector | P5-01 | components/admin/WorldInspector.tsx (new) | Buildings, roads, resources, construction | Delete file |
| P5-07 | Simulation controls | P5-01 | components/admin/SimulationControls.tsx (new) | Pause, resume, speed, single-tick | Delete file |
| P5-08 | Configuration panel | P5-01 | components/admin/ConfigurationPanel.tsx (new) | Time, behavior, economy, population, environment, performance | Delete file |
| P5-09 | Chaos Laboratory | P5-01 | components/admin/ChaosLab.tsx (new) | Intensity, reality mode, manual triggers, incident catalogue, presets | Delete file |
| P5-10 | Developer inspector | P5-01 | components/admin/DeveloperInspector.tsx (new) | State, decisions, economics, spatial, events, performance | Delete file |
| P5-11 | Simulation experiments | P5-01 | components/admin/Experiments.tsx (new) | Snapshot, branch, adjust, advance, compare, adopt/discard | Delete file |
| P5-12 | Administrative safety | P5-01 | simulation/adminCommands.ts (new) | Validated commands, preview/confirm, atomic application, audit history | Delete file |
| P5-13 | Configuration management | P5-01 | simulation/configManager.ts (new) | Current settings, defaults, presets, import/export, schema validation | Delete file |
| P5-14 | Admin acceptance tests | P5-01 | test files | 15 acceptance tests pass | N/A |

### Phase 6: World Forking
| ID | Task | Dependencies | Files | Acceptance Test | Rollback |
|---|---|---|---|---|---|
| P6-01 | Deep copy state domains | P5-14 | simulation/fork.ts (new) | Deep copy all state, no shared references | Delete file |
| P6-02 | Fork creation | P6-01 | simulation/fork.ts (update) | New save slot, RNG seed derivation, event history copy | Revert |
| P6-03 | Fork management UI | P6-02 | components/admin/ForkManager.tsx (new) | List, rename, delete, compare | Delete file |
| P6-04 | Comparison view | P6-03 | components/admin/ForkComparison.tsx (new) | Side-by-side metrics, divergence chart | Delete file |
| P6-05 | Merge | P6-04 | simulation/fork.ts (update) | Adopt fork state with backup | Revert |
| P6-06 | Fork validation | P6-05 | test files | Independent save, divergent states, deterministic replay, save/load, merge backup | N/A |

### Phase 7: Visual Identity
| ID | Task | Dependencies | Files | Acceptance Test | Rollback |
|---|---|---|---|---|---|
| P7-01 | Visual specification | P3-13 | docs/visual-spec.md (new) | Color palette, sprite sizes, perspective angles, lighting direction | Delete file |
| P7-02 | Terrain tile assets | P7-01 | public/assets/terrain/ (new) | Grass, dirt, stone, paved, farmland, water, shorelines, rocks | Delete assets |
| P7-03 | Road asset set | P7-01 | public/assets/roads/ (new) | Straight, corner, T-junction, cross, bridge | Delete assets |
| P7-04 | Building sprite set | P7-01 | public/assets/buildings/ (new) | 8 residential variants, farm, bakery, market, town hall with operational/inactive/damaged states | Delete assets |
| P7-05 | Citizen sprite system | P7-01 | public/assets/citizens/ (new) | Modular: clothing, hair, body, accessories, 24 unique combinations | Delete assets |
| P7-06 | Citizen animation set | P7-05 | public/assets/citizens/ (update) | Idle, walk 4-dir, work, eat, rest, interact, danger reaction | Revert |
| P7-07 | Depth sorting | P7-04 | components/IsoMap.tsx (update) | Correct depth sorting by entity anchor position | Revert |
| P7-08 | Camera pan/zoom | P7-07 | components/IsoMap.tsx (update) | Pixel snapping at low zoom, smooth zoom (lerp) | Revert |
| P7-09 | Day/night lighting | P7-07 | components/IsoMap.tsx (update) | Warm morning, clear day, long evening shadows, dark night, building/street lights | Revert |
| P7-10 | Weather visual effects | P7-09 | components/IsoMap.tsx (update) | Rain particles, snow particles, fog, thunderstorm with lightning | Revert |
| P7-11 | Chaos visual effects | P7-09, P4-01 | components/IsoMap.tsx (update) | Equipment breakdown stopped machinery, road closure obstructions, Bread Rain falling bread | Revert |
| P7-12 | Visual style presets | P7-07 | components/GraphicsSettings.tsx (new) | Authentic PS1, Enhanced PS1, Early PS2 | Delete file |
| P7-13 | Graphics configuration | P7-12 | components/GraphicsSettings.tsx (update) | Resolution, scaling, filtering, dithering, color quantization, lighting, shadows, particles, weather intensity, brightness, animation quality, CRT effect, reduced motion, performance stats | Revert |
| P7-14 | Visual debugging overlays | P7-07 | components/DebugOverlays.tsx (new) | Tile coords, collision, paths, footprints, depth anchors, occlusion, ownership, destinations, resources, incidents, performance | Delete file |
| P7-15 | Sprite atlases & batching | P7-04 | components/IsoMap.tsx (update) | Texture atlases, batched rendering, performance improvement | Revert |
| P7-16 | LOD system | P7-15 | components/IsoMap.tsx (update) | LOD for foliage, buildings, citizens | Revert |
| P7-17 | Comparison screenshots | P7-16 | test artifacts | Screenshots for daytime, nighttime, busy activity, damaged infrastructure, surreal chaos | N/A |
| P7-18 | Visual validation | P7-17 | test files | Tile alignment, depth sorting, occlusion, sprite scale/perspective, persistent appearance, animated activity, road closures, Bread Rain visuals, day/night/weather, settings, no regressions, no unsupported deps, acceptable performance | N/A |

### Phase 8: Foliage, Season, Weather, Wildlife
| ID | Task | Dependencies | Files | Acceptance Test | Rollback |
|---|---|---|---|---|---|
| P8-01 | Foliage types | P3-13 | types.ts, constants.tsx (update) | FoliageType enum, BUILDINGS record with foliage entries | Revert |
| P8-02 | Season/Weather enums | P3-13 | types.ts (update) | Season, Weather, WildlifeType enums | Revert |
| P8-03 | Foliage placement logic | P8-01 | App.tsx (update) | Happiness + pollution integration | Revert |
| P8-04 | Decoration placement logic | P8-01 | App.tsx (update) | Happiness integration | Revert |
| P8-05 | Season calculation | P8-02 | App.tsx (update) | Season in game loop, season-based bonuses | Revert |
| P8-06 | Weather roll | P8-02 | App.tsx (update) | Weather effects on happiness, water, pollution | Revert |
| P8-07 | Foliage rendering | P8-01 | components/IsoMap.tsx (update) | Instanced rendering, season-based colors | Revert |
| P8-08 | Decoration rendering | P8-01 | components/IsoMap.tsx (update) | Instanced rendering | Revert |
| P8-09 | Grass auto-generation | P8-07 | components/IsoMap.tsx (update) | 4 grass blades per empty tile, random rotation | Revert |
| P8-10 | Leaf particles | P8-07 | components/IsoMap.tsx (update) | Instanced small quads falling in autumn | Revert |
| P8-11 | Weather particles | P8-06 | components/IsoMap.tsx (update) | Rain, snow, fog, thunderstorm particles | Revert |
| P8-12 | Season/weather indicators | P8-05, P8-06 | components/UIOverlay.tsx (update) | Season/weather icons in top bar | Revert |
| P8-13 | Wildlife enhancement | P8-01 | components/IsoMap.tsx (update) | Seagulls, pigeons, butterflies, squirrels, fish | Revert |
| P8-14 | Foliage validation | P8-13 | test files | Foliage placement, season changes, weather effects | N/A |

### Phase 9: UI Refactor
| ID | Task | Dependencies | Files | Acceptance Test | Rollback |
|---|---|---|---|---|---|
| P9-01 | UIOverlay refactor | P3-13 | components/UIOverlay.tsx (update) | Remove AI provider badge, rate limit display, scanline overlay; add happiness/crime/pollution/reputation displays, undo/redo, mute, stats panel, achievement panel, blueprint buttons, minimap toggle, locked tooltips, happiness trend, disaster banner, achievement popup | Revert |
| P9-02 | StartScreen refactor | P3-13 | components/StartScreen.tsx (update) | Solid surface, no gradient text, relabel toggle, remove glow divs, remove window.aistudio | Revert |
| P9-03 | IdeaLab refactor | P1-03 | IdeaLab.tsx (update) | Remove react-markdown, local generation, prompt textarea; "Add Random Idea" button; static detail cards | Revert |
| P9-04 | MusicLab refactor | P1-03 | components/MusicLab.tsx (update) | Remove Lyria mode, keep procedural, rename header | Revert |
| P9-05 | App.tsx refactor | P3-13 | App.tsx (update) | Sync calls, reputation state, undo stack, stats tracking, keyboard shortcuts, sound triggers, achievement checks, maintenance costs, connectivity, pollution, happiness, crime, disaster checks, reputation awards, statistics snapshots | Revert |
| P9-06 | UI validation | P9-05 | test files | All UI components render correctly, no cloud imports | N/A |

### Phase 10: Polish & Validation
| ID | Task | Dependencies | Files | Acceptance Test | Rollback |
|---|---|---|---|---|---|
| P10-01 | Update .env.example/metadata.json | P1-01 | .env.example, metadata.json | No API key references | Revert |
| P10-02 | Update package.json | P1-01 | package.json | Remove @google/genai, openai, react-markdown, @types/express | Revert |
| P10-03 | Final validation | P9-06, P7-18, P8-14, P6-06, P5-14, P4-07, P3-13 | All | npm install succeeds, npm run lint passes, npm run build succeeds, app runs without env vars, no cloud imports, all gameplay systems functional, reputation persists, old saves load, sound effects play, undo/redo works, statistics panel works, day/night cycle works, upgrades work, achievements unlock, minimap works, maintenance costs deducted, tutorial shows, keyboard shortcuts work, foliage/season/weather/wildlife work, admin console works, world forking works, visual identity works | N/A |

---

## D. Requirement-to-Task-to-Test Matrix

### Invariants (INV-01 through INV-14)

| Requirement | Task IDs | Test | Expected Result |
|---|---|---|---|
| INV-01 Currency Conservation | P3-01, P3-02 | Sum all account balances after 1000 ticks | Equals initial total ($8000) |
| INV-02 Nonnegative Finances | P3-01 | Attempt purchase with insufficient funds | Transaction rejected, no state change |
| INV-03 Inventory Conservation | P3-01 | Track grain/bread stock through 100 ticks | current = initial + production + imports − consumption − exports − destruction |
| INV-04 Unique Entity Ownership | P3-02 | Assign citizen to two households | Second assignment rejected |
| INV-05 Spatial Validity | P3-02 | Move citizen to invalid tile | Movement rejected |
| INV-06 Atomic Transactions | P3-01 | Interrupt purchase mid-transaction | No partial state change |
| INV-07 Causal Event Integrity | P4-01 | Trigger disaster, check event log | Event has ID, tick, actor, affected entities, preconditions, state changes, outcome |
| INV-08 Valid State Transitions | P3-02 | Invalid state transition (e.g., EXECUTING → IDLE) | Transition rejected |
| INV-09 Deterministic Replay | P2-01 | Run 1000 ticks twice with same seed | State hashes identical |
| INV-10 Persistence Integrity | P2-07 | Save and reload after 500 ticks | All state preserved (citizens, households, money, inventories, employment, goals, actions, treasury, clock, RNG, events, chaos config) |
| INV-11 Need Bounds | P3-02 | Run 1000 ticks, check needs | 0 ≤ hunger ≤ 100, 0 ≤ fatigue ≤ 100 |
| INV-12 No Omniscient NPCs | P3-04 | Inspect citizen decision inputs | Only own needs, location, employment, money, observed prices, learned road network |
| INV-13 Deterministic Conflict Resolution | P3-02 | Two citizens apply for same job | Only one succeeds, other rejected |
| INV-14 Invariant Safety Under Chaos | P4-01 | Trigger Bread Rain, check invariants | All invariants hold, anomalous bread recorded with provenance |

### Reference Run (43,200 ticks)

| Metric | Expected | Test |
|---|---|---|
| Ticks | 43,200 (30 days) | Run headlessly, no real-time waits |
| Citizens | 24 (initial), may change via birth/death | Population tracked per day |
| Households | 8 (initial) | Household count tracked |
| Businesses | 3 (farm, bakery, market) | Business count tracked |
| Total currency | $8000 (conserved) | Sum all account balances = $8000 |
| Grain produced | > 0 | Farm inventory tracked |
| Bread produced | > 0 | Bakery inventory tracked |
| Bread consumed | > 0 | Household consumption tracked |
| Taxes collected | > 0 | Government treasury tracked |
| Wages paid | > 0 | Worker balances tracked |
| Happiness | 0-100 | Happiness tracked per day |
| Crime | ≥ 0 | Crime tracked per day |
| Pollution | ≥ 0 | Pollution tracked per day |

### Competition/Insufficient-Funds Tests

| Test | Scenario | Expected Result |
|---|---|---|
| Job competition | Two citizens apply for same job | Only one hired, other rejected, no ghost assignment |
| Insufficient funds | Citizen buys bread with $0 | Purchase rejected, no money deducted, no bread added |
| Business insolvency | Bakery can't afford grain | Purchase rejected, bakery continues with existing stock |
| Government insolvency | Treasury can't afford road maintenance | Maintenance deferred, road condition degrades |
| Building placement on occupied tile | Place building on existing building | Placement rejected, no state change |
| Disaster on edge of grid | Fire at tile (0,0) with radius 3 | Coordinates clamped, no out-of-bounds access |

### Incident Save/Load Tests

| Test | Scenario | Expected Result |
|---|---|---|
| Save during Equipment Breakdown | Save while farm production stopped | Reload: farm still broken, repair timer preserved |
| Save during Road Closure | Save while road closed | Reload: road still closed, detour routes preserved |
| Save during Bread Rain (Surreal) | Save while bread falling | Reload: bread rain continues, collected bread preserved |
| Save with pending actions | Save while citizen traveling | Reload: citizen continues traveling, destination preserved |
| Save with reservations | Save while purchase reserved | Reload: reservation preserved, completes or expires |

### Replay Equivalence Tests

| Test | Scenario | Expected Result |
|---|---|---|
| Identical seeds | Run 1000 ticks twice with same seed | State hashes identical |
| Different seeds | Run 1000 ticks with different seeds | State hashes different |
| With external inputs | Run 1000 ticks with recorded manual commands | State hashes identical on replay |
| With incidents | Run 1000 ticks with incidents enabled | State hashes identical on replay |

### Invalid Import Tests

| Test | Scenario | Expected Result |
|---|---|---|
| Oversized blueprint | Import 10MB blueprint | Rejected, size limit exceeded |
| Invalid entity count | Import save with 100,000 citizens | Rejected, entity count exceeded |
| Invalid dimensions | Import save with 500×500 grid | Rejected, dimensions exceeded |
| Deep nesting | Import save with 50 levels of nesting | Rejected, nesting depth exceeded |
| Prototype pollution | Import save with `__proto__` keys | Rejected, unsafe field detected |
| Version mismatch | Import v0 save into v1 engine | Migrated successfully, no data loss |
| Corrupted JSON | Import invalid JSON | Rejected, parse error reported |
| Missing fields | Import save without `version` field | Migrated from v0, defaults applied |

### Role Enforcement Tests

| Test | Scenario | Expected Result |
|---|---|---|
| Player triggers incident | Player (no admin) clicks "Trigger Disaster" | Command rejected, audit log entry |
| Admin triggers incident | Admin clicks "Trigger Disaster" | Command accepted, incident triggered |
| Developer inspects state | Developer opens State Inspector | Full state visible |
| Player opens admin console | Player (no admin) opens admin console | Console not accessible |

### Controlled Fork Tests

| Test | Scenario | Expected Result |
|---|---|---|
| Fork creates independent save | Fork at tick 100, advance both by 50 ticks with different policies | States diverge |
| No-intervention equality | Fork at tick 100, advance both by 50 ticks with no changes | State hashes identical |
| Deterministic replay | Fork twice with same inputs | Fork states identical |
| Save/load preserves lineage | Save and reload fork | Fork lineage and divergence data preserved |
| Merge creates backup | Merge fork into main | Main save backed up before overwrite |

### UI Behavior Tests

| Test | Scenario | Expected Result |
|---|---|---|
| Pause | Click pause button | Simulation stops, tick counter frozen |
| Single-step | Click single-step | Exactly one tick executes |
| Speed change | Change speed to 4x | Tick interval becomes 500ms |
| Focus loss | Switch browser tab | Simulation auto-pauses (if configured) |
| Admin console | Open admin console | Tabbed interface with 10 sections |
| Citizen inspector | Click citizen on map | Inspector shows actual state (location, household, employment, income, savings, needs, action, goal) |
| Chaos lab | Set intensity to 0 | No incidents triggered |
| Surreal lockout | Set reality mode to REALISTIC, intensity 100 | No surreal incidents triggered |
| Manual injection | Manually trigger Bread Rain in REALISTIC mode | Rejected (surreal only) |

### Visual Truthfulness Tests

| Test | Scenario | Expected Result |
|---|---|---|
| Day/night | Advance to night (tick 1200) | Lighting changes to dark blue, building lights on |
| Weather | Trigger rain | Rain particles visible, happiness -1, water production +20% |
| Disaster damage | Trigger fire on building | Building shows visual damage, produces 50% output |
| Road closure | Close road | Physical obstruction visible, pathfinding changes |
| Bread Rain | Trigger in Surreal mode | Pixel-art bread falls from sky, accumulates at valid locations |
| Depth sorting | Citizen walks behind building | Citizen correctly occluded |
| Season change | Advance to autumn (tick 181) | Deciduous trees turn yellow/orange, leaf particles |
| Citizen appearance | Save and reload | Citizen appearance identical |

### Numerical Performance Budgets

| Metric | Budget | Condition |
|---|---|---|
| Simulation tick (24 citizens, 3 buildings) | < 5ms | 20×20 grid, no chaos |
| Simulation tick (100 citizens, 20 buildings) | < 20ms | 60×60 grid, no chaos |
| Render frame (24 citizens, 3 buildings) | < 16ms (60fps) | 20×20 grid, Enhanced PS1 preset |
| Render frame (100 citizens, 20 buildings) | < 33ms (30fps) | 60×60 grid, Enhanced PS1 preset |
| Save (24 citizens) | < 50ms | localStorage |
| Load (24 citizens) | < 100ms | localStorage |
| Memory (24 citizens) | < 50MB | Chrome, no dev tools |
| Memory (100 citizens) | < 200MB | Chrome, no dev tools |
| Save file size (24 citizens) | < 100KB | Uncompressed JSON |
| Save file size (100 citizens) | < 500KB | Uncompressed JSON |

---

## E. Remaining Decisions, Source-Access Gaps, and Deferred Scope

### Remaining Decisions (require user input)

1. **Rendering architecture:** Hybrid orthographic 3D with sprite-based assets (recommended) vs pure 2D sprite engine vs pure 3D. The existing codebase is R3F/Three.js. Hybrid preserves existing rendering while achieving PS1 look. **Recommendation: Hybrid.**

2. **State management library:** React context (built-in, no deps) vs Zustand (lightweight, popular) vs Redux (heavyweight). **Recommendation: Zustand for simulation state (decoupled from React), React context for UI state.**

3. **Save storage:** localStorage (simple, 5-10MB limit) vs IndexedDB (larger, async) vs both (localStorage for settings, IndexedDB for saves). **Recommendation: Both — localStorage for settings/config, IndexedDB for save data.**

4. **Testing framework:** Vitest (fast, Vite-native) vs Jest (mature, slower). **Recommendation: Vitest.**

5. **Spatial indexing:** Spatial hash (simple, O(1) average) vs quadtree (better for sparse data). **Recommendation: Spatial hash for 20×20 grid (simple, sufficient). Re-evaluate if grid exceeds 100×100.**

6. **ECS vs OOP:** Entity-component-system (data-oriented, cache-friendly) vs object-oriented (simpler, familiar). **Recommendation: OOP for v1 (simpler, sufficient for 24 citizens). Re-evaluate if citizen count exceeds 1000.**

### Source-Access Gaps

- **No direct repository access:** The plan was written based on web-fetched file contents, not direct filesystem inspection. Some assumptions about code structure may be inaccurate.
- **No runtime testing:** No tests have been run. Performance budgets are estimates, not measurements.
- **No git history analysis:** Could not verify whether API keys were previously committed.
- **No dependency audit:** Could not verify exact versions of installed packages.

### Deliberately Deferred Scope

| Feature | Reason for Deferral |
|---|---|
| Full political simulation | Too complex for v1. Design extension points. |
| Complex relationships/marriage | Not essential for economic loop. |
| Generational inheritance | Requires lifecycle system first. |
| Advanced healthcare | No health system in v1. |
| Large-scale military conflict | Not in scope. |
| Sophisticated crime systems | Simplified crime model (no police buildings). |
| Global trade | Single settlement only. |
| Dozens of professions | 3 jobs in v1 (farmer, baker, market worker). |
| Hundreds of resource types | 2 goods in v1 (grain, bread). |
| Complex technology progression | Simple unlock tree in v1. |
| Hundreds of supernatural event types | 3 incidents in v1 (equipment breakdown, road closure, bread rain). |
| Multiple settlements | Single settlement for v1. Design state to support later. |
| Multiplayer | Offline single-player only. |
| Modding/extensibility | Data-driven content in v3+. |
| Localization/i18n | English only for v1. |
| Keyboard shortcut customization | Hardcoded shortcuts for v1. |
| Grid size > 60×60 | 60×60 max for v1. |
| Citizen count > 300 | 300 max for v1. |
| ECS architecture | OOP for v1. Re-evaluate at 1000+ citizens. |
| Web Worker simulation | Main thread for v1. Re-evaluate if tick > 20ms. |
| Diplomacy/relations | Abstract settlements in v3+. |
| Citizen interactions | Social interactions in v2. |
| Reputation/fame system | Deferred to v3+. |
| Modding/extensibility | Data-driven content in v3+. |

### Final Note

This plan is NOT implementation-ready until the 5 remaining decisions (rendering architecture, state management library, save storage, testing framework, spatial indexing) are resolved. All other contradictions have been addressed with chosen replacements. The plan should be reviewed and the remaining decisions resolved before implementation begins.