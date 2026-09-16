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
bank. Frame *n* of a set is picture `first + n` in the bank's `.TPC`. Banks are numbered in the order
they are **loaded**, which for two of the fourteen kinds is not the order their folder lists them - see
[How a folder is swept](#how-a-folder-is-swept).

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
| `0x3` | 1 byte | How many directions this set stores, `0` if it faces nowhere - see [Directions](#directions) |

### The four groups at `0x14E`

These are read, not merely counted. The loader keeps the bank in a larger record and the four groups
land at `+0x20E` through `+0x21D` in it; a routine picking a sprite's animation takes a state of 0 to
3 and reads that group's four bytes as **two set numbers, each stored one higher than the set it
names, and two further bytes** whose meaning is not identified. A group's set numbers are therefore
`byte - 1`, and a stored `0` means "no set".

## Directions

A set stores its whole run once per direction, one after another, so the pictures for direction *d*
begin at `first + d * framesPerDirection`. **How many directions is written down: it is the set's own
fourth byte.**

An earlier version of this page called that byte a flag and said there were five directions
everywhere. Five is what a *body* stores, and it is what the arithmetic below independently gives, but
it is not universal. Reading the byte across every set in use in all 46 banks:

| Directions | Sets | Which |
| --- | --- | --- |
| `5` | 201 | Every full-body person - guests, staff, costumes |
| `7` | 10 | The eight `Kidsheads` banks and two `Costumeheads` |
| `4` | 1 | `Jungle\Entertainers\SPR_EX` set 3 |
| `0` | 71 | Faces nowhere at all: `Balloons`, `Litter`, `Particles`, `Thoughts`, `SpecialFX` |

The byte agrees with the pictures. A bank's sets are laid end to end, so the span of a set is
`framesPerDirection x directions` and the last set's `first` plus its own span should land on the
bank's picture count: `Generic\Kids\SPR_BE` set 1 is first 135, eight frames a direction, five
directions, and `135 + 8 x 5 = 175` is exactly its pack. **282 of the 283 sets in use fit their pack
this way.** The one that does not is `Generic\Kidsheads\SPR_BE`, which asks for 56 pictures from a pack
holding 55 - a defect in that one file rather than a rule, so a reader must not index blindly.

That also settles what the older gap-counting method could not. It worked by making each consecutive
pair of sets vote, so a bank whose sets are a single run offered nothing to measure, and the `*heads`
banks are all of that shape. They store **seven**.

## Eight headings from five pictures

Five stored directions cover eight compass headings because the game **reflects** them. When the
heading it wants runs past the last one stored, it draws `8 - heading` mirrored instead, which is why a
guest walking away to the left and one walking away to the right are the same pictures.

One shipped set does not survive its own rule: `Jungle\Entertainers\SPR_EX` set 3 stores four
directions, and heading 4 folds to `8 - 4 = 4`, which is still past the end. The game walks on into the
neighbouring set's pictures and draws those.

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

## How a folder is swept

A kind's banks are gathered under **two** roots in turn - `generic\<kind>\` and the current theme's
`<theme>\<kind>\` - both feeding one numbering. The archive keeps the two disjoint: `Generic` holds the
eleven kinds that are not costumes, costume heads or entertainers, and each of the four themes holds
only those three. So exactly one of the two sweeps ever finds anything, and a park never numbers another
theme's banks.

Within a folder the order is **not** simply the order the files are listed, for `kids` and `kidsheads`.
The loader first matches a table of four names, in this order, and loads whichever of them it finds:

| | | | |
| --- | --- | --- | --- |
| `SPR_BI` | `SPR_KI` | `SPR_TA` | `SPR_SU` |

Only then does it sweep up whatever is left, in the order the archive lists it. `Generic\Kids` is listed
as BE, BI, CH, FR, KI, SA, SU, TA, so its banks come out in a different order entirely:

| Bank | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Loaded | `SPR_BI` | `SPR_KI` | `SPR_TA` | `SPR_SU` | `SPR_BE` | `SPR_CH` | `SPR_FR` | `SPR_SA` |
| Listed | `SPR_BE` | `SPR_BI` | `SPR_CH` | `SPR_FR` | `SPR_KI` | `SPR_SA` | `SPR_SU` | `SPR_TA` |

An earlier version of this page said banks were numbered in the order a folder's files are found. That
holds for the other twelve kinds and is wrong for these two, and it matters: the shipped Jungle park's
thirteen guests wear kids banks 0, 2, 4, 5, 6 and 7, so reading them in listed order dresses every one
of them as the wrong child while every count and every trailer still agrees.

How many banks a sweep loads is also capped by the detail setting, so a folder is not always read to the
end - `kids` stops at two, four, six or eight of its eight, and the staff kinds at one or two. The
shipped Jungle park's guests reach bank 7, so that park was saved with all eight loaded.

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
| `0x08` | 2 bytes | Reference width the engine divides by - `128` in every picture checked |
| `0x0A` | 2 bytes | Reference height the engine divides by - `128` in every picture checked |
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

## How big a sprite is in the world

A picture carries its own reference size at `0x08` and `0x0A`, and it is `128` in every picture in the
archive. The engine divides a picture's measurements by that and multiplies by a span of **20.0** held
in read-only data, so a picture's height in the world is its pixels times `20 / 128`.

Nothing else on that chain varies. The vertical factor beside the span is `1.0` and is never written
anywhere in the executable, the draw call passes `1.0` for both of its own multipliers, and all eighteen
sprites in the shipped park carry a scale of `1.0`.

The horizontal chain carries one further factor, and it is **not** a world measurement but the
viewport's: the same value divides the frustum's half-height to give its half-width, which makes it
height over width. It reads as zero in the file on disk because it is filled in when the display is set
up. A reader that has a projection matrix of its own already accounts for that and should not apply it a
second time.
