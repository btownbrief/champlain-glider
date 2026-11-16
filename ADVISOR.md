# Champlain Glider: advisor notes (2026-11-16)

Written after playing `main` (identical to the live site) as a 390x844 touch phone with real CDP touches: about 25 minutes of play across 30+ runs, with four kinds of player (never-lets-go, rhythm tapper, decent human with 160 ms lag and mistakes, and a near-perfect bot). Portrait was played by hiding the `#rotate` overlay, since the live game shows nothing else in portrait. Screenshots and traces are in `/tmp/upgrade-champlain-glider/`.

## Verdict

The flying itself is lovely: one finger, a warm sunset, a bird with real weight. But on Stephen's phone the game is a purple screen that says "Turn your phone sideways to glide", so today the score for portrait is zero. Once you are in, the game has two difficulty settings and nothing between them: ordinary players die at the same spot (the first water crossing, about 850 m, 20 to 30 seconds in) run after run, and a player who has the rhythm cannot die at all (the bot was still flying at 30,000 m after 7 minutes, speed pinned at the cap since second 7). Nine biomes exist, but a normal player only ever sees green hills, and every run of the day is the same hills in the same order. The variety is already built; it is just locked behind a wall almost nobody gets past.

## Scores (1-10)

| | Score | Why |
|---|---|---|
| Fun | 6 | The dive-and-soar loop feels great for the 20 seconds most people get. |
| Difficulty curve | 3 | A cliff at 850 m for most, a flat line forever for the skilled. No ramp. |
| Readability | 4 | Portrait: blocked. Landscape: bird is about 14 px on a 844 px screen at speed. The night (the only threat) is off-screen until the moment it kills you. |
| Taste / looks | 7 | Palette, sun and bird are good. Hurt by text overlap, a six-line rules wall, and fake "players". |
| Phone feel | 2 | Portrait does not work. Touch handling itself is solid (release is never lost). |
| Variety | 4 | Lots of content exists (9 biomes, balloons, Champ, rings), but minute 1 of every run is identical and most runs end inside it. |
| Reason to return | 4 | A daily route and a goal ladder exist, but the route is invisible as a feature and goal 2 ("chain 4 perfect slides") stalls ordinary players for good. |

## Evidence (measured)

| Player | Runs | Survived | Died at |
|---|---|---|---|
| Holds forever | 2 | 24 s | 849 m (first water) |
| Rhythm tapper, landscape | 4 | 19-26 s | 848, 849, 851, 855 m |
| Decent human (lag + slips) | 7 | 17-48 s | 845, 850, 863, 888, 1149, 2122, 2122 m |
| Near-perfect bot | 1 | 422 s and counting | never; 30,163 m, island 18 |

- The first water crossing is at about 830 m and the second at about 2,120 m. 9 of 13 ordinary deaths happened within 60 m of one of them. Cause: a belly-flop into water drops you to 560-620 px/s (`waterDrag 2.2`, `waterMinSpeed 620`) while the night is still doing 1,500+.
- The night is leashed to you: `dusk.maxGap 1150` px, which is under half a second at cruising speed. You can never bank a lead, so one slam is fatal no matter how well you flew before it.
- The night's speed copies yours (`matchStart 0.86` to `matchEnd 1.5`) but is capped at 2,800 while the player cap is 3,350. So a slow player is chased harder than they can fly, and a fast player is never chased at all.
- Today's seed made islands 1, 2 and 3 all "Sugarbush Hills". Ordinary players saw one biome all day.
- Portrait with the overlay removed already runs and looks good (bird about 55 px wide), but the camera shows only about 310 world px ahead of the bird: 0.2 seconds of warning at 1,500 px/s. You cannot see the hill you are about to land on.
- The 15 towns run out at 18,000 m; nothing new after that (only matters once the difficulty wall is fixed).

## Ranked upgrade plan

### 1. Make portrait the main way to play
**Evidence:** `@media (orientation: portrait) and (pointer: coarse) { #rotate { display:flex } }` (about line 286) blocks the whole game. With it hidden, play works but lookahead is 0.2 s and the bottom 55% of the screen is flat green fill.

**Change:**
- Delete the `#rotate` element, its CSS and the media rule. Landscape keeps working as it does now.
- Add a portrait camera profile, chosen in `resize()` when `H > W` (keep the current numbers for landscape):
  - base scale 0.44: `viewVH = H / 0.44` (about 1,920 at 844 tall). This shows about 710 world px ahead at rest and about 850 at speed, roughly three times today's portrait view, and uses the tall screen for the big flights.
  - `anchorX 0.20`, `birdLow 0.50`, `birdHigh 0.34`, `zoomFloor 0.62` (so absolute scale never drops under about 0.27).
- Draw the bird with a floor on its own size so zooming out never shrinks it into a dot: bird draw scale = `max(worldScale, 0.72)` in both orientations (this also fixes the 14 px landscape bird). Collisions stay in world units.
- Give the bird a soft cream halo or 2 px light outline at all times. On the orange Stowe ground the red cardinal nearly disappears.
- Size the layout on orientation change and first load only, not on every `resize` (Safari's toolbar must not rescale the world mid-flight).
- Call `window.BtownTicker?.hide()` when a run starts and `.show()` on the menu and game over.

**Check:** at 390x844 the title shows, one tap starts a run, the bird is at least 38 CSS px wide at every moment of a 60 s run (sample the draw scale), and at least 650 world px are visible ahead of the bird while grounded. Rotate mid-run both ways: no crash, run continues, HUD in place.

### 2. Rebuild the night so mistakes cost a lead, not the run
**Evidence:** deaths cluster at 20-30 s for ordinary players; the bot is immortal. Both come from the same three numbers (leash 1,150 px, speed copies yours, cap 2,800).

**Change (all in `TUNE.dusk` and the dusk block near line 2104):**
- Night speed is set by the island you are on, not by your speed: `target = 850 + 170 * (islandNow - 1)` px/s, no upper cap, eased toward over about 3 s. Remove `matchStart/matchEnd/matchRamp` pacing.
- `maxGap 1150 -> 2600`, `startGap 1200 -> 2000`. A good stretch now banks a lead you can spend on a mistake.
- `pushback 420 -> 300`, `feverPushback 650 -> 450` (with a real lead available, pushback no longer has to carry survival).
- Keep `damping`, `airSafety` and the death beat as they are.
- Make the threat visible: a dark vignette grows in from the left edge, width `lerp(0, 0.22 * W, 1 - gap / 1500)`, on top of the existing sky darkening. The dial stays but maps to the new 2,600 range.
- When a new island begins, tick the dial and show one small line: "Night quickens".

Expected result with these numbers: a 1,450 px/s beginner is outrun on island 4-5 (about 90-120 s), a 2,200 px/s regular on island 9-10 (about 2.5 min), and a perfect player around island 15+ (4-5 min). Tune by playing until those three land in those ranges.

**Check:** rerun the four bot profiles for 5 minutes each. Rhythm tapper median run 45-100 s with no more than a third of deaths inside any single 100 m window; lag-human median 90-180 s; perfect bot dies between 3.5 and 6 minutes.

### 3. Water becomes a skill moment, not a trap
**Evidence:** 9 of 13 ordinary deaths were at a water crossing.

**Change:**
- Stone skipping: if the bird reaches the water while the finger is **up** and the descent angle is shallow (`vy / vx < 0.55`), it skips: bounce with `vy = -0.55 * |vy|`, keep 92% of speed, spray, a rising "plip" note, popup "Skip x2, x3...", +100 points per skip times the skip count. Diving into the water (finger down, or steep) is a splash as today.
- Soften the splash: `waterDrag 2.2 -> 1.2`, `waterMinSpeed 620 -> 1000`. A splash should cost about 700 px of lead, never the run by itself.
- First-crossing coach line, once ever: "Let go over the water to skip".

**Check:** hold-forever bot survives the first crossing with the new night (loses lead, does not die there). A release-over-water bot gets 2+ skips and exits the water above 1,500 px/s.

### 4. Islands as short stages with one twist each, never the same twice in a row
**Evidence:** islands are 600-870 m; ordinary runs end on island 1 or 2; today the first three islands were the same biome. The biomes differ only in hill size and color.

**Change:**
- Shorter early islands: island 1 is 22 keys (first ramp at about 300 m, roughly 12 s in), islands 2-4 are 28-36 keys, then the current 48-68.
- Island 1 stays "Sugarbush Hills" (the gentle teacher). After that, never repeat the previous biome, and the first four islands of a run are four different biomes.
- One twist per biome, code-drawn, no new controls:
  - **Stowe in October:** leaf-swirl thermals, one every 6-8 keys, a 260 px wide column. Soaring through one (finger up) lifts you (`vy -= 2600 * dt`, capped at -900) and scatters leaves; a ring sits at the top.
  - **Quechee Gorge:** 2-3 real gaps per island with a river at the bottom (reuse the water code and the skip rule). Clearing one pops "Gorge jump +300".
  - **Champlain on Ice:** already distinct (flat and fast). Add zero ground friction and a long ice-spray trail so it reads as a speed run.
  - **The Long Trail Down:** already a long descent. Double the coin trails here so it reads as the payoff island.
  - The others keep their current shapes; do not add more.
- Keep the island name banner, but put it in a fixed slot (top center, under the score row) so it never lands on other text.

**Check:** a 90 s run by the lag-human bot visits at least 3 different biomes. Log 20 runs of island sequences: no adjacent repeats.

### 5. Today's tour, fresh hills every run
**Evidence:** `terrain.reset()` seeds everything from `daySeed`, so run 10 today is the same hills as run 1. The title says "Today's route: November 16", which tells the player nothing.

**Change:**
- Split the seed. The **itinerary** (biome order) comes from `mulberry32(daySeed)`. The **hill shapes, coins and decorations** come from `mulberry32(daySeed * 1000 + runsToday)`, where `runsToday` is a counter in a new key `champlain_glider_day` (`{ day, runs, best, streak, lastDay }`).
- Title line becomes the tour, for example "Today: Sugarbush -> Stowe -> Ice -> Gorge".
- Track **today's best** next to the all-time best (do not touch `champlain_glider_best`). Game over shows "Today 12,400 · Best 59,512".
- Day streak: if the last played day was yesterday, streak + 1; show "Day 3" as a small leaf badge on the title only when it is 2 or more. No penalties, no nagging.
- The ghost already stores distance over time, so it still works on fresh hills.

**Check:** two runs in a row have different hill profiles for island 1 (compare the first 20 key heights) but the same biome order. Change the system date by one day: the tour changes, today's best resets, the streak reads 2.

### 6. Goals: three at a time, visible while flying
**Evidence:** goals are strictly one after another. Goal 2 is "Chain 4 perfect slides"; the tapper and lag-human profiles never got past a chain of 2 in 11 runs, so they would be stuck on it forever. Goals are also invisible during the run.

**Change:**
- Hold three active goals at once, drawn from the existing list plus the endless ones (new key `champlain_glider_goalslots`; `champlain_glider_goals` remains the count and still drives the multiplier and the thrush unlock). Finishing one swaps in the next unused goal.
- Always include one easy goal (coins, clouds, distance, skips, visit a biome) among the three.
- Add goals for the new verbs: "Skip 3 times in one crossing", "Ride 2 thermals", "Clear a gorge gap", "Reach island 4".
- In the run: show the three goals as small chips for the first 3 s, then only a brief "Coins 8/12" style tick when one advances, and a real moment (chime, gold chip, "+1 multiplier") when one completes.

**Check:** a fresh save shows three goals; the rhythm tapper completes at least one goal within its first 3 runs; completing a goal mid-run updates the chip and the stored count.

### 7. Subtract: one line of rules, no overlapping text, no fake players
**Evidence (screenshots):** in portrait the title's six-line rules block runs under the "Sound on" pill and over the bird; "1 in a row" prints on top of the score's "x2"; "Perfect!" prints on top of the "KEEP HOLDING" bar; the first-run coach text is cut off at the left edge ("HOLD ON THE DOWNHILL" starts off-screen). Rivals are named like real users ("maple_mike joined the run", "gg").

**Change:**
- Title: logo, one line ("Hold to dive. Let go to soar."), bird chips, today's tour, one big button. Everything else goes.
- Mute becomes a 40 px round icon button, top right, under BEST.
- One popup lane: all text popups stack in a column above the bird, clamped 12 px inside the screen edges, at most 2 visible; the combo counter lives under the score (left), never top center.
- Coach cue is clamped inside the screen and hides popups while it is up.
- Rivals: label them by species ("a blue jay"), drop "joined the run", the chat lines and the usernames. Keep the pass bonus.
- Game over: drop "(progress normalized)"; say "Board score 4,334".
- Greens ground texture (the polka dots): cut its contrast by half; it is the busiest thing on screen at portrait scale.

**Check:** screenshot title, 5 s, 30 s, a Fever moment and game over at 390x844: no text touches other text or the screen edge; nothing on the title needs scrolling.

### 8. One tap to flying
**Evidence:** the first tap starts a 2.4 s wake-up scene; play only starts after it (or a second tap). Small, but it is the first impression every session.

**Change:** cut the wake-up to 0.8 s and let the first press count as the first hold (if the finger is still down when it ends, the bird is already diving). Keep the first-run coach.

**Check:** from page load, one touch-and-hold has the bird moving under player control within 1.0 s.

## Remove

- The rotate overlay and its CSS.
- The six-line `.howto` block (keep the short one, shortened further).
- `dusk.matchStart`, `matchEnd`, `matchRamp` and the 2,800 night cap.
- Fake usernames and rival chat ("gg", "joined the run", "dropped out").
- "(progress normalized)".
- The "Sound on" text pill (replace with an icon).
- "Today's route: November 16" (replace with the tour).

## Keep

- The hold-to-dive, release-to-soar physics and all of `TUNE.phys`. It feels right; do not retune it.
- The sunset palette, the sun, the bird art and the four birds.
- The capture-phase pointer handling (the release is never lost), pause on tab hide, the death beat and the postcard.
- Fever, perfect slides, the crest coach bar, coins and the double-tap boost.
- Balloons, Champ, creemee stands, towns and postcards as quiet garnish.
- The leaderboard exactly as wired: game id `champlain-glider`, `submit_score` with `run.boardScore`, the Supabase calls, and every existing `localStorage` key (`champlain_glider_best`, `_bird`, `_goals`, `_postcards`, `_ghost`, `_bestdist`, `_coach_v2`, `btown-player-*`). New data goes in new keys only.
- The ticker script tag, GoatCounter, manifest and service worker.

## Acceptance tests

1. **Portrait works:** at 390x844 (isMobile, hasTouch) no rotate message appears; one tap from the title starts a run; no console errors.
2. **Readable:** in portrait the bird is at least 38 CSS px wide throughout a 60 s run and at least 650 world px are visible ahead while grounded; the night vignette is visible whenever the gap is under 1,500.
3. **Fair curve:** over 5 minutes per profile, rhythm tapper median run is 45-100 s; lag-human (160 ms, slips) median is 90-180 s; the perfect bot dies between 3.5 and 6 minutes. No profile has more than a third of its deaths inside one 100 m window.
4. **Water:** the hold-forever profile survives the first crossing; a release-over-water profile scores at least 2 skips.
5. **Variety in a run:** a 90 s lag-human run passes through at least 3 different biomes; no biome repeats back to back in 20 logged runs.
6. **Variety across runs and days:** two consecutive runs have different island-1 hills and the same biome order; advancing the date changes the order and resets today's best while `champlain_glider_best` is untouched.
7. **Goals:** three goals show on a fresh save; the tapper profile completes one within 3 runs.
8. **No overlap:** screenshots of title, early play, Fever and game over show no text touching other text or the screen edge.
9. **Phone inputs:** fast repeated taps, a two-finger hold with one finger lifting, a finger sliding off the screen edge, and rotation mid-run all leave the bird controllable (never stuck diving).
10. **Scores intact:** an existing `champlain_glider_best` still shows after the update; a finished run still calls `submit_score` with game `champlain-glider`; landscape and desktop (Space bar) still play.

## Review after upgrade

Written 2026-11-16 after playing branch `upgrade-2026-11-16` fresh as a 390x844 touch phone. Real CDP touches for about 14 minutes across four play styles (never lets go, rhythm tapper, casual with 240 ms lag, decent human), one landscape session, then fast-forward bots for 60+ more runs across four different days. Compared against the live site, which still shows only "Turn your phone sideways to glide". Screenshots and logs: `/tmp/upgrade-champlain-glider/review/`. Console: clean (only GoatCounter's localhost notice). Leaderboard calls still use `champlain-glider`. An old best still shows, and the streak badge read "Day 2 in a row" the next day.

### Scores (before -> now)

| | Before | Now | Why |
|---|---|---|---|
| Fun | 6 | 7.5 | The flying now lasts 90 to 150 s instead of 20, in portrait, with a big bright bird. The gorge is a real highlight. Held back by a chase you cannot feel (below). |
| Difficulty curve | 3 | 6 | The 850 m wall is gone and better players do go further (tapper 79 s, lag human 144 s, near-perfect 231 s median). But never letting go lasts 116 s, longer than tapping, and death arrives as a 3 second ambush. |
| Readability | 4 | 8 | Bird is never under 50 px, the ground is calm, the landing is in view. Loses points for text spam (35 to 53 popups a minute), an invisible thermal, and a rival bird of your own species and color. |
| Taste / looks | 7 | 7.5 | Title is clean and the tour line is charming. Still noisy in play: three stacked text lines over the bird are common, "FEVER!" prints on top of "Perfect! x3", and a gameplay hoop is drawn through the game-over text. |
| Phone feel | 2 | 8 | Portrait works, one touch starts, retry button is on screen without scrolling, touch never stuck in 14 minutes. Not yet seen on a real iPhone. |
| Variety | 4 | 6.5 | Five or six biomes in a normal run, a new one about every 20 s. But the order is identical all day, today's tour has no twist island before island 9, and Ice is a flat line with nothing to do. |
| Reason to return | 4 | 6 | Daily tour, today's best and a quiet streak badge all work. But the last bird unlocks by run 3 or 4, goals fall at 1 to 2 per run without trying, and the growing multiplier makes "New best!" automatic. |

### What clearly improved

- **Portrait plays.** This was the whole problem for the #2 game, and it is fixed. First touch has the bird under control in under a second.
- **Readable at a glance.** Player, ground and next hill are obvious in both orientations. Landscape no longer has a 14 px bird.
- **No more wall.** Every play style now sees 4 to 6 islands. No run in 60+ died at the first water.
- **Quechee Gorge** looks and plays like a different game for 30 s: real chasms, a river, a clear "jump it" moment.
- **Water** is now a beat, not a trap. Zero deaths in water in every log.
- **Subtraction on the menus.** One line of rules, tour, one button. Fake usernames and chat are gone.
- **Fresh hills every run, new tour every day**, old scores and keys untouched.

### What is still weak or broken

1. **The night is invisible, then instant.** Across 24 logged runs the night was within 1,500 px for only 1 to 13% of the run (typically 5%). The lead sits pinned at its 2,600 cap, then one stall on island 5 or later and you are dead 2.1 to 9.9 s later (typically about 3 s). The same slam on islands 1 to 3 costs nothing you can see. So there is no rising tension, and the death reads as sudden rather than earned. All four real-touch styles died between 88 and 115 s.
2. **Holding forever is a winning strategy.** The never-let-go player has a 116 s median (real touch: 115 s, island 6) against 79 s for the tapper, and collects 16 to 33 "Perfect!" slides a run for doing nothing. The game's one skill is optional for the first two minutes.
3. **Run 10 today is run 1 again.** Hills differ, but every run today is Sugarbush, Long Trail, Green Mtns, Ice, Intervale, in that order, ending on island 5 or 6. On day 5 all six runs ended on the same island.
4. **The twists are easy to miss.** Today's tour has no Stowe or Gorge before island 9, so today's players never meet thermals or gorge jumps. When Stowe does appear, the thermal column cannot be seen in screenshots; only the word "Thermal!" tells you it happened.
5. **Ice is dead air.** Flat ground means no dive and no soar. It runs 14 to 19 s normally and 32 to 49 s when it is long (day 5: a third of the run, 31 to 44% airborne against 70 to 85% elsewhere).
6. **Text spam.** "Clean launch! +100" fires 13 to 43 times a run, "Passed a ...!" 6 to 19 times, plus Cloud, Hoop, Perfect, Slam, Boost, and the "LET GO!" / "KEEP HOLDING" coach, which still shows on every hill in later runs. That is a popup nearly every second.
7. **Progress is spent in one sitting.** Six runs completed 6 to 9 goals; the Hermit Thrush (5 goals) unlocks by run 3 or 4 and nothing is left to earn. Meanwhile the multiplier inflates the score: on one day a 85 s run scored 45,000 and a later, shorter 75 s run scored 154,000. The game over then shows a second, smaller "Board score", which is confusing.
8. **Small things.** In landscape a thumb tap at the lower middle of the title lands on "More games" and leaves the game (it happened to the test). A rival cardinal can fly next to the player's cardinal. A hoop is drawn through "Today · Best" on the game over.

### Final punch list (ranked by impact)

1. **Make the night a chase you can see and survive.** Raise `maxGap` from 2,600 to about 4,200 so good flying banks a lead that keeps moving, and raise island pace about 8 to 10% to pay for it. After a slam on land, floor the ground speed at about 1,000 px/s (the same fix already made for water; the 560 to 740 px/s crawl is what kills). Make the dark edge at least 12% of screen width once the gap is under 700. Targets: night within 1,500 px for 20 to 35% of a run; at least 6 s median from full lead to death; lag human still 90 to 180 s, near-perfect still 3.5 to 6 min.
2. **Stop paying for holding forever.** A slide only counts as "Perfect" if the press began in the air during that flight (not held since before takeoff), and holding up a hill for more than about 0.4 s bleeds speed. Targets: never-let-go median under the tapper's and under 70 s, with 3 or fewer perfects a run; other profiles unchanged.
3. **Shuffle the day, and promise a twist.** Keep the day's set of islands, but shuffle islands 2 to 4 each run and draw island 5 onward per run (still no back-to-back repeats). Every day's first four islands must include Stowe or Gorge. Title reads "Today's islands". Target: 10 runs on one day give at least 4 different island orders, and every one of the next 30 days has a twist island by island 4.
4. **Fix the two flat twists.** Ice: cut its length to about 12 s (roughly 16 to 22 keys) and add 2 or 3 snowdrift kickers with a hoop above each, so there is something to time. Thermals: draw a clearly visible column (soft pale band plus rising leaf streaks) readable from 700 px away. Target: Ice never over 15 s and at least 55% airborne; a thermal is identifiable in a still screenshot.
5. **Cut the text to one line.** Remove the "Clean launch" and "Cloud" popups (keep their sounds), limit rivals to one per island and never the player's own species, retire the crest coach after the first two runs, and fix "FEVER!" landing on "Perfect!". Target: 12 or fewer popups a minute and never more than one text line near the bird (plus the banner).
6. **Give tomorrow something to earn.** At most one goal completed per run, so the multiplier and the Thrush take days, not minutes. Add one daily target: reach island 5 on today's tour to stamp the day (a check mark on the title's tour line; the streak counts stamped days). Show one headline number on the game over. Also move the credit links to the bottom edge, away from the start button, and stop drawing hoops behind the game-over text. Leaderboard id, `boardScore` submission and all existing storage keys stay exactly as they are.
