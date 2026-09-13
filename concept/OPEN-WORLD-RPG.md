# HOLLOW VIGIL — THE OPEN-WORLD RPG

*Lead design synthesis, 2026-09-13. Built from four independent lenses (world, systems, experience, tech: `hv-work/concept/lens-*.md`) and checked against the game sources (`hv-work/src/`) and the units module. It is a concept, not a spec: every major idea names the existing system it grows from, so it can be built.*

---

## 1. The pitch

**Hollow Vigil is an open-world action RPG set inside a living drawing that is being erased: you are the last Watchman, you win the page back one lamp at a time, and every enemy you learn to beat can be spared and drawn onto your side.**

Why it works. The game already has three things most action RPGs never get: fights that feel superb (real hitstop, parries, executions, a domain ultimate), a roster of about forty enemies that each have a personality (a move set, a passive, a named once-per-life ultimate), and a team AI where any of them can fight any other. An open world is the obvious place for all three. The danger is the usual one: a big, flat map full of trash fights that wears the combat out. This design prevents that three ways. The world is small and dense. You always walk into fights that are already happening. And you never get stronger by grinding. You get stronger by *understanding* the things you fight.

Why it could only be this game. Every other game that "looks like a sketch" uses the look as a costume. Here the look is the physics of the world. The ink line boils ten times a second because the world is alive and being watched. Bodies are drawn in when they spawn and scrubbed out when they die, because in this world that is what birth and death are. Colour is laid in only where the world is kept. And the enemy of everything is not darkness but **blank paper**: the Hollow, which does not destroy things, it un-draws them. The pencil-and-ink pass, the animation on twos, the manga panels and the killcam are not decoration on top of an RPG. They are the rules of the RPG, and the player can read every rule off the screen without a codex.

---

## 2. The spine

Four designers were each given the same brief and wrote without seeing each other's work. Three of them independently arrived at the same world, and every lens that covered progression arrived at the same verbs. That kind of convergence is the strongest signal a design can get, so it is the spine of this document and nothing below is allowed to dilute it:

1. **The world's drawing is being erased, and lamps keep it drawn.** You light lamps. Lit lamps push the blank back, and the land around them is drawn and coloured in. *Restored colour is the progress bar.* (World: "lamps keep the page inked, coloured and boiling". Systems: "full-saturation colour under the ink is earned back". Experience: "the first thing the player learns, without a word, is that colour comes from lamps. That is the whole game.")
2. **SPARE is the core verb.** At the execution prompt you choose: finish the body, or spare it. Sparing is how you recruit, how you learn, and how the story's rivals come back into your life. (World: recruitable Signed and the SEAL. Systems: SPARE → BIND → CALL. Experience: "execute or spare… he'll show up later".)
3. **In a faction clash, your first blow picks your side.** Factions fight each other across the world on the units' real team AI. You arrive neutral. The side you strike becomes your enemy, and the other side fights beside you. (World: "the side you strike first is the side you are against". Experience: "your first damaging blow picks the side". Tech: a relations table replacing `team !== team`.)
4. **No character levels.** Progress is knowledge, kit, allies and territory, never a bigger damage number. (Systems: "you don't level up, you learn what the world does to you". Experience: "no character levels". World: perks and study sheets.)
5. **The wave game survives whole** as a content type inside the world. All four lenses kept it, and so does this design.

The tech lens added the constraint that makes the other four buildable: **the existing game stays a small arena, and the arena moves with the player.** Only the bodies near you are real. The rest of the world is data.

---

## 3. The look is the world's physics

### 3.1 The six forces

Each force is something the renderer or the sim already does. The fiction only gives it a name.

| Force | What it is in the world | What the player sees | Grows from |
|---|---|---|---|
| **THE LINE** | Existence. A thing is real because it is outlined. | The ink edge on every body, roof and tree. | The post pass `SKETCH_FS`, baked outlines, `ringStack`. |
| **THE BOIL** | Life. Watching re-inks the line ten times a second. A line that stops boiling is dead or held. | The 10 Hz wobble. | The one `BOIL` clock (shader seed, outline row `OV`, ink-canvas wobble), held during hitstop. |
| **COLOUR** | Care. Colour is laid in where the world is kept: lived in, lit, defended. | Full-saturation colour under the ink. The resting look of the whole game. | `FLOOR_TINT`, `COLOUR_SAT`, `CLARITY`, the 10 Sep "proper colour" pass. |
| **INK** | The substance of being. It spills when things die, pools in wells, burns in lamps, and runs loose as slimes. | Kill puddles, splats, slimes, oil slicks. | Decals (`15b`, `DECAL_LIFE` "a RECORD"), slimes, `slickAdd`. |
| **PAPER** | The Hollow: not darkness but blankness. It un-draws. | Clean blank page with a few construction lines, *eating in at the edges*. | `edgePlaneTex()`, the hand-cut window around the town (12-renderer-textures). |
| **THE PANEL** | Attention. When something matters, the Chronicle draws it as a panel, and being drawn in a panel makes a moment permanent. | Cut-ins, killcam, letterbox, results card, style rank. | `renderPanel`, `panel()` priorities, killcam, `STYLE` D→SS, the marks wall. |

### 3.2 The colour rule (non-negotiable)

The user has rejected the white/cream paper look three times ("the entire world is white", "too faded, dirty, hard to see, muddy", "I want proper color"). So:

- **Full-saturation colour under crisp ink is the default and the restored state of the world.** It is what a player sees for the overwhelming majority of play: a green field, red and blue roofs, and the ink line on top, exactly as the game looks after checklist item 0.
- **There is no "faded" middle state.** No region is ever desaturated, greyed or washed toward cream as a resting look. Land is either drawn (in full colour) or erased (blank paper). The change between them is spatial and hard-edged, never a slow global fade. The erasure front is a ragged torn edge that moves, like the hand-cut window `edgePlaneTex` already draws, not a filter over the screen.
- **Blank paper is a threat you look at, not a place you live.** You see it as a tear on the horizon, a bite taken out of a field, a gap where a house used to be. When you step into it, the colour comes with you (§3.4). The paper is clean and bright with a few construction lines, never grey and never dirty, so it reads as *absence* rather than mud.
- **Screen budget:** outside the endgame Margin, blank paper never covers more than about a quarter of a combat frame. Inside the Margin, the watched circle guarantees colour around the player at every rank. No stretch of play longer than a boss phase is spent looking mostly white.
- **The `mono` (graphite) uniform keeps its one current meaning:** the domain finale and the pause page. It never marks territory.

### 3.3 Birth and erasure

- **Spawning is being drawn.** The three-stage ink-in (the line alone, then the pencil under-stroke, then colour and shadow; 15-effects) is what happens to a person arriving on the page. Held on twos, it takes under half a second.
- **Death is being scrubbed out.** The kind-specific death (`deathKind`: pop, crack, scribble, collapse) is erasure, and NPCs talk about "being scrubbed".
- **A lit lamp draws the land around it.** Re-baking the edge-plane texture with one more ragged `destination-out` hole around the lamp; the props inside the hole play the same three-drawing ink-in: fences, then houses, then trees, drawn into existence on the 10 Hz clock. This is the open world's core reward animation. It costs one texture re-bake plus the ink-in the props already have.
- **An untended lamp gutters.** Over in-game days its circle shrinks. The props at the edge scrub out and stop colliding (`c.col.r=-1`, the trick broken barrels already use). The land does not fade grey on the way: it is coloured until the torn edge reaches it.

### 3.4 The watched circle: style rank as a force

Being watched makes things real, and the Chronicle's attention is your style rank. **On blank paper, a circle of colour and line follows you, and its radius is your rank:** about 6 tiles at D (a lamp-sized pool that still holds your whole melee), the full camera frame at SS, where props ink in as you fight past them. Fight badly and the paper closes in. Fight well and you carry a moving lamp.

This is the single highest-value new mechanic in the design. It turns the style rank, which in a wave game is a score, into the thing that keeps the world drawn around you. It gives skilled players a reason to push into the Margin early. And it enforces the colour rule by construction: the frame around the player is never white. It is built on `STYLE.rank` (already read by the pickup magnet in 21-waves) and a radial term in the post pass keyed on the player's screen point.

### 3.5 The regions are drawn in different hands

Regions differ in *how they are drawn*, which is dial work on systems that already exist: line weight (`INK_D`, rim thickness), hatch in true shadow only (`SHADOW_HATCH`, `SHADOW_LO`), boil rate (held or quickened), saturation at or *above* the default and never below it, and fog (`applyFog`). One row per sheet, no new shader. The hands are listed per region in §4.6; the two most striking are the Brushlands' fat wet brush with a faster boil near the war drums, and Vellum Abbey, where gold rims every outline and **the boil is held for the whole sheet** (the same hold hitstop uses): silent, gleaming, still in full colour, and deeply wrong.

### 3.6 Two places that break the rules, once each

- **The back of the page.** A few dog-eared corners in the world can be struck with Iron Fall. The page flips (page tilt at full strength) and you land on its reverse: the drawing held reversed, using the shader's existing `invert` impact-frame path as a place. It is dark, not white. These are the game's rarest secrets, a few rooms each, and the deepest one is the Hollow Knight's lair.
- **The Reference.** The finale's last room shows the pixel world with the sketch pass off: the Draftsman's model, unwatched and perfectly still. It borrows the SKETCH OFF look for one scene. SKETCH OFF itself stays an accessibility and performance setting and never gates content.

---

## 4. The world

### 4.1 Premise

The world is a drawing that stays real only while it is watched. The Draftsman who drew it has put down the pen. Blank paper, the Hollow, is eating in from the margins. The Watch of Ashridge is an old order sworn to keep lamps burning along the roads, because a lamp is a thing that watches when no person can. You are the last Watchman of that Vigil.

The title means two things, and the game lets both be true. It is *a vigil kept over a hollow*: you stand at the edge of an emptiness so it does not spread. And it is *a vigil that is hollow*: nobody knows whether the Draftsman is coming back, so keeping watch may be pointless. The ending does not answer that. It lets you keep watching anyway. The shipped string `THE VIGIL IS KEPT · ENDLESS` (21-waves `onBossDown`) becomes the last line of the story.

**Who you are.** The knight and the archer are **two studies of the same figure**: an artist drawing one person twice on one sheet. The mid-fight swap is turning to the other study, which is why it is instant and why both share one health bar and one rank. **The last stand** ("PRESS J TO RISE") is the Vigil redrawing you once. **The domain ultimate** (IRON VIGIL / HOLLOW VOLLEY, enemies at 0.42× time inside) is the Watchman holding the page still on purpose: short, costly, and the one time you do what the Order wants to do forever. **The abilities are Watch craft drawn in lamp-ink:** SKETCH SIGIL closes a drawn loop, SCRIBBLE DOUBLE is a quick gesture drawing of yourself that enemies believe is real.

**The companion archer** is drawn from the skeleton family (after the 9 Sep fix, she and the archer class share that family only). She is the archer-study of the previous Watchman: the half of him that did not go hollow. Her quiet kill barks ("heh", "got one", 22-companion) are the voice of someone who has done this before. Who she was is the spine of the story.

### 4.2 The factions: team ids with a standing

The units module lets any unit fight for or against anyone (`e.team`, with the player on team 0). A faction is therefore a set of team ids plus a **standing** toward the player (−3 to +3) and toward each other. The tech lens's one structural change makes this real: the ~15 places in the module that test `u.team !== e.team` become `hostile(a,b)`, a lookup in a relations table. Membership follows the units' own accent colours, so a faction reads at a glance in a crowd.

| Faction | Units (accent) | What they want | Territory | How they fight |
|---|---|---|---|---|
| **THE WATCH OF ASHRIDGE** | THE WATCH (the rank-and-file guard, a squad unit), THE CAPTAIN (`#9aa5b1` grey), knight and axeman (`#8fa8c8` steel), pikeman, swordsman, archer | Keep the lamps lit and push the blank back one lamp at a time. | Ashridge and every roadside lamp. | A formation faction: SHIELD WALL, STANDARD, SET SPEAR, STEADY AIM. Strongest beside you. Too few and too tired: lamps you don't help defend fall. |
| **THE GILT ORDER** | priest, templar, THE HIEROPHANT (cream to gold `#e8d9a8`→`#d0b050`), THE SENTINEL (the renamed shield-bearer, §9.5) | To *finish* the world: varnish every line so it never moves again. A still line cannot be erased. They are right that it would stop the Hollow, and wrong that anything would still be alive. | Vellum Abbey, gold-bordered land. | Support wall: BLESSING, WARD, HOLY GROUND, SANCTUARY, BULWARK, AEGIS. They make every body near them better, so they ally with whoever is useful. |
| **THE UNDERDRAWN** | skeleton, BONE GUARD, GREATSWORD, BONE ARCHER (bone `#d8d2c4`), THE REVENANT, the necromancer (`#7fbf5a`) | To be drawn again. A skeleton is the construction-line armature left under a person who was scrubbed out but not all the way. Many are the Watch's own dead. | The Palimpsest. | BONE DEEP: a light hit rubs the drawing and it rises again, a heavy one tears the paper. The necromancer traces over armatures with spilled ink (SOUL HARVEST). |
| **THE GREENHAND WARBAND** | orc, ELITE ORC, IRON ORC, rider (brush green `#6a9a3c`), BERSERKER (`#b8412f`), THE WARCHIEF (`#3f5a2a`) | Land that stays drawn. They are figures from *another artist's page*, drawn in thick brush, whose own sheets are going blank. Conquerors because they are refugees. | The Brushlands, a war camp moving across the eastern sheets. | **The drum is their boil.** WARDRUM ("its side swings 15% faster") is how they keep their line alive without lamps. Kill the drummer and their lines go still. The only faction that can survive without the Vigil, which makes them the most valuable ally. |
| **THE REDRAFTERS** | wizard (`#6f7fd8`), THE WARLOCK, THE MESMER (violet `#9a6fd8`), THE HEADHUNTER (slate `#606878`); the game's sigil caster and summoner | The Draftsman's pen. They worked out that allegiance, colour, even a name, are just ink, and ink can be drawn over. | The Tracing Tower. | MASS HYPNOSIS literally re-inks bodies onto their side (`charm()` changes a unit's team). DEATH MARK marks a body for erasure. The antagonists of the middle act. |

Two forces are not factions:

- **THE WILD** — loose ink and shadow, hostile to everyone, political to no one. The Blot (every slime archetype, bombers, slimelets, THE PLAGUEBEARER) is ink with no figure to hold it: SPLIT is a blot dividing, PLAGUE is ink bleeding through paper. The Hatch (werewolf, werebear, bats) are drawn only in cross-hatch, in true shadow between lamps, which is the shipped rule that hatch appears only in shadow. The Wild is the third party that turns any two-sided clash into a three-way one.
- **THE HOLLOW** — an absence with no units of its own. It **hollows** units of any faction (§4.4).

**Default relations.** The Watch and the Order are uneasy allies. The Order crusades against the Underdrawn. The Warband is at war with the Watch over the eastern lamps and despises the Order (a varnished world is a drumless one). The Redrafters use everyone: they pay THE KNIFE to snuff lamps, feed the Hatch, and sell charms to the Warband. Everything is one team assignment at spawn plus the relations table, and a three-way battle is already legal in the module.

**Standing.** At +2 a faction's units spawn on your side on shared sheets and its trader opens in Ashridge. At −2 its patrols hunt you (`opts.hunt`). Rallying passives become politics you can feel: WARCRY, STANDARD, HOLY GROUND and WARDRUM buff *their team*, so side with the Warband and the drum speeds up *your* swings.

### 4.3 The Signed

The three named rogues, **THE KNIFE, THE BLADE and THE LANCER**, share one accent in the units data, teal `#8fd3d0`, and no one else wears it. The fiction makes that literal: **teal is the colour of a signature.** A figure who signs itself onto the page cannot be quietly erased, because a signature is the one mark the Hollow respects. Each was once a study-figure of the Vigil who took a name and chose a side. They are the story's rivals, the best move sets in the module, and the prize recruits.

| Signed | Kit (units module) | Sworn to | Wants |
|---|---|---|---|
| **THE KNIFE** | backstab, launcher, plunge; ULT SHADOWSTEP (three cutting dashes) | the Redrafters, paid in lamp-ink | a name nobody can trace over |
| **THE BLADE** | tempo, rising slash into slam; ULT ASTRAL TEMPEST | the Gilt Order, as its sworn champion | to be varnished: to fight forever as their perfect figure |
| **THE LANCER** | long thrusts, launcher; ULT SKYFALL spear throws | the Warband, riding with its riders | a page that will not be erased under them |

A Signed is met several times. They ambush, flee at their ULT's health threshold, and **remember** (§5.6). They are beaten in a duel (boss camera, boss music, `bossNameEl`) and can then be spared, which is the only way to recruit them.

### 4.4 The Hollowed

Any unit that stands too long on blank paper becomes **HOLLOWED**: its colour and fill are gone, and only the boiling ink contour walks. It keeps its whole move set, passive and ultimate, and joins the Hollow, which fights everyone.

- **Look:** the first held drawing of the spawn ink-in, which is the line-only contour ghosts already use (`ringStack` / `sheetOutlPlain`). It must never be `sheetHalo`, which prints a black silhouette.
- **Rules:** one new entry in `AFFIX` beside ARMORED, SWIFT, VOLATILE, VAMPIRIC, HEXED and WARDED, so the elite under-stroke, `eliteSeen()` shouts ("HOLLOWED TEMPLAR") and the budget all apply. It gives and takes no hitstop (the Hollow does not boil), cannot be healed, and leaves no kill puddle when it dies: it simply stops being drawn.
- **Why it matters:** the Margin gets a forty-kind bestiary for the cost of one affix and one render state it already has.

### 4.5 The shape of the world: the Folio

**The world is the Folio, a bound sketchbook of sheets.** Each sheet is one continuous playable page. Sheets are joined edge to edge on a grid, and crossing an edge turns the page: the travel plate (`portalSim`, which already swaps the world behind a plate) plus the page-tilt camera beat (the 23-camera `PAGE` layer). Walk into an erased edge and you meet paper, and a construction line scratches under your feet.

- **Size of a sheet:** about 128 × 128 tiles, roughly 2.5× the width of today's town (`T.BOUNDS` is 52 × 68). At the knight's walk speed of 9 tiles a second a straight dash across takes 14 s; walked along roads, with a look at something and one encounter, a crossing takes 60–90 s. That is the experience lens's pacing target, and it is small enough that a whole sheet's props and colliders load at once behind the page turn. **The only thing that streams inside a sheet is people** (§8.2).
- **Count:** **12 sheets across 8 regions**, laid out 4 × 3, plus the hidden backs of pages and the finale. Crossing the whole Folio on foot takes 12–15 minutes, and nobody ever has to.
- **Each sheet has one Great Lamp and 3–5 roadside lamps.** A roadside lamp is lit by standing at it for 1.5 s (`lampRelight`'s stand-still timer). It becomes a respawn point, a fast-travel page turn, and a refill for your last stand. A Great Lamp is lit by keeping a **Vigil** at it (§5.5). That draws in the sheet's erased bites, turns its region toward you, and opens the sheet's lord.
- **The map is the drawing.** The pause page shows the Folio as a spread of sheet thumbnails, each one that sheet's current edge-plane canvas drawn small. You watch your map ink in.

### 4.6 The regions

"Tier" is the CFG `unlock` field (1 = slime, 10 = warchief), which becomes the earliest region a kind may appear in: the same `(c.unlock||1)>n` test `composeWave` already runs, with `n` now the region tier instead of the wave number.

| Region (sheets) | Hand | Who | The conflict | Signature play |
|---|---|---|---|---|
| **Ashridge Junction** (1) — the hub | Exactly today's town: green field, red and blue roofs, eight lamps. The most coloured place in the world. | The Watch; traders of every friendly faction. | The town is a crossroads, and every road out is a different faction's border. | The first Vigil, the shipped wave game untouched. The Well under the North Gate. |
| **The Lamplit Rows** (1) — shipped district 2 | Narrow streets. Deep night-blue shadow between lamps, hatch in true shadow only, bright colour pools under every lamp. Forced `nightfall`. | The Watch's lamplighters, the Hatch, THE KNIFE, bats. | Someone is snuffing the Rows' lamps one by one, and werewolves rise in the dark they leave. | Defend-the-lamps (`lampObj` {hp, lit}, `bindLamps`). Lighting a lamp turns a werewolf in its circle back into a frightened citizen. |
| **Vellum Abbey** (1) | Gold rims, full colour, **the boil held**. Everything boils only when struck (the +17 boil bump impact frames already get). | The Order, THE BLADE, hollowed pilgrims at the gate. | The Order is preparing the Varnish: a rite to hold every line still forever. They need your domain to prove it works. | A fight on a still page. Lord: THE HIEROPHANT. |
| **The Palimpsest** (2) | Full colour with ghost double-lines of what used to be there; heavy decals. | The Underdrawn, necromancers, Order crusaders. | A three-way war: the Order burns armatures, necromancers raise them, the armatures want to be people again. | **The ground remembers:** kill puddles persist across visits and `ncSummon` raises bodies at them, so your old battles rise to fight you. Seal armatures (§6.3). Lord: THE REVENANT. |
| **The Brushlands** (2) | Fat ink, flat washes, long grass, palisades, taiko foregrounded. | The Warband, THE LANCER, Watch axemen who deserted. | The Warband holds the eastern lamps and burns the lamp-ink as war paint, while their home sheets past the far margin go blank. | **Silence the drum**, or win standing and fight *with* it. Lord: THE WARCHIEF. |
| **The Hatchwood** (1) | The densest trees on any sheet, deep saturated greens and blues, heavy hatch in the shadows. `fog` modifier. | Werewolves, werebears, bats; a Redrafter camp breeding the pack. | The Hatch multiply as lamps go out everywhere. | **Carry light:** an escorted lamp-bearer is a moving colour circle that strips FRENZY stacks. Oil barrels burn thickets open. |
| **The Spill** (1) | Oversaturated, rainbow washes, drips and slicks. The only place with more colour than Ashridge, and all of it loose. | Every slime archetype, bombers, THE PLAGUEBEARER, necromancers harvesting ink. | The Well's crack still leaks downhill, and everyone wants the ink. | Ink is loot: the best ink pools in PLAGUE zones. Lord: THE PLAGUEBEARER. |
| **The Tracing Tower** (1) | Every prop drawn twice, the copy faint and offset, boiling out of phase. | The Redrafters; charmed bodies of every faction. | They are tracing the Draftsman's hand from the oldest lines in the Folio. | **Allegiance is ink:** MASS HYPNOSIS flips bodies back and forth all fight long. Lords: THE MESMER, then THE WARLOCK. |
| **The Margin** (2) — shipped district 3 | The most erased land. Blank bites everywhere, lit lamps as islands of colour, and your watched circle. Forced `frenzy`. | The Hollowed of every faction, and the Hollow Knight walking the road. | It grows whenever any region's lamps are untended. | Fight well or be erased. A lamp here costs the most ink and draws the biggest hole. |

### 4.7 The hub: Ashridge Junction

The town you have now, rebuilt building by building with ink. Each project visibly inks itself in.

- **The Watch-house:** the Lamp Board (the Folio map), the bounty board (contracts on Crowned elites), and **the Chronicle**, your quest log, assembled from panels of your own play. When a story beat fires, `renderPanel` already renders the world into a panel texture, and the log keeps a small copy of it.
- **The Scribe:** equip techniques, traits and charms; rank techniques. Free respec at any lit lamp.
- **The Smithy:** Nibs and Inks (§6.5).
- **The Chapel:** last-stand vows and flask size.
- **The Barracks:** the followers you have spared and bound.
- **The Record** (the square's wall): the marks wall (16-marks), grown with region and faction marks, beside the Roster (§6.1).

### 4.8 The two bosses

**THE WARDEN — the Blot that wears a title.** Ashridge's great inkwell, the Well, holds the ink every lamp burns. A generation ago the Order posted a Sentinel as *Warden of the Well*. The Well cracked, the Warden went in after the ink, and the ink kept the title. The boss is the game's giant rainbow slime (`warden` in 10-config, three phases, the triple charge, the slam that throws kittens): it still guards, it still charges like a shield-bearer, and it throws out slimes because a blot splits. Its IT REELS / IT RAGES beats stay. A fourth beat is added in the world: the drowned Sentinel steps out of the ink as a short second fight. Seal it and the Order's standing jumps: you gave them back their Warden. **Killing the Warden returns the Well's ink to the lamps**, so lamps can burn beyond the town. The gold ring that shipped on both gates (`THE GATE OPENS`) now means exactly that.

**THE HOLLOW KNIGHT — the Watchman before you.** Drawn from the skeleton family (`hollow`, sheet `undead`, tinted), with the knight's chain, a gap-closing dash that leaves afterimages, and a fan of bolts at range: both of your studies in one body. He kept the Vigil before you. He believed the Order, held his domain over the whole page, and when he let go, the page had forgotten him. His knight-study went hollow. His archer-study stayed behind and became your companion. **He is seen long before he is fought:** from Chapter I, a line-only figure at the edge of blank land, dashing with afterimages, and the companion's barks go silent when he is near (her kill barks are suppressed). He is fought in his lair on the back of the last sheet (`EBRAIN.hollow`, mark FACE TO FACE), where the page starts half erased and every lamp you light mid-fight draws colour back in. At the end you choose.

### 4.9 The arc: four chapters

Every chapter ends on a results card that is a Chronicle page: the existing card, retitled.

- **CHAPTER I — THE WELL.** Ashridge is the only coloured place left nearby, and its lamps are burning their last ink. The first night is the shipped game: waves in the town, the Warden wave, the gate. You go down the North Gate into the Well, kill the Warden, and seal or kill the drowned Sentinel. The ink flows back. First sight of a line-only knight at the edge of the paper, and the companion stops talking.
- **CHAPTER II — THE LAMPS.** The Folio opens. Relight it in any order: the Knife in the Rows, the Blade at the Abbey, the Lancer in the Brushlands, the Old Watch Barracks in the Palimpsest. Every Signed you beat or spare, every Great Lamp you light, pushes the blank back and moves the standings. Midway, sealing the barracks' skeletons (they wear the Watch accent) reveals that they were the Hollow Knight's company and that the companion was his archer-study.
- **CHAPTER III — THE VARNISH AND THE TRACE.** Two plans to save the world, both terrible, come to a head at once. The Order will perform the Varnish at the Abbey; the Redrafters will trace the Draftsman's hand at the Tower. The Order needs your domain. The Tower needs your companion, the only figure drawn by the Draftsman's own hand in two studies, and the Mesmer takes her with MASS HYPNOSIS. You choose which rite to break first. The other partly succeeds, and its sheets stay varnished-still or traced-and-charmed for the rest of the game.
- **CHAPTER IV — THE HOLLOW.** You go into the Margin with whoever will come: your spared Signed, and allied factions whose units now spawn beside you (the Warband arrives with the drum if you gave them a sheet of their own). The Hollow Knight waits on the back of the last page. At the end, **ERASE** him, and the companion leaves you: you are the Watchman now, alone. Or **SEAL** him with a sigil loop while he is stunned, and the knight-study and archer-study redraw into one figure who walks past the last panel and does not come back.
- **THE GUTTER.** The white strip between two comic panels, with the letterbox bars as its walls, is a corridor thirty seconds long. Past it is the Reference: pixel Ashridge with no ink, no boil and no colour wash, perfectly still, and no Draftsman. The last card reads **THE VIGIL IS KEPT · ENDLESS**. Post-game is the Endless Vigil (§5.5).

---

## 5. Playing it

### 5.1 The first 10 minutes

- **0:00 — The opening image.** A pen draws one lamp post, stroke by stroke on the 10 Hz clock (checklist item 15, "title drawn stroke by stroke"). Then the knight, asleep against it. Then the roofs fan outward, ink first and full colour a beat behind each one, and the field floods green from the lamp. HOLLOW VIGIL is hand-lettered across the sky, holds a second, and tears away. About 12 seconds, skippable, and control arrives before the strip has gone. The first thing the player learns, without a word, is that **colour comes from lamps**.
- **0:15 — Walk.** Two WATCH guards chat in speech balloons that never pause you: *"Third lamp out this week."* *"Blame the Knife."* Each control appears once, as a small paper tag beside the knight, the first time it matters.
- **1:00 — The first fight.** Six slimes spill into the lamp square: scripted waves 1–2, which already teach exactly the right things. But the two guards draw swords and fight **beside** you. In one minute the player sees that units fight units, earns a FROM BEHIND!, and watches the rank letter climb. The last slime gets the first killcam.
- **2:30 — The hook.** A shield-bearer teaches guard-break. Meanwhile a teal figure crosses the rooftops: **THE KNIFE**, an eyes-in-a-slit cut-in for 0.6 s. He snuffs the lamp you woke under. The town does not go white. Instead, at the far end of the square, **the paper tears in**: a ragged blank bite opens past the fence, construction lines where a woodpile was, crawling toward the lamp. The colour is still all around you. What was taken is the thing protecting it.
- **4:00 — The chase.** A guided run through three streets. The Knife throws a slime pot, tips two skeletons off a cart, then does SHADOWSTEP through you at a point the script picks, and the player learns to dash through a telegraph. He escapes over the gate at 50% health, his ULT threshold. The player wants him.
- **6:00 — Relight.** Back at the square, standing at the dead lamp for 1.5 s relights it. The torn bite inks back in, fence, woodpile, grass, in three held drawings. That is the whole loop in miniature.
- **7:00 — The gate.** The town gate is the portal the game already has, but the first time through it is a vista. The camera eases its pitch down to the hero angle, a slow dolly over 2.5 s lets the valley rush away while the knight stays the same size, and "THE VIGIL VALE" is lettered in a thin bottom bar. **The valley is in full colour**, and three things stand out in it: a dark lamp tower to the north with no flame, a column of smoke and drums to the east (the Warband), and on the southern horizon **a white tear in the world**, clean paper where the land should be. Control never stopped.
- **8:30 — The first clash.** On the road, a Watch rider and two pikemen are fighting four skeletons. Neither side has attacked you, and a hand-lettered tag floats over the fight: **"WHOSE SIDE?"** Almost everyone hits the skeletons, and that is fine: it is the tutorial for the game's central verb. The rider salutes: *"Tower's that way. The Knife went there too."*
- **9:30 — The first Vigil begins** at the tower (§5.5).

### 5.2 The first hour

| Time | Beat | What the player learns |
|---|---|---|
| 10–15 min | A three-wave Vigil at the tower: the existing wave banner, choice cards and budgeted rosters, relocated. Light it and colour floods back into two bites of the valley; one card from the night is kept as a Charm. | Vigils claim land. The run's cards are this night's; one page stays. |
| 15–25 min | Free roam of the first sheet. Two roaming skirmishes, a cracked stone that Iron Fall opens, wanted posters for THE KNIFE, THE BLADE and a blank helmet (the Warden). | Posters are quests. Abilities are keys. |
| ~25 min | The companion is found pinned down in a mill by bombers. Saved, she joins and hands you her spare bow: **the class swap unlocks**. The next fight is staged with a shield wall and a sigil caster behind it, so shooting over the wall is a discovery. | Two studies, one figure. |
| ~30 min | The first stunned orc you have *answered* every move of kneels at the execute prompt with a second option: **SPARE**. | The execution is a choice. |
| ~35 min | **First faction clash:** 8 Watch against 8 Warband at a river bridge, THE CAPTAIN on one side and an ELITE ORC on the other, opened with a split-panel cut-in. | The world has politics, played with a sword. |
| ~45 min | **The Blade** steps out of a treeline at dusk: a letterbox slit and a 1-v-1. At 60% he fires ASTRAL TEMPEST. He flees, and remembers. | Named rogues are people. |
| ~55 min | From a ridge, the Well's broken lid below the North Gate, and something huge moving in the ink. A vertigo dolly, the poster inks in, and you are not asked to fight it yet. | Some things are bigger than you. |

At the end of hour one: two lamps lit and a Great Lamp claimed, two Roster pages studied, one follower, one rival who hates you, one faction that likes you, and a silhouette you are afraid of.

### 5.3 A session at hour 15

The player has lit 7 of 12 Great Lamps and most roadside lamps between them. The Watch are allies. The Warband is split: the player spared THE LANCER. The Hollowed hold the south.

- **0:00 — Open the book.** The game resumes on the map spread. Since last session the Warband's hatch has crept over the Old Mill and a new blank bite has opened in the Palimpsest. Three torn notes: a bounty (*"TAKE THE MILL BACK — the Watch pay in ink"*), a rival sighting (*"THE KNIFE seen at Crowfoot"*), a style contract (*"FIVE EXECUTIONS in one clash"*). Twenty seconds on the map, then a page turn to the nearest lamp.
- **1:00 — The walk is not dead time.** Drums from the left before anything is visible. A crow doodle sketches itself on the right edge of the screen; following it finds a dog-eared corner, and Iron Fall flips it to a two-room back of the page with a study sheet and a note in the Hollow Knight's hand. A bat swarm and a plaguebearer at dusk: swap to the archer, one quickdraw, one ricochet, forty seconds.
- **5:00 — The mill.** Six Watch holding against twelve Warband with an ELITE ORC and a BERSERKER. The player comes in from behind the orcs. The elite orc hits 50% and roars WARCRY; the page kicks. The player answers with IRON VIGIL: **the domain freezes only hostile teams**, so the Watch keep swinging inside the frozen circle. Killcam on the elite orc, the leader, not on the last body. The results card reads "THE MILL — RETAKEN — S". The Lancer, the player's follower, fires SKYFALL as a CALL at A rank.
- **12:00 — A Vigil in the Palimpsest.** The Great Lamp there is under siege, because the player let it gutter. Six waves on region-tier rosters, FOG then SPITTERS, an affixed elite hunt, cards after waves 2 and 4, a necromancer who raises the dead at the kill puddles of the player's *last* fight here. At 20% health the last stand fires. They hold it. The bite inks back in.
- **22:00 — The rival.** Crowfoot. THE KNIFE opens with smoke this time, because the player kept parrying his opener. Two minutes later he kneels: execute or spare. The player has spared him once already, and this time he gives up his patron's name.
- **28:00 — Bank at a lamp.** Unbanked ink is written into the lamp, the session page shows today's stamps, and the player stops at a clean end.

Four kinds of play (exploration, clash, Vigil, duel) in 30 minutes, none longer than 10.

### 5.4 Exploration: why walk over that hill

A top-down camera cannot see the horizon, so the world tells you where to look in five ways:

1. **The blank is the invitation.** A white bite in a coloured field itches the way fog of war does, but it is honest: it is a *loss*, and it is yours to undo. The map spread shows every bite.
2. **Marginalia instead of a minimap.** Points of interest outside the frame are doodled into the screen's margin on the side they lie, small and boiling on the shared clock: a flame for an unlit lamp, a smoke curl for a camp, a skull for a lord, a question mark for a rumour, and crossed swords, pulsing, for **a clash happening now**. They grow as you approach and rub out when you arrive. No compass bar, no minimap: the HUD stays clean, which checklist items 1–4 demand.
3. **Sound before sight.** WebAudio played positionally: taiko from a clash, the necromancer's chant, a bell at a lamp. Hear a war 40 tiles off, then see the doodle, then the smoke, then crest the ridge.
4. **Landmarks tall enough to break the frame.** Lamp towers, orc totems and the Abbey spire have tall billboards whose tops poke into view long before their bases. On ridges and bridges the camera pitch lowers slightly. The player learns: walk to high ground to look.
5. **Secrets are ability-shaped, never hard gates on the critical path.** Iron Fall breaks cracked stones and flips dog-ears. The hook crosses gorges to anchor posts. Bomb arrows collapse boarded walls. Ricochet rings two bells in one shot. A sigil drawn in a stone circle opens a barrow. The scribble decoy lures a sleeping werebear off a chest. The crescent wave cuts bramble walls.

**Traversal feel.** Out of combat the run ramps to a jog after 1.5 s; dash, stinger and hook chain into a momentum string, and the style rank does not decay on the road, so arriving at a fight mid-dash with a stinger is the fast and stylish opener.

### 5.5 The Vigil: the wave game's new home

**A Vigil is the shipped wave game, kept whole and placed on a lamp.** Typed pools, `composeWave` budgets, the curve, modifiers, elites with affixes, bounties, the lamp objective, barrels, boss waves and choice cards between waves: all of 21-waves, unchanged, with the rosters drawn from the region's families and the night's forced modifier from the sheet. It has one role: **a Vigil is how land is held.** There are three occasions for it, all the same content type:

- **Claim:** every Great Lamp is lit by keeping a Vigil at it (3–6 waves). This is how a sheet turns toward you.
- **Defend:** a lit Great Lamp you have left untended too long comes under siege, marked on the map. Ignore it and its bites reopen. Nothing is lost for good, but the land visibly frays.
- **Endless:** after the finale, the Margin's Vigil has no dawn: today's Endless mode, with the difficulty curve's uncapped tail and your whole kit behind you. The marks wall and best SS score stay the leaderboard.

**Cards live here.** Cards are run-scoped: they are picked between waves, stack as they do now, and are wiped at dawn by `snapT` / `restoreT`, which were written so runs don't compound. This is the one place the roguelite "build a monster tonight" fantasy lives, and Vigils are tuned to expect it. **At dawn you keep one card from the night** (§6.4). Die in a Vigil and the night's cards are lost, and the lamp stays dark: the stakes of a run survive in the open world.

### 5.6 Faction clashes and rivals

**Clashes.** Roaming squads move along roads and meet each other, so clashes happen on their own where roads cross. The rules:

- **You arrive neutral.** Both sides get a thin accent underline, and nobody targets you (`team = −1`, which the units module already reads as "leave me alone").
- **Your first damaging blow picks the side.** A two-sided cut-in stamps it (*THE WATCH STANDS WITH YOU*). The side you hit becomes hostile; the other side fights beside you for the rest of the fight.
- **One stray hit is forgiven.** A second hit on your own side in the same fight flips it (a "hey!" balloon marks the first). One stray ricochet never starts a war.
- **Or pick neither.** Stand back, let them grind each other down, and clean up the winner: more loot, less standing. The factions notice.
- **Every clash has a leader** with a named ULT (captain, elite orc, necromancer, priest). Its ULT is the climax of the clash, and killing it breaks its side into a rout. The killcam goes on the leader.
- **Standing moves by the outcome.** No dialogue tree: the choice is made with a sword.

**Rivals remember.** Each Signed, and later the Headhunter and the Warchief, keeps a few fields: what beat them last, what you spammed, whether you spared them. They return with one counter from their own move set: the Knife opens with smoke if you always parried, the Lancer throws from range, the Blade fires his ULT earlier.

**The Mesmer is the faction system's villain.** MASS HYPNOSIS flips your allies, including your follower, for 6–8 seconds. A fight you must win without killing your own charmed friend is a quest that costs nothing new to build.

### 5.7 The three stories a player tells a friend

1. **"I walked into a war."** *"I came over a hill and the Watch were holding a bridge against the orcs, like twenty guys. I sided with the orcs because the captain shot me earlier. When he fired LAST STAND and called the volleys, I dropped IRON VIGIL on the bridge. His whole side froze and the orcs kept swinging inside the circle. The killcam went on the captain."* Only possible because units fight units and the domain freezes only hostile teams.
2. **"The Knife remembered me."** *"He ambushed me for hours and started opening with smoke because I always parried him. When I finally had him on his knees I spared him. Ten hours later I'm in the Margin at 5% on my last stand, and he drops off a roof and SHADOWSTEPs through the necromancer for me."*
3. **"I kept the page alive."** *"In the Margin the only colour is a circle around you, and it's as big as your style rank. I got to SS and the whole field drew itself back in around me while I fought, fences, trees, everything. Then I got hit, dropped to B, and watched the paper close back in."*

---

## 6. Progression

**One model, three questions:** *What do you know?* (the Roster), *what do you carry?* (the loadout), and *what have you kept?* (lamps, standing, allies). There are no character levels, no XP bar and no stat points, and no random stat rolls.

### 6.1 The Roster: progress as knowledge

Every unit kind gets a page, grown from the bestiary plan in HVU-map-ui (the marks wall's `.mk` chips, `silhouette()`, the `eliteSeen` first-sight precedent) and filled with the module's best writing: the name, moves, PASSIVE, ULT and ULTDESC lines. A page takes four red stamps (the marks wall's `stampR`):

| Stamp | Earned by | Gives |
|---|---|---|
| **SEEN** | First sight. | Name, silhouette, health line. |
| **STUDIED** | *Answering* every move on its list once: a parry, a perfect dodge, a full guard, or a punish crit in its recovery. Each answered move is inked onto the page ("THE LANCER · 3/5"). | Its PASSIVE sentence is revealed; +8% damage against the kind ("known"); **it can now be spared.** |
| **BESTED** | Executing it three times. | Its technique unlocks at the Scribe, if it teaches one; its drops improve. |
| **BOUND** | Sparing it (or, for a Signed or a lord, beating its duel and sparing it). | It joins the Barracks as a follower. |

Grinding does nothing: a hundred skeleton kills give one BESTED stamp. Parrying the Greatsword's second swing once gives you a line on its page you'll remember. To fill a page you *have* to learn the tell, so the Roster also teaches the combat. **Your Tally**, the count of stamps across the Roster (about 150), is the nearest thing to a level: it offsets enemy weight and nothing else (§7.4).

### 6.2 SPARE, BIND, CALL: the party

**Spare.** An execution already has a stunned telegraph, a full-page splash panel and a ration. Add one branch: **tap attack to execute** (as today: ink, BESTED progress, heal 8, energy 15), or **hold attack for 0.4 s to spare**. The body kneels in the last stand's slow-mo and shade, the splash reads **SPARED**, and after the fight it walks to Ashridge's Barracks. Only STUDIED kinds can be spared: a body must be known before it kneels to you. Bosses are never spared.

**Seal: sparing the erased.** Skeletons, the Hollowed and the Hollow Knight cannot kneel, because there is no one left to kneel. For them the sparing verb is **SEAL**: close a SKETCH SIGIL loop around the stunned body and, instead of damage, it is redrawn as who it was (a watchman, a pilgrim) and walks to the nearest lamp. This reuses the sigil's closed-loop detection and the THE SEAL mark. Spare and Seal are one choice with two gestures: *finish it, or give it back.*

**Bind.** A follower is a units-module body spawned on your team with a leash on you: the companion's leash-and-blink (`ARC.leash`) plus the module's own `pickTarget`, spacing rings, guard, heal and summon logic. Every one of ~40 kits works for free: a priest follower heals you through `woundedAlly`, a necromancer's skeletons fight on your side.

- **In the field:** the companion + **one** follower. **In a Vigil:** two.
- **Each follower adds its threat to the encounter budget** (`ECOST`), so an ally is a different fight, not an easier one.
- **The companion stays outside the team system.** She is the story partner and the MARK executor, with her own small row of upgrades (TWIN VOLLEY and focus-fire speed).
- **A downed follower kneels** and can be revived with a 1.5 s stand, the lamp-relight gesture. Otherwise it walks home and its Call is lost for the region.

**Call.** A follower's named ULTIMATE becomes yours to command, paid for with style. Its pip fills the first time you reach **A** in a fight and again at **SS**. Hold Q (tap Q stays your domain) and it performs its authored ULT through `fx.ult`'s name-card beat: SANCTUARY, CRUSADE, SKYFALL, MASS RISE, RAMPAGE, EARTHSHATTER. The style rank stops being only a score and becomes the button that summons your warchief's RAMPAGE.

**Bond** has three levels: *Bound* (follows and fights), *Sworn* (after enough landed Calls: its aura radius doubles), and *Oathbound* (its PASSIVE can be worn as a trait).

### 6.3 The loadout: two hands

No new classes. The knight and archer, the swap-weave (×1.35 inside 1.2 s) and the recovery handed across on swap cost weeks of feel work, and a third body would dilute them. **The two hands are the class system**, and the loadout fits the HUD that exists:

- **4 techniques:** 2 on the blade hand, 2 on the bow hand (E / R per class, the existing two-slot skill budget). A build is a combo route across a swap: HOOK LINE on the blade, swap, BOMB ARROW point-blank on the yanked body.
- **3 traits:** PASSIVE sentences, worn by you.
- **3 charms:** the perk cards, made permanent (§6.4).
- **1 Law:** a rule for your domain, taken from a lord.
- **2 Nibs and 1 Ink** (§6.5).
- **1 follower.**

**Techniques have teachers.** Every existing ability gets a unit whose move visibly rhymes with it, and BESTED on that page unlocks it: IRON FALL from the Iron Orc's ground slam, INK CYCLONE from the axeman's WHIRLWIND, CRESCENT WAVE from THE BLADE, HOOK LINE from the Headhunter's snare, HOLLOW RAIN from the Captain's volleys, BOMB ARROW from the bomber kittens, SKETCH SIGIL from the sigil caster, SCRIBBLE DOUBLE from THE MESMER. New techniques are built the way 20b built the crescent, from the player's own frames plus ink VFX, with no new player sprites: SET SPEAR (pikeman), SHADOWSTEP (THE KNIFE), NOVA (wizard), SANCTUARY SIGIL (priest), RAISE (necromancer), WARCRY (elite orc). About 14 in all. Ranks I→III follow the perks' three-stack model on the numbers `ABIL` / `AB` already expose. Loadouts swap free at any lit lamp.

**Starting kit, no unlocks:** the knight's 4-hit string, dash, stinger, sweep, parry, riposte and executions; the archer's charged shot, quickdraw and point-blank. The first two teachers are guaranteed on the first sheet, so IRON FALL and HOLLOW RAIN arrive within the first hour, the kit today's run hands out at once.

**Traits wear a body's sentence.** About half the PASSIVE table already speaks the game's crit vocabulary and translates with no new mechanics: BACKSTAB (back crits ×2.5), FIRST BLOOD (first hit on a fresh body ×1.3), STEADY AIM, FLOW, EXECUTION (the execute threshold 34% → 40%), BLOOD RAGE, SECOND WIND (the last stand also fires at 30%), FRENZY, UNSTOPPABLE, DEATH MARK, HEADSHOT, WARD, IRON HIDE. Traits come from Oathbound followers and **directly from beating a Signed** (their passive is the duel's prize, whether or not you spare them). Aura passives (STANDARD, ZEAL, BULWARK, WARDRUM, SHIELD WALL) stay on the follower, where they already work through team scans. Traits make *you* sharper; followers make *the fight* different.

**Laws bend the domain.** Each lord gives one: BULWARK (the Warden: frontal hits halved and bodies inside reel), THE FACE (the Hollow Knight, post-game: a gold ghost of you repeats the teleport finale), WARDRUM (the Warchief), MASS RISE (the Revenant: bodies that die inside rise on your side until it closes), JUDGEMENT (the Hierophant), HARVEST (the Warlock). Built as a `LAW` table and one hook in `ultStep`. Every boss permanently changes your *biggest* moment, not your health bar.

### 6.4 How the roguelite cards survive

The twelve PERKS (KEEN EDGE, IRON SKIN, QUICKFOOT, DEEP WELL, LODESTONE and the rest) live in two forms with the same ids:

- **As cards, inside a Vigil:** exactly as now. Offered between waves, stack to three, wiped at dawn. The run's power fantasy is intact.
- **As charms, outside:** **at dawn you keep one card from the night.** It becomes a charm at rank I; keeping the same card on a later night ranks it to II and III. Three charm slots. Charms are the *only* place flat stats live, so they stay few, capped and familiar.

This gives every Vigil a permanent souvenir, stops cards compounding into invisible numbers, and replays safely on load: `restoreT()`, then each charm's `f()` its rank times, which the tech lens's save rules already require.

### 6.5 Loot that never changes a drawing

A hit here is a drawing: a frame-derived hitbox and fixed wind-up, active and recovery frames. The perks pass already found the limit (LONG REACH was capped at ×1.04 on sweeps). So **gear changes the rules around a hit, never reach, timing or silhouette**, the way an ink artist has only nib, ink and paper:

- **A Nib per hand** rewrites the crit table or the economy for that hand, always with an upside *and* an off-switch. The Flensing Nib: back crits ×2.5, lucky crits off. The Patient Nib: punish window +50%, open crits off. The Long Nib: full-draw pierce 3 → 4, point-blank off.
- **One Ink** tints your ink VFX (the post pass already separates line from colour) and adds a status. **Inks are taken from Crowned elites' affixes:** Vampiric red heals on executions, Volatile orange makes executed bodies burst, Warded gold gives a ward layer on parry. You literally take the Vampiric affix off the thing that kept healing on you.
- **About 40 authored items in total**, each a one-sentence rule, dropping as a torn-leaf pickup through the existing pickup magnet.

### 6.6 The economy and death

- **Ink** (the INK COIN pickup): every kill pays base × the kind's power × `RMUL[rank]` at the moment of the kill, so an SS kill pays double and farming badly pays half. Spent on technique ranks, town projects and lamps in the Margin.
- **Quills** (rare): bounty clauses, Crowned executions, first STUDIED stamps. Spent on charm ranks, vows, Nib and Ink upgrades.
- **Healing stays scarce:** a flask of hearts refilled at lit lamps.
- **The last stand is a Vow** refilled at lamps (extra vows from the Chapel, max 3).
- **Death tears the page:** you wake at the last lit lamp, and your unbanked ink stays where you fell as an ink-pool decal. Walk back and touch it to recover it. **If a Crowned elite killed you, it takes your ink and a second affix** ("THE WARDED VAMPIRIC LANCER", up to three). Your deaths get a face and a grudge.

---

## 7. Combat in an open world

### 7.1 Fights stay the size they were tuned for

The combat was tuned for about 20 tiles of view and about a dozen bodies. The open world never asks it to be anything else.

- **The panel.** When you engage, a faint manga panel border is inked around the fight, about one camera frame wide (roughly 20 × 12 tiles), with the telegraph wobble helpers. It *is* the encounter arena: every `T.CLAMP` read in the spawn code becomes this rect (the tech lens's `ENC.arena`). Hostiles don't chase past it, and return to their post after 3 s with a *"tch"* balloon. Reinforcements wait at its gutter and never pile in off-screen. Walk out and it rubs out: the fight has ended, not been lost. Vigils, lords and duels **lock** it with a heavier line.
- **The awake cap.** At most 24 real bodies near the player (hard ceiling 32). Beyond the panel a clash resolves as numbers, with tally marks in the margin and bodies dying off-frame at a rate, and the front line stays in frame.
- **One attack budget for everything aimed at you.** The module's `TOKENS=2` and the game's `attackCap()` merge into a single count, as HVU-map-ai §9 requires, so stepping into a battle never means being hit by everyone at once.

### 7.2 Feel effects fire only on the player's hits

Today every `impact()` can add hitstop (which scales the entire fixed step), trauma, a killcam, slow-mo, a manga word and a sting. In a world where two factions fight twenty tiles away, that would freeze your screen on hits you didn't throw. **The rule: hitstop, trauma, camera punch, killcam, manga callouts and damage numbers fire only when the attacker or the victim is the player, the companion, or a body currently targeting the player.** Unit-on-unit hits outside that set get damage, a flash, sparks and attenuated sound, and nothing else. It is one predicate in the units host adapter's `fx` calls and one in `impact()`, and it is the difference between a war you walk into and a slideshow. The same rule governs the ink layer: allies get a thin accent underline and no numbers, only your hits print, ally ULTs appear as small balloons rather than stamps, and VFX draw in priority order (your ink, then enemy telegraphs, then ally effects at half strength).

### 7.3 The camera grammar: one fixed meaning per tool

The shot layer already exists (`lensKick` dollies, `pageKick`, the pitch channel and hero angle, the letterbox, the killcam, cut-ins, the domain). In the open world each tool means exactly one thing, so the player learns to read the camera:

| Tool | It always means | Used for |
|---|---|---|
| Slow dolly out + pitch down + thin letterbox | *Look how big this is.* | First arrival at a vista. Control kept. First time full, afterwards just the lettering. |
| Vertigo dolly in | *That is dangerous.* | Discovering a lord or the Hollow Knight. 1.5 s, control kept. |
| Split-panel cut-in | *Choose.* | A clash begins; then the SIDE CHOSEN stamp. |
| Letterbox slit + eyes cut-in | *Someone is hunting you.* | A Signed ambush. |
| Page tilt + page turn | *You are going somewhere else.* | Crossing a sheet edge, fast travel, flipping a dog-ear. |
| Killcam | *That one mattered.* | A clash leader, a Signed's execution or spare, a Vigil's last body, a lord. Never a random kill. |
| The domain | *I end this.* | It freezes only hostile teams; allies keep fighting inside. |
| Colour flood / ink-in | *This is yours now.* | A lamp lit, a bite drawn back, a mill retaken. |
| `mono` graphite | *The finale.* | The domain finale and the pause page, as today. |

**The shot director.** At most one big shot (anything letterboxed or dollied) per 60 s, with priority lord > rival > clash > vista > killcam, first time full and repeats short, and no shot mid-panel except the leader's killcam. The user already finds the current banners overprinted (checklist items 6–8), and an open world multiplies that unless something says no. A setting reduces shots further.

### 7.4 Difficulty is behaviour, not sponge

The existing `curve(n)` returns speed, health, damage, wind-up and attack cap. Split it in two:

- **THREAT** (speed, wind-up, attack cap, affix and modifier chance, whether ULTs and passives are live) comes **only from the region tier.** A tier-8 skeleton winds up fast and uses BONE DEEP however strong you are.
- **WEIGHT** (health, damage) is the region's curve divided by your Tally's, **clamped to 0.8–1.6.** Over-prepared, bodies die in about 20% fewer hits; under-prepared, they take up to 60% more.

Three caps keep the tight verbs meaningful at every stage: **no single enemy hit exceeds 35% of your max health** (lords 50%); **your base damage stays in today's 10–26 range** with growth only through on-screen multipliers under the existing 3× crit ceiling, so damage numbers stay readable; and **the decisive verbs are already percentages** (execute at 34% of max health, stun and poise by hit count, riposte executes from any health, unit ULTs at a health fraction). A harder region feels faster and meaner, not spongier, and going home feels like mastery.

---

## 8. How it gets built

### 8.1 The architecture in plain terms

- **It stays this codebase:** Three.js r128, section files concatenated by `build.sh`, the post pass, the feel package, 21-waves, the units module. No engine change, no framework.
- **The current game becomes a content type.** A Vigil is a hand-authored Place whose `vigil` block runs today's loop unchanged. Ashridge is the first, stamped from data that reproduces today's town exactly, which makes it the regression test for every phase.
- **A sheet loads whole, behind a page turn.** At ~128 × 128 tiles a sheet's props, colliders, lamps and ground all fit in memory at once. `buildTown()` is already data wearing code (the buildings table, prop tuples, the tree ring), so it lifts into per-sheet Place records, hand-authored for towns, camps and arenas, with seeded scatter between them. The page turn's hold simply waits until the sheet is built. Colliders and occluders move into a grid hash so their loops stay cheap (the call sites change only their loop header), and `T.BOUNDS` becomes the sheet's walkable area.
- **Only people stream.** The tech lens's invariant holds: `enemies[]` and the units list only ever contain what is inside the **hot bubble** around the player (promote within ~28 tiles, release beyond ~40). Everyone else is a **squad token** moving along the sheet's road graph at 2 Hz. Tokens become bodies by claiming pooled rigs (`POOL` / `claimBody` / `resetFoe`, generalised from `applyRoster`) and turn back into tokens when you leave. No `makeSprite` after boot. A thin **faction ledger** per region holds strengths and territory so the world changes between visits.
- **Three structural edits to the units module:** `hostile(a,b)` instead of `team !== team`, `host.claim` and `host.park` instead of spawning and splicing, and staggered target picking at 10 Hz.
- **The world layer is new code** in numbered files (`14w-sheets`, `14n-nav`, `21e-encounter`, `21f-factions`, `29-save`), written multi-line from the start so trailing `//` comments can't swallow code, the source of seven live bugs so far.
- **Saves:** one versioned JSON document in IndexedDB; saving only at safe moments (no live encounter, no hitstop), so a load never rebuilds a fight. `resetRun` splits into `resetEncounter`, `onPlayerDeath` and `newGame`.
- **File format:** single HTML through the first shippable sheet, with art families as lazily decoded JSON blocks; multi-file with a service worker once a second region's art arrives, keeping an `--inline` build for sharing as one file.

### 8.2 Reused versus new

| Reused almost as-is | Genuinely new |
|---|---|
| The post pass, boil clock, animation on twos, ink-in / scrub-out, the whole feel package | The involvement gate on feel effects; the shot director |
| 21-waves: pools, `composeWave`, curve, `MODS`, `AFFIX`, `BOUNTIES`, `PERKS` / cards, lamps, boss waves (→ Vigils) | Card-to-charm at dawn; the THREAT / WEIGHT split and hit cap |
| The units module: 40 kits, passives, ULTs, squads, spacing rings, `charm()` | The relations table and standing; follower leash; SPARE branch; CALL input |
| Portals and the travel plate (→ page turns and fast travel) | Sheet loader, grid hash, squad tokens, promote / demote, faction ledger |
| `edgePlaneTex` window (→ erasure and lamp holes), `DISTRICTS` crossfade (→ region hands) | The watched circle radial term; lamp hole re-bake |
| Marks wall, `stampR`, `renderPanel`, results card (→ Roster, Chronicle) | Roster page store and move-answer hooks |
| SKETCH SIGIL loop detection (→ SEAL), `ARC` leash (→ followers), last stand kneel (→ SPARE) | Nav flow field; save system |
| The two bosses, `EBRAIN.hollow`, `onBossDown` | HOLLOWED affix; Law table; Nib / Ink tables |

### 8.3 Roadmap

**Phase 0 — The feel test (prototype, deliberately ugly).** The one question that decides the world's shape: *does Hollow Vigil's combat survive outside a closed box?*
1. **Shared-texture actor material:** one GPU texture per sheet, with the frame window moved into the material. Today every rig clones every sheet it plays, which is the root of the old texture-leak history. This is the enabling change for everything. *Gate:* texture count flat across 20 roster sweeps.
2. **A generic 32-rig pool** with per-sheet anchors, so units draw through the game's sprite path.
3. **One flat 128 × 128 sheet** with Ashridge stamped in the middle from data and seeded trees around it. *Gate:* the Ashridge Vigil plays waves 1–10 exactly as today.
4. **One squad token** (THE WATCH, THE CAPTAIN, THE KNIFE) that promotes and demotes. *Gate:* 50 cycles with no field leakage (a body after `resetFoe` equals a fresh one).
5. **One two-sided clash** (Watch squad vs orc squad) with `hostile()`, first-blow side picking, and the feel involvement gate. *Gate, the one that matters:* **the user plays into it and says the hits still feel like Hollow Vigil, and the screen never freezes on a hit they weren't part of.**
6. **A flow field** for bodies chasing the player. *Gate:* nobody stuck behind a house for more than 2 s across 10 chases.

All performance gates are checked on a real integrated-GPU laptop, not the SwiftShader harness, which can verify state and errors but not look or speed.

**If gate 5 fails, the fallback is already the design.** Each sheet becomes a set of **linked arenas**: Places (a lamp square, a bridge, a camp, a mill) joined by short hand-drawn road corridors and page turns inside the sheet. The Folio, lamps, Vigils, factions, the Roster, spare, the colour rule and the camera grammar are all unchanged. Only in-sheet streaming and the flow field shrink. The player-facing structure (sheets joined by page turns) was chosen so that this fallback costs the fiction nothing.

**Phase 0.5 — The Roster in today's game.** Ship the Roster pages, move-answer hooks and SPARE into the existing wave game before the world exists. It makes the current game deeper immediately and de-risks the progression model on real players.

**Phase 1 — "The First Sheet."** Ashridge plus the Vigil Vale: roadside lamps, one Great Lamp Vigil, the Knife's opening, the companion, one follower with CALL, techniques from teachers, card-to-charm, saves, death and ink recovery, the watched circle and lamp holes. Decor instancing and the density ladder for CPU-bound frames. *Ships as one HTML file.*

**Phase 2 — "The Front."** The Watch, the Warband and the Wild on real relations; standing; a moving front in the Brushlands' first sheet; crowd extras in the fog band; THE LANCER and THE BLADE as rivals with memory; Crowned persistence; the Warden in the Well and the first Law. *Gate:* after 30 minutes and two exits, the front has visibly moved for reasons the player can name.

**Phase 3 — "The Folio."** The remaining sheets and factions, the Hollowed, the Order and the Redrafters, multi-file build, save migrations.

**Phase 4 — The story and the finish.** Chapters III–IV, the Hollow Knight's lair on the back of the page, the Gutter and the Reference, the Endless Vigil, the Chronicle.

### 8.4 The hardest problems

1. **Navigation does not exist.** Everything steers in straight lines today. A flow field for chasers is cheap; units fighting each other among buildings is not. Keep Places open-plan, forests as obstacles with road gaps, and accept units giving up a target.
2. **State leakage on reused bodies.** About 60 per-life fields (`ultUsed`, `charmT`, `team`, a shared `cfg` the berserker mutates). The world re-types bodies hundreds of times a session instead of eleven per wave. A field-snapshot test is mandatory from phase 0.
3. **The feel package is global.** Hitstop scales the whole step; killcam, letterbox and panels are single channels. The involvement gate handles other people's fights, but two player-involved beats in one big clash still compete, so the queue rules need tuning, not just gating.
4. **Every tuning number assumed a closed arena:** aggro radii, the companion leash, spawn clamps, the camera's enemy framing, pickup magnets. Expect a long tail of "felt right in town, wrong on the road".
5. **Keeping the erasure look clean.** Blank bites must read as crisp absence, never as dirt or fade, at every CLARITY setting and on the low-quality ladder rungs. This is an art-direction problem to prove on screenshots in phase 1, with the user.
6. **Legible faction change.** A front that moves while you're away is only a feature if the player can see why. The map spread must show it.
7. **Content volume.** Twelve authored sheets is a lot for a small team. Seeded scatter fills the space between Places; the Places are where authoring time goes.

---

## 9. Decisions

### 9.1 What unlit land looks like

**Decision: full-saturation colour is the default and the restored state. There is no faded, lined or greyed middle state. Erased land is clean blank paper, shown as hard-edged bites and tears that are a threat, and the watched circle guarantees colour around the player inside them.** The user has rejected the white and cream look three times, in words that leave no room ("the entire world is white", "too faded, dirty, hard to see, muddy", "I want proper color"), so the world lens's framing wins: paper is what *erasure* looks like, never what the world looks like at rest. The progress loop survives intact because it was never about saturation. It is about territory: lamps draw blank bites back in, in full colour, with the props inking themselves in. *Rejected:* the experience lens's pencil-on-white valley (the whole unexplored world would be white for hours); the world lens's LINED middle state (it is literally the rejected look, and a region spent "fading" would be a region spent looking muddy); and the systems lens's region-wide `mono` drain (graphite everywhere is the "dirty old look" by another name, and `mono` already means the finale).

### 9.2 World structure

**Decision: the Folio: 12 continuous sheets of about 128 × 128 tiles across 8 regions, joined by page turns, each crossed in 60–90 s of real play. A whole sheet loads behind its page turn; only people stream, in a hot bubble around the player. If the phase-0 feel test fails, each sheet becomes linked arenas inside the same frame.** The world lens's sheets and the tech lens's fallback are the same shape, which is the decisive argument: the player-facing structure is on-theme (a sketchbook has pages) *and* survives the worst technical outcome without changing the fiction, the progression or the camera. Sizing a sheet to the experience lens's 60–90 s crossing keeps it small enough to hold in memory whole, which deletes the hardest parts of the tech lens's plan (chunk streaming order, pop-in at a finished horizon, float precision) while keeping its best part (the invariant that only nearby bodies are real). *Rejected:* a continuous chunk-streamed world across regions (it buys seamlessness nobody asked for at the price of the riskiest engineering); the world lens's town-sized sheets (at 52 × 68 tiles a sheet is crossed in six seconds and page turns would come every minute); the experience lens's 18 regions (more authored content than the density rules can fill).

### 9.3 Progression

**Decision: one model: no levels; knowledge through the Roster (seen → studied → bested → bound); a two-hand loadout of 4 techniques, 3 traits, 3 charms, 1 Law, 2 Nibs, 1 Ink and 1 follower; and territory and standing as the world's record of you. Cards live on inside Vigils, run-scoped as now, and at dawn you keep one as a permanent charm.** The three lenses already agreed on the substance; the merge gives each idea one home. The systems lens's page stamps become the Roster (its "Folio" name is taken by the world). The world lens's "signature becomes a perk card" becomes "beating a Signed gives its PASSIVE as a trait". The experience lens's "no character levels" is enforced by the THREAT / WEIGHT split. Keeping *one* card per dawn gives every night a souvenir without letting cards compound into invisible numbers. *Rejected:* keeping every card permanently with a slot limit (it turns cards into inventory), charms only from taking a card three times in one night (too rare to feel), and buying cards with ink at the Scribe (it deletes the run's surprise).

### 9.4 The wave game's new home

**Decision: it is called a Vigil, and its one role is holding land: every Great Lamp is claimed by keeping a Vigil, a neglected one must be defended by another, and after the finale the Endless Vigil never dawns.** It is the game's title, the tech lens's name for the content type, and already what the fiction calls the Watch's duty. One name for one thing means the night sieges, the lamp holds and the endgame are the same content with different occasions, tuned once. *Rejected:* the experience lens's ink-blot dungeon runs as a separate type (the Hollow is blank paper, not a blot, and a second wave-run format splits the tuning); the systems lens's "Night Siege" as a name (it describes only one of the three occasions); and a free-running day/night clock (the tech lens cut lighting; "night" is the modifier a Vigil brings, which `nightfall` already is).

### 9.5 The two Wardens

**Decision: THE WARDEN stays the slime boss. The units module's shield-bearer (`warden`, sheet `templar`, BULWARK / AEGIS) is renamed THE SENTINEL and belongs to the Gilt Order.** The boss owns the name in the shipped game, its wave card stamp, its mark (WARDEN SLAYER) and its phase shouts. Renaming the unit costs one string in its CFG and keeps its passive and ULT names untouched, so no callout changes. The fiction joins them: the Order posted a Sentinel as Warden of the Well, the ink took the title, and the drowned Sentinel steps out of the boss in its fourth beat. *Rejected:* ORDER WARDEN (still says "Warden" in every callout), and naming the unit after its passive (THE BULWARK would print the same word twice when BULWARK fires).

### 9.6 Other contradictions resolved

- **Who the Signed are.** The world and systems lenses treat THE WATCH as a fourth named rogue. In the units data it is a squad unit (`squad:true`, unlock 1): the Watch's rank-and-file guard. **The Signed are the three teal units, THE KNIFE, THE BLADE and THE LANCER**, whose shared accent becomes the colour of a signature. THE CAPTAIN is the Watch's named leader.
- **What the Hollow is.** Experience draws it as a spreading black ink blot; world draws it as blank paper and puts loose ink in the slimes. **The Hollow is blankness; the Blot is loose ink**, part of the Wild. A black spreading mass would also fight the "nothing darker than the world's ink line" rule.
- **How many factions.** World 7, experience 5, tech 3 for phase 2. **Five political factions** (Watch, Gilt Order, Underdrawn, Warband, Redrafters) plus **the Wild** (the world lens's Blot and Hatch merged, hostile to all) plus **the Hollow** as an affix, not a faction. Phase 2 ships three.
- **Name collisions.** "Folio" is the world, not the bestiary (that is the Roster). The systems lens's player level named "Vigil" becomes **the Tally**. "Blot" is only the Wild's loose ink; your dropped ink on death is an ink pool.
- **Accidental hits on allies.** One stray hit in a fight is forgiven with a "hey!" balloon; a second flips that side. This merges world's "twice flips" with experience's grace.
- **Where things live.** The Warden is in the Well under the North Gate (world), not a burned field (experience) or a hub fight (systems). The Hollow Knight haunts the Margin and is fought on the back of the last page, which turns the experience lens's best secret into the finale's door while keeping dog-ears as optional secrets.
- **Region hands that broke the colour rule.** The world lens's cream-paper Abbey, grey Palimpsest and low-saturation Hatchwood are all redrawn in full colour, with their identity carried by the boil, rims, ghost lines and hatch-in-shadow instead.
- **How many bosses.** Two story bosses (the Warden, the Hollow Knight), six region lords from the custom bodies (Hierophant, Revenant, Warchief, Plaguebearer, Mesmer, Warlock) who give Laws, and three Signed duels. Lords are never spared. The Signed can be spared, and the erased (the drowned Sentinel, the Hollow Knight) can be sealed.

---

## 10. Cut list and risks

### 10.1 What to cut first if scope bites

In order. Everything above the line keeps the spine intact.

1. **Borrowed Form and back-of-page secrets beyond the finale's one.** Wonderful, optional.
2. **Nibs and Inks.** Loot is the least converged idea. Charms, traits and techniques carry build identity alone.
3. **Crowd extras and moving fronts.** Faction territory changes only when the player acts (a clash won, a Great Lamp claimed); the cold ledger shrinks to an owner per Place.
4. **The Tracing Tower and the Redrafters as a region.** THE MESMER and THE WARLOCK become lords visiting other sheets; Chapter III keeps only the Varnish.
5. **Sheets 12 → 8.** One sheet each for the Palimpsest, Brushlands and Margin, and the Spill folds into Ashridge's sheet as the Well's outflow.
6. **Bond levels and Laws beyond the two story bosses.**
7. **In-sheet streaming**, which is to say: take the linked-arenas fallback on purpose.

**Never cut:** full-colour-by-default, lamps drawing the land, SPARE, first-blow side picking, the Vigil, the Roster, the feel involvement gate, the camera grammar.

### 10.2 The biggest risks to fun

| Risk | What it looks like | Mitigation |
|---|---|---|
| **The world goes white.** | Erasure creeps until play is spent on paper; the user's most repeated complaint returns. | The colour rule is a hard budget (§3.2): no faded state, blank ≤ a quarter of a combat frame outside the Margin, the watched circle everywhere else, screenshot review with the user in phase 1 before any second sheet is built. |
| **The combat doesn't survive open space.** | Fights sprawl, enemies trickle, hitstop fires on other people's hits. | Phase 0 gate 5 is decided by the user's hands; the panel leash, the awake cap and the involvement gate; the linked-arenas fallback. |
| **Walking is dead time.** | Empty green between fights. | Sheets sized to 60–90 s; pacing grid of *look every 30 s, do every 90 s, a story every 10 min*, checked by a debug overlay that marks dead zones; fast travel between lit lamps; the momentum string makes travel a skill toy. |
| **Samey fights.** | The fortieth watch squad. | 40 kinds × three-way clashes × modifiers × affixes × your side of the fight (the same bridge with and against the Watch is two fights) × rivals who adapt × style contracts. |
| **Mud at scale.** | 20 bodies, two teams, ink VFX and numbers turn into noise (checklist items 1, 2, 7, 9, 10). | Only your hits print; allies get an underline and half-strength VFX; one callout stream; the shot director. |
| **The Roster feels like homework.** | Checklists of moves to parry. | Every stamp is earned by doing what feels best in the game (the KIIN, the execution); no page is required on the critical path except the teachers of the first two techniques, and those arrive through normal play. |
| **The run's tension goes soft.** | Death is cheap and cards stop mattering. | Vigils keep real stakes (lose the night's cards, the lamp stays dark); ink left where you fell; Crowned elites that grow on your deaths. |
| **Cinematic fatigue.** | Players start skipping the camera. | One big shot per minute, first time full then short, control kept in everything but lord intros, a setting to reduce. |
| **The faction war isn't legible.** | Territory changes and the player doesn't know why. | The map spread shows who took what since last session, pinned notes explain it in eight words, and every change traces to a clash or a lamp. |

---

*The short version: the world is a drawing being erased; lamps draw it back in full colour; the fights stay small, tight and yours; every enemy you understand can be spared onto your side; and the wave game is how you hold the land you've won.*
