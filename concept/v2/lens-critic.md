# Lens: THE ADVERSARIAL CRITIC — what units v30 breaks, obsoletes, improves or leaves standing in Folio v1

*Pass of 2026-09-13. Sources read: BRIEF.md; `concept/OPEN-WORLD-RPG.md` (all 526 lines); `units30/units.js` lines 1-1050, 1540-1640 and the sandbox UI 1837-1975; `units30/head.html` markup and CSS. Line numbers below refer to `units30/units.js` (the extract), not the 2247-line source.*

Labels: **BROKEN** = v30 makes it false. **OBSOLETE** = v30 already provides it. **IMPROVED** = v30 makes it much stronger. **HOLDS** = keep it.

## 0. Facts that change the plan, verified in code (the brief was partly wrong)

These come first because several verdicts below depend on them.

1. **The default team is 1, so `{team:0}` silently becomes team 1.** `spawn()` sets `team:(opts&&opts.team)||1` (line 955). Two things follow:
   - The guide's step 5 `spawnFrom(build,x,z,{team:0})` "works" only by accident.
   - **Every enemy the game spawns without `{team:2}` is an owned unit.** It can earn XP (`die()`, line 1027) and cannot be hit by the hero (sandbox line 1746 skips `team===1`).
   - The host must pass `team:2` on every hostile spawn. v9 had the same `||1` default, but in v9 team 1 meant "enemy".
2. **"Owned" is team 1 and nothing else.** `grantXp` returns early unless `e.team===1` (line 373), and `die()` credits XP and renown only when `a.by.team===1 && e.team!==1`. There is no separate `owned` flag, so any friendly faction unit put on team 1 would level up.
   - **Charm exploit:** `charm()` (line 932) sets `u.team=by.team`. An enemy hypnotised onto team 1 earns XP for its kills for 6-8 s, and your own unit charmed onto team 2 feeds XP to your army when it is killed.
3. **The hero earns no XP, and neither do the hero's kills.** The host calls `HVU.hurt(u,{...,by:null})` (guide step 4), and `die()` only credits a unit that is `a.by`. Army XP comes only from the army's own kills.
4. **ULTs no longer fire at an authored health threshold.**
   - At CFG load every `c.ult.at=Math.max(c.ult.at,0.7)`, and `after` defaults to 9 s.
   - The trigger (line 1333) fires when **hp < 70%, OR `combatT > after`, OR 3+ enemies within 5 after 3 s**. Advanced units (UNIQUE or ASCENDED) fire even sooner.
   - `encore`, UNIQUE and ASCENDED re-arm the ULT 20 s later (line 1311).
   - The result: a unit's ultimate is now a mostly *timed* autonomous beat, not a health-threshold tell.
5. **Tier gates the kit.**
   - `tierHas` (line 422): dash, charge, block and hop at T2; special, heal, summon and **passive** at T3; **ULT and smart AI** at T4. The move list is gated by `tierNeeded`.
   - Enemies default to T5 (`spawn`, line 966). A body spawned at T1-T2 has no PASSIVE and no ULT (`noPassive`).
6. **Levels are big numbers.**
   - Each level adds +2% hp and damage (`applyMods`, line 428), up to level 50: about +98%.
   - On top of that: milestones (VETERAN +5%, MYTHIC +10% to everything), stats (VIGOR up to +48% hp, MIGHT up to +32% damage, WILL up to 32% less damage taken), tier 5 (+10%), renown (up to +15%), ASCENDED (+20% hp and damage, 20% less taken, cannot be knocked down).
   - A finished unit is roughly 3-4x a fresh one. **Enemies take the same multipliers when spawned with `{level}`** (`spawnFrom` works for either team; the sandbox's `BT.enemyLevel` does exactly this).
7. **Evolution destroys the body and spawns a new one.** `evolve()` (line 311) does `serialize` → `onRemove` + `units.splice` → `spawnFrom`. It is a fresh allocation, not a pooled re-type.
8. **The renderer already shares one texture per sheet.** `render.TEX[sheet]` is shared, and the frame is chosen with a `rect` uniform. But `makeView` still builds three ShaderMaterials per body, and **custom, UNIQUE, ASCENDED and `customAccent` bodies get a solid accent-coloured silhouette 1.07x behind them.** That is an outline halo (`info()`, `olm`, `solid=1`).
9. **The sandbox UI is cream paper.** `--paper:#ece5d4`, `--paper2:#f4efe2`, the topbar at `rgba(236,229,212,.92)`, a striped paper sheet header and a paper arena (`PAPER='#ece5d4'`). It is exactly the look the user rejected three times.
10. **Hallokin exists only in the guide's comments** (lines 33, 51, 55). The game has no HVU integration yet: `grep HVU` finds nothing in `hv-work/src/`.
11. **Unit groups are sprite-pack buckets, not families.** `GROUPS` is astra / watch / orcs / men / undead / casters / beasts / demons / shadows, and the three capstones added late are forced into `custom`. **EVOLVE crosses groups freely:**
    - archer → demonarcher; harpy → succubus; wizard → lich; priest → plaguedoctor
    - axeman (men) → ironorc (orcs); lancer (astra) → pike (men); knife (astra) → verdagger (shadows)
    - toxcrawler (shadows) → plaguebearer (beasts); shade and frostfiend → voidreaper
12. **Name collisions v30 adds:**
    - `pumpkinred` is **THE HOLLOW KING**, against v1's Hollow Knight.
    - `warden` is still **THE WARDEN**, and it is now the **T5 evolution of the captain**.
    - `timewarden` displays as THE CHRONO TEMPLAR.
    - `mirrorimage` is a hidden summon.
13. **Team comparisons now appear 59 times**, where v1 counted about 15. fx is called 108 times.

---

## §1 The pitch

- **BROKEN: "you are the last Watchman".** The guide names the hero Hallokin and makes him team 1, "the player's side" with owned units. A *last* Watchman who commands a growing Watch contradicts itself. Evidence: EVOLVE `watch:[captain,knight]`, `captain:[warden]`, `knight:[templar,mirrorknight]`, `templar:[timewarden]`, and `FAMILY_MINIONS.watch`. Recommendation: Hallokin is the last Watchman *at the start*, and the game is him redrawing the Watch.
- **BROKEN: "you never get stronger by grinding".** Army power grows on kills: `grantXp` gives `max(6, power/8)` per kill, renown adds +1% per kill up to 60%, and ASCENDED is literally "a finished build that keeps fighting" (line 362). The pitch can still say *the hero* never grinds, but the army is progression by fighting.
- **IMPROVED: "every enemy you learn to beat can be spared and drawn onto your side".** It is now true all the way down. A spared body becomes a `serialize` build that levels, tiers, takes talents and evolves. The pitch line gets a second half: *"…and grown into something the world fears."*
- **HOLDS:** the sketch look as the world's physics; the small, dense world; walking into fights already happening.
- **Out of date: "a roster of about forty enemies".** v30 has 48 named CFG kinds plus pack bodies, 87 in the sandbox `POOL`.

## §2 The spine

| # | Spine claim | Verdict | Evidence and note |
|---|---|---|---|
| 1 | Lamps keep the drawing; restored colour is the progress bar | **HOLDS** | Nothing in v30 touches it. **IMPROVED:** `rite()` (fx.rite, the flash, `noMoveT` 1.3 s) is a ready ceremony to anchor at lamps (§6). |
| 2 | SPARE is the core verb | **HOLDS, and it is now load-bearing** | SPARE is the *only* door into v30's whole progression layer, because nothing becomes team 1 except by the host's choice. It went from "recruit a follower" to "found a lineage". |
| 3 | Your first blow picks your side | **HOLDS in design, BROKEN in implementation** | Three-way fights are legal (`pickTarget` and `dealHit` test `u.team===e.team`). But "owned" and "friendly" now share team 1, so a sided faction put on team 1 would earn XP and could be opened in the sheet as "yours" (`renderSheet`: `isPlayer=u.team===1`). Needs `hostile(a,b)` **plus an `owned` flag**. |
| 4 | No character levels | **BROKEN as written** ("no levels, no XP bar, no stat points") | v30 *is* `XP_TO`, `STATS` and `POINTS_AT`, and the sandbox sheet draws a literal XP bar (`.bar.xp`). It survives only narrowed to Hallokin (§ Big Question 1). |
| 5 | The wave game survives whole | **HOLDS, IMPROVED** | Vigils become where the army trains: kills mean XP and renown. The sandbox's own `WAVES` and `spawnWave` show the author expects it. |
| – | The arena moves with the player; only nearby bodies are real | **HOLDS, harder** | An owned army has to stream *with* you. `serialize`/`spawnFrom` is exactly a token ↔ body round trip (**IMPROVED**), but `evolve` and `spawnFrom` allocate fresh bodies (fact 7), which fights the "no makeSprite after boot" pool rule. |

## §3 The look is the world's physics

- **§3.1 the six forces: HOLDS.** THE PANEL gains one row: the **rite** is the Chronicle drawing a unit anew.
- **§3.2 the colour rule: HOLDS, and v30 BREAKS it in two places the rewrite must name.**
  - The sandbox UI and arena are cream paper (fact 9). The roster, lab, sheet and evolution previews are the best army UI we have, and **they cannot be lifted as styled**. Keep the markup and the information design, re-token the palette: saturated panels, dark ink text on strong colour, never `#ece5d4`.
  - The accent-silhouette halo on custom and advanced bodies (fact 8) is an outline. The user just asked for "no ugly outlines", and commit 94145d0 removed white outlines. Replace it with a non-outline mark of rank: a rank glyph over the head, an under-stroke, or an accent ground ring.
  - `setLook`'s `customTint` is `0.55+0.45·c`. It multiplies the sprite, so it tints and does not whiten. **HOLDS.**
- **§3.3 birth and erasure: IMPROVED.**
  - Evolution is a scrub-out plus an ink-in of a new body, and `evolve()` already despawns and respawns in place. Play the scrub-out, then the three-stage ink-in, on the 10 Hz clock: **an evolution is a redrawing.**
  - The skeletons' BONE DEEP rise (`die()`, the Summon strip) is already "not scrubbed all the way".
- **§3.4 the watched circle: HOLDS.** Nothing in v30 competes with it. One interaction: the circle is the player's style rank, and style could also scale army XP in the fight (§6.6).
- **§3.5 the region hands: HOLDS.**
- **§3.6 the back of the page and the Reference: HOLDS.**
- **New legibility risk (not in v1):** a fielded army of 4-8 bodies with accent halos, title glyphs, `LEVEL n` marks and `READY TO ADVANCE` shouts (`grantXp` calls `fx.mark` and `fx.shock`) is exactly the "mud at scale" checklist items 1, 2, 7, 9 and 10 warn about. See §7.2.

## §4 The world

### 4.1 Premise
- **HOLDS:** the drawing is being erased, the Draftsman, the double-meaning title, and SPARE as "give it back".
- **BROKEN:** "You are the last Watchman of that Vigil." Hallokin must be named, and the story must accommodate an army (§1).
- **HOLDS:** the two studies, the last stand and the domain as fiction for the *hero's* kit. HVU never touches the hero.
- **HOLDS:** "the companion archer is drawn from the skeleton family". She is the game's own `22-companion`, not an HVU unit. **Watch out:** v30 has an HVU `archer` (THE ARCHER, accent `#c9a86a`) that evolves into headhunter or demonarcher. Keep her outside HVU and do not let the player "own" a second archer that reads as her.

### 4.2 The factions
The table is **BROKEN at the membership level and HOLDS at the political level.**

- **"Membership follows the units' own accent colours": BROKEN.** v30 has many accents that no v1 faction claims: the demons' reds and purples, the shadows' near-blacks (`#2a2a3a`, `#3a3a4a`), the capstones' blues. Evolution changes a body's accent (knife `#8fd3d0` → verdagger `#3a8a5a`). Only `customAccent` survives an evolve (`serialize` carries `tint: e.customAccent`). Colour-as-membership cannot be read off a crowd once armies evolve.

Row by row:

- **The Watch of Ashridge**
  - **IMPROVED:** v30 made the Watch a lineage. watch → captain or knight → templar → timewarden, or knight → mirrorknight; captain → warden. This is the *player's* faction tree now, not an NPC faction.
  - **BROKEN:** "too few and too tired: lamps you don't help defend fall" still works for NPC patrols, but the player's own Watch is the obvious garrison.
  - **Out of date:** axeman now evolves into ironorc (the orcs' line) and pike into rider.
- **The Gilt Order**
  - **IMPROVED:** v30 hands it a canon: templar → **timewarden (THE CHRONO TEMPLAR, `rewind`, `twStasis`)**. A unit whose power is *holding things still* is the Varnish made playable, and it is the ideal Chapter III lieutenant. The **inquisitor** (cannoneer → inquisitor, `pyre`, `iqPillars`) is the Order's crusade against the armatures. The hierophant is now priest's T4.
  - **BROKEN:** "THE SENTINEL (renamed shield-bearer)". `warden` is still THE WARDEN and is now captain's T5, i.e. a *Watch* capstone, not an Order unit.
  - **BROKEN (fiction):** the priest's other branch is the **plaguedoctor**. The Order has a lineage that turns into plague.
- **The Underdrawn**
  - **IMPROVED:** a deep lineage now exists: skel → ironskel or skelarcher → greatskel → **gravelord** (`graveraise`: raises the dead at kills); bonepale → boneblade → bonedark → **revenant**; necro → warlock → **lich** (PHYLACTERY, DEATH WINTER).
  - "Many are the Watch's own dead" gains a mechanic: an armature you SEAL can be *grown*.
  - **Out of date:** the Revenant as region lord is now a T5 evolution the player can own.
- **The Greenhand Warband: HOLDS and IMPROVED.**
  - orc → berserker or eliteorc → warchief; ironorc and axeman feed in.
  - The WARDRUM fiction ("the drum is their boil") is untouched.
  - **Note:** WARDRUM heals the warchief on *enemy* deaths (`die()`: `u.team!==e.team`), exactly as v1 described.
- **The Redrafters: BROKEN (thin) and IMPROVED (re-foundable).**
  - v1 built them from wizard, warlock, mesmer and headhunter. In v30 the warlock is the necromancer's T4 (Underdrawn), the headhunter is the archer's T4, and the mesmer is the wizard's T4.
  - Meanwhile v30 added an unplaced **charm and hex kit**: the succubus's KISS (`charm`), the darkwarlock's HEX (`mark`), the eyeball's GAZE (`mark`), the siren (`drainOnHit`, song).
  - The ten-body **demon** group is also unplaced: demonlord → abysslord, flamegolem → magmacolossus, imp → demonbrute or succubus, the hellbats, the hellhounds.
- **The Wild: HOLDS and grows.**
  - Slimes now have kings (slime → slimekinggreen; slimeblue → slimeking; slimepink → plaguebearer or slimekingpink) and lava slimes (lavaslime → flamegolem → magmacolossus).
  - The Hatch now has the **shadow** lineage: nightfang → verdagger → duskclaw → noctislicer → **voidreaper**, with shade and frostfiend also ending in voidreaper.
  - Bats: bat → hellbat → bloodwing → **stormwing**.
- **The Hollow as an absence: HOLDS.**

**Missing in v1:** there is no place for the demons, the shadows' capstone (voidreaper), the harpy and siren, the pumpkins, the eyeball and ghostfire, the minotaur or the frostfiend. That is about 25 of v30's kinds.

### 4.3 The Signed
- **HOLDS:** the teal accent is still unique to knife, blade and lancer (`#8fd3d0`, lines 78-84), and "a signature is the one mark the Hollow respects".
- **IMPROVED, and it should be canon:** `setLook(e,label,tint)` names a unit and gives it a custom accent that **survives evolution** (`serialize` → `tint`, `spawnFrom` → `setLook`). Naming a unit *is signing it*. The player can sign their own units: a label plus a colour. Signed units are the ones the Hollow cannot erase, which gives the fiction a reason to let a named unit be the one that is never lost (§6.6).
- **BROKEN:**
  - The kits are unchanged: knife `kA1`/`kUp`/`kPlunge`, blade tempo, lancer throw (`HAND_KITS`).
  - But "flee at their ULT's health threshold" and "at 60% he fires ASTRAL TEMPEST" (§5.2) are false. Every ULT's `at` is forced to 0.7, and `after` is 9 s (fact 4).
  - Rival pacing must be redone: SHADOWSTEP fires about 9 s into a duel whether or not you hurt him.
- **BROKEN (fiction):** once recruited, the Signed can evolve out of their identity. knife → verdagger (shadows), blade → spellblade (a *different* blue accent `#40b0e0`), and **lancer → pike at T4**. The pike is a rank-and-file Pikeman, a demotion in fiction, hp 100 against the lancer's 85 base. Pick one:
  - (a) the Signed have no EVOLVE entries in the game's copy; or
  - (b) they evolve but keep teal and their name through `setLook`, and the lancer's branch is removed.
  - Recommendation: (b) for blade → spellblade, and cut lancer → pike.
- **IMPROVED:** "rivals remember" can now *grow*. A rival is a build object. Between meetings the host raises its `level`, picks a TALENT or PATH that counters you (it takes BULWARK if you guard-break it, the DUELIST path if you kite), and gives it a MASTERY (IMPALE on its thrust). The v1 hand-authored counters become picks from tables that exist.

### 4.4 The Hollowed
- **HOLDS:** one AFFIX in the game's `21-waves` plus a line-only render state.
- **Legibility risk:** a line-only contour on a "no ugly outlines" brief. It must be the ink line at full weight over a transparent fill, never a halo. It cannot use v30's `solid` accent silhouette (fact 8), which is a halo by construction.
- **IMPROVED:** the Hollow can take *your* units. A fielded owned unit left on blank paper hollows, and it is a named build you can SEAL back. That makes a deep personal stake without permadeath.
- **Name clash:** `pumpkinred` = THE HOLLOW KING. Rename it (e.g. THE BONFIRE KING, after its ULT BONFIRE) so "Hollow" means only the Hollow.

### 4.5 The Folio
**HOLDS in full:** 12 sheets of about 128 × 128 tiles, page turns, a Great Lamp plus 3-5 roadside lamps, the map as the drawing. v30 does not touch world structure. **IMPROVED:** a lit lamp is where an owned unit *garrisons* (the Big Question 3 candidate) and where a rite is taken.

### 4.6 The regions
- **"Tier is the CFG `unlock` field (1 = slime, 10 = warchief)": BROKEN in range.** `unlock` now runs to **12** (magmacolossus, abysslord, stormwing, voidreaper, timewarden, gravelord at 12; lich, plaguedoctor, mirrorknight, inquisitor, spellblade, siren and frostfiend at 11 or 12), and `mirrorimage` is 99.
- **IMPROVED:** v30's **tier** is a *behaviour* dial that maps exactly onto a region's difficulty:
  - T1: no dash or block, no passive.
  - T3: passive on.
  - T4: ULT and smart AI (focus-fire the wounded, target switching).
  - T5: faster cooldowns.
  - A region can say "skeletons here are T2", and they will literally not use BONE DEEP.
- **Per region:**
  - Ashridge, the Lamplit Rows, the Brushlands: **HOLD**.
  - Vellum Abbey: **IMPROVED** by the timewarden and inquisitor.
  - The Palimpsest: **IMPROVED** by the gravelord's `graveraise`, which raises the dead at kill sites. That is "the ground remembers" on a unit that exists.
  - The Hatchwood: **IMPROVED** by the shadow line and the hellhound PACK.
  - The Spill: **IMPROVED** by the slime kings and lava slimes.
  - The Tracing Tower: **IMPROVED** by the **mirrorknight and mirrorimage** ("every prop drawn twice") and the siren and succubus charm kit.
  - The Margin: **HOLDS**.
- **Missing:** a home for the demons (magmacolossus and abysslord). The Spill's "ink burning" or a new **Burn** region is the natural slot (flamegolem MOLTEN, pumpkinred KING OF FIRE, blackknight BLACK FLAME).

### 4.7 The hub
- **OBSOLETE:** "the Barracks: the followers you have spared and bound". The sandbox's **sheet** (overview / stats / skills / talents / gear / evolve tabs, ADVANCE button, POWER box, title, renown) *is* the Barracks UI. It needs a reskin (fact 9), not a redesign.
- **IMPROVED:** the Scribe, Smithy and Chapel can shrink. The Smithy sells `ITEMS`, and the Chapel is where rites are taken.
- **HOLDS:** the Watch-house, the Lamp Board, the Chronicle, the Record.

### 4.8 The two bosses
- **THE WARDEN (slime boss): HOLDS, and the collision is worse.** An owned captain that reaches T5 prints "EVOLVED" and becomes "THE WARDEN" (`evolve` → `rite(n,'EVOLVED')`, the sheet name). Rename the unit in the game's CFG copy, as v1 decided. v1's pick was THE SENTINEL, but it is now a **Watch** capstone, so its fiction moves from the Order to the Watch. That fits "the Order posted a Sentinel as Warden of the Well" even better if the Watch posted it.
- **THE HOLLOW KNIGHT: HOLDS.** He is game-side (`hollow`, `EBRAIN.hollow`), not an HVU unit. Keep him outside HVU so he can never be evolved into or owned.

### 4.9 The arc
- **HOLDS:** chapters I-IV and the Gutter.
- **BROKEN:** Chapter IV's "ERASE him and the companion leaves you: you are the Watchman now, **alone**". The player leads a levelled, named army. "Alone" must become something the army can witness: the Watch you rebuilt stands at the lamps, and she is not among them.
- **BROKEN:** Chapter II's "every Signed you beat or spare… moves the standings". Sparing a Signed now also adds a T1 or whatever-tier unit to a roster the player invests in, so the three Signed are the *best builds in the game*, not just rivals.
- **IMPROVED:** Chapter III's "the Mesmer takes her with MASS HYPNOSIS" can take *units* too. `charm` works on any non-boss, and a fight against your own ASCENDED unit is a real climax.

## §5 Playing it

### 5.1 The first 10 minutes
- **HOLDS:** the opening image (colour comes from lamps), the first fight with two WATCH guards beside you, the Knife hook, the relight, the gate vista, WHOSE SIDE?.
- **IMPROVED:** the two guards who fight beside you can be the player's first two owned units: THE WATCH, T1, level 1.
  - T1 is exactly "walk up and swing" (tiers comment, line 283). The tutorial army is simple *because* of its tier, and the player watches them earn T2 (dash, block) in the first hour.
  - Their `LEVEL 2` mark is the first "units grow" signal. Gate it through the shot director.
- **BROKEN:** "He escapes over the gate at 50% health, his ULT threshold" (fact 4). The scripted chase must drive SHADOWSTEP itself (set `e.ultUsed` or call the ULT at a scripted moment), or rely on the 9 s timer.

### 5.2 The first hour
- **BROKEN:** "~45 min: at 60% he fires ASTRAL TEMPEST" (fact 4).
- **BROKEN:** "End of hour one: … one follower". The v30 pace is XP 12·l^1.35 per level (12, 31, 53, 78 …; T2 at level 5 ≈ 174 XP ≈ 20-30 kills of `power/8` ≥ 6 XP). A realistic hour-one army is **3-4 owned units, one of them at T2**.
- **HOLDS:** ~30 min SPARE; ~35 min the 8 v 8 bridge clash; ~55 min the Well silhouette.

### 5.3 A session at hour 15
- **BROKEN:** "The Lancer, the player's follower, fires SKYFALL as a CALL at A rank". An owned unit fires its own ULT on the v30 trigger (fact 4), so a CALL needs a module flag that holds owned units' ULTs (see §6.2).
- **IMPROVED:** "Six Watch holding against twelve Warband". The Watch at the mill can be *your garrison*, at the levels you left them.
- **HOLDS:** open the book, the walk, the Palimpsest Vigil (now also the army's training), the Knife remembering, banking at a lamp.

### 5.4 Exploration
**HOLDS in full.** Marginalia, sound first, landmarks, ability-shaped secrets: all hero and game-side.

### 5.5 The Vigil
- **HOLDS:** claim, defend, endless; cards live here.
- **IMPROVED:** a Vigil is where owned units earn XP and renown at scale.
  - `ECOST` (each follower adds its threat) can be **`HVU.unitPower(e)`** directly: a live, level-, stat- and gear-aware number. Enemy budgets can use `HVU.power(cfg)`.
  - v1 needed to invent both numbers. **OBSOLETE** as bespoke work.
- **Watch out:** a Vigil rewards XP to *fielded* units only. The bench does not level, so rotation becomes a real decision. Good, but say so.

### 5.6 Faction clashes and rivals
- **HOLDS:** arriving neutral as team −1 (`pickTarget` and `dealHit` both skip `P.team===-1`; the sandbox's `setGhost` uses it).
- **HOLDS:** first blow, one stray forgiven, pick neither, leaders with ULTs.
- **Broken detail:** "`team = −1`" means "leave me alone" only for the *player* (`pickTarget` and `dealHit` test `P.team!==-1`).
  - A *unit* on team −1 is still targeted by everyone, because the only unit-to-unit skip is `u.team===e.team`.
  - The player's army cannot go neutral through the team id. It must hold: no `hunt`, `aggro=false`, and a host leash that keeps it out of engagement until your first damaging blow picks the side.
  - Without this, your army starts every war for you.
- **"Every clash has a leader with a named ULT … killing it breaks its side": HOLDS**, but the ULT is the timed beat (fact 4), so the leader's climax arrives at about 9 s, not at a health line.
- **Rivals remember: IMPROVED** (§4.3).
- **The Mesmer: IMPROVED, and more dangerous.** MASS HYPNOSIS (radius 6, not bosses) can flip half your fielded army. Add the charm exploit fix (fact 2).

### 5.7 The three stories
- **"I walked into a war": HOLDS.**
- **"The Knife remembered me": IMPROVED.** Ten hours later he is *your* Knife: level 34, signed in your colour, one fight from UNIQUE.
- **"I kept the page alive": HOLDS.**
- **Add a fourth v30 story:** *"My first skeleton, the one I sealed in the Palimpsest on hour two, ascended in the Margin. It's a Gravelord now, it has my name for it, and it raises the dead we kill."*

## §6 Progression

- **Header "no character levels, no XP bar and no stat points, no random stat rolls": BROKEN for units, HOLDS for Hallokin.**
- **"No random stat rolls" HOLDS:** v30 has no RNG in builds. Everything is authored tables.

### 6.1 The Roster
- **SEEN / STUDIED / BESTED: HOLDS.** They are hero-side knowledge, and the tell-learning is the combat teacher. **IMPROVED:** the sandbox `#roster` already lays out each kind's moves, PASSIVE, ULT, "evolves into X at T4" and the T1-T5 kit ladder (lines 1930s). The page content is written.
- **BOUND: BROKEN** as "joins the Barracks as a follower". It must now mean "becomes a build".
- **An open decision v1 never faced: at what tier and level does a spared body join?** Options:
  - (a) T1, level 1, a fresh drawing. Honest to progression, but a spared Blade at T1 has no TEMPO and no TEMPEST, which feels like a punishment.
  - (b) The tier it was fought at, level 1. It skips the whole ladder.
  - (c) **One tier below the region's tier, level 1, with `readyTier` blocked until the level catches up.**
  - Recommendation: (c) for commons, and **the Signed and lords join at T3** so their passive is live from day one.
- **"+8% damage against the kind (known)": HOLDS**, small.
- **The Tally as the nearest thing to a level: HOLDS for the hero.** But now there are two "levels" on screen. Name them apart: the Tally is knowledge, *Level* belongs to units.
- **"Grinding does nothing": BROKEN for units, HOLDS for the Roster.**

### 6.2 SPARE, BIND, CALL
- **Spare and Seal: HOLDS.**
- **Bind: OBSOLETE in mechanism, HOLDS in intent.** v1 said "a follower is a units-module body spawned on your team with a leash on you". v30's `spawnFrom(build)` is the bind, and `pickTarget`, spacing and summons already work for team 1. What v30 lacks is the **leash** (the sandbox has none; `ARC.leash` is the game's), so that stays new.
- **"In the field the companion + one follower; in a Vigil two": BROKEN.**
  - v30 is built for armies. The sandbox spawns 12 v 12 (`spawnBattle`), and `TALENTS.warlord`, `ITEMS.war_banner`, STANDARD, ZEAL and the UNIQUE ally aura all pay only with several allies.
  - With one follower, half the talent and item tables are dead.
  - Recommendation: **a field party of 3 (4 in a Vigil) plus the companion**, drawn from a Barracks roster, under the awake cap.
- **"Each follower adds its threat to ECOST": OBSOLETE** as bespoke. Use `unitPower` (§5.5).
- **"The companion stays outside the team system": HOLDS.**
- **"A downed follower kneels… revived with a 1.5 s stand": HOLDS.** v30 has no downed state (`die` → `dead`), so this is a host rule: intercept a team-1 death and kneel instead.
- **CALL (hold Q, paid with style): BROKEN in mechanism, and the idea is worth saving.** v30 owned units fire their ULT autonomously (fact 4), and T4 is required (`tierHas 'ult'`). Recommendation: add one module flag (`e.holdUlt`), set for owned units, so the trigger at line 1333 never fires for them. The player spends a style pip to call it. **IMPROVED:** CALL now has a natural unlock (the unit reaches T4), a natural upgrade (TALENTS tier 4 EAGER, BRUTAL and ENCORE all modify the ULT), and ENCORE or UNIQUE give a second CALL.
- **Bond (Bound / Sworn / Oathbound): OBSOLETE.** Level, tier, milestones (VETERAN, HARDENED, MASTER, UNIQUE, MYTHIC) and titles (ELITE → LEGEND, ASCENDED) are a richer bond ladder that already exists and shows in the sheet. "Oathbound: its PASSIVE can be worn as a trait" can survive as a *milestone reward*: when an owned unit reaches UNIQUE (level 40), Hallokin may wear its PASSIVE.

### 6.3 The loadout: two hands
- **"No new classes; the two hands are the class system": HOLDS** for the hero.
- **4 techniques, 3 traits, 3 charms, 1 Law: HOLDS**, but the total UI load is now hero loadout *plus* per-unit sheets. Cut the hero side (§10).
- **"1 follower" slot: BROKEN** (§6.2).
- **Techniques have teachers: HOLDS.** Iron orc, axeman, Blade, headhunter, captain and mesmer all still exist with the moves cited. **Watch out:**
  - Many teachers are now evolution *targets*, not region spawns: the headhunter (archer's T4), the mesmer (wizard's T4), the warlock (necro's T4).
  - THE MESMER as SCRIBBLE DOUBLE's teacher is weaker than the new **mirrorknight + mirrorimage** (a gesture copy of yourself). **IMPROVED:** move SCRIBBLE DOUBLE's teacher to the Mirror Knight.
- **"Ranks I → III follow the perks' three-stack model": IMPROVED.** v30's skills are ranks 1-3, then at rank 3 a choice of two MASTERY options by move type (swing: CLEAVE or RUPTURE; thrust: IMPALE or RUN THROUGH; shot: TWIN or PIERCING…). The hero's techniques should copy that grammar exactly, so hero and army speak one progression language.
- **Traits wear a body's sentence: HOLDS** for the hero. **Watch out:** 18 new PASSIVE lines (`NEWPASS`) add more translatable ones (LEECH, SWARM, KEEN, PACK, RIME, IRON WILL, PLATED).
- **Laws bend the domain: HOLDS.** They are hero-side and v30-independent. Lord list update: the gravelord, lich, timewarden, voidreaper and abysslord are better lords than the v9 custom bodies. See §9.6.

### 6.4 Cards → charms
**HOLDS.** Cards are hero-side and run-scoped (`snapT`/`restoreT`). v30 ITEMS are unit-only and do not collide.

### 6.5 Loot that never changes a drawing
- **HOLDS for the hero:** gear never changes reach or timing.
- **BROKEN as a *world* rule.** v30 unit gear and stats change reach and silhouette: `long_reach` +15% reach, the REACH stat +3% per point (up to 24%), `giant_heart` and JUGGERNAUT scale 1.2 and 1.1 (bigger sprite), the REAVER path +20% reach, the ASCENDED scale 1.08. Units' hitboxes are derived from frames times the reach multiplier, so this is *coherent* for units. Restate the rule: **Hallokin's drawing never changes; units may be drawn bigger.**
- **Nibs and Inks: largely OBSOLETE as a loot pillar.** The world now needs to drop `ITEMS` (artifact, weapon, trinket) for units, and two loot tables at once are confusing. Recommendation: loot drops are unit ITEMS; the hero's Nib and Ink shrink to a handful of story rewards, or are cut (they were already cut #2).
- **"Inks are taken from Crowned elites' affixes": IMPROVED** as unit ITEMS. VAMPIRIC ≈ `leech_fang`, WARDED ≈ `ward_stone`, VOLATILE ≈ `ember_core`. "You take the affix off the thing" → a unit artifact.

### 6.6 The economy and death
- **Ink per kill × `RMUL[rank]`: HOLDS.** **IMPROVED:** apply the same style multiplier to army XP in the player's fight. `grantXp(e, n)` takes any amount, so the host can scale it by `RMUL`. It is the one bridge between "the hero never levels" and "the army does": *you fight well, they learn faster.*
- **Ink spent on technique ranks and projects: HOLDS.** Add rites: advancing a tier and evolving cost ink or a quill at a lit lamp. `advance` and `evolve` are free in the module, so the host charges.
- **Death tears the page, ink pool, Crowned elites take ink and an affix: HOLDS.**
- **Unit death is unaddressed in v1.** Recommendation: no permadeath (the build is the save). A unit killed in the field walks home and is out for the region, as v1 already said. A unit lost on blank paper **hollows** and must be SEALED back (§4.4). A *signed* unit (`label` + tint) cannot hollow: the signature fiction.

## §7 Combat in an open world

- **7.1 Panel, awake cap 24 / 32, one attack budget: HOLDS, tighter.** The cap now includes your fielded army (3-4), the companion, and summons from owned CONJURER-path units or necro, warlock and lich lines (`summon.max` 5-6 each; the UNIQUE and MASTERY `extra` add more). **Cap owned summons** or the army alone eats the budget. **IMPROVED:** `TOKENS=2` is per target (`tgtKey`), and T4+ units focus the wounded (`smart`). Merging it with `attackCap()` still stands.
- **7.2 The feel involvement gate: HOLDS. It is now the single most important rule in the plan.**
  - The sandbox host does `hitstop(0.06)` plus trauma on **every** kill (`fx.kill`, sandbox line 1811), including unit-on-unit kills. That is exactly the failure v1 predicted.
  - With an army, every one of your units' kills would freeze the screen.
  - The gate must be extended to the new fx: `fx.rite` (flash, 1.3 s freeze of the unit), `fx.mark` 'LEVEL n' / 'READY TO ADVANCE' / 'EVOLVED' / milestone names, and `fx.ult` for owned ULTs.
  - Rule: owned units' hits get sparks and sound, **no hitstop, no trauma, no numbers.** Level-ups queue to the end of the fight as one card. Rites happen at lamps, never mid-fight. (`grantXp` fires `READY TO ADVANCE` mid-combat today; hold it.)
- **7.3 The camera grammar: HOLDS.** Add one row: **the rite / evolution = "This is yours now"**, the colour-flood row, with the ink-in of the new body. No new tool.
- **7.4 Difficulty is behaviour, not sponge: HOLDS in principle, IMPROVED in means, BROKEN in numbers.**
  - **THREAT = enemy `tier`.** The tier ladder *is* behaviour: moves, dash, block, passive, ULT, smart AI, cooldowns.
  - **WEIGHT = enemy `level`.** +2% per level plus milestones; clamp it by region, never scaled to the player's army.
  - **BROKEN:** "your base damage stays in 10-26" and "no single enemy hit exceeds 35%" were written for one hero against unlevelled enemies. A level-40 enemy is UNIQUE (it rises once, fires its ULT twice, and gives allies +10%). A level-50 ascended-tier enemy hits roughly 2-3x base. **Keep non-lord enemies ≤ level 39** and apply the 35% hit cap in `hitPlayer` (host-side, so it still works).
  - **Watch out:** the Tally's WEIGHT clamp 0.8-1.6 must not also scale by army power, or the world becomes Oblivion-style level scaling.

## §8 How it gets built

- **8.1 "Three structural edits to the units module": BROKEN (too few).** The list is now:
  1. `hostile(a,b)` over **59** team comparisons, not about 15.
  2. `host.claim` / `host.park`, which must also cover **`evolve()`** (it splices and respawns, fact 7).
  3. Staggered targeting.
  4. **An `owned` flag** decoupled from team 1 for `grantXp` / `die()` credit and the charm exploit.
  5. **`holdUlt`** for CALL.
  6. **Pass `team:2` on every hostile spawn**, or change the `||1` default.
  7. Remove or replace the accent outline in `render.info`, if HVU's renderer is used.

  The author calls v30 "final" (`VERSION '2026-09-13 final'`), so these are forks of a frozen module. Keep them as a small, named patch list applied at build time.
- **"Only people stream": IMPROVED.** `serialize` / `spawnFrom` round-trips the entire state of a unit, so a squad token can literally be an array of build objects.
- **Saves: OBSOLETE in part.** The unit save format exists (`serialize`). The versioned IndexedDB document just stores `army: [build…]` plus rivals' builds.
- **File format: HOLDS, heavier.** v30 sheets are 1.33 MB against the game's 927 KB. Do **not** ship the inlined Three.js r128 (603 KB); the game already loads it. Lazy JSON art families (v1) are now necessary from phase 1, not phase 3.
- **8.2 Reused versus new: several "new" rows are now reused.**
  - SPARE's target state is a build.
  - The follower body is `spawnFrom`.
  - Bond ladder → milestones and titles.
  - Encounter cost → `unitPower`.
  - Army UI → the sandbox sheet, reskinned.
  - Save of units → `serialize`.
- **8.3 Phase 0:**
  - **Gate 1, the shared-texture actor material: partly OBSOLETE** if HVU's renderer is used (a shared `TEX` per sheet plus a `rect` uniform). Per-view materials remain.
  - **Gate 5, the user's hands: HOLDS**, and it must now be played **with a fielded army of 3**, because that is where hitstop spam will show.
  - **Phase 0.5, the Roster in today's game: IMPROVED.** Add "a spared body joins as a T1 build and levels in waves" to the existing wave game. v30 was built and demoed for exactly that (sandbox WAVE / BATTLE).
- **8.4 The hardest problems: HOLDS, plus four new ones.**
  1. Army pathing, since a leash is not in the module.
  2. Numeric creep (levels × stats × gear × ascension).
  3. UI load: a sheet with 6 tabs per unit versus checklist items 1-4.
  4. Asset weight.

## §9 Decisions, one by one

- **9.1 Unlit land looks like full colour or blank paper, with no middle state: HOLDS, non-negotiable.** v30 adds two ways to break it that the decision must now name: the cream sandbox UI and the accent-outline halo (fact 8, fact 9).
- **9.2 The Folio, 12 × 128² sheets, the linked-arenas fallback: HOLDS.** v30 changes nothing structural. Its only cost is memory and asset weight per sheet (1.33 MB of sheets).
- **9.3 Progression, "one model: no levels…": BROKEN.**
  - It was an either/or against the systems lens's player level. v30 settles it as a split: **the hero knows, the army grows.**
  - OBSOLETE: the follower slot, bond levels, Nib and Ink as a pillar.
  - HOLDS: the Roster, card → charm at dawn, techniques, Laws.
  - "The Tally" stays but must not be called a level.
- **9.4 The Vigil's one role is holding land: HOLDS, with a second role.** A Vigil is also where the army is trained, since XP comes from kills and Vigils are the densest kills. That does not dilute "one content type".
- **9.5 The two Wardens: HOLDS and is more urgent.** `warden` is now captain's T5 (an owned unit can become "THE WARDEN"), and `timewarden` shares the key. Rename it THE SENTINEL in the game's CFG copy, and move its fiction from the Order to the Watch (`GROUPS.watch` already lists it: `watch:['watch','captain','warden','headhunter']`). Also rename `pumpkinred` THE HOLLOW KING.
- **9.6 Other contradictions:**
  - **Who the Signed are: HOLDS**, with evolution caveats (§4.3).
  - **The Hollow is blankness, the Blot is loose ink: HOLDS.**
  - **"Five political factions plus the Wild plus the Hollow": BROKEN in membership** (§4.2 and Big Question 2). About 25 v30 kinds, including a whole demon family and every T5 capstone, have no home.
  - **Name collisions: HOLDS.** Add the Hollow King, the Chrono Templar and the Warden-as-capstone.
  - **Accidental hits on allies: HOLDS**, plus: the hero cannot hit owned units at all (sandbox line 1746, guide step 4), so the forgiveness rule applies only to sided faction allies.
  - **Where things live: HOLDS.**
  - **Region hands that broke the colour rule: HOLDS.**
  - **"Six region lords from the custom bodies (Hierophant, Revenant, Warchief, Plaguebearer, Mesmer, Warlock)": BROKEN.**
    - All six are now **mid-tier evolution targets** the player can own: hierophant T4 of priest, revenant T5 of ironskel or bonedark, warchief T5 of orc, plaguebearer T4 of slimepink or toxcrawler, mesmer T4 of wizard, warlock T4 of necro.
    - A lord you can raise from a slime in an afternoon is not a lord.
    - v30 supplies proper `boss:true` T5 capstones: **gravelord, lich, magmacolossus, abysslord, stormwing, voidreaper, timewarden** (plus mirrorknight, inquisitor, spellblade, siren, frostfiend, the slime kings, the minotaur and the demonlord).
    - Recommendation: **the region lords are the capstones**, and the v9 customs become lieutenants (Big Question 3).
  - **"Lords are never spared": HOLDS, and it is re-motivated.** You never own a lord's body, but besting one unlocks its *form* for your army.

## §10 Cut list and risks

- **10.1 The cut order: mostly HOLDS, re-ranked.**
  - "2. Nibs and Inks" moves to **cut first**. v30 ITEMS fill loot.
  - "6. Bond levels": **already obsolete**, delete.
  - Add new cuts: the hero's 3 traits and 3 charms, if the army sheet carries build identity; PATHS for units (v30 has them, but they are the least legible layer); MASTERY choice for the hero's techniques (keep ranks only).
- **"Never cut": HOLDS, with additions.** Add **the involvement gate extended to army fx** and **the owned / team split**. Full list at the end of this document.
- **10.2 Risks: all HOLD. Add:**

| Risk | What it looks like | Mitigation grounded in v30 |
|---|---|---|
| **Army hitstop soup** | Your 4 units' kills freeze the screen and print numbers; LEVEL marks overprint the fight. | §7.2 extended gate over `fx.kill`, `fx.number`, `fx.mark`, `fx.rite` and `fx.ult` for owned units; level-ups batched to the results card. |
| **Number creep** | A level-50 ascended army trivialises Vigils; enemy levels turn into sponges to compensate. | THREAT = tier, WEIGHT = region-fixed level ≤ 39; the 35% hit cap in `hitPlayer`; `unitPower` counted into ECOST so a strong army draws a harder roster. |
| **The hero becomes a spectator** | The army kills; Hallokin watches; the "combat feel is identity" rule dies. | Army XP multiplied by the hero's style rank; CALL (`holdUlt`) requires style pips; field party ≤ 3; TOKENS merged so enemies still focus the hero. |
| **Menu game** | Six tabs × N units × 50 levels against checklist items 1-4. | Sheets live in the Barracks and at lamps only; PRESETS as one-tap builds; only ADVANCE and EVOLVE ever demand attention, and they are rites. |
| **Cream UI creeps back** | The sandbox panels are lifted as-is. | Re-token the palette before any panel ships; screenshot review with the user. |
| **Outline halos** | The ASCENDED or custom accent silhouette reads as the "ugly outline" the user removed. | Replace `olm` with a rank glyph or ground ring. |

---

## THE THREE BIG QUESTIONS

### (1) Is "no character levels" still right? Yes, for Hallokin only.

**The judgement.** v30 is a levelling system, but it levels *units*. The hero (the knight and archer, `20-player`, the feel package) sits outside HVU. `grantXp` credits only a team-1 `a.by`, and the hero's hits pass `by:null`. Every reason v1 gave for no levels is about the hero's drawing:
- frame-derived hitboxes;
- the 10-26 damage range under a 3x crit ceiling;
- percentage-based decisive verbs (execute at 34%, riposte);
- readability of damage numbers.

None of those reasons applies to a unit, whose hitboxes already scale with reach multipliers.

**Recommendation.**
- **The hero knows; the army grows.**
- **Hallokin:** no XP bar, no stat points, no level. Progress is the Roster (knowledge), techniques ranked I-III (borrow v30's rank → MASTERY grammar), charms, Laws, territory and standing.
- **Owned units:** the full v30 ladder (level, tier by rite, stats, talents, skills, gear, evolution, titles, ascension).
- **Three bridges** keep the hero central:
  - army XP in a fight is multiplied by the hero's style rank (`RMUL`);
  - an owned unit's ULT is a CALL paid in style (`holdUlt`);
  - rites and evolutions happen only at lamps the hero has lit.
- The Tally stays a hero stat and is never called a level.
- Enemy *level* is a region-fixed WEIGHT (≤ 39 outside lords). Enemy *tier* is THREAT. Neither scales to the army.

### (2) Do five political factions survive, or should the world reorganise around lineages? Politics stays; membership becomes lineage.

**The judgement.** v30's lineages cannot *replace* factions:
- `GROUPS` are sprite-pack buckets, with `custom` as a junk drawer.
- EVOLVE crosses them constantly (archer → demonarcher, axeman → ironorc, wizard → lich, lancer → pike, priest → plaguedoctor).
- Lineages have no wants, no territory, and no standing.

But v1's accent-colour membership is dead (evolution changes accents), and it leaves about 25 kinds, including every capstone and a whole demon family, homeless. The first-blow and standing systems (the spine) need politics.

**Recommendation.** Keep factions as the political layer (team ids, relations, standing, territory, first blow). Re-found each one as **the owner of a set of lineage trees, whose T5 capstones are its lords**:

| Faction | Lineages | Lords |
|---|---|---|
| **The Watch** (Hallokin's own) | watch → captain → SENTINEL (the renamed warden); knight → mirrorknight; pike → rider; swordsman → spellblade | The player's lineage: no lord, you raise them. |
| **The Gilt Order** | priest → hierophant; templar → timewarden (the Varnish made flesh); cannoneer → inquisitor | THE CHRONO TEMPLAR, THE INQUISITOR |
| **The Underdrawn** | skel / ironskel / greatskel → gravelord; bonepale → boneblade → bonedark → revenant; necro → warlock → lich | THE GRAVELORD, THE LICH |
| **The Greenhand Warband** | orc → berserker / eliteorc → warchief; axeman → ironorc; minotaur | THE WARCHIEF |
| **The Redrafters** | wizard → mesmer; harpy → siren / succubus; eyeball → ghostfire; darkwarlock | THE SIREN, THE MESMER. The charm, mark and mirror kit concentrated. |
| **NEW: the Kindled** | imp / gremlin → demonbrute / succubus; hellhound → werewolf; lavaslime → flamegolem → magmacolossus; demonlord → abysslord; blackknight, beamknight, pumpkins | THE MAGMA COLOSSUS, THE ABYSS LORD |

- **The Kindled's fiction:** lamp-ink burned too hot, the dark side of the lamps. It ties to the Warband's "burning lamp-ink as war paint". The palette is reds and oranges, colour-rule safe.
- **The Wild (not political):** slimes → kings and plaguebearer; bat → hellbat → bloodwing → stormwing; the shadow line → voidreaper (the Hatch).
- **The Hollow:** still an affix.
- **Knight's two branches become a political choice** made at a rite: templar leans Order, mirrorknight leans Redrafter. That is the one place lineage and politics touch directly.
- **Phase 2 still ships three:** the Watch, the Warband, the Wild.

### (3) The ONE idea v1 lacks: every lord is a future

**The idea.** The evolution graph is the world's map of power. Every enemy on the page is a possible future of something you own, and every lord is a capstone form. **You cannot evolve into a form you have not BESTED.** Spare a skeleton in hour two, and the Gravelord you fight in the Palimpsest at hour twelve is what it could become. Beat him (you never spare a lord), and at the Great Lamp you have lit, your skeleton takes the rite and is redrawn as him, still carrying your name and colour.

**Why it belongs in the spine.** It is the only idea that fuses all five of v1's spine items with v30's biggest system into a single loop:
- **SPARE** founds a lineage.
- **The Roster's** BESTED stamp on a capstone unlocks the branch.
- **Lamps** are where rites happen.
- **Vigils** are where the unit earns the levels.
- **Faction clashes** decide whose lords you meet and when.

It also gives every fight a question beyond "win": *is that thing my unit's future?*

**Grounding.**
- `EVOLVE`, `evolutionsFor(kind)` (the host filters by Roster stamps), `evolve(e,into)` (keeps the build), `readyTier` / `advance` / `rite` (the ceremony) and `fx.rite`.
- The sandbox evolve tab already draws locked futures as a "?" card ("a new form — at tier N"). Swap the "?" for the capstone's silhouette once SEEN, and its portrait once BESTED.
- `setLook` (the signature survives the redraw).
- `serialize` / `spawnFrom` (the save).

**Cost.** One host-side filter on `evolutionsFor`, plus the rite staged at lamps.

Runners-up, worth keeping as secondary pillars:
- **Garrisons:** an owned unit posted at a lit lamp holds it while you are away. That replaces v1's "Defend" chore and the NPC Watch being "too tired".
- **Signing:** `label` plus a custom accent = a signature; signed units cannot be hollowed.

---

## WHAT v1 GOT RIGHT THAT THE REWRITE MUST NOT LOSE

1. **The colour rule, stated harder.**
   - Full-saturation colour under crisp ink is the resting look.
   - No faded, greyed, lined or cream middle state. Blank paper is hard-edged bites, never more than a quarter of a combat frame outside the Margin, with the watched circle inside it.
   - `mono` means only the domain finale and the pause page.
   - **Extend it** to everything v30 brings: re-token the cream sandbox UI; no outline halos (replace `olm`); crisp nearest-filter sprites at integer-friendly scale so the 14-67 px pixel bodies read "high-res and easily seeable"; Hollowed contours as full-weight ink, never halo; HP bars on saturated grounds, not `PAPER`.
2. **SPARE and SEAL**, one choice with two gestures: *finish it, or give it back*. Now it is also the door to the whole army.
3. **Lamps:** colour comes from lamps; a lit lamp draws land back in with the props inking in; roadside lamps as respawn, travel and refill; Great Lamps claimed by a Vigil. Now also where rites and evolutions happen.
4. **The Vigil:** the shipped wave game kept whole as the one content type for holding land. Cards run-scoped, one kept at dawn as a charm, real stakes on death.
5. **The camera grammar:** one fixed meaning per tool, the shot director (one big shot per 60 s, lord > rival > clash > vista > killcam), killcams only on kills that matter. Add the rite to "This is yours now".
6. **The feel involvement gate:** hitstop, trauma, camera punch, killcam, callouts and numbers only when the player, the companion or a body targeting the player is involved. **Extended to owned units' hits, kills, level marks, rites and ULTs**, which the sandbox host currently fires on every kill.
7. **First blow picks the side**, with arriving neutral, one stray hit forgiven, and "pick neither". The army must hold until you choose.
8. **Difficulty is behaviour, not sponge:** THREAT from the region (now enemy tier), WEIGHT clamped (now region-fixed enemy level), the 35% hit cap, and the decisive verbs as percentages.
9. **The small, dense world:** 12 sheets at a 60-90 s crossing, a sheet loaded whole, only people streamed. The linked-arenas fallback that costs the fiction nothing.
10. **The fiction's physics:** the six forces; spawning is being drawn and death is being scrubbed; the Hollow is blankness, not darkness; the Signed as signatures; the two studies; the domain as the page held still; the Hollowed affix; the two Wardens resolved by renaming the unit.
11. **Phase 0 gate 5 decided by the user's hands.** Does it still feel like Hollow Vigil? Now tested with an army fielded. Plus Phase 0.5: ship the Roster, SPARE and now owned units into today's wave game before any world exists.
12. **The code hygiene lessons:** new world code written multi-line (the swallowed-`//` bugs); the field-snapshot test for reused bodies (now more urgent, since `evolve` and `charm` mutate `team`, `cfg` and `tier`); saves only at safe moments; performance judged on a real laptop, not SwiftShader.
13. **Rivals remember**, and the Mesmer as the villain of allegiance.
14. **The two story bosses kept outside the unit system** (the Warden slime, the Hollow Knight), so they can never be owned or evolved into.

---

## 10-LINE SUMMARY

Core finding: v1's world, look and verbs survive, but its progression and faction foundations were written for units that never grow. v30 makes SPARE the door to a full army RPG, so "no levels", the one-follower party, the accent-colour factions, the six custom-body lords, health-threshold ULT beats and the Bond ladder are all broken or obsolete. Two v30 artefacts also violate the colour rule: the cream sandbox UI and the accent-outline halos.

1. **The hero knows, the army grows.** Hallokin keeps no levels (Roster, techniques, charms, Laws). Owned units get the full v30 ladder. Bridge the two with style-scaled army XP (`grantXp` × RMUL), CALL through a `holdUlt` flag, and rites only at lit lamps.
2. **Keep five-plus factions as politics, but re-found membership on EVOLVE lineages** whose T5 capstones (gravelord, lich, timewarden, magmacolossus, abysslord…) are the region lords. Add a sixth power, the Kindled, for the homeless demon family. The Watch becomes the player's own lineage.
3. **Make "every lord is a future" part of the spine:** you evolve only into forms you have BESTED. That one host filter on `evolutionsFor` fuses SPARE, the Roster, lamps, Vigils and the bosses into one loop. Pair it with non-negotiable engineering items:
   - the involvement gate extended to army fx;
   - an `owned` flag separate from team 1;
   - `team:2` on every hostile spawn;
   - no Warden, Hollow King or cream carried forward.
