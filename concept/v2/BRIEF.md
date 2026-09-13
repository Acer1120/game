# Folio v2 — shared brief: rebuild the plan around HV Units v30

The user, verbatim: "update and make a better plan based on the new hv units 30".

The plan in question is the open-world RPG concept **"Hollow Vigil Folio"**. Its full text is `/home/codespace/.claude/projects/-workspaces-game/hv-work/concept/OPEN-WORLD-RPG.md`, published at https://claude.ai/code/artifact/1ad30a37-a88d-4b2e-a116-a62e8ae0a45a. It was designed against units module **v9**. The user has now supplied **v30**, a major revision that changes the plan's foundations. Read the whole v1 plan first.

This is a DESIGN pass. Do not edit any game source. Write your findings to the file named in your prompt, appending section by section with a shell heredoc so an interruption loses nothing. /tmp is wiped often; everything durable is under `hv-work/`.

## Where the units are (never read the blob lines)
- **v30 source:** `hv-work/hv-units-30-source.html` (2247 lines, 2.24MB). NEVER read line 108 (Three.js r128 inlined, 603KB) or line 115 (`HVU_SHEETS`, 478 sheets, 1.33MB).
- **Extracts (blobs excluded):**
  - `hv-work/units30/head.html`: lines 1-107, the CSS and the demo UI markup. It includes topbar, dock, drawer, roster, lab, sheet, shport and evolution-preview panels.
  - `hv-work/units30/units.js`: lines 116-2247, the HVU module plus its demo page. Read it in chunks.
- **v9, for diffing:** `hv-work/hv-units-source.html` (never read its line 50).
- **The integration guide:** the units author wrote a 6-step drop-in guide for `hollow-vigil.html` at the top of units.js (lines 1-59). READ IT FIRST. It is the author's own contract.
- **The game:** `/workspaces/game/hollow-vigil (2).html`, with sources in `hv-work/src/` split into numbered section files.
- **Earlier analyses of v9** (partly stale now): `hv-work/audit/HVU-map-{render,host,ai,ui}.md`.

## What changed from v9 to v30 (already scouted; verify what you rely on)
1. **The units became the player's army.**
   - Owned units are team 1, the side of the player hero, who is named **Hallokin** in v30. Enemies are team 2 and fight both.
   - Only owned units earn XP (`grantXp` requires team 1), from their own kills.
2. **A full RPG progression system for owned units:**
   - **XP and levels:** `XP_TO(l)=12*l^1.35`, up to level 50. Each level gives +2% stats, stat points every 2 levels, and skill points every 5.
   - **Tiers:** earned at levels 5/12/20/30. `readyTier` flags one and `advance` takes it with a "rite", host fx `rite`.
   - **STATS:** vigor, might, haste, swift, reach, will. Points come from `POINTS_AT(tier)` plus level points, cap 8.
   - **TALENTS:** one pick per tier from tier 2.
   - **PATHS:** chosen at tier 2, one of three per role. A path lends a move the body lacks.
   - **Skills:** every move ranks 1-3, and at rank 3 takes one of two MASTERY options by move type (swing/shot/strike/beam...).
   - **Gear:** ITEMS and ARTIFACT slots (2 at tier 5 or level 20).
   - **MILESTONES:** VETERAN at 10, HARDENED at 20, and more.
   - **TITLES:** ELITE 150, CHAMPION 300, WARLORD 500, LEGEND 800 by `unitPower`. UNIQUE at level 40. ASCENDED at 30% above `fullBuildPower`, with renown (1% per kill, up to 60%) closing the gap.
   - **Builds:** PRESETS are ready-made builds. `serialize(e)` / `spawnFrom(build,x,z)` make one build object the entire save of a unit: `{kind,tier,level,xp,renown,ascended,items,stats,talents,skills,loadout,label,tint}`.
3. **EVOLVE: a full evolution graph.** A body becomes a stronger relative at a tier and keeps its whole build. Families (`FAMILY_MINIONS` keys): undead, casters, demons, beasts, shadows, orcs, watch, men, astra, custom. Examples:
   - skel → ironskel/skelarcher → greatskel → gravelord
   - watch → captain/knight; knight → templar → timewarden, or knight → mirrorknight
   - necro → warlock → lich
   - orc → berserker/eliteorc → warchief
   - gremlin → imp → demonbrute/succubus
   - bat → hellbat → bloodwing → stormwing
   - hellhound → werewolf → werebear
   - nightfang → verdagger → duskclaw → noctislicer → voidreaper
   - slimes → slime kings or plaguebearer; lavaslime → flamegolem → magmacolossus
   - the Signed: knife → verdagger, blade → spellblade, lancer → pike
4. **New tier-5 capstone kinds:** magmacolossus, abysslord, lich, stormwing, voidreaper, timewarden, plaguedoctor, gravelord, mirrorknight (with mirrorimage), inquisitor, spellblade, siren. There are now 48 named CFG kinds, plus the pack bodies.
5. **51 new sprite bodies (+251 sheets, 478 total).** A whole demon family (demonlord, demonbrute, demonarcher, succubus, imp, gremlin, hellhound, hellbat, bloodfiend, bloodwing), more undead (bonepale/boneblade/bonedark, bonearcher2, shade, ghostfire), eyeball, minotaur, flamegolem, frostfiend, lavaslime, slime kings, blue/pink slimes, pumpkins, harpy, toxcrawler, cannoneer, blackknight, beamknight, darkwarlock, the shadow line (nightfang, verdagger, duskclaw, noctislicer), plus beam, bolt, spike and spit projectiles.
6. **Three.js r128 is inlined in the units file.** The game still loads it from a CDN.
7. **The host contract is the same 14 members.** fx gained `rite`, and fx is now called 108 times.
8. **A real army-management UI exists in the demo:** topbar, dock, drawer, roster, lab (build editor), sheet (character sheet), shport (portrait) and evolution previews.
9. **Asset weight:** the game is 927KB. v30's sheets alone are 1.33MB and the inlined Three.js is 603KB.
10. **The file says `HVU.VERSION = '2026-09-13 final'`.** The author considers it final.

## Known stale-guide pitfalls (verify)
- The guide's step 5 says `spawnFrom(build,x,z,{team:0})`, but the code only grants XP to team 1, and the same guide says Hallokin and owned units are team 1.
- The guide says tiers rise "at levels 3,5,7,10", but the code's `TIER_AT_LEVEL` is `{5:2,12:3,20:4,30:5}` and tiers do not rise by themselves (`readyTier` + `advance`).
- The header still says "Twenty seven enemies". Code wins over comments.

## The v1 decisions most at risk
- **"No character levels."** v30 is levels, stats, talents and evolution.
- **A party of the companion plus one follower.** v30 is built to own and develop an army.
- **Five factions.** They were built from v9 accent colours. v30 has families and a demon family v1 never placed.
- **The Roster's seen/studied/bested/bound.** It now has to connect to owning, levelling and evolving.
- **The protagonist "the last Watchman".** v30 names the hero Hallokin.

## Hard constraints still in force
- The user rejected white/cream "paper" looks three times and wants full saturated colour. They also just asked for everything high-res and easily seeable with no ugly outlines. Crisp, colourful, legible art is non-negotiable.
- It stays a browser game on this codebase. The game's combat feel is its identity.
- Ground every major idea in a named v30 or game system.

## Output
Return a 10-line summary: your core finding in one sentence, then your three strongest recommendations.
