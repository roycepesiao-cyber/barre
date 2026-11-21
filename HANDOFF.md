# barre: handoff guide

A soft-pink, mobile-first barre/posture workout app. It is installable as a PWA and needs only a **mat, a wall and a chair**.

- **Live:** https://roycepesiao-cyber.github.io/barre/
- **Repo:** https://github.com/roycepesiao-cyber/barre (GitHub Pages serves `main` from the root)
- **Owner's phone:** Tecno Camon 50 Ultra (Android), installed from Chrome

---

## 1. Hard constraints (keep these)

1. **Single-file app.** All HTML, CSS and JS live in `index.html`. There is no build step, no framework and no npm.
2. **`sw.js` is the only other file.** Browsers require a service worker to be its own same-origin JS file, so it cannot be inlined.
3. **No external assets** except Google Fonts (Playfair Display + Nunito). Sounds are synthesized with Web Audio and icons are an inline SVG sprite.
4. **Mobile-first.** Design for 360–430 px wide portrait screens. The timer must be readable from about 2 m away.
5. **Equipment:** mat, wall and chair only. No weights or bands.
6. **Aesthetic:** soft pink, ballet-inspired. Italic Playfair headings, rounded Nunito text, custom line icons (bow, pearl, tiara, ballet slipper…). **No emoji in the UI.**
7. **Data stays on the device** (`localStorage`). There is no backend, and the leaderboard is simulated deterministically.

## 2. Release / deploy

1. Edit `index.html` (and `sw.js` if needed).
2. **Bump `CACHE` in `sw.js`** (`barre-v4` → `barre-v5` …) every release.
3. Commit to `main` and push. Pages rebuilds in about 1 minute.
4. Phones pick up the new version the next time they open the app online. The service worker is network-first for the page and cache-first for fonts.

Commits use the GitHub no-reply email `325006110+roycepesiao-cyber@users.noreply.github.com`. Keep it that way: don't put a personal email in public commits.

## 3. Local testing

```bash
python -m http.server 8765
```

- Open `http://localhost:8765/` at a 375×812 mobile viewport.
- **Separate test storage:** `localhost` and `127.0.0.1` have different `localStorage`. Test on one so you don't wipe the owner's data on the other. Two tabs on the same origin overwrite each other's saves.
- **Fast-forward a workout** from DevTools:
  ```js
  const o = performance.now.bind(performance); let off = 0;
  performance.now = () => o() + off;
  setInterval(() => off += 3000, 20); // then click Start
  ```
  A session counts only if at least 70% of the planned time elapsed.
- **Reduced motion:** the dev Windows machine has OS "reduce motion" ON. With Settings → Theme → Animations on **Auto**, most animations are skipped. Set it to **On** to test animations.
- `navigator.vibrate` logs console errors with scripted clicks. That's harmless, because real taps are allowed to vibrate.
- Save-key: `barre.save.v1`. Export/Import save codes use the prefix `BARRE1:` followed by base64 JSON.

## 4. Feature summary

| Area | What it does |
|---|---|
| **Tabs** | Train · Player · Ranks (League / Badges) · Shop & Challenges · Settings |
| **Workout** | 10/15/20-min sessions. Each block is one move: `work` s + `rest` s (set by the training level), plus a 10 s get-ready. Pause, Skip, **Gentler** swap (same category, gentle move), End. Wake Lock, 3-2-1 beeps + chime, haptics. |
| **Pose engine** | SVG stick figures with 2 frames, tweened each second. The target muscle glows in the accent colour. Props: wall, wall-behind, door post, chair (side/front), mat. |
| **Library** | 38 moves in 5 categories: `p` Posture, `l` Barre, `g` Glutes & Back, `c` Core, `f` Stretch. Each is `lvl` 1 (gentle) or 2 (standard). |
| **Programs** | Daily Mix, Posture Reset, Barre Burn, Core & Lengthen, Stretch & Release. Each has a category pattern. The planner is seeded by date/program/shuffle/session-count, avoids recent moves and always ends with 2 blocks of stretch. |
| **Training levels** | 8 levels (progressive overload): density 30/30 → 50/10, then slow tempo (L5+), advanced variation cue per move (L7+), burnout banner in the last 10 s (L8). Gentle moves are weighted down as the level rises. |
| **Autoregulation** | After a session, "How did that feel?": Easy +2, Just right +1, Too hard 0. Two Too-hard in a row = level −1. Skipping more than 25% of blocks = no progress. 10+ days off = level −1 (once per gap). Manual Easier/Harder lives in the level modal and does **not** raise `train.best` (badges need earned levels). |
| **Economy** | Pearls (coins): 1 per minute × division bonus, daily +15 (+division bonus), weekly +50 each, badges 10–300, promotions 30–150. |
| **Streak** | One workout a day. A Streak Freeze is spent automatically for each missed day (cap 2, or 3 from Ribbon). Price 50/40/30 depending on division. |
| **League** | 10 rivals + you. Rival points = `base + dayProgress × rate`, seeded by week+division. You score 10 pts/min. Resets Monday 00:00 local. Top 3 are promoted and the bottom 2 drop (if possible). |
| **Divisions** | Blush → Rosé → Pearl → Ribbon → Satin → Swan → Prima. Benefits are **implemented**, not just text: pearl bonus %, promotion gift, daily bonus, freeze cap/price, free Monday freeze (Swan+), +50/week stipend (Prima), theme unlocks. |
| **Achievements** | 37 badges in rarities Petal/Pearl/Diamond (sessions, streak, minutes, variety, flawless, time of day, posture check-ins, league, levels). |
| **Animations** | Direction-aware tab slides, staggered card entrances, bar fills, sheet open/close, timer tick, move cross-fade, pearl count-up, theme blend. Post-workout **rank climb** (your row passes rivals one by one, petals at the promotion zone). Monday **division level-up** and **strength level-up** overlays. |
| **Themes** | Ballet Pink (default), Blush Night (dark), plus 6 unlocked by division best. |
| **Other** | 5-second posture check + posture trend chart, `.ics` daily reminder download, Export/Import save code, install prompt, iOS hint. |

## 5. Code map (`index.html`)

Search for these section headers (`/* ====`):

**CSS:** `THEMES` (CSS variables per `[data-theme]`), `BASE`, `MOTION (minimal)`, `TRAINING LEVEL`, `RANK CLIMB + LEVEL-UP`, then the `.calm` reduced-motion rule.

**HTML:** an inline SVG icon sprite (`<symbol id="i-…">`, used via `ic('name')`), topbar, the 5 `<section class="tab">` elements, tabbar, unlock splash, toast, `#modalRoot`.

**JS (inside one IIFE), in order:**

| Section | Notes |
|---|---|
| Date helpers | `ymd()`, `weekKey()` (Monday), `dayNum()`, seeded `rng()` / `hashStr()` |
| Move library | `EX` object: moves, `CATS`, `EQUIP`, pose templates (`SUPINE`, `SEAT`, `QUAD`, `PRONE`…) |
| Progressive overload | `LEVELS`, `ADV` (advanced cue per move), `ISO` (hold moves), `lvl()`, `levelCue()` |
| Programs / Divisions / Themes | `PROGRAMS`, `DIVS`, `THEMES` |
| Achievements | `ACH` tuples: `[id, name, desc, icon, rarity, reward, valueFn, goal]` |
| Challenges | `DAILY` (rotates by day number), `WEEKLY_POOL` (3 picked per week, seeded) |
| State | `DEFAULT`, `hydrate()` (sanitizes every field, migrates old saves), `S`, `save()` |
| Pose engine | `SEGS`, `normFrame()`, `propsSvg()`, `class PoseView` (`set`, `play`, `stop`, `reset`) |
| Sound | `Sound.beep/chime/tick/fanfare` |
| Wake lock, Streak | `requestWake()`, `checkStreak()`, `bumpStreak()` |
| Evaluation | `evalDaily`, `evalWeekly`, `evalAchievements`, `weekStats` |
| League | `boardFor(week, dayProgress, div)`, `rollLeague()`, `announceLeague()` |
| Planner | `buildPlan(n, programId, seed)`, `currentPlan()` |
| Workout engine | `W` state, `DUR` (set from level at start), `startWorkout`, `tick` (100 ms), `advance`, `skip`, `swap`, `finishWorkout` |
| Completion | `showComplete()`: rewards, effort rating, posture check. Done runs strength level-up → rank climb → Train tab |
| Motion helpers | `reduced()`, `applyMotion()`, `countTo()`, `petals()`, `showClimb()`, `showLevelUp()` |
| Training level | `progressPlan`, `describeProgress`, `applyProgress`, `checkDetrain`, `renderLevelCard`, `showLevels`, `showStrengthUp` |
| UI | `openModal/closeModal`, `toast`, `showTab` |
| Renderers | `renderTop/Train/Ranks(light)/Badges/Shop/Settings/Library`, `previewMove` |
| Misc | `.ics` export, save codes, PWA manifest + canvas-drawn icons, events (delegated `data-*`), boot |

## 6. How to add things

**A new move.** Add an `EX` entry:

```js
myMove: { name, short, cat: 'g', lvl: 2, eq: 'mat', target: 'Glutes · Core',
  props: [{ t: 'mat' }], hl: ['glutes', 'thL'], gl: [dx, dy], view: 'front' /*optional*/, bilateral: true /*optional*/,
  cues: ['…', '…', '…'],              // exactly 3 short cues
  frames: [ {joints…}, {joints…} ] }  // start pose, active pose
```

- **Coordinates:** a 200×200 box. The floor is y = 184 and the mat top is y = 177. Side views face right.
- **Joints:** `head, neck, mid, hip, sh, eL, hL, kL, fL, tL` (+ `R` versions). `L` = near limb (solid), `R` = far limb (faded). Missing `R` joints default to `L`. Front views use `shL/shR`, `hipL/hipR`.
- **Highlight names:** `torsoUp, torsoLow, shoulders, pelvis, thL/thR, shinL/shinR, uaL/uaR, faL/faR, neck, glutes`.
- **Also add:** an `ADV[myMove]` string (advanced cue). If it is a static hold, add it to `ISO`.
- **Check:** view it in Player → Move library (frame 0), and tap the card to see the animation.

**A new saved field.** Add it to `DEFAULT` **and** sanitize it in `hydrate()`. Imported save codes and old saves go through `hydrate()`.

**A new badge.** Add a tuple to `ACH`. It is evaluated after workouts, ratings, league rolls and level-ups.

## 7. Known limitations / ideas backlog

- The stick-figure poses are approximations; a few could be refined.
- The leaderboard is simulated, so there are no real friends or accounts.
- Progress lives only on one device. A cross-device sync would need a backend.
- Ideas: voice cues for the next move, per-category strength levels, weekly summary card, more advanced moves, deload week every 4th week, friend codes for a real league.
