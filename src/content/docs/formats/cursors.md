---
title: Cursors (*.ani, *.cur)
---

The pointer's pictures are in `data\ui\cursors`: 21 Windows animated cursors (`.ani`), one plain cursor
(`c_noplace.cur`), and 22 `.tga` strips beside them (`Cnor.tga`, `Ccro.tga`...), each four 32 x 32 frames
side by side in a 128 x 32 picture. The game registers the `.ani` files by name (`c_crosshair.ani` is
first person's cursor).

## The animated cursor

A standard RIFF file of form `ACON`: an `anih` chunk (36 bytes: its own size, the frame count, the step
count, then width, height, bit count and planes left at zero, the display rate in jiffies, and the flags, 3 or 1),
then a `LIST` of form `fram` holding one `icon` chunk per frame. Each `icon` chunk is a whole `.cur` file:
the six-byte directory (reserved 0, type **2**, count 1) and one 16-byte entry, then the image.

In a cursor's entry the two words an icon gives to planes and bit count are the **hotspot**: the pixel of
the picture that stands on the pointer.

| Offset in the entry | Size | Description |
| --- | --- | --- |
| `0x00` | 1 | Width (32 in every file) |
| `0x01` | 1 | Height (32 in every file) |
| `0x02` | 1 | Colour count |
| `0x03` | 1 | Reserved |
| `0x04` | 2 | Hotspot x |
| `0x06` | 2 | Hotspot y |
| `0x08` | 4 | Image size |
| `0x0C` | 4 | Image offset from the `.cur`'s start |

Measured over all 22 files, every frame of each: the hotspot is **(0, 0)** - the picture's top-left
corner - in all of them but `c_crosshair.ani`, whose three frames all stand by **(16, 16)**, their middle.
A pointer drawn from the `.tga` strips must take its hotspot from here, or the crosshair stands 16 pixels
right of and below the point it aims at.

The frame counts are 1 to 4 (`c_busy`, `c_path` and `c_track` 4; `c_cam` and `c_cash` 2; the eight
1-frame cursors are `c_end`, `c_height`, `c_move`, `c_pivot`, `c_pylon`, `c_queue`, `c_rotate`, `c_swapdown`
and `c_swapup`, nine in all; the rest 3). No file holds a `rate` chunk; all but `c_busy`, `c_path` and `c_track` hold a `seq ` chunk.
