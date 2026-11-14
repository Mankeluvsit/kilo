# LIVING WORLD — Master Implementation Plan (Revised)

## A. Readiness Verdict & Contradiction Register

**Verdict: NOT IMPLEMENTATION-READY** — 10 blocking contradictions resolved below, 8 missing contracts filled in. 5 genuine decisions remain (see §E).

### Contradiction Register (resolved)

| # | Old Clause | Chosen Replacement | Rationale |
|---|---|---|---|
| 1 | §2 "Math.random — no seed plumbing" | **Seeded persisted RNG** (mulberry32, separate streams per domain). Cosmetic randomness (particles, UI) may use Math.random but must not affect simulation state. | Deterministic replay (INV-09) requires reproducible randomness. |
| 2 | §§7,12,15 expand React game loop first; §23.8 defers engine extraction | **Pure authoritative simulation engine first** (pure TS, no React). UI/rendering consume read-only views and submit validated commands. | Commands required for invariants. Cannot add systems before engine boundary exists. |
| 3 | §5c passive income, §10 goal rewards, §11.7.H treasury-only upkeep, §20.5 sandbox issuance | **Single conserved-accounting model.** Every money flow has defined source and destination account. Passive income debits "city economy" account. Goal rewards debit "reward pool" account. Upkeep debits building owner, credits maintenance fund. Sandbox issuance uses explicit issuance account with audit trail. | INV-01/INV-02 require equal debit/credit on every flow. |
| 4 | §8-9 "defaulting corrupt saves to new world, silently failing on quota" | **Versioned migration with validated staged writes.** Save to temp key → validate → swap. On corruption: retain original, attempt migration, report failure. On quota: report error, offer backup export. Never silently lose data. | INV-10 requires explicit failure reporting. |
| 5 | §5d flood-fill "seeds every road adjacent to any building" | **Reachability-based connectivity.** A building is connected if there exists a path along roads from that building to a defined destination (town hall, market, or any road tile on map boundary). Disconnected road islands do not qualify. Test: disconnected island, complete isolation, alternate routes. | Current implementation marks all roads adjacent to any building as "connected" regardless of actual connectivity. |
| 6 | §11.4 pollution-only disasters with mandatory demolition | **Replace with §19.8-14 causal incident model.** Equipment Breakdown (production stop, recover via funded repair), Road Closure (route block, recover via repair/reopening), Bread Rain (Surreal only, anomalous bread creation). No mandatory demolition. Eligibility based on exposure, intensity, cooldowns. | Pollution-only disasters conflict with exposure-based incidents and funded repair. |
| 7 | §11.7.B grid-only undo (snapshot stack) | **Validated inverses only when safe; otherwise checkpoint branching or compensation.** Grid undo restores building placement. Money/inventory/employment changes require checkpoint branching (World Forking) or compensating transactions. Never claim arbitrary undo. | Grid-only undo cannot restore money, inventories, occupants, jobs, reservations, RNG, or scheduled events. |
| 8 | §1/§17 3D meshes vs §22 modular sprite spec | **Hybrid: orthographic 3D rendering with sprite-based assets.** Keep R3F/Three.js. Buildings/citizens as textured quads (billboards) with depth sorting by anchor. Terrain as 3D mesh with pixel-art textures. Post-processing for retro aesthetic. | Existing codebase is R3F. Migration to 2D sprite engine would lose instanced rendering, LOD, post-processing. |
| 9 | §21.3 reseed forks (new seed from parent + fork index) | **Controlled comparison mode:** preserve initial RNG states for identical no-intervention branches. Offer reseeded exploration as separate "experimental" mode, labeled distinctly. "Merge" = adopt-as-replacement with backup, not a true merge. | Reseeding makes policy comparisons differ in both intervention and randomness. |
| 10 | §§12,18.10,20.18,21.5,22.18,23.8 competing roadmaps | **Single gated backlog** (see §C). Admin mutation UI follows its safety task. Foliage/progression deferred until first economy passes. One dependency-ordered task graph. | Competing roadmaps cause admin mutation UI to precede safety task. |

### Security Completeness

- **Client-bundled API keys:** Removing providers prevents future inclusion but does not establish whether keys were previously exposed. Owner must check git history, built assets, source maps, runtime network activity. If exposed, revoke/rotate without displaying secrets.
- **Trust model:** In offline browser-owned world, roles enforce application workflows but cannot protect against device owner. Local append-only history is useful provenance, not tamper-proof security.
- **Blueprint/save/config imports:** Limits on compressed/decompressed size (max 5MB), entity count (max 10,000), dimensions (max 200×200), nesting depth (max 10). Schema/version checks. Safe field handling. Reference validation. Atomic preview/apply. Reject invalid input without overwriting current world.

---

## B. Consolidated Specification

### B.1 Clock & Time

- **Timestep:** 1 simulated minute = 1 tick. 1440 ticks/day. 30 days = 43,200 ticks.
- **Tick phases (deterministic order):** (1) needs decay, (2) movement, (3) work/production, (4) shopping/purchases, (5) wage payments, (6) tax collection, (7) business decisions, (8) chaos events, (9) maintenance/recovery, (10) statistics snapshot, (11) save.
- **Pause:** stops tick advancement. **Step:** executes exactly one tick. **Catch-up:** on tab focus, execute at most 10 ticks then resume normal rate.
- **Testing:** 43,200 ticks headlessly. No real-time waits. Each tick is synchronous.
- **Needs rates:** hunger +0.5/tick, fatigue +0.3/tick (working), -1.0/tick (resting at home), -0.5/tick (idle at home).
- **Work/production periods:** work from tick 480 (08:00) to tick 1080 (18:00). Production completes at end of work period.

### B.2 Economy Bootstrapping (corrected)

**Initial balances (reference settlement):**
- 8 households: $200 each = $1,600
- Farm business: $1,000 cash, 0 grain stock
- Bakery business: $800 cash, 0 grain, 0 bread
- Market business: $600 cash, 0 bread
- Government treasury: $0
- **Total initial currency: $1,600 + $1,000 + $800 + $600 = $4,000**

**Prices:** Grain $2/unit, Bread $5/unit (wholesale), $8/unit (retail). Farm sells grain to bakery at $2. Bakery sells bread to market at $5. Market sells bread to households at $8.

**Recipes:** Bakery: 2 grain → 1 bread, 10 ticks processing time, 1 worker.

**Wages:** Farm worker $2/tick, Bakery worker $3/tick, Market worker $2/tick. Paid at end of work period (tick 1080). Daily payroll: Farm $24 (12 hours × $2), Bakery $36 (12 hours × $3), Market $24 (12 hours × $2). Total daily payroll: $84.

**Tax:** 10% income tax. Tax = floor(grossWage × 0.1). Employer debited gross wage. Worker receives net (90%). Government receives 10%.

**Production rates:**
- Farm: 1 grain per 5 ticks (with 1 worker). Daily output: ~144 grain (1440 ticks / 5 × 1 worker, capped by work period: 600 ticks / 5 = 120 grain/day).
- Bakery: 1 bread per 10 ticks (2 grain per bread). Daily output: 60 bread/day (600 ticks / 10). Consumes 120 grain/day.
- Market: sells bread to households. No production.

**Food consumption:** Each household consumes 1 bread/day (1 citizen per household = 3 citizens per household? No — 24 citizens / 8 households = 3 citizens per household). Each citizen consumes 0.5 bread/day. Total daily consumption: 24 × 0.5 = 12 bread/day.

**Starting stock:** Farm: 0 grain. Bakery: 0 grain, 0 bread. Market: 0 bread. Households: 0 bread.

**Liquidity bootstrapping:** Farm starts with $1,000 cash to pay first worker. First grain sale to bakery at tick 5 (first grain produced) provides revenue. Bakery starts with $800 to buy grain and pay worker. Market starts with $600 to buy bread.

**Daily revenue/cost analysis (steady state):**
- Farm: produces 120 grain/day. Sells at $2 = $240 revenue. Worker cost = $24. Net = $216/day.
- Bakery: produces 60 bread/day. Buys 120 grain at $2 = $240 cost. Sells 60 bread at $5 = $300 revenue. Worker cost = $36. Net = $24/day.
- Market: sells 12 bread/day to households at $8 = $96 revenue. Buys 12 bread at $5 = $60 cost. Worker cost = $24. Net = $12/day.
- Households: each earns wages. Farm worker earns $24/day (net $21.60). Bakery worker earns $36/day (net $32.40). Market worker earns $24/day (net $21.60). Total household income from 3 employed citizens = $75.60/day. Bread cost = 12 × $8 = $96/day. **Shortfall: households can't afford bread.**

**Resolution:** Adjust prices. Market sells bread at $5 (not $8). Household bread cost = 12 × $5 = $60/day. Household income = $75.60/day. Surplus = $15.60/day. Sustainable.

**Revised prices:** Grain $2/unit (farm→bakery), Bread $5/unit (bakery→market), Bread $5/unit (market→household). Market margin = $0 (break-even on bread, earns from volume). Actually, market needs income. Let market sell at $6. Household bread cost = 12 × $6 = $72/day. Surplus = $3.60/day. Tight but sustainable.

**Final prices:** Grain $2, Bread (wholesale) $4, Bread (retail) $6. Farm revenue = 120 × $2 = $240. Farm cost = $24 (wage) + $0 (no inputs). Farm net = $216. Bakery revenue = 60 × $4 = $240. Bakery cost = 120 × $2 (grain) + $36 (wage) = $276. **Bakery loses $36/day.** Not sustainable.

**Resolution:** Increase bread wholesale price to $5. Bakery revenue = 60 × $5 = $300. Bakery cost = $240 + $36 = $276. Net = $24/day. Sustainable. Market revenue = 12 × $6 = $72. Market cost = 12 × $5 + $24 = $84. **Market loses $12/day.**

**Resolution:** Increase retail price to $7. Market revenue = 12 × $7 = $84. Market cost = $60 + $24 = $84. Break-even. Households: income $75.60, bread cost $84. **Shortfall again.**

**Resolution:** Increase wages. Farm worker $3/tick ($36/day), Bakery worker $4/tick ($48/day), Market worker $3/tick ($36/day). Total household income = $36 + $48 + $36 = $120/day (net $108). Bread cost = 12 × $7 = $84. Surplus = $24/day. Sustainable.

**Final numbers:**
- Wages: Farm $3/tick, Bakery $4/tick, Market $3/tick
- Prices: Grain $2, Bread wholesale $5, Bread retail $7
- Farm: revenue $240, cost $36, net $204/day
- Bakery: revenue $300, cost $240+$48=$288, net $12/day
- Market: revenue $84, cost $60+$36=$96, **loses $12/day**

**Problem:** Market still loses money. Reduce market worker to $2/tick ($24/day). Market cost = $60 + $24 = $84. Break-even. Households: $36+$48+$24 = $108 (net $97.20). Bread cost = $84. Surplus = $13.20/day. Sustainable.

**Final final numbers:**
- Wages: Farm $3/tick, Bakery $4/tick, Market $2/tick
- Prices: Grain $2, Bread wholesale $5, Bread retail $7
- Farm: revenue $240, cost $36, net $204/day
- Bakery: revenue $300, cost $240+$48=$288, net $12/day
- Market: revenue $84, cost $60+$24=$84, net $0/day (break-even)
- Households: income $108 (net $97.20), bread cost $84, surplus $13.20/day

**Worked first-day ledger:**
- Tick 0: Business decisions. Farm hires 1 worker (cash $1000 → $997? No — hiring doesn't cost money in v1, just assigns). Actually, hiring is free (no recruitment cost). Worker assigned.
- Tick 0-480 (night): No work. Needs decay. Citizens rest at home.
- Tick 480 (08:00): Work begins. Farm worker starts producing grain. Bakery worker starts (but no grain yet, waits). Market worker starts (but no bread yet, waits).
- Tick 485: Farm produces first grain (5 ticks). Grain stock = 1.
- Tick 490: Farm produces second grain. Grain stock = 2. Bakery begins production (2 grain → 1 bread, 10 ticks).
- Tick 500: Farm produces 3rd grain. Grain stock = 3 (1 used by bakery, 2 remaining). Bakery continues processing.
- Tick 500-1080: Farm continues producing grain. Bakery completes bread at tick 490+10=500. Bread stock = 1. Market buys bread (5 ticks later? No — instant transfer). Market bread stock = 1.
- Tick 1080 (18:00): Work ends. Wages paid. Farm worker: gross $36, tax $3.60→$3 (floor), net $33. Farm cash: $1000 - $36 = $964. Worker cash: +$33. Government treasury: +$3. Bakery worker: gross $48, tax $4.80→$4, net $44. Bakery cash: $800 - $48 - $2 (grain purchased) = $750. Worker cash: +$44. Government treasury: +$4 (total $7). Market worker: gross $24, tax $2.40→$2, net $22. Market cash: $600 - $24 = $576. Worker cash: +$22. Government treasury: +$2 (total $9).
- Tick 1080-1439 (evening): Citizens travel home. Some citizens (workers) have money to buy bread. Market sells bread. Household with employed citizen buys bread ($6). Market cash: $576 + $6 = $582. Household money: -$6. Market bread stock: -1.
- Tick 1439 (end of day): Statistics snapshot. Population: 24. Money: $4,000 (conserved). Happiness: 50 (initial). Save.

**30-day validation design:**
- Run 43,200 ticks headlessly.
- Verify: total currency = $4,000 at every tick (INV-01).
- Verify: no account goes negative (INV-02).
- Verify: grain stock = 0 + production - consumption (INV-03).
- Verify: bread stock = 0 + production - consumption (INV-03).
- Verify: at least 1 bread purchased and consumed by a household.
- Verify: at least 1 wage paid and tax collected.
- Verify: government treasury > 0 after day 1.
- Verify: all 3 businesses still operating after 30 days.
- Verify: no invariant violations logged.
- Verify: deterministic replay (same seed → same state hash after 43,200 ticks).
- Verify: save/load preserves all state at tick 21,600 (mid-run).
- Report: PASS/FAIL for each criterion. No claim of PASS without actual execution.

### B.3 Physical Logistics (restored)

**Goods movement:** Deterministic delivery with travel duration. When a buyer initiates purchase, goods are not instantly transferred. Instead:

1. **Order creation:** Buyer creates a delivery order (buyer, seller, good, quantity, price). Order status = PENDING.
2. **Route calculation:** System calculates shortest path along roads from seller to buyer. If no path, order fails (BLOCKED).
3. **Travel:** Goods travel along the route at 1 tile/tick. Order status = IN_TRANSIT. Seller's stock decremented, buyer's reserved stock incremented.
4. **Delivery:** When goods arrive, buyer's inventory incremented, reserved stock decremented. Order status = DELIVERED. Payment processed (buyer debited, seller credited).
5. **Cancellation:** Order can be cancelled before delivery. Goods return to seller, reservation released.
6. **Blocked route:** If road closure blocks the route mid-transit, goods are held at last reachable tile. Order status = BLOCKED. When route reopens, transit resumes.

**Ownership transfer:** Ownership transfers at delivery (not at purchase initiation). Seller retains ownership during transit.

**Reservations:** Seller reserves stock when order created (decremented from available, added to reserved). On delivery, reserved → buyer. On cancellation/failure, reserved → available.

**Capacities:** No storage capacity limits in v1. Unlimited inventory.

**Delivery delays:** 1 tile/tick travel time. A purchase across a 10-tile road takes 10 ticks to deliver.

**Blocked routes:** Road closure blocks transit. Goods held at last reachable tile. When road reopens, transit resumes from held position.

### B.4 Citizen Planning (GOAP) — unchanged from previous

Action catalogue, bounded search budget, knowledge boundaries, utility ties, reservation expiry, failure/replanning rules, memory. (See previous version.)

### B.5 Time & Scheduling (detailed) — unchanged from previous

Canonical clock, needs rates, work/production periods, wage/tax timing, business decisions, chaos events, maintenance, statistics snapshot, save, pause, step, catch-up. (See previous version.)

### B.6 State Ownership & Replay — unchanged from previous

Canonical owners, command ordering, RNG algorithm, state hashing, event retention, replay compatibility. (See previous version.)

### B.7 Gameplay Formulas (resolved) — unchanged from previous

Happiness, crime, pollution, population growth, reputation vs research, repair vs demolition, tax units. (See previous version.)

### B.8 Migration & Regressions (lossless or blocked)

**Legacy save mapping:**
- Old saves (v0, no version field) → migrate to v1: add `happiness: 50`, `pollution: 0` to tiles, add `reputation: 0`, add `version: 1`.
- Old buildings without `pollution`/`upkeep` → default to 0.
- Old citizens without `needs`/`skills`/`memories` → default to initial values.
- Old goals without `rewardRep` → default to 1.

**Intentional removals (not regressions):** Cloud AI providers (Gemini, OpenAI), Lyria audio mode, react-markdown, AI key checks. These are explicitly removed, not broken.

**Unmatched entities:** If a legacy save contains entities not in the new schema, do NOT skip. Instead: retain original save data, report incompatibility to user, require deliberate choice (migrate with defaults or abort). Never silently lose data.

**Preservation:** Always retain original save data before migration. If migration fails, restore original. Atomic: write to temp key → validate → swap. On any failure, original is untouched.

**Lossy conversion:** If migration cannot preserve all data (e.g., old format has fields with no new equivalent), report which fields will be lost and require user confirmation before proceeding.

### B.9 Security Boundary — unchanged from previous

Local owner-controlled roles, permission enforcement, client-key exposure assessment, import validation. (See previous version.)

### B.10 Rendering Architecture — unchanged from previous

Hybrid orthographic 3D with sprite-based assets. (See previous version.)

---

## C. Dependency-Ordered Task Graph (corrected)

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
| P2-02 | Implement command pattern | P2-01 | simulation/commands.ts (new) | All state changes go through commands, each has execute/undo (validated inverses only) | Delete file |
| P2-03 | Implement seeded RNG | P2-01 | simulation/rng.ts (new) | Same seed → same sequence, separate streams (mulberry32) | Delete file |
| P2-04 | Implement invariants | P2-01 | simulation/invariants.ts (new) | All 14 invariants checked per tick, violations logged | Delete file |
| P2-05 | Implement spatial indexing | P2-01 | simulation/spatial.ts (new) | O(1) neighbor lookup, 3600-tile grid benchmarks | Delete file |
| P2-06 | Implement clock/scheduling | P2-01 | simulation/engine.ts | Fixed timestep, tick phases in order, pause/step/catch-up | Revert |
| P2-07 | Implement save format v1 | P2-01 | simulation/save.ts (new) | Versioned save, migration from v0 (lossless or blocked), atomic write (temp key → validate → swap), backup restoration | Delete file |
| P2-08 | Implement configuration store | P2-01 | simulation/config.ts (new) | Centralized config, schema validation, defaults | Delete file |
| P2-09 | Implement error boundaries | P2-01 | components/ErrorBoundary.tsx (new) | Per-system error boundaries, graceful degradation | Delete file |
| P2-10 | Implement structured logging | P2-01 | services/logService.ts (new) | Log levels, component tags, optional file output | Delete file |
| P2-11 | Implement automated testing | P2-01 | vitest.config.ts, test files (new) | Vitest runs, 80% coverage for simulation logic | Delete test files |
| P2-12 | Implement performance profiling | P2-01 | components/PerfOverlay.tsx (new) | Dev-mode overlay showing FPS, tick duration, entity counts | Delete file |

### Phase 3: Complete Economic Loop
| ID | Task | Dependencies | Files | Acceptance Test | Rollback |
|---|---|---|---|---|---|
| P3-01 | Implement economy module | P2-01, P2-02, P2-03, P2-06, P2-07 | simulation/economy.ts (new) | Farm produces grain, bakery converts to bread, market sells, currency conserved ($4,000 total) | Delete file |
| P3-02 | Implement citizen module | P2-01, P2-02, P2-03, P2-06 | simulation/citizens.ts (new) | Citizens travel, work, earn wages, pay tax, buy bread (with delivery), eat, rest | Delete file |
| P3-03 | Implement building module | P2-01, P2-02 | simulation/buildings.ts (new) | Building yields, upgrades, damage, repair (funded), demolition (manual only, no auto-demolish) | Delete file |
| P3-04 | Implement GOAP planner | P3-02 | simulation/goap.ts (new) | Citizens select actions via utility + GOAP, bounded search, replanning | Delete file |
| P3-05 | Implement road connectivity | P3-01, P2-05 | simulation/spatial.ts (update) | Reachability-based connectivity, disconnected islands don't qualify, buildings marked connected | Revert |
| P3-06 | Implement pollution system | P3-01 | simulation/pollution.ts (new) | Industry adds pollution to 3-tile radius, happiness penalty, decay | Delete file |
| P3-07 | Implement happiness/crime | P3-01, P3-06 | simulation/happiness.ts (new) | Happiness formula with previous-tick crime, crime formula with previous-tick happiness (no circular dependency) | Delete file |
| P3-08 | Implement population growth | P3-02, P3-07 | simulation/population.ts (new) | Population grows based on happiness, decreases if unhappy. Population changes create/remove valid citizens and households (no fractional aggregate) | Delete file |
| P3-09 | Implement maintenance costs | P3-01, P3-03 | simulation/maintenance.ts (new) | Upkeep deducted per tick, buildings unpowered if can't pay | Delete file |
| P3-10 | Implement tech/unlock tree | P3-01 | simulation/tech.ts (new) | Reputation unlocks buildings, research unlocks recipes | Delete file |
| P3-11 | Implement blueprints | P3-01 | simulation/blueprints.ts (new) | Export/import grid as base64 compressed JSON (with validation limits) | Delete file |
| P3-12 | Implement undo/redo | P2-02 | simulation/undo.ts (new) | Grid placement undo via command pattern (with cost reversal), checkpoint for money/inventory | Delete file |
| P3-13 | Implement logistics/delivery | P3-01, P3-02, P3-05 | simulation/logistics.ts (new) | Deterministic delivery with reservations, travel duration, ownership transfer, blocked-route handling | Delete file |
| P3-14 | Integration test: 30-day run | P3-01 to P3-13 | test files | 43,200 ticks headless, all invariants pass, economy reconciles, worked first-day ledger matches | N/A |

### Phase 4: Chaos Engine
| ID | Task | Dependencies | Files | Acceptance Test | Rollback |
|---|---|---|---|---|---|
| P4-01 | Implement chaos engine | P3-14 | simulation/chaos.ts (new) | Eligibility, incident state, recovery, 3 events | Delete file |
| P4-02 | Equipment Breakdown event | P4-01 | simulation/chaos.ts (update) | Production stops, inventory shortages, recover via funded repair | Revert |
| P4-03 | Road Closure event | P4-01 | simulation/chaos.ts (update) | Route blocked, alternate routes, travel time increases, recover via repair | Revert |
| P4-04 | Bread Rain event (Surreal) | P4-01 | simulation/chaos.ts (update) | Anomalous bread creation, collection rules, market effects, only in Surreal | Revert |
| P4-05 | Chaos config | P4-01 | simulation/chaos.ts (update) | Intensity 0-100, reality mode, presets, validation | Revert |
| P4-06 | Chaos Director | P4-01 | simulation/chaos.ts (update) | Chooses incidents based on stability, obeys realism mode | Revert |
| P4-07 | Invariant stress tests | P4-01 | test files | Repeated purchases, job competition, insufficient funds, shortages, overlapping incidents | N/A |

### Phase 5: Admin Console
| ID | Task | Dependencies | Files | Acceptance Test | Rollback |
|---|---|---|---|---|---|
| P5-01 | Admin control center | P3-14 | components/AdminConsole.tsx (new) | Tabbed interface, 10 sections | Delete file |
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
| P6-05 | Merge (adopt-as-replacement) | P6-04 | simulation/fork.ts (update) | Adopt fork state with backup, stale-parent check | Revert |
| P6-06 | Fork validation | P6-05 | test files | Independent save, divergent states, deterministic replay, save/load, merge backup | N/A |

### Phase 7: Visual Identity
| ID | Task | Dependencies | Files | Acceptance Test | Rollback |
|---|---|---|---|---|---|
| P7-01 | Visual specification | P3-14 | docs/visual-spec.md (new) | Color palette, sprite sizes, perspective angles, lighting direction | Delete file |
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
| P8-01 | Foliage types | P3-14 | types.ts, constants.tsx (update) | FoliageType enum, BUILDINGS record with foliage entries | Revert |
| P8-02 | Season/Weather enums | P3-14 | types.ts (update) | Season, Weather, WildlifeType enums | Revert |
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
| P9-01 | UIOverlay refactor | P3-14 | components/UIOverlay.tsx (update) | Remove AI provider badge, rate limit display, scanline overlay; add happiness/crime/pollution/reputation displays, undo/redo, mute, stats panel, achievement panel, blueprint buttons, minimap toggle, locked tooltips, happiness trend, disaster banner, achievement popup | Revert |
| P9-02 | StartScreen refactor | P3-14 | components/StartScreen.tsx (update) | Solid surface, no gradient text, relabel toggle, remove glow divs, remove window.aistudio | Revert |
| P9-03 | IdeaLab refactor | P1-03 | IdeaLab.tsx (update) | Remove react-markdown, local generation, prompt textarea; "Add Random Idea" button; static detail cards | Revert |
| P9-04 | MusicLab refactor | P1-03 | components/MusicLab.tsx (update) | Remove Lyria mode, keep procedural, rename header | Revert |
| P9-05 | App.tsx refactor | P3-14 | App.tsx (update) | Sync calls, reputation state, undo stack, stats tracking, keyboard shortcuts, sound triggers, achievement checks, maintenance costs, connectivity, pollution, happiness, crime, disaster checks, reputation awards, statistics snapshots | Revert |
| P9-06 | UI validation | P9-05 | test files | All UI components render correctly, no cloud imports | N/A |

### Phase 10: Polish & Validation
| ID | Task | Dependencies | Files | Acceptance Test | Rollback |
|---|---|---|---|---|---|
| P10-01 | Update .env.example/metadata.json | P1-01 | .env.example, metadata.json | No API key references | Revert |
| P10-02 | Update package.json | P1-01 | package.json | Remove @google/genai, openai, react-markdown, @types/express | Revert |
| P10-03 | Final validation | P9-06, P7-18, P8-14, P6-06, P5-14, P4-07, P3-14 | All | npm install succeeds, npm run lint passes, npm run build succeeds, app runs without env vars, no cloud imports, all gameplay systems functional, reputation persists, old saves load, sound effects play, undo/redo works, statistics panel works, day/night cycle works, upgrades work, achievements unlock, minimap works, maintenance costs deducted, tutorial shows, keyboard shortcuts work, foliage/season/weather/wildlife work, admin console works, world forking works, visual identity works | N/A |

---

## D. Requirement-to-Task-to-Test Matrix

### Invariants (INV-01 through INV-14)

| Requirement | Task IDs | Test | Expected Result |
|---|---|---|---|
| INV-01 Currency Conservation | P3-01, P3-02 | Sum all account balances after 1000 ticks | Equals initial total ($4,000) |
| INV-02 Nonnegative Finances | P3-01 | Attempt purchase with insufficient funds | Transaction rejected, no state change |
| INV-03 Inventory Conservation | P3-01 | Track grain/bread stock through 100 ticks | current = initial + production + imports − consumption − exports − destruction |
| INV-04 Unique Entity Ownership | P3-02 | Assign citizen to two households | Second assignment rejected |
| INV-05 Spatial Validity | P3-02 | Move citizen to invalid tile | Movement rejected |
| INV-06 Atomic Transactions | P3-01 | Interrupt purchase mid-transaction | No partial state change |
| INV-07 Causal Event Integrity | P4-01 | Trigger disaster, check event log | Event has ID, tick, actor, affected entities, preconditions, state changes, outcome |
| INV-08 Valid State Transitions | P3-02 | Invalid state transition (e.g., EXECUTING → IDLE) | Transition rejected |
| INV-09 Deterministic Replay | P2-01 | Run 1000 ticks twice with same seed | State hashes identical |
| INV-10 Persistence Integrity | P2-07 | Save and reload after 500 ticks | All state preserved |
| INV-11 Need Bounds | P3-02 | Run 1000 ticks, check needs | 0 ≤ hunger ≤ 100, 0 ≤ fatigue ≤ 100 |
| INV-12 No Omniscient NPCs | P3-04 | Inspect citizen decision inputs | Only own needs, location, employment, money, observed prices, learned road network |
| INV-13 Deterministic Conflict Resolution | P3-02 | Two citizens apply for same job | Only one succeeds, other rejected |
| INV-14 Invariant Safety Under Chaos | P4-01 | Trigger Bread Rain, check invariants | All invariants hold, anomalous bread recorded with provenance |

### Reference Run (43,200 ticks = 30 days)

| Metric | Expected | Test |
|---|---|---|
| Ticks | 43,200 | Run headlessly, no real-time waits |
| Citizens | 24 (initial) | Population tracked per day |
| Households | 8 (initial) | Household count tracked |
| Businesses | 3 (farm, bakery, market) | Business count tracked |
| Total currency | $4,000 (conserved) | Sum all account balances = $4,000 at every tick |
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