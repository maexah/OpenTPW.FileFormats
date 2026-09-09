---
title: Park signs (*.sgn)
---

A `.sgn` holds everything a park's name board needs: the artwork painted on the board, and the
fonts its name is lettered in. The name itself is not in here — it comes from the park's script
(see [Where the name comes from](#where-the-name-comes-from)).

Five ship with the game:

| File | Park |
| --- | --- |
| `lobby.wad` → `terrain\Jun_isle.sgn` | Jungle ("Lost Kingdom") |
| `lobby.wad` → `terrain\Hal_isle.sgn` | Hallow ("Halloween") |
| `lobby.wad` → `terrain\Fan_gate.sgn` | Fantasy |
| `lobby.wad` → `terrain\Spa_gate.sgn` | Space |
| `levels\<park>\features\sign1.wad` → `sign1.sgn` | the placeable in-park sign |

Note the inconsistent naming. A `.sgn` is named after **the model that carries the `sign1` and
`sign2` materials**, which is the island model for jungle and hallow but the gate model for
fantasy and space — so a lookup has to try both rather than assume either.

## File format

The header is a fixed size: the image body starts at `0x43DD` in all five files.

| Offset | Size | Description |
| --- | --- | --- |
| `0x0000` | 17 bytes | Unknown |
| `0x0011` | 436 bytes | Font record 0 |
| `0x01C5` | 436 bytes | Font record 1 |
| `0x0379` | 76 bytes | Unknown |
| `0x03C5` | 16384 bytes | 32-bit pixel data — see below |
| `0x43C5` | 4 x float | Y, Cb, Cr and A quantisation scales (`6, 10, 2, 6` in all five files) |
| `0x43D5` | 4 bytes | Colour chunk size |
| `0x43D9` | 4 bytes | Alpha chunk size |
| `0x43DD` | | Colour chunk, then alpha chunk |

### Font record

| Offset | Size | Description |
| --- | --- | --- |
| `+0x000` | 64 bytes | Display name, NUL-padded ASCII (e.g. `Young Itch AOE`) |
| `+0x040` | 64 bytes | TrueType file name (e.g. `YOUNIA__.TTF`) — the file itself is in `fonts.wad` |
| `+0x168` | 64 bytes | Display name again |
| `+0x18C` | 8 x float | Parameters — see below |

There is one record per line of the sign. Jungle letters its two lines in different fonts
(`Young Itch AOE` then `Clunker AOE`); Space names the same font twice.

Of the eight floats, indices 2, 3 and 4 are always in the range 0..1 and are the **text colour**
— jungle's first line is `0.40, 0.87, 0.31`, a green, and the four parks' values are distinct and
park-appropriate. The remaining five are not established. Index 0 ranges 1.75..13.5 and index 7
sits between 248 and 360, which would suit a size and an angle, but neither has been confirmed.

### The pixel data at `0x03C5`

16384 bytes, which is 64x64 at 32 bits per pixel. It decodes to a small tiling pattern rather
than to anything resembling the sign, so while it is clearly image data, its dimensions and
purpose are not established. It is **not** the sign artwork — that is the image body below.

## The image body

Everything from `0x43C5` is a [`.wct`](/formats/texture/) version 4 image with its header fields
rearranged: four quantisation scales, two chunk sizes, then the colour and alpha chunks, each
wrapped in the same `BILZ` zlib container a `.wct` uses. A decoder can be handed these directly.

The image is **256x128**, decoded at an aligned size of 256 and with the half-scale path (the
`FullScale` flag clear). That gives the expected chunk sizes:

- colour inflates to 98304 = `256*256` (Y) + `128*128` (Cb) + `128*128` (Cr)
- alpha inflates to 65536 = `256*256`

The alpha channel is uniform and opaque in all four lobby signs — it compresses to 190 bytes in
every one — so the board is a solid picture, not a cut-out.

**The artwork is a blank board.** Jungle's is cracked bark, fantasy's a mint wall, hallow's a
framed maroon panel, space's a starfield. No lettering is baked in; the park name is composited
on top at runtime using the fonts named above. The clearest evidence is that the colour chunk's
*compressed* size tracks the length of the park name — 5,607 bytes for "Space" (5 characters),
7,126 for "Halloween" (9), 11,760 for "Lost Kingdom" (12) and 14,606 for "Fantasy Island" (14) —
even though every file inflates to the same 98304 bytes.

## How the board maps onto the model

The model draws the board as **two 128x128 panels side by side**, using materials `sign1` and
`sign2`, each taking a full 0..1 UV range. `sign1` is the left half of the sheet and `sign2` the
right. Each material appears twice, once per face, mirrored, so the sign reads correctly from
either side.

Laying a two-line name out across the whole 256x128 sheet and then cutting it down the middle is
what puts "Lost" above "Kingdom" on the assembled board.

## Where the name comes from

Each park has a script at the root of `lobby.wad`, named after the park, holding a single
`ISLAND()` line whose fourth quoted field is the sign text:

```
ISLAND(0,"data\lobby\terrain","jun_isle","jun_gate","Lost Kingdom",90.0,12.5)
```

The others read `"Fantasy"`, `"Halloween"` and `"Space"`. The same line also names the terrain
directory and the two models, and ends with two numbers that look like a heading and a size.
