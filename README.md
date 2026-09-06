# airhockey

The band site for **airhockey** — shoegaze / dream-pop out of Miami, FL — and
**Airhockey Soundcheck**, the 8-bit game that lives on it.

Two hand-written files are the whole site:

| File | What it is |
|---|---|
| `index.html` | The band site: releases, shows, the store, and the band. |
| `game.html` | The game. One self-contained page — canvas 2D, sprites hand-coded as pixel-row strings, no image assets and no build step. |

Everything else here is what those two pages load: sleeve art in `art/`, headshots in
`members/`, live shots in `photos/`, product cut-outs in `merch/`, and the soundtrack —
the 8-bit cut of *felt* — in `music/`.

## Running it

There is nothing to build and no dependencies. Open `index.html`, or serve the folder:

```bash
python3 -m http.server 4321
```

The game needs to be *served* rather than opened from disk to get its seamless
soundtrack: browsers block `fetch` on `file://`, so opening the page directly falls
back to a looser loop. Everything else works either way.

## Hosting

The repository root is the site root, so any static host serves it as-is — including
GitHub Pages, which is what `.nojekyll` is here for.

## Notes

The soundtrack is a **built loop**, not the song: 46 bars at 83.5 BPM, with the
master's final bar folded back over its first so it wraps on a downbeat rather than
cutting off a ringing chord. Replacing it means re-deriving that fold, not just
dropping in a new file.

The build scripts, test suite and local version history are deliberately not tracked
here — see `.gitignore`. This repository is the website.
