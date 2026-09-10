# HOLLOW VIGIL — polish checklist

Rule: one item at a time. Each item ends with the game file (`hollow-vigil (2).html`) updated and
verified, then it STOPS and waits for you to check it. Nothing on this list moves until you say so.

Status key: `[ ]` not started · `[~]` in progress · `[x]` done, waiting for your check · `[✓]` you approved

> Items 1-16 are the preliminary list from a first look. After item 0 is approved, an audit
> makes every one of them precise (exact file/line, what you'll see) and may add or split items.

## The look (your words on 9 Sep: "too faded, dirty, hard to see, muddy, doesn't feel crisp or real; I want a super crisp, sketchy, animated, popping-out look, not a dirty old look")
- [x] **0. Crisp, clean, vivid, popping.** Two halves, one item:
  - (a) Ship the already-finished "clean page" + motion-smoothing sources (the file you play is the old grey Sep 7 build; the sources were cleaned on Sep 8 but never copied over). Verified first.
  - (b) Push it to CRISP: hard dark ink outlines with no dropout or grey pressure, no graphite double-line fuzz, no paper grain/speckle/smudge, no grey hatch over mid-tones, full-saturation colour on a bright paper, sharp (not blurred) upscale, crisp cast shadows, actors popping off the ground with a paper gap and a heavy rim. The line boil and animation on twos stay; the dirt goes.
  - Check: play a wave. Roofs, trees, slimes and the knight should look like clean vivid ink drawings on bright paper, sharp at any zoom, nothing grey or fuzzy.
  - Done 9 Sep: (a) the clean-page + motion-smoothing sources are shipped in `hollow-vigil (2).html`; (b) CRISP pass behind one `CLARITY` dial (0..1, default 1, in the renderer/textures section): floor ticks cut to a few faint marks, mid-tone hatch removed, grain/speckle/smudge off, wash pulled back to full-saturation colour, pen pressure 1.0 / dropout 0, graphite double-line removed, heavier ink rim + paper gap on actors. Verified: 0 page/console errors, reset 19/19, settings 25/25. SKETCH OFF in the pause menu is still the original look. Waiting for your check.
  - Fixes from your play-test (9 Sep, later): (1) the knight no longer turns black mid-swing — afterimages/smears are faint paper-line copies trailing behind the blow, never ink shapes on the body; (2) damage numbers now print big ink digits with a paper halo (gold for crits, red for damage taken) and always show on light hits; (3) the archer (player class and companion) uses one skeleton sprite family, so no more elf↔skeleton flashing. Verified: 0 page/console errors, reset 19/19, settings 25/25, mid-swing frame bursts checked.
  - Second play-test fixes (10 Sep): the world is back to PROPER COLOUR — the original green field and full-saturation roofs/trees with the ink line drawn on top (no more white page; floor tint and saturation are named constants next to CLARITY); the knight stays a coloured figure with one rim while MOVING and swinging (afterimages/smears now use a line-only outline instead of the filled halo); damage numbers are big saturated yellow/orange digits with a thick ink outline (gold crits, red taken), ~0.9 s on screen; and the black blob on roofs was the D/C style-rank letter closing into a solid shape — now a legible letter. Verified: 0 page/console errors, reset 19/19, settings 25/25, moving-swing bursts checked. Waiting for your check.

## UI that doesn't fit
- [ ] **1. Wave banner off the play area.** The WAVE N card sits in the middle of the screen over the player, "WAVE N CLEARED" prints on top of it, and the boss stamp collides with it.
- [ ] **2. Top-right HUD cluster.** The combo counter gets clipped under the letterbox and crowds the FELLED / WAVE chip; stray tally marks are cut off in the corner.
- [ ] **3. Controls legend.** The big keys bar sits over the world and the dials; make it compact, auto-hide, and readable.
- [ ] **4. Every viewport.** HUD, dials, cards and banners checked at 960×540, 1366×768, 1920×1080, ultrawide and at UI SIZE 75/125%.
- [ ] **5. Pause / death / title pages.** World captions bleeding through the pause plate, death flood, title layout.

## Banners and cut-ins (not satisfying)
- [ ] **6. Ability / ult banner (IRON VIGIL card).** A real entrance (anticipation → slam → settle), a real exit, right timing, no lingering.
- [ ] **7. Splash band and kill cut-ins.** SLAIN / execute / chain bands: shorter, punchier, animated stamp landing, never overlapping other UI.
- [ ] **8. Wave-clear beat.** One clean celebration moment (stamp, sound, brief slow) instead of overprinted text.

## Ink too heavy
- [ ] **9. Attack ink.** The grey cloud / smoke blobs around the knight mid-swing and sweep, ghost copies, ribbons and speed-line bundles thinned; the hit signal kept.
- [ ] **10. Page noise.** Ground flecks, scrawls, decals, drips and the death-screen flood brought down to a record, not a mess.

## Animations that feel half done (your words 9 Sep: "cooler, smoother, more well-built animations")
- [ ] **11. Player.** Each of the 4 hits reads differently, anticipation and follow-through, dash smear, landing, hurt, death; nothing snaps.
- [ ] **12. Enemies + companion.** Wind-up telegraphs, hit reactions, deaths (corpses fade), spawn-in, the Warden's entrance.
- [ ] **13. Transitions.** Title → game, wave → wave, portal travel, death → restart: every transition has both a start and an end.

## Feel and cool animations
- [ ] **14. Hit feel.** Hitstop, shake, impact frame, kill pop, chain feedback and sound sync tuned so a hit feels like a hit.
- [ ] **15. New signature animations.** A short list of high-impact additions in the sketchbook idiom (title drawn stroke by stroke, kill cut-in, boss entrance, ult finale…) — you pick which.

## Cleanup
- [ ] **16. Code hygiene.** Swallowed-comment audit, dead code (e.g. the unreachable kill panel), unapplied hooks, stray elements.
