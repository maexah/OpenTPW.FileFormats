---
title: Procedural Ground (base.lnd)
---

A `base.lnd` is **the source of a park's composited ground**: a map saying where the ground is composited and how
much of each layer shows there, and the textures the layers are made of. The game cuts one small tile a ground cell
out of it, blending up to thirteen textures (OpenTPW's `docs/exe/park-engine.md`, "The procedural ground", says how).
It is not the heightfield: the heights are a block inside `base.MD2` ([models](/formats/models/)).

## Verified against every file

There are **4** `base.lnd` files, one in each theme's `terrain.wad` (jungle 2,417,116 bytes, fantasy 2,196,802,
hallow 2,447,361, space 2,241,690); no other wad holds a `.lnd`. Each is read to its last byte by the layout below,
with no remainder. The layout is the reader's (`FUN_00560ff0`, `FUN_0055f470` and `FUN_005e78b0` in the 2.0
executable), and the game's own tile compositor, run on each file under an emulator, makes the tiles the running
original makes from it (OpenTPW Q483).

## Layout

All values are little-endian.

| Offset | Size | Field |
| ------ | ---- | ----- |
| `0x00` | 1 | Version: always `3`; the game refuses any other |
| `0x01` | 10 | Five ramp pairs, each a centre byte then a width byte (below) |
| `0x0B` | 12 | The map's size: width 384, height 344, depth 4, as three `uint32` |
| `0x17` | 528,384 | The map: 344 rows of 384 samples of 4 bytes |
| then | ... | Three texture sets, for 64-, 32- and 16-pixel tiles in that order |

A texture set is a `uint32` tile size (64, 32, 16), one byte (0, 1, 2), then thirteen textures. Each texture is three
`uint32`, width, height and depth (bytes a texel), then width x height x depth bytes, row by row. In every file the
first six textures of a set have depth 4 and the last seven depth 1, and each set's textures are the 64-pixel set's
halved and halved again, rounded down (jungle's first: 212, 106, 53 pixels square).

| In the file | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| The game's slot | 0 | 1 | 3 | 4 | 8 | 9 | 2 | 5 | 10 | 6 | 7 | 11 | 12 |
| Depth | 4 | 4 | 4 | 4 | 4 | 4 | 1 | 1 | 1 | 1 | 1 | 1 | 1 |

The slots pair up: colour textures 0 and 1 mixed by grey 2, 3 and 4 by 5, 8 and 9 by 10; greys 6 and 7, and 11 and 12,
are the two masks that lay the second and third pairs over the first. A colour texel's bytes 1, 2 and 3 are red, green
and blue; byte 0 is nought in every file.

## The map

Four samples to a cell along each axis: cell column x uses sample columns 4x to 4x + 4, and **the rows run the other
way from the park's**: map row y uses sample rows 4r to 4r + 4, r = 84 - y. Cell column 95 is composited too, and
its fifth sample column is the next sample row's first: the game reads the map as one run of rows. A cell is
composited only where byte 0 of its four middle samples is 255; in the jungle, fantasy and hallow files byte 0 is only
255 or nought, and in the space file 58 samples hold 1 to 30, which refuse a cell as nought does. In the jungle file the
refused cells are 1,153 cells the scenery draws (the river, the approach and the cells of the stock park's mType 7;
OpenTPW Q488), and every path cell is composited. Bytes 1, 2 and 3 are the three weights the layers are mixed by.

## The ramps

Each pair (centre c, width w) makes a 256-entry ramp the game looks weights up in: nought below c - (w + 1) / 2, 255
above c + (w + 1) / 2, and `(i x 255 - lo x 255) / (w + 1)` between, lo being the lower end (integer division, halves
dropped). Jungle's five pairs are (48, 40), (40, 90), (68, 40), (97, 30), (55, 50).
