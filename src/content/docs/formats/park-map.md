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
