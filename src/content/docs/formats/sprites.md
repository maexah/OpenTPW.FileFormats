---
title: Sprites (*.esp, *.tpc)
---

The game's flat, camera-facing pictures - guests walking about, staff, balloons, litter, thought
bubbles and particles - are sprites, kept in `data\esprites.wad`. Each folder there holds one or more
**banks**: a `.ESP` file saying which pictures make up each of its sixteen sets, and a `.TPC` of the
same name holding the pictures.

Some banks have a `.FPC` beside the `.TPC`: the same figure seen from ground level, for first person,
where the `.TPC` is seen from above. It is not a level of detail. **Both are version 3** - every one of the
archive's 46 `.TPC` and 29 `.FPC` files is, so the layout below reads either. 28 of the 29 pairs hold the
same number of pictures; `Jungle\Entertainers\SPR_EX` holds 219 in its `.TPC` and 210 in its `.FPC`. In
all 29 pairs the `.FPC`'s pictures are taller for their width on average (the kids' 1.56 to 1.62 against
1.34 to 1.39), as a figure seen from lower down is: the pair differ in viewing elevation, not only in size.

The flag at `0x10C` says whether first person draws the bank from its `.FPC`. It is set in 27 of the 46
banks, and every one of them has a `.FPC`; it is clear in all 17 banks without one, and in two that have
one, `Jungle\Entertainers\SPR_EX` and `Space\Costumes\SPR_SK`, which first person leaves on their
`.TPC`. In Lost Kingdom the flagged banks are the eight kids, the guards, the handymen, both mechanics, the
researchers, `SPR_TI` and the entertainers `SPR_DI` and `SPR_NA`; the kid heads, the costume heads and the
balloons have no `.FPC` and stay as they are.

The game asks for a sprite by a number whose low four bits are the set and whose other bits are the
bank. Frame *n* of a set is picture `first + n` in the bank's `.TPC`, or its `.FPC` in first person. Banks are numbered in the order
they are **loaded**, which for two of the fourteen kinds is not the order their folder lists them - see
[How a folder is swept](#how-a-folder-is-swept).

The particle effects use the two banks in `Generic\Particles`: `SPR_PA` (bank 0) and `SPR_PB` (bank 1).

## Bank (*.esp)

350 bytes:

| Offset | Size | Description |
| --- | --- | --- |
| `0x000` | 12 bytes | Magic, `ESP_FILE2.00` |
| `0x00C` | 256 bytes | Name, NUL-padded - `SPR_PA.TPS` |
| `0x10C` | 1 byte | `1`: first person draws this bank from its `.FPC` (see above); `0`: always the `.TPC` |
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
banks are all of that shape. They store **seven** - but for a head the seven are not compass directions. Laid out, a
head bank's 56 pictures (`Generic\Kidsheads\SPR_BI` looked at) are **eight headings to a row, a full turn with no
reflection, and seven rows from straight above to straight below**: row 0 the crown, row 3 level, row 6 the chin. The
game draws a head with its own direction switched off and picks the picture itself, heading + 8 × row (OpenTPW's
`docs/exe/ride-operation.md`, "Which picture a head shows").

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
| `12` | 0 | 4 | A jump with both arms up |
| `4` | 20 | 4 | |
| `11` | 40 | 4 | |
| `2` | 60 | 8 | |
| `0` | 100 | 1 | A single standing picture per direction |
| `14` | 105 | 2 | Standing with hands on hips, the two pictures a pixel or so apart |
| `6` | 115 | 4 | Bending forward at the waist and straightening |
| `1` | 135 | 8 | Walking |

`135 + 8 x 5 = 175`, the bank's whole pack.

Sets 12, 14 and 6 have the same frame counts in all twelve guest banks, these eight and the four themes'
`Costumes`. Every staff and entertainer bank has a set 6 and neither 12 nor 14.

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
needs a sprite, the game picks a bank from its kind **at random**, and a balloon its set too (see
[Balloons](#balloons)); the choice is stored on the thing rather than made again, so a park reloaded from a save
keeps the guests it had.

## How a folder is swept

A kind's banks are gathered under **two** roots in turn - `generic\<kind>\` and the current theme's
`<theme>\<kind>\` - both feeding one numbering. The archive keeps the two disjoint: `Generic` holds the
eleven kinds that are not costumes, costume heads or entertainers, and each of the four themes holds
only those three. So exactly one of the two sweeps ever finds anything, and a park never numbers another
theme's banks.

The loader gathers both roots' files into one list and **sorts it by name**, ignoring case, before it loads
anything; every one of the archive's 21 sprite folders is already listed in that order, so for the shipped
files the two agree. For `kids` and `kidsheads` the order is then **not** simply name order: the loader
first matches a table of four names, in this order, and loads whichever of them it finds:

| | | | |
| --- | --- | --- | --- |
| `SPR_BI` | `SPR_KI` | `SPR_TA` | `SPR_SU` |

Only then does it sweep up whatever is left, in name order. `Generic\Kids` is listed, and sorts, as BE,
BI, CH, FR, KI, SA, SU, TA, so its banks come out in a different order entirely:

| Bank | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Loaded | `SPR_BI` | `SPR_KI` | `SPR_TA` | `SPR_SU` | `SPR_BE` | `SPR_CH` | `SPR_FR` | `SPR_SA` |
| Listed | `SPR_BE` | `SPR_BI` | `SPR_CH` | `SPR_FR` | `SPR_KI` | `SPR_SA` | `SPR_SU` | `SPR_TA` |

The other twelve kinds are numbered in name order. For kids it matters: the shipped Jungle park's
thirteen guests wear kids banks 0, 2, 4, 5, 6 and 7, so reading them in name order alone dresses every one
of them as the wrong child while every count and every trailer still agrees.

How many banks a sweep loads is also capped by the detail setting, so a folder is not always read to the
end - `kids` stops at two, four, six or eight of its eight. Generic staff kinds 5–8 stop at one or two;
entertainers (kind 4) are uncapped. Lost Kingdom has three entertainer banks, one handyman, two mechanics,
one guard and one researcher: seven staff banks at low detail, eight above low. These counts come from the
archive folders, so a fresh park needs no saved staff to discover their artwork. The
shipped Jungle park's guests reach bank 7, so that park was saved with all eight loaded. The detail files
`med.sam` and `high.sam` set the kids' `NUMKIDS` to 2 and `low.sam` to 0, which the executable reads as six
and two, not as the files' own comment says (`0->4, 1->6, 2->8`); parks the original saved at those settings
bear it out, their guests' kids banks all 0 to 5.

## Balloons

There is one balloon bank in the archive, `Generic\Balloons\SPR_BL` - a 350-byte `.esp` and a 4,546-byte `.tpc` of
eight pictures - and every theme uses it. Four sets have frames, two each, facing nowhere:

| Set | First picture | Colour |
| --- | --- | --- |
| 0 | 4 | red |
| 1 | 2 | green |
| 2 | 0 | blue |
| 3 | 6 | yellow |

Frame 0 is the whole balloon, 16 x 19 or 16 x 20 pixels; frame 1 is the same balloon burst, 24 x 27 to 26 x 30.
Every picture's origin down is -73 to -79, so its top is 73 to 79 pixels above the point it is drawn at: a balloon
floats above its guest's head. **The palette's alpha is meant**: the whole balloon's body is palette alpha 218 of 255
with softer edges, and the bursts' spray is near-black at alpha 8 to 80 - about two-thirds of the non-clear pixels in
the blue, green and red bursts, two-fifths in the yellow - so drawing every non-clear pixel opaque turns a burst into a
dark blob.

A balloon's set is a draw below the count of sets with frames, `(r >> 2) % 4`, used as the set number itself; it
lands on a set with frames because `SPR_BL`'s four are sets 0 to 3.

## Thought bubbles

Two banks, `Generic\Thoughts\SPR_TB` (a 14,198-byte `.tpc` of 16 pictures) and `SPR_TC` (11,664 bytes, 10 pictures),
neither with a `.fpc`. Every set in use is one frame, facing nowhere. `SPR_TB`'s sixteen sets are its sixteen
pictures in order; `SPR_TC` uses sets 0 to 9, whose first pictures are 0, 1, 8, 9, 7, 2, 3, 4, 5, 6. A bubble is
32 x 31 pixels with its origin down at -64; the four arrows are 41 or 42 x 51 at -119.

The executable names a thought picture by one number, the bank in its high bits and the set in its low four, so
`SPR_TC`'s sets are 16 to 25:

| Number | Bank, set | Picture |
| --- | --- | --- |
| 0 | `TB` 0 | yellow smiling face |
| 1 | `TB` 1 | orange level face |
| 2 | `TB` 2 | red angry face |
| 3 | `TB` 3 | blue sad face |
| 4 | `TB` 4 | grey yawning face |
| 5 | `TB` 5 | yellow laughing face |
| 6 | `TB` 6 | green sick face |
| 7 | `TB` 7 | pink frightened face |
| 8 | `TB` 8 | burger |
| 9 | `TB` 9 | thumbs down |
| 10 | `TB` 10 | drink |
| 11 | `TB` 11 | hourglass and thumbs down |
| 12 | `TB` 12 | drink and burger |
| 13 | `TB` 13 | toilet sign |
| 14 | `TB` 14 | litter |
| 15 | `TB` 15 | question mark |
| 16 | `TC` 0 | smiling face, pink bubble |
| 17 | `TC` 1 | sad face, pink bubble |
| 18 | `TC` 2 | sleeping face, pink bubble |
| 19 | `TC` 3 | placard, pink bubble |
| 20 | `TC` 4 | question mark, pink bubble |
| 21 | `TC` 5 | thumbs up |
| 22 to 25 | `TC` 6 to 9 | a blue, a green, a red and a yellow arrow pointing down, no bubble |

The blue bubbles are guests' and the pink ones staff's. Which thought shows which is the executable's table, in
OpenTPW's `docs/exe/ride-operation.md`.

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
| `0x08` | 2 bytes | Reference width the engine divides by - `128` in all 10,223 pictures |
| `0x0A` | 2 bytes | Reference height the engine divides by - `128` in all 10,223 pictures |
| `0x0C` | 4 bytes | Origin across, negated: `-8` for a 15-pixel-wide picture |
| `0x10` | 4 bytes | Origin down, negated |
| `0x14` | | Picture data |

The data is one run-length-coded row after another, top first. A row starts with a byte giving how many
bytes follow for that row. Then come codes, each a signed byte:

- **Below zero:** repeat the next byte, a palette index, *-n* times.
- **Above zero:** copy the next *n* bytes as palette indices.
- **Zero:** the engine treats it as a repeat, so it reads one more byte and draws nothing. No shipped picture
  has one.

A row ends when it reaches the picture's width; the engine steps over a row by its length byte only to skip it
while scaling.

For example `07 F1 00 02 F6 8F F1 00` is fifteen of index `00`, then `F6` and `8F`, then fifteen more of
`00` - a 32-pixel row.

This is the engine's own decoder (`0x00564790`, reached through slot `+8` of the vtable at `0x00701168`, with a
16-bit path at `0x005648c0` and a 32-bit one at `0x0056492c`). It decodes every picture the game ships exactly:
the 75 packs in `esprites.wad` (46 `.tpc`, 29 `.fpc`, the only ones in any of the 312 wads), 10,223 pictures and
500,222 rows. Every row reaches its width exactly and ends exactly on its length byte, and every picture ends
exactly on its data size. All 75 packs are version 3. The longest copy is 55, the longest repeat 85.

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
