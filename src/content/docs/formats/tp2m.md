---
title: Attribute map (base.map, terrain.map)
---

A `TP2M` file is a 128 x 128 map of one attribute byte per cell. Exactly **five** files in the game open
with `TP2M`: the four parks' `base.map` and Lost Kingdom's `terrain.map`. All five are 16,464 bytes and
share the layout below, measured over all 15,570 files of the game's wads and loose folders.

## Layout

All values are little-endian.

| Offset | Size | Description |
| --- | --- | --- |
| `0x00` | 4 bytes | Magic, `TP2M` |
| `0x04` | 32 bytes | The text `Theme Park 2 Attribute Map File`, NUL-padded |
| `0x24` | 4 bytes | The chunk tag `MAP ` |
| `0x28` | 4 bytes | Chunk length, 16,412 in all five |
| `0x2C` | 4 bytes | Width, 128 |
| `0x30` | 4 bytes | Height, 128 |
| `0x34` | 20 bytes | Five `uint32`, all `8` in all five files. The loader never reads them (below) |
| `0x48` | 16,384 bytes | The cells: one byte each |
| `0x4048` | 8 bytes | The tail: the chunk tag `END ` and a `uint32` `0` |

The cells start at **`0x48` (72), not 80**. A reader that skips 80 bytes reads every cell eight bytes late
and takes the `END ` tail for its last cells: its grid differs from the 72-byte read in 1,156 bytes of
`terrain.map` to 1,846 of Lost Kingdom's `base.map`.

## What the original's loader reads

The `MAP ` chunk handler (`0x004d8cf0`) takes the width at the payload's `+0` and the height at `+4`, then
steps over the next 20 bytes (`add esi,0x14`, `0x004d8d17`) and copies the cells from the payload's `+0x1C`
(file offset `0x48`). Nothing reads the five words, so the original gives them no name; calling them "bits
per cell" is a guess the data cannot settle (the cells reach 148, which takes eight bits either way).

## Cell order

The cells are **x-major**: the byte for column `x`, row `y` is at `0x48 + x * 128 + y`. The handler places the
byte for file position (`i`, `j`) at cell (`i - XStart`, `j - YStart`), the `MapInfo` limits of the
[`.sam`](/formats/sam/), both 0 in every shipped `Standard.sam`, and the running park numbers its cells
`y * 128 + x + 1`. That is the opposite way round from a park file's own cell array
([saves](/formats/saves/)).

## Values

Across the five files the values that occur are 0, 1, 3, 8, 16, 17, 128, 144 and 148. Lost Kingdom's
`terrain.map` holds only 0 and 1. The values look like a bitfield; only some bits are decoded
(OpenTPW's `docs/exe/park.md`, ".MAP — the TP2M attribute map", lists what the fixed infrastructure's cells hold).

(upstream look 2026-10-19, `verify-terrain.md` #1, #2)
