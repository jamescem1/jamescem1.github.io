# Stub It Out — handover

A stop-smoking app disguised as a city builder. You build an island town out of the cigarettes you *don't* smoke; go over your limit after locking in and today's progress burns.

- **Live:** https://jamescem1.github.io/stub-it-out/ (GitHub Pages, served from `master`)
- **Code:** everything is in one file, [`index.html`](index.html) (~2,200 lines of HTML + CSS + vanilla JS). No build step, no dependencies except Google Fonts.

## Where things stand

| PR | Branch | What | State |
|----|--------|------|-------|
| #1 | `claude/stub-it-out` | First tracker game | merged |
| #2 | `claude/stub-it-out-city` | Lego-style cigarette builder | merged |
| #3 | `claude/ash-town` | "Ash Town" isometric builder (v2) | merged — **this is what's live** |
| #4 | `claude/ash-town-v2` | Full island city builder (v3) + this doc | **open, draft** |

To ship v3: mark PR #4 ready and merge. Pages redeploys in ~1 minute.

## Run it locally

```bash
git clone https://github.com/jamescem1/jamescem1.github.io.git
cd jamescem1.github.io
git checkout claude/ash-town-v2      # until PR #4 is merged; then just use master

python3 -m http.server 8000          # any static server works
# open http://localhost:8000/stub-it-out/
```

Opening the file directly (`file://…/index.html`) also works. Note that saves live in `localStorage`, which is per-origin, so `localhost:8000`, `file://` and the live site each have their own separate save.

On a phone, open `http://<your-computer's-LAN-IP>:8000/stub-it-out/` while on the same Wi-Fi.

## How the game works (the rules the code enforces)

- **Daily grant.** Each day gets `committed = baseline − limit` 🚬 (e.g. 20 − 11 = 9). `baseline` and `limit` are snapshotted onto the day when it's created, so changing settings doesn't rewrite history.
- **Building.** Buildings cost 🚬 (some also 🪙 or 💎). Anything built today is *under construction* until the day is locked in.
- **Lock-in.** Happens automatically when today's count reaches the limit, or early via the 🔒 button. Early lock-in pays a bonus of `limit − smoked` 🚬. Past days count as locked at midnight.
- **Ruin.** A day is *bust* if `smoked > limit` **or** any cigarette was logged after `lockedAt`. On a bust day: today's 🚬-funded buildings burn, today's upgrades are void, crops that were growing at that moment wither, and today's grant and bonus don't count. Earlier days are untouched.
- **Undo.** Popping the last cigarette re-derives everything (see below), so an accidental tap is fully reversible. If the popped cigarette was the one that auto-locked the day, the lock and its XP are undone too.
- **Quit mode.** `limit = 0` locks the day in as soon as it's created.

### Most state is derived, not stored

This is the key design choice. The 🚬 balance, burned buildings, void upgrades and withered crops are all **computed from the day records** every time, rather than being mutated when you go bust. That's what makes undo trivial and keeps the economy consistent.

- `cigBalance()` = Σ over non-bust days of (`committed` + bonus if locked) − Σ 🚬 cost of standing buildings and non-void upgrades
- `isBurned(o)`: the building cost 🚬 and its `day` is bust
- `upVoid(u)`: the upgrade's `day` is bust
- withered crop: `crop.wd` (set at the moment of bust) points at a bust day

Coins, gems and XP **are** plain counters (`state.coins`, `state.gems`, `state.xp`).

## Code map (`index.html`)

Sections are marked with `// ---------- Name ----------`; search for those.

| Lines (approx.) | Section | What's there |
|---|---|---|
| 11–182 | `<style>` | HUD, dock, sheets (bottom modals), shop, onboarding, mini-game |
| 183–386 | HTML | Canvas `#map`, HUD, dock, every sheet (`m-shop`, `m-info`, `m-plant`, `m-confirm`, `m-crave`, `m-smoke`, `m-quests`, `m-stats`, `m-help`, `m-settings`, `m-breathe`), onboarding `#onb`, game `#game` |
| 388 | Utilities | date keys, formatting, `rng`, `hash2` |
| 409 | State | `DEFAULTS`, `load()` (incl. migration), `save()` |
| 440 | Days | `today()`, `committedOf`, `bonusOf`, `isBust`, `isLocked` |
| 463 | Player | `xpTo(level)`, `playerLevel()`, `landSide()`, `unlocked(i, j)` |
| 470 | Catalog | **`DEF`** (every building/decor), **`CROPS`**, **`OBST`**, `UP_TIME`, `PACKS` colours |
| 509 | Town | `ensureTown`, levels/upgrades, `cigBalance`, `production`/`available`, `cropInfo`, placement (`canPlace`, `findSpot`), `spawnLitter` |
| 617 | Drawing kit | iso helpers: `P(i,j)`, `corners`, `box`, `band`, `patch`, `win`, `cig`, `cyl`, `cone`, `pyr`, `flag`, `extras` |
| 713 | Sprites | one `sXxx(c, o, f)` per building/decor/obstacle, plus ruins, flames, scaffold; `SPRITES` / `OSPRITES` maps |
| 1098 | Scene | camera, ground cache (`buildGround`), `draw()` render pass, bars, bubbles, placement ghost |
| 1305 | Ash people | walkers, flee-on-fire |
| 1369 | Effects | floating text, dust, confetti; `frame()` loop (~30 fps) |
| 1396 | Input | pointer pan, pinch/wheel zoom, `tap()` hit-testing |
| 1459 | HUD | `renderHud()` (runs every second) |
| 1486 | Sheets | `openSheet`/`closeSheet`, `toast`, `confirmBox` (queued, one at a time) |
| 1513 | Progress & rewards | `gainXp` (level-ups), `lockIn`, `ruin`, `logCigarette`, `undoLast`, `beatCraving` |
| 1600 | Objects | collect, harvest/plant, obstacles, building info sheet, upgrades |
| 1757 | Shop & placing | shop tabs, `startPlace`, confirm/cancel placement |
| 1828 | Quests | `QUEST_POOL`, 3 seeded per day |
| 1873 | Stats & badges | `stats()`, `BADGES`, chart |
| 1959 → 2186 | Wiring | smoke/craving buttons, settings, new-day summary, onboarding, breathing, Craving Crusher, boot |

## Data model

One `localStorage` key: **`stubItOut.v1`** (the name stuck from v1; the shape is versioned by `state.v`, currently `3`).

```js
{
  v: 3, onboarded: true,
  settings: { baseline: 20, limit: 11, packPrice: 15, packSize: 20, currency: '£', reason: '', name: 'Ashville' },
  days: {
    '2026-10-26': {
      smoked: [timestamps], resisted: 2, limit: 11, baseline: 20,
      lockedAt: ts | null, autoLockCig: ts, rewarded: true, lockXp: 175,
      q: { collects, harvests, clears, games, builds, breathes },   // quest counters
      quests: ['crave2', 'lock', 'gap'], claimed: ['crave2'], gameBest: 21
    }
  },
  coins: 150, gems: 5, xp: 0,
  life: { harvests, clears, collects },
  town: {
    seed, nextId,
    objs: [{ id, type, x, y, day, ts, cig, ups: [{ day, cig, at, done }], col, crop: { type, at, wd } }],
    obstacles: [{ id, type: 'litter'|'rock'|'stump'|'bush', x, y }]
  },
  badgesSeen: [...], lastSeen: '2026-10-26', dropDismissed: ''
}
```

**Migration:** `load()` turns any pre-v3 save into v3. It keeps `days` and `settings`, starts a fresh island, and marks old badges as seen without announcing them. **If you change the shape again, bump `v` and add a branch there.**

## Common changes

**Add a building**
1. Add an entry to `DEF`, for example: `{ name, cat: 'build'|'farm'|'decor', w, h, ht, cig?, coins?, gem?, lvl, rate?, capH?, pop?, upBase?, max?, desc }`.
   - `w`/`h` are the footprint in tiles.
   - `ht` is the height in world units, used for tap targets, scaffolding and bubbles.
   - `rate` is 🪙 per hour and `capH` is how many hours it can store.
   - `max: 1` means it can't be upgraded.
2. Write `function sMyThing(c, o, f) { … }` using the drawing kit. `f.lvl` is the current level, `f.night` is true at night, and you push glow spots onto `f.lights`.
3. Register it in `SPRITES`. The shop, placement, previews and info sheet then pick it up automatically.

**Add a crop:** add an entry to `CROPS`, then add a drawing branch in `crop()` (in the Sprites section).

**Add a quest:** add an entry to `QUEST_POOL` with `prog: d => …` reading the day's counters. If you need a new counter, increment `today().q.myCounter` wherever the action happens.

**Tune the balance:** these all live near the top of the script:
- `xpTo(l)`: XP curve
- `landSide(l)`: land growth
- `UP_TIME`: upgrade timers
- each `DEF` entry's `rate`, `capH` and costs
- `upCost()`
- each `CROPS` entry's time and yield
- `OBST` costs and rewards

### Drawing coordinates

- Isometric world units: a tile is 64 × 32. `P(i, j)` gives the world point at the *top corner* of tile `(i, j)`.
- `corners(x, y, w, h, inset)` returns the footprint's T, R, B and L points.
- `upz(p, z)` raises a point by `z`.
- **Faces:** the left face (L→B) is the lit side and the right face (B→R) is the shaded side.
- **Depth sort:** objects sort by `x + y + (w + h) / 2` and people by `u + v`.
- The ground is pre-rendered into an offscreen canvas and only rebuilt when the land size changes.

## Debugging in the browser console

The script is a classic `<script>`, so top-level names (`state`, `save`, `today`, `DEF`, `cam`, …) are reachable from DevTools.

```js
state.coins += 5000; state.gems += 50; state.xp += 2000; save(); renderHud()   // give yourself resources
Date.prototype.getHours = () => 22                                              // see night mode (20:00–06:00)
state.lastSeen = shiftKey(todayKey(), -1); save(); location.reload()           // trigger the morning summary
objs().forEach(o => (o.ups || []).forEach(u => u.done = 0)); save()            // finish all upgrades
objs().forEach(o => o.crop && (o.crop.at = 0)); save()                         // ripen all crops
localStorage.removeItem('stubItOut.v1'); location.reload()                     // full reset (shows onboarding)
```

## Testing

There's no test suite in the repo. Everything was verified by driving the page with Playwright in headless Chromium at a phone-sized viewport and checking screenshots and console errors. To do the same locally:

```bash
npm i -D playwright && npx playwright install chromium
```

```js
// smoke.js — node smoke.js  (with the static server running)
const { chromium } = require('playwright');
(async () => {
  const b = await chromium.launch();
  const p = await b.newPage({ viewport: { width: 390, height: 844 }, deviceScaleFactor: 2 });
  p.on('pageerror', e => console.log('ERROR', e.message));
  await p.goto('http://localhost:8000/stub-it-out/');
  for (const s of ['[data-next]', '[data-next]', '[data-next]', '[data-done]', '#m-help .btn.g']) await p.click(s);
  await p.click('#bShop'); await p.click('#shopItems [data-t="teepee"]'); await p.click('#placeOk');
  await p.click('#lockBtn'); await p.click('#cfBtns .btn.g');
  console.log('🚬', await p.textContent('#rCig'), '| lock:', await p.textContent('#lockBtn'));
  await p.screenshot({ path: 'smoke.png' });
  await b.close();
})();
```

Things worth re-checking after any change to the rules: build → lock in → log one more (it should ruin the day) → undo (it should all come back), plus loading an older save.

## Known limitations and ideas

- **Saves are per device.** It's `localStorage` only, with no accounts or sync. That's the big gap for a multi-user app. A backend (Supabase/Firebase) or at least export/import of the save would be the next step.
- **Not installable yet.** There's no web app manifest or service worker, so there's no "Add to Home Screen" icon polish or offline support. Adding a PWA manifest and icon is a small, high-value change.
- **One big file.** That's fine for now. If it keeps growing, split it into `styles.css`, `game.js` and `sprites.js`. GitHub Pages serves them as-is, so there's still no build step needed.
- **Emoji vary by platform.** The HUD and bubbles use emoji (🚬 🪙 💎), which look different on iOS, Android and Windows.
- **Days follow the device clock and time zone.** The day boundary is local midnight. Changing the clock can game it, which is acceptable for a self-help app.
- **The 🚬 balance can go negative** if someone lowers their *baseline* in Settings after building. `settleCig()` only takes back today's builds. The HUD shows the negative number honestly.
- **Performance.** The forest is drawn every frame with view culling. On very old phones, consider baking the far forest into the ground cache.
- **Accessibility.** The town is a canvas. All the actions are real buttons in the HUD and sheets, but the map itself isn't screen-reader friendly.
- **No sound.** Collect, build and burn sound effects would add a lot of feel.
- **Content ideas:**
  - more building tiers, and visible upgrade differences on every building
  - seasonal events
  - a weekly "smoke-free Sunday" challenge
  - sharing a town snapshot
  - a nicotine-replacement or support-service link per country
