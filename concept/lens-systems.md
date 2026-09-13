# LENS: SYSTEMS AND PROGRESSION — Hollow Vigil as an open-world RPG

## 0. The core idea

**You don't level up. You learn what the world does to you.** You get stronger by answering each body's moves, then besting it, then binding it to you. Progress is recorded in a sketchbook: the **Folio**, one page for each of the ~40 unit kinds. A page fills in stages. The first stage is filled by parrying, perfect-dodging, guarding or punishing each move the body has. Its later stages give you:
- a technique (an ability slot),
- a trait (its PASSIVE sentence, worn by you),
- a follower (the body itself, with its named ULTIMATE on call).

Enemy numbers still grow by region. But the game's decisive verbs are already relative, so skill stays the main lever:
- execute is a % of max hp,
- stun and poise do not depend on damage,
- a riposte executes from any hp,
- crits are a multiplier capped at 3x,
- unit ults fire at a fraction of health.

Levels mostly buy you fewer hits, not safety.

Design rule for every system below: **it writes a table the game already reads.** This is the pattern 21-waves' PERKS already uses (they rewrite `T.ATK`, `T.DASH_CD` and `ARC.fireEvery`). Gear, traits and laws change *rules*: which crit kinds fire, what a parry pays, what `STYLE_G` rewards, what a domain does. They never change drawings, frame timing or reach beyond a few percent. The hand-drawn animation and the frame-derived hitboxes stay the truth.

Format for each system: **Reuses / New / Why it's fun, not a chore.**

---
## 1. The Folio: progression as knowledge, not XP

Every CFG kind gets a page. The idea starts from the HVU-map-ui §8 bestiary: the `.mk` chip wall, `silhouette()`, `hv_seen`, first-sight gating like `eliteSeen`. It then grows it into the spine of progression. A page has four stamps, in order. Each stamp uses the red `stampR` animation the marks wall already has.

| Stamp | How you earn it | What it gives |
|---|---|---|
| **SEEN** | First sight (`noteSeen`, the `eliteSeen` precedent) | Name, silhouette, hp/poise line |
| **STUDIED** | *Answer* every move on its list once. `c.moves` already counts its strips (A1/A2/A3/Up/Down/Spin/Throw/Summon/Heal…). An answer is a parry (KIIN), a perfect dodge, a full-shield guard (KLANG), or a `punish` crit in its recovery. Each answered move id is inked onto the page as a line with its `HVU.describe()` gloss. | The PASSIVE sentence is revealed. Damage against this kind gets a small permanent **+8% "known"** bonus: the only flat number the Folio gives. |
| **BESTED** | Execute it 3 times (the `exec` rule in `impact()`) | Its **technique** unlocks at the Scribe (§3), if it teaches one. Its drops improve (§6). |
| **BOUND** | Spare it at an execution (§4), or finish its quest if it's a named rogue | It joins your roster. At full bond its **trait** can be worn (§2). |

What counts as your "level" is the **Vigil**: the number of stamps across the Folio (max ~160). The Vigil offsets enemy *weight* (hp and damage, §7), and nothing else.

- **Reuses:** CFG / PASSIVE / ULT / ULTDESC, `HVU.describe`, `c.moves`, the marks wall `.mk` chips and `stampR`, `silhouette()`, `teach()` for first-time prompts, the parry / perfect-dodge / guard / punish code paths (each gains a one-line `folioAnswer(e, atkId)` call).
- **New:** a `hv_folio` store `{kind: {seen, answered: Set(atkId), bested: n, bound}}`, the page detail plate (reusing the pause plate's torn-paper CSS), and the "+8% known" read in `impact()`.
- **Why it's fun:** it rewards exactly what the combat package celebrates: the KIIN, the ZAN, the Executed panel. A new enemy stops being a damage sponge to farm and becomes a puzzle with a visible checklist: "THE LANCER · 3/5 moves". Grinding does nothing. Killing a hundred skeletons gives you one BESTED stamp. Parrying the Greatsword's gsA2 once gives you a line you'll remember. It also teaches the game: to fill a page you *have* to learn the tell.

---
## 2. The character: Two Hands, Four Techniques, Three Traits, One Law

### 2.1 The class swap stays a swap, and becomes the class system

Don't add player classes. The knight and archer sheets, the swap-weave (x1.35 inside 1.2 s), the SWITCH beat and the recovery handed across as `P.lock` are the identity. They cost weeks of feel work, and a third body would dilute them.

Instead, **the two hands are the class system.** The loadout fits the HUD that exists today:
- **Blade hand (knight):** 2 technique slots (E / R).
- **Bow hand (archer):** 2 technique slots (E / R).
- **3 trait slots** (PASSIVE sentences you have inherited).
- **1 Domain Law** (§5.4), bent onto IRON VIGIL / HOLLOW VOLLEY.
- **1 follower** plus the companion archer (§4).

A "build" is how the four techniques weave across a swap. For example: HOOK LINE on the blade, then swap, then BOMB ARROW point-blank on the yanked body. Your build is a combo route, not a stat sheet. `canSwap()`, `ABIL`, `useSkill/useInk` and the two `setDial` dials per class already carry all of this. The only change is that `ABIL` rows are *assigned* to slots rather than hard-bound.

### 2.2 Traits: wear a body's PASSIVE

Once a follower reaches full bond (§4.3), its PASSIVE can go into one of 3 trait slots. The sentence stays exactly as written, re-aimed at you. About half the table translates cleanly with no new mechanics, because it speaks the game's crit and hit vocabulary:

| Trait (source) | On the player | Hooks into |
|---|---|---|
| BACKSTAB (knife) | `back` crits x2.5 instead of x2 | `impact()` ck==='back' |
| FIRST BLOOD (lancer) | first hit on a fresh body x1.3 | the `open` crit branch |
| STEADY AIM (archer) | 1 s still = next release x1.4 | the charged-shot path |
| FLOW (swordsman) | 3 hits in 2 s = the next sweep is free and instant | `AB.SWEEP_COST` / `SWEEP_CD` |
| EXECUTION (axeman) | execute threshold 34% becomes 40% | `exec` rule |
| BLOOD RAGE (berserker) | +5% per hit taken, up to 30%, lost on a heal | `hurtPlayer` / `heal` |
| SECOND WIND (knight) | the last stand also fires at 30% hp, once per rest | `lastStand()` |
| BLOODSUCKER (bat) | executions heal 16 instead of 8 | `exec` heal |
| FRENZY (werewolf) | +6% move speed per landed hit, 5 deep | `T.WALK` read |
| UNSTOPPABLE (werebear) | armor does not break above half hp | the armor/guard path |
| DEATH MARK (warlock) | a MARKed body takes +30% from *everyone* (you, companion, follower) | `e.marked` (already set by heavy hits) |
| HEADSHOT (headhunter) | a still target takes x1.5 from arrows, and arrows pierce | `arrowSim` pierce |
| WARD (wizard) | the first hit every 8 s is absorbed | `hurtPlayer` |
| IRON HIDE (ironorc) | light hits on you do half; you take no stagger from them | `hurtPlayer` + `hurtT` |

The aura passives stay on the *follower*, where they already work through team scans: STANDARD, ZEAL, BULWARK, WARDRUM, SHIELD WALL. That split matters. Traits make **you** sharper. Followers make **the fight** different.

- **Reuses:** the PASSIVE table as content, `impact()`'s crit kinds, `perk()`-style reads, `snapT/restoreT` (to rebuild T cleanly when a trait is swapped).
- **New:** a `TRAIT` table with one small function per sentence, and the equip plate.
- **Why it's fun:** each trait is one sentence you can say out loud and a way to play: "I'm a backstab knight." They don't stack into invisible percentages, because 3 slots force real choices. Every trait is also a memory of a body you spared.

---
## 3. Techniques: abilities unlock from the world

Every ability that exists today gets a **teacher**: a unit kind whose move visibly rhymes with it. BESTED on that page unlocks the technique at the Scribe in Ashridge. You then rank it I→III with ink and quills. The ranks follow the PERKS' three-stack model and the same numbers `ABIL` / `AB` already expose (cost, cd, dmg, radius).

| Technique (exists) | Hand | Teacher | Why that body |
|---|---|---|---|
| IRON FALL (`startSlam`) | blade | THE IRON ORC | QUAKE / aoA3, its ground slam |
| INK CYCLONE (`startSpin`) | blade | THE AXEMAN | WHIRLWIND |
| CRESCENT WAVE (the FULL SWING's throw) | blade | THE BLADE | ASTRAL TEMPEST, the astral duelist |
| HOOK LINE (`startHook`) | either | THE HEADHUNTER | hhSnare |
| HOLLOW RAIN (`startRain`) | bow | THE CAPTAIN | calls an arrow volley on where you stand |
| BOMB ARROW (`armBomb`) | bow | the bomber kittens (game archetype) | the fuse and the blast |
| SKETCH SIGIL (`sigilRelease`) | bow | the sigil caster → THE WARLOCK at rank III | the drawn circle |
| SCRIBBLE DOUBLE (`dropDecoy`) | bow | THE MESMER | turning a body's attention |

**New techniques**, each built from a unit move *re-drawn in the player's own frames plus ink VFX*. No new player sprite strips. This is the same trick 20b used for the crescent (a trail of `slash()` arcs, no new sprite):

| New technique | Hand | Teacher | Built from |
|---|---|---|---|
| SET SPEAR: hold guard; any body that charges or leaps inside 3 tiles eats a stinger | blade | THE PIKEMAN | the parry-on-windup check with a longer window, plus T.ATK[4] |
| SHADOWSTEP: dash x3 chained, each cutting | blade | THE KNIFE | `startDash` re-armed inside `STINGER_WIN`; ghost afterimages |
| SKYFALL: throw the blade; you're disarmed (bow only) until you pick it up | blade | THE LANCER | an arrowPool shaft drawn as a blade, `P.cls` forced |
| NOVA: an ink ring burst around you that knocks bodies off | bow | THE WIZARD | `ringFx` + `shockDecor` + the slam's wave loop at r 2.6 |
| SANCTUARY SIGIL: a closed sigil heals and blesses allies instead of sealing | bow (sigil variant) | THE PRIEST | `sigilRelease` loop branch, `blessT` on followers |
| RAISE: a closed sigil on a fresh corpse raises it as a 10 s ally | bow (sigil variant) | THE NECROMANCER | `charm()` on a body that is `e.hvu` and parked |
| WARCRY: rally your side; you and your followers +25% speed for 5 s | either | THE ELITE ORC | the WARCRY ult's loop pointed at team 0 |

That makes about 15 techniques for 4 slots. The Scribe (§8) swaps loadouts for free at any lit lamp.

Starting kit, no unlocks needed:
- **Knight:** 4-hit string, dash, stinger, sweep, parry, riposte, executions.
- **Archer:** charged shot, quickdraw, point-blank.
- **Both:** the swap-weave.

The first two teachers are guaranteed in Ashridge's first region: the Iron Orc's cousins (orcs) and the Captain's Watch. The player has IRON FALL and HOLLOW RAIN inside the first hour, the kit today's run gives immediately.

- **Reuses:** every 20b ability as-is, `ABIL/AB` for ranks, the player's frame map, `slash()`, `ghost()`, `ringFx`, `sigilRelease`, `charm()`.
- **New:** the slot assignment, 7 technique behaviours (each roughly the size of one 20b ability), and the Scribe UI.
- **Why it's fun:** unlocks have a *place and a face*. You get HOOK LINE because the Headhunter snared you across a ravine four times until you parried the snare. The teacher's move and your move rhyme, so learning it feels like stealing it.

---
## 4. The party: SPARE, BIND, CALL

### 4.1 How you get a follower: spare it instead of executing it

An execution already has a telegraph (the stunned state and the FINISH marker hook), a full-page splash panel and a 2.5 s ration. Add one branch:

- **Tap attack:** EXECUTED, as today. It counts toward BESTED and pays ink.
- **Hold attack for 0.4 s (or hold X / R3):** SPARED. The body kneels. It reuses the last stand's kneel: slow-mo, shade, the 'HOLD' word, but on *it*. The splash panel is worded **"Spared"**. After the fight the body walks to Ashridge's barracks.

You can only spare a kind whose page is STUDIED. You have to know it before it will kneel to you. Named rogues (THE KNIFE, THE BLADE, THE LANCER, THE WATCH) and the custom bodies (captain, berserker, warlock, warden, headhunter, warchief, revenant, hierophant, mesmer, plaguebearer) can't be spared in the field. Each has a short quest ending in a duel, because they are the characters of the story.

Bosses are never spared. They give Domain Laws (§5.4).

### 4.2 How followers fight: the units' team AI, on your team

HVU-map-ai §2 already settles the mechanics: player is `team 0`, enemies `team 1`, and `charm()` flips `u.team`. A follower is simply an HVU body spawned on team 0 with `hunt:false` and a leash on `P`. It uses the same trick as the companion (`AR`'s leash 10 / blink) plus HVU's own `pickTarget`, spacing rings, squads, tokens, guard, heal and summon logic. All ~40 kits come for free: a priest follower heals you through `woundedAlly`, a captain rallies, a necromancer's skeletons fight on your side.

Party rules, for readability and budget:
- **In the field:** the companion archer (always) + **one** follower.
- **In a Night Siege (§6.2):** two followers.
- **Enemy budget:** each follower adds `ECOST`-equivalent threat to `composeWave`'s budget, costed from its `c.moves`-scaled hp. An ally that soaks tokens isn't a free difficulty cut; the fight grows to match.
- **The companion stays outside the team system,** as HVU-map-ai recommends. She's the story partner and the MARK executor. Her progression is her own small row of charms: TWIN VOLLEY and the focus-fire speed.

### 4.3 CALL: the unit's named ULTIMATE, on your command, paid for with style

For an enemy, an ULT fires once per life under a health fraction (`at` ≥ 0.7 or `after` 9 s). For a follower it becomes a **CALL**:
- The follower's ult pip fills from **your style gauge**.
- It is ready the first time you reach **A rank** in a fight.
- A second charge comes at **SS**.
- Press **hold Q** (tap Q stays the domain) and the follower performs its authored ULT: SANCTUARY, CRUSADE, SKYFALL, MASS RISE, STAMPEDE, EARTHSHATTER.
- It plays through `fx.ult`'s page-level beat, so it gets a manga name card.

Bond has three levels:
1. **Bound:** follows and fights.
2. **Sworn:** reached after N CALLs landed. Its aura passive is doubled in radius, and it gets a second Call charge at SS.
3. **Oathbound:** its PASSIVE becomes a wearable trait (§2.2). If it teaches a technique, that technique's rank III is free.

A downed follower isn't dead. It kneels, and you can revive it with a 1.5 s stand-by, the lamp-relight interaction. If nobody revives it, it walks back to the barracks, and that Call is lost for the region.

- **Reuses:** HVU team AI, `charm()` (with the `e.hvu` guard), `pickTarget`, the aura passives, the ULT table and `fx.ult`, the companion leash/blink, the last stand kneel, the execute panel, `lampRelight`'s stand-still timer, `ECOST`.
- **New:** the SPARE branch on exec, follower spawn with a P-leash, the CALL input, bond counters, the barracks roster (the bestiary wall filtered to BOUND).
- **Why it's fun:** the most dramatic beat in the game, the execution, becomes a *choice*. Style rank stops being a score and becomes the button that summons your warchief's RAMPAGE. A party of two can't turn into a management screen. And since every follower is an authored enemy you learned to beat, you already know exactly what it does for you.

---
## 5. The five combat systems as long-term hooks

### 5.1 Style rank D→SS: the payout multiplier and the key

Today the ladder `RANK_AT=[0,22,45,68,86,98]` gives in-fight perks: C +10% damage, B dash cd and a ricochet, A a free FULL SWING, S a forced crit every third hit. Keep all of that. Add three long-term reads:

1. **Payout.** `RMUL=[1,1.1,1.25,1.45,1.7,2]` already exists and becomes the **ink multiplier** on every kill, keyed to the rank *at the moment of the kill*. An SS kill pays double. Farming badly pays half of farming well.
2. **Encounter grade.** Each encounter ends on the peak rank reached, stamped on a small results card (the death card's idiom). A region's map shows your best stamp per encounter site, so there's a reason to go back.
3. **Rank gates.** CALL (A / SS, §4.3). Some quests: the named rogues' duels are won by *reaching S rank before they reach half health*, not by killing them. And some Domain Laws need S rank to open.

Freshness already discounts spam (`STYLE_G` × freshness), so the long-term payout can't be cheesed with one move.

### 5.2 The last stand: a resource you carry, refilled at lamps

Today it's once per run (`P.stood`). In the world:
- **The Vow** is one last stand, refilled when you rest at a lit lamp.
- The Chapel (§8) sells **extra vows** (max 3) for quills.
- The trait SECOND WIND adds an early trigger.
- A priest or hierophant follower can spend its CALL to restore a vow.

THE VIGIL HOLDS keeps paying ultGain 0.3 and style +40, so rising is a *comeback*, not only survival. Death itself is "the page tears": you return to the last lamp and leave your unbanked ink as a **blot** (the ink-pool decal) where you fell. Walk back and touch it to recover it.

### 5.3 Executions: the harvesting verb

Executing is how the world pays you, which keeps the feel package front and centre:
- **Folio:** BESTED needs executions, not kills.
- **Loot:** an execution rolls the drop table with advantage; a plain kill rolls once.
- **Elite affixes drop as Inks** only when the elite is executed (§6.1).
- **Energy / heal:** unchanged (+15 energy, heal 8).
- **The SPARE branch** (§4.1).

The execute rule is `hp <= maxHp*0.34` while stunned, so it is *scale-free*. At region level 30 it's the same window as at level 1. That is the long-term guarantee that the tight verbs stay meaningful.

### 5.4 The domain ultimate: Laws taken from bosses

IRON VIGIL and HOLLOW VOLLEY keep their identities: the ZA… stop, the splash, auto-parry, fan releases, the teleport finale, the manual CUT. Each **boss** you defeat grants a **Law**, one rule that holds inside the circle. You equip one Law, and it applies to whichever class domain you open.

| Law | From | Inside the domain |
|---|---|---|
| BULWARK | The Warden (game boss) | Hits from in front are halved; bodies inside **reel** (1 s stun) at your 66/33% style thresholds, echoing its phases |
| THE FACE | The Hollow Knight | The finale is performed twice: a gold ghost of you (`ghost()`) repeats the teleport chain on the survivors |
| WARDRUM | The Warchief | You and your followers swing and move 15% faster; kills inside heal you 4 |
| HARVEST | The Warlock (custom body, as a region boss) | The finale drains 12 hp from each body hit, to you |
| MASS RISE | The Necromancer / Revenant region boss | Bodies that die inside rise on your side until the domain closes (`charm` on parked `e.hvu` bodies) |
| JUDGEMENT | The Hierophant | The opening stop marks every body (DEATH MARK rules); the companion's volley ignores cover |

- **Reuses:** `ULT` / `openDomain` / `ultFinale` / `inDomain`, the boss kills' `onBossDown`, `ghost()`, `charm()`, `e.marked`.
- **New:** a `LAW` table, a single `law.inside(e,dt)` / `law.finale(list)` hook in `ultStep`.
- **Why it's fun:** the ult is the most cinematic moment in the game, and bosses are the most memorable fights. Linking them means every boss permanently changes your *biggest* moment, not your hp bar.

### 5.5 Elites with affixes: the Crowned, a nemesis that remembers

Today an elite is a normal body with one `AFFIX` (Armored, Swift, Volatile, Vampiric, Hexed, Warded), a gold under-drawing and a name caption. In the world:
- **Crowned elites roam** each region. They're persistent, with a map pin and a name built from their affix plus kind: "THE VAMPIRIC LANCER".
- **If a Crowned kills you, it takes your blot and gains a second affix** (max 3). Its name grows ("THE WARDED VAMPIRIC LANCER"), and its gold under-drawing gets a second step out. It is now worth more.
- **Executing a Crowned drops each of its affixes as an Ink** (§6.1). Killing it without executing drops one.
- **The bounty board** (Watchtower, §8) turns `BOUNTIES` into contracts on specific Crowned, with the old conditions as bonus clauses: TAKE NO HIT, FINISH TWO, CLEAR IN 40s, A CHAIN OF TEN. The clauses pay quills.

- **Reuses:** `AFFIX` / `makeElite` / `eliteSeen` / `wardBlock`, the gold `outl[1]` step, `BOUNTIES` and the bounty counters in `onFoeDown`, the CROWN BREAKER mark.
- **New:** multi-affix stacking (`makeElite` accepts a list, capped at 3), a persistent `hv_crowned` record, the map pin.
- **Why it's fun:** your deaths get a face and a grudge. The rematch is a story you tell. And the enemy's power becomes *your* loot: you literally take the Vampiric affix off the thing that kept healing on you.

---
## 6. Loot and equipment in a game of hand-drawn hits

### 6.0 The constraint, and why it's a gift

A hit here is a drawing: a frame-derived hitbox, a fixed wind / active / rec, and hitstop scaled from the damage. The PERKS pass already found the limit. LONG REACH at three stacks took the FULL SWING to 4.21 tiles, so it was capped at x1.04 for sweeps. Loot that changes swords, reach or attack speed would break both the art and the feel.

So **equipment never changes what you look like or how a swing is timed.** It changes the *rules around the hit*, the way a traditional ink artist has only nib, ink and paper. Three slot kinds, all authored, each described by a single sentence like a PASSIVE:

### 6.1 NIB (one per hand): how a hit is judged

A Nib rewrites the deterministic crit table (`sure / air / back / punish / open / style / lucky`) or the energy/style economy for that hand. Examples:
- **The Flensing Nib (blade):** `back` crits x2.5; `lucky` crits disabled.
- **The Patient Nib (blade):** `punish` window +50%; `open` crits disabled.
- **The Hairline Nib (blade):** poise breaks 25% sooner (HAIRLINE as gear); parries stun 0.3 s less.
- **The Long Nib (bow):** full-draw pierce 3 → 4; point-blank disabled.
- **The Quick Nib (bow):** quickdraw fans 5 shafts; charged shots cost 10 energy.
- **The Juggler's Nib (blade):** airborne hits also build +50% style; stinger dunks cost the combo.

Every Nib has an upside and an **off-switch**, so gear is about choosing a *style*, not a higher number.

### 6.2 INK (one, shared): what your hits leave behind

An Ink tints the *ink VFX*: slash arcs, rings, sigil strokes, the crescent. The post pass already separates ink lines from colour wash, so the new colour shows exactly on the lines. Each Ink adds a status. Inks come from **elite affixes** (§5.5) and from region families:
- **Vampiric (red ink):** executions and crits heal 2.
- **Volatile (orange):** an executed body bursts (`bomberBlast`, friendly-fire safe).
- **Warded (gold):** a parry grants one ward layer (`wardBlock` on you).
- **Hexed (violet):** kills leave a small sigil that erupts on enemies.
- **Swift (pale blue):** a perfect dodge refunds the dash.
- **Armored (graphite):** guard (KLANG) regenerates 10 armor.
- **Plague (green, from the plaguebearer):** the sweep poisons (HVU `poison()`).
- **Bone (off-white, from the revenant):** once per fight, a lethal hit on a follower leaves it at 1 hp.

### 6.3 CHARMS (three slots): the choice cards, made permanent

The 12 PERKS become **Charms** with the same ids and the same three stacks: KEEN EDGE, IRON SKIN, QUICKFOOT, DEEP WELL, LODESTONE, FIELD DRESSING, and the rest. A Charm is found at rank I. Ink it up to II and III at the Smithy with quills. Charms are the *only* place flat stats live, and they are few, capped and familiar.

### 6.4 How loot shows up

- Drops ride the existing pickup system: magnet, 20 s life, cap 14.
- A Nib, Ink or Charm drops as a **torn leaf**, a new pickup kind with a paper sprite and a stamp shout when collected.
- Common drops stay what they are (heart, orb, quill, charm-dash, ink coin), with the economy tuning 21-waves already did (fewer hearts as danger rises).
- **No random stat rolls.** About 40 authored items total. Each named item drops from a specific family, Crowned or siege. The Folio page lists what a kind can drop once it's BESTED.

- **Reuses:** `pickups` / `dropPickup` / magnet, `impact()`'s crit kinds, `STYLE_G`, `AFFIX` behaviours, `bomberBlast`, `wardBlock`, HVU `poison()`, the PERKS table and `perk()` reads.
- **New:** `NIB` / `INK` tables (each entry a small rules object), the ink-tint uniform on the VFX layer, the leaf pickup, the equip plate.
- **Why it's fun:** each drop is a sentence that changes how you play the *next* fight, instead of a number you compare in a tooltip. No inventory Tetris: three slot kinds, ~40 items, all readable on one paper plate. And because Inks come from affixes you've fought, the gear list doubles as a trophy wall.

---
## 7. Regions with levels, and the Night Siege where the wave system lives on

### 7.1 Regions: the districts, grown up

`DISTRICTS` already holds the right fields: a name, a floor colour, a paper tint, a fog range and a forced modifier. It also already crossfades them. Each region adds a **level band**, a **unit family** (a `composeWave`-style pool drawn from the CFG `unlock` tiers), a **Crowned** roster and a **boss**:

| Region | Levels | Family (CFG kinds) | Forced omen | Boss → Law |
|---|---|---|---|---|
| Ashridge Junction (hub) | 1–4 | slime family, watch, orc, archer | — | The Warden → BULWARK |
| The Lamplit Rows | 4–9 | watch, captain, knight, pike, templar, THE KNIFE | NIGHTFALL | The Hollow Knight → THE FACE |
| The Bone Fields | 7–13 | skel, ironskel, greatskel, skelarcher, necro, revenant | FOG | The Revenant → MASS RISE |
| The Orc Marches | 10–16 | orc, ironorc, eliteorc, axeman, berserker, rider | FRENZY | The Warchief → WARDRUM |
| The Moonwood | 13–19 | bat, werewolf, werebear, wizard, mesmer, THE LANCER | FOG + SPITTERS | The Mesmer (as a boss) → a charm-Law |
| The Plague Mire | 16–22 | slimes, plaguebearer, priest, headhunter | IRONHIDE | The Plaguebearer |
| The Hollow | 20–30 | everything, plus warlock, hierophant, THE BLADE | FRENZY | The Warlock → HARVEST; the Hierophant → JUDGEMENT |

The unit's `unlock` field (1–10) becomes the **earliest region tier it may appear in**. That's the `(c.unlock||1)>n` test `composeWave` already runs, with `n` now the region tier instead of the wave.

### 7.2 The two-axis curve: behaviour comes from the region, weight is offset by you

`curve(n)` returns `{spd, hp, dmg, wind, cap}`. Split it:

- **THREAT** = `{spd, wind, cap}` plus affix chance, omen chance, and whether the unit's **ULT/PASSIVE are live**. It comes **only from the region level L**. A level-25 skeleton has fast windups, 4 attack tokens and its BONE DEEP rise, even if you're over-levelled.
- **WEIGHT** = `{hp, dmg}` from `curve(L)`, divided by `curve(V)` (your Vigil, §1) and **clamped to [0.8, 1.6]**. Over-levelled, bodies die in ~20% fewer hits. Under-levelled, they have up to 60% more health and hit 60% harder.

Three hard caps keep the tight combat meaningful at any number:
1. **No single enemy hit exceeds 35% of your max hp** (bosses 50%). Parrying, dodging and guarding always remain the answer, and an under-levelled expert can always win.
2. **The player's number range stays readable.** Base damage stays in today's T.ATK range (10–26). Growth comes from multipliers already on screen (crit, weave, rank C, known +8%, traits), all capped by the existing "3x base" crit ceiling. Damage numbers stay 5–80, so the recent readable-damage-numbers work keeps its value.
3. **The verbs are percentages:** execute at 34% maxHp, stun and poise by hit count, riposte executes from any hp, unit ULT at a health fraction. None of them inflate.

This makes difficulty *behavioural*. Going to a harder region feels like a faster, meaner fight, not a spongier one, and returning home feels like mastery, not boredom. The uncapped tail past wave 20 (+.06 hp / +.03 dmg per level) is kept for The Hollow's endgame.

### 7.3 The Night Siege: the wave run, kept whole

The between-wave cards, bounties, modifiers, lamp defence, typed pools, budgeted rosters and district bosses are the game's most finished loop. Keep all of it as a **world event**:

- **Trigger:** resting at a lamp in a contested region (or accepting at the Watchtower) can call **nightfall**. A siege is `WAVE.n = 1..N` at that region's tier, using `composeWave`, `rollMod`, `offerCards`, `lampsWaveSetup` and `bossFor` as they are today.
- **Cards survive, and they're run-scoped:** picked between waves, wiped at dawn by `snapT/restoreT` (already written so runs don't compound). They're the one place the roguelite "build a monster tonight" power fantasy lives, and sieges are tuned to expect it.
- **Dawn rewards:**
  - Ink by nights survived × peak rank.
  - Quills per bounty paid.
  - **One card taken three times becomes that Charm, permanently.**
  - Holding all four lamps restores the region's colour (§9.2).
- **Endless:** The Hollow's siege has no dawn. It's today's Endless mode with the whole Folio and gear behind you, and the marks wall and SS scores keep their place as the leaderboard.

- **Reuses:** `21-waves.js` nearly verbatim: `curve`, `composeWave`, `POOL` / `ECOST` / `EMAX`, `MODS`, `BOUNTIES`, `PERKS` / `CARDS`, lamps, `bossFor`, `DISTRICTS`, the results card.
- **New:** the region table, splitting `curve()` in two with the clamp, the damage cap in `hurtPlayer`, the siege trigger.
- **Why it's fun:** the open world gives a long arc, and the siege keeps the tight, escalating 20-minute run the game was built around. You choose when you want it.

---
## 8. The economy

Two currencies plus knowledge. There are no durability, crafting materials, food or repair sinks.

| Currency | Earned from | Spent on |
|---|---|---|
| **INK** (soft; the INK COIN pickup already exists) | Every kill: base × the kind's `c.moves` power factor × `RMUL[rank]`; executions ×1.5; siege dawns; selling duplicate leaves | Technique ranks (I→II 150, II→III 400); Scribe re-inking of traits (free at lamps, so never a respec tax); town projects (1–3k); lamp fast-travel between *unlit* pairs |
| **QUILLS** (rare; the QUILL pickup becomes a rare drop) | Bounty clauses, Crowned executions, first STUDIED stamp per page, siege bounties | Charm ranks II/III; extra last-stand vows (Chapel); Nib and Ink upgrades (a second line on the item's sentence) |
| **PAGES** (knowledge, not spendable) | The Folio stamps | Unlock techniques, spare-ability, traits; set your Vigil level |

Healing stays scarce, in the 21-waves tuning: the **Flask** holds hearts (start 3, +1 per Chapel project, max 6) and refills at lit lamps. The world's heart drops still thin with danger (the `WAVE.n>=6` heart→orb swap reads region tier instead).

### 8.1 Ashridge Junction: the hub you rebuild

The starting town is the one place everything is spent. Each building is a project bought with ink (and some quills), and the building visibly *inks itself in* (§9.2):
- **The Scribe:** equip techniques and traits; rank techniques.
- **The Smithy:** Nibs, Inks, Charm ranks.
- **The Chapel:** vows (last stands), flask size.
- **The Watchtower:** bounty board (Crowned contracts), siege calls, the map of best-rank stamps.
- **The Barracks:** follower roster, bond levels, which follower travels with you.
- **The Wall of Marks:** the existing marks wall and bestiary, plus the Folio.

- **Why it's fun, not a chore:** ink payouts scale with *how well* you fight (`RMUL`), so the economy is style-positive. Every sink is a thing you'll use in the next hour. Respeccing is free, so you experiment instead of hoarding. The hub is one small town you walk through, not a menu.

---

## 9. The loops

### 9.1 The 5-minute loop: an encounter

1. **Approach.** Walk the road; `THREATS` chevrons show a pack off-screen; the ink boil and music layer rise.
2. **Read.** A SEEN chip stamps for any new kind. The page ticker shows "THE PIKEMAN · 1/3 answered".
3. **Fight.** The existing kit. One follower is on the field; the companion MARKs; style climbs. A bounty clause might be live ("TAKE NO HIT").
4. **Decide.** The stun lands: execute (ink, BESTED progress) or spare (a follower, if STUDIED).
5. **Cash.** At A rank, CALL the follower's ULT. The pack falls; the pickups magnet in; a leaf drops.
6. **Stamp.** A small results card: peak rank, ink ×RMUL, moves answered, clause paid. It lands as a rubber stamp on the map site.

The hook is that every encounter moves at least one visible bar: a page line, a bond pip, a bounty clause or a rank stamp.

### 9.2 The 1-hour loop: taking a region back

A region starts **drained**. The post pass's existing `mono` uniform (today used for the finale and pause) runs at a region-wide value, so the region is pencil on paper with no colour wash. Taking it back:

1. **Relight lamps** (`lampRelight`'s 1.5 s stand) across the region. Each lit lamp is a checkpoint, a fast-travel point and a vow refill. Each also **paints colour back within its radius**: `mono` masked by distance to the nearest lit lamp.
2. **Fill the family's pages.** Learn one technique and spare one follower from the region's family.
3. **Hunt the region's Crowned** from the Watchtower.
4. **Survive one Night Siege** with all four lamps held.
5. **Duel the region boss.** It gives a Law; the region goes full saturation; Ashridge unlocks a project.

This is the game's art direction as the progress bar: full-saturation colour under the ink is *earned back*. It reuses the shader's `mono` path and the district crossfade (`districtStep`) instead of a map UI.

### 9.3 The 20-hour loop: the Folio, the Vigil, the Hollow

- **Fill the Folio:** ~40 pages, ~160 stamps. The last pages are the custom bodies and bosses.
- **Bind the four named rogues:** THE KNIFE, THE BLADE, THE LANCER, THE WATCH. They're the story's "vigil" of companions, each with a duel quest gated by a style condition, and they can travel as your follower. THE BLADE's ASTRAL TEMPEST CALL and TEMPO trait are the endgame's showpiece.
- **Collect Laws** from six bosses. Build a final loadout: 4 techniques, 3 traits, 2 Nibs, 1 Ink, 3 Charms, 1 Law, 1 follower.
- **Rebuild Ashridge** completely; the town finishes inking in.
- **The Hollow's Endless siege** is the capstone: uncapped curve tail, all omens, the marks wall and the SS best score.
- **Crowned rematches** keep the late game personal, because your own deaths keep writing new enemies.

**Stretch capstone: BORROWED FORM.** An Oathbound named rogue can be *worn* for the 6 s of your domain. You play its HVU `ATK` table: the knife's kA1→kA3 chain, kUp launcher and kPlunge; the Blade's bUp→bSlam and bSpin. The inputs are attack / hold / dash-attack, and the moves already have frame-derived hitboxes. This is the one place the 227 sheets become *player* animation. It's contained inside a timed state the game already isolates (the ult), so the knight/archer identity outside it is untouched.

---
## 10. What we deliberately don't build (anti-chore rules)

- **No XP bar and no stat points.** Your level is your Folio; killing weak things repeatedly gives nothing past BESTED.
- **No random affix rolls on gear.** ~40 authored items, each a sentence.
- **No gear that changes reach, frame timing or silhouette** beyond ±5%. The LONG REACH cap lesson becomes law.
- **No durability, hunger, follower upkeep or respec cost.**
- **No party bigger than companion + 1 in the field (+2 in sieges).** The screen is a drawing, and the ink pass and the token system both need to stay readable. HVU-map-ai's warning against a general army mode stands; sieges and the MASS RISE Law are the only times a crowd fights for you.
- **No difficulty via hp sponges.** Threat comes from behaviour. Weight is clamped to [0.8, 1.6], and single hits are capped at 35% of max hp.

## 11. Reuse ledger (summary)

| System | Mostly reused | Genuinely new |
|---|---|---|
| Folio | CFG / PASSIVE / ULT text, `c.moves`, marks wall, `silhouette`, parry/dodge/guard/punish paths | Page store, answer hooks, detail plate |
| Two Hands loadout | `ABIL` / `AB`, `canSwap`, `setDial`, swap-weave | Slot assignment, Scribe plate |
| Techniques | 8 existing 20b abilities | 7 technique behaviours in player frames + ink VFX |
| Traits | PASSIVE sentences, `impact()` crit kinds, T reads | `TRAIT` functions |
| Party | HVU team AI, `charm`, aura passives, ULT + `fx.ult`, companion leash, last stand kneel | SPARE branch, follower leash, CALL input, bond |
| Laws | `ULT` / `openDomain` / `ultFinale`, `ghost`, `charm` | `LAW` table + one hook |
| Crowned | `AFFIX` / `makeElite` / `wardBlock`, `BOUNTIES` | Stacking, persistence, pins |
| Loot | pickups + magnet, crit table, AFFIX behaviours, PERKS | `NIB` / `INK` tables, ink tint, leaf pickup |
| Regions + curve | `DISTRICTS`, `curve`, `composeWave`, `unlock` tiers | Region table, THREAT/WEIGHT split, hit cap |
| Night Siege | All of 21-waves, `snapT` / `restoreT` | Trigger, dawn rewards, card→Charm |
| Economy | INK COIN, QUILL, heart, `RMUL` | Currencies' store, town projects, Flask |
| Region colour | Post pass `mono` uniform, `districtStep`, `lampRelight` | Per-lamp colour mask |

## 12. Suggested build order (systems only)

1. **Save model** (`hv_folio`, `hv_loadout`, `hv_crowned`, `hv_town`), plus splitting `curve()` into THREAT/WEIGHT with the hit cap. Invisible, and it de-risks everything else.
2. **Folio answer hooks and the bestiary plate.** Immediately makes the current single-town game deeper.
3. **Night Siege as-is inside Ashridge,** with dawn rewards and card→Charm. This is today's game with persistence.
4. **SPARE → one follower → CALL on A rank.** Needs the HVU host bridge (HVU-map-host) landed first.
5. **Technique slots and the first new technique** (SET SPEAR: the cheapest, built on the parry window).
6. **Nibs and Inks** from affixes; Crowned persistence.
7. **The second region,** the Lamplit Rows with the Hollow Knight's Law, and the `mono` colour mask on lamps.
8. **Remaining regions and Laws; Borrowed Form last.**
