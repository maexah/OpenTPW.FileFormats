---
title: Park signs (*.sgn)
---

A `.sgn` holds everything a name board needs: the fonts its name is lettered in, the ink each line
is lettered in, the pattern the letters are filled with, and — sometimes — the artwork painted on
the board behind them. The name itself is not in here.

Both the lobby's park boards and the name boards on rides inside a park use this format. Walking
the lobby's terrain directory and the four item folders of every theme finds **84** of them:

| File | Board |
| --- | --- |
| `lobby.wad` → `terrain\Jun_isle.sgn` | Jungle — lettered "Lost Kingdom" |
| `lobby.wad` → `terrain\Hal_isle.sgn` | Hallow — lettered "Halloween World" |
| `lobby.wad` → `terrain\Fan_gate.sgn` | Fantasy — lettered "Wonder Land" |
| `lobby.wad` → `terrain\Spa_gate.sgn` | Space — lettered "Space Zone" |
| `levels\<theme>\features\gates\gates.sgn` | the park's own entrance gate |
| `levels\<theme>\features\sign1\sign1.sgn` | the placeable in-park sign |
| `levels\<theme>\rides\<ride>\<ride>.sgn` | a ride's name board |

A lobby `.sgn` is named after **the model that carries the `sign1` and `sign2` materials**, which is
the island model for jungle and hallow but the gate model for fantasy and space — so a lookup has
to try both rather than assume either. A ride's is named after the ride's own folder.

## File format

> **The file is read strictly in order, and almost nothing in it is at a fixed offset.** Three
> things move everything after them, so a reader has to walk the file rather than index into it:
> each line's 20-byte ink block is written **only** when that line's mode is non-zero; each of the
> two fill textures states **its own** width, height and bytes per pixel; and the artwork is present
> **only** when the byte at `0x0008` says so.

Walked that way the parse lands exactly on the end of all 84 shipped signs: for the 23 that carry
artwork it arrives precisely at the `BILZ` chunk magic, 23 times out of 23, and for the 61 that do
not it arrives precisely at end-of-file, 61 times out of 61.

Only the first seventeen bytes and the two font records are at guaranteed offsets:

| Offset | Size | Description |
| --- | --- | --- |
| `0x0000` | 4 bytes | Version — the engine refuses anything below `100` (engine-confirmed). Shipped files are `100` or `101` |
| `0x0004` | 4 bytes | Selects which of the two lines is inked first; `0` and `1` both occur. Not fully established |
| `0x0008` | 1 byte | **Artwork flag** — non-zero if artwork follows the fill textures. **Clear in 61 of the 84** (engine-confirmed) |
| `0x0009` | 4 bytes | **Line 0 mode** — see **Line modes** below (engine-confirmed) |
| `0x000D` | 4 bytes | **Line 1 mode** (engine-confirmed) |
| `0x0011` | 436 bytes | Font record 0 |
| `0x01C5` | 436 bytes | Font record 1 |

> Rows marked **(engine-confirmed)** were checked against the original game's own reader and sign
> compositor rather than inferred from the shipped files alone — the reader loads the file field by
> field in order, which is what fixes the sizes and the boundaries below.

After the second font record, at `0x0379`, the walk begins:

1. **Line 0 ink**, 20 bytes — present only if line 0's mode is non-zero.
2. **Line 1 ink**, 20 bytes — present only if line 1's mode is non-zero.
3. **Fill texture 0** — a 12-byte header, then `width * height * bytesPerPixel` bytes.
4. **Fill texture 1** — the same again.
5. **The artwork** — only if the byte at `0x0008` is non-zero.

Because both ink blocks are present in all but three shipped signs, and because every shipped fill
texture is 16 x 128 x 4, the common case works out to fixed-looking numbers — which is how this
header came to be believed fixed:

| Both ink blocks | Fill textures start | Artwork header | `BILZ` | Header total |
| --- | --- | --- | --- | --- |
| present | `0x03A1` | `0x43B9` | `0x43DD` | 17,373 |
| absent | `0x0379` | `0x4391` | `0x43B5` | 17,333 |

The 40 bytes between them are exactly the two ink blocks. `jelly.sgn`, `zob.sgn` and `C_SCAT.sgn`
set both modes to `0` and are the three that take the short layout. A sign with no artwork simply
ends after fill texture 1: those files are 17,337 and 17,297 bytes, each **36 bytes** — one artwork
header — shorter than the corresponding total above.

### Font record

| Offset | Size | Description |
| --- | --- | --- |
| `+0x000` | 64 bytes | Display name, NUL-padded ASCII (e.g. `Young Itch AOE`) |
| `+0x040` | 260 bytes | TrueType file name (e.g. `YOUNIA__.TTF`) — the file itself is in `fonts.wad` |
| `+0x144` | 4 bytes | Unknown |
| `+0x148` | 4 bytes | Unknown |
| `+0x14C` | 60 bytes | A Windows **`LOGFONTA`** (engine-confirmed) |
| `+0x188` | 44 bytes | Read as a block by a separate helper; contents not established |

There are always two records, read unconditionally, with no count anywhere in the file. Every one of
the 84 signs fills both, and every one names a TrueType file to go with the display name. Jungle
letters its two lines in different fonts (`Young Itch AOE` then `Clunker AOE`); Space names the same
font twice.

The original letters its signs with GDI, so it stores a `LOGFONT` rather than a size and weight of
its own. That structure is why the display name appears to occur a second time part-way through the
record: `LOGFONTA.lfFaceName` sits at `+28` within it, which is `+0x168` from the record start. Its
`lfHeight` is the usual negative character height (`-144` for jungle's first line).

> **The floats in the tail are not a colour.** Reading the record as two 64-byte name fields
> instead gives eight tidy floats at `+0x18C`, three of which always land in `0..1` and look
> convincingly like a text colour — jungle's are `0.40, 0.87, 0.31`, a green. They are not. That
> offset falls past the end of the `LOGFONT`, inside the 44-byte tail above, and the values belong
> to whatever the helper reads there.
>
> The trap is worth spelling out because the wrong reading survives a casual check: three of the
> four lobby boards are dark, so lettering them in some wrong colour still shows up. Only Fantasy
> gives it away — its board is painted pale mint and those floats are very nearly the same mint, so
> its name comes out invisible. The real ink is below.

### Line ink

Each line's colour is a four-byte block, followed by four more 4-byte fields that are not yet
identified — 20 bytes per line.

| Offset | Size | Description |
| --- | --- | --- |
| `+0x00` | 1 byte | Red |
| `+0x01` | 1 byte | Green |
| `+0x02` | 1 byte | Blue |
| `+0x03` | 1 byte | Opacity |
| `+0x04` | 4 x 4 bytes | Unknown — an int, a float, then two more ints |

Three steps in the engine fix that ordering, and none of it has to be guessed. The loader reads the
four bytes **singly** into consecutive bytes of its sign object. The renderer hands them to the
compositor with the fourth byte first and the other three after it. The compositor walks the glyph's
coverage mask and moves each board pixel that fraction of the way toward the three channels, writing
them into bytes 1, 2 and 3 of a pixel whose byte 0 is the coverage — and that same function later
packs the buffer as **ARGB4444**, which is what makes byte 1 red rather than blue.

The opacity is a genuine blend and not a threshold, so a line set below full strength tints the
board and lets the artwork show through the lettering.

> **A line whose mode is `0` has no ink block at all**, and the engine composites it out of a field
> it never wrote, so nothing appears. Three shipped signs are in that state — `jelly.sgn`,
> `zob.sgn` and `C_SCAT.sgn` — and two of those carry artwork with a name already painted into it.

What the four lobby signs ask for:

| File | Mode | Line 0 | Line 1 |
| --- | --- | --- | --- |
| `Fan_gate.sgn` (Fantasy) | 1 | `#808000` olive, 67% | `#808000` olive, 66% |
| `Hal_isle.sgn` (Hallow) | 1 | `#00FF00` green, 79% | `#00FF00` green, 84% |
| `Jun_isle.sgn` (Jungle) | 2 | `#000000` black, 60% | *(never read)* |
| `Spa_gate.sgn` (Space) | 2 | `#FF80FF` pink, 100% | *(never read)* |

### Line modes

The two 4-byte fields at `0x0009` and `0x000D` say how each line is laid down. `1` inks the line on
its own; `2` means the two lines are drawn as one; `0` means the line is not lettered.

That distinction matters for colour. At mode `2` the engine maxes both glyph masks into a single
surface and then runs **one** colour over the result, so the second line's own four bytes are never
reached and both words come out in the first line's ink. `Jun_isle.sgn` and `Spa_gate.sgn` are both
mode `2`; `Fan_gate.sgn` and `Hal_isle.sgn` are mode `1` and ink each line separately. A file whose
two modes disagree takes a third path that nothing in the shipped data exercises.

Across all 84 signs the pairs that occur are `(1,1)`, `(1,2)`, `(2,1)`, `(2,2)` and `(0,0)`.

### Fill textures

Two images follow the ink, one per line, each a 12-byte header and then its pixels:

| Offset | Size | Description |
| --- | --- | --- |
| `+0x00` | 4 bytes | Width — `16` in every shipped sign |
| `+0x04` | 4 bytes | Height — `128` in every shipped sign |
| `+0x08` | 4 bytes | Bytes per pixel — `4` in every shipped sign |
| `+0x0C` | `w * h * bpp` | Pixels |

That makes `2 * (12 + 8192) = 16,408` bytes. The engine multiplies the three values rather than
assuming them, so a reader must too.

These are the pattern the letters are **filled with** — not a thumbnail of the sign, and not
padding. In many signs every 16-pixel row is identical, making the image a pure vertical gradient;
in others it varies horizontally as well. Byte 0 of each pixel is constant within a block and
appears not to be read.

> **This resolves a guess this page used to carry.** It described "three ints (`16`, `128`, `4`)
> then six 4-byte entries" at `0x03A1`, noting they were a constant `FF DB FF 30` repeated in three
> of the four lobby signs but six steadily climbing values in `Jun_isle.sgn`, and wondered whether
> they were a small gradient or a palette. The instinct was right and the geometry wrong: the three
> ints are this header, and the "entries" are simply the first pixels of fill texture 0 — identical
> along a row in a vertical ramp, varying per pixel in `Jun_isle`.

## The artwork

Present **only** when the byte at `0x0008` is non-zero — 23 of the 84 signs. With it clear the
original clears the board to transparent black and skips the artwork entirely, so the name is
lettered onto nothing and floats with the model showing through behind it. A sign without artwork is
a normal sign, not a truncated one.

The artwork is a [`.wct`](/formats/texture/) image with its header fields rearranged:

| Offset | Size | Description |
| --- | --- | --- |
| `+0x00` | 4 bytes | Width — `256` in every shipped sign |
| `+0x04` | 4 bytes | Height — `128` in every shipped sign |
| `+0x08` | 4 bytes | Type — `4` in every shipped sign, meaning a colour chunk and an alpha chunk |
| `+0x0C` | 4 x float | Y, Cb, Cr and A quantisation scales (`6, 10, 2, 6` in every shipped sign) |
| `+0x1C` | 4 bytes | Colour chunk size |
| `+0x20` | 4 bytes | Alpha chunk size |
| `+0x24` | | Colour chunk, then alpha chunk — each in the same `BILZ` container a `.wct` uses |

The two chunks can be handed to a `.wct` decoder directly. `BILZ` begins at `+0x24`, so
`BILZ + colourChunkSize + alphaChunkSize` is the file's length — exact in all 23.

> **The stated height is the artwork's, not the buffer's.** The chunks are encoded at the wavelet's
> aligned **256 x 256** with the half-scale chroma path, and only the top 128 rows are board. The
> colour chunk cannot tell the two readings apart — `256*128*3` and `256*256 + 2*128*128` are
> **both** 98304 — but the alpha chunk can, and it inflates to **65536**, which is `256*256` and not
> `256*128`. So decode at 256 x 256 and take the top half.

**No lettering is baked into the artwork.** Jungle's is cracked bark, fantasy's a mint wall,
hallow's a framed maroon panel, space's a starfield; the name is composited on top at runtime with
the fonts and ink above. The plainest evidence is that the 61 signs carrying no artwork at all still
show their names.

> An earlier note offered a different argument — that the colour chunk's *compressed* size tracks
> the park name's length. Do not rely on it. It rests on four samples, one of them mislabelled
> ("Fantasy Island" is not that park's name), and the correlation would be evidence **for** baked-in
> lettering rather than against it.

### The alpha channel

The artwork's alpha matters, because the engine draws a sign's material see-through and so honours
it. Measured over the drawn region of all 23:

- **15 are solid art.** Their only partly-clear texels are about 1.6% of the board, never below
  alpha 77 — the same figure on every one of them, unrelated artwork included, which is the `.wct`
  codec ringing partly-clear texels around a hard edge rather than anything authored.
- **8 are cut-outs**, reaching alpha **0** across 5% to 43% of the board, so the board is a shape
  rather than a rectangle. All are ride boards: the Bumper Cars (42.6%), Candy Cabin (32.9%), Cat
  Coaster (17.6%), Ferris Wheel, Tour Ride, both Coasters and the Drip.

> This page used to say "the alpha channel is uniform and opaque in all four lobby signs… so the
> board is a solid picture, not a cut-out". That holds for those four bar the rim, but it is not
> true of the format: a third of the artwork-bearing signs are deliberate cut-outs, and a reader
> that ignores their alpha draws them as opaque rectangles.

## How the board maps onto the model

The model draws the board as **two 128x128 panels side by side**, using materials `sign1` and
`sign2`, each taking a full 0..1 UV range. `sign1` is the left half of the sheet and `sign2` the
right. Each material appears twice, once per face, mirrored, so the sign reads correctly from either
side.

Laying a two-line name out across the whole 256x128 sheet and then cutting it down the middle is
what puts "Lost" above "Kingdom" on the assembled board.

The engine finds those two surfaces by their material's texture name — literally `sign1.tga` and
`sign2.tga` — and then **replaces the texture with one it builds at runtime**, 128 x 128 per face.
That is why no model ships a `sign1.tga` or `sign2.tga` file to go with the name. Painting is gated
on finding exactly two such surfaces; with one, nothing is painted, and the shipped executable says
nothing about it.

> **The engine also marks those two materials see-through as it substitutes the texture**, rather
> than trusting how the model was authored — a ride's sign material is authored solid. A
> reimplementation that swaps the texture but leaves the material alone draws the transparent board
> as solid black, and the cut-out boards above as opaque rectangles.

## Where the name comes from

Each park has a script at the root of `lobby.wad`, named after the park, holding a single `ISLAND()`
line whose fourth quoted field is a sign text:

```
ISLAND(0,"data\lobby\terrain","jun_isle","jun_gate","Lost Kingdom",90.0,12.5)
```

The others read `"Fantasy"`, `"Halloween"` and `"Space"`. The same line also names the terrain
directory and the two models, and ends with two numbers that look like a heading and a size.

> **Open question: that is not what the boards say.** Three of the four are lettered "Wonder Land",
> "Halloween World" and "Space Zone", not "Fantasy", "Halloween" and "Space" — only Jungle's script
> text matches its board. `THEMENAMES.str` holds a set of park names, and is the likelier source for
> what is actually lettered. Which of the two the compositor reads has not been established here.

A ride's board takes its name from the ride's own [`.sam`](/formats/sam/) item description instead.
The engine splits a two-word name across the two lines by searching outward from the middle of the
string for a space, so the break lands as near the centre as the name allows.

The `.sgn` beside a ride is looked up with a prefix first — `<dir>\<prefix>_<name>.sgn` — falling
back to `<dir>\<name>.sgn` when that does not exist. Every shipped sign uses the plain form.
