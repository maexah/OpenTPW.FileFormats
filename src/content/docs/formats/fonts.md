---
title: Bitmap fonts (*.bf4)
---

Everything the interface letters — button labels, dialog text, the help bar, the park name on the
lobby panel — is drawn in a bitmap font from `data\Language\<language>\*.bf4`. The park signs are
the exception: they use TrueType fonts from `fonts.wad` (see [Park signs](../sgn/)).

Thirty-seven ship with the English data: `CASH*`, `DATE*`, `MENU*`, `SESH*` and `TITLE*` in
`SMALL`/`MED`/`BIG` sizes (`DATETINY` too), `GAME5AA` to `GAME12AA`, the plain `GAME6` to `GAME12`
and `GAMEBOLD9` to `GAMEBOLD12`, and `CONSOLE6`, `COURIER8`, `MATISSE18`, `MATISSE36` and
`POSTCARD`.

## Finding a font

The game never names a font file. It asks for a resource id, and `residx.dat` in the same language
folder turns the id into a file name.

### residx.dat

| Offset | Size | Description |
| --- | --- | --- |
| `0x00` | 4 bytes | Magic, `BFRI` |
| `0x04` | 4 bytes | Entry count — `56` in the English data |
| `0x08` | 12 bytes each | Entries |
| after the entries | | File names, NUL-terminated, in entry order |

Each entry:

| Offset | Size | Description |
| --- | --- | --- |
| `+0x0` | 4 bytes | Resource id |
| `+0x4` | 2 bytes | Type — `4` for a font; the font loader refuses anything else |
| `+0x6` | 4 bytes | Where the name starts, counted from offset `0x08` |
| `+0xA` | 2 bytes | `0` in every entry |

The English fonts are ids `1021` (`GAME5AA.bf4`) to `1052` (`POSTCARD.bf4`). Ids `1000`–`1020` are
the string tables (`ERRORMSG.str`, `UITEXT.str` and so on) with type `3`, and ids `1`–`3` are
`MBToUni.dat` and `UniToMB.dat` with types `0`–`2`.

### The interface's thirteen font slots

A control asks for a font by slot number, and the slot is looked up in whichever of four sets the
language loader picked for the screen resolution:

| Slot | Set 0 | Set 1 | Set 2 | Set 3 | Used for |
| --- | --- | --- | --- | --- | --- |
| 0 | `MENUSMALL` | `MENUMED` | `MENUBIG` | `MENUBIG` | The park name on the lobby panel |
| 1 | `CASHSMALL` | `CASHMED` | `CASHBIG` | `CASHBIG` | |
| 2 | `SESHSMALL` | `SESHMED` | `SESHBIG` | `SESHBIG` | The lobby's key count |
| 3 | `DATETINY` | `DATESMALL` | `DATEBIG` | `DATEBIG` | |
| 4 | `GAME6AA` | `GAME10AA` | `GAME8AA` | `GAME12AA` | Message box text |
| 5 | `TITLESMALL` | `TITLEMED` | `TITLEBIG` | `TITLEBIG` | Purple buttons, dialog titles |
| 6 | `GAME5AA` | `GAME7` | `GAME8` | `GAME9` | The new player dialog |
| 7 | `GAME6AA` | `GAME8` | `GAME9` | `GAME10` | The help bar |
| 8 | `GAME5AA` | `GAME8AA` | `GAME8AA` | `GAME12` | |
| 9 | `GAME5AA` | `GAME7` | `GAME8` | `GAME9` | |
| 10 | `CONSOLE6` | `CONSOLE6` | `CONSOLE6` | `CONSOLE6` | |
| 11 | `GAME6AA` | `GAME7` | `GAME8` | `GAME9` | |
| 12 | `POSTCARD` | `POSTCARD` | `POSTCARD` | `POSTCARD` | |

The loader picks set 0 for the lowest resolution setting, sets 1 and 2 for the next two, and set 3
for the fourth and every setting above it.

> Which resolutions those settings are is inferred, not read: the options screen's steps run
> 512x384, 640x480, 800x600, 1024x768 and up, which would make set 3 everything from 1024x768.

## File format

| Offset | Size | Description |
| --- | --- | --- |
| `0x00` | 4 bytes | Magic, `F4FB` — the only thing the loader checks |
| `0x04` | 1 byte | Version — `1` in the anti-aliased fonts checked, `2` in `GAME8` and `GAME9` |
| `0x05` | 1 byte | Line height in pixels — `22` for `TITLEBIG`, `55` for `MENUBIG`, `11` for `GAME8` |
| `0x06` | 2 bytes | Glyph count — `249` in every English font |
| `0x08` | 4 bytes each | Offset of each glyph record |

The records are not sorted by character.

### Glyph record

| Offset | Size | Description |
| --- | --- | --- |
| `+0x00` | 4 bytes | Character, as a Unicode code point (up to `U+2019`, the apostrophe) |
| `+0x04` | 4 bytes | Stored size of the pixel data, padded to a multiple of 4 |
| `+0x08` | 4 bytes | Unpacked size: half the pixel count, rounded up — one nibble a pixel |
| `+0x0C` | 4 bytes | Packing — see below |
| `+0x10` | 2 bytes | Width |
| `+0x12` | 2 bytes | Height |
| `+0x14` | 1 byte, signed | Left — offset from the pen |
| `+0x15` | 1 byte, signed | Top — offset from the top of the line |
| `+0x16` | 2 bytes | Advance — how far the pen moves |
| `+0x18` | | Pixel data, the stored size long |

Across every glyph of `TITLEMED`, `TITLEBIG`, `MENUBIG`, `SESHBIG`, `GAME8`, `GAME9`, `GAME8AA` and
`GAME10AA`, `0x18` plus the stored size is exactly the distance to the next record, and the unpacked
size is always the pixel count halved and rounded up.

### Pixels

A pixel is four bits of coverage, `0` to `15`, laid out row after row with nothing between rows,
high nibble first. How the pixel data stores them depends on the packing:

| Packing | Storage |
| --- | --- |
| `0` | The nibbles as they are |
| `1` | Run-length coded nibbles: a non-zero nibble is itself; a zero is followed by a count and a value to repeat that many times; a zero count ends the glyph |
| `2` | One bit a pixel, high bit first — a set bit is full coverage (`15`) |

The anti-aliased fonts mix `0` and `1` (`TITLEBIG` is 123 of one and 126 of the other); the plain
`GAME*` fonts use `2` for most glyphs and `0` for the rest.

The engine unpacks packing `1` and `2` into nibbles before drawing, and draws by moving each pixel
underneath towards the text colour by `coverage / 15`, scaled by the colour's alpha. The destination's
own alpha is left alone.

## Open questions

- What version `1` against `2` changes beyond which packing the glyphs use.
- Whether any language needs the multi-byte path: the font object has a decoder that reads a second
  byte when the first is above a lead-byte threshold, which the English strings never reach.
