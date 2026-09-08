# airhockey — band site + 8-bit game

Sebastian's band site, plus **Airhockey Soundcheck**, a 2D 8-bit game that lives on
it. Two hand-written files are the entire project; everything else is generated.

## Source of truth

| File | What it is |
|---|---|
| `index.html` | The band site. Already links to the game — "play ▸" in both the desktop nav and the mobile menu, so a game change needs no porting. |
| `game.html`  | The game. One self-contained file: canvas 2D, hand-coded pixel-art sprites as literal pixel-row strings, no image assets, no build step. |

The only asset either page loads that is not an image is `music/felt-8bit-loop.m4a`,
the soundtrack — see **The soundtrack** below before touching it.

**Never hand-edit these** — they are rebuilt from the two files above and any edit
is silently thrown away on the next sync:

- `.claude/game-artifact.html` → the published game Artifact
- `.claude/site-artifact.html` → the published site Artifact
- `airhockey-site.zip` → the upload bundle for a real host

## Start of every chat

`python3 .claude/status.py` runs automatically (SessionStart hook in
`.claude/settings.json`) and prints whether source, builds, and the two published
artifacts agree. **Trust that banner over anything you remember from an earlier
conversation** — Sebastian edits these files, and the published artifacts, outside
of chat. If it reports anything STALE, resolve that before starting new work.

## After ANY edit to `index.html` or `game.html`

1. `python3 .claude/sync-all.py` — rebuilds all three generated copies and fails
   loudly if any drift.
2. Republish **both** artifacts to their existing URLs in
   `.claude/artifact-links.json`. Publishing without passing the URL mints a *new*
   artifact instead of updating, which breaks the cross-links between them.
3. `python3 .claude/status.py --mark-published` — records what actually went live.

   The site's fingerprint covers `index.html` **and the content of every asset the
   build inlines** (`site_fingerprint()`). It has to: hashing the markup alone once
   reported "everything matches" straight after all three member photos were replaced,
   because the markup never moved — the published site would have kept the old faces
   indefinitely. Hashing by content also catches a re-crop that reuses a filename.

   One wrinkle worth recognising rather than re-deriving: an edit that only touches
   the **`<head>`** of either page (SEO, OpenGraph, canonical) changes the source
   fingerprint but **not** the built artifacts, because both build scripts strip the
   head and write their own. The banner will say STALE while the live artifacts are in
   fact byte-identical to a fresh build. Diff the built file before assuming a republish
   is owed; if it is unchanged, stamping alone is the honest move.

Step 3 is the whole reason a later chat can tell current from stale. Skipping it
makes every future session report the artifacts as out of date.

4. `git add -A && git commit` — the site is a git repository now (see **Git and
   GitHub Pages**). Only the website is tracked; the toolchain and `memory/` are not,
   so a commit here is always a site change and never 79 MB of history.

5. `python3 .claude/snapshot.py` — optional, but this is the only real safety net.
   It freezes source, both published builds and the upload zip into `memory/` under one
   timestamp and logs it in `memory/restore-points.md`. It refuses to run if anything
   is stale, so a snapshot is always of a coherent state.

**Publishing is not the same as sharing.** Both artifacts are shared by link, and a
link viewer keeps seeing the version the *share pin* points at — a new publish does
not move it. **There is no way around this**, and it has been asked for: a second
artifact that links to the first does not help, because viewers of that link still land
on the first artifact's pinned version, and the artifact CSP blocks framing one page
inside another. The only setup where a URL always serves the latest build is a real
static host fed `airhockey-site.zip` — which is what that zip is for, and which would
also finally give the project a canonical domain. After republishing, Sebastian has to move the share pin from each
artifact's own share menu, or everyone with the link still sees the old build.
Also keep each artifact's favicon emoji stable; the ones that went live are recorded
under `favicon` in `.claude/artifact-links.json`.

Both artifacts must go together: they cross-link to each other the way `index.html`
and `game.html` do locally, so republishing one alone leaves a half-updated site.

## The artifact is sometimes the newer copy

Sebastian also edits the **published artifacts directly** and then asks for those
changes to be pulled back down — the reverse of the normal direction. An artifact is
a *build*, not source, so converting one back into `index.html` / `game.html` means
undoing three things or the site quietly regresses:

1. **Re-externalise images** — the build inlines every local image as a `data:` URI;
   map them back to `art/…`, `members/…`, `photos/…`. Leave `data:image/svg+xml`
   alone — those are the film-grain noise filters and belong inline.
2. **Fix cross-links** — artifacts link to each other by artifact URL with
   `target="_top"`; locally these must be relative.
3. **Restore SEO metadata** — the build drops the full `<title>` and ~14
   description / keywords / author / theme-color / OpenGraph / Twitter tags. Recover
   them from `memory/archive/`, and watch for the resulting duplicate `<title>`.

Also strip the Claude `frame-runtime` wrapper and the trailing `</body></html>`.
**Archive the outgoing local files first**, into `memory/archive/<timestamp>/`.

## Verifying a change — there is no working browser here

macOS TCC blocks the preview helper from reading `~/Desktop`, and both browser panes
refuse `http://localhost`. A plain `python3 -m http.server 4321` from Bash does work
for Sebastian to look at by hand, but it dies whenever the shell resets.

The whole suite lives in `.claude/test/` and runs with one command:

```bash
python3 .claude/test/run-all.py          # --quick skips the slow WebKit checks
python3 .claude/test/page.py             # the site, separately — index.html in a real WebKit
```

`page.py` is the site's half and is **not** part of `run-all.py`, which is the game's.
It stages `index.html` and every asset folder into `/private/tmp` first — TCC blocks
the WebKit helper from reading `~/Desktop`, and WebKit's own sandbox cannot read the
per-user `/var/folders` tree that `tempfile` defaults to, so either mistake gives you a
page that loads completely blank with no error.

It covers: the script parses; the shipped maps flood-fill (every elite reachable
while its gate is shut, no `D` reachable until it opens, no walled-off floor); the
clock beats the walking distance; ~70 gameplay assertions; the touch stick's eight
directions; every sprite's row widths; both screen layouts measured in a real
WKWebView; and a pathfinding bot that plays all six rooms through to the finale.

**Keep it current.** When you add a mechanic, add its check — that is the only
reason the suite has ever caught anything. The pieces, if you need to iterate on one:

| File | What it does |
|---|---|
| `syntax.py` | pulls the last `<script>` out and parses it with `osascript -l JavaScript` |
| `runtime.py` | runs the game headlessly in JavaScriptCore, stepping frames by hand |
| `checks/*.js` | the assertions `runtime.py` drives — gameplay, controls, playthrough |
| `levels.py` | **authors** the maps and validates them; STAGE_DEFS is generated from it |
| `recheck.py` | re-validates the maps parsed back out of the shipped `game.html` |
| `sprites.py` | composes and width-checks the 20x24 elites |
| `budget.py` | BFS walking distance per room vs. that room's clock |
| `mkdebug.py` | writes `.claude/test/game.html`: the game plus a `window.__game` hook |
| `shot.js` | WKWebView screenshotter — `canvas.toDataURL()` to a PNG |
| `rescale.py` | resamples a capture to a real phone width, with no image library — this is how the floor-banding bug was found |
| `layout.js` | measures the wide and 9:16 portrait layouts in a phone-sized webview |
| `page.py` / `page.js` | the **site**, not the game: loads `index.html` in a real WKWebView and reports broken images, store-tile layout, release order, every bigcartel link, and any asset that ships without being referenced |
| `music.py` | the soundtrack asset: bar alignment, the level either side of the join, and that the head is not silent |
| `sfx.py` | **authors** all five audio files from the archived master; `--check` validates what shipped |
| `audio.py` | decodes the artifact's inlined audio in a real WKWebView and reads the loop points back |
| `checks/music.js` | the wiring: gain, the mute key, and `loopWindow()` against synthetic buffers |

The traps behind all of this, each of which silently yields a blank or wrong result:

- `new Function(src)` returns only what `src` **returns** — the headless driver's last
  line must be `return JSON.stringify(...)`, not a bare expression. And never pass
  stubs as parameters named `document` / `window`; JXA's own globals shadow them, so
  inline the stubs into the source.
- A WKWebView must be the content view of an `NSWindow` **ordered front**, with the
  run loop pumped by `runUntilDate`, or the page never loads at all.
- Off-screen, `requestAnimationFrame` is throttled to nothing, so screenshots pump
  frames through the debug hook's `render(n)` — which muzzles rAF while it steps,
  because `frame()` reschedules itself and would otherwise leave one live loop per
  pumped frame.
- A `file://` logo taints the canvas and makes `toDataURL` **throw**, so `mkdebug.py`
  inlines it as a data URI the way the artifact build does.
- Hand-counted pixel rows have been wrong more than once. Compose sprites in
  `sprites.py`, which asserts row widths, rather than typing strings.
- **`fetch()` cannot read a `file://` sibling** in WebKit, so no local page can be made
  to decode `music/…` in a webview. `audio.py` gets around this by driving the *artifact*
  build, where the audio is already a `data:` URI. And the decoder hands the buffer back
  at the **output device's** sample rate, not the file's — 48 kHz here for a 44.1 kHz
  file — so never assert the rate that comes out of `decodeAudioData`.
- The debug hook's `render(n)` pumps frames **ahead of the wall clock**, so the next
  call can start with `performance.now()` behind `last` and hand `frame()` a negative
  `dt`. That runs every effect timer backwards, and an fx radius derived from one goes
  negative and makes `ctx.arc` throw `IndexSizeError` — which surfaces as a screenshot
  that fails only at certain frame counts. `frame()` clamps `dt` at **both** ends now;
  keep it that way.

## How the game is put together

- **Tile alphabet** (map rows, 40x24): `#` wall  `.` floor  `~` rug  `T` counter
  `A` amp  `c` crate  `r` rope pole  `b` barricade  `s` PA stack  `p` planter
  `g` guitar stand  `y` keyboard  `k` drum kit  `i` mic stand  `o` bar stool
  `B` back bar  `j` jukebox  `x` candelabra  `f` flyer wall  `t` toilet
  `w` washbasin  `S` range/stove  `F` fridge  `P` pot rack
  `G` security gate  `D` exit door. Everything but `.`, `~`, `G`
  and `D` is solid. Add a character in three places: `SOLID`, the `drawMap`
  dispatch, and `levels.py`.
- **The gear moves every run.** `rollItemSpot()` re-rolls the item's tile in
  `loadStage`, so `stage.itemCol/Row` is *not* `def.itemCol/Row` — the authored pair is
  only the fallback and the thing `levels.py` flood-fill proves. A candidate has to be
  reachable from the spawn with the gate **shut** (only `.` and `~` are walked, so `G`
  blocks — this is what keeps the item out of the exit vestibule), at least
  `ITEM_MIN_TILES` away, and not on the elite's doorstep. `budget.py` measures the
  **worst** tile the roll can produce, not the authored one, which is the only figure
  that says anything about the clock.
- **Exits are multi-tile.** `doorTilesOf()` takes the bounding box of every `D`, so a
  doorway is three tiles wide; `stage.doorCol/Row` is only a label.
- **One clock for the whole venue.** `RUN_TIME` (currently 360s, shown as `6:00 min`)
  is set in `resetGame()` and never reset per room; `loadStage` deliberately leaves it
  alone. Each elite adds its `bonus` on death, which is what makes six mandatory
  fights affordable — an invincible perfect-pathing bot takes ~155s, so the clock
  leaves a person roughly 2.9x that. Running out ends the run rather than restarting
  the room. `budget.py` reports the whole-run figure.
- **One elite per room, and it is the room's lock.** `BOSS_KINDS` holds seven: the
  doorman, attendant, bartender, line cook, roadie and security boss all carry
  `gates: true`, so `gatesOpen()` is simply "is this room's elite still standing".
  The ladder is `hp` 4/6/8/10/12/14 and `charge` 70/84/100/114/126/132 — deliberately
  soft at the start, and **frozen at the top**: the security boss's numbers are
  asserted in `checks/gameplay.js` so a rebalance cannot quietly move the final fight.
  The wild drummer is the exception — `gates: false`, optional, and only changes the
  finale. He is also the only elite with a **body of his own**: `w: 16, h: 20` on his
  entry in `BOSS_KINDS`, which `makeBoss` prefers over `BOSS_W`/`BOSS_H`. He shares
  the backroom with the roadie, so he is smaller, slower and quicker to put down than
  any gating elite; the hitbox and the sprite have to stay the same size, and
  `checks/gameplay.js` asserts they do.
  They share one state machine (wander → charge → special) and differ only in
  their special: nothing, `spill`, `throw`, `lash`, `rings`. Every one pays out
  `bonus` seconds on the clock when it goes down, which is what makes a mandatory
  fight per room affordable.
- **`odds` and `rest` throttle a special.** Both live on the `special` object and both
  default to firing every time (`odds: 1`, `rest: 1`). `odds` is the chance an elite
  in range actually unloads rather than just charging; `rest` is how long he stands
  off afterwards. Only the bartender and the line cook use them — unloading every
  single time you closed made those two read as harder fights than the chief two rooms
  later. The chief is deliberately **not** throttled, and a check says so.
- **The gear is a ladder, and every rung has to beat the last.** Pickup order is
  `strings → cable → pedal → pick → mic → guitar` (`WEAPON_LADDER`, and the six rooms
  hand them out in exactly that order). Each step is strictly more damage, a strictly
  shorter cooldown and strictly more knockback — no sidegrades, so a piece you just
  found is never worse than the one it replaced. The `dmg` values are **fractional on
  purpose**: elite health only climbs 4 → 14 across the whole venue, so stepping damage
  by a whole point per tier would double the ladder's output halfway up and leave the
  chief dying faster than the doorman. `checks/gameplay.js` asserts both halves — the
  four monotonic runs, and that every room's own gear against that room's own elite
  stays inside a 1.0–2.5s band of contact time. Raise a weapon and the band check is
  what will catch you.
- **The cable is a whip, not a coil.** It throws out along the facing on a sine ease
  over the first third of its life and then recoils — there is no rotation left in it.
  The lead bows sideways while travelling and straightens as it lands, which is what
  reads as a crack rather than a poke; at full stretch it throws a ring and three sparks
  off the plug. Two earlier passes are worth not repeating: a bare semi-circle read as a
  scythe, and the helix that replaced it read as a spring being wound. It is outlined in
  **white** under a dark core, because this is the one weapon carried through several
  rooms and the black cord vanished into the late floors. The hit box and damage are
  untouched from the sweep it replaced — if the snap should reach further than the old
  arc, that is a balance change and `checks/gameplay.js` will hold you to the 1.0–2.5s
  contact band.
- **Unarmed, you shove.** `SHOVE` is knockback and nothing else — it never touches
  `hp`, so it cannot kill. It needs no stage check: gear is never taken away once you
  have it, so the only time you carry nothing is the front entrance before the strings
  turn up. It shares `reachBox()` with the melee weapons. The half of this that is easy
  to miss is in `updateEnemy` — `en.knock > 0` now short-circuits the targeting chain,
  because without it enforcers and bouncers re-aim on the very next frame and a push
  moves them precisely nothing.
- **Three things can be on the floor after a fight**: `hazards` (soap you slide on,
  glass and grease fire that bite), `bolts` (thrown), `lashes` (the roadie's cable,
  a hit-scan not a projectile). Player damage all funnels through `hurtPlayer()`.
- **A tile renderer must not paint outside its own 16x16.** `drawWallTile` courses
  bricks from `-4` and each brick is 7 wide, so before it clamped them it painted up to
  4px left and 3px right of the tile, over whatever sat beside it. Against floor — which
  is exactly what a wall butts onto at a gate alcove — that read as colour smearing out
  of the outlines. Every brick is clipped to `[0, TILE]` now; anything new that draws a
  pattern with an offset needs the same treatment.
- **The objective banner** is the big glowing line under the HUD. `objectiveText()`
  names the item, then the room's elite, then the door. There is deliberately **no**
  off-screen pointer — finding things is meant to be the work. Landmarks mounted on a
  room's top wall must sit below world y ≈ 28 or the banner covers them.
- **Mel is not a companion.** She runs the room on her own (`updateMel`), faster and
  further away each stage, and drops her **bass boost** wherever she happens to be.
  She leaves fading footprints (`mel.trail`, alternating left/right along her heading,
  dark-cored so they hold on pale tile) — a lead to follow, not a marker. The
  drawn circle of the boost is **exactly** the area that heals; an earlier version
  drew a halo well past `MEL_ZONE_RADIUS` and made the heal feel centre-only, so
  never draw a ring outside that radius. Mike is found in **level 2** so the two
  never arrive in the same stage; he still trails the player off `companionPos()`
  and is a missable walk-over pickup by design.
- **The finale is staged, not static.** `winClock` counts seconds since the lights
  came up (`update()` advances it; `resetGame()` clears it). Two optional band members
  give it **four endings**, all routed through `finaleLine()` so they can be asserted
  without a canvas. The drummer either sits on the back riser or the riser is set and
  empty — `drawEmptyStool()` is the alternate, and it has to stay visibly a different
  stage rather than the same one with a different caption. Mike's place in the line is
  **always reserved**: found him and he is there from frame one, missed him and he
  jogs in from off the right edge at `MIKE_LATE_AT` over `MIKE_WALK_TIME`. Every
  ending line is kept under ~28 characters because `drawFittedCenter` shrinks to fit
  rather than wrapping and the portrait screen is only 180 wide — there is a check.
- **The drummer's sticks are drawn in three passes.** Sprite, then the kit over him,
  then `drawDrumsticks()` over the kit. They are **static** — they used to bob on a
  wall-clock sine, which read as a twitch rather than a performance — and they carry no
  outline, because the dark backing behind them read as a smudge at this size. `DRUMMER_SEATED` strips the `Y` pixels out of
  `DRUMMER_SHAPE` first, or he ends up holding four sticks. Do not draw a forearm —
  the sprite already has arms, and a drawn one grows out of his jaw.
- **The guitar and bass are pixel sprites**, `GUITAR_SHAPE` (27x11) and `BASS_SHAPE`
  (31x11, the same instrument with four more frets), drawn from Sebastian's reference:
  an offset solid body with horns, a fretted board, a pickguard and a bridge. They used
  to be composed from three ellipses, which is an acoustic silhouette however you tune
  it. `sprites.py --check` asserts every row's width and that every character has a
  palette entry. Two things the screenshots caught and code review would not: at full
  band scale the sprites are **taller than the people holding them**, so they draw at
  ~0.7 of it; and the reference's charcoal fretboard is invisible against the dark
  stage, so `n` is warm rosewood here. `hang` survives from the old version and is still
  load-bearing — Mel's face is four pixels wide, so a neck on centre covers a lens of
  her glasses.
- **The bathroom is deliberately dim.** It used to be near-white and glared next to
  every other room, and nothing dropped on that floor stood out. The whole `bathroom`
  theme sits about 20% darker now, and the cable's plate is yellow (`ITEM_COLOR.cable`)
  because a cold blue room is the one place a warm plate reads from across the map.
- **Two layouts, one game.** `LAYOUTS` holds `wide` (320x218, desktop) and `portrait`
  (180x224 canvas + a 96-unit control strip, so screen + controls is exactly 9:16 on
  a phone). `applyLayout()` switches on `(pointer: coarse) and (orientation:
  portrait)` and rebuilds the CRT overlay; `BASE_W`/`BASE_H`/`HUD_H`/`VIEW_W`/`VIEW_H`
  are all layout-derived, so **never hard-code 320 or 218** in drawing code. Portrait
  is the same map at the same tile size through a narrower window — a cut-out, not a
  redraw.
- **Portrait sizing is CSS, not whole-number scaling.** The block's width is
  `min(100vw - 8px, calc((100dvh - 46px) * 9 / 16))` and the canvas stretches to fill
  its 70% share. The first build snapped to integer scales instead, and because a
  phone showing browser chrome is usually under ~710px tall it fell to 1x and opened
  at half size. If you are ever tempted back to integer scaling here, `layout.js`
  sweeps five phone viewports — including a short, chrome-showing one — and will fail.
- **The canvas is drawn oversize on purpose.** `setBacking()` sizes the backing store
  to `BASE_W x renderScale` where `renderScale = ceil(cssWidth / BASE_W)`, and
  `frame()` opens with `ctx.setTransform(renderScale, ...)`. Stretching a 1:1 canvas
  to a non-whole CSS size made the browser resample nearest-neighbour, and the 4px
  dither cells beat against that into diagonal stripes across every floor. Always
  render at least as many pixels as the screen shows and let the browser come *down*.
  Two rules follow: portrait CSS sets `image-rendering: auto` (forcing `pixelated`
  there is what caused the banding), and nothing may assume `canvas.width === BASE_W`.
  `rescale.py` reproduces the artefact from a capture if you need to see it again.
- **The objective banner wraps rather than shrinking away.** Anything that will not
  fit on one line at 8px breaks into two — that is what keeps `BEAT THE SECURITY
  BOSS` readable on a 180px-wide screen.
- **Touch is a swivel stick**, not a D-pad: the thumb angle snaps to one of eight
  compass directions and sets the very same `keys.up/down/left/right` flags the
  keyboard sets, so desktop behaviour is untouched by anything in that block.

## The soundtrack and the sound effects

`.claude/test/sfx.py` **authors all five audio files** from the archived master, the way
`levels.py` authors the maps. The files in `music/` are output — never hand-edit one,
and never swap one in without re-running the generator, or the loop drifts off the beat
with nothing to catch it.

- **The loop is the whole 47-bar master, wrapping at its true end** (135.090s, 12ms off
  the grid — inaudible). It used to be 46 bars with the final chord folded back over the
  first; Sebastian asked for the plain end instead. The catch that fold was hiding: a
  ringing chord meeting a sparse intro is a **+7.7 dB jump**. Measured against the first
  250ms of the track, a **250ms release** lands the tail within +2.2 dB of the intro —
  400ms overshoots to −3.5 dB. It costs a third of a beat of decay, and because it is a
  fade the file length and so the grid are untouched.
- **The effects are cut from the same master**, which is why a hit sounds like it
  belongs in the song. `sfx-hit` is the x518 attack at 11.494s pitched down; `sfx-collect`
  is the isolated intro blip at 0.720s played twice a fifth apart; `sfx-heal` is a chord
  from 32.9s an octave up, eased in and out. **`sfx-cheer` is the exception — it is
  synthesised**, filtered noise plus clap transients, because there is no crowd anywhere
  in the master and a real recording sits badly against square waves.
- **AAC, because this Mac cannot encode MP3.** No `ffmpeg`, no `lame`, and `afconvert`
  has no MP3 encoder. The 320 kbps master, the source WAV and a seam preview are in
  `memory/archive/2026-09-06_1435/original-music/`.
- **Web Audio, not `<audio loop>`.** A media element re-primes its decoder on every wrap
  and drops a hole in the join. Both pages decode once and loop one buffer; `loopWindow()`
  finds the real edges by amplitude, capped at `MUSIC_PAD`, so a decoder that ignores the
  MP4 edit list cannot introduce a gap. There is an `<audio>` fallback for `file://`.
- **The effects share the music's context, gain policy and mute.** `playSfx(name, rate)`
  returns immediately unless `music.on`, so the one ♪ button silences everything — which
  is what a player expects from a single control. `rate` detunes repeats so a run of hits
  does not machine-gun one sample.
- **The game plays it; the site is silent.** Game: gain `0.35`, starts on the first key
  or tap, `M` or the ♪ button mutes, remembered in `localStorage`. The site briefly had
  an opt-in toggle in the nav; Sebastian removed it on 2026-09-07, which took 2.16 MB
  out of the site artifact. **Which pages want audio is derived, never assumed** —
  `sync-all.py`, `music.py` and `build-site-artifact.py` all test `index.html` for the
  reference rather than hard-coding a rule, so putting it back is a one-file change and
  taking it away again does not leave a check asserting something stale. `page.py`
  asserts the reverse too: a toggle reappearing in the nav is a failure.
- **The build inlines every audio file**, globbed rather than listed, and `fingerprint.py`
  normalises every `music/*.m4a` and `data:audio/…` string so the artifact does not read
  as drift. `sync-all.py` then asserts each file individually — the game carries all five,
  the site carries only the loop — because none of it appears in the fingerprints and a
  silent build otherwise looks perfectly healthy.

## The site's live data

`index.html` carries hand-written copy, but three parts of it mirror things that live
elsewhere and go stale:

- **Latest release / discography** — from `https://airhockeyband.bandcamp.com/music`.
  That page is **hand-arranged, not date-ordered** (it puts *hazel* above *little
  friday*, which is wrong by three months), so take the date off each track page and
  sort by that. The site was curated for a long time; as of 2026-09-03 it is the
  **complete catalogue**, all 16, newest first: felt (7 Aug 2026), OK (I Got You)
  (1 Jul 2026), Hazel (7 Nov 2025), Little Friday (17 Jan 2025), Oscuridad (26 Apr
  2024), Something (23 Feb 2024), Always Late (14 Apr 2023), Close (20 Jan 2023),
  Grapefruit (1 Apr 2022), Walkthrough EP (17 Jul 2020), Not Enough (26 Jun 2020),
  Try (5 Jun 2020), Just Not Fair (17 Apr 2020), The Signs (4 Oct 2019), hidden by
  the sun (7 Jun 2019), you don't mind (23 Jan 2019). Newest as of 2026-09-06 is
  **nice day (4 Sep 2026)**, which is 17 with the rest. The grid is newest-first and
  the "latest release" block at the top of the page has to show whatever sits first in
  it — adding a release and forgetting to re-feature it looks fine until someone reads
  the date, so `page.py` asserts the two match.
- **Store** — the real shop is `https://airhockey.bigcartel.com`. **No `band`** — the
  `airhockeyband.bigcartel.com` subdomain the site used to link 404s on every path,
  which silently broke all six store links. Verify with `curl -o /dev/null -w '%{http_code}'`
  before believing either. The five products and prices on the site are the real ones,
  each linking to its own product page, and their photos live in `merch/`: cut-out
  **PNGs with real transparency** (palette index 0 has `alpha 0`), so they keep the
  pastel `.product-art` tile showing through and must not be flattened to JPEG.
- **Band photos** — `photos/` is **globbed into the upload zip**, so it holds only what
  `index.html` actually references; unused frames belong in `memory/archive/<stamp>/
  original-photos/`. The three member headshots in `members/` are 400px crops taken out
  of one group shot (`original-photos/band-bench.jpg`) with
  `sips -c H W --cropOffset Y X` — that offset is the crop's **top-left corner** in
  source pixels, except that `0 0` is special-cased to mean centred. `page.py` checks
  each headshot is at least twice its rendered circle, since a crop that is too small
  goes soft rather than failing. As of 2026-09-07 the three are 400px crops cut from
  2048x2560 live shots in `memory/archive/2026-09-07_2047/original-members/`, replacing
  night-shot crops with laughing expressions. That check earned itself the same day:
  Sebastian first sent phone screenshots of the frames he wanted, 238-282px, and it
  refused all three rather than shipping faces that upscale.
- **Sleeve art** — every release on the site has real art in `art/`, named
  lowercase-hyphenated to match its Bandcamp slug, 700px square, ~60-140KB. New art
  arrives full-size; run it through `sips -Z 700 -s format jpeg -s formatOptions 78`
  and keep the original under `memory/archive/<stamp>/original-art/`. **macOS is
  case-insensitive**, so `sips` reading `art/Felt.jpg` and writing `art/felt.jpg` is
  reading and writing the same file — go via a temp path. Remote art can never be
  hotlinked: the artifact CSP blocks it, which is why the build inlines every local
  image, and why `sync-all.py` globs `art/`, `members/`, `photos/` and `merch/`
  rather than listing them (new art used to work locally and silently miss the upload
  zip). A new asset folder has to be added to `IMAGE_DIRS` — or `MEDIA_DIRS`, which is
  the same idea for `music/` — or it ships from nowhere.
  Afterwards run `python3 .claude/test/page.py`, which is the only thing here that
  can tell you an image reference actually resolves.

## Git and GitHub Pages

The project became a git repository on 2026-09-06, on `main`, so that a URL can serve
the current build instead of the version an artifact's share pin happens to point at.

- **The repository root is the site root.** Every path in both pages is relative, so
  the site works unchanged at a user-site root, at a `/repo/` project-site subpath, or
  from a local `python3 -m http.server`. Never introduce a leading-slash asset path —
  it works at the root and breaks under a project subpath.
- **`.nojekyll` skips the Jekyll build.** Nothing here needs Jekyll, and skipping it
  means a stray `{{` or `{%` inside the game's script can never be eaten as a Liquid
  tag. Do not delete it.
- **Only the website is tracked.** `.gitignore` excludes `.claude/`, `memory/` and
  `airhockey-site.zip`. That keeps the repository at ~5 MB and every commit a real site
  change — but it also means **the build scripts and the test suite are not backed up by
  git**. Their only history is `memory/` on this Mac. That is a deliberate trade, not an
  oversight; if it ever needs reversing, drop the `.claude/` line and add
  `.claude/*-artifact.html` instead, which is where the 9 MB actually is.
- **The upload zip is still the right thing for any non-Pages host.** `sync-all.py`
  builds it from the same globs, so the zip and the repository always hold the same
  files.
- **Publishing to Pages does not replace the artifacts.** They remain the shareable,
  self-contained copies; the repository is the always-current one. Both still have to be
  kept in sync by the loop at the top of this file.

Enabling it, once a remote exists: repository **Settings -> Pages -> Deploy from a
branch -> `main` / `/ (root)`**. There is no build step to configure.

### The custom domain

`CNAME` in the repository root holds `airhockeymusic.com` — the bare apex, no `www`.
GitHub Pages reads that file to decide what host to serve, so **deleting it un-sets the
custom domain**, which is the classic way a Pages site silently reverts to `github.io`.
`sync-all.py` does not glob it into the zip either, so it is a repository-only file.

The Pages host is `airhockey-music.github.io`. DNS lives at GoDaddy and needs **both**
halves, even though only the apex is canonical: four `A` records on `@` pointing at
GitHub's addresses (an apex cannot be a CNAME, and GoDaddy has no ALIAS/ANAME), plus a
`CNAME` on `www` pointing at the Pages host so that spelling still resolves and Pages
redirects it to the apex. Which way that redirect runs is decided by the one field in
**Settings -> Pages -> Custom domain**, not by the records. Enforce HTTPS only after the
certificate has been issued, or Pages will report the domain as unverified.

## What the live site can and cannot be hardened with

GitHub Pages serves static files and **does not let you set response headers**. That
rules out HSTS, `X-Frame-Options` and a header-based CSP no matter what you read
elsewhere — `curl -sI https://airhockeymusic.com/` comes back with essentially nothing
but `server: GitHub.com`. What *is* available, and the state as of 2026-09-07:

- **Enforce HTTPS** — on. `http://` 301s to `https://`.
- **A `<meta http-equiv="Content-Security-Policy">` tag** is the only CSP route. Both
  build scripts slice from `<style>` onward, so a meta tag in either `<head>` reaches the
  real site and is correctly absent from the artifacts. Not currently added; it needs
  `'unsafe-inline'` for both style and script because the pages are self-contained, so
  the win is blocking external script loads, framing and form posts rather than XSS.
- **No CAA record** is set. The domain is on Let's Encrypt via Pages.
- **No MX and no SPF**, so no mail is sent from this domain; DMARC is `p=quarantine`.
- Every one of the 36 `target="_blank"` links already carries `rel="noopener"`.

## Project facts worth not re-deriving

- **The canonical domain is `https://airhockeymusic.com`** (Sebastian's, at
  GoDaddy, as of 2026-09-06). The bare apex is canonical; `www` redirects to it. Both
  pages now carry `rel=canonical` and `og:url`; `og:image` and `twitter:image` are
  **absolute** URLs on that domain, which is what makes a link preview show an image at
  all — they were relative before and every scraper ignored them. Any new share tag has
  to be absolute for the same reason.
- `memory/` in the project root is version history, not code: `archive/` (previous
  source), `artifacts/` (published snapshots), `snapshots/` (full site zips).
  `memory/README.md` is its manifest and the restore instructions.
- Artifacts are one self-contained page under a CSP that allows only Google Fonts,
  which is why the build inlines images and swaps the Font Awesome icons for inline
  SVG, and why the `<!DOCTYPE>` / `<html>` / `<head>` / `<body>` wrappers are stripped.
- Sebastian's standing preferences for this project: ask questions along the way,
  work in stages rather than one huge change, and show a sample or screenshot before
  updating assets wholesale.
