# LENS: THE WORLD AND ITS FICTION — "Hollow Vigil" as an open-world RPG

Concept pass, 2026-09-13. No code touched. Grounded in `hv-work/src/*` (10-config, 12-renderer-textures, 14-world,
15-effects, 19b-archetypes, 20b-abilities, 21-waves, 21u-ult, 22-companion, 27-hud) and the units module
(`hv-units-source.html` CFG / PASSIVE / ULT / ULTDESC, teams). Every idea names the system it reuses.

---

## 0. The premise in one sentence

**The world is a drawing that only stays real while it is watched; the Draftsman who drew it has put down the pen,
blank paper (the Hollow) is eating in from the margins, and you are the last Watchman of the Vigil, whose lamps keep
the page inked, coloured and boiling.**

"Hollow Vigil" means two things at once, and the game should let both be true:
1. *A vigil kept over a hollow*: the Watch of Ashridge stands at the edge of an emptiness so it does not spread.
2. *A vigil that is hollow*: nobody knows whether the Draftsman is ever coming back. Keeping watch may be pointless.
   The ending does not answer this. It lets you choose to keep watching anyway ("THE VIGIL IS KEPT · ENDLESS"
   already exists in `21-waves.js onBossDown`; it becomes the last line of the story).

Why this premise and not "a fantasy world rendered as ink": it turns every rendering decision the codebase already
made into a rule of the world. The player never needs a codex to understand it, because they can SEE it:
the line boils (the world is alive), the town's edge fades to bare paper with construction lines (the world is
unfinished and fraying), bodies ink in when they spawn and are scrubbed out when they die (existence and erasure are
literal), kill puddles accumulate as a record (the ground remembers), and a cut-in panel freezes a moment (the
Chronicle has noticed you).

## 1. The physics of the page (fiction → the system that already draws it)

Six forces. Each one is something the renderer or the sim already does. The fiction only names it.

| force | what it is in the world | what the player sees | existing system |
|---|---|---|---|
| **THE LINE** | Existence. A thing is real because it is outlined. | the ink edge on every body, prop, roof | post pass `SKETCH_FS` (26-panel), baked outlines, `ringStack` |
| **THE BOIL** | Life. The line is re-inked ten times a second by everyone watching it. A line that stops boiling is dead or held. | the 10 Hz wobble | `BOIL` clock (one clock for shader seed, outline row `OV`, ink-canvas wobble); held during hitstop |
| **COLOUR** | Memory / warmth. Colour is laid in only where the world is *loved*: lived-in, lit, remembered. | the full-saturation wash under the ink | `FLOOR_TINT`, `COLOUR_SAT`, wash / paper-mix dials next to `CLARITY` (12-renderer-textures) |
| **INK** | The substance of being. Spilled when things die, pooled in wells, burned in lamps, raw in slimes. | kill puddles, splats, drips, the blot creatures | decals (15b-decals, `DECAL_LIFE` 55 s "a RECORD"), slimes, oil slicks |
| **PAPER** | The Hollow. Not darkness: blankness. It does not destroy, it *un-draws*. | bare paper with construction lines past `BOUNDS` | `edgePlaneTex()` hand-cut window (12-renderer-textures:166) |
| **THE PANEL** | Attention. When something matters enough, the Chronicle draws it as a panel. Being drawn in a panel makes a moment permanent. | cut-ins, killcam, letterbox, results card, style rank | `renderPanel`, `panel()` PRIO, killcam, `STYLE` D→SS, marks wall |

Three corollaries fall out and become design rules:

- **Land has three states, and all three are looks the renderer can already produce.**
  **COLOURED** (alive: the current shipped look, green field under crisp ink) →
  **LINED** (fading: colour drains to the cream-paper, pencil-wash look — *the exact look the user rejected on 9 Sep
  as "the entire world is white"*. That rejection is a gift: the dead look is now the look of dying land) →
  **BLANK** (erased: bare paper and a few construction lines, which `edgePlaneTex` already draws).
  Transition = the district palette crossfade that `districtStep()` already runs over 1.2 s (floor colour, clear
  colour, fog), extended to also lerp `COLOUR_SAT` and paper mix.
- **Death is erasure, spawning is being drawn.** The 3-stage spawn ink-in (line alone → pencil under-stroke →
  colour and shadow, 15-effects:325) and the kind-specific scrub-out (`deathKind`: pop / crack / scribble / collapse)
  are not effects any more, they are what happens to a person. NPC dialogue can refer to "being scrubbed".
- **Stillness is dangerous both ways.** The Hollow erases what nobody watches; the Gilt Order (below) wants to
  varnish the world so the line never moves again. Hitstop (the boil held) is a sliver of that stillness; the domain
  ultimate is you holding the whole page still on purpose. Two headhunter / archer passives already reward stillness
  (HEADSHOT "a still target takes 1.5x", STEADY AIM): the world agrees that a still figure is easy to trace.

### What the SKETCH OFF setting is

The pause-menu SKETCH OFF shows the un-inked pixel world. Do NOT make the setting a mechanic (it is an accessibility
and performance switch and must stay one). Instead the fiction borrows its *look* for one place only: **the Reference**,
the Draftsman's pixel model the drawing was copied from, seen once in the finale (§9). Buildable: the post pass can
already be bypassed; drive the same uniform path the setting uses from a story flag for one sheet.

## 2. The shape of the open world: the Folio

### 2.1 Sheets, not a seamless map
The world is **the Folio**: a bound sketchbook of *sheets*. Each sheet is one playable page the size of today's town
(`T.BOUNDS` 52 × 68 units), with its own prop layout, palette, fog, rhythm and population. Sheets are joined edge to
edge on a map grid. Walking into an **inked** edge of a sheet turns the page: the travel plate (`portalSim`, the 0.8 s
swap behind a plate), a page-tilt camera beat (23-camera `PAGE` layer: "the camera orbits the look point, the page's
top edge lifts") and you arrive at the matching edge of the neighbour sheet. Walking into a **blank** edge just meets
paper: the clamp in `14-world.js` stops you and a construction line scratches under your feet.

Why sheets: it is the only open world this codebase can carry honestly (one scene, one fog volume, pooled actors,
software fallbacks). It is also the fiction: a sketchbook has pages. Nothing in the world has to pretend to be seamless.

Per-sheet data is a superset of the existing `DISTRICTS[]` rows (name, floor, paper, fog, forced `mod`), plus:
`props` (the `natureSprite`/house list 14-world builds at boot, now per sheet), `lamps` (positions, `lampObj`),
`edges` {n,e,s,w: sheetId | null}, `gates` (portal pairs), `roster` (HVU kinds + game types by `unlock` tier),
`factions` (who holds it, team ids), `hand` (line weight / hatch / boil rate, §2.4), `state` (COLOURED/LINED/BLANK).

### 2.2 The Margin: the edge that moves
Every sheet has a **Margin**: the blank region past its inked window. Today that window is fixed (`edgePlaneTex`
cuts one ragged rectangle out of a 512-px paper plane at |x|<30, |z−6|<40). In the open world the window is the
*claimed* part of the sheet and it changes:

- **Lighting a lamp inks the land around it.** Re-bake `edgePlaneTex` with one more `destination-out` hole: a ragged
  circle around the lamp. Props inside the new hole play the spawn **ink-in** (line → under-stroke → colour), so
  houses, trees and fences are literally drawn into existence in three held drawings on the 10 Hz clock. This is the
  open world's core reward animation and it costs one texture re-bake plus the existing ink-in on props.
- **An untended lamp gutters.** Its land goes COLOURED → LINED → BLANK over in-game days (the `districtStep`
  crossfade, local). Props in a BLANK hole play the scrub-out and stop colliding (`c.col.r=-1`, the trick broken barrels
  already use).
- **The map is the drawing.** The pause page shows the Folio as a spread of sheet thumbnails; each thumbnail is that
  sheet's current edge-plane texture (already a 512-px canvas) at small size. You literally watch your map ink in.

### 2.3 The watched circle: style rank as a force (strongest mechanical idea)
Being *witnessed* makes things real, and the Chronicle's attention is the style rank. So: **on BLANK or LINED land, a
circle of colour and line follows you, and its radius is your style rank.** At D you fight inside a lamp-sized pool
of ink; at SS the land around you floods back to full colour and props ink in as you fight past them. Stop fighting
well and the paper closes in.

Buildable on: `STYLE.rank` (already read by the pickup magnet in 21-waves), the edge-plane window (a second
dynamic hole centred on `P.x,P.z`, or cheaper: a radial term in the post pass keyed on the player's screen point that
lerps paper-mix/`COLOUR_SAT` by distance), and ink-in for props crossing the radius. It makes the style system, which
in a wave game is a score, into the thing that keeps you alive in the Hollow. It also gives the Margin a *skill*
reason to go in early.

### 2.4 Every faction draws in its own hand
Regions differ by *how they are drawn*, which is pure dial work on systems that already exist:
line weight (`INK_D`, rim thickness), hatch (`SHADOW_HATCH`, `SHADOW_LO`), boil rate (the `BOIL` step, held or
quickened), colour saturation (`COLOUR_SAT`), paper tint (`DISTRICTS[].paper`), fog (`applyFog`). One table per sheet;
no new shader.

### 2.5 Day and night
Day: explore, ink land, talk, trade. Night: **Vigils**, the existing wave game relocated to a lamp site under assault
(§8). The `THE NIGHT DEEPENS` shout (27-hud `countKill`) and the `nightfall` modifier (no armour regen, fog 20–50)
already exist; night is `nightfall` applied to the sheet plus the Hatch roster (§3).

## 3. Who you are: the Watchman and the two studies

- **You are the Watchman of Ashridge** (10-config's own header: "Ashridge Junction: the watchman, the town, the
  portals"). The Watch's oath is to keep the lamps.
- **Knight and Archer are two studies of the same figure.** An artist draws a person twice on one sheet, two ways.
  The mid-fight class swap is turning to the other study. This is why swapping is instant and why both share one
  health bar and one rank. More studies are found in the world as **study sheets** (§8.3).
- **The companion archer is not a hireling.** She is drawn from the skeleton family (`undead*` sheets; after the
  9 Sep flashing fix both she and the archer class use only that family). In the fiction she is **the archer-study of
  the previous Watchman**, the half of him that did not go hollow. Her quiet kill words ("heh", "got one", "mine",
  22-companion) are the voice of someone who has done this before. Her true name and the other half are the main
  arc's spine (§9).
- **THE LAST STAND / RISE** ("PRESS J TO RISE", mark ROSE ONCE): the Vigil redraws you once. Out of that, death
  sends you back to the last lit lamp: you are re-inked there (spawn ink-in on the hero).
- **The domain ultimate** IRON VIGIL / HOLLOW VOLLEY (the "ZA…" beat, enemies at 0.42 time inside, `21u-ult`):
  the Watchman holds the page still. Short, costly, and the one time the Vigil does what the Gilt Order wants to do
  forever. NPCs of the Order react to it (§4.B).
- **The abilities are Watch craft, drawn with the lamp-ink**: SKETCH SIGIL (close a drawn loop = "THE SEAL"),
  SCRIBBLE DOUBLE (a quick gesture drawing of yourself that enemies believe is real), INK CYCLONE, IRON FALL
  ("THE EARTH ANSWERS"). The sigil's closed loop becomes the world's universal verb for *binding and restoring* (§9).

## 4. The factions: eight hands on one page

The units module lets any unit fight for or against anyone (`e.team`; the player is team 0; a unit targets the
nearest body of *another* team; `dealHit` lands on anyone of another team). So factions are simply **team ids with a
standing toward the Vigil and toward each other**. The accents already group the ~40 kinds into families by colour;
the faction lines below follow the accents, so every faction reads at a glance in the world.

### A. THE WATCH OF ASHRIDGE — the Vigil (steel blue, red for blades)
- **Units:** THE WATCH, THE CAPTAIN, archer, pikeman, swordsman, knight, axeman (knight and axeman share accent `#8fa8c8`).
- **Want:** keep the lamps lit; hold the Junction; push the Margin back one lamp at a time.
- **Territory:** Ashridge Junction, the Lamplit Rows, the lamp-posts along every road.
- **How they fight:** SHIELD WALL (less damage near another watch), STANDARD (captain: allies take 15% less), SET SPEAR,
  STEADY AIM. They are a *formation* faction: they are strongest when you fight beside them.
- **Weakness:** too few, too tired. Every Vigil costs watchmen; lamps you do not help with fall.

### B. THE GILT — the Illuminated Order (cream and gold)
- **Units:** priest, templar, THE HIEROPHANT, the Order's WARDENS (the HVU `warden` body: BULWARK, AEGIS).
- **Want:** *finish* the world. Gild every line and varnish it so it never moves again; a still line cannot be
  erased. They are right that it would stop the Hollow. They are wrong that anything would still be alive.
- **Territory:** Vellum Abbey and its chapter-houses; gold-leaf borders around their land.
- **Signature look:** on Gilt land **the line does not boil**. The `BOIL` clock is held for the whole sheet (the same
  hold hitstop uses). It is silent, beautiful, gold-bordered and deeply wrong-feeling. No new rendering.
- **How they fight:** BLESSING, ZEAL, WARD, HOLY GROUND, SANCTUARY, JUDGEMENT pillars. A support-heavy wall that makes
  every other body near it better, so they ally with whoever is useful.
- **Relation to the Vigil:** allies of convenience who think the Watch is a stopgap. When you use your domain ultimate
  near them, templars shout approval ("HOLD IT."): they believe you are proving their point.

### C. THE UNDERDRAWN — the undead (bone white; the necromancer green)
- **Units:** skeleton, BONE GUARD, GREATSWORD, BONE ARCHER, THE REVENANT, the necromancer; bats follow them.
- **What they are:** people who were erased, but not all the way. A figure is drawn over an armature of construction
  lines; a skeleton is exactly that armature, the pencil stick-figure under a scrubbed-out person. That is why they
  rise again (BONE DEEP "rises once more… unless a heavy blow ended it": a light hit rubs the drawing, a heavy one
  tears the paper) and why the REVENANT "rises three times".
- **The necromancer** traces over armatures with spilled ink (its accent `#7fbf5a` is the slime green: it uses Blot
  ink). SOUL HARVEST: a death nearby feeds it, because a death spills ink.
- **Want:** to be *drawn again*. To be finished. Many are the Watch's own dead.
- **Territory:** the Palimpsest, where the land has been erased and redrawn so many times that old lines show through.
- **Relations:** the Gilt call them heresy (a figure is drawn once) and crusade against them. The Hatch hunt them for
  sport. The Vigil's attitude is the player's choice (§6).

### D. THE GREENHAND WARBAND — the orcs (brush green)
- **Units:** orc, ELITE ORC, IRON ORC, BERSERKER, THE WARCHIEF, the rider (its accent `#6a9a3c` is the orc green: it
  rides with them).
- **What they are:** figures from **another page**, drawn by a different hand, in thick wet brush rather than pen.
  Their own sheets are being eaten from the far margin, so they are crossing onto the Vigil's page.
- **The drum:** WARDRUM ("its side moves and swings 15% faster near it") is their secret. The orcs do not need lamps,
  because **the drum is their boil**: they keep their own line alive by rhythm. Around a warchief, orc sprites
  step their outline row faster and the music's taiko layer (17-audio) takes the beat. Kill the drummer and their
  lines go still and start to fade.
- **Want:** land that stays inked. Conquerors because they are refugees.
- **Territory:** the Brushlands, a war camp moving across the eastern sheets.
- **Relations:** at war with the Watch over the eastern lamps. Despise the Gilt (a varnished world is a drumless one).
  They are the one faction that can survive the Hollow without the Vigil, which makes them the most important possible ally.

### E. THE BLOT — loose ink (slime greens, rainbow washes, plague purple)
- **Units:** the game's slimes (brown, rainbow, kitten, brute, bombers), HVU slime and slimelet, THE PLAGUEBEARER.
  Boss: **THE WARDEN** (§7).
- **What they are:** ink with no figure to hold it. Spilled from broken wells, from battles, from the dead. SPLIT
  "dies into two small slimes" because a blot divides; PLAGUE is **the Bleed**, ink soaking through the paper into
  whatever it touches; VOLATILE bombers are ink gone to gas.
- **Want:** nothing. They spread and pool. They are the only faction that attacks everyone, and the raw material of
  everyone else (necromancers, lamps and the Warband's paint all use Blot ink).
- **Territory:** the Spill, where Ashridge's great inkwell cracked. The oil-barrel fire ("on nobody's side",
  19b-archetypes:238) is how the world burns ink off.

### F. THE HATCH — creatures of shadow (moonlit blue-grey, bear brown)
- **Units:** werewolf, werebear, bat.
- **What they are:** the world only puts hatching in true shadow (the shipped rule: hatch only in shadows). The Hatch
  are people who are drawn one way in lamplight and another way in cross-hatch: citizens of the Lamplit Rows by day,
  beasts in the unlit spaces at night. FRENZY and UNSTOPPABLE are what happens when nobody is watching you.
- **Want:** darkness between lamps. Not the Hollow: they die on blank paper like everyone else (no shadow, no hatch).
- **Territory:** the Hatchwood by night, the gaps between lamps in the Rows.
- **Relations:** hunt the Underdrawn; hate the Watch's lamps; secretly fed by the Redrafters, who want lamps out.

### G. THE REDRAFTERS — the cabal (violet, indigo, slate)
- **Units:** wizard, THE MESMER, THE WARLOCK, THE HEADHUNTER; the game's sigil caster and summoner archetypes.
- **What they are:** scholars who worked out that a figure's allegiance, its colour, even its name, are just ink, and
  ink can be drawn over. MASS HYPNOSIS / HYPNOSIS literally re-inks a body onto the mesmer's side for 8 seconds (the
  module's `charm()` changes whose side a unit fights for). DEATH MARK is a mark for erasure ("marked bodies take 30% more
  from everyone"). The Headhunter traces still targets.
- **Want:** the Draftsman's pen. If the hand is gone, someone should hold it.
- **Territory:** the Tracing Tower, a sheet overlaid with a faint offset copy of itself (tracing paper).
- **Relations:** use everyone. Snuff lamps to make the Hatch stronger; sell charms to the Warband; the real
  antagonist of the middle act (§9).

### H. THE HOLLOW — not a faction, an absence
The Hollow has no units of its own. It **hollows** units of any faction: see §5.

### Standing matrix (the default teams on shared sheets)

| | Watch | Gilt | Underdrawn | Warband | Blot | Hatch | Redrafters |
|---|---|---|---|---|---|---|---|
| **Watch** | – | uneasy allies | player's choice | war (east lamps) | hostile | hostile | hostile once exposed |
| **Gilt** | | – | crusade | contempt | purge | purge | unaware, then war |
| **Underdrawn** | | | – | indifferent | feed on it | hunted by | enslaved by (necromancer bargains) |
| **Warband** | | | | – | hostile | wary | buy charms from |
| **Blot** | | | | | – | hostile | harvested by |
| **Hatch** | | | | | | – | fed by |

Everything in that table is one `team` assignment at spawn plus an ally table the host consults. A three-way battle
is already legal in the module: bodies of three team ids all fight each other.

## 5. The Hollowed and the Signed

### 5.1 HOLLOWED: one affix, forty enemies
Any unit that stands too long on blank paper becomes **HOLLOWED**: its colour and fill are gone and only the boiling
ink contour remains, walking. It keeps its whole move set, passive and ultimate, and it joins team Hollow, which
fights every other team.

- **Look:** exactly the first held drawing of the spawn ink-in ("the ink line alone"), and the line-only
  `ringStack` / `sheetOutlPlain` contour that ghosts use since FIX4, over the paper silhouette sprite
  (15-effects:314, "under the ink line it turns the rig into a line drawing"). It is a state the renderer already has.
  It must be the line-only contour, never `sheetHalo` (the filled dilation prints a black silhouette: memory note).
- **Rules:** a new entry in `AFFIX` (19b-archetypes) next to ARMORED / SWIFT / VOLATILE / VAMPIRIC / HEXED / WARDED,
  so elites, the gold under-stroke, `eliteSeen()` shout ("HOLLOWED TEMPLAR") and the wave budget all apply.
  Mechanical identity: takes no hitstop and gives none (the Hollow does not boil); cannot be healed by any faction;
  on death it does not leave a kill puddle (it has no ink to spill) and does not scrub out: it just stops being drawn.
- **Why it matters for scope:** the Margin gets a full bestiary (40 hollowed kinds) for the cost of one affix and one
  render state.

### 5.2 THE SIGNED: the four named figures
A figure who **signs itself** onto the page cannot be quietly erased: a signature is the one mark the Hollow respects.
THE KNIFE, THE BLADE, THE LANCER and THE WATCH are the four Signed, custom bodies with the best move sets in the module.
Each is a former study-figure of the Vigil who took their own name, and each has chosen a side:

| Signed | kit (module) | allegiance | region | what they want |
|---|---|---|---|---|
| **THE WATCH** | shield cuts, guarded thrust, SHIELD WALL | the Watch: the last loyal one, your sergeant | Ashridge Junction | someone to finally relieve him of the post |
| **THE KNIFE** | backstab, launcher, plunge, SHADOWSTEP (three dashes) | the Redrafters | Lamplit Rows | pays in lamp-ink, snuffs lamps for them; wants a name nobody can trace over |
| **THE BLADE** | tempo, rising slash into slam, ASTRAL TEMPEST spins | the Gilt, as its sworn champion | Vellum Abbey | to be varnished, to fight forever as their perfect figure |
| **THE LANCER** | long thrusts, launcher, SKYFALL spear throws | the Warband (rides with the riders) | the Brushlands | a page that will not be erased under them |

Beating a Signed in a duel (it is a boss fight: `bossNameEl`, boss camera, boss music mode) **takes their signature**:
- their PASSIVE becomes a choice card for your Watchman (BACKSTAB, TEMPO, FIRST BLOOD, SHIELD WALL → the `PERKS`
  table and card offer 21-waves already has), and
- they may be **recruited**: respawned as team 0, following you like the companion archer (the module's AI already
  fights for team 0 against everyone else). A Signed companion is the reward for sparing them (§6).

## 6. How allegiance works in play (all on the team system)

- **Skirmishes.** The world's random encounters are the units demo's INK-vs-RED battles moved into sheets: you walk
  onto two factions already fighting (Gilt templars and a necromancer's bones; Warband riders and the Watch at a
  lamp). **The side you strike first is the side you are against.** The other side goes to team 0 for the fight.
  Standing with each faction moves by the outcome. No dialogue tree needed; the choice is made with a sword.
- **Standing.** Per faction, −3..+3. At +2 their units spawn on team 0 on shared sheets and their vendor (the hub,
  §8.1) opens. At −2 they spawn hunting you (`opts.hunt`). Hitting an allied body once is forgiven (a "!" mark over its
  head, `fx.mark`), twice flips it.
- **Mesmer is the faction mechanic's villain.** HYPNOSIS / MASS HYPNOSIS flips your allies for 6–8 seconds, including
  a recruited Signed. A fight where you must win without killing your own charmed companion is a quest pattern that
  costs nothing new.
- **Rallying calls read as politics.** WARCRY (elite orc: "rallies its side"), STANDARD, HOLY GROUND, WARDRUM,
  SANCTUARY all buff *their team*. If you are on their team, they buff you. Siding with the Warband means the drum
  speeds *your* swings. That is a felt reward for diplomacy using passives that already exist.
- **The Underdrawn choice.** A necromancer can be killed (Gilt standing up) or its armatures can be **sealed**: close a
  SKETCH SIGIL loop around a risen skeleton and, instead of damage, it is redrawn as the person it was (a watchman,
  a villager), who walks to the nearest lamp. This is the redemptive verb of the whole game and it reuses the sigil's
  closed-loop detection (`SIG`, mark THE SEAL).

## 7. The two bosses, the gates and the districts

### THE WARDEN — the Blot that wears a title
The game's Warden is a giant tinted rainbow slime (`T.SLIME.warden`, 3 phases, triple charge, a slam that throws its
own kittens out). The units module also has a **warden** body: a shield-bearer with BULWARK and AEGIS. The fiction
joins them:
- Ashridge's great inkwell, the **Well**, holds the ink the Watch burns in every lamp. The Gilt Order sent a Warden
  to keep it. A generation ago the Well cracked, the Warden went in after the ink, and the ink kept the title.
- **THE WARDEN** is the Well's Blot with the office's name in it: it still guards (it hunts whoever comes near the Well),
  it still charges like a shield-bearer (the triple charge), and it throws out kittens because a blot splits.
- Its phase shouts IT REELS / IT RAGES stay. A fourth beat is added in the world version: at the end, the Order's
  own warden unit (the HVU body, AEGIS up) steps out of the Blot as the last of its ink, as a short second fight.
  Seal it with a sigil loop and the Gilt Order's standing jumps: you gave them back their Warden.
- **Where it lives:** the Well under Ashridge's North Gate. It is the first boss and the key to the open world:
  **killing it returns the Well's ink to the lamps**, so lamps can be lit beyond the town. The gold ring on both gates
  when a boss falls ("THE GATE OPENS", `onBossDown`, `p.gold`) becomes "the lamps can burn again".

### THE HOLLOW KNIGHT — the Watchman before you
The Hollow Knight is drawn from the skeleton family (`undead` sheet, tinted), fights with the knight's three-hit chain,
"closes the gap the way you do" with a dash that leaves afterimages, and fires a fan of bolts at range. It is both of
your studies in one body.
- **He is the last Watchman**, the one who kept the Vigil before you. He tried to end it: he believed the Gilt, held
  his domain over the whole page, and when he let go the page had forgotten him. The knight-study went hollow; the
  archer-study stayed behind and became **your companion**.
- **Where he lives:** at the heart of the Hollow, inside the Margin, walking the unlit road toward Ashridge.
- **He appears before he is fought.** On BLANK land, from Act I on, a line-only figure is seen at the edge of the
  fog, afterimage dash and all (the afterimage ghost system). The companion goes quiet when he is near (her kill
  barks are suppressed).
- **The fight:** the existing `EBRAIN.hollow`, in his own sheet of bare paper where your watched circle (§2.3) is the
  only colour on screen. The mark FACE TO FACE is already named for it.
- **The choice:** kill him (erase), or close a SKETCH SIGIL loop around him while he is stunned and redraw him. Redrawn,
  the companion and he merge back into one figure (§9).

### The gates and districts
- The existing **North Gate / South Gate** linked pair (`makePortal(0,-22)` / `(0,36.5)`) becomes the Folio's
  **binding**: every lit gate-lamp in the world is a portal endpoint, and any two lit gates are linked. The travel card
  "to the North Gate" (`travelTo`) names the destination sheet. Fast travel is walking into a lamp you lit.
- The three shipped districts become the first three sheets of the story in the order they already run:
  ASHRIDGE JUNCTION (no modifier) → THE LAMPLIT ROWS (forced `nightfall`) → THE HOLLOW (forced `frenzy`, darker
  paper `0xd9ceb6`, fog 16–44). That progression (brighter town, night streets, the dim Hollow) is already the arc's
  palette.

## 8. The regions

Each region is 2–5 sheets. "Hand" = the dial settings that make it look different (§2.4). "Tier" = the CFG `unlock`
tiers of what lives there, which already rank the 40 kinds from 1 (slime) to 10 (warchief).

### 8.1 ASHRIDGE JUNCTION — the hub (the shipped town)
- **Look:** exactly today: green field, red/blue/grey roofs, market square, eight lamps, six barrels, tree ring. The
  most COLOURED place in the world, and the only place the colour never drains while the Watch holds.
- **Who:** the Watch (THE WATCH, THE CAPTAIN, archers, pikemen); traders from every faction with standing ≥ +2.
- **Conflict:** the town is a crossroads. Every road out is a different faction's border, which is why it is a Junction.
- **What the hub does, and on what system:**
  - *The Lamp Board* (the market square's centre lamps): the Folio map, sheet thumbnails from each edge-plane texture.
  - *The Record* (the town wall): the marks wall (16-marks / 27-hud `MARKS_HOW`), grown with faction and region marks.
  - *The Chronicle* (the Watch-house): your quest log is a manga page assembled from **your own captured panels**:
    when a quest beat fires, `renderPanel` already renders the world into a panel texture; keep a small copy. The log
    is literally drawn from what you did.
  - *The Roster* (the Captain): the bestiary. The HVU name + moves + PASSIVE + ULT text for all 40 kinds ("the best
    writing in either codebase … currently has nowhere to live", HVU-map-ui) unlocked by fighting each kind.
  - *The Scribe*: spend ink on choice cards (the between-wave `PERKS`, now permanent up to 3 stacks) and on studies.
  - *The Well* under the North Gate: the Warden's lair, then the ink source for every lamp.
- **Hub Vigils:** if you leave the Junction's lamps untended too long, the night brings a siege: the full shipped wave
  game (typed pools, budget, elites, bounties, modifiers, choice cards, boss waves) plays out in town. The shipped game
  *is* the hub's defence event, untouched.

### 8.2 THE LAMPLIT ROWS — back streets that exist only under light (shipped district 2)
- **Hand:** narrow sheets of houses; every lamp throws a COLOURED pool, the space between lamps is LINED and hatched.
  Forced `nightfall` (no armour regen, fog 20–50).
- **Who:** the Watch (lamplighters), the Hatch (werewolves in the gaps between lamps), THE KNIFE, bats.
- **Conflict:** someone is snuffing the Rows' lamps one by one and werewolves are rising in the dark they leave.
  It is THE KNIFE, paid in lamp-ink by the Redrafters.
- **Signature play:** the existing *defend-the-lamps* waves (`lampObj` {hp, lit}, `bindLamps` sends enemies after lamps)
  become the region's quest verb. Lighting a lamp turns a werewolf in its circle back into a frightened citizen
  (a scripted team flip to a neutral team, then despawn with ink-in reversed).

### 8.3 VELLUM ABBEY — where the line stands still (Gilt)
- **Hand:** cream paper, heavy gold rims (elite gold under-stroke colour `0xffd24a` on every prop outline), full
  saturation, **the boil held** for the whole sheet. Music: the title-mode pad without the pulse.
- **Who:** priests, templars, THE HIEROPHANT, Order wardens, THE BLADE. Hollowed pilgrims at the gate.
- **Conflict:** the Order is preparing **the Varnish**: a rite to hold every line in Ashridge still, forever. They need
  the Watchman's domain ultimate to prove it works.
- **Landmark:** the Scriptorium of Studies, where the Order keeps sheets of figures they have varnished. Study sheets
  for new player studies are found here (and as the reward from THE BLADE).
- **Signature play:** a fight on a still page feels different by itself: hitstop has no boil to hold, so the Order's
  sheet uses the *inverse*: everything boils only when struck (a local boil bump on hit, the same +17 BOIL bump impact
  frames already get).

### 8.4 THE PALIMPSEST — the land that was erased and redrawn (Underdrawn)
- **Hand:** LINED land with **older drawings showing through**: faint offset copies of houses and walls that are not
  there any more, drawn with the afterimage/ghost sprite system (line-only contour, 0.45 alpha) on props. Paper tint
  slightly grey. Heavy accumulated decals.
- **Who:** skeletons, bone guards, greatswords, bone archers, THE REVENANT, necromancers; Gilt crusaders in the field.
- **Conflict:** a three-way war. The Order is burning armatures; necromancers are raising them with Blot ink; the
  armatures just want to be people again.
- **Signature play:** **the ground remembers.** Kill puddles persist here across visits (the decal pool, `DECAL_LIFE`
  as a RECORD). A necromancer's `ncSummon` raises bodies at existing kill-puddle positions instead of randomly, so the
  battles you fought here come back to fight you. Seal armatures with sigil loops to redraw them (§6).
- **Named place:** the Old Watch Barracks, where the Hollow Knight's company was scrubbed out. Its skeletons wear the
  Watch accent. Sealing all of them is the quest that reveals who the companion was.

### 8.5 THE BRUSHLANDS — another artist's page (Warband)
- **Hand:** thick ink (`INK_D` up, fatter rims), little hatch, broad flat colour washes, faster boil near drums,
  long grass sheets with palisades. Music: taiko layer foregrounded (17-audio already has taiko crits).
- **Who:** orcs, elite orcs, iron orcs, berserkers, riders, THE WARCHIEF, THE LANCER; axemen deserters from the Watch.
- **Conflict:** the Warband has taken the eastern lamps. They do not need them for light: they are burning the lamp-ink
  as war paint. The Watch wants the lamps back; the Warband's own home sheets past the far margin are going BLANK.
- **Signature play:** **silence the drum.** The Warchief's WARDRUM is an aura; a camp at night is a wave defence where
  killing the drummer (an elite orc carrying the drum affix) drops the whole side's speed and starts their lines fading.
  Or win standing +2 and fight *with* the drum.
- **The deal:** the Warband will stand with the Vigil at the end if you give them a lit lamp-sheet of their own: a
  quest to ink a whole BLANK sheet for them, lamp by lamp.

### 8.6 THE HATCHWOOD — a forest drawn only in shadow (Hatch)
- **Hand:** the darkest sheets: `SHADOW_HATCH` and `SHADOW_LO` pushed up so nearly every surface is cross-hatched,
  low saturation, `fog` modifier (aggro 0.6, fog 13–42), dense `tree_*` props with sway.
- **Who:** werewolves, werebears, bats; hunted Underdrawn; a Redrafter camp feeding the pack.
- **Conflict:** the Hatch are multiplying because lamps are going out everywhere; the Redrafters are breeding them
  with charms to take the Rows.
- **Signature play:** carry light. A lamp-bearer escort (the companion or a watchman) makes a moving COLOURED circle; inside
  it werebeasts lose FRENZY stacks. Oil barrels (the fire "on nobody's side") burn hatch-thickets away and clear paths.
  Mark ASHRIDGE BURNS / bounty BURN TWO already exist.

### 8.7 THE SPILL — where the Well broke (Blot)
- **Hand:** vivid, oversaturated puddle colour, rainbow-slime washes, drips and splats everywhere, oil slicks
  (`slickAdd`). The only place with MORE colour than Ashridge, and all of it loose.
- **Who:** every slime archetype, bombers, THE PLAGUEBEARER (the Bleed), necromancers harvesting ink.
- **Conflict:** the Well's crack is still leaking downhill from Ashridge. Everyone wants the ink.
- **Signature play:** ink is loot. Slime kills drop **lamp-ink** (the pickup / `M.pending` drop system, `LODESTONE`
  magnet), the currency for lighting lamps and buying cards. PLAGUE / OUTBREAK zones are where the best ink pools.

### 8.8 THE TRACING TOWER — the page laid over the page (Redrafters)
- **Hand:** tracing paper: the whole sheet drawn twice, the second copy faint and offset a few pixels, boiling out of
  phase (the ghost/afterimage system on props, plus the paper tint pale blue-violet).
- **Who:** wizards, THE MESMER, THE WARLOCK, THE HEADHUNTER, sigil casters, summoners; charmed bodies of every faction.
- **Conflict:** the Redrafters are trying to trace the Draftsman's hand from the oldest lines in the Folio, so they
  can draw the world themselves.
- **Signature play:** **allegiance is ink.** Every fight has MASS HYPNOSIS flipping bodies back and forth; charmed
  enemies fight for you, charmed allies fight you. DEATH MARK makes the mark visible as a hand-drawn cross on the target.
  The game's SKETCH SIGIL and hostile sigil casters make this the "drawing duel" region.

### 8.9 THE MARGIN and THE HOLLOW — where the page is blank (shipped district 3)
- **Hand:** BLANK. Paper, construction lines, a few unfinished props (line only). Colour exists only inside your watched
  circle (style rank) and around lamps you light. Forced `frenzy` (the shipped district's modifier).
- **Who:** HOLLOWED units of every faction (§5.1). The Hollow Knight at the centre.
- **Conflict:** it is growing. Every region's untended lamps feed it.
- **Signature play:** fight well or be erased. Lamps here cost the most ink and each one inks a big hole in the page.

### 8.10 THE GUTTER — the white space between panels (finale)
The space between two comic panels is called the gutter. The last sheet is the gutter of the Folio: a white strip
between two panels, with the letterbox bars (the killcam / boss-cam bars) as its actual walls. Beyond it is the
**Reference** (§1): the pixel world with SKETCH off, where the Draftsman's model of Ashridge sits, unwatched and
perfectly still.

## 9. The main arc: "The Vigil Book", four chapters

Every chapter ends on a results card that is a Chronicle page (the existing results card, retitled per chapter).

**CHAPTER I — THE WELL.** Ashridge Junction is the only coloured place left nearby, and its lamps are burning the
last of their ink. The first night is the shipped game: waves in the town, the Warden wave, the gate. You go down the
North Gate into the Well, kill THE WARDEN, and seal (or kill) the Order's lost warden inside it. The Well's ink flows back to
the lamps. "THE GATE OPENS" means the lamps can now burn beyond the town. First sight of a line-only knight at the
fog's edge; the companion stops talking.
*Systems:* the shipped wave game, `onBossDown`, gold gates, district crossfade.

**CHAPTER II — THE LAMPS.** Open world. Relight the Folio. Four regions, each with one Signed figure and one faction
problem, in any order: the Knife in the Rows, the Blade at the Abbey, the Lancer in the Brushlands, the Palimpsest's
barracks. Every Signed you beat or recruit, every lamp you light, pushes the Margin back and changes the standings.
Mid-chapter reveal (the Old Watch Barracks): the sealed skeletons are the Hollow Knight's company, and the companion
is his archer-study.
*Systems:* sheets, lamps and edge-plane re-bake, skirmishes and teams, Signed duels, perks.

**CHAPTER III — THE VARNISH AND THE TRACE.** Two plans to save the world, both terrible, come to a head at once.
The Gilt will perform the Varnish at Vellum Abbey (hold every line still); the Redrafters will trace the Draftsman's
hand at the Tower (redraw the world as theirs). Both need the Watchman: the Order needs your domain, the Tower needs
your companion (the only living figure drawn by the Draftsman's own hand in two studies). The Mesmer takes her.
You choose which rite to break first; the other partly succeeds, and that region's sheets go to that faction's
extreme (varnished still, or traced and charmed) for the rest of the game.
*Systems:* faction standings drive who spawns on your team in the two assaults; MASS HYPNOSIS flips the companion.

**CHAPTER IV — THE HOLLOW.** You go into the Margin with whoever will come: recruited Signed, allied factions whose
units now spawn on team 0 around you (the Warband, if you gave them a sheet, arrives with the drum). The Hollow Knight
waits at the centre in a sheet of bare paper. Fight him where your style rank is the only colour. At the end, choose:
- **ERASE** him. The companion leaves you; the Margin recedes; you are the Watchman now, alone.
- **SEAL** him with a sigil loop while he is stunned. Knight-study and archer-study redraw into one figure, who walks
  into the Gutter and does not come back.

**THE GUTTER.** Past the last panel is the Reference: the pixel Ashridge with no ink, no boil, no colour wash, perfectly
still. Nobody is there. No Draftsman. The Watchman turns back (or does not: the credits wait until you do). The last
card: **THE VIGIL IS KEPT · ENDLESS** (the shipped string). Post-game is the Endless Vigil: the Margin creeps forever,
lamps gutter on their own, nights get harder (the shipped difficulty curve, `WAVE.n` never resets), and the question
of whether it is worth watching an empty page is answered only by playing.

## 10. Reasons to explore (and what each one pays)

1. **Ink the map.** The Folio map is blank sheets that fill in as drawings. The most direct exploration drive there is:
   you see the world being drawn by you. (edge-plane re-bake + prop ink-in)
2. **Unfinished props.** Some houses, bridges, wells and shrines are only line drawings on LINED land. Reaching one and
   holding the ground inks it in and gives what it holds: a heart well, a bench (rest = armour/energy refill), a
   shortcut gate. (ink-in stages, collider toggled by `col.r`)
3. **Study sheets.** Loose pages of the Scriptorium scattered across sheets. Each teaches a new ability slotting into
   the two-slot skill budget (IRON FALL, INK CYCLONE, HOLLOW RAIN, BOMB ARROW, HOOK LINE, SIGIL, SCRIBBLE DOUBLE are the
   first seven). Collecting a Signed's full sheet set is the long-term path to new studies. (`ABIL` table)
4. **The Roster.** 40 kinds × (seen, fought, beaten, beaten hollowed). Each unlock fills in that body's page: name,
   moves, PASSIVE, ULT and ULTDESC. A collector's reason to visit every faction. (HVU CFG text)
5. **The Record.** Marks per region and faction (THE SEAL, TURNED IT, CROWN BREAKER and the rest are already the right
   voice). (16-marks)
6. **Named elites.** Crowned bodies (affixes) roam sheets with a bounty poster in the Watch-house: FELL THE CROWNED,
   FINISH TWO, TAKE NO HIT. (`BOUNTIES`, `AFFIX`)
7. **Revenge on the page.** Where you die, the thing that killed you comes back HOLLOWED and crowned next time you pass.
   (death position + HOLLOWED affix + elite)
8. **Chronicle panels.** Rare moments (a 10-chain, an execution on a Signed, an SS kill) are captured into the log as
   panels. A player's journal of their own best frames. (renderPanel, style rank, killcam)

### Quest patterns (every one is an existing mode with a fiction attached)

| pattern | fiction | shipped system |
|---|---|---|
| **Vigil** | hold a lamp site through a night | the whole wave game: roster budget, elites, modifiers, cards, boss waves |
| **Lamplighter** | relight / defend lamps | `lampObj` defend-the-lamps waves, `bindLamps` |
| **Skirmish** | pick a side in someone else's battle | HVU teams, INK vs RED demo spawns |
| **Duel** | beat a Signed | HVU boss body + boss cam/music + `bossNameEl` |
| **Seal** | redraw instead of kill | SKETCH SIGIL loop detection |
| **Burn** | clear a nest | oil barrels, `igniteAt`, BURN bounty |
| **Contract** | kill a crowned body a certain way | `BOUNTIES` |
| **Drum** | silence (or join) a warband camp | WARDRUM aura, taiko layer |
| **Charmed** | win without killing your charmed friend | `charm()` |
| **Escort the light** | move a lamp-bearer through the Hatchwood | companion follow AI, local colour circle |

## 11. Buildability ledger

Cost: **S** = data/dials on an existing system; **M** = a new function on an existing system; **L** = a new system.

| idea | reuses | cost | note |
|---|---|---|---|
| Three land states COLOURED / LINED / BLANK | `districtStep` crossfade; `FLOOR_TINT`, `COLOUR_SAT`, paper mix, `CLARITY`; `applyFog` | S | extend the tween to two more dials |
| Per-faction "hand" | `INK_D`, rim, `SHADOW_HATCH`, `SHADOW_LO`, `BOIL` rate/hold, paper tint | S | one row per sheet |
| Still boil at the Abbey | the `BOIL` hold already used by hitstop | S | + on-hit boil bump (impact frames already +17) |
| Sheets and page-turn travel | `DISTRICTS[]`, `portalSim` travel plate, `PAGE` tilt, per-sheet prop list from 14-world | L | the real work: props/colliders/lamps rebuilt per sheet instead of once at boot |
| Lamps ink the land | `edgePlaneTex` (512-px canvas, `destination-out` holes), prop ink-in | M | re-bake on light; staged, never per frame |
| Watched circle = style rank | `STYLE.rank`, post pass radial term or a moving edge-plane hole | M | the single highest-value new mechanic |
| Factions as teams + standing | HVU `e.team`, `opts.hunt`, `charm()`, ally table in the host | M | the host contract in HVU-map-host already routes `hitPlayer` |
| Skirmishes | HVU demo's INK vs RED army spawn | S | spawn two teams on arrival |
| HOLLOWED affix | `AFFIX`, `makeElite`, spawn ink-in stage 1 (line only), `ringStack`/`sheetOutlPlain` contour | M | never `sheetHalo` (prints a black silhouette) |
| Signed duels + recruitment | HVU custom bodies, boss cam/music/`bossNameEl`, companion follow, team 0 | M | recruited Signed use the module AI as-is |
| Signature → perk card | `PASSIVE` text + `PERKS`/card offer | S | rules mostly already in `dealtMod` |
| Seal (redraw instead of kill) | SKETCH SIGIL closed-loop test, THE SEAL mark | M | replace damage with a team flip + ink-in |
| Warband drum | WARDRUM aura, 17-audio taiko layer, outline row step rate near the warchief | M | per-rig boil phase |
| Palimpsest raises at kill puddles | decal pool positions, `ncSummon` spawn points | S | persist a few decal positions per sheet |
| Chronicle journal from panels | `renderPanel` render target, `panel()` beats | M | keep 256-px copies |
| Folio map from edge planes | the per-sheet 512-px canvases | S | draw them small on the pause page |
| Hub Vigil sieges | the whole shipped wave game | S | the town's defence event, unchanged |
| The Reference finale | the SKETCH OFF path driven by a story flag on one sheet | S | the setting itself stays a setting |
| Save state | `localStorage` pattern already used for `hv_marks`, `hv_set`, `hv_sketch` | M | sheet states, standings, lamps, perks |

### Rules and risks this lens insists on
- **Keep the setting a setting.** SKETCH OFF must never gate content. The Reference borrows its look for one scene.
- **The rejected cream look is only for dying land.** The default world stays the user's approved "proper colour"
  (green field, full saturation, crisp ink). LINED land is a warning state, never the look of a region you live in.
- **Name collision.** The HVU `warden` body is also named THE WARDEN (sheet `templar`). In the world it becomes
  **ORDER WARDEN**; THE WARDEN stays the Blot boss.
- **Nothing blacker than the world's line.** Hollowed units, gold Gilt rims and tracing-paper doubles all obey the
  shipped rule: nothing on screen darker than `0x14151c`, nothing whiter than the paper, and no ink text inside an ink stroke.
- **Sheets load in stages.** 227 HVU sheets are too much for boot (HVU-map-render already calls for staged baking);
  each sheet's roster loads with the sheet, which the region design (one faction per region) makes natural.
- **One faction per region, with visitors.** It is fiction, but it is also the performance budget: a sheet bakes its
  faction's families plus two visiting kinds.
- **Never lose the fight.** The open world is a delivery system for the combat. Every region's signature play above is a
  way to *fight differently*, not a reason to fight less.

## 12. Summary (10 lines)

1. CORE: the world is a drawing that stays real only while watched; the Draftsman is gone, blank paper (the Hollow) eats in from the margins, and you are the last Watchman whose lamps keep the page inked, coloured and boiling.
2. The render states already shipped are the world's physics: line = existence, boil = life, colour = memory, paper = erasure, panels = the Chronicle's attention; spawn ink-in and death scrub-out are birth and erasure.
3. Land has three states the renderer already draws (COLOURED → LINED → BLANK, via the district crossfade on the CLARITY/colour/paper dials); the rejected cream look becomes the look of dying land.
4. STRONGEST IDEA 1 — Lamps ink the land and your style rank is a force: lighting a lamp re-bakes `edgePlaneTex` with a new hole and props ink in; on blank land a circle of colour follows you whose radius is your style rank, so fighting well literally keeps the world drawn.
5. STRONGEST IDEA 2 — Eight hands on one page: the ~40 units split by their own accent colours into Watch, Gilt (varnish the world still; their sheet does not boil), Underdrawn (skeletons are the construction lines of erased people), Greenhand Warband (another artist's page; the drum is their boil), Blot, Hatch, Redrafters (allegiance is ink; the mesmer's charm) and the Hollow, all as HVU team ids with standings; skirmishes are the demo's INK vs RED battles, and the side you hit first is your enemy.
6. STRONGEST IDEA 3 — The two bosses are the story: the Warden is Ashridge's cracked inkwell wearing a dead Order warden's title (killing it lets lamps burn beyond town); the Hollow Knight (undead sheet, your chain and dash) is the previous Watchman, and the companion archer (same sheet) is his surviving archer-study; the finale is erase him or seal him with a sigil loop.
7. HOLLOWED is one new AFFIX rendering any unit as its line-only contour, giving the Margin 40 enemy kinds; THE KNIFE / BLADE / LANCER / WATCH are the Signed, one per faction and region, beaten for their PASSIVE as a perk and recruitable as team-0 companions.
8. The open world is a Folio of town-sized sheets joined by page-turn travel (the portal plate + page-tilt camera); regions: Ashridge hub, Lamplit Rows, Vellum Abbey, Palimpsest, Brushlands, Hatchwood, the Spill, Tracing Tower, the Margin, and the Gutter before the pixel "Reference" (SKETCH OFF's look, borrowed once).
9. Ashridge Junction is the hub (Lamp Board map, Record wall, Chronicle journal made from your own captured panels, Roster bestiary, Scribe), and its night sieges are the shipped wave game unchanged.
10. Four chapters (The Well, The Lamps, The Varnish and the Trace, The Hollow) end on the shipped string "THE VIGIL IS KEPT · ENDLESS"; the ledger in §11 marks each idea S/M/L on named systems; the only large new system is per-sheet world building.
