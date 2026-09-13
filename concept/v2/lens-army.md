# Lens: PROGRESSION AND THE ARMY (Folio v2)

*Design pass against HVU v30 (`hv-work/units30/units.js`). Every number below was read from the code or computed by loading the module in node (lines 60-1639 of units.js plus the META and SHEETS lines, no game code). Scripts: the scratchpad `a.cjs`..`d.cjs` for this session; the method is reproducible in 20 lines.*

## 0. Core finding

**v30 already draws the line v1 needed: the hero does not level and the army does.** In the v30 demo `heroHitUnits` calls `HVU.hurt` with no `by`, so Hallokin's kills grant nothing, and `grantXp` only accepts team 1 units. And v30's two enemy dials map one-to-one onto v1's two difficulty dials: **tier is THREAT** (`tierHas`: dash/block at T2, passive/special at T3, ULT and smart AI at T4) and **level is WEIGHT** (+2% hp and damage per level in `applyMods`). So v2 should adopt the split whole, and fix the three places where the raw v30 numbers break the game:
1. **Pacing.** Killer-only XP takes a fielded unit to tier 5 in about 55 hours.
2. **Scale.** A tier-5 capstone has 3000+ HP next to a 100-HP hero.
3. **Dead or leaky parts.** PATHS is unreachable, spawn's default team is the player's side, and ascension is impossible for about a quarter of the kinds.

---

## 1. The real numbers (read from the code)

| Table / function | Value (units.js line) |
|---|---|
| `XP_TO(l)=round(12*l^1.35)`, `MAX_LEVEL=50` (349) | L1 12 · L5 105 · L10 269 · L20 685 · L30 1184 · L49 2296 |
| Cumulative XP to reach a level | L5 **174** · L10 1012 · L12 **1587** · L20 **5491** · L30 **14528** · L40 **28848** · L50 **49028** |
| XP per kill (1027) | `max(6, round(power(victim.cfg)/8))`, to the killer only, if killer.team===1 and victim.team!==1. **It ignores the victim's tier and level.** Also `renown+=1`. |
| XP per kill by kind | 6 for 27 kinds (slime, skel, watch, knife, orc, wizard, mesmer…). knight 11, blade 12, warchief 41, gravelord 54, abysslord 77 |
| Average XP per kill for a region's roster (kinds with unlock in t-3..t, non-boss) | r1-2: 6.0 · r3: 6.5 · r5: 8.2 · r7: 10.5 · r9: 13.5 · r10: 16.4 · r12: 18.8 |
| `TIER_AT_LEVEL={5:2,12:3,20:4,30:5}` (349) | Only flags a tier as ready. `readyTier` + `advance` (= `advanceTier`, 371) takes it with `rite()` |
| `POINTS_AT(t)` {0,2,4,6,9}, +1 per 2 levels (`levelStatPoints`), `STAT_CAP=8` (293, 356) | T3 L12 = 9 pts · T5 L30 = 23 · T5 L50 = **33** of the 48 needed to max all six stats |
| `SKILL_POINTS_AT(t)` {1,3,5,7,9}, +1 per 5 levels (330, 357) | T3 L12 = 7 · T5 L30 = 14 · L50 = 18. Rank costs 1 per rank, a mastery 2. Kits are 1-4 moves (skel 1, knight 2, gravelord 3, necro 4) |
| `skillOf` (347) | per rank: +10% dmg, -8% cd, +5% reach. L30 gives +1 free rank (max 3), ascended +1 (max 4) |
| `TALENTS[2..5]` (295) | 3 options per tier, 1 pick each |
| `MILESTONES` (350) | 10 VETERAN +5% · 20 HARDENED -8% taken, 2nd artifact · 30 MASTER free rank · 40 UNIQUE encore+warlord, "rises once per fight" · 50 MYTHIC +10% all |
| `TITLES` by `unitPower` (351) | ELITE 150 · CHAMPION 300 · WARLORD 500 · LEGEND 800 |
| `ascendAt(e)=max(1400, 1.3*fullBuildPower(kind))` (367) | Renown gives up to +60% base and +15% via mods |
| `ITEMS` (393) | 10 artifacts, 3 weapons, 3 trinkets. Slots: 1 weapon, 1 trinket, artifacts `artifactSlots(e)` = 2 at T5 **or L20**, else 1 |
| `EVOLVE` (301) | 53 source kinds (11 branch). The capstones gravelord/timewarden/warchief/lich/voidreaper need T5 |

**Power samples** (my spawnFrom builds with greedy stats, ranks, talents and gear):

| Build | unitPower | HP | Title |
|---|---|---|---|
| skel T1 L1 | 29 | 42 | - |
| skel T3 L12 | 101 | 86 | - |
| **ironskel** T3 L12 (same build, evolved) | **215** | 197 | ELITE |
| greatskel T4 L20 | 745 | 408 | WARLORD |
| **gravelord T5 L30** | **5016** | **3089** | LEGEND |
| gravelord L40 | 8116 | 3648 | UNIQUE |
| gravelord L50 | 10366 | 5604 | ASCENDED |
| watch T3 L12 | 119 | 142 | - |
| knight T3 L12 | 345 | 339 | CHAMPION |
| templar T4 L20 | 880 | 510 | LEGEND |
| timewarden T5 L30 | 3286 | 1844 | LEGEND |
| timewarden T5 L50 | 6791 | 3345 | ASCENDED |
| blade T5 L30 / L50 | 1115 / 2304 | 991 / 1798 | LEGEND / ASCENDED |
| knife T5 L50, full build | **762** | 210 | UNIQUE, **can never ascend** (floor 1400) |
| bare warchief T5 L1 | 682 | 829 | WARLORD with no build at all |

Three things follow from this table:
- **Evolution, not levels, is where power comes from.** The same build doubles from skel to ironskel. The T5 capstone step is roughly ×4-7.
- **Titles are a kind label, not a progress label, on big bodies.** A naked warchief is already WARLORD.
- **At T5, owned units live on a different HP scale from Hallokin** (100 HP, 10-26 damage).

---

## 2. Reconcile "no character levels" with v30: sharpened, not replaced

**Decision: Hallokin never levels. The army carries every RPG number. Hallokin's growth is COMMAND, the right to field army power, and it comes from territory and knowledge, not XP.**

### 2.1 Why the candidate survives the code test
- **v30 already says it.** The demo hero is team 1 and deals damage through `HVU.hurt(e,{...})` with no `by`, so no XP, and it has no sheet. The guide line 51 ("Hallokin is team 1 … owned units never target him, his swings never hit them") treats him as the army's commander, not a member.
- **The tuned combat stays honest.** The knight's 10-26 damage, the 34% execute, stun by hit count and the swap-weave are all fixed numbers. If Hallokin had `+2%/level` for 50 levels (×1.98) plus MIGHT 8 (×1.32), every feel number would drift.
- **v30's enemy dials ARE v1's THREAT/WEIGHT split.** v1 §7.4 wanted threat (wind-up, abilities live) separate from weight (hp, damage). In v30:
  - **THREAT = `tier`** via `tierHas` (422): `{dash:2,charge:2,block:2,hop:2,band:2,special:3,nova:3,heal:3,summon:3,passive:3,ult:4,smartAI:4}`, plus the cd multiplier `M.cd` 1.3/1.15/1/0.92/0.85.
  - **WEIGHT = `level`** via `applyMods` (428-429).
  - v1 did not have to invent these dials; they already exist.

### 2.2 Why it could fail on fun, and the fix
**The risk:** at hour 15 your gravelord is a LEGEND with 3089 HP while the figure in your hands is exactly as strong as at minute one. The player may feel like a spectator of his own army.

Three rules keep the hands central:
1. **THE SHARE: Hallokin's kills are the army's XP engine.** Every hostile kill inside the encounter panel pays each alive fielded unit 40% of that victim's XP value (`max(6,power/8)`), × `RMUL[style rank]` at the moment of the kill. The host calls `HVU.grantXp(u, n)`, which is exported. The module's own grant to the killer still happens on top. So **fighting well with your own hands levels your army faster**, and the style rank becomes the army's XP multiplier. (Model in §5.)
2. **COMMAND caps fielded power.** The fielded units' summed `HVU.unitPower(u)` must be ≤ COMMAND. COMMAND is Hallokin's only "number", and it grows from Great Lamps lit and Roster stamps (the v1 Tally), never from XP (§4.3). This makes the capstone chase a *fielding decision*, and it stops the army from trivialising the combat.
3. **CALL stays Hallokin's verb** (v1 §6.2): at A and SS rank, hold Q to fire a fielded T4+ unit's ULT through `HVU.ULT[kind].run(u, target)` (ULT is exported). The army's biggest moments are triggered by the hands.

Hallokin's own growth list is unchanged from v1 and deliberately flat: techniques from teachers, traits, charms, one Law. Everything that says "+%" lives on units.

### 2.3 Enemies: tier by region, level capped as WEIGHT
World enemies spawn through `HVU.spawnFrom({kind,tier,level},x,z,{team:2})`. Use `spawnFrom`, not `spawn`, because `spawn` ignores `level` (947-972).

| Region tier (CFG `unlock` band) | Enemy tier (THREAT) | Enemy level (WEIGHT) | HP multiplier from `applyMods` |
|---|---|---|---|
| 1-2 (Ashridge, Vale) | T1-2 | 1-3 | 1.00-1.04 (×1.15 cd at T2) |
| 3-4 | T2-3 | 3-6 | ~1.05-1.10 |
| 5-7 | T3-4 | 6-10 | ~1.10-1.30 |
| 8-10 | T4 | 10-15 | ~1.30-1.40 |
| 11-12 (Margin) | T5 | 15-20 | up to **1.59** (L20 ×1.38, VETERAN ×1.05, T5 ×1.1) |
| Lords / Signed duels | T5 | 20 | 1.59, and HARDENED -8% damage taken |
| Endless Vigil (post-game) | T5 | climbs past 20 | uncapped |

**L20 lands exactly on v1's WEIGHT clamp of 1.6**, so the main game caps enemy level at 20. Enemies never level during a fight: `grantXp` refuses team 2.

---

## 3. Acquiring units: SPARE → BIND produces a build object

### 3.1 The Roster gates stay, and gain a job
v1's four stamps stay: SEEN / STUDIED / BESTED / BOUND. Two of them now gate army actions:
- **STUDIED** (you answered every move once) gates **sparing** that kind *and* **evolving INTO** it. You must know a drawing before you can redraw yourself as it. To evolve a greatskel into a gravelord you must first have *fought* a gravelord (a Margin lord) well enough to fill its page. The Roster becomes the evolution tree's key ring, and every capstone becomes a boss worth studying.
- **BESTED** still unlocks Hallokin's techniques (v1 §6.3, unchanged).
- **BOUND** = you own at least one unit of the kind.
- New gold rim on a page when any unit you own of that kind reaches ASCENDED. It is cosmetic, but it is the long-tail completion mark.

### 3.2 What binding writes
At the SPARE prompt (hold attack 0.4 s), the kneeling unit `e` is removed and the host writes a fresh build, **not** `HVU.serialize(e)` (that would copy an enemy's tier 5 and its level):

```
{ kind: e.kind, tier: bindTier, level: FLOOR[bindTier], xp: 0, renown: 0, ascended: false,
  items: [], stats: {}, skills: {}, talents: {}, loadout: null, label: <player-inked name or null>, tint: null }
```
with `FLOOR = {1:1, 2:5, 3:12, 4:20}`, the inverse of `TIER_AT_LEVEL`, and **`bindTier = max(1, enemyTier − 1)`**, capped at 4.

- **Does sparing a tier-3 enemy give a tier-3 unit? No, a tier 2 at level 5.** Fiction: a spared figure is redrawn and loses its last stage of ink (§3.4). Mechanics:
  1. **It keeps evolution meaningful.** A T4 bind skips 5491 XP. A T5 bind would skip the whole capstone chase, so capstone kinds and lords can't be bound at all (lords are never spared, v1 §9.6).
  2. **It is catch-up.** A late bind in region 8 (T4 enemies) arrives at T3 L12, which is fieldable at once. Model: a T3 L12 bind at hour 14 reaches T5 by hour ~24 (§5).
- **Unspent points are the reward screen.** `statBudget` and `skillBudget` are live, so a T2 L5 bind arrives with 4 stat points, 3 skill points and a T2 talent pick. Binding opens the character sheet: this is where the player *makes* the unit.
- **The Signed** (knife, blade, lancer) are bound after their duel at their duel tier − 1 (T4 L20). The v1 prize stays: their PASSIVE becomes Hallokin's trait whether or not you spare them.
- **Sealed bodies** (skeletons, the Hollowed; v1 SEAL) bind the same way. The Hollowed come back as their pre-hollow kind.
- **Never use `HVU.spawn(kind,…)` for a bound unit.** `spawn` defaults `tier` to 5 (`e.tier=…:5`), which gives the full kit and `POINTS_AT(5)=9` stat points at level 1. Always go through `spawnFrom(build)`.

### 3.3 Owning many: the Barracks and garrisons
**Owned cap = 6 + 2 × Great Lamps lit** (max 30 at 12 lamps). Territory is army size.

- **Fielded:** the Lance (§4).
- **Posted:** each lit Great Lamp holds up to **2 garrison units**. They are spawned at the lamp during that lamp's Defend Vigil (v1 §5.5) and earn XP there, so bench units level in the content that already exists.
- **Resting:** in the Barracks, not in the sim. Just a build object.

The demo's army UI (`#dock`, `#roster`, `#lab`, `#sheet` with tabs OVERVIEW/STATS/SKILLS/TALENTS/GEAR/EVOLVE, `renderSheet` 1851) is the Barracks screen, already built. **It must be re-skinned:** `#dock` is `rgba(236,229,212,.9)` cream over an ink rule, which is the paper look the user rejected three times.

### 3.4 Fiction: tiers are stages of a drawing
v1's ink-in has three held drawings (line, under-stroke, colour). v30's tiers extend it:

| Tier | Stage | What `tierHas` turns on |
|---|---|---|
| **T1** | **the line** | first move, plain AI |
| **T2** | **the under-stroke** | dash, block, second move |
| **T3** | **the colour** | passive, special |
| **T4** | **the shadow** | ULT, smart AI |
| **T5** | **the signature** | second artifact, ×1.1, cd 0.85 |

A rite (`advance`, fx `rite`) is Hallokin inking the next stage onto the figure, so the host renders `fx.rite` as that stage's ink-in. Sparing scrubs one stage off. Evolution redraws the figure as a relative.

---

## 4. Army size and shape

### 4.1 Fielded counts, against the ~24 awake cap
Bodies are counted as `HVU.units` plus the game's `enemies[]` inside the hot bubble. **Army-side summons count as army bodies**:
- the necro's summon, `MASTERY.summon` LEGION (+1),
- the CONJURER path's `pathRise`, which raises "two of its family every 9s (up to four)",
- the pumpkin/warlock/lich kits.

| Occasion | Hallokin + companion | Fielded units | Army body cap (incl. summons) | Hostile bodies | Total |
|---|---|---|---|---|---|
| Open world / roaming skirmish | 2 | **3 (the Lance)** | 6 | ≤ 14 | ≤ 22 |
| Faction clash (you picked a side) | 2 | 3 | 6 | ≤ 10 per side, with allied faction bodies on your side | ≤ 24 (ledger resolves the rest off-frame, v1 §7.1) |
| Vigil (claim / defend) | 2 | 3 + **2 garrison** at the lamp | 8 | ≤ 11 per wave (`composeWave` cap is already `min(11,…)`) | ≤ 23 |
| Lord / story boss | 2 | **2** | 3 | boss + adds ≤ 8 | ≤ 15 |
| Signed duel | 1 | **0** (they wait at the panel gutter) | 0 | 1 | 2 |
| Endless Vigil | 2 | 3 + 2 | 8 | ≤ 13 | ≤ 24 |

- **Why 3 and not more.** At 3 the eye can track the Lance as three accents. Past that, v1's "mud at scale" risk (checklist items 1, 2, 7, 9, 10) returns.
- **Why 2 in a boss fight.** The boss framing and telegraphs must read. Two allies still let CALL matter.
- **Why 0 in a duel.** A Signed duel is a test of the hands. v1's promise that "a rival who remembers you" is a fight you win yourself stays true.
- **The companion stays outside the army.** She is not a build, never levels and is not counted in COMMAND. She is the story partner and the MARK executor (v1 §6.2).

### 4.2 Readability rules for owned units
- **Owned units (team 1):** the thin accent underline (v1 §7.2) in `customAccent || cfg.accent`, and one small tier pip above the head: 1-5 dots, gold at UNIQUE/ASCENDED.
- **Never** the demo's `'T'+tier+' L'+level+' P'+power` overhead text (2116). The name and title show only on the selected unit or while the Barracks is open.
- **No damage numbers, trauma or hitstop on unit-on-unit hits** (v1's involvement gate). Unit ULTs appear as small balloons, not stamps.
- **Rites are deferred out of combat.** `grantXp` itself fires `rite()` at milestone levels 10/20/30/40/50 and on `checkAscension`. `rite` sets `poise=0` and `noMoveT=1.3`, which freezes the unit mid-fight, and `fx.rite` fires. In the host `fx.rite`, only queue a mark during an encounter, and play the ink-in at the next quiet moment. Tier `advance` and `evolve` are only offered at a lit lamp (§6), never in a fight.
- **Level-up heals 25%** (`grantXp` 376). Early levels come every 2-3 kills, so during the first hour a fielded T1 unit is quietly regenerating. That is acceptable, but the Share must not pay XP to units above the COMMAND budget.

### 4.3 COMMAND: what can be fielded
`sum(HVU.unitPower(u) for fielded u) ≤ COMMAND`. COMMAND comes from Great Lamps lit, plus 5 per Roster stamp (the v1 Tally, about 150 stamps, so up to +750):

| Great Lamps lit | 0 | 2 | 4 | 6 | 8 | 10 | 12 | post-game |
|---|---|---|---|---|---|---|---|---|
| COMMAND (lamps part) | 150 | 400 | 900 | 1800 | 3000 | 5000 | 8000 | uncapped |
| What fits (from §1 samples) | three T1-T2 units (29-42 each) | three T2-T3 | knight 345 + ironskel 215 + a T3 | templar 880 + greatskel 745 | three T4 / a blade T5 1115 + two T4 | **one gravelord 5016**, or timewarden 3286 + a templar | a gravelord + a timewarden | the whole Lance at any power |

Good things fall out of this:
- **The late game asks "one LEGEND capstone or three WARLORDs?"** That is a real, visible choice.
- **Stat builds get a Command price.** `unitPower` reads only hp, dmg and tf mods. VIGOR, MIGHT and HASTE cost COMMAND; SWIFT, REACH and WILL are free. **A WILL/REACH tank is cheap to field.** Tank builds get a natural niche instead of a tax.
- **Encounter budget:** each Vigil's `composeWave` budget and each clash's size add `0.4 × fielded power / 100` in ECOST. The extra buys **tier and affixes, not bodies** (the body caps above are fixed), so a stronger army meets meaner fights, not bigger crowds.

A unit over COMMAND can still be bound, levelled at garrisons and posted. It just can't walk with you yet.

---

## 5. Pacing over a ~20-hour playthrough

### 5.1 The kill model
- **Kill rate.** v1's hour-15 session (§5.3) holds about 70 hostile kills in 30 minutes (a 12-body clash, a 6-wave Vigil, skirmishes), so **140 kills/hour** of total play. That already includes walking.
- **Region curve.** The player reaches region tier t at about 1.67·t hours, so tier 12 by hour 20.
- **XP per kill** is the region's average from §1: 6.0 at the start, 18.8 at the end.
- **The fielded unit** lands about 12% of kills itself. It is bound at hour 0.5 (v1: SPARE unlocks at ~30 min).

Hours (and the unit's own kills) to reach each level:

| Scheme | L5 (T2) | L12 (T3) | L20 (T4) | L30 (T5) | L40 UNIQUE | L50 / ASCENDED |
|---|---|---|---|---|---|---|
| **Raw v30** (killer only, 140/h) | 2.2 h / 29 kills | 12.3 h / 198 | 25.9 h / 427 | **54.5 h** / 908 | never | never |
| Raw v30, double the kill rate (260/h) | 1.4 h | 7.9 h | 17.8 h | 33.3 h | 57.7 h | never |
| Share 64% of kills split over 3 | 1.1 h | 5.8 h | 14.2 h | 25.1 h | 41.4 h | never |
| **THE SHARE: 40% of every kill to each fielded unit** (≈ tithe 1.28/3) | 0.9 h / 6 | 3.9 h / 57 | 10.2 h / 163 | 18.6 h / 305 | 28.6 h | 42.6 h |
| **THE SHARE × RMUL (average rank ≈ B, ×1.3)** | **0.8 h** | **3.2 h** | **8.4 h** | **16.1 h** | **24.0 h** | **34.7 h** |
| Late bind T2 L5 at hour 10 (Share, no RMUL) | - | 11.8 h | 15.7 h | 22.2 h | 32.2 h | 46.2 h |
| Late bind T3 L12 at hour 14 | - | - | 17.3 h | 23.6 h | 33.6 h | 47.6 h |

**Verdict: the raw formula does not fit.** On killer-only XP a fielded unit ends a 20-hour game at about L16 (tier 3). No capstone evolution and no UNIQUE are reachable in the playthrough, and v30's whole top half would be unreachable content. XP_TO itself is fine: it is the *income* that is wrong. So do not touch the author's tables. Add **THE SHARE × RMUL in the host** (one `grantXp` loop in the game's kill handler). That gives the curve we want:

| Step | When | What it means |
|---|---|---|
| **T2** | first hour | first talent and path pick: the unit becomes *yours* |
| **T3** | hour 3 | first branch evolution (skel → ironskel/skelarcher, watch → captain/knight, orc → berserker) |
| **T4** | hour 8-10 | ULT online: CALL becomes available on that unit; second evolution |
| **T5** | hour 16 | capstone evolution, lined up with Chapter III-IV and 10-12 lamps of COMMAND |
| **UNIQUE (L40)** | hour 24 | just after the credits: the post-game's first chase |
| **L50 / ASCENDED** | hour 35+ | the Endless Vigil's long tail |

### 5.2 Ascension, in practice
`ascendAt = max(1400, 1.3·fullBuildPower(kind))`, and renown multiplies base power by up to 1.6 and mods by 1.15, reached after only 60 kills. In my simulations:
- **gravelord:** L40 with 60 renown is 8116 vs 9166, not yet; **L50 ascends** (10366).
- **timewarden:** L30 3286 vs 6005, no; **L50 ascends**.
- **blade:** ascends at **L50**.

So **ASCENDED ≈ L50 + a finished build**, about 35 hours. Right for an endgame, and the renown part is trivial (60 kills).

**Three code facts the design must respect:**
1. **27 kinds can never ascend** at any build (simulated with a full L50 build, 60 renown and top gear): knife watch orc hierophant mesmer skel skelarcher archer wizard priest bat demonarcher gremlin eyeball ghostfire hellbat lavaslime slime slimeblue slimepink bonearcher2 pumpkin pumpkinred nightfang, plus the non-recruitable slimelet, blob and mirrorimage. The 1400 floor is the cause. They include slime, bat, skel, watch, archer, wizard, priest, orc, mesmer, hierophant and **THE KNIFE** (762 at a full L50 build, simulated). They must *evolve* to ascend. That is fine for skel → gravelord, but THE KNIFE is a prize recruit. Tell the player on its sheet: "evolves to verdagger at T4 to ascend".
2. **Ascension depends on stat choice.** A gravelord at L50 with the same items and ranks ascends with VIGOR/MIGHT/HASTE/WILL 8 (10865), but **not** with HASTE/SWIFT/REACH/WILL 8 (8993 < 9166). This is the flip side of §4.3's cheap tank. Say so on the sheet ("SWIFT, REACH and WILL do not count toward ascension").
3. **`fullBuildPower` assumes 48 stat points** that no unit can ever have (33 max). `setStats_` truncates them in STATS key order (vigor, might, haste, swift, reach). The ascension mark is therefore "the best might-first build at L50 ×1.3", which is still reachable. **`ascended` is sticky:** `setStats` / `setSkills` can trigger ascension and nothing ever clears it. So respec to a might build, ascend, respec back. Either allow that knowingly or lock respec on ASCENDED units. **Recommendation: lock stats once ascended** ("the signature is dry").

---

## 6. Evolution as the long-term chase: choices, not a checklist

### 6.1 What the graph actually is
- **Evolution sources.** `EVOLVE` has 53 source kinds: **11 branch** (two options) and 42 have a single road.
- **Convergence points** (several roads into one capstone):

| Capstone | Reached from |
|---|---|
| **warchief** | berserker / eliteorc / ironorc |
| **lich** | wizard / warlock / darkwarlock |
| **voidreaper** | noctislicer / shade / frostfiend |
| **succubus** | harpy / imp / bloodfiend |
| **revenant** | ironskel / bonedark |
| **spellblade** | swordsman / blade |
| **plaguebearer** | slimepink / toxcrawler |
| **verdagger** | nightfang / knife |

- **The two flagship chains are exactly one evolution per tier from T3:**
  - **gravelord:** skel (T1) → ironskel (T3, L12) → greatskel (T4, L20) → gravelord (T5, L30).
  - **timewarden:** watch → knight (T3) → templar (T4) → timewarden (T5).
  - Both have branches off them: skel → **skelarcher** at T3; ironskel → **revenant** at T5 (skipping greatskel); watch → **captain** → warden (the v1 SENTINEL); knight → **mirrorknight** at T5 (skipping templar).

**Code facts that turn this into a checklist unless the host intervenes:**
1. **`evolve` is free and instant once the tier is met, and it can chain.** Simulated: a knife at T5 L30 went knife → verdagger → duskclaw → noctislicer → **voidreaper** in four calls in the same frame.
2. **`evolve` wipes every skill rank and mastery.** No two kinds share move ids (skel `skA1`, ironskel `asA1,asA2`, gravelord `glA1,glA3,glStorm`), and `setSkills_` keeps only ids in the new kit. The guide's "keeps … skills" is true only of the *budget*, which is refunded. Stats, talents, items, level, renown and label do carry.
3. **`evolve` replaces the object:** it calls `host.onRemove`, splices, then `spawnFrom`, so there is a new `id`. The host's leash, selection and garrison references must re-point to the returned unit.
4. **`b.loadout=null`** on evolve. Loadouts are per-kind.

### 6.2 The rules that make it a choice
- **One redraw per rite.** When `readyTier(u)` is set and you are at a lit Great Lamp, the rite offers **either** `HVU.advance(u)` (ink the next stage) **or**, if the unit's tier already meets an `evolutionsFor(u.kind)` entry, `HVU.evolve(u, into)`. It never offers both, and evolving marks that tier's rite as spent.
  - The consequence: a unit that stops to evolve at T3 advances to T4 one rite later, so evolution costs tempo.
  - Chains like knife → voidreaper take four rites, which are only offered as T4/T5 are reached. For the T5 steps, gate on **one evolve per 10 levels past 30**: L30, L40, L50.
- **The STUDIED gate** (§3.1): you can only evolve into a page you have studied. The capstones are Margin and region lords, so **the chase sends Hallokin to fight the thing his skeleton wants to become.**
- **Branches are identity decisions** made at T3, where a unit has 9 stat points, 7 skill points and 2 talents: early enough to matter for 13 more hours.
  - skel → ironskel (melee line to gravelord) or skelarcher (ranged, dead-ends at bonearcher2 T4 but fields cheaply).
  - watch → captain (a squad caller: rally, volley) or knight (the timewarden or mirrorknight road).
  - orc → berserker at T3, *or wait* for eliteorc at T4. Waiting is a real option, because both converge on warchief.
- **Evolution refunds skills, and that is the respec moment.** Present it as "the new body has new moves: spend your points again" and open the SKILLS tab automatically. The MASTERY pair per move type (`MASTERY[moveType(id)]`: CLEAVE/RUPTURE, IMPALE/RUN THROUGH, TWIN/PIERCING, LEGION/HARDY, LONG/SEAR…) is re-chosen for the new kit.

### 6.3 Where the decisions live, by tier

| When | Decision | Real trade (from the tables) |
|---|---|---|
| Bind | Which kind, and its name (`label`) | Kit size: skel has 1 move, necro 4 |
| T2 (hour 1) | **PATH** (VANGUARD/DUELIST/REAVER or MARKSMAN/CONJURER/HEXER) + T2 talent (BRUISER/FINISHER/LIFELINE) | CONJURER adds up to 4 bodies, which count against the army body cap (§4.1). **Needs a module patch, §10.** |
| T3 (hour 3) | Branch evolution; talent OPENER/BULWARK/MOMENTUM | Identity. The COMMAND cost roughly doubles on the stronger branch (skel 101 → ironskel 215) |
| T4 (hour 9) | ULT talent EAGER/BRUTAL/ENCORE; second evolution; CALL | ENCORE vs the L40 UNIQUE flag, which also grants encore: ENCORE is wasted on a unit you plan to take to L40 |
| T5 (hour 16) | Capstone; JUGGERNAUT/DUELIST/WARLORD; 2nd artifact | COMMAND: is it one gravelord, or keep three T4s? |
| Every level | 1 stat point per 2 levels, cap 8, 33 max of 48 | Command-costing stats (VIG/MIG/HAS) vs free ones (SWI/REA/WIL); ascension reads only the first three |
| Rank 3 | MASTERY a/b | Wiped on evolve |

- **PRESETS are drill manuals, not builds.** The five PRESETS (BULWARK, BLADE DANCER, REAVER, DEADEYE, GRAVE VOICE) are an **AUTO** button for players who don't want the sheet.
  - Their stats sum to exactly 9 = `POINTS_AT(5)` and every one has `path:null`, so they are shaped for a tier-5 level-1 body. Applied at T2 the budget truncates in key order: BLADE DANCER's `might:5,haste:4` becomes might 2. At L30 it leaves 14 of 23 points unspent.
  - The host should scale a preset's stat ratios to `statBudget(u)` and add a path per preset (BULWARK → VANGUARD, BLADE DANCER → DUELIST, REAVER → REAVER, DEADEYE → MARKSMAN, GRAVE VOICE → CONJURER).
  - The demo's preset button (1891) also does `u.items=[]` and re-equips the preset's items. That would **delete gear the player found**, so the game must not copy it.

---

## 7. The economy: what goes to units and what goes to Hallokin

**Principle: gear and every percentage go to units. Hallokin buys verbs.** The v1 rule "loot never changes a drawing" is kept by moving all loot off the hero.

| Currency | Earned by | Spent on HALLOKIN | Spent on UNITS |
|---|---|---|---|
| **Ink** (common, v1 §6.6: base × kind power × `RMUL`) | every kill; banked at lamps | technique ranks I-III; town projects; lamps in the Margin | **the 3 weapons and 3 trinkets** in `ITEMS` (KEEN EDGE, HEAVY HAFT, LONG REACH; SWIFT BOOTS, LUCKY COIN, SECOND SKIN), bought at the Smithy |
| **Quills** (rare: bounty clauses, Crowned executions, first STUDIED stamps) | | charm ranks; Vows | unlocking a PATH respec (paths are otherwise permanent) |
| **Trophies** (never bought) | Crowned elites, lords, Signed duels | Laws (lords), traits (Signed) | **the 10 artifacts** |

**v1's Inks become unit artifacts.** You still "take the affix off the thing that kept healing on you", but you hand it to a unit. Mapped from the game's `AFFIX` (19b-archetypes 258) onto v30 `ITEMS`:

| Crowned affix | Artifact |
|---|---|
| VAMPIRIC | LEECH FANG |
| VOLATILE | EMBER CORE |
| WARDED | WARD STONE |
| ARMORED | IRON ROOT (or THORN MAIL from an ARMORED melee elite) |
| SWIFT | HOURGLASS |
| HEXED | VENOM GLASS / FROST SEAL |

Lords drop the two that don't map: GIANT HEART (the Warden) and WAR BANNER (the Warchief).

**Rules:**
- **Artifact slots follow `artifactSlots(e)`:** 1, or 2 at T5 **or L20**. Don't use the `ARTIFACT_SLOTS(t)` helper, which is tier-only and not exported (§10). One weapon, one trinket. `equip_` silently swaps out the oldest item of the same slot, so the Barracks UI must confirm before a swap.
- **Items are never consumed and are movable between units** at a lamp (`unequip` / `equip`). Sixteen items is a small pool, so a unit's identity must come from kind, branch, path and talents, not a loot treadmill.
- **Respec at a lit lamp:** stats and skills are free (`setStats` / `setSkills` validate budgets). Talents and path cost a Quill. Evolution and branches are permanent, and ascended stats are locked (§5.2).
- **Rites and evolutions cost nothing but the lamp visit.** The cost is tempo (§6.2) and COMMAND. The army's economy should not be a second grind on top of XP.

---

## 8. When a levelled unit dies

**Decision: SCRUBBED, not erased, except on blank paper.** Death is erasure in the fiction (v1 §3.3), so the rules follow *how much* of the drawing was lost.

- **The save is the build object, and it is taken at safe moments.** The host keeps `build = HVU.serialize(u)` for every owned unit and refreshes it at every lamp bank and every out-of-combat level-up. A unit in the sim is a *copy*.
- **Killed in an ordinary fight → SCRUBBED.** The body plays its `deathKind` scrub-out and leaves the Lance for the rest of the encounter. At the next lit lamp it is **redrawn from its saved build**: `spawnFrom(build)` with the ink-in.
  - It loses the unbanked `xp` since the last bank. **Level is never lost** (`level` is in the build).
  - It loses 10 `renown`, clamped at 0: the figure's reputation smudges. Renown is the cheap part of ascension, so this is a sting, not a setback.
  - v1's revive stays: a downed unit kneels, and a 1.5 s stand beside it (the lamp-relight gesture) brings it back at 30% hp with nothing lost.
- **Killed on blank paper, or by a HOLLOWED body → HOLLOWED.** The unit is un-drawn and its page in the Barracks goes line-only. It reappears later in that region as an enemy: **`HVU.spawnFrom(build, x, z, {team:2})` with the HOLLOWED affix, carrying your own build, level, talents and artifacts.**
  - **SEAL it** (v1's sigil loop around the stunned body) and it comes home, redrawn at its build.
  - **Kill it** and it is gone for good. That is the one permadeath, and it takes two failures, the second one fought against your own gravelord's kit.
  - This is v1's "Crowned elite takes your ink and grows a grudge", promoted to the army.
- **In a Vigil:** a dead unit is scrubbed for the night. If the Vigil is lost, garrison units at that lamp are scrubbed too, and the lamp stays dark (v1 stakes).
- **Why not straight permadeath:** a gravelord is about 16 hours of play. Losing it to one mistimed boss slam punishes the army for Hallokin's hands, and players would bench their best units, which kills the chase.
- **Why not free rebuilds:** a death with no sting makes the army decoration.

The Hollowed rule gives the Margin's blank paper a cost that only an army player feels, and it uses only the build object v30 already provides.

---

## 9. What survives from v1, what changes, what is cut

| v1 system | v2 verdict | Why / what backs it |
|---|---|---|
| **"No character levels"** (§6, §9.3) | **KEPT for Hallokin, reworded:** "Hallokin never levels. The army does." | §2. The demo hero already has no XP; v1's THREAT/WEIGHT split is v30's tier/level |
| **The Tally** | **KEPT**, now feeds COMMAND (+5 per stamp) instead of offsetting enemy weight | Enemy WEIGHT is now region-set enemy level ≤ 20 (§2.3) |
| **Roster: SEEN/STUDIED/BESTED/BOUND** | **KEPT, extended.** STUDIED gates spare *and* evolving into a kind; a gold rim for an ASCENDED owner | `evolutionsFor` + host gate |
| **SPARE / SEAL / BIND** | **KEPT.** Bind writes a v30 build at enemy tier − 1, at the floor level for that tier (§3.2) | `spawnFrom`, `statBudget`, `skillBudget` |
| **Party: companion + 1 follower (field), 2 (Vigil)** | **REPLACED:** companion + Lance of 3 (field), 3 + 2 garrison (Vigil), 2 (lord), 0 (duel); COMMAND caps power | §4 |
| **CALL** (hold Q for a follower's ULT at A/SS) | **KEPT**, on any fielded T4+ unit (`tierHas(u,'ult')`) | `HVU.ULT[kind].run` |
| **Bond: Bound / Sworn / Oathbound** | **CUT.** Levels, titles and MILESTONES are the bond | Oathbound's "wear its PASSIVE as a trait" moves to the T5 rite: while fielded, a T5 unit lends Hallokin its passive as a 4th trait |
| **Techniques from teachers** | **KEPT unchanged** (Hallokin's verbs) | |
| **Traits** (PASSIVE sentences on Hallokin) | **KEPT**: 3 slots from Signed duels, +1 from a fielded T5 unit | |
| **Charms and card-to-charm at dawn** | **KEPT unchanged.** Cards are Hallokin's `PERKS` inside a Vigil; one kept at dawn as a charm | Cards never touch units: units have ITEMS/STATS. Two separate stat economies stay separate |
| **Laws** (lord → domain rule) | **KEPT unchanged** (Hallokin's domain) | The domain already freezes only hostile teams, so the Lance keeps fighting inside it (v1 §7.3) |
| **Nibs** | **CUT** | Hallokin gets no gear, so his hits never change (v1 already listed Nibs as cut #2) |
| **Inks** | **REPLACED** by affix → artifact trophies for units (§7) | `ITEMS` artifacts |
| **Ink / Quills currencies** | **KEPT**, with unit weapons and trinkets added to the Ink sink | §7 |
| **Followers add ECOST** | **KEPT, generalised:** `0.4 × fielded unitPower / 100` added to the budget, spent on tier and affixes, not bodies | `unitPower` |
| **Crowned elites grow on your deaths** | **KEPT**, plus your own Hollowed units (§8) | `spawnFrom(build,{team:2})` |
| **Five factions from v9 accents** | Out of scope for this lens. v30 `group` (undead, casters, demons, beasts, shadows, orcs, watch, men, astra, custom) is the recruitable family, and **demons have no home in v1**: the world lens must place them | `FAMILY_MINIONS` keys |

**One system is new: garrisons** (2 per lit Great Lamp). The army then has a job while you are away, and bench units level in the Defend Vigils v1 already has.

---

## 10. Stale-guide pitfalls and code traps (confirmed)

| # | The guide / comment says | The code does | Consequence for v2 |
|---|---|---|---|
| 1 | Step 5: `spawnFrom(build,x,z,{team:0})` (line 37) | `grantXp` returns unless `e.team===1` (373). The kill grant also needs `a.by.team===1` (1027) | **Owned units must be team 1.** On team 0 they never earn XP. |
| 2 | Header line 14 and step 2 line 29: "the player is team 0"; `player()` exposes `team (0)` | The demo hero is `P.team=1` (`setGhost`: -1/1). Line 51 says Hallokin is team 1 | Hallokin is **team 1**. A team-0 player would be targeted by his own units. |
| 3 | Line 14: "spawn opts.team, default 1"; step 6: spawn enemies with `HVU.spawn(kind,x,z,{awake:true})` | `team:(opts&&opts.team)\|\|1` (955) | **Following step 6 literally spawns every enemy on the player's side**: it earns XP, ignores Hallokin, and is never hit by him. Always pass `{team:2}`. |
| 4 | Line 44 and comment 348: tiers rise "by themselves at levels 3, 5, 7 and 10" | `TIER_AT_LEVEL={5:2,12:3,20:4,30:5}`, flag only; `advance` takes it (line 47 of the same guide contradicts 44) | Use 5/12/20/30. Build a rite UI. |
| 5 | Comment 291: stats "each capped at 5" | `STAT_CAP=8` | |
| 6 | Header: "Twenty seven enemies" | 88 CFG entries (including slimelet, blob, mirrorimage) | |
| 7 | Brief: PATHS "chosen at tier 2" | `PATHS`, `setPath`, `pathsFor`, `FAMILY_MINIONS` are **not in the export object** (1625). `path` is **not in `serialize` or `spawnFrom`**, every PRESET has `path:null`, and nothing calls `setPath` | PATHS is dead code. **Module patch (3 edits):** export `setPath` + `PATHS`, add `path` to `serialize`, call `setPath(e,b.path)` in `spawnFrom` before `applyMods`. Otherwise cut PATHS from the design. |
| 8 | Comment 288 + `ARTIFACT_SLOTS(t)` tier-only | `artifactSlots(e)` also grants 2 at L20, and neither is exported | The UI must read slots from the unit, not from a tier table. |
| 9 | Guide line 51: enemies "spawn with {tier, level}" | `spawn` ignores `level`; only `spawnFrom` sets it. XP per kill ignores the victim's tier and level (`power(e.cfg)/8`) | Spawn world enemies via `spawnFrom`. A level-20 lord pays the same XP as a level-1 one, so region XP scaling only comes from kind choice (§1) and the host's Share. |
| 10 | Line 45-46: evolve "keeps the whole build (… skills …)" | Skills are keyed by move id; no kinds share ids, so **evolve wipes all ranks and masteries**. It also nulls the loadout, replaces the object (new `id`, `host.onRemove`) and can chain several evolutions in one frame (knife → voidreaper in 4 calls) | §6.1-6.2 |
| 11 | "ASCENDED at 30% above fullBuildPower" | `fullBuildPower` assumes 48 stat points (the max is 33), truncated in key order. `unitPower` ignores SWIFT/REACH/WILL. The 1400 floor makes 27 kinds unascendable (simulated) (the KNIFE included). `ascended` is never cleared, so respec exploits work | §5.2 |
| 12 | (not mentioned) | `rite()` fires inside `grantXp` at milestone levels and on ascension: `poise=0`, `noMoveT=1.3` mid-fight. Level-up heals 25% | Defer rites in the host (§4.2). |
| 13 | Step 4 "HVU.tick after the slimes": units coexist with HV's enemies | `dealHit` and targeting iterate only `HVU.units` and `playerT()`. There is no hook for foreign bodies. **Owned units cannot see, target or hit the game's own `T.SLIME` enemies** (brown, kitten, rainbow, brute, bomber, caster, shield, archer, summoner, warden, hollow), and killing them grants nothing | **Blocker for Vigils with an army.** Either the Vigil pools spawn HVU kinds (the slime/slimeblue/slimepink/slimeking/lavaslime bodies exist), or the module gains a `host.foes()` hook. The Warden and Hollow Knight bosses are HV enemies, so the Lance is blind in both story boss fights until this is solved. |
| 14 | Demo preset button | `u.items=[]` before equipping the preset's items (1891) | Don't copy it: it deletes found gear. |
| 15 | Demo UI | Cream `rgba(236,229,212)` panels | Re-skin the Barracks screens to the full-colour rule. |

---

## 11. The 10-line summary

**Core finding:** v30 already draws v1's line in code: the demo hero never levels and only team-1 units earn XP. v30's tier and level are exactly v1's THREAT and WEIGHT dials. But the raw income (killer-only XP of `max(6,power/8)`) gets a fielded unit to tier 5 in about 54 hours, so the whole top half of v30 would be unreachable in a 20-hour game.

**Recommendations:**
1. **Hallokin never levels; he feeds and commands the army.**
   - THE SHARE: each kill pays every fielded unit 40% of its XP × RMUL, via exported `grantXp`. That gives T2 at 0.8 h, T3 at 3 h, T4 at 8 h, T5/capstone at 16 h, UNIQUE at 24 h and ASCENDED at 35 h, with XP_TO untouched.
   - COMMAND (from Great Lamps + Roster stamps) caps the summed `unitPower` of a Lance of 3 (2 in lord fights, 0 in duels, +2 garrison in Vigils), so a 5016-power gravelord is a fielding choice, not a steamroller.
2. **Bind = a fresh build at enemy tier − 1 and that tier's floor level (T3 enemy → T2 L5), never `spawn()`.** STUDIED gates both sparing and evolving INTO a kind, so capstones send you to fight what your unit wants to become. One redraw per rite at a lamp (advance OR evolve) stops the free chain-evolve (knife → voidreaper in one frame) and the silent skill wipe.
3. **Death is SCRUBBED** (redrawn from the saved build at the next lamp, losing unbanked XP and 10 renown), **except on blank paper, where the unit becomes a HOLLOWED enemy with your own build:** seal it home or lose it forever.
   - Cut Nibs, Bond levels and Inks (Inks become artifact trophies). Keep the Roster, charms and card-to-charm, Laws and techniques for Hallokin.
   - Fix the traps first: team 1 not 0, `spawn` defaulting to the player's team, PATHS unexported and unsaved, and owned units being blind to HV's own slimes and bosses.
