---
title: Sprites (*.esp, *.tpc)
---

The game's flat, camera-facing pictures - guests walking about, staff, balloons, litter, thought
bubbles and particles - are sprites, kept in `data\esprites.wad`. Each folder there holds one or more
**banks**: a `.ESP` file saying which pictures make up each of its sixteen sets, and a `.TPC` of the
same name holding the pictures. Some banks have a `.FPC` beside the `.TPC`; it is not covered here.

The game asks for a sprite by a number whose low four bits are the set and whose other bits are the
bank. Banks are numbered in the order a folder's files are found. Frame *n* of a set is picture
`first + n` in the bank's `.TPC`.

The particle effects use the two banks in `Generic\Particles`: `SPR_PA` (bank 0) and `SPR_PB` (bank 1).

## Bank (*.esp)

350 bytes:

| Offset | Size | Description |
| --- | --- | --- |
| `0x000` | 12 bytes | Magic, `ESP_FILE2.00` |
| `0x00C` | 256 bytes | Name, NUL-padded - `SPR_PA.TPS` |
| `0x10C` | 1 byte | Flag; affects which picture file is loaded |
| `0x10D` | 1 byte | Flag, not identified |
| `0x10E` | 16 x 4 bytes | Sets |
| `0x14E` | 16 bytes | Four groups of four flags; counted by the loader but not seen used |

Each set:

| Offset | Size | Description |
| --- | --- | --- |
| `0x0` | 2 bytes | First picture |
| `0x2` | 1 byte | Frames per direction |
| `0x3` | 1 byte | Directional: when set, the sprite's direction picks a run of pictures, frames-per-direction apart |

## Pictures (*.tpc)

| Offset | Size | Description |
| --- | --- | --- |
| `0x00` | 2 bytes | Version - `3` |
| `0x02` | 2 bytes | `3`, not identified |
| `0x04` | 4 bytes | Picture count |
| `0x08` | 256 x 4 bytes | Palette, each colour blue, green, red, alpha (version 3 only) |
| `0x408` | | Pictures |

Version 2 stores its pictures differently and is not covered here.

Each picture:

| Offset | Size | Description |
| --- | --- | --- |
| `0x00` | 4 bytes | Size of the picture data |
| `0x04` | 2 bytes | Width |
| `0x06` | 2 bytes | Height |
| `0x08` | 2 bytes | `128` in every picture checked; not identified |
| `0x0A` | 2 bytes | `128` in every picture checked; not identified |
| `0x0C` | 4 bytes | Origin across, negated: `-8` for a 15-pixel-wide picture |
| `0x10` | 4 bytes | Origin down, negated |
| `0x14` | | Picture data |

The data is one run-length-coded row after another, top first. A row starts with a byte giving how many
bytes follow for that row. Then come codes, each a signed byte:

- **Below zero:** repeat the next byte, a palette index, *-n* times.
- **Above zero:** copy the next *n* bytes as palette indices.

For example `07 F1 00 02 F6 8F F1 00` is fifteen of index `00`, then `F6` and `8F`, then fifteen more of
`00` - a 32-pixel row.

Every picture in `SPR_PA.TPC` and `SPR_PB.TPC` decodes to exactly its width and height and uses exactly
its data size this way.
