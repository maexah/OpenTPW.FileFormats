---
title: Item Height Maps (*.hmp)
---

An `.hmp` is **a height map of a buildable thing, with a plane of shape marks beside it**. It says how high the
thing stands over each part of each cell it covers, in its own unturned space. The game reads it to lift the
placement squares over something already built (see "What the game does with it", below).

Every item ships one inside its own WAD, named after it (`gates.wad` holds `gates.hmp`). So do the pieces that are
not items: a track ride's track and pylons (`wr_trckB.hmp`, `StdPylon.hmp`), the queue pieces in `queue.wad`
(`queend.hmp`, `quebnd1.hmp` ...) and the park's hoarding in `hoarding.wad` (`ho_fob1.hmp` ...).

## Verified against every file

There are **435** `.hmp` files: jungle 110, fantasy 106, hallow 110, space 109. Every one fits the layout below with
no remainder, and every one is `48 + 27 × cells` bytes. That is a 48-byte header, then 25, 1 and 1 bytes for each
cell. 274 of them are an item's own file, beside its `.sam`; in all 274 the columns and rows are the footprint that
`.sam` gives (`Info.Shape`'s box, or its `Info.EngineFootprintWidthOverride` / `HeightOverride`).

## Layout

All values are little-endian.

| Offset | Size | Field |
| ------ | ---- | ----- |
| `0x00` | 4 | Signature `0xAB1E0003` |
| `0x04` | 4 | Signature `0x00640005` |
| `0x08` | 2 | Columns: cells along the thing's own x |
| `0x0A` | 2 | Rows: cells along its own z |
| `0x0C` | 4 | Offset of the raster, from the file's start (always `0x30`) |
| `0x10` | 4 | Offset of the cell grid (`0x30 + 25 × cells`) |
| `0x14` | 4 | Offset of the mark plane (the grid's offset + cells) |
| `0x18` | 24 | The thing's box: six floats, min x, y, z then max x, y, z (min ≤ max in all 435) |
| `0x30` | 25 × cells | **Raster**: (5 × columns) by (5 × rows) bytes, row by row, five samples a cell each way |
| ... | cells | **Cell grid**: one byte a cell, row by row, the highest of that cell's 25 raster bytes |
| ... | cells | **Mark plane**: one byte a cell, row by row, 0 or 1 |

A height is a byte × 2.55: the game reads a byte back by multiplying it by 1 / 2.55 (`0.3921569`).

Row `r`, column `c` of the grid is byte `r × columns + c`. The raster's samples for that cell are rows `5r` to
`5r + 4` and columns `5c` to `5c + 4` of a raster `5 × columns` wide.

### The cell grid is the raster's maximum, except at the gates

In 431 files each grid byte is exactly the highest of its cell's 25 raster bytes. The four park gates (one a theme)
are the exception. Their raster is 0 everywhere except in its last cell, and their grid holds one value in every
cell: the raster's maximum in jungle, hallow and space, and 97 under a maximum of 101 in fantasy.

29 files hold a 255 somewhere in their grid.

### The box

The six floats at `0x18` have the same shape as the box at [model](/formats/models/) offset `0x80`. They match it byte
for byte in 53 of the 173 files that have a model of the same name beside them, and differ in the rest. When the game
loads an `.hmp`, it copies these six floats over the model's own (see below).

### The mark plane

The mark plane carries `Info.Shape`'s marks: 1 where the picture marks a cell and 0 where it leaves the cell blank.
The 2026-09-30 fork review measured this over all 274 items, and found nothing the `.sam` does not already say.

## The size gives the cell count

The size alone gives how many cells the thing covers:

```text
cells = (size - 48) / 27
```

So a one-cell item is 75 bytes, and the park gate (six cells by three) is 534. The count is the **box** the
`Info.Shape` picture is drawn in, not the cells marked inside it. `4x4rock` draws 14 stars in a 4×4 box and its file
says 16; `ground`, `groundc` and `mystery` draw none in a 2×2 box and their files say 4. The jungle gate's picture is
a single cell, and its override keys declare the 6×3 its file holds.

## What the game does with it

These are notes on the 2.0 executable (OpenTPW's `docs/exe/park-engine.md`, "Placement feedback").

- **Loading** (`FUN_00451640`). The game reads the file and tests both signature dwords. It turns the three offsets
  into pointers and copies the box over the model's at `+0x80`. If the file is missing, the signature is wrong, or
  the columns or rows differ from the footprint the item declares, it **rebuilds** the file from the model
  (`FUN_00451880`) and writes the rebuild back to disk. No shipped file triggers a rebuild.
- **The preview's fit** (`FUN_004689f0`). The turning model in an object's window is sized and placed by the box
  at `0x18` alone: half the footprint's diagonal in two thirds of the panel's width, or the distance from the box's
  low corner at height nought to its high corner in the panel's height, whichever is tighter. The model turns
  about half the box's width and depth from its own origin, which is the footprint's middle where the box's low
  corner is the origin.
- **Reading** (`FUN_00452ae0`). For a cell, it finds the thing standing there and carries the cell back into the
  thing's unturned space by its quarter turn. It reads the **cell grid** byte there and answers byte / 2.55 plus the
  height of the thing's root node. The raster and the mark plane are not read here.
