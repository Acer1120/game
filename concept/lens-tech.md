# Lens: HOW TO ACTUALLY BUILD IT — the open-world technical architecture
2026-09-13. Concept pass: no source was edited. Everything here is from reading `hv-work/src/{12,13,14,15,21,26,28}`, the HVU module (`hv-units-source.html`, blob line skipped), `audit/HVU-map-render.md` and `audit/HVU-map-ai.md`.

## 0. The core idea in one sentence
**The existing game stays a small arena. The arena moves with the player: a streamed "hot bubble" about 60 tiles across, where every current system runs unchanged against a pooled `enemies[]`, fed by a new world layer. That layer knows the map, the factions and the war only as data, and it turns data into bodies (materialise) and bodies back into data (demote) at the edge of the bubble.**

Everything in this document follows from one invariant:

> **`enemies[]`, `colliders`, `occluders`, `portals` and the ink layer only ever hold what is inside the hot bubble.**

There are 63 `for(const e of enemies)` sites across 15 files (HVU-map-ai §0), about 40 `for(const u of units)` scans inside HVU, and linear scans of `colliders` and `occluders` in 14/15/28. Every one of them is O(bubble), not O(world). Keep it that way and none of them needs rewriting. Break it once, for example by putting a distant faction army into `HVU.units` "so it keeps fighting", and the frame dies in `pickTarget` and `resolve`, which are O(N²) at 120 Hz.

What the code already has that makes this feasible:
- **Pre-spawned pools** (`POOL`/`ensurePool`/`pickBody`/`claimBody`/`resetFoe`). The "never a makeSprite mid-run" rule means materialising is just re-typing a parked body.
- **A deterministic LCG world** (`srand` in `buildTown`).
- **A page-turn travel plate** (`portalSim`: an 0.8 s swap behind `#travel`). It is a free, on-brand loading screen.
- **Staged GPU uploads** (`WARMQ`/`warmStep`) and a "safe moment" predicate (`ladderSafe`).
- **An adaptive resolution ladder.**
- **A screen-space post pass** whose cost does not grow with the size of the world.
- **A team AI** where any unit can fight anyone.

What makes it hard is covered in section 9. The short list: there is no pathfinding at all, sprite textures are cloned per rig, the random generator is order-dependent, `T.BOUNDS` and `T.CLAMP` are hard-coded into 28 call sites, and the whole feel package freezes the entire world on every hit.

## 1. World representation

### 1.1 Chosen shape: a hybrid. Hand-authored places, procedural wilderness, persistent deltas
- **Towns, camps, fronts, boss grounds and waystones are hand-authored.** `buildTown()` is already data wearing code: the `BUILDINGS` table, the lamp list, the prop tuples `[name,x,z,h,r]`, and the tree ring from `srand`. Lift exactly that into a **Place** record: `{id, anchor:[x,z], placements:[{nat,x,z,h,r,flags}], lamps, crates, portals, arena:{x0,z0,x1,z1}, vigil:{...}}`. Ashridge Junction becomes `places/ashridge.json`, bit-identical to today's town. That makes it phase 0's regression test.
- **The wilderness between places is procedural.** It is generated per chunk from `hash(worldSeed, cx, cz, salt)`: tree scatter, rocks, grass-tick variation, road decoration. Humans still author the macro layout (roads, rivers, region borders, place anchors) as a small vector map per region.
- **Anything the player changes is a sparse per-chunk delta.** Broken barrels, a cleared camp, a lit lamp, a dead named unit. Content = f(seed, chunk) + authored overlay + delta. That keeps saves tiny (section 4).

Rejected alternatives:
- **Fully hand-authored.** Even a modest map is about 100 chunks of placements, and one person with a text editor cannot fill that.
- **Fully procedural.** It gives no landmarks and no staged fights. The combat feel is best in hand-shaped arenas (the lamp square, the gate), so procedural generation should decorate the space between those arenas, not replace them.

### 1.2 Coordinates: chunks, regions and rebasing
- **Chunk = 32 × 32 tiles.** Today's playable town (`T.BOUNDS` 52 × 68) is roughly a 2 × 2 chunk footprint, which is a sensible size for one Place. The camera sees about 45 × 35 tiles at the resting pitch, and the sketch fog (`scene.fog` near 30, far 60) swallows everything past that. So a **3 × 3 loaded ring** covers the view with a margin, and a **5 × 5 warm ring** holds prebuilt-but-hidden chunks so a dashing player never outruns the builder.
- **Region = up to 16 × 16 chunks (512 tiles)**, each with its own local origin. Crossing a region border is a page turn (the existing `travel` plate). This gives three things for free:
  1. **Float precision.** The floor UVs and sprite positions stay under about 512, so mediump fragment precision on integrated or mobile GPUs never shows as texel swim at x = 5000.
  2. **A hard streaming boundary.** The atlas families, the audio palette and the fog colour swap behind the plate. That is exactly what `enterDistrict`/`districtStep` already tween.
  3. **A natural unit for the cold faction ledger** (section 3).

  Seamless region borders are the first thing to cut (section 10).
- **The random generator must change.** `srand` is ONE global sequential LCG (`let seed=7`), so the tree ring is only deterministic because `buildTown` always runs in the same order. Chunks load in whatever order the player walks, so every generator becomes `chunkRand(cx,cz,salt)`, a counter-based hash (for example splitmix32 over `cx*73856093 ^ cz*19349663 ^ salt`). `wrand` (the wave RNG) is fine as it is.

### 1.3 What a chunk holds (runtime record)
```
Chunk { cx,cz, state:'cold'|'building'|'warm'|'live',
        ground: Uint8Array(32*32)          // biome/paint index per tile: grass, dirt, road, cobble, ash, water edge
        walk:   Uint8Array(32*32)          // 0 free, 1 blocked, 2 slow  (nav, section 1.5)
        colliders:[{x,z,r}]                // circles, same shape moveCollide uses today
        occluders:[{x,z,w,h,thin,inst}]    // same shape place() pushes today, plus instance index
        decor: InstancedBillboard|null     // one draw per atlas page (section 5.2)
        placeIds:[...], delta:{...} }      // authored Places overlapping, persistent changes
```

### 1.4 Colliders, occluders and bounds become spatial queries
Today:
- `moveCollide` walks every collider three times per body per 1/120 s step.
- `blocked`/`freeSpot` walk every collider.
- `behindDecor` (x-ray), `occRects` (FX depth ordering) and `shockDecor` walk every occluder.

The town has about 108 occluders (10 buildings, 8 lamps, 12 props, about 78 trees) and a similar number of colliders, so this is already about 1M checks a second with a full wave on screen. **Replace both arrays with a uniform grid hash (4-tile cells) owned by the loaded chunks**, and give them one query API: `forColliders(x,z,r,fn)` and `forOccluders(x0,z0,x1,z1,fn)`. The call sites change their loop header and nothing else. `HVU.host.obstacles` is a static array read in `resolve()`; it becomes a per-body query from the same grid.

`T.BOUNDS` (8 sites, all in 14-world) and `T.CLAMP` (12 in 21-waves, 8 in 23-camera) split into two new concepts:
- **World walkability.** `moveCollide`'s final clamp becomes a `walk[]` test at the chunk edge, plus region-edge colliders (cliffs, rivers, the unfinished page).
- **The encounter arena**, `ENC.arena`, a rect around the current encounter. Every `T.CLAMP` read in `placeBody`/`applyRoster`/`spawnWave` becomes `ENC.arena`. The camera's `T.CLAMP` framing clamp is removed in the open world. It becomes a soft clamp to `ENC.arena` only while a Vigil event (a defended-town wave fight, section 6) holds the camera.

### 1.5 Navigation: the missing system
There is no pathfinding anywhere today. Game enemies and HVU units both steer in a straight line (`steerTo`) and rely on collider push-out plus the `e.blocked` counter. That works in an open square. It fails the first time a werebear chases you around a house or into a forest.

Proposal, sized for integrated hardware:
- **One flow field toward the player**, recomputed at 4 Hz over the hot area (64 × 64 cells at 1 tile). That is a BFS over 4096 cells on a preallocated `Int16Array`, well under 1 ms. Every body targeting the player (`tgtKey==='P'`) samples it instead of the straight vector once it is farther than 4 tiles or `e.blocked>0`. Inside 4 tiles, the existing spacing rings, orbit and token logic stay in charge, unchanged, because that is where the combat feel lives.
- **Unit-vs-unit targets** (faction fights) use direct steering plus the same push-out. They fight in the open, and blocked-by-wall cases give up the target after `lostT`, which already exists.
- **The warm tier moves squads along an authored road graph** (nodes at Places and junctions). No grid is involved.

The walkability grid is generated from the same placements as the colliders (a building footprint marks its cells), so authors never paint it by hand.

## 2. Streaming and memory

### 2.1 The real memory problem is per-rig texture clones, not the world
`13-sprites.js` makes the frame window by writing `texture.offset/repeat`, and a THREE r128 `SpriteMaterial` reads its UV transform from `map.matrix`. So every rig clones every sheet it plays: `cloneTex(sheet(k))` for colour, the outline stack and the halo. **Each clone is a separate WebGL texture and a separate GPU upload of the same pixels.** That is the root of:
- PERF-1 (43 → 387 textures over three roster sweeps);
- the `SPR_CACHE` MRU eviction;
- the `WARMQ` upload queue;
- the `reink()` re-clone on a sketch toggle;
- the context-restore walk over `o.tex/o.otex/o.utex/o.htex`.

HVU-map-render measured about 5A texels per rig per sheet. That is fine for 14 rigs. It is not fine for an open world with 40 kinds, 32 awake rigs re-typed on every materialise, and three factions.

**Fix: share one texture per baked sheet and move the frame window into the material.** Give actor sprites an `onBeforeCompile` hook that replaces `#include <uv_vertex>` with `vUv = uv*uvRep + uvOff;`. `uvRep`/`uvOff` are per-material uniforms written by `_win()` and `boil()` in place of `tex.offset`. Every actor material shares one program through `customProgramCacheKey`. The result:
- One upload per sheet for the life of the session, however many bodies use it.
- `cloneTex`, `SPR_CACHE`, `touch`/`drop` and most of `warmStep` are deleted. Warming becomes `renderer.initTexture(sheetTex[k])` once per sheet.
- `reink()` becomes a map swap.
- Context restore walks the sheet registry, not the rigs.

This change is small (13-sprites plus the three ghost pools in 15 that clone) and it is **the enabling change for the whole project**. It goes in phase 0.

### 2.2 The second problem: baked canvases in CPU memory
The five baked canvases per sheet (colour, ring stack at 3× height, plain stack, bare ring, halo) are about 11× the sheet's pixel area. HVU-map-render measured about 102 MB of canvas backing store if all 227 HVU sheets are resident, on top of about 29 MB for the game. The bake is about 4.2 ms per sheet (median) on this machine, and 12 sheets exceed 16 ms.

- **Refcounted asset families.** The unit of loading is an art family: the 25 HVU prefixes, the game's knight/archer/slime families, and one nature atlas per region. A family loads (decode → bake → upload) when the loaded chunks or the region's faction palette reference it. It unloads 30 s after the last reference goes away: dispose the GPU textures, drop the canvases.
- **Drop the canvases after upload.** Set `texture.image` to a 1×1 stub once `initTexture` has run. On `webglcontextlost`, re-bake from the base64 source; the bake is deterministic. This trades a rare re-bake on context loss for about 100 MB of steady-state RAM.
- **Bake off the main thread, then bake at build time.** The bakers (`hatchSheet`, `outlineSheet`, `ringStack`, `haloSheet`, `natBake`) are pure 2D-canvas code:
  1. Phase 1: run them in a Worker on `OffscreenCanvas` (current Chrome, Firefox and Safari 16.4+), with the existing main-thread path as the fallback.
  2. Phase 3: run them at build time (Playwright, which the harness already uses) and ship the stacks as PNG. Boot bake cost goes to zero.

  The headless `window.HV_MAKE_TEX` guards stay, so the harness keeps working.
- **Budgets, enforced by a test rather than by hope.** GPU textures ≤ 192 MB, canvases ≤ 128 MB, `renderer.info.memory.textures` shown on the debug HUD. A harness route walks through two regions and back and asserts that texture and geometry counts return to their baseline. The GPU-leak history (PERF-1, REG-1) says this must be a gate, not a one-off check.

### 2.3 The streaming loop
The model is `warmStep`: one small job per frame, never on a held frame. Generalise it into a **world job queue with a per-frame millisecond budget**:

| situation | budget |
|---|---|
| `hitstop>0`, `M.panel`, `impactT>0` (warmStep's guard today) | 0 ms |
| an encounter is live | 1.5 ms |
| exploring | 4 ms |
| title, `travel.phase`, pause | unlimited |

Jobs, in priority order:
1. **Chunk data generation** (hash RNG → placements, colliders, walk grid, ground paint). Pure JS producing typed arrays, so it moves to a Worker in phase 1 and transfers back.
2. **Build the instance buffers** for a chunk's decor (section 5.2).
3. **Family bake** (big sheets only when `ladderSafe()`).
4. **One GPU upload.**

The **warm ring (5 × 5)** is built ahead of the **live ring (3 × 3)**. At walking speed a 32-tile chunk takes several seconds to cross, which leaves many frames to build the next row. Region changes and waystone travel happen behind the `travel` plate, whose `hold` phase is widened to "until the destination's live ring is built, minimum 0.55 s". The page turn then covers the load, and it already looks like part of the game.

**Pop-in is hidden by the drawing itself.** In sketch mode `refreshSketch` sets the fog colour to `PAPER`, so everything past the fog is unpainted page. Chunks become visible inside that paper band and never pop against a finished horizon. Newly built decor can additionally draw itself in over about 0.3 s using the ink-in (`15b` `o._ink`), only when it is inside the view.

## 3. Simulation LOD: three tiers, one invariant

### 3.1 What it costs today
HVU's `update(e)` starts with `pickTarget(e)`, which scans every unit, then does its passives, cooldowns and brain. `resolve()` is an all-pairs separation plus every obstacle. Many passives and ultimates (priest, templar, warchief aura, captain and watch damage cuts, necro and warchief death reactions) scan `units` again. All of this runs every fixed step at `T.DT = 1/120`. The game's own `enemySim` has its own all-pairs separation. At N = 30 that is a few hundred thousand cheap checks a second, which is fine. At N = 150 (the "armies" of the demo page) it is several million, with closures (`consider`), `asTarget()` allocations per candidate per step, and `Object.assign` in `dealHit`, which means GC pauses in the middle of a hitstop. **Awake bodies must be capped. Do not try to rewrite the AI to scale.**

### 3.2 The tiers

| tier | where | what exists | tick | cost model |
|---|---|---|---|---|
| **HOT** | within about 28 tiles of the player (promote), released beyond about 40 (hysteresis), live chunks only | real bodies in `enemies[]` and `HVU.units`, pooled rigs, the full AI, collisions, the feel package | 120 Hz (the existing fixed step) | **cap 24 awake, hard ceiling 32**, stepped down by the CPU ladder (section 5.5) |
| **WARM** | the loaded region, outside HOT | **squad tokens**: `{id, faction, pos, dest, nodePath, composition:{kind:count}, hp:0..1 per kind, morale, order, engagedWith}` moving on the road graph | 2 Hz, spread across frames | O(squads). About 30–60 squads per region, microseconds each |
| **COLD** | every region, including unloaded ones | **the faction ledger**: per region, per faction: strength, supply, the fronts `{a,b,strength a,b,line polyline}`, territory per Place | lazy: advanced on region entry by elapsed world time in 60 s steps, plus a 0.1 Hz tick for the current region | O(regions × factions) |

**Promote (materialise).** A warm squad whose position enters the HOT radius (or a Place whose arena the player enters) produces an **EncounterSpec**:
```
EncounterSpec { anchor, arena:{x0,z0,x1,z1}, level, mods:[...], sides:[{team, faction, roster:[kind...], hp:[0..1...], reserve:n}] }
```
`materialise(spec)` is today's `applyRoster(n, around=true)` generalised:
1. Claim a parked rig (kind-agnostic once `SHEET_ANCHOR` carries per-sheet size and pivot, per HVU-map-render §3) and a parked state record.
2. Run the complete `resetFoe` list (HVU-map-ai §4 has the exhaustive per-life field list, including `ultUsed`, `riseN`, `charmT`, `team`, and a per-body `cfg = Object.create(CFG[k])`).
3. Scale by `curve(level)` and place with `freeSpot`.

Bodies appear **outside the view** and walk in. When they must appear on screen, they use the "falls into the wave" arrival (`ED.ARRIVE_H`, `ringFx` first) or the ink-in. **Never call `makeSprite` or `spawn()` after boot.** HVU's mid-fight `spawn()` calls (slimelet split, plaguebearer, necro and warlock summons) go through `host.claim()`, so an empty pool means "the split doesn't happen", exactly as the summoner's litter already behaves.

**Demote.** When every body of a squad is beyond the release radius, not engaged with the player for 5 s, and not in `ENC.lock` (a Vigil or boss arena), write the survivors back into the token: kind counts, mean hp fraction, position centroid. Then park the bodies (`resetFoe`, `respawn=1`, `grp.visible=false`). Dead named units write a delta, so they stay dead.

**The feel package must learn who is involved.** Today every `impact()` can add hitstop (and `hitstop` scales the entire fixed step through `hsK`), `trauma`, a killcam, slow-mo, manga words and `AudioSys` stings. In a faction fight twenty metres away, that means the player's screen freezes on hits the player did not throw. Rule:
- **Feel beats fire only when the attacker or the victim is the player, the companion, or a body currently targeting the player.**
- Unit-on-unit hits outside that set get damage, flash, sparks, attenuated sound and nothing else. No hitstop, no trauma, no manga callout, no damage numbers, or small grey ones behind a setting.
- The gate is one predicate in the HVU host adapter's `fx.*` and `hitPlayer`, plus one in `impact()`.
- This is the difference between a war you walk into and a slideshow.

### 3.3 A faction war the player walks into
A **Front** lives in the cold ledger and becomes a warm object when its region loads: two opposing strengths, a line (polyline) between two Places, and a composition palette per side. Warm resolution is a simple attrition rule at 2 Hz, `dA = −kB·B·qualityB·dt` and the reverse, with the line drifting toward the weaker side. That is enough for territory to change hands while the player is elsewhere.

When the player comes within HOT range of the line:
1. **Materialise a slice, not the war.** Take the section of the line nearest the player and a budget of about 20 bodies, split by side strength. Leave headroom under the cap for the bodies that target the player.
2. **Reinforcement queue.** Each side keeps `reserve = strength − onField`. When a hot body dies, a replacement is claimed from the pool and walks in from that side's rear edge of the arena, outside the view. **Every hot death decrements the token's strength**, so the player's kills genuinely move the front. The ledger and the bubble are one number seen at two resolutions.
3. **Teams generalise from "not my team" to a relations table.** HVU compares `u.team !== e.team` in about 15 places (`pickTarget`, `dealHit`, the projectile hit tests, the ultimates). Replace these with `hostile(a,b)`, a lookup `REL[a][b]` over faction ids. The player's row is driven by reputation. `team = −1` ("leave me alone") keeps its meaning for travel, death and cards (HVU-map-ai §2). `charm` and MUTINY become temporary relation overrides.
4. **Crowd extras sell the scale.** Beyond the HOT slice, inside the fog band (30–45 tiles), draw **non-simulated extras**: one instanced billboard draw per atlas page, using the families' existing idle, walk and attack loops. No collision and no hits. They drift along the front line and thin out as the token strengths fall. Stay honest here: they must stay out of reach (in the fog, across the line), or players will swing at them and the illusion breaks. A shared ambient bed ("front" layer) in `AudioSys` carries the rest.
5. **The HVU token cap protects readability.** HVU already limits attackers per target (`TOKENS=2`, and the game's `attackCap()`). HVU-map-ai §9's unified count (units plus slimes against `P`) is mandatory here. Otherwise the player who steps into a battle gets hit by everyone at once.

What you get: 20–24 real, fully animated, fully feel-equipped bodies in a melee that the player can tip, embedded in a war that exists as numbers. What you do not get: 200 simulated soldiers. Section 10 cuts that.

### 3.4 Smaller sim changes inside HOT
- **Target selection at 10 Hz.** Run `pickTarget` staggered (`(e.id + step) % 12 === 0`, or when the target died or `e.threat` changed). The rest of `update` stays at 120 Hz. `pickTarget` already has hysteresis (`cd < bd*1.8`), so this is invisible and it removes the largest per-step scan.
- **No closure or object allocation per candidate.** Remove `asTarget()` allocations per candidate per step (cache one target proxy per unit, refreshed in place) and `Object.assign` inside `dealHit`. At a cap of 32, the O(N²) loops stay: they are cheaper than a spatial grid at that N, and leaving them alone avoids touching about 40 proven loops.
- **Per-body dt for the domain.** `HVU.tick(dt, scaleFn)` applies the 0.42× domain slowdown per body, as HVU-map-ai §5 says. The same hook gives distant-but-hot bodies a cheaper "background" tick: skip brain decisions on alternate steps when the body is more than 20 tiles away and has no player token.

## 4. Save/load and persistence

### 4.1 What exists
Seven `localStorage` keys (`hv_set`, `hv_sketch`, `hv_marks`, `hv_taught`, `hv_best`, `hv_rank`, `hv_score`), all small and all meta. There is no run state to save, because a run is meant to end. The perks write straight into `T` and `ARC`, and `snapT`/`restoreT` put the table back.

### 4.2 What the open world saves
One versioned JSON document per save slot, in **IndexedDB**. `localStorage` is synchronous, capped at about 5 MB, and a stall on the main thread is a visible hitch. The settings, marks and taught keys stay in `localStorage` as they are.
```
Save v1 {
  v:1, gen:{world:"<seed>", generators:{scatter:3, poi:1}},   // generator versions (see 4.3)
  t: worldSeconds, region:id, pos:[x,z], vigil:placeId,        // last Vigil = respawn point
  player:{cls, hp, energy, perks:{id:level}, style:{score,bestRank}, abilities:{...}, inventory:[...], rep:{factionId:n}},
  companion:{...},
  ledger:{ regions:{id:{factions:{id:{str,supply}}, fronts:[...], owner:{placeId:factionId}}} },
  squads:{ regionId:[token...] },                              // only the CURRENT region's warm tokens; others regenerate from ledger
  places:{ placeId:{cleared:t, vigilLevel:n, lamps:[...], flags:{...}} },
  chunks:{ "r:cx:cz":{ broken:[poiId...], opened:[poiId...], t } },   // sparse deltas, pruned by respawn timers
  named:{ unitId:{dead:true} | {alive, hp, pos} },
  quests:{...}
}
```

**What is never saved:** pooled bodies, rigs, FX pools, pickups, the HOT bubble, or any encounter in progress. Saving is only allowed at a **safe moment** (no live encounter, no `travel.phase`, no `CARDS.on`, no `hitstop`; a stricter `ladderSafe()`), so load never has to reconstruct a mid-fight state. Autosave triggers are Vigil lit, region entered, waystone used, and every 3 minutes of exploring. Writes are debounced and serialised off the frame via `requestIdleCallback`.

### 4.3 The determinism trap
Deltas refer to things by id. Procedural things need ids that survive a change to the generator. Rules:
- **Decorative scatter (trees, rocks, grass) never carries state.** It can be regenerated freely and has no ids.
- **Anything that can carry state** (a chest, a breakable shrine, a camp, a named spawn) comes from a separate **point-of-interest generator** with its own version number, or is hand-placed. POI ids are `hash(region, cx, cz, poiIndex)` under a frozen generator version. Changing that generator requires a migration that either keeps the old version for visited chunks or drops their deltas.
- **Barrels and crates respawn on a timer** (today they come back on even waves via `restoreCrates`), so their deltas expire and the delta table stays pruned.

### 4.4 Perks and the `T` table
The perk functions multiply or add into `T` and sometimes into `P` (`vigour` adds hp, `well` refills energy). Load = `restoreT()`, then replay each perk's `f()` `level` times in a fixed order, then **overwrite `P.hp`, `P.energy` and `P.armor` from the save**. The multiplicative ones commute, so order does not matter for them. Put a harness assertion on this: `T` after save → load equals `T` before. It will catch a future perk that is not replay-safe.

### 4.5 Three resets instead of one
`resetRun()` today does everything at once: it restores `T`, clears the perks, cards, wave, bounties, sigils, slicks, booms, pickups and lamps, un-golds the portals, and resets the district fog. `startGame()` resets the player, companion, camera, feel timers and M-panels. Split these into three functions:
- **`resetEncounter()`**: the WAVE fields, sigils/slicks/booms, the bounty, parking every hot body, `clearFxPools()`. Called on demote, on travel, and when an encounter ends.
- **`onPlayerDeath()`**: the death card → respawn at the last Vigil, `resetEncounter()`, plus the camera and feel resets from `startGame`. It keeps perks, rep and the ledger. The design lenses decide the death penalty (for example, lose unbanked style score); this lens only guarantees the state split.
- **`newGame()`**: today's `resetRun` + `startGame` + a new world seed.

## 5. Rendering and performance on integrated graphics

### 5.1 Where the frame goes today, and what the open world changes
- **The post pass (`SKETCH_FS`).** One full-screen quad over `sceneRT` (NEAREST, at the drawing-buffer size). 8 texture reads per pixel on the clean full path, 2 on `cheap`. **Its cost is screen-sized and does not grow with the world.** The adaptive ladder (`prLadder`, four rungs down to 0.5× DPR, `cheap` from rung 2) already governs it.
- **The scene render into `sceneRT`.** Its cost is draw calls (CPU) plus overdraw (fill). **This is what the world size changes.**
- **The 2D ink canvas (`25-manga`).** A full-screen 2D canvas redrawn every frame at `IDPR ≤ 1.25` in sketch mode. Its cost scales with how much is drawn: callouts, bars, stun stars, telegraphs, damage numbers, and 7 loops over `enemies`. The browser composites it over the WebGL canvas.
- **`renderPanel`.** It only re-renders the subject into its own targets during cut-ins. Unchanged.

### 5.2 Floor and decor
**The floor.** Today it is one 160 × 160 plane with a 1024 px wash tiled 10×, a detail layer at 7×, and a 200 × 200 `edgePlane` whose hand-cut window is the edge of the town. Replace it with:
- **One ground quad that follows the camera**, snapped to whole chunks so the texture never swims, drawn with a small `ShaderMaterial`. It takes the existing `paperFloorTex` and `paperFloorDetailTex` in world-space UVs, relative to the region origin.
- **A "paint map"** `DataTexture` (the 5 × 5 warm ring's ground indices, 160 × 160 texels, NEAREST, updated in place when the ring shifts). It selects per-tile tints and tick textures: grass, dirt road, cobble, ash, and the Hollow's grey (the `DISTRICTS[i].floor` colours are exactly this palette).
- **Deliberate road edges.** The post pass's Sobel inks luminance steps, so a road edge that is a hard value step comes out as an ink line. That is what a drawn map should look like. Biome blends must be soft, or they print as noise lines.
- **The `edgePlane`'s construction-line window becomes the region border and the unexplored edge**: undrawn page with pencil construction lines.

Draw calls for all of this: 1–2.

**Decor.** Today every prop is its own `THREE.Sprite` with its own `SpriteMaterial` (the map is shared, so there is no duplicate upload) plus a shadow mesh. The town has about 108 props, which is about 216 draws. Wilderness density with forests is 2–4× that per chunk, so this has to change:
- **Instanced billboards per chunk per atlas page.** An r128 `InstancedBufferGeometry` with a sprite-equivalent vertex shader (camera-aligned quad, `center` offset, per-instance position, scale, atlas rect, tint, sway phase and shake rotation). The nature art (`NAT`) is packed into 1–2 atlas pages per region at bake time, padded by at least 3 texels because `natBake` dilates the rim by 2 and the filter is NEAREST.
- **Depth order.** The camera yaw is fixed (`T.CAM_OFF`; the page tilt and dutch are small rolls), so instance draw order is sorted once by z when the chunk is built. Chunks get `renderOrder` bands by chunk row, inside today's prop band (`layer()` puts props at 1000 + z·8, below actors at 5000+). That preserves the current painter's order exactly.
- **Sway and shake.** Tree and lamp sway moves into the vertex shader (`phase` attribute + a time uniform), which costs zero CPU. `shockDecor`/`stepShake` write a `rot` attribute for only the shaken instances, using `updateRange`.
- **One shared shadow material.** Prop shadows become a second instanced draw of the hard ellipse, keeping `natShadowMat`'s single-map swap on the BOIL clock.
- **Occluder records keep an instance index**, so the x-ray (`behindDecor`), `occRects` and `shockDecor` still find the drawing they refer to.

**Estimated draws in the open world:**

| layer | draws |
|---|---|
| ground | 2 |
| decor (9 live chunks × 1–2 pages × 2) | about 20–36 |
| actors (24–32 × about 4: shadow, line, body, flash; halo and silhouette only when used) | about 100–130 |
| crowd extras | 1–2 |
| FX pools (unchanged, fixed rings) | as today |

That is fewer than today's town, in a larger world.

**Overdraw is the integrated-GPU risk.** Twenty overlapping tree canopies at full DPR can cost more fill than the post pass. Mitigations, in order:
1. A generator rule capping canopy coverage per screen-sized window.
2. "Treeline mass" drawings: one baked wide sprite for a forest edge instead of 30 trees.
3. An experiment worth one day: the decor is pixel art with hard alpha, so it can draw in the **opaque pass with `alphaTest 0.5` and `depthWrite:true`**, front to back, so early-z rejects hidden canopy. Because every billboard faces the camera, depth-testing them gives the same painter's result as the current `renderOrder` sort. The actors' x-ray silhouette would then need a `depthFunc: GreaterDepth` pass. Keep this only if a screenshot A/B shows zero change to the ink look.

### 5.3 Actors
- The shared-texture sprite material from section 2.1.
- The per-sheet anchor change from HVU-map-render §3.
- A **generic rig pool of 32**, allocated at boot.

That is the whole actor story. Draws per actor stay as today, and none of the look systems change (boil on `BOIL.ov`, on-twos through `commitPose`, x-ray, ghosts using `sheetRing`, and the hard ellipse shadow). HVU-map-render §6's rule stands: **never draw `sheetHalo` as a line** (the FIX4 black-body bug).

### 5.4 The ink canvas
- **Only HOT bodies that are on screen and engaged** get per-body ink (bars, stun stars, guard chevrons, telegraph clocks). A faction fight in view but not involving the player draws no bars.
- **Manga words and callouts obey the involvement rule** from section 3.2.
- **New open-world chrome (compass, quest marker, minimap, region name) lives in DOM/SVG** (the paper UI already does bars and dials this way), updated at 4 Hz or on change. Never per frame on the ink canvas.
- **The minimap is a small offscreen 2D canvas** redrawn when the chunk ring shifts, not a second WebGL render.

### 5.5 A second ladder: CPU
Today's ladder assumes the GPU is the bottleneck: resolution is the only knob. An open world adds CPU-bound frames: AI, streaming jobs, the flow field. Time the fixed-step loop and `renderUpdate` separately with `performance.now()`. The GPU-side time is roughly the frame time minus that.
- **GPU-bound** (`emaMs` over target while CPU time is under half the target): walk `prLadder` exactly as now.
- **CPU-bound:** walk a **density ladder**, one rung per 2.5 s under the same safe-moment rules:
  - awake cap 32 → 24 → 16;
  - `pickTarget` 10 Hz → 6 Hz;
  - crowd extras 100% → 50% → 0;
  - the decor small-prop instance count (drop every second rock or tuft instance; trees stay);
  - the streaming job budget 4 → 2 ms.
- **Target hardware:** Chrome on an Intel UHD 620 or Iris Xe laptop at 1080p, DPR 1. **The harness cannot measure this.** SwiftShader runs at about 5 fps and its timings are meaningless, so every performance gate in the roadmap needs a real machine, and a `?perf` overlay that logs frame, sim, render and job ms plus draw calls and texture count.

## 6. Refactor, don't rewrite: the existing systems in the open world

**The single most important product decision in this lens: the current game becomes a content type.** A **Vigil** is a hand-authored Place whose `vigil` block runs today's loop unchanged: the typed pool, `composeWave`, the curve, modifiers, elites, bounties, the lamp objective, the barrels, the boss at wave 5 and 10, and cards between waves. Ashridge Junction is the first Vigil. Lighting a Vigil makes it the respawn point and a waystone. So the thing that is already fun ships on day one of the open world, and every open-world phase can be regression-tested against it.

| existing system (file) | becomes | what changes |
|---|---|---|
| `buildTown()`, `BUILDINGS`, the lamp, prop and tree lists (14) | `loadPlace(placeJson)` stamping into chunks; the tree ring moves to the Place's authored scatter with its own `chunkRand` salt | the data moves out of the code. `place()`/`addCollider()` push into the chunk grid instead of the globals |
| `colliders[]`, `occluders[]`, `moveCollide`, `blocked`, `freeSpot`, `behindDecor`, `occRects`, `shockDecor` (14/15) | grid queries (section 1.4) | loop headers only |
| `T.BOUNDS` / `T.CLAMP` (14/21/23) | world walkability + `ENC.arena` | 28 call sites, mechanical |
| typed pool `POOL`/`ensurePool`/`pickBody`/`claimBody` (21) | **the encounter pool**: 32 generic rigs + per-kind state records, boot-allocated | kind-agnostic rigs (per-sheet anchor). `POOL` counts become the per-region palette's maximums |
| `applyRoster(n, around)` / `placeBody` / `resetFoe` (21) | `materialise(EncounterSpec)`; `resetFoe` + HVU-map-ai §4's unit field list | `n` becomes `level`; `around` becomes `spec.anchor`; `T.CLAMP` becomes `spec.arena` |
| `composeWave(n)`, `ECOST`, `EMAX`, `curve(n)`, `MODS` (21) | `composeRoster(budget, palette, mods)`; `curve(level)`; region and front modifiers | the budget comes from the squad token or the Vigil. `DISTRICTS[i].mod` ('nightfall', 'frenzy') becomes the region's ambient modifier |
| `WAVE` gap, `spawnWave`, bounties, lamps (21) | unchanged inside Vigils. Outside, an encounter clears when its hot bodies are dead or have retreated | `WAVE` becomes `ENC.wave`, present only while a Vigil runs |
| choice cards `CARDS`/`offerCards`/`takeCard`, `PERKS`, `snapT`/`restoreT` (21, 27) | same UI and data. Offered on Vigil wave clears, at shrines, on region-lord kills, and on level thresholds of style score | perks persist (section 4.4). The 3-stack cap stays. Respec happens at a lit Vigil (`restoreT()` + replay). `RUN.extra` (the bounty-paid extra pick) stays |
| portals `makePortal`/`portalSim`/`travel` (14) | **waystones**: `link` becomes a destination picker over lit Vigils. The travel plate's `hold` waits for streaming | the swap already moves `P`, snaps the camera and blocks input. The companion's leash blink (`22` `ARC.leash`) already recovers her |
| `DISTRICTS`, `enterDistrict`, `districtStep` tween (21) | `REGIONS` + `enterRegion(i)`: same fog, paper and floor crossfade | driven by crossing a region border, not by `RUN.gate` |
| `onBossDown` → `RUN.gate` → gold portals (21) | the region lord's kill opens that region's far waystone and flips the ledger | the same shape, one level up |
| `resetRun` / `startGame` (21/27) | `resetEncounter` / `onPlayerDeath` / `newGame` (section 4.5) | a split, no new logic |
| `enemies[]` (63 sites), `enemySim` | unchanged, holding only HOT bodies | the invariant |
| HVU `units`, `pickTarget`, `resolve`, `dealHit` | unchanged except: `hostile(a,b)` in place of `team!==team`; `host.obstacles`/`bounds` as queries; `host.claim` and `host.park` in place of `spawn` and splice; staggered `pickTarget`; the feel-involvement gate | about 20 small edits inside the module |
| HVU `render` object | **deleted** (HVU-map-render §1); units draw on `makeSprite` | as that map says |
| FX pools (15), decals (15b), manga (25), panel (26), HUD (27) | unchanged; `clearFxPools` on travel; involvement gate on callouts | nothing structural |
| marks, style rank, audio (16/17/27) | persist in the save; `AudioSys` voice cap follows HOT bodies; a front ambience bed | small |
| adaptive ladder (28) | + the CPU density ladder (section 5.5) | additive |

## 7. File format: single HTML, or a small build with asset files?

### 7.1 Where it stands
- **The game:** 927 KB (`hollow-vigil (2).html`), built by `build.sh` concatenating `src/`. The sprite blob is 118 KB of base64.
- **The units module:** 825 KB, with a 669 KB base64 blob. Together about 790 KB of base64 sprites.
- **Three.js r128 from cdnjs**, so the "single file" already needs the network.
- **The two blobs are not the heavy part.** Base64 costs parse time and about 33% size overhead. What costs memory is decode + bake (about 130 MB of canvases if everything is resident), and both of those are avoidable on demand whatever the container is.

### 7.2 Recommendation: stay single-file through phase 2, go multi-file in phase 3
- **Phases 0–2 stay single-file, with lazy families.** Keep `build.sh`. Each art family and each region's Place and scatter data is its own `<script type="application/json" id="fam-orc">` block. Base64 strings in the DOM cost little until they are decoded. Decode and bake by family through the job queue. That keeps the property the user relies on today, one file that opens anywhere, while the world is one region of about 1.5–2.5 MB.
- **Phase 3 goes multi-file** (`index.html` + `assets/`), when the game crosses about 4 MB or a second region's art arrives. A single inlined file then means every player downloads every region to see the title. The steps:
  - `build.sh` gains a release mode that emits `index.html`, `game.js`, `vendor/three.r128.min.js` (vendored; this also removes the cdnjs dependency), `assets/fam-*.png` (build-time baked stacks, section 2.2), `assets/regions/*.json` and `assets/nat-*.png` atlases.
  - Loading uses `fetch` + `createImageBitmap`, which decodes off the main thread.
  - A **service worker** caches the lot, so the game is still offline after the first visit.
  - **Caveat:** `fetch` from `file://` is blocked in Chrome. Once multi-file, the game must be served (any static host, or `npx serve`). The build therefore also keeps an **`--inline` target** that produces today's single-file form (all assets base64-inlined, larger and slower to boot) for sharing as one file.
- **Do not move to a bundler or a framework, or off Three r128.** Section files plus a shell script have carried a 30-file, 900 KB game through three overhauls, and every audit, probe and harness in `hv-work/` assumes that layout. New modules slot into the numeric order (`14w-chunks.js`, `14n-nav.js`, `21e-encounter.js`, `21f-factions.js`, `29-save.js`), and the TDZ-safe "declare early, call at runtime" pattern the units map documents still holds.
- **Run the swallowed-comment audit on every new file.** Seven live bugs came from dense one-liners with trailing `//` comments swallowing code. New world-layer files should be written multi-line from the start.

## 8. Phased roadmap
Each phase ends in a build the user can play, with gates checked on **real hardware** (not the SwiftShader harness) plus harness regression checks for errors, state and screenshots.

### Phase 0: prove the open world works with this combat (prototype, not shippable)
**The question it answers:** does the feel survive when the arena is not a closed box, and does the engine hold at 60 fps while streaming?

Deliberately ugly: one flat region, procedural trees, a straight road, no new art.
1. **Shared-texture actor material** (section 2.1). *Gate:* `renderer.info.memory.textures` is flat across 20 roster sweeps and 5 restarts. The existing smoke harness passes, and a screenshot diff of the knight, a slime and an HVU unit against today shows no change.
2. **Per-sheet `SHEET_ANCHOR`** plus a 32-rig generic pool. HVU bodies draw on `makeSprite` (HVU-map-render §3–§10, minimal).
3. **Chunk grid + camera-following ground + streaming of plain `natureSprite` props** (no instancing yet), with `chunkRand` replacing `srand`. Ashridge is stamped from a JSON Place at the region centre; `T.BOUNDS` becomes walkability and `T.CLAMP` becomes `ENC.arena`. *Gate:* walk 3 minutes in a straight line and back. No frame over 50 ms on the reference laptop, zero chunk-build frames inside a hitstop, and the Ashridge Vigil plays wave 1–10 exactly as today.
4. **One warm squad token on the road** (6 HVU bodies: watch, captain, knife) that materialises inside 28 tiles and demotes beyond 40. *Gate:* 50 scripted promote/demote cycles via `__HV` with no field leakage (a snapshot of every body's fields after `resetFoe` equals a fresh one), and a materialise frame under 4 ms.
5. **One two-sided skirmish** (a watch squad against an orc squad) with `hostile()`, the reinforcement queue and the **feel-involvement gate**. *Gate, subjective and the most important:* the user plays into it and says the hits still feel like Hollow Vigil, and the screen never freezes on hits they did not take part in.
6. **The flow field** for player-targeting bodies. *Gate:* no body stuck behind a house for more than 2 s across 10 scripted chases.

If step 5's gate fails, stop. The open world must then be built as linked arenas (Vigils plus waystones plus handcrafted roads as corridors). That is still a strong shape, and the later phases scale down to it (section 10).

### Phase 1: "The Road" (the first shippable open world)
- **Region 1:** about 8 × 8 chunks (256 × 256 tiles). Ashridge Junction as the Vigil, two hamlets (small Places), one bandit camp (a handcrafted arena encounter), one waystone pair, and roads with procedural wilderness between.
- **Decor instancing and atlases** (section 5.2) and the paint-map ground.
- **Job queue with a Worker** for chunk data generation.
- **Family refcounting** and dropping canvases after upload. *Gate:* the texture and geometry counts return to baseline after leaving and re-entering the region 5 times.
- **Save v1** in IndexedDB (player, perks, Vigil, chunk deltas, cleared camps). *Gate:* `T` is identical across save → load; the save is under 50 KB.
- **The three resets**; death respawns at the Vigil.
- **Cards** from Vigil waves, the camp and shrines.
- **The CPU ladder.**
- **DOM compass and region title.**
- *Ships as the single HTML file.*

### Phase 2: "The Front" (factions and war)
- **Three factions from existing HVU families.** For example an Order (watch, captain, knight, templar, priest, pike), the Hollow (skel, ironskel, necro, revenant, warlock, greatskel) and the Clans (orc, eliteorc, ironorc, berserker, warchief, axeman). The final cast is the design lenses' decision.
- **The cold ledger, warm squads on the road graph, and one moving front** in region 1.
- **Relations and reputation**; charm and MUTINY as relation overrides.
- **Crowd extras** and the front ambience.
- **Named units** as persistent records (the four named rogues are the obvious first set).
- **A region lord boss** whose kill flips the ledger and opens the far waystone.
- *Gate:* play 30 minutes and leave the region twice. The front has visibly moved, the numbers are consistent (ledger strength equals hot deaths plus warm attrition), and there are no GC pauses over 16 ms during a front fight (Chrome performance panel on the real laptop).
- *Still single-file*, with lazy families (it will sit around 2.5–3 MB).

### Phase 3: "The Map" (scale and the build)
- **Regions 2 and 3** (the Lamplit Rows and the Hollow, the existing district fiction) with their own atlases and palettes. Region-border page turns.
- **Multi-file release build**, build-time baked PNG stacks, vendored Three, a service worker, and the `--inline` share target kept.
- **Bakes in a Worker** (OffscreenCanvas) for dev and for the inline build.
- **Save migrations** (v1 → v2) and POI generator versioning.
- *Gate:* a cold boot to the title in under 2 s on the reference laptop from a warm cache, and a region change behind the plate in under 1.5 s.

### Phase 4: content and chrome
Quests (data-driven flags on Places and named units), the minimap, interiors as portal-linked micro-arenas, more Vigils, Endless as a late-game Vigil modifier, and the remaining HVU kinds (only once a faction palette needs them: fewer families means less memory).

### De-risking order, restated
1. Texture sharing (it unblocks memory for everything).
2. Streaming plus bounds removal, with Ashridge unchanged (it proves the refactor-not-rewrite claim).
3. Promote/demote correctness (the leak class of bug this codebase is prone to).
4. The involvement-gated feel in a multi-side fight (the actual design risk).
5. Navigation.
6. Only then scale: instancing, factions, saves, more regions.

Art and content come last, because nothing about them is technically uncertain.

## 9. What is genuinely hard (honest list)
1. **Navigation does not exist.** A flow field for chasers is cheap. Units fighting each other among buildings, ranged units finding line of sight, and squads crossing forests are all new problems. Keep Places open-plan and forests as obstacles with road gaps, and accept "gives up the target" behaviour.
2. **State leakage on re-typed bodies.** HVU-map-ai §4 lists about 60 per-life fields. The open world re-types bodies hundreds of times per session instead of about 11 per wave. A missed field such as `ultUsed`, `charmT`/`team` or a shared `cfg` mutation (the berserker's `Object.assign` onto `cfg`) becomes a heisenbug. A field-snapshot test is mandatory from phase 0.
3. **The feel package is global.** `hitstop` scales the whole fixed step, `timeScale`/`slowT` slow the world, and killcam, letterbox, `trauma` and `M.panel` are single global channels. The involvement gate handles NPC-vs-NPC hits. Two simultaneous player-involved beats in a big fight still compete for one killcam, so the queueing rules must be tuned, not just gated.
4. **Tuning numbers that assumed a closed arena.** Aggro radii, `ARC.leash`, spawn clamps, the camera framing that includes nearby enemies (23-camera), and pickup magnets were all tuned for a 28 × 48 play rect. Expect a long tail of "this felt right in town and wrong on the road".
5. **Streaming hitches you cannot see in the harness.** Every performance claim needs the real laptop. The project has repeatedly shipped performance work that was reasoned about but never profiled on real hardware (MEMORY: "never profiled on real GPU hardware").
6. **Determinism across generator changes** (section 4.3). Easy to get wrong once and corrupt saves.
7. **Faction simulation that is legible.** A front that moves while you are away is only a feature if the player can see why. That is a UI and design problem more than a technical one, but it decides whether phase 2 is worth its cost.
8. **Headless coverage.** `HV_MAKE_TEX` returns unbaked images and SwiftShader runs at about 5 fps, so the harness can verify state and errors but not look or performance. The open world multiplies the paths to test, so the `__HV` hook needs world verbs: `__HV.world.teleport`, `.promote`, `.demote`, `.ledgerStep`, `.save`, `.load`.

## 10. What to cut
- **Seamless borders between regions.** Use page turns. They are on-brand and bound float precision and memory.
- **A simulated army.** Cap HOT at 24–32 bodies. Wars are ledger plus slice plus extras.
- **Camera rotation and free yaw.** The fixed yaw is what makes the static z-sort, the instancing order, the x-ray and billboards all work.
- **Terrain height.** The ground stays flat. `hy` stays jump and fly height only. Cliffs and rivers are drawn edges with colliders.
- **Dynamic lighting and a day/night cycle.** Fog, paper tint and the existing NIGHTFALL-style modifiers give "night" without any lighting system; the post pass would fight real lighting anyway.
- **Procedural towns.** Towns are hand-authored; only the wilderness is procedural.
- **All ~40 HVU kinds in phase 1.** Ship the families the faction palettes need. Every family cut is memory and bake time saved.
- **Seamless interiors.** Interiors are portal-linked micro-arenas.
- **The HVU `render` object, the drawn shadow strips, and team-2 "army mode" as a player-facing feature** (HVU-map-render §1, §6; HVU-map-ai §2).
- **If phase 0's feel gate fails, cut the continuous open world** in favour of **linked arenas**: Vigils, camps and fronts connected by short handcrafted road corridors and waystones. Sections 2, 3.3, 4, 6 and 7 all still apply unchanged. Only chunk streaming and the flow field shrink.
