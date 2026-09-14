---
title: Texture Correspondence Table (*.tct)
---

Each theme ships one `.tct`, named after itself — `Jungle.tct`, `Fantasy.tct` — **inside that theme's
`terrain.wad`**, so it is reached as `levels/<theme>/terrain/<Theme>.tct`. It is plain ASCII with CRLF
line endings and the name is the original author's: "Texture Correspondance Table".

It exists because some surfaces are textured by an index that nothing else names. A park's ground takes
its texture names from the frame table inside `base.MD2`, but a **path** has no such table — a path cell
stores a number, and this file is the only thing that turns that number into art.

## File format

```text
#  comment lines start with a hash
SectionName
0	name.tga
1	othername.tga

AnotherSection
0	something.tga
```

A line alone is a section heading; a line of `index`, a tab, and a file name is a row. Rows belong to
the most recent heading.

> **Honour the `#`.** Jungle's file ends with a fully commented-out `#Water` section. A reader that
> ignores comment markers invents a third section the game does not have, with rows the theme never
> uses.

> **The names end `.tga`; the files on disk are `.wct`.** Swap the extension when resolving. The art
> sits beside the table in the same archive — full size in `pathtex/`, low detail in `spathtex/`.

## Jungle's table

| Index | PathTex | Index | PathTex |
| ----- | ------- | ----- | ------- |
| 0 | `jpa_squ1` square | 11 | `jpa_icn1` |
| 1 | `jpa_end1` end | 12 | `jpa_icn2` |
| 2 | `jpa_str1` straight | 13 | `jpa_ctr1` centre |
| 3 | `jpa_cnr2` corner | 14 | `jpa_ctr1` **again — see below** |
| 4 | `jpa_tju1` T-junction | 15 | `jpa_tju4` T-junction |
| 5 | `jpa_xrd1` crossroads | 16 | `baseblue` |
| 6 | `jpa_tju2` T-junction | 17 | `grnarrow` |
| 7 | `jpa_tju3` T-junction | 18 | `redarrow` |
| 8 | `jpa_xrd2` crossroads | 19 | `jpa_str2` straight |
| 9 | `jpa_cnr1` corner | 20 | `jpa_edg2` edge |
| 10 | `jpa_edg1` edge | 21 | `dark` |

`QueueTex` has four rows: 0 `jpa_que4`, 1 `jpa_que3`, 2 `jpa_que1`, 3 `jpa_que2`.

> **Two oddities that are the file's own, not parsing faults.** 13 and 14 name the same art, and the
> author says why in a comment at the top of the file: *"14 has be put back as a dummy texture — maybe
> fix later ?"*. And `QueueTex`'s names are **not** in numeric order — row 0 is `que4`. So an index is a
> position in this table and nothing may be inferred from the name sitting at it.

Indices 16 to 18 (`baseblue`, `grnarrow`, `redarrow`) have no file in `pathtex/` at all. They are
interface or editor art rather than ground tiles, and no cell of the shipped park uses one.

## What the indices mean

A park's map cells store a tile index per cell (see [Saves](/formats/saves/)), and those indices address
this table. The mapping was confirmed by cross-referencing every index the shipped Jungle park uses
against the shape that cell's neighbours actually make:

| Index | Name | Cells | Neighbour shape |
| ----- | ---- | ----- | --------------- |
| 2, 19 | `str1`, `str2` | 28, 16 | degree 2, collinear — a straight, in two art variants |
| 3, 9 | `cnr2`, `cnr1` | 3, 2 | degree 2, bent — a corner |
| 4, 6, 7, 15 | `tju1`–`tju4` | 6, 3, 1, 1 | degree 3 — a T-junction |
| 5 | `xrd1` | 1 | degree 4, mask `0x55` (N+E+S+W) — a crossroads |
| 10, 20 | `edg1`, `edg2` | 6, 11 | degree 3, masks `0x1f` and `0xf1` |

> **The two edge tiles are the proof, and they say what kind of tile set this is.** `0x1f` is N, NE, E,
> SE, S and `0xf1` is N, S, SW, W, NW — every connection on one side, which is what the *edge* of a
> walkway more than one cell wide looks like. Together with `ctr1` (centre) and `squ1` (isolated square)
> that makes this an **area** tile set, not a set of one-cell-wide lines. Jungle's entrance avenue is
> two cells wide and is built from edge tiles facing each other.

This also fixes the compass for the neighbour mask: `0x01` N, `0x02` NE, `0x04` E, `0x08` SE, `0x10` S,
`0x20` SW, `0x40` W, `0x80` NW, with north at **decreasing y**. A cell with `0x44` (E+W) is a horizontal
straight and one with `0x11` (N+S) a vertical one.

## A queue tile index does not address this file at all

`QueueTex` has rows 0 to 3, but the Jungle park's four queue cells carry indices **5, 2, 2 and 3**, and
five cannot address a four-row table. The resolution is that a queue index is not a texture row: **it
names a model.**

A queue is built from seven small models that ship in the theme's own `queue.wad` — one cell each, each
with a flat `base` plate and a `.hmp` of 75 bytes, which is `48 + 27 × (1 × 1)`. The executable holds
them in a fixed table of twelve-byte records at **0x76338c**, and a cell's tile index is the position in
that table:

| Index | Model | | Index | Model |
| ----- | ----- | - | ----- | ----- |
| 0 | `quedead` | | 4 | `quebnd1` |
| 1 | `quedead` | | 5 | `queend` |
| 2 | `questra` straight | | 6 | `quebin1` |
| 3 | `quebnd2` bend | | 7 | `quebin2` |

Lost Kingdom's four cells read 5, 2, 2, 3 — **end, straight, straight, bend** — and that is exactly the
shape their stored neighbour masks make: the cell indexed 5 is the one that meets the path, the two
indexed 2 have mask `0x44` (east–west, collinear), and the one indexed 3 has mask `0x50` (south and
west, a corner). Four cells out of four, against a table read from the program rather than guessed.

`QueueTex` is still real and still used — it names the art those models are skinned with. Note the
theme's `queue.wad` ships six such textures, `jpa_que1` to `jpa_que6`, where this table names only four.

> **This is why a queue looks like fencing rather than paving.** Paths are an area tile set drawn onto
> the ground; queues are placed models with their own floor. A reader that treats tile set 2 the way it
> treats tile set 1 will look up art that was never meant to be laid as a tile.

## How the engine reads these fields

`FUN_005365d0` is where a map cell's tile fields turn into something drawn, and it settles two things
that the shipped data could only suggest. It reads them as **three consecutive dwords** — set, index,
then angle — confirming the split described in [Saves](/formats/saves/); and it derives the cell from
the record's own id as `x = (id − 1) & 0x7f` and `y = (id − 1) >> 7`, confirming the `y * 128 + x`
order. When the set is 2 it passes the index straight into the queue table above.

It also states the rotation convention outright: a piece is placed at **360 minus** the stored angle
(`0x168 - angle`, with 360 folded back to 0), so a saved angle turns the opposite way from a positive
rotation about the engine's up axis.
