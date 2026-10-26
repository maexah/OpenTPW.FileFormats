---
title: Park map picture (2dmap.tga)
---

Each theme's `data/levels/<theme>/2dmap.tga` is the picture the park map screen shows under its live overlay. Measured
over all four (`jungle`, `fantasy`, `hallow`, `space`; Q624):

| | |
|---|---|
| Kind | Truevision TGA, image type 2 (uncompressed true colour), no image id, no colour map |
| Size | 512 x 512, 24 bits a pixel (B, G, R) |
| Descriptor | `0x00`: no alpha bits, **origin bottom-left**, so the file's first row is the bottom of the picture as a viewer shows it |
| Footer | 26 bytes after the pixels: the TGA 2.0 footer (two zero offsets, then `TRUEVISION-XFILE.` and a nought), 786,476 bytes in all |
| Field | Black outside columns 16 to 495 and rows 43 to 467 (rows counted from the top, as a viewer shows it): 480 x 425 |

**The field is the heightfield's 96 by 85 cells at five pixels a cell**: cell (x, y) is pixels `16 + 5x` across and
`43 + 5y` down, in all four themes (480 = 96 x 5, 425 = 85 x 5; the executable's own numbers are `480`, `425`, `15`
and `42` at `0x00703044`..`0x00703058`, OpenTPW `docs/exe/hud.md`, "The map's view"). The park gate is near the top
of the picture as a viewer shows it; the game draws the picture upside down, so on the map screen the gate is at the
bottom.

## The overlay's six sprites (`data/2dmap/*.tga`)

The map's live overlay draws its people, litter and ranges with six small pictures, the folder `data/2dmap/` (beside a
`qickload.txt` naming the six). Measured over all six (Q664):

| File | Bytes | Size | What it holds |
|---|---|---|---|
| `vsprite.tga` | 1,068 | 16 x 16 | a white disc with a dark edge, solid inside (alpha 255 across its middle) |
| `ssprite.tga` | 1,068 | 16 x 16 | the same bytes as `vsprite.tga` |
| `rsprite.tga` | 1,068 | 16 x 16 | a ring: the same white and dark edge, its middle clear (alpha 0 across six pixels of the middle row) |
| `lsprite.tga` | 16,428 | 64 x 64 | a soft disc: every pixel white (255, 255, 255), the alpha falling from 255 at the middle to 0, 2,966 pixels above 0 |
| `esprite.tga` | 16,428 | 64 x 64 | the same bytes as `lsprite.tga` |
| `gsprite.tga` | 16,428 | 64 x 64 | the same bytes as `lsprite.tga` |

All six are Truevision TGA image type 2, no image id, no colour map, 32 bits a pixel (B, G, R, A), descriptor `0x08`
(eight alpha bits, origin bottom-left), with the 26-byte TGA 2.0 footer. Their colour is the game's: it tints each
white picture per sprite (OpenTPW `docs/exe/hud.md`, "The map's overlay", has which colour and which scale each is drawn at).
