---
title: Sprites (*.esp, *.tpc)
---

The game's flat, camera-facing pictures - guests walking about, staff, balloons, litter, thought
bubbles and particles - are sprites, kept in `data\esprites.wad`. Each folder there holds one or more
**banks**: a `.ESP` file saying which pictures make up each of its sixteen sets, and a `.TPC` of the
same name holding the pictures.

Some banks have a `.FPC` beside the `.TPC`. The two carry the same number of pictures and differ only
in size, and **both are version 3** - every one of the archive's 46 `.TPC` and 29 `.FPC` files is, so
the layout below reads either. Which of the pair is loaded is chosen from the bank's flag at `0x10C`.

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
| `0x14E` | 4 x 4 bytes | Four groups, selected by a state of 0 to 3 - see below |

Each set:

| Offset | Size | Description |
| --- | --- | --- |
| `0x0` | 2 bytes | First picture |
| `0x2` | 1 byte | Frames per direction |
| `0x3` | 1 byte | Directional: when set, the sprite's direction picks a run of pictures, frames-per-direction apart |

### The four groups at `0x14E`

These are read, not merely counted. The loader keeps the bank in a larger record and the four groups
land at `+0x20E` through `+0x21D` in it; a routine picking a sprite's animation takes a state of 0 to
3 and reads that group's four bytes as **two set numbers, each stored one higher than the set it
names, and two further bytes** whose meaning is not identified. A group's set numbers are therefore
`byte - 1`, and a stored `0` means "no set".

## Directions

A directional set stores its whole run once per direction, one after another, so the pictures for
direction *d* begin at `first + d * framesPerDirection`. **There are five directions.**

That number is not written down anywhere in the file, but the sets of a bank are laid end to end in
its pictures, which makes it measurable: the gap from one set's `first` to the next set's is exactly
`framesPerDirection x directions`, so every consecutive pair of sets is an independent vote. Across
all 46 banks, **169 gaps give five**; the only other answer is one, and it comes from the banks that
are not directional at all (`Balloons`, `Litter`, `Particles`, `Thoughts`, `SpecialFX`). The check
that settles it is the picture count: taking the last set's `first` plus its own span, **29 banks land
exactly on the number of pictures in their `.TPC`** at five directions, and the seven non-directional
ones land exactly at one.

A bank whose sets are all a single run offers no gap to measure, so this method says nothing about it.
The `*heads` banks are all of that shape, and **their direction count is not established here** - do
not assume it is five.

A guest's bank - `Generic\Kids\SPR_BE`, and the seven beside it - is 175 pictures in eight sets:

| Set | First | Frames per direction | What it is |
| --- | --- | --- | --- |
| `12` | 0 | 4 | |
| `4` | 20 | 4 | |
| `11` | 40 | 4 | |
| `2` | 60 | 8 | |
| `0` | 100 | 1 | A single standing picture per direction |
| `14` | 105 | 2 | |
| `6` | 115 | 4 | |
| `1` | 135 | 8 | Walking |

`135 + 8 x 5 = 175`, the bank's whole pack.

## Banks by kind

The executable groups the archive's folders into fourteen kinds and holds them in one table, in this
order:

| Index | Folder | Index | Folder |
| --- | --- | --- | --- |
| 0 | `kids` | 7 | `guards` |
| 1 | `kidsheads` | 8 | `researchers` |
| 2 | `costumes` | 9 | `thoughts` |
| 3 | `costumeheads` | 10 | `balloons` |
| 4 | `entertainers` | 11 | `litter` |
| 5 | `handymen` | 12 | `specialfx` |
| 6 | `mechanics` | 13 | `particles` |

Each entry records where that kind's banks start and how many there are. When something in the park
needs a sprite, the game picks a bank from its kind **at random**, and then a variant within the bank
the same way; the choice is stored on the thing rather than made again, so a park reloaded from a save
keeps the guests it had.

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
