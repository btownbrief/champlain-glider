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
