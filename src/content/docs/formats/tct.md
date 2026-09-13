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

## Open: how a queue tile is indexed

`QueueTex` has rows 0 to 3, but the Jungle park's four queue cells carry indices **5, 2, 2 and 3**. Five
cannot address a four-row table.

The suggestive fit is that the index numbers *topology* on `PathTex`'s scale rather than art on
`QueueTex`'s — there 2 is a straight (and both those cells are straight), 3 a corner (and that cell's
mask really is a corner), and 5 a crossroads (and that cell really is where the queue joins the path).
That is a good fit on four cells and nothing more, and no other theme ships a saved park to check it
against, so it is recorded here as a lead rather than as the answer.
