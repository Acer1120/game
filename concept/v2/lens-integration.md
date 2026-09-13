# Lens: how HV Units v30 gets into the game, and its UI

Design pass, 2026-09-13. No game source edited. Everything below was checked against:
- `hv-work/units30/units.js` (v30 module = lines 60-1638; demo = 1639-2132)
- `hv-work/units30/head.html` (demo CSS and markup)
- `hv-work/src/*` (the game as shipped in `/workspaces/game/hollow-vigil (2).html`, 927,028 B)

The v30 module was also **loaded headless in node** from the real source with the blobs parsed but not printed. The script is `scratchpad/load.js` (`new Function` over lines 60-1638 plus `HVU_SHEETS` / `HVU_META`). Numbers marked *(measured)* come from that run or from `sheets.js` over line 115.

## 0. Verdict

**Vendor v30's module unchanged apart from five named one-line patches. Wire the 14-member host to the game's real functions in one new section file, assigning `HVU.host` wholesale (never `HVU.attach`). Draw units through the game's own sprite rig from `HVU.drawInfo()`, never through `HVU.render`.**

The author's guide is a good map of the contract but a bad map of the game:
- Five of the game functions it names do not exist.
- Three of its instructions silently do the wrong thing: team 0, `render.load(startGame)`, `spawn` from `rosterFor`.
- One of its host shortcuts (`attach`) drops the new `fx.rite` hook.

v30 contributes one idea the game should adopt outright: **one GPU texture per sheet with the frame window in the material.** That is exactly v1 plan phase 0 step 1. The game gets it by porting the idea into `makeSprite`, not by installing v30's renderer.

The delivery stays **one HTML file**. It inlines Three.js r128 (dropping the CDN), and unit sheets travel as **per-family JSON blocks decoded lazily**. The budget is ≤ 3.1 MB raw / ≈ 1.5 MB gzip with every family.

The demo's army UI is a developer's instrument panel. In the game:
- It shrinks to an **army strip of portrait chips** in combat.
- It gets a **one-key order dial**.
- Rites and evolution happen **only in the wave gap**.
- The character sheet becomes a **pause page**.
- The Lab, drawer, dock and topbar stay dev-only.

---

## 1. The author's drop-in guide, walked against the real game

The header says "five steps" and then numbers six. They are taken in order below.

### Step 1 — "Paste the `<script>` with HVU_SHEETS / HVU_META and the HVU block after HV_SPRITES, before the game code."

**Works, with one correction on where it goes.**
- `build.sh` cats `00,01,02,03-sprites,04-script-open` and then globs `1*.js 2*.js`.
- The game's script is one IIFE opened in `10-config.js` (`(() => { 'use strict'; ...`).
- The HVU block is a top-level `const HVU=(function(){...})()`. So:
  - The sheets/meta go in a new **`03u-units.html`** `<script>` tag, after `03-sprites.html`. `build.sh` must name it explicitly.
  - The module goes in its own `<script>` tag, **before** `04-script-open.html`. It then sits outside the game IIFE as a global `HVU`, exactly like `window.HV_SPRITES`.
  - Do not concatenate it into the `1*.js` glob. It would run inside the game's IIFE, but that buys nothing and costs two things:
    - the module would silently switch to strict mode, a mode it was never tested in;
    - the vendored file would stop being a separate, byte-diffable artefact.
  - Keep it as `03v-hvu-module.html`, so the five patches below stay a readable diff against `hv-units-30-source.html`.
  - Extend `build.sh`'s syntax check to cover this `<script>` too. Today it checks only the last `<script>`.
- *(measured)* The module evaluates with **no DOM and no THREE** at load: node ran it with a `document` stub returning null. Only `render.install` touches THREE. Load order relative to `three.min.js` is therefore free.
- Do **not** paste the 1.33 MB `HVU_SHEETS` literal as-is. See §5: it becomes per-family blocks.
- Do **not** paste lines 1639-2132 (the sandbox: arena, stand-in hero, demo `HVU.host`, UI). The demo reassigns `HVU.host` wholesale at line 1821 and would clobber the game's.

### Step 2 — "Wire the host once, with HV's own functions: `HVU.attach({...})`"

**Does not work as written.** First, `attach()` (units.js:1609) is lossy:
- It copies `player, playerVel, playerWinding, playerRetreating, hitPlayer, freeSpot, telegraph, schedule, spawnFx, sfx, onRemove`, plus `bounds, obstacles`.
- For fx it copies only `puff, shock, number, bits, kill, spinFx`, and `shout|mark`.
- **`fx.rite` and `fx.ult` are not copied.** The v30 rite (level milestones, `advance`, `evolve`, ascension) would stay a no-op.

**Rule: assign `HVU.host = HVU_HOST` wholesale in `boot()`.**

The 14 members, the guide's suggestion, and what actually exists:

| member | guide says | exists in game? | real implementation |
|---|---|---|---|
| `player()` | `()=>P` | `P` exists (18-state-input.js:4) but lacks `r, maxHp, dir, stunned, team, velx, velz` | `hvuPlayer()` decorates `P` in place (never allocate: called per unit per tick). Sets `P.r=T.RADIUS` (0.40), `P.maxHp=T.HP`, `P.dir=P.facing`, `P.stunned=P.lastStand>0`, and `P.velx/velz=(P.x-P.px)/T.DT` (`playerSim` writes `px/pz` first, 20-player.js:246). **`P.team = (P.dead\|\|travel.phase\|\|!running) ? -1 : 1`** (see §2). |
| `playerVel()` | — | — | `()=>[P.velx,P.velz]`. **Not `P.vx/vz`**: that is the game's knockback channel only. |
| `playerWinding()` | — | `atkDef(idx)` 20-player.js:15 | `()=>P.atk>=0&&P.atkT<atkDef(P.atk).wind \|\| P.charging \|\| P.aiming` |
| `playerRetreating(e)` | — | — | the demo's dot product (`>2.5`) on `velx/velz`, plus `P.dashT>0` pointing away |
| `hitPlayer(h,e)` | `hurtPlayerFromUnit(h,e)` | **does not exist** | `hvuHitPlayer(h,e)`: <br>• return false unless `running && !P.dead && !travel.phase` <br>• `if(!HVU.inHit(h,P.x,P.z,T.RADIUS)) return false` <br>• if `P.iframes>0`: when `P.dashT>0 && e`, call `perfectDodge(e._foe,nx,nz)`; return false <br>• `hurtPlayer(h.dmg,nx,nz,e?e._foe:null,!!h.poisonTick)` (20-player.js:207) <br>• `if(h.knock){P.vx=nx*Math.max(T.HURT_KNOCK,h.knock);...}` <br>• return true <br>`e` is **null** for v30's player poison tick (units.js:1613), so every `e.` must be guarded. |
| `freeSpot(x,z,r)` | `freeSpot` | yes, 14-world.js:17, same signature and `[x,z]` return | direct |
| `bounds` | `{x0,z0,x1,z1}` | `T.CLAMP` (10-config.js) | `T.CLAMP`, not `T.BOUNDS` (that is the hard world edge 12 tiles outside the fight) |
| `obstacles` | `decorCircles` | **does not exist**; the real list is `colliders` (14-world.js:2, `{x,z,r}`) | `colliders` (live array; broken crates set `r=-1`) |
| `telegraph(t)` | `M.telegraphs.push` | **`M.telegraphs` does not exist** (M literal, 25-manga.js:19) | add `M.hvuTele:[]`. Step `t.t+=dt` and drop at `t.t>=t.life\|\|t.life<=0` in the sim. Draw it in `mangaStep` after the `telDraw` block (25-manga.js:617). v30 telegraphs carry `arc`, `shape{F,B,D,kind}`, `capsule{len,r}`, `single`, and beam `arc:deg(28)`: wedges, eggs, capsules and beams, which the game's circle-only `ringFx` telegraph cannot draw. |
| `schedule(d,fn)` | `M.timers.push` | **`M.timers` does not exist**; the only scheduler is the typed `schedAdd` (15b) | add `M.hvuTimers:[]` stepped on the sim clock. v30's default is `setTimeout` (wall clock, runs through pause and hitstop). |
| `spawnFx(sheet,x,z,y,dir)` | — | — | push a `type:'fx'` entry into `HVU.proj` (the shape the priest heal uses, units.js:251), so the unit renderer draws it and `onRemove` frees it |
| `sfx(name,e)` | `AudioSys[name]&&AudioSys[name]()` | `AudioSys` has `swing, telegraph, loose, thok, sting, slam, voice, hit, pop, hop...` | **the guide's lookup plays 1 of 7 tokens.** v30 emits `wind, swing, heavy, throw, arrow, rally, ult` *(counted)*, and only `swing` is an `AudioSys` key. Map them: <br>• `wind`→`AudioSys.voice('wind',e._foe)` <br>• `swing`→`AudioSys.swing(false)` <br>• `heavy`→`AudioSys.swing(true)` <br>• `throw`→`AudioSys.loose(false)` <br>• `arrow`→`AudioSys.thok(x,z)` (patch V2) <br>• `rally`→`AudioSys.sting()` <br>• `ult`→`AudioSys.slam()` |
| `onRemove(e)` | — | — | release the unit's pooled rig. Clear `M.lock`, `P.lockE`, `KC.e`, `P.riposte` if they point at it. Splice it from `enemies` if it was mirrored there. Drop its `M.hvuTele` entries. Called for projectiles too. |
| `fx.puff(x,z,s)` | `puff(dust,x,z,s)` | `puff` exists but is `puff(p,x,z,y,vx,vz,vy,life,size,color)` (15-effects.js:77) | the guide's call puts `s` into `y`. Loop `inkNum(4,2)` × `puff(dust,x,z,0.1,cos*sp,sin*sp,rnd(.4,1),.35+s*.2,...)` |
| `fx.shock(x,z,r)` | `ringFx(x,z,r*0.4,r,0.35,0x17171a)` | `ringFx(x,z,s0,s1,life,color,hold,driver)` 15-effects.js:106 | signature right, colour wrong. Use `ringFx(x,z,r*.3,r*1.9,.32,0x14151c)` + `shockDecor(x,z,r*1.4,..)` when `r>=1.2` + `trauma+=min(.18,.06r)*SHK()`. **31 calls**: without `SHK()` the camera setting is ignored. |
| `fx.mark(e,text,life)` | `shout:` → `M.ticks.push(...)` | `M.ticks` exists; `tickTxt(text,x,z,life,big)` 20b:38 is the idiom | `tickTxt` for `life<1.2`; `shout()` for ult names. Skip `text===''` (the guard clears with `mark(e,'',0)`). **46 calls**: throttle per body. |
| `fx.number(e,dmg,big,crit)` | `dmgNum(e.x,e.z,dmg,big,false,e,false,crit)` | yes, 15-effects.js:344 `dmgNum(x,z,n,heavy,red,tgt,ally,crit,heal,en)` | correct as given. **But `hurt()` does not pass `a.by`**, so the host cannot tell your blow from unit-on-unit damage: set a module-scope `HVU_BY_PLAYER=true` around the host's own `HVU.hurt` call. Unit-on-unit numbers go small or off the page, like the companion's (`ally:true`). |
| `fx.bits(e,color,n)` | `burstBits(e,color,n)` | **does not exist**; the real one is `bit(x,z,vx,vz,vy,size,life,color,y)` (22-companion.js:17) | loop `bit(...)`, `color` = `e.cfg.accent` string (three `.set()` accepts it) |
| `fx.kill(e,a)` | `onEnemyKilled(e,a)` | **does not exist.** The kill is split between `impact()`'s lethal block (20-player.js:~170-186: `countKill`, beat, `killCam`, heal, `ultGain`, drops) and `hurtEnemy`'s death block (19-enemies.js:~214) | `hvuKill(e,a)`. The body half (ring, bits, `decal`, `AudioSys.voice('die')`) always runs. The reward half runs **only when `!a\|\|!a.by`** (your blow). It must be the *only* trigger for unit rewards (§1 step 4). |
| `fx.ult(e,name)` | — | — | set directly: `shout(name)`, `M.title`, rings via `M.hvuTimers`, `hitstop`/`trauma` gated on involvement (the unit within ~10 tiles of `P`, or targeting `P`) |
| `fx.rite(e,label)` | — | — | **new in v30, called by `rite()` (units.js:372)** for LEVEL milestones, TIER n, EVOLVED, ASCENDED. Out of combat it is the full beat: `killCam(e)`-style hold, `shout(label)`, staggered `ringFx`, accent `bit`s. In combat it is a chip stamp + `tickTxt` only, with **no hitstop** (§6). |
| `fx.spinFx(e)` | — | — | `ringFx(e.x,e.z,.6,2.5,.25,0x14151c)` |

**Score: 5 named functions don't exist** (`hurtPlayerFromUnit`, `decorCircles`, `M.telegraphs`, `burstBits`, `onEnemyKilled`), plus `M.timers`.
- **2 have the wrong signature or colour** (`puff`, `shock`).
- **1 silently plays nothing** (`sfx`).
- **2 hooks are unreachable through `attach`** (`rite`, `ult`).

### Step 3 — "Render: `HVU.render.install(THREE,scene,camera); HVU.render.load(startGame); HVU.render.sync()`"

**Must not be used.** See §4. Also:
- `render.load(startGame)` would call `startGame()` (27-hud.js:233) as soon as 478 textures decode, **skipping the title screen**.
- The game's boot is its own `async boot()` (28-boot-frame.js:51), which enables `#play` when ready.

The guide's own alternative, "draw from `HVU.drawInfo(e)` / `HVU.projInfo(p)` with HV's own sprite quads", is the route.

### Step 4 — "Sim: `HVU.tick(1/120)` in the fixed step, after the slimes. The hero hits units with `HVU.hurt(u,{...by:null})` for every `u` with `u.team!==1`."

**Mostly right. Three corrections.**

1. **Where and with what dt.** In `frame()` (28-boot-frame.js:181) the step is `playerSim(sdt); ultStep; waveStep; pickupStep; for(e of enemies)enemySim(e, inDomain?sdt*0.42:sdt); companionSim; arrowSim; spitSim; portalSim`.
   - Insert `hvuStep(sdt)` right after the `enemySim` loop.
   - It must take **`sdt`** (`T.DT*hsK`, the hitstop-held step), not a constant `1/120`, or units keep moving through every hitstop.
   - `hvuStep` first steps `M.hvuTimers` and `M.hvuTele` and decays `P.markedT`, then calls `HVU.tick(sdt)`.
   - `HVU.tick` has one global dt, so the domain's 0.42 crawl (`inDomain`) cannot be applied per unit. Write `u.slowT=0.1` on hostile units inside the domain each step: v30 reads `slowT` as ×0.7 speed. A true crawl needs patch V1's per-unit time factor.
2. **Hitting units through the game's hit pipeline, not a bare `HVU.hurt`.** A bare `HVU.hurt` skips the whole feel package in `impact()` (20-player.js:130): crit ladder, style, hitstop scaling, camPunch, sparks, `vfxHit`, `beat`, the kill featured/killCam logic. So:
   - Units go through `impact(u,a,nx,nz,ranged)`.
   - `hurtEnemy` (19-enemies.js:179) gets a first-line dispatch: `if(e.hvu){ hvuHurt(e,a,ax,az); return; }`.
   - `hvuHurt` calls `HVU_BY_PLAYER=true; HVU.hurt(e,{dmg:a.dmg,ax,az,knock:a.knock,big:a.big,crit:a.crit,by:null,stun:a.launch?1:0}); HVU_BY_PLAYER=false` and writes `e.freezeT=a.big?.12:.06` (patch V1).
   - **`impact()`'s lethal block predicts death as `e.hp-a.dmg<=0` before the blow lands.** v30 can cancel a lethal blow through `WARD`, `GUARD`, `AEGIS`, `SECOND WIND`, `IRON WILL`, `BONE DEEP` rise-again (35% for skel) and UNIQUE rise. So for `e.hvu` the lethal block must be skipped and run from `fx.kill` instead: extract it verbatim into `onKill(e,nx,nz,ranged)`. This was blocker B4 in the v9 host map and **is still valid in v30** (verified: `die()` returns early on rise, units.js:1029).
3. **The team test.** `u.team!==1` is right for the hero's swing: owned units are team 1, and a charmed owned unit becomes team 2 and is hittable. For the game's other ~30 hit sites see phase 3.

### Step 5 — "Builds: `HVU.spawnFrom(build,x,z,{team:0})`... the tier rises by itself at levels 3, 5, 7 and 10"

- **`{team:0}` "works" by accident.** `spawn` writes `team:(opts&&opts.team)||1`, and `0||1` is 1 *(measured: `spawn('watch',0,0,{team:0}).team === 1`)*. Write `{team:1}` explicitly.
- **The tier sentence is stale.** The same paragraph later says tiers do *not* rise by themselves. See §2.
- **The build shape in step 5 is stale.** It omits `level, xp, renown, ascended, skills`, which `serialize` does write. See §7.

### Step 6 — "Waves: `HVU.CFG[kind].unlock` is on the same scale as `T.SLIME.unlock`. Spawn with `HVU.spawn(kind,x,z,{awake:true})` from `rosterFor`."

- **Unlock scale: true.** `T.SLIME.unlock` is a wave number (2/4/5), and CFG unlocks run 1-12 (99 for `mirrorimage`) *(counted)*.
- **`rosterFor(n)` returns `WAVE.pick`, a `Set` of pooled game enemy *bodies*** (21-waves.js:85), not kinds. The kind list is `composeWave(n)` over `ECOST`/`EMAX`, keyed on `T.SLIME`. Units need their own `ECOST` entry: use `HVU.power(cfg)/k`.
- **`HVU.spawn(kind,x,z,{awake:true})` spawns an ALLY at FULL POWER:**
  - the default team is **1** (the player's side);
  - the default tier is **5** *(measured: `spawn('orc',1,1,{}).tier === 5`)*, with passive, ult and smart AI.
  - Enemies must be `HVU.spawnFrom({kind,tier:tierFor(n),level:levelFor(n)},x,z,{team:2,awake:true})`, as the demo's own `spawnKind` does for team 2 (units.js:1901).

### The five vendoring patches (the only edits inside the module)

| id | where | edit | why |
|---|---|---|---|
| V1 | `tick()` units.js:1611 | at the top of the per-unit loop: `if(e.freezeT>0){e.freezeT-=dt;continue;}` | the game's per-body hit freeze (`e.freeze` in `enemySim`) and perfect-dodge freeze have no channel in v30 |
| V2 | units.js:1126 | `HVU.host.sfx('arrow',p.owner,p.x,p.z)` | the thunk should pan at the arrow, not the archer |
| V3 | `update()` units.js:1316 | nothing; in the host instead: `P.markedT` decays in `hvuStep` | v30 decrements `e.markedT` for units only; DEATH MARK on the player (`t.markedT=6`) never ends |
| V4 | wander branch units.js:1346 | `const h=HVU.host.home&&HVU.host.home(e); if(h){e.wx=h[0]+..;e.wz=h[1]+..}` | owned units have no follow or leash at all in v30 *(grep: no leash/follow/order field)*; orders need a home point |
| V5 | return object units.js:1625 | export `PATHS, setPath` **or** leave PATHS out of the design | `setPath` (units.js:324) is never called and not exported, and `serialize` doesn't write `path`. **PATHS are dead code in v30.** |

---

## 2. The brief's stale-guide pitfalls: confirmed or refuted

| pitfall | verdict | evidence |
|---|---|---|
| **Team 0 vs team 1 for owned units** | **Confirmed stale, and worse than the brief says.** | 1. The header (units.js:14) says "default 1; the player is team 0". Step 5 says `{team:0}`. Step 5's last line says "Hallokin is team 1". <br>2. The code: `grantXp` returns unless `e.team===1` (units.js:373). `die()` grants only when `a.by.team===1&&e.team!==1` (units.js:1027). The demo host sets `P.team=1` (units.js ~1826). <br>3. `{team:0}` is coerced to 1 by `\|\|1`, so the step-5 call accidentally works. <br>4. **The real trap is the player:** `playerT()` (units.js:860) writes `P.team=0` when `P.team` is undefined, and the game's `P` has no `team`. Owned units (team 1) then see the hero as *another team*: they **target him and their swings land on him** (`dealHit`: `P.team!==team`). <br>**Rule:** the host sets `P.team=1` every call, or `-1` while dead, travelling or on the title. |
| **Tier levels "3,5,7,10"** | **Confirmed stale.** | `TIER_AT_LEVEL={5:2,12:3,20:4,30:5}`, `MAX_LEVEL=50` (units.js:349). Tiers never rise by themselves: `grantXp` only flags `e.tierReady` and marks `READY TO ADVANCE`, and `advance()` = `advanceTier` (units.js:371) takes it with `rite(e,'TIER n')`. Cumulative XP *(measured)*: L5 174, L12 1,587, L20 5,491, L30 14,528, L50 49,028. XP per kill = `max(6, power/8)`: 6 for watch/skel/orc, median 10, 77 for abysslord. **A tier-3 unit is ~160 median kills of its own**, which matters for pacing (a design lens question, flagged here). |
| **"Twenty seven enemies"** | **Confirmed stale.** | *(measured)* **88 CFG kinds, 3 hidden** (`mirrorimage`, `blob`, `slimelet`). 10 groups: astra 3, watch 4, orcs 6, men 5, undead 11, casters 8, beasts 13, demons 16, shadows 7, custom 15. **53 kinds have an evolution.** 478 sheets over 74 sheet prefixes. (The brief's "48 named kinds" undercounts too.) |
| **Spawn defaults (not in the brief)** | **New pitfall.** | `spawn()` defaults to **team 1 and tier 5** *(measured)*. Any wave code that calls `HVU.spawn(kind,x,z,{awake:true})` as step 6 says makes full-power allies. |
| **PATHS (brief item 2)** | **New pitfall: dead in v30.** | `PATHS`, `pathsFor` and `setPath` exist (units.js:315-326), but `setPath` has no caller, is not exported, and `path` is not in `serialize`. The demo's Lab keeps `path:null` in its build. Do not design on paths unless patch V5 exports them. |
| **`attach` is lossy** | **New pitfall** (v9 map §0 said this of `ult`; v30 adds `rite`). | units.js:1609-1611 |
| **Items on load are not slot-validated** | **New pitfall.** | `spawnFrom` passes `b.items` straight into `spawn` opts (units.js:392); it never goes through `equip_`'s slot limit. Unknown ids are ignored by `applyMods` but kept in `e.items` and re-serialized. |
| **Sheet-key collisions with the game (not in the brief)** | **New blocker for any "merge into SHEETS" plan.** | *(measured)* `knightIdle`, `knightWalk`, `knightDeath` and `archerIdle` exist in both `HV_SPRITES.sheets` (19 keys) and `HVU_SHEETS`. Unprefixed, the unit knight overwrites the hero. Every unit sheet enters the game's tables as `'u.'+key` (§8). |

### What v30 (and the 2026-09-13 hires/render1 passes) make stale in the v9 audits

**`audit/HVU-map-host.md`** (still ~80% right):
- §1 `P.team = ... ? -1 : 0` → **must be 1** (above).
- §0 "fx.mark 33 calls" → **46**. fx calls total **108** *(counted: mark 46, shock 31, puff 19, bits 6, number 2, kill/rite/ult/spinFx 1 each)*.
- B5 (the demo's broken `fx.number` signature) → **fixed in v30.** The demo FX `number(x,z,...)` is still wrong, but the module and guide use `(e,dmg,big,crit)` and the game's `dmgNum` matches. Irrelevant, since the demo host is not copied.
- B2 (`P.markedT` never decays) → **still true.** v30 now ticks `P.poisonT` itself (units.js:1613) but not `markedT`.
- §12 sfx table → the token set is now `wind/swing/heavy/throw/arrow/rally/ult`. The v9 table holds, minus `summon`.
- New in v30 and absent from the map: `fx.rite`, `grantXp`/levels, `evolve` (which calls `onRemove` + `spawnFrom`, i.e. **a unit's object identity changes on evolve**: every reference the game holds, including `M.lock`, `P.lockE` and the army roster, must be re-pointed; `evolve` returns the new body), `mirrorimage` / `slimelet` / grave-raise spawns mid-tick.
- B4 (`impact()` lethal prediction), B3 (`perfectDodge` shim), B7 (`M.hvuTimers` in the literal): **still valid.**

**`audit/HVU-map-render.md`** (the decision stands; most of its costs are gone):
- §5/§8 **bake cost and CPU canvas memory are obsolete.** Since REPORT-hires, boot (28-boot-frame.js:55) bakes *nothing texel-sized* on the page:
  - `hatchSheet` with alpha 0 returns the art;
  - `bodyStack` is the art three times;
  - the line is `inkLine()`'s screen-space shader (12-renderer-textures.js:134-160).
  
  The 1.67 s / 102 MB estimate for staged `ringStack/outlineStack/haloSheet` bakes no longer applies. Only SKETCH OFF still needs `outlineStack` + `haloSheet`, and those can be made lazily.
- §6 "ink line = `outl[0]` + `sheetOutl` (`ringStack(...)`)" → now `outl[0]` = `bodyStack` + `inkLine(mode 1)`. The unit rig should use the same material patch. "Never point a ghost at `sheetHalo`" still holds for OFF.
- §2 sheet inventory → *(measured v30)* **478 sheets, 4.80 Mpx, 1,124 KB of base64**. By kind: body 438 / 4.51 Mpx, fx 16, shadow strips 6, projectiles 18. The game's own sheets are 0.67 Mpx.
- **109 sheets carry per-frame pivots `pxs`** (all the slimes, the shadow line, the demons). The v9 map's `SHEET_ANCHOR` per-sheet `{w,h,cx,feet}` is not enough: `cx` must be set **per frame** from `pxs[frame]`, in `show()`.
- §1 reason 5 ("wrong billboard") is **not a real difference.** A camera-quaternion plane and a `THREE.Sprite` both lie in the view plane, including roll. Drop that argument. The other four (sort order, line, boil, on-twos) stand, and v30 adds three more (§4).
- §7 `uFast` for the ANIM clock, §6 x-ray via `occlusionTint`, §8 draw budget: **still valid.**

---

## 3. Where a unit lives in the game's sim

The game touches its foes in ~100 places (`grep -o enemies`: 20-player 14, 20b-abilities 20, 21-waves 11, 25-manga 9, 19b 8, 21u-ult 6, 23-camera 5, 24-render 5...).
- **~30 hit call sites** go through `impact(` / `hurtEnemy(` / `onHit(`.
- **7 direct state writes** (`e.state='stun'|'hurt'|'idle'|'dead'`): 20-player 2, 20b 4, 21u 1.

Two options:

**A. A separate `HVU.units` list plus a foe iterator.** Every loop that should include units is edited to iterate `foes()`. ~40 edits. Safe, but slow to reach full coverage, and every missed loop is a silent "abilities don't hit units" bug.

**B. Hostile units are mirrored INTO `enemies[]`, flagged `e.hvu=true`.** Coverage is automatic. The guard list is short and greppable:
- `enemySim` → `if(e.hvu)return;` (HVU.tick sims it)
- `hurtEnemy` → dispatch to `hvuHurt`
- the enemy loop in `renderUpdate` (24) → skip; units draw in `unitRender`
- `ensurePool`/`applyRoster`/`resetFoe`/`pickBody`/`claimBody` (21) → skip `e.hvu`
- `telDraw` (25-manga.js:626) → null for `e.hvu`; units telegraph through `M.hvuTele`
- `waveCard`/`bossChip` (27) → key off the unit's own `Idle` sheet, not `e.kind+'SlimeIdle'`
- `lastStand`/`vigilHolds` shoves write `e.vx/vz`, which v30 also reads as knockback: compatible
- the 7 state writes. v30's `setState` is just `e.state=s;e.t=0` plus beam cleanup, and the game writes `e.stunT` with `'stun'`, which v30 also reads. **Compatible** except `'idle'` (21-waves, pool only, already skipped) and the guard branch's `src.state='hurt'`: route that through the `_foe` shim.

**Decision: B, from phase 3, behind a field-compatibility probe.** The probe runs every ability, the ULT, the companion's arrows, `lastStand` and `vigilHolds` against a unit, then diffs the unit's fields against an HVU-only run.

Only **team-2** units are mirrored. Owned units never enter `enemies[]`, and a unit charmed into team 1 is skipped by one test at the top of `impact()`: `if(e.hvu&&e.team===1)return`.

v30 splices dead units after their death strip (`units.splice`, units.js:1538, inside `HVU.tick`). `onRemove` splices the same object out of `enemies[]`. That is safe because `hvuStep` runs outside every `enemies` loop.

Owned units carry fields the game has never had:
- `e.level/xp/renown`
- an **object identity that changes on `evolve`** (units.js:311-313 removes the body and returns a new one).

The army roster (§6) therefore holds a stable `uid → current unit object` map, and `evolve` goes through a host wrapper `hvuEvolve(u,into)` that re-points `M.lock`, `P.lockE` and the roster entry.

---

## 4. Rendering: `HVU.render.install` versus the game's sprite path

### 4.1 What `render.install` actually draws (units.js:1567-1606)

Per unit it builds:
- a `THREE.Group`;
- a billboard group copying `camera.quaternion`;
- a `ShaderMaterial` body quad with `transparent:true, depthWrite:true`, `NearestFilter`, renderOrder never set;
- an fx quad **tinted (0.09,0.09,0.1)**;
- a flat shadow-strip quad at alpha 0.34 ink;
- a blob `CircleGeometry` shadow;
- for "custom / advanced / ascended" units, a **solid accent-colour silhouette 1.07× behind the body** (`olm`, `solid:1`).

Textures load eagerly: one `TextureLoader.load` per data URI for **all 478 sheets**, shared per sheet (`TEX[k]`), frame window in the `rect` uniform.

### 4.2 Why it cannot go into the game's current look

Checked against REPORT-hires (native resolution, screen-space line, `SOB_K=0`) and REPORT-render1 (no white rims, fx ordered by `fxOrder`):

1. **It is an outline generator the user just had removed.**
   - The `olm` accent silhouette is a coloured rim 7% wider than the body, drawn behind it. For any unit that is `isAdvanced`, custom-tinted, UNIQUE or ASCENDED (gold `#d4a017` / `#f0e6c0`), it prints a coloured outline. **Every developed army unit gets the "ugly outline" back.**
   - The game's actors instead carry `inkLine()` mode 1: a 1.35 framebuffer-px anti-aliased ring. A `ShaderMaterial` is not a `SpriteMaterial`, so `inkLine`'s `map_fragment` patch never reaches it, and units would have **no line at all**. REPORT-hires decided actors need that line ("without it the brown kittens sink into the green").
2. **It desaturates every body by team.** `drawInfo().tint` multiplies the art by `[1,.72,.72]` for team 2 and `[.86,.92,1]` for team 1 (units.js:1552). Every enemy is pinked and every ally blued, which breaks "full saturated colour".
3. **It inks the effect layers.** The fx quad is tinted near-black. The Tiny RPG swing layers (`*A1Fx`, 16 sheets) become dark smears drawn *over* the body. That is the "black blob on the body" bug class this project already fixed twice (memory: ghosts/smears).
4. **Sort order.**
   - Game actors are `depthWrite:false` with `renderOrder 5000+z*8` (`actorLayer`, 13-sprites.js). Decor is `1000+z*8`, and fx use `fxOrder` (REPORT-render1).
   - v30 quads sit at renderOrder 0 with `depthWrite:true`, so they draw before every decor and actor and then write depth.
   - Result: roofs occlude units while the hero and slimes x-ray through the same roofs (`occlusionTint`). Units also never enter `occRects`/`fxOrder`, so the dash strobe cards render1 just fixed would sort wrongly against them.
5. **No boil, no on-twos, no fog.**
   - It advances `e.frame` smoothly at the sheet fps instead of committing on the `ANIM` beat (`commitPose`, 24-render.js:18).
   - Its `ShaderMaterial` has `fog:false`, so units never fade with `scene.fog`, which the districts and the FOG modifier drive (`applyFog`, 21-waves.js).
6. **It ignores SKETCH OFF / LINES / FULL** (`refreshSketch` walks `SPRITES`), context loss (28-boot-frame.js:201 re-flags `SPRITES` clones only), the hit-flash grammar (`outl[0]` going red, the two-frame `halo[0]` paper silhouette), and `occlusionTint` x-ray.
7. **Shadows.** The drawn shadow strips plus blob circle sit beside the game's single hard ink ellipse on the `BOIL` clock (`o.shadow`, `scribTex`). Two shadow languages on one page.

### 4.3 Decision: units draw through the game's rig, from `HVU.drawInfo(e)` and `HVU.projInfo(p)`

Keep the module's `render` object (it is inert unless installed) and never call it.

The work, in two steps so phase 0 stays small.

**Step R1 (phase 0): per-sheet and per-frame geometry on the existing rig.** In `13-sprites.js`:
- `SHEET_ANCHOR` gains `{w,h,cx,feet}` in tiles for unit sheets:
  - `ppu = META[prefix].ppu/(HVU.SCALE*(cfg.scale||1)*(e.scaleMod||1))`, the same expression as `drawInfo`
  - `w = fw/ppu`, `h = fh/ppu`, `feet = 1-py/fh`
- `play(key)` applies `w/h` through `setScale`.
- `show(idx)` takes `cx` from `SH[key].pxs[idx]/fw` when present (109 sheets).
- The unit renderer (new `unitRender(rdt)`, called from `renderUpdate` after the enemy loop) does, per unit:
  1. `d=HVU.drawInfo(e)`
  2. `spr.play(d.sheet)`, then `show(d.frame)` **only on `ANIM.step`** (HVU owns `e.frame` for hitboxes; the game owns when it is drawn)
  3. `spr.flip=d.dir<0` (xor `d.faceLeft`)
  4. `setZ(e.z)`, `occlusionTint`, `silTint`, `boil(OV)`
  5. `commitPose(spr,1,1,d.tilt,d.hy,force)`
  6. `spr.flash.material.opacity=d.flash`
  7. `outl[0].material.color` red on hit
  8. **ignore `d.tint`'s team factor**. Use `e.customTint||e.cfg.tint||null` only.
  9. **ignore `d.custom/d.accent`**: no silhouette rim.
- Flag `uFast` so `animClock` runs at 24 Hz while any unit is in `attack|dash|leap|volley` (v9 render map §7, still right).
- Unit sheets go through the game's existing per-rig `cloneTex` path, so phase 0 needs no material work. One family (watch, 13 sheets) × 4 rigs is trivially within budget.
- **Fx layers:** one lazy extra sprite per rig drawn in the **art's own colour** (no ink tint), mode 0, `renderOrder = sp.renderOrder+0.1`. **Shadow strips:** dropped (6 sheets, ~1 KB); the rig's `o.shadow` ellipse sizes itself from `bodyBox`.
- **Projectiles** (18 sheets incl. `beamFx`, `darkboltFx`, `spit*`, `cannonball`): a pool of 24 rigs, `material.rotation=p.rot`, placed through `fxOrder(x,z,...)`.

**Step R2 (phase 2): the shared-texture rig, v30's good idea in the game's material.** Today every rig clones every sheet it plays, three times:
- `tex` for colour;
- `otex`, three rows, now identical copies since `bodyStack`;
- `htex`.

That is 5A texels per rig per sheet. With 30 bodies across 10 families it is the texture-leak surface PERF-1 capped with an MRU.

The shared rig works like this:
- `sheetTex[k]` is **one** GPU texture per sheet, never cloned.
- `inkLine(mat,mode)` gains a per-material `uRect` (vec4) uniform. `onBeforeCompile` replaces the sprite vertex shader's `vUv = (uvTransform*vec3(uv,1)).xy` with `vUv = uRect.xy + uv*uRect.zw` (flip = negative `z`). The frame-window clamp in `INKLINE_GLSL` reads `uRect` instead of `uvTransform`. The same program serves every material, since `customProgramCacheKey` is already `'hv-inkline'`.
- The body sprite is mode 0 and the line sprite mode 1, **sampling the same texture**. Boil row offsets vanish, because the three `bodyStack` rows are identical since HIRES. `boil(v)` becomes a no-op for the page and stays live only for SKETCH OFF.
- SKETCH OFF keeps the legacy `outlineStack`/`haloSheet`, baked **lazily** for unit sheets only when `SET.sketch===0`, in `refreshSketch`.

Memory, *(measured pixel area)*:
- all 478 sheets resident, shared = 4.80 Mpx × 4 B = **19.2 MB VRAM** once;
- one family = 0.1-1.2 MB;
- decoded `Image`s mirror that in CPU memory;
- no canvas bakes on the page.

Versus per-rig clones: 30 rigs × ~8 sheets × 5A ≈ 30 × 8 × 5 × 10.7 kpx × 4 B ≈ **51 MB**, with churn. The shared rig is also what v1 plan phase 0 step 1 asked for, and it retires PERF-1's MRU for units.

**Gate for R2:** `__HV.texCensus().gl` flat across 20 roster sweeps over 10 families, and a 1:1 2x crop of a unit next to the knight showing the same line weight and no rim (`mk-hires-sheet.py` BEFORE|AFTER).

### 4.4 What the look loses by not using `render.install`, and why that's fine

- **The drawn shadow strips** (death and leap shadows animate): 6 sheets. The page's hard ellipse is the shadow language.
- **The accent silhouette as an "advanced" read.** Replace it with a non-outline read: the army chip's gold ring (§6), and a thin accent tick under owned units' feet, drawn on the ground like the game's shadow, not around the sprite.
- **Team tints.** Replace them with the ground tick (owned units only). Enemies need no tint: they are the things attacking you.

---

## 5. Asset weight and delivery

### 5.1 The measured pieces

| piece | raw | gzip | note |
|---|---|---|---|
| the game today (`hollow-vigil (2).html`) | 927,028 B | 394,757 B | fonts already inlined (01-style.css: "fonts.googleapis.com is GONE"); **three.js is the only network dependency** (`03-sprites.html`: cdnjs r128) |
| HVU module (units.js 60-1638) | 202,984 B | 60,389 B | includes the unused `render` object (~7 KB, can stay) |
| `HVU_META` | 3,073 B | ~1 KB | |
| `HVU_SHEETS` (line 115) | 1,333,613 B | 881,211 B | 478 sheets. Base64 PNG does not compress (0.66). 1,124 KB is image data; the rest is per-sheet metadata incl. 109 `pxs` arrays |
| Three.js r128 inlined (line 108) | 603,352 B | 149,126 B | verified the same revision the game loads (`const e="128"`) |
| demo sandbox + UI (1639-2132 + head.html) | 78 KB + 15 KB | — | **not shipped** |

Per family, as JSON *(measured)*:
- watch 17 KB, skel 14 KB, orc 16 KB, knight 22 KB, lancer 110 KB (the heaviest single prefix).
- By group: demons ~290 KB, custom ~211 KB, astra ~197 KB, undead ~125 KB, beasts ~107 KB, shadows ~107 KB, men ~96 KB, orcs ~85 KB, casters ~76 KB, watch ~45 KB (sheet-prefix sums; custom kinds borrow bodies, so groups overlap slightly).
- Prunable with no loss: the 6 shadow strips and 6 `slime_old` sheets (no CFG kind uses that prefix), 12 KB.

### 5.2 Decisions

1. **The game stays one HTML file.**
   - The user plays a downloaded file (`/workspaces/game/hollow-vigil (2).html`). One file is the delivery format that already works, and nothing in v30 needs a server.
   - The v1 plan's "multi-file with a service worker once a second region's art arrives" stays a *later* option, with the `--inline` build kept.
2. **Adopt the inlined Three.js r128 and drop the CDN.**
   - +603 KB raw / +149 KB gzip buys offline play and no cold-start fetch, and removes the only failure mode where the game shows "load failed" with no network.
   - The game already chose this for fonts. v30 already proves r128 inline works with this exact code.
   - Implementation: `03t-three.html` replaces the `<script src=cdnjs...>` line. `build.sh --cdn` keeps a light build for sharing.
   - Check before shipping: the inlined bytes are the stock `three.min.js` r128 (the license header and `REVISION "128"` match; hash-compare against a cdnjs copy when network is available).
3. **Sheets ship as per-family JSON blocks, decoded lazily.**
   - `03u-units.html` holds `window.HVU_META` plus one `<script type="application/json" id="hvu-fam-<group>">` per group, holding that group's sheets.
   - JSON script blocks are **not parsed or decoded until asked**: the browser only tokenises them as text. A 1.3 MB literal inside a normal `<script>` costs a parse at load, and v30's `render.load` decodes all 478 images at boot.
   - `window.HVU_SHEETS` starts as metadata only (`n, fw, fh, px, py, pxs, fps, loop`, *without* `d`), because the module's sim reads `SH[k].n/fps/loop/fw/fh` for animation and hitbox shapes (`frameShape`) whether or not an image exists. **The sim must run for a kind before its art has decoded** (a spawn in the gap), so the metadata is always present and only `d` is lazy.
   - Loader, new in `13u-unit-sprites.js`:
     - `hvuNeedFamily(group)` parses the block, sets `SH[k].d`, fires `loadImage(d)` for its sheets (≤ 24 in flight), and on decode makes `sheetTex[k]` (shared, R2) or registers for `cloneTex` (R1).
     - `hvuFamilyReady(group)` is the gate.
     - **Callers:** `boot()` for the starter family; `waveStep`'s gap for the next wave's families (`composeWave(n+1)` is deterministic under `wseed`); the pause page's EVOLVE tab for target kinds' portraits; save load for owned units' kinds.
     - **Never decode on a spawn frame.** Same rule as `warmSheets`.
   - v30's own `render.load` is not used.
4. **Prune at build time, not by hand.** `build.sh` drops sheets no visible CFG kind, ATK projectile or fx references: the 12 KB above, plus any family a build excludes (`--families=watch,undead`).

### 5.3 The byte budget

| build | contents | raw | gzip |
|---|---|---|---|
| today | game | 927 KB | 395 KB |
| **phase 0** | game + module + META + **watch family only**; three still CDN (one variable at a time) | ≈ 1,150 KB | ≈ 470 KB |
| phase 2 | + three inline + 3 families (watch, undead, orcs ≈ 255 KB) | ≈ 2,010 KB | ≈ 800 KB |
| **full** | + all families, pruned | **≈ 3,050 KB** | **≈ 1,470 KB** |
| ceiling | alarm if a single-file build passes this | 3,500 KB | 1,700 KB |

Load-time budget, to be checked on a real integrated-GPU laptop (the harness's SwiftShader timings are meaningless, per the project memory):
- the first frame of the title is no later than today's;
- decoding one family takes < 150 ms, spread one image per frame on the title and in gaps.

---

## 6. The army UI: what each demo panel becomes

### 6.1 The rule

The demo UI (head.html + units.js 1840-1983) is a **tool for building and inspecting any unit at any moment**:
- a topbar with team/tier/level spinners, BATTLE and GHOST;
- a dock (UNITS, LAB, ROSTER, SQUAD, WAVE, CLEAR, HITBOXES, SLOW);
- a 320 px drawer of every body;
- a 440 px Lab;
- a 470 px Roster;
- a 640 px six-tab Sheet refreshed by a 200 ms `setInterval` that rebuilds `innerHTML`;
- `T L P R TITLE` text over every owned unit (demo ink pass, units.js:2116);
- team ellipses under every body;
- a HUD bar per unit whenever wounded.

It is drawn in cream paper panels with `box-shadow:5px 5px 0` and a `repeating-linear-gradient` paper stripe. That is the look the user has rejected three times.

The game's idiom, from 27-hud.js:
- **ink-bordered DOM plates** that boil on the world's beat (`sketchBorder` → `--bd0..2` border-image, `body[data-boil]`);
- **`makeBar`** SVG bars with a ragged brush edge and ghost trail;
- **`makeDial`** rings;
- **`FONT_POW`** (Bangers) for numerals and stamps, **`FONT_HAND`** (Patrick Hand SC) for text;
- **`uiToast`** (queue of 3, 1.6 s);
- **`bossChip`**: an 88 px canvas portrait with an `inkCircle` ring;
- **`#cards`**: three between-wave choice cards on keys 1-3;
- **the pause plate**: `.opt` rows, `PM` cursor, hold-to-confirm;
- **`HUD_RECTS`/`hudRects()`**, which keep ink callouts off the chrome;
- **`body.cine`**, which clears the HUD for casts, travel, death and letterbox.

**The rule: combat shows *state*, never *editing*. Editing happens only at safe moments. Everything reuses those parts, filled with each unit's `cfg.accent` at full saturation, not cream.**

### 6.2 In combat: what the player sees

| need | demo | game |
|---|---|---|
| who is in my army, their health | roster panel / per-unit HUD bars / text over heads | **The army strip** `#army` under `#hud`'s three stat rows (top-left, inside `HUD_RECTS[0]`'s column; extend that rect's `h`). ≤ 4 **chips** plus a `+N` chip. Each chip is: <br>• a 46 px canvas portrait: `Idle` frame 0 in full colour, `imageSmoothingEnabled=false`, drawn the `bossChip` way **without** its multiply-tint pass <br>• an `inkCircle` ring ×3 boil variants stroked in `cfg.accent` (or gold for UNIQUE/ASCENDED, the demo's `titleOf` colours) <br>• a flat `makeBar(host,46,6,'#4d86c2','#2b4d73',seed,true)` HP bar, using the armour-blue family so allies never read as the player's red HP <br>• the level in `FONT_POW` bottom-right <br>A kneeling or dead unit's chip drops to 40% opacity with a struck line (the combo-break idiom). |
| an owned unit's health in the world | a bar over every wounded body, always | **The game's existing wounded-bar rule** (25-manga.js:576): a 26×4 ink bar only for 3 s after a hit (`dmgT`), extended to owned units in the ally blue, and always shown under 35%. **No names, no T/L/P numbers over bodies.** |
| which bodies are mine | team ellipses under all, blue tint | A short **accent-colour ground tick** under owned units' feet, drawn by the ink layer at the feet point like the guard marks (25-manga.js:637). **Not a sprite rim.** Hidden while the unit is behind decor (`o.behind`). |
| orders | none (demo has SQUAD/BATTLE buttons) | **One order dial** in `#slot`, beside DASH/ULT, via `makeDial`: **FOLLOW → HOLD → CHARGE** on **V**. It needs patch V4's `host.home(e)`. <br>• FOLLOW: `home` = the player's position; units re-home when more than 6 tiles away and not engaged. <br>• HOLD: `home` = where V was pressed. <br>• CHARGE: `u.hunt=true` for 8 s, then back to FOLLOW. <br>The dial fills while CHARGE runs. The pad binding is to be confirmed: D-pad is used only by `pauseNav` today. |
| level up | `mark('LEVEL n')` + bits + log | `fx.mark` → `tickTxt('LEVEL n',...,big)` over the unit, **throttled to one level tick per 1.2 s across the army**. The rest fold into `kick(chip,'pop')` (the `pulse` queue, one reflow per frame). |
| tier earned | `mark('READY TO ADVANCE')` + shock | **The chip grows a gold Bangers "▲".** One `uiToast('BRAMBLE CAN ADVANCE')` per unit per tier. No world text beyond the first mark. |
| milestone rite mid-fight (VETERAN at 10...) | `fx.rite`: `hitstop(0.25)`, trauma .35, rings, halo, shout | **In combat: no hitstop, no camera.** `tickTxt(name,big)` + a chip stamp (`wcStamp`'s red-stamp animation on the chip). The full rite is queued and replays in the next gap. This is the involvement gate of v1 §8.4 applied to your own army: a unit's milestone must never freeze *your* swing. |
| ally ULT | `fx.ult`: page-level shout, hitstop .16, trauma .45 | `shout(name)` at the unit, rings via `M.hvuTimers`, **no hitstop unless the ult hits something within 4 tiles of the player**. Never `beat('ult')` or `panel('ult')`: those stay the player's domain. |
| enemy unit reads | red egg/wedge/capsule on the ink canvas with diagonal hatch | `M.hvuTele` drawn in the game's telegraph idiom: `wpath` dashes plus the closing arc turning `#ff3a2a` after 0.7, **no hatch fill** (v9 host map §9, still right). Enemy `cfg.boss` units use `#bossBar` with `bossChip` keyed on `u.sheet+'Idle'`. `HVU.titleOf(u)` (ELITE/CHAMPION/…) prints as the `wcStamp` under the name. |

Budget check against "easily seeable":
- The strip adds **one column of ≤ 4 small portraits and one dial.**
- The world adds nothing permanent except a foot tick on owned units.
- The per-body text layer and team ellipses of the demo are gone.
- `HUD_RECTS` gains an `'army'` rect so balloons, POWs and numbers are placed away from it (`placeAway`).
- `body.cine` hides the strip with the rest of `#hud`.

### 6.3 Out of combat: rites, the sheet, evolution

**Rites belong to the wave gap.**
- When `WAVE.gap>0` and any owned unit has `tierReady`, `#cards` (21-waves `offerCards` / 27 `cardsStep2`) shows a **fourth card, "RITE"**, on key **4** beside the three perk cards.
- Taking it plays `HVU.advance(u)` for the first ready unit. The game's `fx.rite` then runs the full beat:
  - `killCam(u,..)`-style hold;
  - `slowReq` breath;
  - staggered `ringFx` in accent;
  - accent `bit` fountains;
  - `shout('TIER 3')`, with the unit's label in `FONT_POW` as the card stamp.
- The perk card and the rite are independent picks, so the card economy is untouched.
- Queued mid-fight milestone rites replay here too, one per 1.6 s.

**The Sheet becomes a pause page: ARMY.**
- The pause plate (`#pause .plate`, `PM` rows, arrows plus hold-J confirm, 27-hud.js:152-186) gains a page switch: **Q / E** while paused. The sim is frozen then, but both still set `pressed.ult` / `pressed.skill`, so `pauseNav` must consume them the way it already consumes `pressed.atk`/`pressed.dash` (27-hud.js:180), or a page flip fires the ULT on resume, **LB / RB** on pad.
- The page is chips down the left and the selected unit on the right. The demo's six tabs fold to four:
  - **OVERVIEW.** Portrait (the same canvas at 96 px, accent `inkCircle`). Name; kind; tier / level. HP and XP as `makeBar`s. POWER in Bangers with the `titleOf` stamp. The moves as rows with the demo's glyphs (`GLYPH` swing/thrust/shot/strike/heal/summon/beam) and `HVU.describe(id)`. Passive and ult sentences (`cfg.passive`, `cfg.ult.desc`). "Next tier at level N brings …" (the demo's `nextBits`).
  - **GROW.** `HVU.STATS` as 8-pip rows (budget `statBudget-spent`, cap `STAT_CAP`). Skills: rank pips from `kitOf(cfg).moves` filtered by `tierNeeded`, with the rank-3 mastery choice from `MASTERY[moveType(id)]`. `PRESETS` as one "quick build" row.
  - **TALENT & GEAR.** `TALENTS[tier]` choice cards for each earned tier ≥ 2. Gear slots (artifact ×1-2, weapon, trinket) from `ITEMS`, **equipped through `HVU.equip`** (validates slots, §7).
  - **EVOLVE.** One card per `evolutionsFor(kind)`. The target's portrait decodes its family lazily (`hvuNeedFamily`): a `?` until ready, then the game's ink-in. Each card shows name, `HVU.power(CFG[into])` against `unitPower(u)`, the ult name, the required tier, and a hold-J **EVOLVE**. It routes through `hvuEvolve` (§3: object identity changes) and plays the `EVOLVED` rite on resume.
- **Tab badges** ("3" points to spend) are the demo's `.tab b`, in Bangers red.
- **Editable only when safe:** `WAVE.gap>0`, or `engagedCount()===0` (21u-ult.js:11) and no hostile unit within 12 tiles. Otherwise the page renders read-only, like the demo's enemy mode (`isPlayer` false disables buttons).
- Pausing mid-fight lets you *read* your army, never respec it under a swing.
- **Render on change, not on a timer.** The demo's 200 ms `setInterval` + `innerHTML` rebuild would fight `PM`'s row cursor. Rebuild on `sheetKey(u)` change (the demo's own key: level, tier, tierReady, budgets, items, kind) checked in `hudTick`.
- **Label edit:** keyboard only, a 16-character inline field (the demo's `setLook(u,label,tint)` limit). The colour picker (`<input type=color>`) is dropped: **tint is chosen from the family's accent palette** (6 swatches) so no player can paint a unit muddy.

### 6.4 The rest of the demo UI

| demo | fate |
|---|---|
| **Lab** (build a unit before it spawns, presets, tier radio, loadout checkboxes, COPY/PASTE BUILD JSON) | **Dev-only.** It mounts as a fifth ARMY tab when the URL has `?lab`, and as `__HV.hvu.lab` for probes. The player-facing parts already live in GROW (presets) and TALENT & GEAR. The bench-a-move `setLoadout` is dropped from the player UI: a benched move is a hidden nerf the player will forget. |
| **Roster** (every CFG body: power, hp, moves, passive, ult, "evolves into") | Becomes the **v1 Roster pages**: title-screen marks wall now, the Barracks later. It shows only SEEN kinds, and its "evolves into X at T4" line is the hook that links studying a kind to owning one. **POWER RANK is dev-only.** |
| **Drawer** (spawn any body), **dock**, **topbar** (SPAWN FOR, ENEMIES T/L, BATTLE, GHOST), **#help**, **#log** (already disabled: `LOG.add` returns at once) | **Dev harness only** (`__HV.hvu.spawnKind/battle/ghost`). Nothing ships. |
| **shport** (96 px pixelated square, accent box-shadow) | The chip and sheet portraits above: full colour, `inkCircle` accent ring, no square box, no offset shadow. |
| **evolution previews** (`kindPortrait` from `HVU.render.TEX[...].image`) | EVOLVE cards. They read decoded `Image`s from the family loader, because `HVU.render.TEX` never exists in the game. |
| **Demo keys** E sheet, Tab next tab, F focus, P pause, H hitboxes, T slow | E is SKILL, Tab the controls card, F fullscreen, P pause in the game (18-state-input.js `KEY`). The only new play key is **V** (orders). The sheet is reached through pause. |

---

## 7. The save model: v30's build object inside the plan's save

### 7.1 What `serialize(e)` gives, and what it doesn't

*(measured)* `serialize` of a fresh watch is **157 bytes**:

```
{kind, tier, level, xp, renown, ascended, items[], stats{vigor,might,haste,swift,reach,will}, skills{id:{rank,mastery}}, talents{2..5:id}, loadout[]|null, label, tint}
```

`spawnFrom(b,x,z,opts)` restores it. It validates stats against `statBudget` (level-aware, because `level` is set first), skills against `skillBudget`, talents by tier, and loadout against the kit, then recomputes `ascended`.

Gaps to cover on the game side:

| gap | fix |
|---|---|
| **No identity.** Two identical watches serialize identically. | The army record wraps the build: `{uid, build, order, bond, state:'ready'\|'kneeling'\|'lost', origin, since}`. `uid` is stable across `evolve` (§3). |
| **`path` not saved** (and PATHS dead, §2) | Schema v1 omits it. Adding PATHS later is a schema bump plus a migration defaulting to `null`. |
| **Items not slot-validated on load** (`spawnFrom` → `spawn` opts) | The loader spawns with `items:[]`, then `for(id of b.items) HVU.equip(u,id)` (`equip_` enforces slots and drops unknown ids). |
| **Tier not checked against level.** A `tier:5, level:1` build loads. | The loader clamps `tier ≤ 1 + #{L in TIER_AT_LEVEL : L ≤ level}`. Tampering is irrelevant; migrations and bugs are not. |
| **Unknown kind throws** (`spawn`: `throw new Error('unknown unit')`) | `if(!HVU.CFG[b.kind])` → mark `lost`, keep the record, never throw during load. |
| **`tierReady` not saved** | Derived: after `spawnFrom`, `u.tierReady = HVU.readyTier(u)\|\|0`. Otherwise a unit that earned a tier before saving shows no ▲ until its next kill. |
| **HP, poison, charm, cooldowns, ult-used not saved** | Correct by design: saves happen only at safe moments, and units come back whole. |
| **`label` ≤ 16, `tint` must be `#rrggbb`** | `setLook` already validates. The UI only offers family swatches (§6.3). |

### 7.2 The document and when it is written

- **One versioned JSON document**, as v1 §8.1 says: `{v, hero{...}, army:[record...], roster{...}, ...}`. The army part is ~250 B per unit, so a 12-unit army is ~3 KB.
- **Store:**
  - **Phases 0-4 (inside today's wave game):** `localStorage['hv_army']`, beside the game's existing `hv_best`/`hv_score`/`hv_rank`/`hv_sketch`/settings keys. Same try/catch and clamp discipline (RB-02 in 27-hud.js `persistBest`). Zero new infrastructure.
  - **World phase (v1 "29-save"):** move to **IndexedDB**, key `hv_folio`, one document. Migrate by reading `hv_army` once.
- **When to write: only at safe moments.** These are wave-gap start (`WAVE.gap` rising edge in `waveStep`), after a RITE card or EVOLVE commit, after the ARMY page closes with changes, on death's results card, and `pagehide`/`visibilitychange` **only if** `WAVE.gap>0 || !running`.
  - A save during a live wave is refused, not deferred into the middle of the fight.
  - `serialize` runs on live units that are not in combat, so `hp` exclusion loses nothing.
- **Load:** on `startGame()`, owned units spawn from records next to the player (`freeSpot` around `P`). They are placed with `{team:1, awake:false}` so they don't aggro before the first wave. The army's families are requested with `hvuNeedFamily` in `boot()` from the saved kinds, so portraits and bodies are ready before the title's PLAY enables.
- **Run versus meta.** The wave game's `resetRun()` (21-waves.js:234) wipes run state. The army is meta and survives `resetRun`. Death in a run follows the design lens (kneel / lost). The integration only guarantees that `HVU.clear()` is never called on owned units: `resetRun` clears hostile units (`for u of HVU.units if u.team!==1 → remove`) and keeps team 1.

### 7.3 Probe

`pw-hvu-save.js`:
1. Give a unit level 13 (`grantXp`), stats, skills incl. a mastery, a talent, two items, a label, and an evolve.
2. Save, reload the page, load.
3. Assert `JSON.stringify(HVU.serialize(u))` is byte-equal before and after.
4. Assert `u.tierReady===HVU.readyTier(u)`.
5. Assert a hand-corrupted record (unknown kind, tier 5 at level 1, three artifacts, `stats.might=99`) loads without a throw and is clamped.

---

## 8. The phased integration roadmap

### Build and rollback rules for every phase

- **Build flag, not a fork.**
  - `build.sh` takes `HVU=0|1`. With `HVU=0` it omits `03u`, `03v` and `29-hvu-host.js`, and every game-side hook is written `if(window.HVU){...}`. **An `HVU=0` build must behave byte-for-byte like today.**
  - Gate: the existing `pw-smoke.js` (ok, 0 errors), `pw-reset-probe.js` 19/19 and `pw-set-probe.js` 25/25 pass on both builds.
  - Rollback of any phase = ship `HVU=0`, or `git revert` that phase's section-file commits. The vendored module is its own file, so a bad patch reverts alone.
  - Snapshot `src/` to `hv-work/snapshots/src-pre-hvuN.tgz` before each phase, as the render1/hires passes did.
- **Namespace every unit sheet.** *(measured)* **Four v30 sheet keys collide with the game's own**: `knightIdle`, `knightWalk`, `knightDeath` (the hero) and `archerIdle`. Unit sheets enter the game's tables as `'u.'+key` (`SHEETS['u.knightIdle']`, `sheetTex['u.knightIdle']`). `unitRender` maps `drawInfo().sheet` through `'u.'+`. **The v9 render map's "merge `USH` straight into `SHEETS`" would have replaced the hero's knight with the Tiny RPG knight.**
- **Harness access.** `window.__HV.hvu = {HVU, spawnFoe, spawnOwned, army, step:hvuStep}`.
- **Performance truth** comes only from a real laptop. Headless SwiftShader verifies state, errors and screenshots.
- **Every new line is multi-line-safe:** no trailing `//` comment that can swallow the next statement (seven live bugs so far). Run `audit/swallow-audit.js` on every phase.

### Phase 0 — "One watch in Ashridge": v30 runs inside the game with the game's feel and look

**The smallest proof.** One kind (`watch`, 13 sheets, 17 KB) is spawned hostile, hit by the knight, killed; one owned watch fights beside you and levels.

Files:
1. **`src/03u-units.html`**: `window.HVU_META` (3 KB) + `window.HVU_SHEETS` holding **only** the 13 `watch*` sheets, with `d` inline. There is no lazy loader yet.
2. **`src/03v-hvu-module.html`**: units.js lines 60-1638 in a `<script>`, **unpatched**.
3. **`src/29-hvu-host.js`**:
   - `HVU_HOST` with all 14 members as §1's table, where phase 0 simplifies:
     - `telegraph` pushes to `M.hvuTele` and draws a plain `ringFx` at `t.x,t.z`, radius `t.reach`, for `t.life`: a circle for every shape;
     - `sfx` uses the 7-token table;
     - `fx.rite` / `fx.ult` = `tickTxt` + `ringFx` only;
     - `fx.kill` = the body half only. In phase 0 the reward (`countKill`, heal, `ultGain`) still comes from `impact()`'s predicted-lethal block, so `fx.kill` must not count again.
   - `hvuPlayer()` with **`P.team=1`** (or -1).
   - `hvuStep(sdt)`.
   - `spawnFoe(kind,x,z)` = `HVU.spawnFrom({kind,tier:1,level:1},x,z,{team:2,awake:true})` then `u.hvu=true; u.cfg.drawH=META.bh/ppu`.
   - `spawnOwned(build,x,z)` = `spawnFrom(build,x,z,{team:1})`.
   - `HVU.host=HVU_HOST` in `boot()`.

Edits:
4. **`13-sprites.js`** (R1): `SHEET_ANCHOR['u.'+k]={w,h,cx,feet}`; `play()` applies `w/h`; `show()` takes per-frame `cx` from `pxs`.
5. **`28-boot-frame.js`**:
   - In `boot()`, **before** the `keysList` bake loop (line 53), copy the 13 watch sheets into `SHEETS` as `'u.watch*'`, so the existing loop makes their `sheetTex`/`bodyStack`/`outlineStack`/`haloSheet`. 13 small sheets, a few ms.
   - Allocate 4 unit rigs with `makeSprite(1,1,0,0.5)`, `cap=8`, hidden.
   - In the sim step (line 181), `hvuStep(sdt)` after the `enemySim` loop.
6. **`24-render.js`**: `unitRender(rdt)` after the enemy loop (§4.3 R1 steps 1-9); `uFast` in `animClock`.
7. **`20-player.js`**, blade loop (line ~109): after `for(const e of enemies){...}` add the same loop over `HVU.units` with `u.team!==1 && !u.dead && u._hitBy!==P.atkId`. It calls `onHit(u,ad,nx,nz)`, i.e. `impact()`.
8. **`19-enemies.js` `hurtEnemy`**, first line: `if(e.hvu){ hvuHurt(e,a,ax,az); return; }`.

**Known phase-0 shortcut, stated.**
- `impact()`'s lethal prediction is trusted.
- It is sound only because a **tier-1 watch** has no passive (`noPassive` below T3), no guard (block is a T2 move) and no rise.
- The probe asserts that. Phase 1 removes the shortcut.

**Probe `pw-hvu0.js`** (headless, `__HV` + `__HV.hvu`):

1. Boot, `startGame()`, freeze camera drift. Spawn one hostile watch 2.5 tiles in front of `P` and one owned watch 1.5 tiles behind.
2. **Spawned:** `HVU.units.length===2`; the hostile has `team===2`, `tier===1`; `sheetTex['u.watchIdle']` exists; the rig is visible; `renderer.info.memory.textures` rises by ≤ 13×3 per unit rig (the R1 clone path).
3. **Hit:** drive `keys.atk` taps aimed at it. Assert `hp` drops, `M.nums` gains an entry targeting it, `hitstop>0` on the contact step, `hitChain` increments, and the unit's rig `flash.material.opacity>0` that frame.
4. **Killed:** repeat until `u.dead`. Assert `kills` +1 exactly once, and `u.state==='dead'` plays `watchDeath` frames on the ANIM beat. Within ~2 s the unit leaves `HVU.units` (`onRemove`) and its rig is back in the pool and hidden.
5. **Player safety:** over 20 s of the owned unit fighting, `P.hp` never drops from the owned unit (`P.team===1` check). Then set `P.team` undefined for one tick in a negative-control run and show that it would.
6. **Owned levelling:** spawn 2 more tier-1 hostile watches away from `P` with `P.team=-1`, so the owned unit must fight them. Wait until its `renown>=2`. Assert `level===2` (`XP_TO(1)=12`, 6 XP per watch kill), `M.ticks` contains `LEVEL 2`, and `u.hp` was topped up 25%. Fallback if the AI stalls in headless: `HVU.grantXp(u,12)` asserts the same hook path, flagged as a weaker proof.
7. **Hygiene:** 0 page errors, 0 console errors; `__HV.texCensus().gl` equal before and after 10 spawn/kill cycles.

**Screenshots** (`pw-hvu0-shots.js` at 2x DPR, the `pw-hires-proof.js` harness):
- **A:** the knight mid-swing into the watch, `__HV.freeze`.
- **B:** the owned and hostile watch beside a house.
- **C:** a 1:1 crop through `mk-hires-sheet.py`: knight | slime | unit.

Reviewed for:
- the same 1.35 px line on the unit as on the knight;
- no rim or silhouette;
- art colours untinted;
- the hard ink shadow ellipse under the unit;
- the unit x-rayed through the roof like a slime.

**The gate that matters:** the user plays the `HVU=1` build and says the watch takes hits like Hollow Vigil.

**Rollback:** `HVU=0`.

### Phase 1 — The host contract made correct (still one or two kinds)

- **Lethal refactor (B4):** lift `impact()`'s `if(lethal){...}` into `onKill(e,nx,nz,ranged)`. For `e.hvu`, `impact` skips it and `fx.kill` calls it when `!a.by`. Refactor gate: `pw-smoke.js` and a kill-count run on slimes give identical `kills`, `ULT.meter` and heal totals before and after.
- **`_foe` shim** for `hurtPlayer`'s GUARD branch and `perfectDodge` (20-player.js:20). The `state='hurt'` setter maps to `HVU.hurt(u,{dmg:0,stun:.35,by:null})`. Patch V1 `freezeT` gives the dodge freeze.
- **Real telegraph drawer** for `M.hvuTele` (egg via `HVU.frameShape`/port of the demo's `eggPts`, wedge, capsule, beam, `single`) in 25-manga after `telDraw`.
- **CFG bridge** once at boot: `cfg.drawH`, `cfg.type` → `VOICE` key, `cfg.lungeV`, `cfg.dmg`, so `AudioSys.voice` never gets `520/undefined` (v9 host map §16, still right).
- Patches **V2** (arrow pan), **V3** (`P.markedT` decay).
- `fx.rite` / `fx.ult` in the game's grammar, with the involvement gate.
- Kinds: add `skel` (BONE DEEP rise) and `wizard` (WARD).

Probe `pw-hvu1.js`:
- a skel killed by a light blow rises, and `kills` does not move until it truly dies;
- a WARD-blocked lethal blow gives no kill;
- a perfect dodge through a watch swing sets `P.perfect` and freezes the unit `freezeT`;
- a full-armour KLANG staggers the unit (`u.state==='stun'`);
- `AudioSys` throws nothing across 60 s of a 6-unit brawl.

Rollback: revert phase-1 commits (phase 0 still stands).

### Phase 2 — Shared-texture rig, lazy families, three inline

- **R2** (§4.3): `uRect` in `inkLine`, one texture per sheet, unit rigs on it. Lazily bake the OFF stacks.
- **`03u` becomes per-group JSON blocks**, with `hvuNeedFamily` / `hvuFamilyReady` in `13u-unit-sprites.js`. Metadata (`n,fw,fh,px,py,pxs,fps,loop`) is always present; only `d` is lazy. The boot bake loop no longer sees unit sheets.
- **`03t-three.html`** inlined r128. `build.sh --cdn` stays available.
- `build.sh` pruning (§5.2.4).

Probe `pw-hvu2.js`:
- `renderer.info.memory.textures` **flat across 20 sweeps** that spawn and kill one of each of 10 families;
- a second sweep adds zero textures;
- `hvuNeedFamily('demons')` resolves, and no decode lands on a frame with `hitstop>0` (instrument `loadImage` resolve times against `hitstop`);
- the build loads with the network disabled (`page.route('**',abort)` for non-file URLs): title → play → a unit spawns;
- file size within §5.3.

Screenshot: the same 1:1 crops as phase 0, **pixel-diffed** against phase 0 (`ink-compare.py`). The shared rig must not change the look.

Rollback: `HVU_RIG=clone` build flag keeps R1.

### Phase 3 — Hostile units in the wave game

- Mirror team-2 units into `enemies[]` with the §3 guard list.
- `composeWave` gains unit entries: `ECOST[kind]=HVU.power(cfg)/POWER_PER_COST`, `EMAX` from group, and `unlock` straight from CFG. Tier/level by wave: `tierFor(n)`, `levelFor(n)` (design lens owns the curve). Spawn through `spawnFrom(...,{team:2})` with the `ED.ARRIVE_H` drop, placed like `applyRoster(n,true)`.
- Domain slow via `u.slowT`. `waveCard` silhouettes and `bossChip` for units. `isLiveBoss` accepts `e.hvu&&e.cfg.boss`.
- Families for wave n+1 requested in wave n's gap.
- Draw budget: the unit roster **replaces** slime slots within `waveSize(n)`; it never adds on top (v9 render map §8).

Probe `pw-hvu3.js`:
- **field-compatibility sweep:** every ability (20b: IRON FALL, CYCLONE, HOOK, INK/SIGIL, DOUBLE, BOMB, HOLLOW RAIN), the ULT domain, the companion's arrows, `lastStand`, `vigilHolds` and a spit parry, each against a unit. Snapshot and diff the unit's field set against a pure-HVU run. **No NaN, no field outside the allow-list, and `HVU.tick` never faults** (`e.faulted2` stays false).
- waves 1-10 on a unit roster, with 0 errors and `WAVE.alive` reaching 0 each wave.
- the existing `pw-reset-probe.js` 19/19, extended with "no unit survives `resetRun` except team 1".

Rollback: `HVU_WAVES=0` keeps phases 0-2 with units only in probes.

### Phase 4 — The army

- Patch **V4** `host.home`; orders on V (`makeDial` in `#slot`).
- `#army` strip; ally wounded-bar rule; foot tick; level/tier/milestone notifications with the combat gate (§6.2).
- RITE card on key 4 in the gap. ARMY pause page with four tabs, read-only while engaged. `hvuEvolve` with uid re-pointing.
- `hv_army` save/load at safe moments (§7).

Probes:
- `pw-hvu4.js`: across 6 waves, an owned unit reaches level 5 (grant-assisted if needed); the ▲ chip appears; key 4 in the gap advances to T2 and the rite plays (`M.title`/`shout` present, **`hitstop` stays 0 during any live wave**); the ARMY page edits stats in the gap and refuses edits mid-wave; evolve changes `u.kind` and keeps the uid, label and stats.
- `pw-hvu-save.js` (§7.3).
- Screenshots:
  - the HUD at 1366×768 and 1920×1080 with 4 chips;
  - the ARMY page;
  - the RITE card.

  Reviewed with the user for "easily seeable" and colour (no cream panels).

Rollback: `HVU_ARMY=0`. Hostile units stay.

### Phase 5 — Hand-off to the world plan

- The Roster pages link SEEN/STUDIED to "evolves into" and to owning.
- `hv_army` migrates to the IndexedDB `hv_folio` document.
- The v1 plan's squad tokens claim pooled unit rigs through the same `unitRender` pool.
- `hostile(a,b)` replaces `team!==team` inside the module: a sixth patch, needed only when factions arrive.

---

## 9. Risks and open questions this lens hands on

1. **Pacing of levels is a design question the integration surfaces.** At 6 XP per basic kill, tier 2 is ~29 kills and tier 3 ~265 (median ~160) of the unit's *own* kills. In a wave game where the hero takes most kills, owned units may never reach a rite. Options:
   - scale XP per wave in the host (`grantXp` is public);
   - share a fraction of the hero's kills;
   - accept that it is slow.
   
   Design lens owns this; the integration exposes `hvuXpShare`.
2. **`HVU.tick` is one dt for everyone.** The domain crawl (`sdt*0.42` for foes inside) becomes ×0.7 via `slowT`. A true crawl needs a per-unit time factor patch. Flag it for the ULT feel review.
3. **The module's AI runs on `enemies`-mirrored bodies whose fields the game's abilities also write.** The phase-3 field sweep is the guard. Expect 2-3 findings (candidates: `e.tilt` spring vs HVU tilt, `e.hy/hvy` launch vs flying `cfg.fly` bodies, `e.vx` shoves during HVU `charge`).
4. **Object identity on `evolve`** (and `mirrorimage`, `slimelet` and grave-raise spawns inside `tick`). Every game-side reference must go through `onRemove` or the uid map. The phase-4 probe covers evolve; lock-on to a unit that evolves mid-lock is the case to watch.
5. **The RITE card on key 4** assumes the gap's card flow can host a fourth card. If the waves/UI lens moves cards, the rite moves with the gap, not with the cards.
6. **Order dial pad binding** and **ARMY page Q/E** need a key audit against 18-state-input's `KEY` and the pad map before phase 4.
7. **Portrait art at chip size.** Tiny RPG idle frames are 13-40 px wide. At 46 px a nearest-neighbour upscale of 1-3× reads well for most, but `shade` (ppu 44.7, 67 px tall) and `frostfiend` need a crop box (`bodyBox`). Screenshot all 85 visible kinds' chips once in phase 4.
8. **The legacy SKETCH OFF path** for unit sheets is lazily baked (R2). OFF is a minority setting, but its first toggle with 10 families resident will hitch. Acceptable; measure it.

---

## 10. Summary (the 10 lines)

**Core finding:** v30's module slots into the fixed step and host contract cleanly, but the author's guide does not.
- It names five game functions that don't exist (`hurtPlayerFromUnit`, `decorCircles`, `M.telegraphs`, `burstBits`, `onEnemyKilled`).
- Its `attach` drops `fx.rite`, and its sfx lookup plays 1 of 7 sounds.
- Its `spawn` defaults make full-power allies (team 1, tier 5).
- The game's `P` has no `team`, so v30 makes it team 0 and owned units attack the hero.
- `HVU.render.install` would bring back tinted, rimmed, unsorted, line-less sprites.
- Four sheet keys collide with the hero's.

**Recommendation 1 — Vendor, don't adopt.** Ship the module unchanged in its own `<script>` with five named patches. Assign `HVU.host` wholesale from `29-hvu-host.js` with `P.team=1`. Draw units through the game's own rig from `drawInfo()`, with `'u.'`-namespaced sheets. Port v30's one great idea, one texture per sheet with the frame in the material, into `inkLine`.

**Recommendation 2 — One file, lazily.** Inline Three.js r128 (drop the CDN; the game becomes offline-complete). Ship sheets as per-group JSON blocks decoded on demand, with metadata always present. Budget: ≈ 1.15 MB for phase 0, ≈ 3.05 MB / 1.47 MB gzip with every family, alarm at 3.5 MB.

**Recommendation 3 — Combat shows state, safe moments allow editing.**
- In combat: a ≤ 4-chip army strip, a V order dial, the existing 3-second wounded bars and a foot tick; no text over bodies.
- Rites: a RITE card on key 4 in the wave gap, with no hitstop mid-wave.
- The sheet: a four-tab ARMY pause page, editable only when disengaged.
- Saves: `serialize` wrapped in a uid record, written only at safe moments (localStorage now, IndexedDB in the world phase).
- The Lab, drawer and dock stay dev-only.
- Phase 0 proves it with one tier-1 watch hit, killed and levelled, behind an `HVU=0` byte-identical rollback.
