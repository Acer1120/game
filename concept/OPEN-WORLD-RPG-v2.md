# HOLLOW VIGIL — THE OPEN-WORLD RPG · FOLIO v2

*Lead design synthesis, 2026-09-13, replacing Folio v1. It is rebuilt around HV Units v30 (`hv-work/units30/units.js`, `VERSION '2026-09-13 final'`) and draws on four new lenses: the army (`concept/v2/lens-army.md`, whose XP numbers came from running the real module in node), the roster (`lens-roster.md`, whose lineages were computed by script over `EVOLVE`), integration (`lens-integration.md`, audited against `hv-work/src/`) and the critic (`lens-critic.md`, v1 section by section). Every claim this plan depends on was re-checked in the module source. It is a concept, not a spec: each major idea names the v30 or game system it grows from, so it can be built.*

---

## 1. The pitch

**Hollow Vigil is an open-world action RPG set inside a living drawing that is being erased: you are Hallokin, the last Watchman, you win the page back one lamp at a time, and every enemy you learn to beat can be spared, drawn onto your side, and grown into the very lord you had to beat to get there.**

**Why it works.** The game already has three things most action RPGs never get.
- **Fights that feel superb:** real hitstop, parries, executions, a domain ultimate.
- **A roster with personality.** Since v30, that means 87 unit kinds in 21 family lines, each with its own moves, a passive and a named ultimate.
- **A complete army RPG,** written, tested and frozen by its author: levels to 50, five tiers taken by rite, six stats, talents, skill ranks with masteries, gear, titles, ascension, and an evolution graph that runs from a tier-1 skeleton to THE GRAVELORD.

An open world is the natural home for all three. The usual danger is a big, flat map full of trash fights, plus, now, an army that plays the game for you. Four rules prevent that:
1. **The world is small and dense.** You walk into fights that are already happening.
2. **Hallokin never levels.** His hands stay exactly as tuned. The army levels, and it levels fastest when *he* fights well.
3. **You can only grow what you understand.** Sparing a kind and evolving into it are both gated on having studied it.
4. **Every lord is a future.** The thing ruling each region is the finished form of a unit you could own. You cannot bind it. You must beat it, then raise your own.

**Why it could only be this game.** Other "sketch" games wear the look as a costume. Here the look is the physics of the world:
- The ink line boils ten times a second because the world is being watched.
- Bodies are drawn in when they spawn and scrubbed out when they die, because in this world that is what birth and death are.
- Colour is laid in only where the world is kept.
- The enemy of everything is not darkness but **blank paper**: the Hollow, which does not destroy things, it un-draws them.

v30 slots into that physics without a seam. A tier is a stage of a drawing: line, under-stroke, colour, shadow, signature. A rite is Hallokin inking the next stage onto a figure. An evolution is a scrub-out and a redraw. A spared body is a figure countersigned into Hallokin's kin. The player can read every rule off the screen without a codex.

---

## 2. What v30 changed, and what this plan changes

### 2.1 What was wrong with v1

v1 was designed against units v9, a library of enemies that never grew. The critic walked it section by section against v30. Its world, look and verbs survive. Its foundations do not:

| v1 said | What v30 does | Verdict |
|---|---|---|
| "No character levels, no XP bar, no stat points." | `XP_TO(l)=round(12·l^1.35)`, `MAX_LEVEL=50`, `STATS`, `POINTS_AT`, `TALENTS`, `MASTERY`, `ITEMS`, `TITLES`, ascension. | **Broken as written.** It survives only as a rule about the hero. |
| A party of the companion plus one follower. | Built for armies. `TALENTS` WARLORD, `war_banner`, STANDARD, ZEAL and the UNIQUE ally aura pay only with several allies. | **Broken.** With one follower, half the tables are dead. |
| Five factions, with membership read off accent colours. | Evolution changes accents (knife `#8fd3d0` → verdagger `#3a8a5a`). About 25 kinds have no home, including a whole demon family and every tier-5 capstone. | **Broken at membership, holds at politics.** |
| Six region lords from custom bodies (Hierophant, Revenant, Warchief, Plaguebearer, Mesmer, Warlock). | All six are mid-tier evolution targets the player can own. | **Broken.** A lord you can raise from a slime in an afternoon is not a lord. |
| Rivals "flee at their ULT's health threshold"; "at 60% he fires ASTRAL TEMPEST". | At load every `c.ult.at=Math.max(c.ult.at,0.7)` with `after` 9 s. The trigger (units.js 1333) fires under 70% health, or after about 9 s engaged, or when 3+ foes crowd it. | **Broken.** Ultimates are timed beats. |
| CALL: hold Q to fire a follower's ULT. | Owned units fire their own ULT on that timer, and only from tier 4 (`tierHas 'ult'`). | **Broken in mechanism, worth saving** (§10.3). |
| Bond: Bound, Sworn, Oathbound. | Levels, `MILESTONES` (VETERAN, HARDENED, MASTER, UNIQUE, MYTHIC) and `TITLES` are a richer bond ladder that already exists. | **Obsolete.** |
| Nibs and Inks as the loot pillar. | The world must now drop unit `ITEMS`. Two loot tables at once confuse. | **Obsolete.** |
| THE SENTINEL (renamed `warden`) belongs to the Gilt Order. | `captain →@5 warden`, and GROUPS lists it under `watch` with steel-blue `#7fa0d0`. | **Broken.** It is the Watch's top rank. |
| "Three structural edits to the units module." | 59 team comparisons, `evolve` splices and respawns, and there is no leash, no owned flag and no ult hold. | **Too few.** |

The critic also found two things v30 brings that break v1's one non-negotiable rule. The demo UI is cream paper (`--paper:#ece5d4`). And custom, UNIQUE and ASCENDED bodies get a solid accent silhouette 7% wider than the body, drawn behind it: an outline. The integration lens found a third: `drawInfo` multiplies every team-2 body by `[1,.72,.72]` and every team-1 body by `[.86,.92,1]`.

### 2.2 What v2 does better

1. **One loop, not five systems.** "Every lord is a future" (§3) joins SPARE, the Roster, lamps, Vigils and the bosses into a single chase. In v1 they were parallel pillars.
2. **The hero/army split is settled, and the numbers are real.**
   - Hallokin never levels. The army climbs v30's full ladder.
   - THE SHARE, COMMAND and lamp-bound rites keep the hands central.
   - The pacing table (§7.4) comes from the module itself. Raw v30 income would take a unit to tier 5 in about 54 hours. With THE SHARE, capstone evolution lands at hour 16, as Chapter III opens.
3. **A world organised by lineage.**
   - Six houses (the new KINDLED included), plus the Wild and the Dusk.
   - Every one of v30's 87 kinds is placed, and every region has a capstone lord.
   - The fifteen evolution edges that cross sprite groups become the world's history of defections.
4. **Chapter III becomes a triad of false cures:** the Order's Varnish, the Redrafters' Trace and the Kindled's Burn. You can break only one in person. Your posted army holds the other two.
5. **A de-risked build path.** Vendor the module untouched apart from five named patches. Draw units through the game's own sprite path. Ship one HTML file with lazy art families. Every phase has a probe, a screenshot and an `HVU=0` rollback that builds byte-identical to today's game.
6. **The traps are named.** Twenty-odd places where the module, its guide or the demo would silently do the wrong thing in this game are listed in §9.4, each with its fix, so nobody meets them live.
7. **The colour and feel rules are extended** to everything an army brings: its UI, its outlines, its tints, its kills, level marks, rites and ultimates.

---

## 3. The spine

### 3.1 Every lord is a future

**A unit can only evolve into a form you have beaten.** Three lenses reached this rule independently, from three directions:

- **The critic** called it "the one idea v1 lacks": *every lord is a future, and you cannot evolve into a form you have not BESTED.*
- **The army lens** made STUDIED the gate for both sparing a kind *and* evolving into it: "you must know a drawing before you can redraw yourself as it".
- **The roster lens** found the rule in the data. Capstones can only be grown, never recruited. `evolve()` works on any tier-5 body, enemies included. So every lord can evolve *in front of the player*, mid-fight, playing the same `rite(n,'EVOLVED')` the player's own unit will play later.

The loop, played out:

> **Spare** a tier-1 skeleton in the Palimpsest at hour two. It is countersigned into your kin as a build object. → **Fight beside it** as it levels on your kills. At hour three, at a lamp you lit, it takes the rite of tier 3 and redraws as THE BONE GUARD. → **Meet its future:** at hour twelve, THE GREATSWORD who rules the Palimpsest kneels at half health, then scrubs out and redraws, in the middle of the fight, as **THE GRAVELORD**. Your own greatsword stands beside you and watches the rite it will one day take. → **Beat him.** His page stamps BESTED, and the gravelord edge on your unit's evolution tree inks in. → **Hold the land:** keep the Vigil at the Palimpsest's Great Lamp, and your unit earns the levels. → **At a lit lamp, redraw it** as THE GRAVELORD, still carrying your name for it. If the crown is vacant, it is crowned. → **Post it** as WARDEN OF THE PALIMPSEST, or march with it, if your COMMAND can carry 5,016 power.

Each spine item gains a job from it:

| Spine item | Its job in the loop |
|---|---|
| **SPARE / SEAL** | The only door into v30's progression layer. Nothing becomes yours except by the hero's choice. |
| **The Roster** | Its key ring. STUDIED opens sparing and ordinary evolution edges. BESTED opens a capstone's edge. |
| **Lamps** | Where rites happen. No lamp lit, no redraw. |
| **Vigils** | Where the army earns its levels at scale, and where garrisons hold. |
| **Lords** | Futures. Seven boss-flagged capstones plus four more evolve live in their fights (§5.5). |
| **Faction clashes** | Decide whose lords you meet, and when. |

Every fight also gets a question beyond "win": *is that thing my unit's future?*

### 3.2 v1's spine, where v30 leaves it standing

1. **Lamps draw the world back in, and restored colour is the progress bar.** v30 does not touch it. A lit lamp re-bakes the edge-plane with one more ragged hole, and the props ink in. It gains one job: rites are taken only at lamps Hallokin has lit.
2. **SPARE is the core verb.** Now load-bearing. It went from "recruit a follower" to "found a lineage".
3. **Your first blow picks your side.** It holds in design. In implementation the army must hold too: owned units stay out of a clash until your first damaging blow (§8.4). Owned and allied must also be split, because v30 credits XP to anyone on team 1 (§9.4).
4. **The Vigil survives whole,** and gains a second role: it is where the army trains.
5. **No character levels, for the hero.** Hallokin's hands never change. The world, meaning the army, levels.

The tech constraint stays under all of it: **the game remains a small arena, and the arena moves with the player.** Only bodies near you are real. The rest of the world, owned units included, is data. v30 makes that easier than v1 hoped, because `serialize(e)` / `spawnFrom(build)` is a complete token-to-body round trip.

### 3.3 The four convergences

Lenses that wrote without seeing each other agreed on four things. Nothing below is allowed to dilute them.

1. **Every lord is a future** (critic, army and roster). STUDIED/BESTED gates evolution; capstones are grown, never bound; lords evolve live.
2. **Hallokin never levels, and the army does** (army, critic, and the v30 code itself). The demo hero damages units through `HVU.hurt` with no `by`, and `grantXp` refuses anything not on team 1.
3. **The bridges that keep the hero central** (army and critic):
   - THE SHARE: style-scaled XP from Hallokin's kills;
   - rites only at lamps he has lit;
   - a cap on fielded power that grows from his territory and knowledge.
4. **A unit is its build object** (army, integration and critic). Binding, saving, death, the Hollowed, rivals and squad tokens all speak `{kind,tier,level,xp,renown,ascended,items,stats,talents,skills,loadout,label,tint}`.

## 4. The look is the world's physics

### 4.1 The six forces

Each force is something the renderer or the sim already does. The fiction only names it. v30 adds a row to four of them.

| Force | What it is in the world | What the player sees | Grows from | v30 adds |
|---|---|---|---|---|
| **THE LINE** | Existence. A thing is real because it is outlined. | The ink edge on every body, roof and tree. | `inkLine()` mode 1, a 1.35 framebuffer-px screen-space ring (REPORT-hires) | Units get the same line through the game's rig. They never get v30's accent silhouette. |
| **THE BOIL** | Life. Being watched re-inks the line ten times a second. A line that stops boiling is dead or held. | The 10 Hz wobble. | The `BOIL` clock, held during hitstop | Unit frames commit on the `ANIM` beat (`commitPose`), not v30's smooth frame clock. |
| **COLOUR** | Care. Colour is laid in where the world is kept. | Full-saturation colour under the ink. | `FLOOR_TINT`, `COLOUR_SAT`, the "proper colour" pass | The Kindled: colour *taken* instead of kept (§5.2). |
| **INK** | The substance of being: it spills, pools, burns in lamps and runs loose as slimes. | Kill puddles, splats, slimes, slicks. | Decals (`15b`), `slickAdd` | Lava slicks in the Kiln. |
| **PAPER** | The Hollow: not darkness but blankness. It un-draws. | A clean blank page with construction lines, eating in at the edges. | `edgePlaneTex()` | Units lost on blank paper come back HOLLOWED (§7.3.7). |
| **THE PANEL** | Attention. Being drawn in a panel makes a moment permanent. | Cut-ins, killcam, letterbox, results card, style rank. | `renderPanel`, `panel()`, killcam, `STYLE` | **The rite:** the Chronicle drawing a figure anew. |

### 4.2 Tiers are stages of a drawing

v1's ink-in has three held drawings. v30's five tiers extend it, and `tierHas` (units.js 422) makes each stage mean exactly what it looks like:

| Tier | Stage of the drawing | What `tierHas` turns on | Level (`TIER_AT_LEVEL`) |
|---|---|---|---|
| **T1** | **the line** | first move only, plain AI | 1 |
| **T2** | **the under-stroke** | dash, charge, block, hop | 5 |
| **T3** | **the colour** | passive, special, heal, summon | 12 |
| **T4** | **the shadow** | ULT, smart AI (focus the wounded) | 20 |
| **T5** | **the signature** | full kit, ×1.1, cooldowns ×0.85, second artifact slot | 30 |

- **A rite** (`advance`, host `fx.rite`) is Hallokin inking the next stage onto a figure. It plays that stage's ink-in on the 10 Hz clock.
- **Sparing** scrubs one stage off: a bound unit arrives one tier below the body that knelt (§7.3.2).
- **Evolution** is a scrub-out and a full three-drawing ink-in of a relative. `evolve()` really does despawn and respawn in place (units.js 311-313).
- **Enemies wear their stage honestly.** A tier-2 skeleton in the Vale does not use BONE DEEP, because its colour has not been laid in. Region tier is readable off the bodies.

### 4.3 The colour and outline rule, extended (non-negotiable)

The user has rejected the white or cream paper look three times ("the entire world is white", "too faded, dirty, hard to see, muddy", "I want proper color"). They have just had every ugly outline removed and the game moved to native resolution. v1's rule stands, and v2 extends it to everything v30 brings.

**The world (v1, unchanged).**
- **Full-saturation colour under crisp ink is the default and the restored state.** No faded, greyed, lined or cream middle state exists. Land is either drawn in full colour or erased to blank paper, and the edge between them is spatial and hard: a torn, moving front, never a filter.
- **Blank paper is a threat you look at, not a place you live.** It is clean and bright, never grey or dirty. Outside the Margin it never covers more than about a quarter of a combat frame. Inside the Margin, the watched circle (§4.5) guarantees colour around the player.
- **The Scorch's char holes obey the same budget.** They are hard-edged and dark (the back of the page, shown through the shader's `invert` path), never white, and never more than a quarter of a frame.
- **`mono` graphite** keeps its one meaning: the domain finale and the pause page.

**The units (new).**
1. **No v30 outline, ever.** The `olm` accent silhouette behind custom, UNIQUE, ASCENDED and `customAccent` bodies is not drawn: the game's `unitRender` ignores `drawInfo().custom` and `.accent`. Rank is read elsewhere:
   - a gold ring on the unit's army chip;
   - a crown glyph over a *crowned* unit, the same glyph v30's demo draws for group `custom`;
   - an accent-colour ground tick at the feet of owned units, drawn by the ink layer like the guard marks (25-manga.js 637). It is not a sprite rim.
2. **No team tints.** `drawInfo().tint` multiplies team 2 by `[1,.72,.72]` and team 1 by `[.86,.92,1]`, which pinks every enemy, blues every ally, and turns a hostile Watch squad the Kindled's colour. The renderer uses only `e.customTint || e.cfg.tint`. Relation is shown by:
   - the ground tick (owned units);
   - v1's thin ink underline (sided house allies);
   - nothing at all (hostiles are the things attacking you).
3. **Charm is a tick, not a wash.** `charm()` sets `tintOverride=[0.8,0.55,1.0]`, which darkens a green orc toward mud. The renderer ignores it, flips the body's ground tick to violet, and prints one `HYPNOTISED` tick.
4. **Effect layers draw in their own colour.** v30's renderer tints its fx quad near-black (0.09, 0.09, 0.1). That produces the "black smear on the body" bug class this project has fixed twice already. Unit swing layers draw untinted at `renderOrder + 0.1`.
5. **Crisp at native resolution.** Nearest filtering and the 1.35 px line. A 1:1 crop of a unit beside the knight must show the same line weight and no rim (the phase 0 screenshot, §9.9).
6. **HOLLOWED bodies** are the ink line at full weight over a transparent fill: `inkLine` mode 1 with the body alpha at 0. Never `sheetHalo`, never `olm`, never a black silhouette.

**The army UI (new).**
- **No cream.** `#ece5d4`, `#f4efe2`, `rgba(236,229,212,…)` and the striped paper sheet header are never lifted. Keep the demo's information design and rebuild it in the game's idiom:
  - ink-bordered plates that boil (`sketchBorder`);
  - `makeBar` brush bars and `makeDial` rings;
  - Bangers (`FONT_POW`) numerals and Patrick Hand (`FONT_HAND`) text;
  - each unit's `cfg.accent` at full saturation as the colour of its ring and bars.
- **Portraits in full colour:** `Idle` frame 0 with `imageSmoothingEnabled=false`, drawn the `bossChip` way *without* its multiply-tint pass.
- **Tint choices come from the family's accent palette** (six swatches). The demo's free colour picker is dropped, so no player can paint a unit muddy.

### 4.4 Birth, erasure and redrawing

- **Spawning is being drawn.** The three-stage ink-in, held on twos, under half a second.
- **Death is being scrubbed out.** The kind's `deathKind`. For an owned unit, being *scrubbed* is recoverable: it is redrawn at the next lamp (§7.3.7).
- **A rite is the next stage inked on.** The unit holds (`noMoveT` 1.3 s), its accent `bit`s rise, rings stagger out, and the stage's ink pass sweeps the body. It happens only out of combat.
- **An evolution is a redraw.** A scrub-out, one held blank frame, then the relative inks in line-first. When a lord does this mid-fight, it is the most important image in its region.
- **A lit lamp draws the land around it.** One more ragged `destination-out` hole in the edge-plane, and fences, then houses, then trees ink in on the 10 Hz clock. This remains the open world's core reward animation.
- **An untended lamp gutters.** Its circle shrinks, and props at the edge scrub out and stop colliding. A lamp with a posted Warden never gutters.

### 4.5 The watched circle

Kept from v1 whole. On blank paper, a circle of colour and line follows you, and its radius is your style rank: about 6 tiles at D, the full camera frame at SS. It is built on `STYLE.rank` and a radial term in the post pass. v30 gives it a second meaning with no new code: `RMUL[STYLE.rank]` (`[1,1.1,1.25,1.45,1.7,2]`, 20b-abilities.js 28) also multiplies THE SHARE. **Fight well and you carry a lamp, *and* your army learns twice as fast.**

### 4.6 The regions are drawn in different hands

Regions differ in *how they are drawn*: line weight, hatch in true shadow only, boil rate, saturation at or *above* the default and never below, and fog. It is dial work, one row per sheet, with no new shader. The hands are listed in §5.4. The Scorch is the new extreme: **too much colour**, hot orange rims and a heat-shimmer boil.

### 4.7 Two places that break the rules, once each

- **The back of the page.** Iron Fall on a dog-eared corner flips the page (full-strength page tilt) to its reverse, drawn with the `invert` path: dark, never white. A reversed drawing is a mirror, so **each back of a page holds a MIRROR KNIGHT duel** (§5.5), and the deepest one is the Hollow Knight's lair.
- **The Reference.** The finale's last room shows the pixel world with the sketch pass off: the Draftsman's model, unwatched and perfectly still.

## 5. The world

### 5.1 Premise and the hero

The world is a drawing that stays real only while it is watched. The Draftsman who drew it has put down the pen. Blank paper, the Hollow, is eating in from the margins. The Watch of Ashridge is an old order sworn to keep lamps burning along the roads, because a lamp is a thing that watches when no person can.

**HALLOKIN, THE LAST WATCHMAN.** The office is Watchman and the name is Hallokin. The title card, the save slot and the Chronicle all read HALLOKIN. Neither word is on screen today ("Hallokin" appears only in three guide comments, "the watchman" only in `10-config.js` line 3), so nothing shipped changes.
- **The name is a signature.** Under the Draftsman's study sheet of the knight and the archer ("two studies of the same figure") is one word in his hand: *Hallokin*. A signature is the one mark the Hollow respects. That is why the hero can walk onto blank paper and keep a watched circle when nobody else can, and why the three Signed wear teal.
- **The name is handed down.** The Watch has always had one Hallokin. **THE HOLLOW KNIGHT held it before you.** He held his domain over the whole page, and when he let go, the Hollow took the one thing it respects, the signature. That is why the shipped boss is named only by what he became. The reveal, "his name was yours", comes from THE REVENANT at the Old Watch Barracks. Its accent `#9aa0b0` is almost the Watch's `#9aa5b1`: the dead captain of his company, still in Watch grey.
- **Kin.** Recruiting a unit **countersigns** it into Hallokin's kin. That is the fiction for v30's team 1: owned units never hit Hallokin and he never hits them, because they share a signature.
- **"The last Watchman" is true at the start and false at the end.** By the finale a crowned Warden keeps every lamp you posted, and **THE VIGIL IS KEPT · ENDLESS** becomes a plural sentence.

**The two studies (v1, unchanged).** Knight and archer are one figure drawn twice. The swap is turning to the other study. The last stand is the Vigil redrawing you once. The domain (IRON VIGIL / HOLLOW VOLLEY) is the Watchman holding the page still on purpose. None of it touches HVU: the hero is `20-player`, outside the module.

**The companion** is the Hollow Knight's archer-study, the half of him that did not go hollow. She stays outside the army: not a build, never levelled, not counted in COMMAND, and the MARK executor. v30 has its own `archer` (THE ARCHER, `#c9a86a`). The Barracks never lets a player own an archer that reads as her: owned archers wear the headhunter or demon-archer tint from bind.

### 5.2 The houses

**The rule: a house is a set of lineages, not a set of colours.** A lineage belongs to the house holding its root and its capstone. Evolution edges that cross between houses are defections (§5.3). Accents still read at a glance, because lineage accents mostly agree: Watch steel, Order gold, bone, Redrafter violet, Warband green, Kindled ember.

Team ids: **Hallokin and owned units are team 1** (v30's own convention). Hostile spawns default to **team 2**. **Houses take ids 3 and up**, and `hostile(a,b)` consults the relations table (§9.2, patch X2).

| House | Lineages | Capstones (★ `boss:true`) | False cure | Fights with |
|---|---|---|---|---|
| **THE WATCH OF ASHRIDGE** (steel `#9aa5b1`, `#8fa8c8`) | watch → captain → **sentinel**; knight → mirror knight; swordsman; axeman (before desertion); pike; headhunter | THE SENTINEL `#7fa0d0`, THE MIRROR KNIGHT `#c0c8d8` | none: the Watch tends lamps, which is care | SHIELD WALL, STANDARD, LAST STAND, SET SPEAR, BULWARK / AEGIS, HEADSHOT |
| **THE GILT ORDER** (gold `#d0b050` → `#e0c060`) | templar → chrono templar; priest → hierophant; cannoneer → inquisitor; THE BLADE, sworn (→ spellblade) | ★ THE CHRONO TEMPLAR `#d0b040`, THE INQUISITOR `#e0c060`, THE SPELLBLADE `#40b0e0`; THE PLAGUE DOCTOR `#5a9a3a` as heretic | **the Varnish:** hold every line still. CHRONOBREAK and REWIND are the Varnish working. | BLESSING, ZEAL, HOLY GROUND, WARD, JUDGEMENT, PURGE |
| **THE UNDERDRAWN** (bone `#d8d2c4`) | skel → bone guard → greatsword → gravelord; bone archer → bone stalker; pale bones → bone blade → dark blade → revenant | ★ THE GRAVELORD `#a0a8b0`, THE REVENANT `#9aa0b0` | none: they want to be people again | BONE DEEP, UNDYING, MASS GRAVE |
| **THE REDRAFTERS** (violet `#9a6fd8`) | wizard → mesmer / lich; necromancer → warlock → lich; dark warlock → lich | THE LICH `#3ac8c8` | **the Trace:** draw over everything, allegiance and death included. PHYLACTERY is a caster who traced out the part of himself that could die. | STORM, MASS HYPNOSIS, MASS RISE, HARVEST, DEATH WINTER |
| **THE GREENHAND WARBAND** (brush green `#6a9a3c`) | orc → berserker / elite orc → warchief; iron orc (deserters); rider; THE LANCER, sworn | ★ THE WARCHIEF `#3f5a2a` | none: refugees whose own sheets are going blank | WARDRUM, WARCRY, RAMPAGE, QUAKE |
| **THE KINDLED** (ember `#a03030` → `#c05a20`) | demon lord → abyss lord; lava slime → flame golem → magma colossus; the Choir (gremlin → imp → demon brute / succubus; blood fiend and harpy → succubus; harpy → siren); the Cinder Watch (beam knight → black knight); blood wing; hellhound; demon archer | ★ THE ABYSS LORD `#2a2a8a`, ★ THE MAGMA COLOSSUS `#8a1a1a`, THE SIREN `#40c8b0`, THE DEMON BRUTE `#7a3a2a` | **the Burn:** a page on fire cannot be erased | HELLFIRE, OBLIVION, MELTDOWN, CHORUS, BLACK FLAME, LEECH, SWARM |

**The Kindled, where the demons go.** Lamp-flame is ink turned into light, and a lamp burns only Well-ink. When the Well cracked, the southern Great Lamp at **Cinder Gate** ran dry. Its Watch kept it lit by feeding it **colour scraped off the land around it**. The flame took the colour and came alive. The Kindled are colour that was *taken instead of kept*: the world's colour-means-care rule turned inside out. The data reads as hunger:
- LEECH on the blood fiend, blood wing, succubus and shade;
- drain-on-hit on the Abyss Lord;
- SWARM on the imp, gremlin and hellbat;
- charm as care taken by force (KISS, CHORUS);
- fire pillars (HELLFIRE, INFERNO, BLACK FLAME).

The characters:
- **THE DEMON LORD** is the Cinder Gate flame, the first thing that walked out of the lamp.
- **THE ABYSS LORD** is that flame burned hot enough to go blue and through the paper. OBLIVION pulls bodies into the hole it burned, and behind a burn hole is the back of the page.
- **The Cinder Watch** (THE BEAM KNIGHT `#5a6a8a`, THE BLACK KNIGHT `#4a4a58`) are the Watchmen of Cinder Gate who fed the lamp. They are the Watch's secret shame, and the reason the Watch and the Kindled can never make peace.

**The three false cures.** The Order would *still* the page. The Redrafters would *trace over* it. The Kindled would *burn* it. Each would stop the Hollow and end everything alive. The Watch's answer is the only slow one: tend the lamps.

**Not houses.**
- **THE WILD**, hostile to all and political to none:
  - **The Blot** (loose ink): slime `#5aa83a`, frost slime `#3a7ac8`, venom slime `#c85a9a`, their three kings, toxcrawler → plaguebearer `#8a4fb8`, the slimelet.
  - **The Hatch** (beasts in true shadow): bat → hellbat; hellhound-born werewolf `#8fa0b8` → werebear `#6a4a3a`; the minotaur. Its crown is **THE STORM WING** `#3aa0e0`, the one boss capstone no house can claim.
  - **The Lampless** (new): a snuffed lamp does not simply go out. Its watching wanders off as an EYEBALL `#4a7a8a` (GAZE marks what it sees), its flame as a GHOSTFIRE `#5a8aa0`, its lantern as a PUMPKINHEAD `#d08a20`. Relighting a lamp calls them home: they walk into the flame and scrub out.
- **THE DUSK** (the shadow line): night fang `#5a3a6a`, verdagger `#3a8a5a`, duskclaw, noctislicer, floating shade, frost fiend `#4a8ac0`. It is shadow growing *without* a beast inside it, and the Hollow herds it. Shadow is the last drawn thing before blank, so the Dusk walks ahead of the erasure like crows ahead of a storm. THE FROST FIEND's ULT is literally WHITEOUT, the Hollow's weather. The Dusk has colour (deep violets and greens), it boils, and it can be bound. Its crown is **THE VOID REAPER** `#5a3aa0`, lord of the Margin.
- **THE HOLLOW** stays an absence with no units. The **HOLLOWED** affix (one `AFFIX` entry in the game) covers all 87 kinds. A HOLLOWED capstone is the Margin's elite, and a HOLLOWED *owned* unit is the army's one real loss (§7.3.7).

**Standing reads your banner.** Owned units never change sides. But when you walk into a clash neutral, each side reads the capstones and defectors in your Lance:
- field a crowned Warchief, and the Warband opens at +1;
- field a Cinder Watch Black Knight, and the Watch opens at −1.

It is one lookup from fielded kinds into the relations table. Army composition becomes politics without a line of dialogue.

### 5.3 The lineages, root to capstone

`EVOLVE` splits into **21 connected lineages plus 3 loners** (the minotaur, the hidden blob and the summon-only mirror image). **17 terminal capstones** sit behind tier-5 edges. Notation: `@T` is the tier the edge needs. Accents are the CFG values.

| # | Line | Home | Root → … → capstone |
|---|---|---|---|
| L1 | **Watch** | Ashridge | watch `#9aa5b1` →@3 captain →@5 **THE SENTINEL** `#7fa0d0`; watch →@3 knight `#8fa8c8` →@5 **THE MIRROR KNIGHT** `#c0c8d8`, or knight →@4 templar `#d8c890` →@5 **★ THE CHRONO TEMPLAR** `#d0b040` |
| L2 | **Sword** | Abbey | swordsman `#c85a5a` or THE BLADE `#8fd3d0` →@5 **THE SPELLBLADE** `#40b0e0` |
| L3 | Spear | Brushlands | THE LANCER `#8fd3d0` →@4 pike `#a8b8a0` →@4 rider `#6a9a3c` *(ends at T4)* |
| L4 | **Warband** | Brushlands | orc `#6a9a3c` →@3 berserker `#b8412f` / →@4 elite orc; axeman `#8fa8c8` →@4 iron orc; all three →@5 **★ THE WARCHIEF** `#3f5a2a` |
| L5 | **Bone** | Palimpsest | skel `#d8d2c4` →@3 bone guard →@4 greatsword →@5 **★ THE GRAVELORD** `#a0a8b0`; bone guard →@5 **THE REVENANT** `#9aa0b0`; pale bones `#e0dac8` →@3 bone blade →@4 dark blade `#6a6a80` →@5 revenant; skel →@3 bone archer →@4 bone stalker `#8a8a9a` |
| L6 | **Pen** | Tower | wizard `#6f7fd8` →@4 mesmer `#9a5fd8` / →@5 lich; necromancer `#7fbf5a` →@4 warlock `#9a6fd8` →@5 lich; dark warlock `#6a4a9a` →@5 **THE LICH** `#3ac8c8` |
| L7 | **Chapel** | Abbey (heretic: Spill) | priest `#e8d9a8` →@4 hierophant `#d0b050` / →@5 **THE PLAGUE DOCTOR** `#5a9a3a` |
| L8 | **Cannon** | Abbey | cannoneer `#5a4a3a` →@5 **THE INQUISITOR** `#e0c060` |
| L9 | Bow | Ashridge | archer `#c9a86a` →@4 headhunter `#606878` or demon archer `#6a4a7a` *(ends at T4)* |
| L10 | **Burning Host** | Cinder Gate | demon lord `#6a2a6a` (★ root) →@5 **★ THE ABYSS LORD** `#2a2a8a` |
| L11 | **Forge** | the Kiln | lava slime `#c04a20` →@5 flame golem `#c05a20` →@5 **★ THE MAGMA COLOSSUS** `#8a1a1a` |
| L12 | **Choir** | Cinder Gate | gremlin `#5a8a4a` →@3 imp `#b05030` →@5 **THE DEMON BRUTE** `#7a3a2a` / →@4 succubus `#a04a8a`; blood fiend `#8a2a2a` →@4 succubus; harpy `#7a6a9a` →@4 succubus / →@5 **THE SIREN** `#40c8b0` |
| L13 | Iron | Cinder Gate | beam knight `#5a6a8a` →@4 black knight `#4a4a58` *(ends at T4)* |
| L14 | **Wing** | Hatchwood | bat `#6a5a78` →@3 hellbat `#8a3a2a` →@4 blood wing `#a03030` →@5 **★ THE STORM WING** `#3aa0e0` |
| L15 | **Pack** | Hatchwood | hellhound `#a04020` →@4 werewolf `#8fa0b8` →@5 **THE WEREBEAR** `#6a4a3a` |
| L16 | **Dusk** | Margin (nursery: Rows) | night fang `#5a3a6a` →@3 / THE KNIFE `#8fd3d0` →@4 verdagger `#3a8a5a` →@4 duskclaw `#3a3a4a` →@5 noctislicer `#2a2a3a` →@5 **★ THE VOID REAPER** `#5a3aa0`; shade `#4a4a6a` and frost fiend `#4a8ac0` (★ root) →@5 void reaper |
| L17 | Venom | Spill | venom slime `#c85a9a` →@4 venom king (★) or plaguebearer `#8a4fb8`; toxcrawler `#6a9a3a` →@4 plaguebearer *(ends at T4)* |
| L18-19 | Blot Kings | Spill | slime `#5aa83a` →@4 green king (★); frost slime `#3a7ac8` →@4 slime king (★) *(end at T4)* |
| L20 | Eye | Rows | eyeball `#4a7a8a` →@4 ghostfire `#5a8aa0` *(ends at T4)* |
| L21 | Lantern | Rows | pumpkinhead `#d08a20` →@4 **THE LANTERN KING** `#c04020` (renamed from THE HOLLOW KING) *(ends at T4)* |

Three facts about the graph shape play:
- **Seven lines end at T4.** Their tops (rider, headhunter, bone stalker, black knight, the kings, ghostfire, Lantern King) are cheap to field under COMMAND, and several can never ascend (§7.3.6). Their unit sheets say so.
- **Convergence points make branches into real choices.** An orc may take the berserker at T3 *or wait* for the elite orc at T4, because both end at the warchief. Three Redrafter roots all end in THE LICH.
- **Enemy unlock values that run backwards are fixed in the game's CFG copy.** The black knight (7) sits under the beam knight (9), and the werewolf (5) under the hellhound (6). Children are raised to at least their parent's unlock, so region bands read cleanly.

### 5.4 Defections: the evolution graph as the world's history

Fifteen edges cross v30's sprite groups. Each is a migration in the world *and* a choice when you evolve an owned unit.

| Edge | In the world | With your unit |
|---|---|---|
| axeman →@4 iron orc | Watch axemen desert to the Warband. | Watch squads bark "deserter". The Warband opens one step warmer. |
| THE LANCER →@4 pike →@4 rider | A Signed takes a Watch spear, then rides with the orcs. | You decide which side of the Brushlands your Lancer ends on. |
| THE KNIFE →@4 verdagger | The man snuffing lamps is becoming a creature of the Dusk. | Evolving him is his dark ending. Leaving him at T3 is his redemption. |
| THE BLADE →@5 spellblade | He wants to be varnished as the Order's perfect figure. | You can give him what he wanted. |
| knight →@4 templar / →@5 mirror knight | Every Watch knight chooses: take the gilt at T4, or hold out to T5 and become himself twice. | The Watch's only fork between two houses. |
| archer →@4 headhunter / demon archer | Bows for hire: the Watch's bounty or the Kindled's pay. | A mercenary's choice. |
| hellhound →@4 werewolf →@5 werebear | Hounds that run out of the Scorch cool into wolves in the Hatchwood's shade. | Fire into fur. |
| bat → hellbat → blood wing → storm wing | Bats that roost in the Kindled's smoke go red; those that fly above it become the storm. | Wild, then Kindled, then above every house. |
| priest →@5 plague doctor | An Order healer learned that contagion can be used on purpose. | A heresy the Order remembers (−1 Order standing while fielded). |
| lava slime →@5 flame golem; toxcrawler →@4 plaguebearer | Ink that caught fire; crawlers bloated on Spill ink. | |
| imp / blood fiend / harpy →@4 succubus; wizard / warlock / dark warlock →@5 lich; shade / frost fiend →@5 void reaper | Three roads into temptation; every Redrafter road ends in the same man; a ghost and a blizzard both end in the Void. | |

**The Signed keep their signature through every redraw.** When a Signed evolves, `setLook(e,label,tint)` carries its name and teal accent forward (`serialize` → `tint`, `spawnFrom` → `setLook`). So THE LANCER on a pike's body still reads THE LANCER in teal. **The world uses the same edges between visits:** a winning Warband front means the next Watch squad on that sheet has one fewer axeman, and the camp has one more iron orc. That is "the front moved for reasons the player can name", straight from the graph.

### 5.5 The Folio: 10 regions on 12 sheets, a capstone lord in every one

**The world is the Folio, a bound sketchbook of sheets** (v1, unchanged in shape).
- **A sheet** is about 128 × 128 tiles, crossed in 60–90 s of real play.
- **Crossing an edge turns the page:** the travel plate (`portalSim`) plus the page-tilt camera beat.
- **Each sheet has one Great Lamp and 3–5 roadside lamps.**
- **The map is the drawing:** the pause page shows each sheet's edge-plane canvas, drawn small.
- **A sheet loads whole behind its page turn.** Only people stream.

The count is settled at **ten regions on twelve sheets**. Two regions take two sheets: the Brushlands (the War Camp and the Long Ride) and the Scorch (Cinder Gate and the Kiln).

```
          WEST                    ·                     ·                       EAST
 NORTH  [ THE HATCHWOOD     ][ VELLUM ABBEY       ][ THE TRACING TOWER  ][ BRUSHLANDS: WAR CAMP ]
        [ ★ STORM WING      ][ ★ CHRONO TEMPLAR   ][ THE LICH           ][ ★ THE WARCHIEF       ]
        [ THE MARGIN (+back)][ ASHRIDGE JUNCTION  ][ THE PALIMPSEST     ][ BRUSHLANDS: LONG RIDE]
        [ ★ VOID REAPER     ][ THE SENTINEL       ][ ★ THE GRAVELORD    ][ (Warchief's 2nd sheet)]
 SOUTH  [ THE SPILL         ][ THE LAMPLIT ROWS   ][ SCORCH: CINDER GATE][ SCORCH: THE KILN     ]
        [ THE PLAGUE DOCTOR ][ THE SIREN          ][ ★ THE ABYSS LORD   ][ ★ MAGMA COLOSSUS     ]
```

- **Ashridge is central.** It touches the Abbey, the Palimpsest, the Rows and the Margin.
- **The Tower sits between the Abbey and the Warband,** because it uses everyone.
- **The Palimpsest is the most fought-over sheet,** between the town, the Tower (which raises its dead) and Cinder Gate.
- **The Kindled hold the south-east** and press north into the Rows, one sheet from town.
- **The Margin is the page's western edge,** and every outer sheet has torn edges where the Dusk walks.

**Region tier** comes from the CFG `unlock` band of the kinds found there, the same `(c.unlock||1)>n` test `composeWave` already runs. It sets **THREAT (enemy tier)** and **WEIGHT (enemy level)** through `spawnFrom({kind,tier,level},x,z,{team:2})`:

| Region band | Enemy tier (THREAT) | Enemy level (WEIGHT) | HP factor from `applyMods` |
|---|---|---|---|
| 1–2 | T1–T2 | 1–3 | 1.00–1.04 |
| 3–4 | T2–T3 | 3–6 | ~1.05–1.10 |
| 5–7 | T3–T4 | 6–10 | ~1.10–1.30 |
| 8–10 | T4 | 10–15 | ~1.30–1.40 |
| 11–12 | T5 | 15–20 | up to 1.59 |
| Lords, champions | T5 | 20 | 1.59, HARDENED −8% taken |
| Endless Vigil | T5 | past 20 | uncapped |

Level 20 lands exactly on v1's WEIGHT clamp of 1.6. **The main game never spawns an enemy above level 20.**

| Region (sheets), band | Hand (always full colour) | Home lineages | Conflict | **Capstone lord** (live evolution) · champions · others |
|---|---|---|---|---|
| **Ashridge Junction** (1), 1–6 | Today's town exactly: green field, red and blue roofs. The most coloured place in the world. | L1 Watch, L9 Bow, swordsman and axeman roots | The Well under the North Gate, and every road out crossing into a different house. | **THE SENTINEL** (a drowned captain →@5 sentinel, in the Warden's fourth beat). Story boss THE WARDEN. The player's first crown, WARDEN OF ASHRIDGE. |
| **The Lamplit Rows** (1), 2–9 | Night-blue shadow between lamps, colour pools under each lamp, hatch only in true shadow, forced `nightfall`. | L20 Eye, L21 Lantern, the Dusk nursery (night fang, verdagger), bats | THE KNIFE snuffs lamps for pay. Each snuffed lamp lets loose its eye, flame and lantern, and the Kindled's Choir sings the lamplighters off their posts. | **THE SIREN** (harpy →@5 siren; CHORUS turns bodies for 8 s). Street boss THE LANTERN KING (T4, bindable). Signed: THE KNIFE. |
| **Vellum Abbey** (1), 5–12 | Gold rims on every outline, **the boil held** for the whole sheet. Things boil only when struck. | Templar branch of L1, L7 Chapel, L8 Cannon | The Varnish: the Order is rehearsing a rite to hold every line still forever. | **★ THE CHRONO TEMPLAR** (templar →@5; REWIND, CHRONOBREAK). Champion THE SPELLBLADE. Head of state THE HIEROPHANT. Signed: THE BLADE. |
| **The Tracing Tower** (1), 5–11 | Every prop drawn twice, the copy faint and boiling out of phase. | L6 Pen | The Trace: copying the Draftsman's hand from the oldest lines in the Folio. MASS HYPNOSIS flips bodies back and forth all fight long. | **THE LICH** (warlock →@5 lich; PHYLACTERY, DEATH WINTER). Lieutenants THE MESMER and THE WARLOCK (T4). |
| **The Palimpsest** (1), 2–12 | Full colour with ghost double-lines of what used to be there, heavy decals. | L5 Bone | Three ways: the Inquisitor's crusade burns armatures, necromancers raise them, and the armatures want to be people. **The ground remembers:** kill puddles persist, and the Gravelord's rise raises your old battles. | **★ THE GRAVELORD** (greatsword →@5; MASS GRAVE). Champions THE REVENANT (the Old Watch Barracks) and THE INQUISITOR (PURGE). |
| **The Brushlands** (2), 4–10 | Fat wet brush, flat washes, long grass, a faster boil near the drums. | L4 Warband, L3 Spear | *War Camp:* the Warband burns lamp-ink as war paint. *Long Ride:* their home sheets past the far margin are going blank, and the Kiln's fire presses on the south. | **★ THE WARCHIEF** (elite orc →@5; RAMPAGE). Signed: THE LANCER. THE MINOTAUR chained in the camp pit (a bindable tyrant). Silence the drum, or fight with it. |
| **The Hatchwood** (1), 2–12 | The densest trees, deep saturated greens and blues, heavy hatch in shadow, `fog`. Lightning is the only light under the canopy. | L14 Wing, L15 Pack | The Hatch multiplies as lamps go out elsewhere. Hellhounds from the Scorch cool into wolves in the shade. | **★ THE STORM WING** (blood wing →@5, above the canopy; THUNDERHEAD lights the forest in nine flashes). Champion THE WEREBEAR (EARTHSHATTER). |
| **The Spill** (1), 1–11 | Oversaturated rainbow washes, drips and slicks: more colour than Ashridge, all of it loose. | L17 Venom, L18–19 Blot Kings | The Well's crack leaks downhill. The Order sent physicians, and one learned to *spread* plague. The Kindled want the ink for fuel. | **THE PLAGUE DOCTOR** (priest →@5; PANDEMIC), the Order's heretic. Great beasts: THE THREE KINGS (the Green King, the Slime King and the Venom King, fought as the Spill Vigil's boss wave) and THE PLAGUEBEARER. |
| **The Scorch** (2), 3–12 | **Too much colour:** saturated reds, oranges and violets, hot orange rims, a heat-shimmer boil, embers. Where it has burned through: **char holes**, hard-edged and dark, never white. | L10 Burning Host, L12 Choir, L13 Iron, L11 Forge | *Cinder Gate:* the first lamp ever fed with colour, and the Cinder Watch's betrayal. *The Kiln:* where the Kindled make fire from stolen ink, with lava slicks and forge glow. | **★ THE ABYSS LORD** (demon lord →@5; OBLIVION pulls toward a char hole) at Cinder Gate. **★ THE MAGMA COLOSSUS** (flame golem →@5; MELTDOWN) at the Kiln. Champion THE DEMON BRUTE (BLOODBATH). |
| **The Margin** (1 + the back of the page), 3–12 | The most erased land: blank bites everywhere, lamps as islands of colour, your watched circle. Frost weather is paper-clean, never grey. Forced `frenzy`. | L16 Dusk | It grows whenever any region is untended. The Dusk walks ahead of the erasure. | **★ THE VOID REAPER** (noctislicer →@5; EVENT HORIZON). Tyrant THE FROST FIEND (WHITEOUT). Story boss THE HOLLOW KNIGHT on the back of the page. Champion THE MIRROR KNIGHT on every back of a page. |

**All 17 capstones are placed, so every capstone edge can be opened by a fight you can win:**
- **Eleven lords**, one per sheet with a Great Lamp, except the Brushlands' two sheets share THE WARCHIEF:
  - boss-flagged: the Storm Wing, Chrono Templar, Gravelord, Warchief, Abyss Lord, Magma Colossus and Void Reaper;
  - not boss-flagged: THE SENTINEL, THE SIREN, THE LICH and THE PLAGUE DOCTOR.
- **Six champions:** the Mirror Knight, Spellblade, Revenant, Inquisitor, Demon Brute and Werebear.

The Long Ride's Great Lamp is held by the Warchief's war band. It is claimed by a Vigil against his lieutenants, and the Warchief himself is fought at the War Camp.

### 5.6 The capstone's double role: the lord you fought is the unit you become

**The fiction.** An artist draws a figure again and again, each study stronger, until one drawing is *finished*. Every lineage is a study sequence, and its capstone is the finished figure the Draftsman meant. The lords are those figures. They took the land their drawing was made for, then kept their drawing and let the lamps go out: hoarding colour (the Kindled burn it, the Order varnishes it, the Gravelord keeps the dead) instead of caring for it.

**You cannot take a lord's drawing. You can only draw it again yourself.** When your unit becomes THE GRAVELORD, you have made a second finished figure of the line, and this one keeps the lamps.

**The rules.**
1. **Capstones are grown, never bound.** None of the 17 terminal T5 kinds can be spared onto your side: lord, champion or Hollowed. The only way to own one is to raise a unit of its line to tier 5 and `evolve` it.
2. **Everything else can be bound,** at one tier below the body that knelt (§7.3.2). That includes the boss-flagged roots (THE DEMON LORD, THE FROST FIEND), the minotaur and the slime kings. **Binding a Demon Lord is the only road to your own Abyss Lord.** Demon Lords lead Kindled raiding parties in the Palimpsest and the Rows as clash leaders, and one of those can kneel.
3. **You cannot draw what you have never seen finished.** In the ARMY page's EVOLVE tab:
   - an edge into an ordinary kind shows its silhouette at SEEN and opens at STUDIED;
   - an edge into a capstone stays a silhouette captioned *a figure you have not seen finished* until that page is **BESTED**, which for a capstone is stamped only by winning its lord or champion fight.

   The demo's evolve tab already draws locked futures as a "?" card. The game swaps in the silhouette, then the portrait.
4. **Lord fights end with a live evolution.** Each lord is met in its parent body, spawned at T5 L20 so the edge is legal (`evolve` returns null below `opt.tier`). At the phase threshold (50% health) the host calls `HVU.evolve(e, into)`. The body scrubs out, the rite plays, and the capstone steps out at full health: a second phase the module already knows how to do. `evolve` keeps `team` and `hunt`. The fight's host re-points `M.lock`, `P.lockE` and the boss bar to the returned body. Champions are fought already finished, as duels.
5. **Beaten, a lord offers two endings.**
   - **TAKE THE CROWN (execute).** You gain its Law (§7.2), and the crown is vacant.
   - **LET IT KEEP THE LAMP (spare).** It withdraws to its Great Lamp and tends it for you. The region's Defend Vigils stop and its house gains +2 standing, but you get no Law and the crown stays with the lord.

   Either ending stamps BESTED and opens the capstone edge. Neither makes the lord yours.
6. **The crown.** If a lord's crown is vacant, **the first of your units to evolve into that kind is crowned.** It gets a `label` ("THE GRAVELORD OF THE PALIMPSEST"; `serialize` already carries `label`) and the crown glyph. You may own several Gravelords, but only one is crowned. Champions have no crown.
7. **"Warden" is a post.** A crowned unit can be **posted as WARDEN of its region's Great Lamp**. While posted, that sheet is tended: no Defend Vigils, and no lamp inside it gutters. But the unit is not in your Lance. **Your best units either hold the land or fight beside you, never both.**

### 5.7 The hub: Ashridge Junction

The town you have now, rebuilt building by building with ink.
- **The Watch-house:** the Lamp Board (the Folio map), the bounty board (contracts on Crowned elites), and the Chronicle, a quest log assembled from `renderPanel` copies of your own play.
- **The Barracks:** the ARMY page as a place. Resting units stand in the yard as idle bodies (at most six drawn; the rest are chips). Here you rename, re-tint from the family palette, and move items between units.
- **The Smithy:** sells unit weapons and trinkets for Ink (§7.3.8).
- **The Scribe:** equip Hallokin's techniques, traits and charms; rank techniques.
- **The Chapel:** last-stand Vows and the flask.
- **The Record:** the marks wall (16-marks), beside the Roster's 85 visible pages. The hidden `mirrorimage` and `blob` are folded into the Mirror Knight's and the Slime's pages.

Rites and evolutions need no building: **any lamp you have lit is a place of rites**, and Ashridge has eight of them.

### 5.8 The two story bosses

Both stay **outside the unit system** (game enemies in 19b, `EBRAIN`), so neither can ever be owned or evolved into. Fighting either with a Lance needs patch X1, `host.foes()` (§9.2), because v30 units cannot see the game's own enemies.

**THE WARDEN, the Blot that wears a post.** Ashridge's great inkwell, the Well, holds the ink every lamp burns. A generation ago *the Watch* posted its greatest Sentinel as Warden of the Well. The Well cracked, the Sentinel went in after the ink, and the ink kept the post. The boss is the game's giant rainbow slime: three phases, the triple charge, the slam that throws kittens, IT REELS / IT RAGES, and every shipped string (the wave card, WARDEN SLAYER). It still guards, still charges like a shield-bearer, and throws out slimes because a blot splits.

**The fourth beat** is the region's lord. A drowned CAPTAIN steps out of the ink (a real HVU `captain` at T5 L20). At half health it redraws live into **THE SENTINEL**, the unit keyed `warden`, whose CFG `name` becomes THE SENTINEL in the game's copy; BULWARK and AEGIS keep their names.
- **Execute it:** you take BULWARK as a Law, and the post of WARDEN OF ASHRIDGE is vacant.
- **Seal it:** it walks back to the North Gate lamp and keeps Ashridge for you, and Watch standing jumps.
- **Either way, the Well's ink flows back to the lamps,** so lamps can burn beyond the town, and the Sentinel edge on every watch → captain you own opens. **The first capstone a player can earn is the title the Well stole.**

**THE HOLLOW KNIGHT, Hallokin before you.** Drawn from the skeleton family (`hollow`), with the knight's chain, a gap-closing dash with afterimages and a fan of bolts: both your studies in one body. He is seen from Chapter I as a line-only figure at the edge of blank land, and the companion's barks go silent near him. He is fought in his lair on the back of the last sheet (`EBRAIN.hollow`, mark FACE TO FACE), where the page starts half erased and every lamp you light mid-fight draws colour back in. Two of your Lance may follow you in.

### 5.9 The arc: four chapters and the Gutter

Every chapter ends on a results card retitled as a Chronicle page.

- **CHAPTER I — THE WELL.** Ashridge is the only coloured place nearby, and its lamps are burning their last ink. The first night is the shipped game: waves in the town, the Warden wave, the gate. Your first bound unit (a snuffer's watch, §6.1) takes the first rite at the town's Great Lamp. You go down the North Gate into the Well, beat the Warden, and meet the drowned captain's redraw. The ink flows back. First sight of a line-only knight at the edge of the paper, and the companion stops talking.
- **CHAPTER II — THE LAMPS.** The Folio opens. Relight it in any order: the Knife and the Siren in the Rows, the Blade and the Chrono Templar at the Abbey, the Lancer and the Warchief in the Brushlands, the Gravelord in the Palimpsest. Every Signed you beat or spare, every lord you crown or leave tending, pushes the blank back and moves the standings. At the Old Watch Barracks, sealing the Watch-grey skeletons and beating THE REVENANT reveals that they were the Hollow Knight's company, that the companion was his archer-study, and that **his name was yours.**
- **CHAPTER III — THE VARNISH, THE TRACE AND THE BURN.** Three plans to save the world, all terrible, come to a head on one night:
  - **the Order** performs the Varnish at the Abbey, and needs your domain;
  - **the Redrafters** trace the Draftsman's hand at the Tower, and need your companion, the only figure drawn by his own hand in two studies (THE MESMER takes her with MASS HYPNOSIS);
  - **the Kindled** feed Cinder Gate a signature, because a burned name burns forever, and come for your spared Signed with THE SIREN's CHORUS.

  **You can break only one rite in person. Your posted army holds the other two.** Each is a Defend Vigil resolved off-frame by the ledger: the summed `unitPower` of the Wardens and garrisons you have posted on that rite's sheets, against the rite's budget. A held rite fails cleanly. A rite that breaks through partly succeeds, and its sheets stay changed until the post-game:
  - Varnish: the boil held and the lamps unrelightable;
  - Trace: every body there charm-flipping;
  - Burn: char holes at the budget's limit.

  **The army you built is how the story is won.**
- **CHAPTER IV — THE HOLLOW.** You go into the Margin with the Lance you can command and the houses that will march. Allied units spawn beside you, and the Warband brings the drum if you gave them a sheet. THE VOID REAPER stands before the last page, and the Hollow Knight waits on its back. At the end you choose:
  - **ERASE** him, and the companion leaves you. The Watch you rebuilt stands at the lamps, and she is not among them.
  - **SEAL** him with a sigil loop while he is stunned. The knight-study and archer-study redraw into one figure, who gives you back the signature's history and walks past the last panel.
- **THE GUTTER.** The strip between two comic panels, walled by the letterbox bars: a corridor thirty seconds long. Past it is the Reference, pixel Ashridge with no ink and no boil, perfectly still, and no Draftsman. The last card reads **THE VIGIL IS KEPT · ENDLESS**. Post-game is the Endless Vigil, and the army's long tail: UNIQUE, MYTHIC and ASCENDED.

## 6. Playing it

### 6.1 The first 10 minutes

- **0:00 — The opening image.** A pen draws one lamp post, stroke by stroke on the 10 Hz clock. Then Hallokin, asleep against it. Then the roofs fan outward, ink first and full colour a beat behind, and the field floods green from the lamp. HOLLOW VIGIL is hand-lettered across the sky and tears away. About 12 seconds, skippable. The first thing the player learns, without a word, is that **colour comes from lamps**.
- **0:15 — Walk.** Two Watch guards (house Watch, team 3, not yours) chat in balloons that never pause you: *"Third lamp out this week."* *"Blame the Knife."* Each control appears once, as a small paper tag beside the knight, the first time it matters.
- **1:00 — The first fight.** Six slimes spill into the lamp square (scripted waves 1–2), and the two guards fight beside you. In one minute the player sees units fight units, earns a FROM BEHIND!, and watches the rank letter climb. The guards' kills print no numbers and freeze nothing: the feel gate is taught by never being noticed.
- **2:30 — The hook.** A teal figure crosses the rooftops: **THE KNIFE**, an eyes-in-a-slit cut-in for 0.6 s. He snuffs the lamp you woke under. The town does not go white. At the far end of the square **the paper tears in**: a ragged blank bite where a woodpile was, crawling toward the lamp. The colour is all around you still. What was taken is the thing protecting it.
- **4:00 — The chase.** Three streets. The Knife's paid **snuffers** block the way: two watchmen in Watch grey with soot on their hands, THE WATCH at **tier 1**. A T1 watch has exactly one move and plain AI, so it is the perfect first tell. The Knife tips a slime pot and fires SHADOWSTEP through you at a point the script picks. The host calls `HVU.ULT.knife.run` directly, because v30's ultimates fire on their own clock and never at a health line. He escapes over the gate.
- **6:00 — The first SPARE.** One snuffer is left at the dead lamp. Parry its single swing and a line inks onto its Roster page: *THE WATCH · 1/1 · STUDIED*. Stun it, and the execute prompt shows a second option for the first time. **Hold attack for 0.4 s.** The snuffer kneels in the last stand's slow-mo, and the splash reads **SPARED**.
- **6:30 — Relight and countersign.** Stand at the dead lamp for 1.5 s. The bite inks back in: fence, woodpile, grass, in three held drawings. In the new light the kneeling snuffer is **countersigned**. It scrubs out and redraws at tier 1, and an accent tick inks in under its feet. A portrait chip appears under your health bars. A 16-character field offers a name (type one, or accept the Chronicle's: *BRAMBLE*). The whole loop in miniature: **light a lamp, give a figure back, and it is yours.**
- **7:00 — The gate.** The first time through, it is a vista. The camera eases to the hero angle, and a slow 2.5 s dolly lets the valley rush away. "THE VIGIL VALE" is lettered in a thin bottom bar. The valley is in full colour, with three things standing out: a dark lamp tower to the north, smoke and drums to the east (the Warband), and on the southern horizon **a white tear in the world**. BRAMBLE walks at your shoulder on FOLLOW. The V dial's tag appears once.
- **8:30 — The first clash.** On the road, a Watch rider and two pikemen are fighting four skeletons. Nobody has attacked you, and a hand-lettered tag floats over the fight: **"WHOSE SIDE?"** BRAMBLE **holds** beside you, because the army never starts a war for you. Hit a skeleton, and the Watch side is chosen, a cut-in stamps THE WATCH STANDS WITH YOU, and BRAMBLE joins. Every kill you land pays BRAMBLE THE SHARE. After the third, a small **LEVEL 2** ticks over its head, and its chip pops.
- **9:30 — The first Vigil begins** at the tower (§6.5).

### 6.2 The first hour

| Time | Beat | What the player learns |
|---|---|---|
| 10–15 min | A three-wave Vigil at the Vale's watchtower: the existing wave banner, choice cards and budgeted rosters, relocated. BRAMBLE fights inside the lamp circle. At dawn one card is kept as a charm, and colour floods back into two bites of the valley. | Vigils claim land. The army trains in them. |
| 15–25 min | Free roam of the first sheet. A cracked stone that Iron Fall opens. Wanted posters for THE KNIFE, THE BLADE and a blank helmet (the Warden). A skeleton cart overturned on the road: parry a skeleton's one swing, stun it, hold attack, and it is **SEALED**. It cannot kneel because there is no one left to kneel, so it is redrawn and bound as a T1 skeleton. | Armatures are sealed, not spared. |
| ~25 min | The companion is found pinned down in a mill by bombers. Saved, she joins and hands you her spare bow: **the class swap unlocks**. | Two studies, one figure. She is not in the army. |
| ~30 min | A Warband scout, an orc at tier 2, is the first body you answer move by move (2/2). It kneels, and it joins at **tier 1**: a spared figure loses its last stage of ink. The ARMY page opens once to show three chips, and three accents walk behind you. | Binding scrubs one stage off. |
| ~35 min | **First faction clash:** 8 Watch against 8 Warband at a river bridge, THE CAPTAIN against an ELITE ORC, opened with a split-panel cut-in. Your orc is on the Watch's side of the bridge. The Warband barks at it. | The world has politics, and your army wears them. |
| ~45 min | **THE BLADE** steps out of a treeline at dusk: a letterbox slit, and a duel. **Your Lance waits at the panel's gutter**, their chips greyed with a small hourglass. He fires ASTRAL TEMPEST about nine seconds in, as he always will: a Signed's tell is a clock, not a health line. He flees, and remembers. | Duels are for the hands. |
| ~50 min | BRAMBLE's chip has worn a gold ▲ since the bridge. At a roadside lamp, the pause page's ARMY tab offers the **first rite**. Hold J: BRAMBLE holds still, the under-stroke sweeps its body, and *TIER 2* is stamped on the chip. The page opens on GROW, with 4 stat points, 3 skill points and a talent choice (BRUISER / FINISHER / LIFELINE). Dash and block ink onto its move list. | Tiers are stages of a drawing, and the rite is yours to give. |
| ~55 min | From a ridge: the Well's broken lid below the North Gate, and something huge moving in the ink. A vertigo dolly, the poster inks in, and you are not asked to fight it yet. | Some things are bigger than you. |

**At the end of hour one:** two lamps lit and a Great Lamp claimed; three owned units (a T2 watch, a T1 orc, a T1 skeleton) inside a COMMAND of about 300; four Roster pages studied; one rival who hates you; one faction that likes you; and a silhouette you are afraid of.

### 6.3 A session at hour 15

The player has lit 7 of 12 Great Lamps. COMMAND is about 3,000: roughly 2,400 from lamps plus 5 × about 120 Roster stamps.

Their Lance:
- **BRAMBLE**, a knight at L28, holding out at T4 for the Mirror Knight;
- **OSSIAN**, the skeleton sealed at minute 20, now THE GREATSWORD at T4 L29;
- **THE KNIFE**, spared at hour 6 and deliberately left at T3, keeping his redemption.

Standings: the Watch are allies, the Warband is split, and the Hollowed hold the south.

- **0:00 — Open the book.** The game resumes on the map spread. Since last session a blank bite has opened in the Palimpsest, and its Great Lamp is marked under siege. Three torn notes:
  - a bounty: *TAKE THE MILL BACK — the Watch pay in ink*;
  - a sighting: *THE LANCER at Crowfoot*;
  - an army note beside OSSIAN's chip: *one level to the signature.*

  Twenty seconds on the map, then a page turn to the nearest lamp.
- **1:00 — The walk is not dead time.** Drums from the left before anything is visible. A crow doodle in the right margin leads to a dog-eared corner, and Iron Fall flips it to the back of the page: dark, reversed, quiet. **A MIRROR KNIGHT** waits there. It is a champion duel, so the Lance waits at the gutter. MIRROR throws back 60% of your hits while it blocks, so the player learns to break its guard instead of hammering it. MIRROR IMAGE sends out two images at nine seconds. Beaten, its page stamps **BESTED**, and in BRAMBLE's EVOLVE tab the silhouette captioned *a figure you have not seen finished* inks into a portrait.
- **6:00 — The mill.** Six Watch hold against twelve Warband with an ELITE ORC and a BERSERKER. The player arrives neutral, and the Lance holds. First blow on an orc picks the Watch. V to CHARGE: the three accents surge into the orcs' flank. The elite orc hits 70% and roars WARCRY. The player answers with IRON VIGIL: **the domain freezes only hostile teams**, so the Watch and the Lance keep swinging inside the frozen circle. The rank reaches A, and a CALL pip fills. **Hold Q:** BRAMBLE fires CRUSADE on your command. It is the only ULT any owned unit fires, and it lands in the frozen crowd. The killcam goes on the elite orc, the leader. Twenty-odd kills across the Lance, and the screen froze only on the player's own hits. The results card reads *THE MILL — RETAKEN — S*, and one card lists the army's level-ups, folded.
- **12:00 — THE GRAVELORD.** The Palimpsest's lord waits at the Old Crossing: **a GREATSWORD at T5 L20.** The panel locks with a heavy line, and the player chooses two of the Lance at the panel's edge (OSSIAN and BRAMBLE; the Knife waits). At half health the greatsword **kneels, scrubs out and redraws**, and THE GRAVELORD steps out at full health. Every body that died within 8 tiles starts to rise, and so do the kill puddles from the player's *last* fight here. OSSIAN, the same drawing one stage short, fights beside you against the thing it wants to become. The player wins, and executes: **TAKE THE CROWN.** The Law MASS RISE goes on the domain. The page stamps BESTED, and OSSIAN's EVOLVE tab shows THE GRAVELORD in full colour.
- **20:00 — The Defend Vigil.** The Palimpsest's Great Lamp, six waves, with the Lance plus the two garrison units posted there (a bone archer and a pikeman who have been levelling in these sieges all along). OSSIAN reaches L30 in wave 3: a LEVEL 30 tick, MASTER folded into the chip, no hitstop, and the gold ▲. In the gap after wave 4 the three perk cards appear, **plus a fourth on key 4: RITE — OSSIAN, TIER 5.** The signature inks on in a held beat the wave clock waits for. At 20% health the last stand fires. They hold it, and the bite inks back in.
- **28:00 — The redraw.** The next lamp on the road out is the Crowfoot roadside lamp. The Vigil was this unit's rite for the last rest, so the evolution waits for this one. ARMY → EVOLVE → hold J. OSSIAN scrubs out and redraws as **THE GRAVELORD**, still named OSSIAN, in the tint the player chose at hour two. Its skill ranks are refunded, and the page opens on GROW: *the new body has new moves; spend your points again.* The crown is vacant, so it is crowned: **THE GRAVELORD OF THE PALIMPSEST**. Its power reads **5,016**, and the player's COMMAND is 3,000. **It cannot march yet.** The player posts it as **WARDEN OF THE PALIMPSEST**: that sheet stops needing Defend Vigils, and its lamps will not gutter. Bank at the lamp. The session page shows today's stamps, and the player stops at a clean end.

Four kinds of play (exploration, clash, lord, Vigil) and a redraw in thirty minutes, none longer than ten. The army was present in every one, and the hands decided every one.

### 6.4 Exploration: why walk over that hill

Kept from v1 whole, because none of it touches HVU:
1. **The blank is the invitation.** A white bite in a coloured field is an honest fog of war: a loss that is yours to undo.
2. **Marginalia instead of a minimap.** Points of interest outside the frame are doodled into the screen's margin on their side: a flame for an unlit lamp, a smoke curl for a camp, a skull for a lord, crossed swords for a clash happening now. v2 adds **a crown for a vacant crown**, and **a small portrait for a HOLLOWED unit of yours** somewhere on this sheet.
3. **Sound before sight:** taiko from a clash, a bell at a lamp, positional WebAudio.
4. **Landmarks tall enough to break the frame:** lamp towers, orc totems, the Abbey spire, the Cinder Gate flame.
5. **Secrets are ability-shaped,** never hard gates on the critical path: Iron Fall breaks cracked stones and flips dog-ears, the hook crosses gorges, bomb arrows collapse boarded walls. And now **army-shaped**: a posted Warden opens its sheet's sealed gate, and a fielded Lich reads the Tower's ciphered doors.

**Traversal feel.** Out of combat the run ramps to a jog. Dash, stinger and hook chain into a momentum string, and style rank does not decay on the road. The Lance follows on the leash (patch V3), and blinks to your side past 14 tiles like the companion, so nobody gets lost behind a house.

### 6.5 The Vigil

**A Vigil is the shipped wave game, kept whole and placed on a lamp:** typed pools, `composeWave` budgets, the curve, modifiers, affixed elites, bounties, the lamp objective, boss waves and choice cards. Its first role is unchanged: **a Vigil is how land is held.**
- **Claim:** every Great Lamp is lit by keeping a Vigil at it (3–6 waves).
- **Defend:** a lit Great Lamp left untended comes under siege. A posted Warden, or a lord you let keep its lamp, stops that sheet's sieges.
- **Endless:** after the finale, the Margin's Vigil has no dawn.

**Its second role is new: a Vigil is where the army trains.** Vigils are the densest kills in the game, and THE SHARE pays every fielded unit, garrisons included. Each lit Great Lamp holds **two garrison units**, spawned at the lamp for that lamp's Vigils. **The bench does not level**, so rotation is a real decision: posting a young unit on a lamp you expect to be besieged is how it catches up.

**Rosters.** Until patch X1 lets units see the game's own enemies, any Vigil with an army fielded draws its pools from HVU kinds only. The module has the bodies (slime, frost and venom slimes, their kings, the lava slime), so a Spill Vigil still looks like the Spill. The Ashridge Vigil with no army fielded is the shipped game, byte for byte, which makes it the regression test.

**Cards and rites.** Cards stay run-scoped: picked between waves, stacked, and wiped at dawn by `snapT` / `restoreT`. At dawn one card is kept as a charm. When any fielded or garrison unit has `tierReady`, the gap offers a **fourth card, RITE, on key 4**, which advances that unit (§10.4). Die in a Vigil, and the night's cards are lost, the lamp stays dark, and the garrison is scrubbed for the night.

**Budget.** The fielded army adds `0.4 × fielded unitPower / 100` to the wave budget. The extra buys **tier and affixes, not bodies**, because the body caps are fixed (§8.1). A stronger army meets meaner fights, not bigger crowds.

### 6.6 Faction clashes and rivals

**Clashes** (v1, with the army's rules added):
- **You arrive neutral.** `P.team=-1` makes `pickTarget` and `dealHit` leave the hero alone. That skip exists only for the player: a *unit* on team −1 is still a target. So **the Lance goes neutral by holding**: aggro off, no `hunt`, and a home point on your position. Nobody targets it, and it starts nothing.
- **Your first damaging blow picks the side.** A two-sided cut-in stamps it, and the Lance releases. The side you hit becomes hostile for this fight.
- **One stray hit is forgiven** with a "hey!" balloon, and a second flips that side. The hero cannot hit owned units at all, so forgiveness only concerns sided house allies.
- **Or pick neither.** Let them grind each other down and clean up the winner: more loot, less standing.
- **Every clash has a leader** with a named ULT. That ULT is now a timed beat (about 9 s engaged, or crowded), so the clash's climax comes early and the leader is the priority kill. Killing it routs its side, and the killcam goes on the leader.
- **Your banner is read** as you arrive (§5.2).

**Rivals remember, and now they grow.** A rival is a build object. Between meetings the host raises its `level` and gives it a counter from real tables:
- **BULWARK** (T3 talent) if you guard-break it;
- a **MASTERY** on the move you keep eating (IMPALE on the Lancer's thrust);
- **EAGER** (T4) if you kite out its ULT.

The Knife opens with smoke if you always parried him. **Rivals are spared only after their duel,** and bind at their duel tier minus one. Duel tier is the region tier plus one, capped at 5: the Knife in the Rows arrives at T2 or T3. Beating a Signed also gives its PASSIVE to Hallokin as a trait, whether or not you spare it.

**THE MESMER and THE SIREN are the villains of allegiance.** MASS HYPNOSIS and CHORUS can flip half your Lance for 6–8 s. A fight you must win without killing your own charmed Gravelord costs nothing new to build. Patch V5 makes sure a charmed body earns nobody XP (§9.2).

### 6.7 The four stories a player tells a friend

1. **"I walked into a war."** *"Twenty guys on a bridge. I sided with the orcs because the captain shot me earlier. When he fired LAST STAND I dropped IRON VIGIL on the bridge, his whole side froze, my three kept swinging inside the circle, and I CALLed my knight's CRUSADE into the middle of it."*
2. **"The Knife remembered me."** *"He ambushed me for hours and started opening with smoke because I always parried. I finally spared him. Ten hours later he's mine, level 34, still in teal, and I never evolved him, because I didn't want him to go dark."*
3. **"I kept the page alive."** *"In the Margin the only colour is a circle around you as big as your style rank. I got to SS and the whole field drew itself back in while I fought."*
4. **"My skeleton became the lord."** *"The skeleton I sealed off a cart in the first twenty minutes watched the Gravelord redraw himself in the middle of the fight. Two lamps later it took the same rite. It's a Gravelord now, it has the name I gave it, and it's crowned: Warden of the Palimpsest."*

## 7. Progression: the hands and the army

**One sentence: the hero knows, the army grows.** Everything that says "+%" lives on units. Everything that changes what Hallokin can *do* is a verb, a rule or a piece of knowledge.

| Question | Hallokin (the hands) | The army |
|---|---|---|
| What gets stronger? | His options: techniques, traits, charms, Laws, Vows | Its numbers: level, tier, stats, skills, talents, gear, form |
| From what? | Knowledge (the Roster), duels, lords, Vigil dawns | Kills (THE SHARE, plus their own), rites at lamps, lords beaten |
| Is there a level? | **Never.** No XP bar, no stat points. | Levels 1–50 on v30's `XP_TO`, untouched |
| What caps it? | Slots (4 techniques, 3 + 1 traits, 3 charms, 1 Law) | COMMAND (fielded power), the owned cap, two garrison per Great Lamp |
| What does death cost? | Unbanked ink, left as a pool where you fell | Unbanked XP and 10 renown, or the whole unit if it hollows and you fail to seal it |

### 7.1 Why the hero never levels

- **v30 already draws the line.** In its own demo the hero damages units through `HVU.hurt(u,{…,by:null})`, so his kills credit nobody. `grantXp` returns unless `e.team===1`, and the hero has no sheet.
- **The tuned combat stays honest.** The knight's 10–26 damage under a 3× crit ceiling, the 34% execute, stun and poise by hit count, the swap-weave ×1.35, frame-derived hitboxes: all are fixed numbers. Fifty levels of +2% (×1.98) plus MIGHT 8 (×1.32) would drift every one.
- **v30's enemy dials are v1's difficulty dials.** Tier is THREAT (`tierHas`, and the cooldown multipliers 1.3 / 1.15 / 1 / 0.92 / 0.85). Level is WEIGHT (`applyMods`, +2% health and damage per level). v1 invented neither; they already exist.

v1's rule survives as a rule about the hero, not the world. **The Tally** (Hallokin's stamp count) is never called a level. It feeds COMMAND.

### 7.2 Hallokin: the Roster, techniques, charms, Laws

**The Roster: progress as knowledge.** Every visible kind (85) gets a page. The demo's `#roster` already lays out each kind's moves, passive, ult, "evolves into X at T4" and the T1–T5 kit ladder: the content is written. A page takes four red stamps (`stampR`):

| Stamp | Earned by | Gives |
|---|---|---|
| **SEEN** | First sight. | Name, silhouette, health line. Evolution edges *into* it show a silhouette. |
| **STUDIED** | *Answering* every move on its list once: a parry, a perfect dodge, a full guard, or a punish crit in its recovery. Each answer inks a line ("THE LANCER · 3/5"). Kits are 1–4 moves, gated by tier, so a T1 body is studied in one answer. | Its PASSIVE sentence; +8% damage against the kind; **it can now be spared**; **ordinary evolution edges into it open.** |
| **BESTED** | Executing it three times, **or, for a capstone, winning its lord or champion fight.** | Its technique unlocks at the Scribe, if it teaches one. **A capstone's evolution edge opens.** |
| **BOUND** | Owning at least one unit of the kind. | A gold rim is added when any unit you own of that kind reaches ASCENDED. |

Grinding does nothing: a hundred skeleton kills give one BESTED stamp. **COMMAND gains 5 per stamp.** A thorough playthrough holds about 200 by the finale.

**Techniques have teachers** (v1, re-checked against v30). BESTED on a page unlocks a technique whose move visibly rhymes with the teacher's:
- IRON FALL from THE IRON ORC's slam;
- INK CYCLONE from THE AXEMAN's WHIRLWIND;
- CRESCENT WAVE from THE BLADE;
- HOOK LINE from THE HEADHUNTER's snare;
- HOLLOW RAIN from THE CAPTAIN's volleys;
- BOMB ARROW from the bomber kittens;
- SKETCH SIGIL from the sigil caster;
- **SCRIBBLE DOUBLE now from THE MIRROR KNIGHT** (MIRROR IMAGE is a gesture copy of yourself; THE MESMER was the weaker rhyme).

New techniques are built from the player's own frames plus ink VFX, with no new player sprites: SET SPEAR, SHADOWSTEP, NOVA, SANCTUARY SIGIL, RAISE, WARCRY. About 14 in all, on the two-hand loadout (2 on the blade hand, 2 on the bow hand). **Ranks I–III** follow the perks' three-stack model on the numbers `ABIL` / `AB` already expose. The hero gets no MASTERY choice; that grammar belongs to units. Loadouts swap free at any lit lamp.

**Traits wear a body's sentence.** Three slots filled from PASSIVE lines that already speak the game's crit vocabulary: BACKSTAB, FIRST BLOOD, EXECUTION, SECOND WIND, HEADSHOT, WARD, IRON HIDE, and v30's KEEN, IRON WILL and PLATED. They come **from beating a Signed** (the duel's prize, spared or not). **A fourth, lent slot:** while a T5 unit is fielded, Hallokin may wear its PASSIVE. This replaces v1's Oathbound. Aura passives (STANDARD, ZEAL, BULWARK, WARDRUM) stay on the units, where team scans already make them work.

**Charms: the cards, kept.** The twelve PERKS stay cards inside a Vigil (offered between waves, stacking to three, wiped at dawn). **At dawn you keep one** as a charm at rank I, and keeping it again ranks it to II and III. Three slots. Charms are the only flat stats on the hero, and they never touch units: units have ITEMS and STATS. The two stat economies stay separate.

**Laws bend the domain.** Each lord you **execute** gives one, through a `LAW` table and one hook in `ultStep`:

| Lord | Law |
|---|---|
| THE SENTINEL | BULWARK: frontal hits halved, and bodies inside reel |
| THE GRAVELORD | MASS RISE: hostile bodies that die inside rise on your side until the domain closes |
| THE WARCHIEF | WARDRUM: your side swings 15% faster inside |
| THE CHRONO TEMPLAR | REWIND: the domain's first hit on you un-happens |
| THE LICH | HARVEST: kills inside refill energy |
| THE SIREN | CHORUS: the first two hostiles inside are charmed for its duration |
| THE STORM WING | THUNDERHEAD: the domain strikes the three highest-power hostiles once |
| THE ABYSS LORD | OBLIVION: hostiles are pulled toward its centre |
| THE MAGMA COLOSSUS | MELTDOWN: bodies inside burn |
| THE PLAGUE DOCTOR | CONTAGION: a mark spreads to the nearest hostile on death |
| THE VOID REAPER | EVENT HORIZON: the domain's radius grows with each kill |
| THE HOLLOW KNIGHT (post-game) | THE FACE: a gold ghost of you repeats the teleport finale |

Every boss permanently changes your biggest moment, not your health bar. The domain freezes only hostile teams, so the Lance keeps fighting inside it.

**Vows and the flask** (v1): healing stays scarce. A flask of hearts is refilled at lamps. The last stand is a Vow, refilled at lamps, with extra Vows from the Chapel (max 3).

### 7.3 The army: v30's full ladder

#### 7.3.1 The ladder, as the code has it

| Table | Value |
|---|---|
| `XP_TO(l)` | `round(12·l^1.35)`. Cumulative XP to reach L5 is 174, L12 1,587, L20 5,491, L30 14,528, L40 28,848, L50 49,028. |
| XP per kill (`die()`) | `max(6, round(power(victim.cfg)/8))`, to the killer only, and only if the killer is team 1. **It ignores the victim's tier and level.** 6 for 27 kinds; the knight 11, the warchief 41, the gravelord 54, the abyss lord 77. Renown +1. |
| Level-up | +2% health and damage, heals 25%. Stat points: `POINTS_AT(tier)` {0,2,4,6,9} + 1 per 2 levels, cap 8 per stat. Skill points: {1,3,5,7,9} + 1 per 5 levels. |
| Tiers | Flagged at L5 / 12 / 20 / 30 (`readyTier`), taken by rite (`advance`). **Tiers never rise by themselves.** The guide's "3, 5, 7, 10" is stale. |
| Skills | Ranks 1–3 per move (+10% damage, −8% cooldown, +5% reach each). At rank 3, a MASTERY pair by move type: CLEAVE / RUPTURE, IMPALE / RUN THROUGH, TWIN / PIERCING, LEGION / HARDY… |
| Talents | One pick per tier from T2: T2 BRUISER / FINISHER / LIFELINE, T3 OPENER / BULWARK / MOMENTUM, T4 EAGER / BRUTAL / ENCORE, T5 JUGGERNAUT / DUELIST / WARLORD. |
| Milestones | L10 VETERAN +5%; L20 HARDENED −8% damage taken and a second artifact slot; L30 MASTER a free rank per move; **L40 UNIQUE** (rises once per fight, ULT twice, allies within 6 tiles +10%); L50 MYTHIC +10% to everything. |
| Titles (`unitPower`) | ELITE 150, CHAMPION 300, WARLORD 500, LEGEND 800. |
| Ascension | `ascendAt = max(1400, 1.3 × fullBuildPower(kind))`. Renown (1% per own kill, up to 60%) closes the gap. ASCENDED: +20% health and damage, 20% less taken, cannot be knocked down. |

Evolution, not levels, is where power comes from:
- the same T3 L12 build doubles from a skeleton (101) to a bone guard (215);
- the T5 capstone step is roughly ×4–7 (a greatsword at 745 becomes a gravelord at 5,016, with 3,089 HP next to Hallokin's 100).

That scale is exactly why COMMAND exists.

#### 7.3.2 Binding

At the SPARE prompt (hold attack 0.4 s) or the SEAL prompt (the same hold on an armature at T3 or below; a closed SKETCH SIGIL loop on T4+ armatures and the Hollowed), the kneeling body is removed and the host writes a **fresh build**:

```
{ kind, tier: bindTier, level: FLOOR[bindTier], xp: 0, renown: 0, ascended: false,
  items: [], stats: {}, skills: {}, talents: {}, loadout: null, label, tint: null }
bindTier = max(1, enemyTier − 1), capped at 4        FLOOR = {1:1, 2:5, 3:12, 4:20}
```

- **Never `HVU.serialize(e)`,** which would copy an enemy's tier and level.
- **Never `HVU.spawn(kind,…)`:** `spawn` defaults to team 1 *and* tier 5 (full kit, 9 stat points at level 1) and ignores `level`. The body is created with `spawnFrom(build, x, z, {team:1})`.
- **Why one tier below.** A spared figure is redrawn and loses its last stage of ink. It keeps evolution meaningful (a T4 bind skips 5,491 XP), and it is catch-up: a region-8 bind arrives at T3 L12 and can be fielded at once.
- **Unspent points are the reward screen.** A T2 L5 bind arrives with 4 stat points, 3 skill points and a talent pick, and the ARMY page opens on it. Binding is where the player *makes* the unit.
- **Capstones and lords are never bound** (§5.6). The Signed bind after their duel.
- **The owned cap is 6 + 2 × Great Lamps lit** (30 at 12). Territory is army size.

#### 7.3.3 THE SHARE: the hands feed the army

**Every hostile kill inside the encounter panel made by Hallokin, the companion or the Lance pays each alive fielded unit, garrisons in a Vigil included, 40% of the victim's XP value × `RMUL[STYLE.rank]` at the moment of the kill,** through the exported `HVU.grantXp(u, n)`. The killing unit's own module grant still happens on top. Kills by sided house allies pay nothing, or a big clash would farm.

- **Fighting well with your own hands levels your army faster.** At SS the Share doubles, and at D it pays the base.
- **Renown is not shared.** It comes only from a unit's own kills, so ascension still means the unit fought.
- **The Share never fires during a charm.** Patch V5 credits only `e.owned`, not team 1 (§9.2).

#### 7.3.4 COMMAND: what can walk beside you

**The summed `HVU.unitPower(u)` of the fielded Lance must not exceed COMMAND.** COMMAND is Hallokin's only number. It comes from Great Lamps lit plus 5 per Roster stamp, never from XP.

| Great Lamps lit | 0 | 2 | 4 | 6 | 8 | 10 | 12 | post-game |
|---|---|---|---|---|---|---|---|---|
| COMMAND from lamps | 150 | 400 | 900 | 1,800 | 3,000 | 5,000 | 8,000 | uncapped |
| What fits | three T1–T2 units (29–60 each) | three T2–T3 | a knight (345) + a bone guard (215) + a T3 | a templar (880) + a greatsword (745) | three T4s, or the Blade at T5 (1,115) + two T4s | **one gravelord (5,016)**, or a chrono templar (3,286) + a templar | a gravelord + a chrono templar | the whole Lance at any power |

- **The late game asks "one LEGEND capstone, or three WARLORDs?"** That is a real, visible choice.
- **Stats get a COMMAND price.** `unitPower` reads only health, damage and their modifiers. VIGOR, MIGHT and HASTE cost COMMAND; SWIFT, REACH and WILL are free. **A WILL/REACH tank is cheap to field.** The unit sheet says so.
- **An over-COMMAND unit** can still be bound, levelled in garrisons and posted. It just cannot walk with you yet (OSSIAN at hour 15).

#### 7.3.5 Rites, evolution and the lord gate

- **Rites happen only at lamps Hallokin has lit, never in a live fight.** A rite is `HVU.advance(u)` or `HVU.evolve(u, into)`, taken through the ARMY page at any lit lamp, or the RITE card in a Vigil gap (advance only).
- **One redraw per unit per lamp rest.** Each rest offers a unit **either** advance **or** one evolution, never both, and a whole Vigil counts as one rest. This kills the free chain: a knife at T5 L30 would otherwise go knife → verdagger → duskclaw → noctislicer → void reaper in four calls in one frame. It also makes evolution cost tempo. Tier-5 chains (lava slime → flame golem → magma colossus; noctislicer → void reaper) take one evolution per 10 levels past 30.
- **The gate:** evolving into a kind needs its page STUDIED, and into a capstone needs it BESTED (§5.6).
- **Evolution refunds skills, and that is the respec moment.** `evolve` keeps stats, talents, items, level, renown, label and tint. **It wipes every skill rank and mastery**, because no two kinds share move ids, and it nulls the loadout. The page opens GROW on the new body: *new moves, spend your points again.*
- **Evolution replaces the object.** It calls `host.onRemove`, splices, then `spawnFrom`, so the unit gets a new `id`. All game references go through the army record's stable `uid` and the `hvuEvolve` wrapper.
- **Branches are identity decisions** made at T3, where a unit has 9 stat points, 7 skill points and 2 talents: skeleton → bone guard (melee, toward the gravelord) or bone archer (ranged, cheap to field); watch → captain (squad caller, toward the Sentinel) or knight (toward the Chrono Templar or the Mirror Knight).
- **Respec:** spending unspent points is allowed anywhere out of a fight. Refunding stats and skills is free at a lit lamp. Talents cost a Quill. Evolutions are permanent.
- **PRESETS are an AUTO button, not builds.** The five (BULWARK, BLADE DANCER, REAVER, DEADEYE, GRAVE VOICE) sum to 9 stat points, the T5 level-1 budget, so the host scales their ratios to `statBudget(u)`. It never copies the demo's `u.items=[]`, which deletes found gear.
- **No PATHS.** PATHS are dead code in v30 and are cut (§10.5).

#### 7.3.6 Titles, UNIQUE and ascension

Titles are kind labels on big bodies (a bare warchief at T5 L1 is already a WARLORD), so the ARMY page shows the title as a stamp and nothing more. The long tail is **UNIQUE at L40** and **ASCENDED**, which in practice means L50 plus a finished build:
- a gravelord at L40 with 60 renown is 8,116 against 9,166, and ascends at L50;
- a chrono templar and the Blade also ascend at L50.

Three code facts the ARMY page states plainly:
1. **27 kinds can never ascend** at any build. The 1,400 floor causes it: the slime, bat, skeleton, watch, archer, wizard, priest, orc, mesmer, hierophant, pumpkinheads, bone stalker… and **THE KNIFE** (762 at a full L50 build). Their sheets say *"evolves to ascend"*. The Knife's says *"verdagger at T4"*, which is his dark ending. The player chooses between ascension and redemption.
2. **Ascension reads VIGOR, MIGHT and HASTE only.** A gravelord at L50 with WILL/REACH/SWIFT builds (8,993) does not ascend. The sheet says *SWIFT, REACH and WILL do not count toward ascension.*
3. **`ascended` is sticky**, and `fullBuildPower` assumes 48 stat points that no unit can have (33 is the maximum). **Stats lock once a unit ascends.** "The signature is dry" stops the respec-ascend-respec exploit.

#### 7.3.7 Death: scrubbed, or HOLLOWED

- **The save is the build,** refreshed only at safe moments: every lamp bank, and every out-of-combat level-up. A unit in the sim is a copy.
- **Downed:** a unit at 0 health kneels, a host rule intercepting a team-1 death, because v30 has no downed state. A 1.5 s stand beside it (the lamp-relight gesture) brings it back at 30% health with nothing lost.
- **Killed in an ordinary fight: SCRUBBED.** It plays its scrub-out and leaves the Lance for the encounter. At the next lit lamp it is **redrawn from its saved build** with the ink-in. It loses its unbanked XP and **10 renown** (clamped at 0), and never a level.
- **Killed on blank paper, or by a HOLLOWED body: HOLLOWED.** Its Barracks page goes line-only. It reappears later on that sheet as an enemy, `spawnFrom(build, x, z, {team:2})` with the HOLLOWED affix, **carrying your own build, level, talents and artifacts**, and its portrait joins the margin doodles. **SEAL it** (a sigil loop around the stunned body) and it comes home at its build. **Kill it** and it is gone for good. That is the army's one permadeath, and it takes two failures, the second against your own gravelord's kit.
- **In a Vigil:** a dead unit is scrubbed for the night. If the Vigil is lost, that lamp's garrison is scrubbed too.
- **Why not permadeath:** a gravelord is about sixteen hours of play. Losing it to one mistimed slam punishes the army for the hands, and players would bench their best units. **Why not free rebuilds:** a death with no sting makes the army decoration.

#### 7.3.8 The economy

**Gear and every percentage go to units; Hallokin buys verbs.**

| Currency | Earned by | Spent on Hallokin | Spent on units |
|---|---|---|---|
| **Ink** (every kill: base × kind power × `RMUL`, banked at lamps) | kills | technique ranks, town projects, Margin lamps | the 3 weapons and 3 trinkets (KEEN EDGE, HEAVY HAFT, LONG REACH; SWIFT BOOTS, LUCKY COIN, SECOND SKIN) at the Smithy |
| **Quills** (bounty clauses, Crowned executions, first STUDIED stamps) | rare | charm ranks, Vows | talent respecs |
| **Trophies** (never bought) | Crowned elites, lords, Signed duels | Laws, traits | **the 10 artifacts** |

**v1's Inks become artifact trophies.** You still take the affix off the thing that kept healing on you, but you hand it to a unit:

| Crowned affix | Artifact |
|---|---|
| VAMPIRIC | LEECH FANG |
| VOLATILE | EMBER CORE |
| WARDED | WARD STONE |
| ARMORED | IRON ROOT (THORN MAIL from a melee elite) |
| SWIFT | HOURGLASS |
| HEXED | VENOM GLASS / FROST SEAL |

The Warden drops GIANT HEART and the Warchief WAR BANNER. Slots follow `artifactSlots(e)`: one, or two at T5 **or** L20. Items are never consumed and move between units at a lamp, with a confirm before `equip_` silently swaps the oldest item out. **Rites cost nothing but the lamp visit.** Their price is tempo and COMMAND, not a second grind.

### 7.4 Pacing: the real numbers

**The model.**
- About 140 hostile kills per hour of total play, walking included (v1's hour-15 session).
- The player reaches region tier *t* at about 1.67·*t* hours.
- A fielded unit lands about 12% of kills itself.
- Average XP per kill comes from each region's roster: 6.0 at the start, 18.8 at the end.

These numbers come from the army lens loading the module in node.

| Scheme | T2 (L5) | T3 (L12) | T4 (L20) | T5 (L30) + capstone | UNIQUE (L40) | ASCENDED (≈L50) |
|---|---|---|---|---|---|---|
| **Raw v30** (killer-only XP) | 2.2 h | 12.3 h | 25.9 h | **≈ 54 h** | never | never |
| Raw, double the kill rate | 1.4 h | 7.9 h | 17.8 h | 33.3 h | 57.7 h | never |
| THE SHARE, 40% per fielded unit | 0.9 h | 3.9 h | 10.2 h | 18.6 h | 28.6 h | 42.6 h |
| **THE SHARE × `RMUL`** (average rank about B) | **0.8 h** | **3 h** | **8 h** | **16 h** | **24 h** | **≈ 35 h** |
| Late bind at T2 L5, hour 10 (Share, no RMUL) | — | 11.8 h | 15.7 h | 22.2 h | 32.2 h | 46.2 h |
| Late bind at T3 L12, hour 14 | — | — | 17.3 h | 23.6 h | 33.6 h | 47.6 h |

**On raw v30 income, a fielded unit ends a 20-hour game at about L16, tier 3.** No capstone and no UNIQUE would be reachable, and v30's whole top half would be dead content. `XP_TO` is fine; the *income* is wrong. So the author's tables stay untouched, and the host adds one `grantXp` loop in the kill handler. The curve that results:

| Step | When | What it means in the story |
|---|---|---|
| **T2** | first hour | first talent: the unit becomes *yours* |
| **T3** | hour 3 | first branch evolution (bone guard / bone archer, captain / knight, berserker) |
| **T4** | hour 8 | ULT online, so CALL works on that unit; the second evolution |
| **T5 + capstone** | hour 16 | lined up with Chapter III and 10–12 lamps of COMMAND |
| **UNIQUE** | hour 24 | just after the credits: the post-game's first chase |
| **ASCENDED** | about hour 35 | the Endless Vigil's long tail |

## 8. Combat in an open world

### 8.1 Fights stay the size they were tuned for

The combat was tuned for about 20 tiles of view and about a dozen bodies. The open world never asks it to be anything else.

- **The panel.** On engaging, a faint manga panel border is inked around the fight, about one camera frame (roughly 20 × 12 tiles). It *is* the encounter arena: v30's `host.bounds` and every `T.CLAMP` read in the spawn code become this rect. Hostiles return to their post 3 s after leaving it, with a *"tch"* balloon. Reinforcements wait at its gutter. Walk out and it rubs out. Vigils, lords and duels **lock** it with a heavier line.
- **The Lance is leashed to it.** Patch V3's `host.home(e)` gives owned units a home point, and they never chase past the panel line.
- **One attack budget.** The module's `TOKENS=2` (per target) and the game's `attackCap()` merge into one count, so stepping into a battle never means being hit by everyone at once. Smart T4+ enemies focus the wounded, and the budget still guarantees the hero is not dog-piled.
- **The awake cap: about 24 real bodies, hard ceiling 32.** Beyond the panel, a clash resolves as numbers in the ledger. **Army-side summons count as army bodies**, and the host caps them: MASS GRAVE, DEATH WINTER, MIRROR IMAGE, LEGION and the necromancer's rise.

| Occasion | Hallokin + companion | Fielded | Army bodies incl. summons | Hostile bodies | Total |
|---|---|---|---|---|---|
| Open world, skirmish | 2 | **3** (the Lance) | ≤ 6 | ≤ 14 | ≤ 22 |
| Faction clash (side chosen) | 2 | 3 | ≤ 6 | ≤ 10 a side, allied house bodies on yours | ≤ 24 (the ledger resolves the rest) |
| Vigil | 2 | 3 + **2 garrison** | ≤ 8 | ≤ 11 a wave (`composeWave` already caps at `min(11,…)`) | ≤ 23 |
| Lord or story boss | 2 | **2** | ≤ 3 | boss + ≤ 8 adds | ≤ 15 |
| Signed or champion duel | 1 | **0** (they wait at the gutter) | 0 | 1 | 2 |
| Endless Vigil | 2 | 3 + 2 | ≤ 8 | ≤ 13 | ≤ 24 |

### 8.2 The feel gate, extended to the army

**The rule (v1):** hitstop, trauma, camera punch, killcam, manga callouts and damage numbers fire only when the attacker or the victim is Hallokin, the companion, or a body currently targeting Hallokin. **It is now the single most important rule in the plan.** v30's demo host does `hitstop(0.06)` plus trauma on *every* `fx.kill`, unit-on-unit kills included. With a Lance of three, every kill they land would freeze your swing.

| v30 hook | Your hits and kills | The Lance's hits and kills | Hostile units' ULTs and marks |
|---|---|---|---|
| `fx.kill` | Full beat through `onKill`: hitstop, killcam logic, heal, ult gain, drops | Body half only: ring, bits, decal, voice. **No hitstop, no trauma.** | The same body half |
| `fx.number` | Numbers print (`HVU_BY_PLAYER` set around the host's own `HVU.hurt`) | **None.** Wounded bars instead (§8.5). | None, unless the victim is you |
| `fx.shock` | Trauma × `SHK()` | Ring only | Trauma only within 4 tiles of you |
| `fx.mark` 'LEVEL n' | — | One `tickTxt` per 1.2 s across the whole army; the rest fold into a chip pop | — |
| `fx.mark` 'READY TO ADVANCE' + `fx.shock` (fired inside `grantXp`, mid-fight) | — | **Swallowed.** The chip grows a gold ▲, and one `uiToast` per unit per tier. | — |
| `fx.rite` (milestones at L10–50 and ascension also fire mid-fight inside `grantXp`) | — | **In a live fight:** the host undoes `noMoveT` and `poise` (both are written before `fx.rite` is called), stamps the chip, queues the full rite for the next gap or lamp, and applies **no hitstop and no camera.** | A lord's live evolution plays the full rite: the one mid-fight rite beat the shot director allows |
| `fx.ult` | — | Owned units never auto-fire (patch V4). A CALL is the hero's act: a name card, rings, and hitstop only if it hits within 4 tiles of you. Never `beat('ult')` or `panel('ult')`. | Shout at the unit; hitstop and trauma only when involved |
| sound | Full | Attenuated by distance | Attenuated |

**No mid-wave milestone hitstop, ever.** A unit's VETERAN must never freeze *your* swing. The gate is one predicate in `29-hvu-host.js`'s fx members and one in `impact()`.

### 8.3 The camera grammar: one fixed meaning per tool

| Tool | It always means | Used for |
|---|---|---|
| Slow dolly out + pitch down + thin letterbox | *Look how big this is.* | First arrival at a vista. |
| Vertigo dolly in | *That is dangerous.* | Discovering a lord, a vacant crown's owner, or the Hollow Knight. |
| Split-panel cut-in | *Choose.* | A clash begins; then the SIDE CHOSEN stamp. |
| Letterbox slit + eyes cut-in | *Someone is hunting you.* | A Signed ambush, or your own HOLLOWED unit. |
| Page tilt + page turn | *You are going somewhere else.* | A sheet edge, fast travel, a dog-ear. |
| Killcam | *That one mattered.* | A clash leader, a Signed spared or executed, a Vigil's last body, a lord. Never a random kill, and never a kill by the Lance. |
| The domain | *I end this.* | It freezes only hostile teams; the Lance keeps fighting inside. |
| Colour flood / ink-in | *This is yours now.* | A lamp lit, a bite drawn back, a mill retaken, **a figure countersigned**. |
| **The rite / redraw** (new) | ***This is drawn anew.*** | Your unit's tier or evolution at a lamp, and a lord's live evolution mid-fight. The player first sees the beat on the enemy, then earns it for their own. |
| `mono` graphite | *The finale.* | The domain finale and the pause page. |

**The shot director:** at most one big shot per 60 s, with priority lord > rival > clash > vista > killcam. First time full, repeats short. No shot mid-panel except the leader's killcam and a lord's redraw. Rites queued from a fight replay one per 1.6 s in the gap. A setting reduces shots further.

### 8.4 Army orders and CALL

- **One order dial on V** (`makeDial` in `#slot`, beside DASH and ULT), cycling FOLLOW → HOLD → CHARGE:
  - **FOLLOW:** `home` is Hallokin's position. Units re-home when more than 6 tiles away and not engaged, and blink past 14.
  - **HOLD:** `home` is where V was pressed, so a Lance can hold a bridge while you circle.
  - **CHARGE:** `u.hunt=true` for 8 s, then back to FOLLOW. The dial fills while it runs.
- **Neutral arrival** forces HOLD with aggro off until your first damaging blow (§6.6).
- **Lord fights:** at the locked panel line, the two chips you pick stay lit and the third greys and waits at the gutter. **Duels** grey all three.
- **CALL: the Lance's ultimates are Hallokin's to fire.**
  - Patch V4 sets `e.holdUlt` on owned units, so the trigger at units.js 1333 never fires for them.
  - A **CALL pip** fills the first time you reach **A** in a fight, and again at **SS**.
  - **Hold Q for 0.3 s** (tap Q stays your domain) to fire the ULT of the next ready unit in chip order, through `HVU.ULT[kind].run(u, target)`. The ready chip's ring is lit, so the player always knows who answers.
  - Only T4+ units have a ULT (`tierHas 'ult'`). EAGER shortens its cooldown for re-use, and ENCORE or UNIQUE give a unit a second CALL per fight.
  - **One exception:** while Hallokin is on his last stand, held ULTs are released, and the Lance may fire on its own clock. That is the "he dropped off a roof and saved me" moment.
- **Pad bindings** for V and the ARMY page's Q/E page flip need a key audit against 18-state-input's `KEY` and the pad map before phase A4. `pauseNav` must consume `pressed.ult` and `pressed.skill`, or a page flip fires the ULT on resume.

### 8.5 Readability with an army on screen

"Easily seeable" is a budget, and this is how it is spent:
- **Three accents,** the most the eye tracks as individuals. The fourth chip slot exists only in a Vigil, where the two garrison units share one double chip.
- **No text over bodies.** No names, no `T/L/P` numbers (the demo's overhead ink pass is dev-only), no team ellipses. Names and titles show only on the ARMY page or on a unit you lock onto.
- **Ground ticks, not rims:** a short accent tick under owned feet, hidden when the unit is behind decor.
- **Wounded bars on the game's own rule:** a 26 × 4 ink bar for 3 s after a hit, in armour blue for owned units so allies never read as your red health, and always shown under 35%.
- **One callout stream.** Your hits print, and the Lance's do not. Ally ULTs are balloons at the unit, never page stamps. VFX draw in priority order: your ink, then enemy telegraphs, then ally effects at half strength.
- **Telegraphs you can read:** v30's eggs, wedges, capsules and beams are drawn in the game's idiom (`wpath` dashes, a closing arc turning `#ff3a2a` after 0.7, no hatch fill).
- **Capstones that do not twin.** The Sentinel (1.22×) and the Chrono Templar (1.25×) both borrow the `templar` body. Each capstone gets a distinct tint family, its crown when crowned, and one signature ink effect (a clock ring on REWIND, a mirror flash on MIRROR). No new sprites.

### 8.6 Difficulty is behaviour, not sponge

- **THREAT is the enemy's tier, and it comes only from the region.** A T2 skeleton does not use BONE DEEP. A T4 elite focuses the wounded and fires its ULT at nine seconds.
- **WEIGHT is the enemy's level, fixed by region and capped at 20** in the main game (1.59 at most). **Neither dial ever scales to the army or to COMMAND.** No Oblivion-style level scaling.
- **A stronger fielded army buys the encounter tier and affixes, not bodies** (§6.5).
- **Three caps keep the tight verbs meaningful:**
  1. **No single enemy hit exceeds 35% of Hallokin's max health** (lords 50%), enforced in `hvuHitPlayer`, host-side, so a level-20 capstone cannot one-shot the tuned hero.
  2. **Hallokin's damage stays in 10–26,** with growth only through on-screen multipliers under the 3× crit ceiling.
  3. **The decisive verbs are percentages:** execute at 34%, stun by hit count, riposte executes from any health.
- **The honest cost:** a level-20 lord at 1.59× health is long for the hands alone. That is what the two fielded units and a CALL are for, and why a lord fight is where the army proves itself.

## 9. How it gets built

### 9.1 The architecture in plain terms

- **It stays this codebase:** Three.js r128, section files concatenated by `build.sh`, the post pass, the feel package, 21-waves. No engine change, no framework.
- **The units module is vendored, not adopted.** v30 goes in unchanged apart from five named patches, in its own `<script>`, as a byte-diffable artefact. The author calls it final, so every edit is a small fork kept as a readable diff.
- **The game drives it.** One new section file, `29-hvu-host.js`, wires the 14-member host to the game's real functions. Units are simulated by `HVU.tick` and drawn by the game's own sprite rig.
- **The current game becomes a content type.** A Vigil is a Place whose `vigil` block runs today's loop unchanged. Ashridge is stamped from data that reproduces today's town exactly, which makes it the regression test for every phase.
- **A sheet loads whole behind a page turn; only people stream.** `enemies[]` and `HVU.units` hold only what is inside the hot bubble (promote within ~28 tiles, release beyond ~40). Everyone else is a **squad token**, and v30 makes a token literally an array of build objects: `serialize` on release, `spawnFrom` on promote.
- **The world layer is new code** in numbered files (`14w-sheets`, `14n-nav`, `21e-encounter`, `21f-factions`, `29-save`), written multi-line from the start. Trailing `//` comments that swallow code have caused seven live bugs so far, and `audit/swallow-audit.js` runs on every phase.

### 9.2 Vendoring and the five patches

**Where it goes.** `build.sh` globs `1*.js 2*.js` into one IIFE opened in `10-config.js`. The module is a top-level `const HVU=(function(){…})()` that evaluates with no DOM and no THREE (measured in node), so:
- `src/03u-units.html` holds `window.HVU_META` and the sheet blocks (§9.6);
- `src/03v-hvu-module.html` holds units.js lines 60–1638 in its own `<script>`, before `04-script-open.html`. It stays a global beside `HV_SPRITES`, and never silently switches into strict mode;
- the demo sandbox and UI (lines 1639–2132 plus `head.html`) are **never shipped**. The demo reassigns `HVU.host` wholesale and would clobber the game's.
- `build.sh`'s syntax check is extended to every `<script>`.

**The five patches** (the only edits inside the module, each a one- to three-line diff with its own probe):

| id | Where | Edit | Why |
|---|---|---|---|
| **V1** | `tick()`, top of the per-unit loop | `if(e.freezeT>0){e.freezeT-=dt;continue;}` | The game's per-body hit freeze and perfect-dodge freeze have no channel in v30. |
| **V2** | arrow landing (units.js ~1126) | `HVU.host.sfx('arrow',p.owner,p.x,p.z)` | The thunk pans at the arrow, not the archer. |
| **V3** | the wander branch of `update()` | `const h=HVU.host.home&&HVU.host.home(e); if(h){e.wx=h[0]+…; e.wz=h[1]+…}` | Owned units have no follow or leash at all in v30. Orders need a home point. |
| **V4** | the ULT trigger (units.js 1333) | add `&&!e.holdUlt` to the condition | CALL: owned units never fire their own ultimate (§10.3). |
| **V5** | `grantXp` (373) and the kill credit in `die()` (~1027) | test `e.owned` / `a.by.owned` instead of `team===1` | XP counts anyone on team 1: a charmed enemy, and later allied houses. The owned flag is set only by the host's bind and load. |

**PATHS get no patch.** They are cut (§10.5).

**Game-data edits need no patch,** because `CFG` is exported. At boot the host writes:
- `HVU.CFG.warden.name='THE SENTINEL'`;
- `HVU.CFG.pumpkinred.name='THE LANTERN KING'` (and HOLLOW → HUSK on the pumpkin);
- the unlock fixes on the black knight and the werewolf.

**Two more patches arrive only with the world, each behind its own phase gate:**
- **X1 `host.foes()`** (world phase W1). `pickTarget` and `dealHit` iterate only `HVU.units` and the player, so owned units **cannot see the game's own enemies**: slimes, kittens, bombers, casters, and both story bosses. W1 adds a host-supplied list of foreign bodies, with a `_foe` shim giving them `team`, `r`, `hp` and a `hurt` route into `hurtEnemy`. Until X1, Vigils with an army use HVU rosters, and story bosses are fought with no Lance.
- **X2 `hostile(a,b)`** (world phase W0). It replaces `u.team===e.team` style tests (59 team comparisons) with a relations lookup, so houses on ids 3+ can ally, war and be neutral.

### 9.3 The host: `29-hvu-host.js`

**Assign `HVU.host = HVU_HOST` wholesale in `boot()`. Never use `HVU.attach`:** it copies only `puff, shock, number, bits, kill, spinFx` and `mark`, and silently drops `fx.rite` and `fx.ult`.

| Member | Real implementation |
|---|---|
| `player()` | `hvuPlayer()` decorates `P` in place, with no allocation per call: `r=T.RADIUS`, `maxHp=T.HP`, `dir=P.facing`, `stunned=P.lastStand>0`, `velx/velz` from `px/pz`. **`P.team = (P.dead‖travel.phase‖!running) ? -1 : 1`**, written every call. Without it, `playerT()` makes the hero team 0 and owned units attack him. |
| `playerVel()`, `playerWinding()`, `playerRetreating(e)` | `[P.velx,P.velz]` (not `P.vx/vz`, the knockback channel); `atkDef(P.atk).wind`, `P.charging`, `P.aiming`; the demo's dot product plus `P.dashT` |
| `hitPlayer(h,e)` | `hvuHitPlayer`: guards for `running` and travel; `HVU.inHit`; iframes → `perfectDodge` when dashing; `hurtPlayer(min(h.dmg, 35% cap), …)`; knockback. **`e` is null for the poison tick**, so every `e.` is guarded. |
| `freeSpot`, `bounds`, `obstacles` | `freeSpot` directly; the encounter panel rect (today `T.CLAMP`, not `T.BOUNDS`); `colliders` (a live array, broken crates at `r=-1`) |
| `telegraph(t)`, `schedule(d,fn)` | `M.hvuTele` and `M.hvuTimers`, both stepped on the **sim** clock inside `hvuStep` (v30's default `setTimeout` runs through pause and hitstop) |
| `sfx(name,e)` | The 7-token map: `wind`→`voice('wind')`, `swing`→`swing(false)`, `heavy`→`swing(true)`, `throw`→`loose`, `arrow`→`thok(x,z)`, `rally`→`sting`, `ult`→`slam`. The guide's lookup plays only one of the seven. |
| `onRemove(e)` | release the pooled rig; clear `M.lock`, `P.lockE`, `KC.e`, `P.riposte`; splice from `enemies`; drop its telegraphs. Called for projectiles too. |
| `home(e)` | FOLLOW, HOLD or CHARGE point for owned units (V3) |
| `fx.*` | The gated table of §8.2: `puff` via the real 10-argument `puff`; `shock` via `ringFx`, `shockDecor` and trauma × `SHK()`; `number` via `dmgNum`; `bits` via `bit`; `kill` → `hvuKill` (body half always, reward half only when `!a.by`); `rite` and `ult` in the game's grammar |

**`hvuStep(sdt)`** runs in `frame()` right after the `enemySim` loop, with **`sdt`** (the hitstop-held step), never a constant `1/120`. It steps timers and telegraphs, decays `P.markedT` (v30 never ends DEATH MARK on the player), writes `u.slowT` on hostile units inside the domain, then calls `HVU.tick(sdt)`.

**The hero hits units through the game's pipeline.** The blade loop in `20-player.js` also iterates `HVU.units` with `u.team!==1`, calling `impact()`, so crits, style, hitstop scaling, camPunch and sparks all apply. `hurtEnemy` dispatches `e.hvu` to `hvuHurt`, which sets `HVU_BY_PLAYER` around `HVU.hurt(…,by:null)` and writes `freezeT` (V1). **`impact()`'s lethal prediction is skipped for units:** v30 can cancel a lethal blow through WARD, GUARD, AEGIS, SECOND WIND, IRON WILL, BONE DEEP and UNIQUE rise. So `onKill` is extracted verbatim and runs from `fx.kill`.

**Spawning through two wrappers only:**
- `spawnFoe(build,x,z)` = `spawnFrom(build,x,z,{team:2,awake:true})`, then `u.hvu=true`;
- `spawnOwned(record,x,z)` = `spawnFrom(record.build,x,z,{team:1})`, then `u.owned=true; u.holdUlt=true`.

A lint in `build.sh` fails any other call to `HVU.spawn` or `spawnFrom` outside `29-hvu-host.js`.

### 9.4 The traps

Each trap below was confirmed in code by more than one lens. The rewrite names them so nobody meets them live.

| # | The trap | What goes wrong | The fix |
|---|---|---|---|
| 1 | **`spawn()` defaults to team 1** (`team:(opts&&opts.team)‖1`) and tier 5; the guide's step 6 spawns enemies with `{awake:true}` | Every enemy spawned that way is on the hero's side, at full kit, earning XP and immune to his swing | Only `spawnFoe` with `team:2`, plus the build lint |
| 2 | **The game's `P` has no `team`** | `playerT()` writes team 0, so owned units target the hero and their swings land | `hvuPlayer()` writes `P.team=1` or −1 every call; probe A0 includes a negative control |
| 3 | **XP counts anyone on team 1** | Allied houses level; a hypnotised enemy earns XP; your charmed unit feeds XP when killed | Patch V5, the `owned` flag |
| 4 | **The hero's kills give no XP** (`by:null`) | On raw income the army never reaches its top half | THE SHARE through `grantXp` |
| 5 | **Owned units cannot see the game's own enemies** | The Lance is blind to slimes and to the Warden and the Hollow Knight, which blocks Vigils and story bosses with an army | HVU-only Vigil rosters until patch X1 `host.foes()` |
| 6 | **`attach()` drops `fx.rite` and `fx.ult`** | Rites and ultimates become silent no-ops | Assign `HVU.host` wholesale |
| 7 | **The guide names six game functions that do not exist:** `hurtPlayerFromUnit`, `decorCircles`, `M.telegraphs`, `burstBits`, `onEnemyKilled`, `M.timers` | Copy-pasting the guide throws at the first hit | The §9.3 table's real implementations |
| 8 | **The guide's `puff` and `shock` calls have the wrong signature or colour, and its `sfx` lookup plays one token in seven** | Smoke at the wrong height, the camera setting ignored, a silent army | §9.3 |
| 9 | **`render.install` and `render.load(startGame)`** | Outlines (`olm`), team tints, fx inked black, no line, wrong sort order, no boil, no fog; and `startGame` fires as the textures decode, **skipping the title** | Never call them. Draw from `drawInfo()` (§9.5). |
| 10 | **Four sheet keys collide:** `knightIdle`, `knightWalk`, `knightDeath`, `archerIdle` | The Tiny RPG knight replaces the hero | Every unit sheet enters the game's tables as `'u.'+key` |
| 11 | **PATHS are dead code:** `setPath` has no caller and is not exported, and `path` is not saved | A design built on paths silently does nothing | Cut (§10.5) |
| 12 | **`evolve` wipes skills, nulls the loadout, replaces the object and can chain** | Lost ranks, dangling lock-ons, knife → void reaper in one frame | Respec framing; the uid map and `hvuEvolve`; one redraw per rest |
| 13 | **27 kinds can never ascend;** `fullBuildPower` assumes 48 points; `ascended` is sticky | Broken promises on the sheet; a respec exploit | Sheet notes; stats lock on ascension |
| 14 | **Rites fire mid-fight** inside `grantXp` (milestones, ascension): `noMoveT` 1.3 s and `poise` 0 | Your unit freezes in the middle of a swing exchange | The host undoes both in `fx.rite` during a live fight and queues the beat |
| 15 | **Ultimates are timer-based:** `ult.at` forced to ≥ 0.7, `after` 9 s, or crowded | Health-threshold beats (the Knife at 50%, the Blade at 60%) never happen; owned units fire on their own | Scripts call `ULT[kind].run`; rivals read as clocks; V4 `holdUlt` for owned units |
| 16 | **The tier gates the kit:** passive at T3, ULT and smart AI at T4 | A T1 bind has one move; CALL needs T4; low-tier enemies have no passive | Designed in: tiers as stages (§4.2) |
| 17 | **`spawn` ignores `level`**; XP per kill ignores the victim's tier and level | Region scaling that doesn't scale | `spawnFrom` for every body; income scaled by kind choice and THE SHARE |
| 18 | **The demo host hitstops on every kill** | Hitstop soup with an army | The extended gate (§8.2) |
| 19 | **`impact()` predicts death before the blow lands** | Kills counted for a skeleton that rises | `onKill` from `fx.kill` for units |
| 20 | **`spawnFrom` does not slot-validate items; a tier above its level loads; an unknown kind throws** | Corrupt saves crash the load | The loader clamps the tier, equips through `HVU.equip`, and marks unknown kinds as lost |
| 21 | **The demo UI is cream paper; the preset button deletes gear; overhead text over every unit** | The rejected look returns; lost items; mud | Rebuilt UI (§9.8); presets scaled, never clearing items |
| 22 | **Stale comments:** "team 0", "tiers at 3, 5, 7, 10", "stats capped at 5", "Twenty seven enemies" | Wrong numbers designed in | Code wins: team 1, `TIER_AT_LEVEL {5:2,12:3,20:4,30:5}`, `STAT_CAP=8`, 88 CFG entries |
| 23 | **A unit on team −1 is still targeted** (the −1 skip exists only for the player) | Your army starts every war | Neutral HOLD with aggro off (§6.6) |
| 24 | **The kind id `warden` exists twice:** `HVU.CFG.warden` and the game's slime boss | Roster and save collisions | Namespace kind ids `hvu:warden` / `hv:warden` in every store |

### 9.5 Rendering through the game's own sprites

Units draw through the game's rig from `HVU.drawInfo(e)` and `HVU.projInfo(p)`. The module's `render` object stays inert.

**Step R1 (phase A0):**
- `SHEET_ANCHOR['u.'+k]` gains `{w,h,cx,feet}` in tiles from the same `ppu` expression as `drawInfo`;
- `show(idx)` takes a **per-frame** `cx` from `pxs` (109 sheets carry per-frame pivots);
- `unitRender(rdt)`, after the enemy loop in `24-render.js`, plays `d.sheet` and commits `d.frame` **only on `ANIM.step`** (HVU owns the frame for hitboxes; the game owns when it is drawn). It then flips, sets z, applies `occlusionTint` x-ray, the hit flash and the red line on hit, and `commitPose`;
- it **ignores `d.tint`'s team factor and `d.custom` / `d.accent`**;
- fx layers draw in the art's own colour, one lazy extra sprite per rig; drawn shadow strips are dropped for the page's hard ink ellipse; projectiles use a 24-rig pool placed through `fxOrder`;
- `uFast` runs the animation clock at 24 Hz while any unit is attacking, dashing, leaping or volleying.

**Step R2 (phase A2): v30's one great idea, ported.** One GPU texture per sheet, never cloned. `inkLine` gains a per-material `uRect` uniform that replaces `uvTransform` in the vertex patch and the frame clamp, so body (mode 0) and line (mode 1) sample the same texture. All 478 sheets resident come to 4.80 Mpx × 4 B = **19.2 MB VRAM**, against about 51 MB with churn for per-rig clones. This is v1's old phase-0 step 1, and it retires PERF-1's MRU for units. SKETCH OFF bakes its legacy stacks lazily, for unit sheets only.

### 9.6 The asset budget and lazy loading

| Piece | Raw | Gzip |
|---|---|---|
| The game today | 927 KB | 395 KB |
| HVU module | 203 KB | 60 KB |
| `HVU_META` | 3 KB | 1 KB |
| `HVU_SHEETS`, 478 sheets (base64 PNG does not compress) | 1,334 KB | 881 KB |
| Three.js r128, as inlined in v30 (the same revision the game loads) | 603 KB | 149 KB |
| Demo sandbox and UI | 93 KB | not shipped |

**Decisions.**
1. **The game stays one HTML file.** The user plays a downloaded file, and nothing in v30 needs a server. A multi-file build with a service worker stays a later option, with `--inline` kept.
2. **Inline Three.js r128 and drop the CDN.** +603 KB raw (+149 KB gzip) buys offline play and removes the only "load failed" failure mode. `03t-three.html` replaces the cdnjs line; `build.sh --cdn` keeps a light build. Hash-check the bytes against stock r128.
3. **Sheets ship as one JSON block per group, decoded on demand.** `<script type="application/json" id="hvu-fam-<group>">` blocks are tokenised as text, not parsed. `HVU_SHEETS` always carries each sheet's metadata (`n, fw, fh, px, py, pxs, fps, loop`), because the sim reads frames and hitbox shapes before any art exists. Only `d` is lazy.
   - `hvuNeedFamily(group)` decodes up to 24 images in flight; `hvuFamilyReady(group)` is the gate.
   - Callers: `boot()` for the saved army's families; a page turn for the next sheet's families; the Vigil gap for wave *n*+1 (deterministic under `wseed`); the EVOLVE tab for target portraits.
   - **Never decode on a spawn frame or during hitstop.**
4. **Prune at build time.** Unreferenced sheets (the 6 shadow strips, 6 `slime_old` sheets) are dropped, and `--families=` builds exclude groups.

| Build | Contents | Raw | Gzip |
|---|---|---|---|
| **Phase A0** | game + module + META + the watch family only; three still on the CDN (one variable at a time) | **≈ 1,150 KB** | ≈ 470 KB |
| Phase A2 | + three inline + watch, undead, orcs | ≈ 2,010 KB | ≈ 800 KB |
| **Every family** | + all groups, pruned | **≈ 3,050 KB** | **≈ 1,470 KB** |
| **Alarm** | a single-file build past this fails `build.sh` | **3,500 KB** | 1,700 KB |

On a real integrated-GPU laptop (never the SwiftShader harness, whose timings are meaningless): the title's first frame arrives no later than today's, and one family decodes in under 150 ms, spread across frames on the title and in gaps.

### 9.7 Saves

- **Each unit is a record wrapping its build object:**
  ```
  { uid, build: HVU.serialize(u), order, state: 'ready'|'kneeling'|'scrubbed'|'hollowed'|'posted',
    post: sheetId|null, crown: kind|null, origin, since, bankedAt }
  ```
  `uid` is stable across `evolve`. A fresh watch serializes to 157 bytes, and a record is about 250 B, so a 30-unit army is about 8 KB.
- **The document:** one versioned JSON `{v, hero{…}, army:[record…], roster{…}, world{…}, rivals:[build…]}`. It lives in `localStorage['hv_army']` during phases A0–A5 (beside `hv_best` and settings, with the same try/catch discipline), then moves to IndexedDB `hv_folio` in the world phases, migrating `hv_army` once.
- **Written only at safe moments:** a lamp bank; the Vigil gap's rising edge; after a RITE or an EVOLVE commit; on closing the ARMY page with changes; on a death's results card; and on `pagehide` **only if** no fight is live. A save during a live fight is refused, not deferred.
- **Load:** tiers clamped to the level; items equipped through `HVU.equip` (slot-validated); `tierReady` re-derived from `readyTier`; an unknown kind marks the record lost and never throws; owned units spawn beside Hallokin with `awake:false`. `resetRun` removes hostile units and keeps team 1. `HVU.clear()` is never called on owned units.
- **Not saved, by design:** HP, poison, charm, cooldowns and ult-used. Units come back whole.

### 9.8 The army UI

**The rule: combat shows state, never editing. Editing happens only when no fight is live.**

**In combat:**
- **The army strip** (`#army`), under the three HUD stat rows inside the top-left column (`HUD_RECTS` gains an `'army'` rect so callouts place away from it). Up to **four portrait chips**: the three Lance units, plus in a Vigil one double chip for the garrison pair. Each chip is:
  - a 46 px full-colour portrait;
  - an `inkCircle` ring in the unit's `cfg.accent` (gold at UNIQUE and ASCENDED, lit when its CALL is next);
  - a `makeBar` HP bar in armour blue;
  - the level in Bangers.

  A kneeling unit's chip drops to 40% with a struck line. A tier earned shows as a gold ▲. `body.cine` hides the strip with the HUD.
- **The order dial on V,** 3-second wounded bars, the foot tick, and nothing else.
- **No mid-wave milestone hitstop.**

**In the gap:** the RITE card on key 4 (advance one ready unit; §10.4).

**The pause page's ARMY tab** (Q / E page flip; LB / RB on pad): chips down the left, the selected unit on the right, in four tabs.
- **OVERVIEW:** portrait, name, kind, tier and level, HP and XP bars, POWER with the title stamp, moves with glyphs and `HVU.describe`, passive and ult sentences, "next tier brings…".
- **GROW:** stats as 8-pip rows, skill ranks and the rank-3 MASTERY choice, a quick-build row.
- **TALENT & GEAR:** `TALENTS[tier]` cards; artifact, weapon and trinket slots through `HVU.equip`, with a confirm before a swap.
- **EVOLVE:** one card per `evolutionsFor(kind)`, with the target's silhouette, portrait or "?" by Roster stamp, its power against the unit's, the required tier, and a hold-J EVOLVE (at a lit lamp only).

**Editable only when no fight is live.** Otherwise the page is read-only. It rebuilds on a change of the demo's `sheetKey`, never on a 200 ms `innerHTML` timer that would fight the row cursor. It also shows the COMMAND meter (fielded power against COMMAND), the owned cap, and each unit's post.

**Developer-only:** the Lab (a fifth tab under `?lab` and `__HV.hvu.lab`), the spawn drawer, the dock, the topbar, POWER RANK, `#help`, `#log` and the overhead ink text. **Nothing cream ships.**

### 9.9 The phased roadmap: units into today's game (Track A)

**Rules for every phase:**
- **`build.sh HVU=0|1`.** An `HVU=0` build omits `03u`, `03v` and `29-hvu-host.js`, every game-side hook is written `if(window.HVU){…}`, and it must **build byte-for-byte like today**. `pw-smoke.js`, `pw-reset-probe.js` 19/19 and `pw-set-probe.js` 25/25 pass on both builds.
- Snapshot `src/` to `hv-work/snapshots/src-pre-hvuN.tgz` first.
- The swallow audit runs.
- Performance is judged only on a real laptop.

| Phase | What ships | Probe | Screenshot | Rollback |
|---|---|---|---|---|
| **A0 — One watch in Ashridge** | `03u` holding only the 13 `watch*` sheets; `03v` **unpatched**; `29-hvu-host.js` with all 14 members (telegraphs as plain rings, rite and ult as `tickTxt`); R1 in `13-sprites.js`; `unitRender`; the blade loop over `HVU.units`; the `hurtEnemy` dispatch. The lethal-prediction shortcut is sound only because a T1 watch has no passive, guard or rise, and the probe asserts that. | `pw-hvu0.js`: **one hostile T1 watch is hit and killed** (hp drops, a number prints, `hitstop>0` on contact, `kills` +1 exactly once, the death strip plays on the beat, the rig returns to the pool); **one owned watch reaches level 2** (`XP_TO(1)=12`, 6 XP per watch kill; a `LEVEL 2` tick; +25% hp); over 20 s `P.hp` never drops from the owned unit, with a negative control where `P.team` is undefined; 0 errors; texture census flat over 10 cycles | 2x DPR: the knight mid-swing into the watch; owned and hostile watches by a house; a 1:1 crop knight / slime / unit showing **the same 1.35 px line, no rim, untinted colour**, the hard ink shadow, and x-ray through a roof | `HVU=0` |
| **A1 — The host made correct** | `onKill` extracted (B4); the `_foe` shim for GUARD and `perfectDodge`; the real telegraph drawer (eggs, wedges, capsules, beams); the CFG bridge (`drawH`, voice keys); **V1, V2**; `fx.rite` and `fx.ult` in the game's grammar with the §8.2 gate; skel and wizard added | `pw-hvu1.js`: a skeleton killed by a light blow rises and `kills` waits; a WARD-blocked lethal blow gives no kill; a perfect dodge freezes the unit; a KLANG staggers it; 60 s of a 6-unit brawl throws nothing in `AudioSys`; **`hitstop` stays 0 on unit-on-unit kills** | the telegraph shapes over a brawl | revert A1 commits |
| **A2 — Shared rig, lazy families, three inline** | R2 `uRect`; per-group JSON blocks with `hvuNeedFamily`; `03t-three.html`; pruning | `pw-hvu2.js`: texture count flat across 20 sweeps of 10 families, a second sweep adds none; no decode lands on a hitstop frame; the build runs with the network aborted; size inside §9.6 | the A0 crops **pixel-diffed** against A0 (`ink-compare.py`): the shared rig must not change the look | `HVU_RIG=clone` |
| **A3 — Hostile units in the wave game** | team-2 units mirrored into `enemies[]` with the guard list; `composeWave` entries (`ECOST` from `HVU.power`, tier and level by wave); the domain slow via `slowT`; boss chips for units; the next wave's families requested in the gap; unit rosters **replace** slime slots | `pw-hvu3.js`: the field-compatibility sweep (every ability, the domain, the companion's arrows, `lastStand`, a spit parry) against a unit, diffed against a pure-HVU run, with no NaN and no fault; waves 1–10 on unit rosters; "no unit survives `resetRun` except team 1" | a wave-8 frame with 11 units for mud review | `HVU_WAVES=0` |
| **A4 — The army** | **V3, V4, V5**; `spawnOwned`; THE SHARE; a COMMAND meter; the `#army` strip, V dial, foot tick, wounded bars; the RITE card on key 4; the ARMY page; `hvuEvolve` with uid re-pointing; `hv_army` saves | `pw-hvu4.js`: across 6 waves an owned unit reaches L5 through THE SHARE; the ▲ appears; key 4 advances to T2 and **`hitstop` stays 0 during every live wave**; the ARMY page edits in the gap and refuses mid-wave; evolve keeps uid, label and stats and refunds skills; a charmed enemy earns no XP (V5); an owned T4 unit never fires its ULT until CALLed (V4). `pw-hvu-save.js`: byte-equal `serialize` across a reload; a corrupted record loads clamped without a throw | HUD at 1366×768 and 1920×1080 with 4 chips; the ARMY page; the RITE card, **reviewed with the user for "easily seeable" and colour** | `HVU_ARMY=0` |
| **A5 — The Roster and SPARE in today's game** (v1's phase 0.5) | Roster pages with move-answer hooks; the SPARE and SEAL hold; binding writes a tier-minus-one build; STUDIED and BESTED gate the EVOLVE tab | `pw-hvu5.js`: answering a T1 watch's one move stamps STUDIED; hold-attack spares it; the unit arrives at T1 L1, team 1, owned; an evolve edge into an unstudied kind is refused | the SPARED splash; a Roster page | `HVU_ROSTER=0` |

After A5, today's wave game already contains the whole loop in miniature: spare a body, level it on your kills, take its rite in the gap, study what it could become.

### 9.10 The world phases (Track W, from v1)

| Phase | What ships | The gate | Rollback |
|---|---|---|---|
| **W0 — The feel test, with an army** (deliberately ugly) | One flat 128 × 128 sheet with Ashridge stamped from data; a generic rig pool; **patch X2 `hostile()`** and house ids; one squad token that promotes and demotes through `serialize` / `spawnFrom`; one two-sided clash (Watch vs orcs) with first-blow side picking, the neutral HOLD and the §8.2 gate; a flow field for chasers | Texture count flat; 50 promote/demote cycles with no field leakage; nobody stuck behind a house for more than 2 s; **the user plays into the clash with a Lance of three fielded and says the hits still feel like Hollow Vigil, and the screen never froze on a hit that wasn't theirs** | The linked-arenas fallback: each sheet becomes Places joined by short corridors. Folio, lamps, Vigils, army and Roster are unchanged. |
| **W1 — "The First Sheet"** | Ashridge + the Vigil Vale; roadside lamps and one Great Lamp Vigil; the Knife's opening and the first SPARE; the companion; **patch X1 `host.foes()`**; the Warden in the Well with **the drowned captain's live redraw into THE SENTINEL**; the crown and WARDEN OF ASHRIDGE; garrisons; scrubbed death; the watched circle and lamp holes; card-to-charm; IndexedDB saves. Ships as one HTML file. | Screenshots of blank bites and char-free full colour **reviewed with the user before any second sheet**; the Lance fights the Warden's slimes (X1 probe: owned units target and kill a `T.SLIME` body, and the kill credits them) | `HVU_ARMY=0` world build |
| **W2 — "The Front"** | The Watch, the Warband and the Wild on real relations; standing and banner reads; the War Camp and Long Ride; **THE WARCHIEF's live evolution**; THE LANCER and THE BLADE as growing rivals; Crowned persistence; defection edges moving bodies between visits | After 30 minutes and two exits, the front has visibly moved for reasons the player can name | revert W2 |
| **W3 — "The Folio"** | The remaining sheets and houses: the Order, the Redrafters, **the Kindled and the Scorch** (char holes on the budget); the Dusk and the Margin; HOLLOWED owned units and SEAL; every family lazy by sheet; save migrations | Every one of the 11 lords' redraws probed (`evolve` returns a body, the boss bar re-points, BESTED stamps); build under the 3.5 MB alarm | revert W3 per region |
| **W4 — The story and the finish** | Chapters III–IV with the triad resolved by posted power; the Hollow Knight's lair on the back of the page; Mirror Knight duels; the Gutter and the Reference; the Endless Vigil; the UNIQUE and ASCENDED tail; the Chronicle | A full playthrough on the pacing table's marks: T2 by hour 1, T5 near hour 16 | revert W4 |

### 9.11 The hardest problems

1. **Navigation does not exist.** A flow field serves chasers. The Lance gets the V3 home point plus a blink leash, and accepts giving up a target. Places stay open-plan.
2. **Object identity.** `evolve`, mirror images, slimelets and grave-raised bodies all allocate mid-tick. Every game reference goes through `onRemove` or the uid map. A lock-on to a lord that redraws mid-lock is the case to watch.
3. **State leakage on reused bodies.** About 60 per-life fields (`ultUsed`, `charmT`, `team`, `tintOverride`, a `cfg` the berserker mutates). The field-snapshot test is mandatory from W0.
4. **The feel package is global.** Hitstop scales the whole step, and the killcam and letterbox are single channels. The gate handles other people's fights, but two player-involved beats in one big clash still compete.
5. **Numeric creep.** Levels × stats × gear × ascension. Contained by region-fixed enemy levels, the 35% hit cap, COMMAND, and budget spent on tier rather than bodies.
6. **Menu load.** Four tabs × N units × 50 levels, against the checklist's clean HUD. Only ▲ and a CALL pip ever ask for attention in play. PRESETS scale for players who never open GROW.
7. **Keeping the erasure and the burn clean:** blank bites and char holes must read as crisp absence at every CLARITY setting, proven on screenshots with the user.
8. **Content volume.** Twelve sheets, 85 Roster pages, eleven lords, six champions. Seeded scatter fills between Places. The cut list (§11.1) is ordered.

## 10. Decisions

### 10.1 Regions and sheets

**Decision: ten regions on twelve sheets.**
- The regions: Ashridge, the Lamplit Rows, Vellum Abbey, the Tracing Tower, the Palimpsest, the Brushlands (two sheets), the Hatchwood, the Spill, the Scorch (two sheets) and the Margin.
- **Every lineage has a home** (§5.3).
- **Every region has a capstone lord**, eleven in all (the Scorch has two).
- **The other six capstones are champions** with a place, so all 17 capstone edges can be opened.

The roster lens's count did add up if the Scorch counts as one region, but its table also left three regions without a capstone lord: the Rows (the Lantern King is tier 4), the Spill (the Three Kings are tier 4) and Ashridge (the Warden is a game enemy). v2 closes all three from capstones that had no lord role:
- **the Rows get THE SIREN**, the Kindled's Choir pushing north and singing lamplighters off their posts;
- **the Spill gets THE PLAGUE DOCTOR**, the heretic physician, with the Three Kings as its Vigil's boss wave;
- **Ashridge gets THE SENTINEL**, as the Warden's fourth beat.

The Palimpsest drops to one sheet (the Old Watch Barracks becomes a Place inside it), and so does the Margin (plus the back of the page). That pays for the Scorch's two.

*Rejected:*
- v1's 8 regions on 12 sheets, which leave the Kindled homeless;
- the roster lens's table as written, with lordless regions breaking "every lord is a future";
- the Siren at Cinder Gate, which would give one sheet three capstone fights and the Rows none;
- a thirteenth sheet for the Scorch, more authored content than the density rules can fill.

### 10.2 Army size

**Decision:**
- **three fielded** in the open world and in clashes (the Lance);
- **three plus two garrison** in a Vigil;
- **two** in lord and story-boss fights;
- **none** in duels;
- the companion outside all of it.

**The strip holds up to four chips:** three for the Lance, and in a Vigil one double chip for the garrison pair.

The body budget decides it. At three fielded plus summons capped at six army bodies, every occasion stays at or under the ~24 awake cap (§8.1). Three accents are the most the eye tracks as individuals, which is what "easily seeable" means in a crowd. Two in a boss fight keeps telegraphs readable while CALL still matters. None in a duel keeps the rival's promise that you beat him with your hands. The integration lens's four chips survive as UI capacity, not as a fourth fielded body.

*Rejected:*
- four fielded (integration, critic), which pushes a clash past the awake cap with summons and makes the fourth accent unreadable;
- v1's companion plus one follower, which leaves half of `TALENTS` and `ITEMS` dead;
- an uncapped army with COMMAND as the only limit, since power is not the only budget: bodies are.

### 10.3 Who fires a unit's ultimate

**Decision: Hallokin does.** Owned units carry `holdUlt` (patch V4), and their ultimates fire only as a **CALL**: a style pip at A and SS, then hold Q. The one exception is Hallokin's last stand, when held ultimates are released. **Hostile units keep v30's timer.**

v30's trigger fires under 70% health, after about nine seconds engaged, or when crowded. For an army that means three ultimates going off on their own clock in every fight, each one a hitstop and shout candidate, burying the hero's own domain beat. CALL turns style rank into the button that summons your warchief's RAMPAGE, gives the ULT a natural unlock (T4) and upgrades (EAGER, ENCORE, UNIQUE), and costs one condition in the module. For enemies, the timer is a feature: a rival's ultimate becomes a *clock* the player learns to count.

*Rejected:*
- v30's timer for owned units (the army plays its biggest moments without you);
- a host-side cancel after the trigger fires, which is racy: `ultUsed` and the rite have already run;
- cutting unit ultimates from the army, which throws away v30's best writing.

### 10.4 Where rites happen

**Decision: both occasions exist, under one rule. A rite is taken only at a lamp Hallokin has lit, only when no fight is live, and a unit gets one redraw per lamp rest.**
- **At any lit lamp,** the ARMY page offers advance **or** evolve.
- **In a Vigil's wave gap,** which is by definition at a lit Great Lamp between live waves, a **fourth card, RITE, on key 4** offers **advance only**, one unit per gap. It uses that unit's redraw for the whole Vigil.
- **Evolution waits for the next lamp rest,** because it wipes skills and needs a respec screen, and a respec does not belong in a wave clock.

The army lens's lamp rite and the integration lens's gap card are the same ceremony at the same place. The gap is simply the most common moment a tier is ready, and the card keeps it one keypress. The one-redraw rule is what stops v30's free chain-evolve.

*Rejected:*
- lamp rites only (it wastes the moment the ▲ is most often earned, and forces a menu trip after every Vigil);
- evolving in the gap (a skill wipe and respec mid-Vigil is a menu in a fight);
- rites anywhere out of combat (the lamp stops meaning anything).

### 10.5 PATHS

**Decision: cut.** PATHS are dead code in v30: `setPath` has no caller and is not exported, `path` is not in `serialize` or `spawnFrom`, and every PRESET has `path:null`. Reviving them costs a three-edit module patch, a save schema field and a migration. It adds the least legible layer of a unit's identity, and CONJURER would add up to four raised bodies against a body cap that is already the tightest budget in the plan. Unit identity is carried by kind, branch evolution, talents, masteries and gear, all of which work today.

*Rejected:* the three-edit patch (V5's slot goes to the owned flag, which fixes a real exploit). If playtests later show T2 lacks a decision, PATHS return as a schema bump defaulting to `null`.

### 10.6 Cuts and keeps

**Decision: confirm the army lens's cuts and keeps, with four adjustments.**

| System | Verdict | Reason |
|---|---|---|
| **Nibs** | **Cut** | Hallokin gets no gear, so his hits never change. |
| **Bond levels** | **Cut** | Levels, milestones and titles are the bond. **Adjustment:** Oathbound's "wear its passive" becomes the lent fourth trait slot from a fielded **T5** unit. The critic proposed UNIQUE at L40, but that lands at hour 24, after the credits, and T5 lands at hour 16, inside the story. |
| **Inks** | **Become artifact trophies** from Crowned affixes | Same fantasy, one loot table. |
| **The Roster** | **Kept, extended** | STUDIED gates sparing and ordinary evolution; BESTED gates capstones. **Adjustment:** for a capstone, BESTED is stamped by winning its lord or champion fight, not by three executions (lords are fought once). |
| **Charms and card-to-charm** | **Kept unchanged** | Hero-side, run-scoped cards; units never touch them. |
| **Laws** | **Kept**, from executed lords | **Adjustment:** a lord allowed to keep its lamp gives no Law, which makes the choice cost something. |
| **Techniques and traits** | **Kept on Hallokin** | Ranks I–III with no hero MASTERY. **Adjustment:** SCRIBBLE DOUBLE's teacher moves to THE MIRROR KNIGHT. |
| **SEAL** | **Kept** | The gesture is hold-attack for armatures at T3 or below, and a sigil loop for T4+ armatures, the Hollowed and the Hollow Knight. The first-hour skeleton needs no ability. |

*Rejected:*
- the critic's "signed units cannot hollow" (a label is free, so every unit would be signed and the Hollowed would vanish);
- the critic's rites costing ink or a quill (the price is tempo and COMMAND, not a second grind);
- cutting the hero's traits and charms (the hands would have no build left to make).

### 10.7 v1 decisions that changed

| v1 decision | v2 resolution |
|---|---|
| **9.1 Unlit land: full colour or blank paper, no middle state** | **Holds, extended** to the army UI (no cream), to units (no `olm` outline, no team tints, no charm wash) and to the Scorch (char holes are dark and on the same quarter-frame budget). |
| **9.2 The Folio, 12 sheets of 128²** | **Holds.** Twelve sheets now carry ten regions (§10.1). The linked-arenas fallback stands, tested with a Lance fielded. |
| **9.3 One progression model, no levels** | **Replaced:** the hero knows, the army grows. No levels for Hallokin; v30's full ladder for units, bridged by THE SHARE, COMMAND and lamp rites. The Tally feeds COMMAND and is never called a level. |
| **9.4 The Vigil's one role is holding land** | **Holds, with a second role:** the army trains there, garrisons hold there, and the RITE card lives in its gap. |
| **9.5 The two Wardens** | **Changed.** The unit keyed `warden` is renamed THE SENTINEL and moves from the Order to **the Watch** (watch → captain → sentinel). "Warden" becomes a **post** held by crowned units at Great Lamps. The slime boss keeps THE WARDEN and every shipped string. Your first crown is **WARDEN OF ASHRIDGE**. |
| **Protagonist "the last Watchman"** | **Named:** HALLOKIN, THE LAST WATCHMAN. The signature is handed down, the Hollow Knight held it before you, and recruiting countersigns a unit into his kin. |
| **Five political factions + the Wild + the Hollow** | **Six houses** (the Watch, the Gilt Order, the Underdrawn, the Redrafters, the Warband, **the Kindled**) plus **the Wild** (the Blot, the Hatch, the Lampless) and **the Dusk**. The Hollow stays an affix. |
| **Membership by accent colour** | **By lineage.** Accents still read at a glance, but they no longer define membership. |
| **Six custom-body region lords** | **Eleven capstone lords** that evolve live, plus six champions. The v9 customs are lieutenants and mid-tier units the player can own. |
| **"Lords are never spared"** | **Lords never join.** Beaten, a lord is executed for its Law or allowed to keep its lamp. Either way you earn its *form*, never its body. |
| **Two story bosses outside the unit system** | **Holds, and matters more:** neither can be owned or evolved into. The Lance fights them only after patch X1. |
| **The Hollow King vs the Hollow Knight** | `pumpkinred` is renamed **THE LANTERN KING**, so "Hollow" means only erasure. |
| **The player on team 0, −1 for neutral** | **The player is team 1** with owned units, hostile spawns are team 2, and houses take 3+. Neutral is `P.team=-1` for the hero plus a HOLD for the Lance. |
| **Rivals flee at their ULT's health threshold** | **Rivals' ultimates are clocks** (about 9 s). Scripted beats call `ULT[kind].run` directly. Rivals *grow* between meetings through real talents and masteries. |
| **Chapter III: the Varnish and the Trace** | **A triad:** the Varnish, the Trace and the Burn. You break one in person, and your posted power holds the other two. |
| **Chapter IV: "you are the Watchman now, alone"** | **The Watch you rebuilt stands at the lamps, and she is not among them.** |
| **Followers add their threat to ECOST** | **`0.4 × fielded unitPower / 100`**, spent on tier and affixes, never bodies. |
| **"Three structural edits to the units module"** | **Five vendoring patches** (V1–V5) for the army, **two world patches** (X1 `host.foes`, X2 `hostile`), and game-data edits through the exported CFG. |
| **Death: an ink pool; Crowned elites take your ink** | **Holds for the hero.** Units are scrubbed (unbanked XP, 10 renown) or HOLLOWED (return as enemies with your build; seal or lose them). |
| **The companion outside the team system** | **Holds:** outside the army, COMMAND and XP. No owned archer may read as her. |
| **File format: multi-file once a second region arrives** | **One HTML file** with Three.js inlined and lazy JSON families. Multi-file stays a later option. |

## 11. Cut list and risks

### 11.1 What to cut first if scope bites

In order. Everything on this list leaves the spine intact.

1. **Back-of-page secrets beyond the Hollow Knight's lair.** The Mirror Knight is fought once, at the lair's door.
2. **Crowd extras, moving fronts and defection migrations between visits.** Territory changes only when the player acts, and the ledger shrinks to one owner per Place.
3. **The Kiln folds into Cinder Gate** (11 sheets). The Magma Colossus becomes the Scorch's second lord on one sheet.
4. **The Three Kings become one king,** and the Lantern King becomes a Rows street elite.
5. **Champions without a story role become bounty-board elites:** the Demon Brute and the Werebear. Their capstone edges open from a Crowned kill.
6. **The Tracing Tower as a region.** THE LICH and THE MESMER visit other sheets, and the Trace's rite moves to the Palimpsest.
7. **Garrisons resolved off-frame.** Garrisons fight only in Vigils you attend, and Chapter III's held rites read posted Wardens alone.
8. **In-sheet streaming,** which means taking the linked-arenas fallback on purpose.
9. **Ascension's gold rim and the lent trait slot.**

**Never cut:**
- full colour by default, with no cream, no outlines and no team tints;
- lamps drawing the land, and rites only at lit lamps;
- SPARE and SEAL;
- **every lord is a future:** the STUDIED/BESTED gate, capstones grown and never bound, and live lord redraws;
- **Hallokin never levels,** with THE SHARE and COMMAND;
- first-blow side picking with the Lance's neutral hold;
- the Vigil;
- the Roster;
- **the feel gate extended to the army's kills, level marks, rites and ultimates;**
- `team:2` on every hostile spawn and the `owned` flag;
- the camera grammar;
- the Kindled as a house;
- the `HVU=0` byte-identical rollback.

### 11.2 The biggest risks to fun

| Risk | What it looks like | Mitigation |
|---|---|---|
| **The world goes white, or cream creeps back** | Erasure creeps until play is spent on paper; the demo's `#ece5d4` panels are lifted as they are. | Blank paper and char holes at most a quarter of a combat frame outside the Margin; the watched circle; the ARMY UI rebuilt from the game's plates; screenshot review with the user in A4 and W1 before anything else is built on them. |
| **Outlines return** | v30's accent silhouette on every advanced unit; HOLLOWED drawn as a halo. | `unitRender` never reads `drawInfo().custom`; rank lives on the chip, crown and foot tick; the A0 1:1 crop is the standing test. |
| **Army hitstop soup** | Three units' kills freeze your swing; LEVEL and READY TO ADVANCE overprint the fight; a VETERAN rite stops a unit mid-exchange. | The §8.2 gate over `fx.kill`, `number`, `shock`, `mark`, `rite` and `ult`; the host undoes mid-fight `noMoveT`; level ticks throttled; rites queued. The A1 and A4 probes assert `hitstop` stays 0 on unit kills and during live waves. |
| **The hero becomes a spectator** | The army kills and Hallokin watches. | Hallokin's kills fuel THE SHARE × style rank; ultimates are CALLs; COMMAND caps fielded power; the Lance stays at three; the merged attack budget still focuses enemies on the hero; duels field nothing. |
| **Number creep** | A crowned, ascended army trivialises Vigils, and enemies turn to sponges to compensate. | Enemy level region-fixed at ≤ 20; the 35% hit cap; a stronger army buys tier and affixes, not health; COMMAND. |
| **Evolution becomes a checklist** | Chain-evolve to the top in a frame; every branch taken by rote. | One redraw per lamp rest; the lord gate; COMMAND prices on stronger branches; defection edges with political consequences; the Knife's ascension-versus-redemption choice. |
| **Losing a sixteen-hour unit** | One mistimed slam deletes a Gravelord, and players bench their best. | Kneel and revive; scrubbed rather than erased; permadeath only through HOLLOWED, and only after a failed SEAL. |
| **Menu game** | Four tabs × 30 units × 50 levels. | Only ▲ and a CALL pip ask for attention in play; the RITE card is one key; presets scaled to budget; editing only when no fight is live. |
| **The combat doesn't survive open space** | Fights sprawl and the Lance chases off-frame. | W0 decided by the user's hands, with a Lance fielded; the panel leash and V3 home points; the linked-arenas fallback. |
| **Walking is dead time** | Empty green between fights. | Sheets sized to 60–90 s; *look every 30 s, do every 90 s, a story every 10 min*; the momentum string; margin doodles including crowns and your own Hollowed. |
| **Samey fights** | The fortieth watch squad. | 87 kinds × tier as behaviour × three-way clashes × modifiers × affixes × which side you chose × rivals who grow × lords who redraw mid-fight. |
| **The Roster feels like homework** | Checklists of moves to parry. | T1 bodies study in one answer; stamps come from what feels best (the KIIN, the execution); the evolution tree makes each page *want*-driven: you study the Gravelord because your skeleton needs it. |
| **Asset weight** | A slow first load; one family per spawn hitch. | Lazy JSON families; never decode on a spawn or hitstop frame; the 3.5 MB alarm fails the build. |
| **The traps bite anyway** | A contributor copies the guide. | §9.4 lives in the repo beside `29-hvu-host.js`; the `spawn` lint; A0's negative control for `P.team`. |
| **The faction war isn't legible** | Territory changes and the player doesn't know why. | The map spread shows who took what; every change traces to a clash, a lamp, a lord's ending or a defection edge; banners read your Lance. |

---

*The short version: the world is a drawing being erased and lamps draw it back in full colour; Hallokin never levels, but every enemy he understands can be spared onto his side and raised, on his kills and at his lamps, into the very lord he had to beat to earn that form; the fights stay small, tight and his; and the wave game is where the land is held and the army is trained.*
