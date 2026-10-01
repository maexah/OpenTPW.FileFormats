---
title: Video (*.tgq)
---

The game's movies: `Data\Movies\bf.tgq` (the Bullfrog logo) and eight park movies, `bub`, `buc`, `grav`, `jug`,
`mir`, `plan`, `roc` and `roll`. They are Electronic Arts multimedia files: TQI video (`pIQT`) with EA ADPCM audio
(`SCxl`), the same pair Dungeon Keeper 2 uses. The [EA uV/uV2 Video Player](https://lubiki.keeperklan.com/html/dk2_tools_other.php)
and ffmpeg (`eatqi`, `adpcm_ea`) both play them. Every fact below was measured in all nine files; what the executable
does with them is OpenTPW's `docs/exe/boot.md`, "The movie player".

Some of the general EA information was first taken from
<https://wiki.multimedia.cx/index.php/Electronic_Arts_Formats>. EA files have no single byte order: each number's
order suits the target platform. In these PC files the chunk sizes are little-endian and the `SCHl` tag values
big-endian.

## Chunks

A file is a run of chunks with no file header, and the chunks tile the file exactly to its end:

```text
bytes 0-3    FourCC
bytes 4-7    u32 LE size, counting these 8 bytes
bytes 8..    payload
```

Each of the nine files holds one `SCHl`, one `SCCl`, the `pIQT` frames, the `SCDl` audio chunks and one `SCEl`, in
the order `SCHl`, `pIQT`, `SCCl`, `pIQT`, `SCDl`, `pIQT`, ... There is no other chunk.

| File | `pIQT` frames | `SCDl` chunks | Samples per channel (`SCHl` tag `0x85`) |
|---|---|---|---|
| `bf.tgq` | 255 | 265 | 194,815 |
| `bub.tgq` | 1,174 | 1,174 | 862,999 |
| `buc.tgq` | 1,141 | 1,142 | 839,443 |
| `grav.tgq` | 1,212 | 1,213 | 891,701 |
| `jug.tgq` | 1,150 | 1,151 | 846,094 |
| `mir.tgq` | 1,360 | 1,361 | 1,000,591 |
| `plan.tgq` | 1,138 | 1,525 | 1,120,542 |
| `roc.tgq` | 990 | 991 | 728,347 |
| `roll.tgq` | 992 | 993 | 729,817 |

9,412 frames in all. The game's reader accepts each FourCC in both byte orders, and other EA video codecs too (TGV,
TGQ, MAD); none of them ships. It reads `UV2f` exactly as `pIQT`.

## `pIQT`: one video frame

```text
+0   u16 LE   width                320 in every frame
+2   u16 LE   height               352 in every frame
+4   u8       quant                99 in every frame
+5   u8       macroblocks across   20 in every frame; the game does not read it
+6   u8       macroblocks down     22 in every frame; the game does not read it
+7   u8       flags                3 in every frame. Bit 1 picks the game's output path; bit 0 is stored, its use unknown
+8   ...      bitstream
```

The bitstream is read as little-endian 32-bit words, each from its most significant bit. Macroblocks are 16 × 16, in
raster order, with six 8 × 8 blocks each: four luma, then Cb, then Cr (4:2:0). A block is an intra MPEG-1 block:

- **DC**: an MPEG-1 DC size code (the luma table for the first four blocks, chroma for the last two), then that many
  bits of difference, added to the previous DC of the same kind. The three predictors start at 0 in each frame.
- **AC**: MPEG-1's table B.14 codes, end-of-block included. The escape is a 6-bit run and an 8-bit level; level 0
  means the next 8 bits are the level, and level `0x80` means the next 8 bits less 256.

The game's own DC tables are MPEG-1's, entry for entry. For the AC codes the evidence is that ffmpeg's `eatqi`, which
uses MPEG-1's table, stayed in step with the game's own decoder, run beside it, on every one of the 9,412 frames. How
the coefficients are then scaled and transformed is the executable's own and not FFmpeg's (`boot.md`).

## `SCHl`: the audio header

`PT\0\0`, then one-byte tags. A tag is followed by a one-byte length and a big-endian value of that length, except
`0xFD` and `0xFF`, which stand alone. All nine files carry the same tags, apart from the count:

```text
00=2  06=101  1B=30  FD  85=<samples per channel>  82=2  83=7  8A=0  FF
```

`0x82` is the channel count (2, stereo), `0x83` the codec (7, EA ADPCM), `0x85` the samples per channel. There is no
`0x84` (sample rate) tag, so the rate is the game's default, 22,050 Hz. In every file the `SCDl` counts sum exactly to
tag `0x85`. The game reads no tag after `0x8A`.

## `SCDl`: one audio chunk

```text
+0    u32 LE   samples per channel in this chunk (n)
+4    s16 LE   left: the newest sample before the chunk
+6    s16 LE   left: the one before that
+8    s16 LE   right: the newest
+10   s16 LE   right: the one before that
+12   ...      ceil(n / 28) blocks of 30 bytes, then 2 bytes of padding when needed to end on a multiple of 4
```

A block is one byte of predictor indexes (left in the high nibble, right in the low), one byte of shifts (the same
way round), and 28 bytes holding one stereo sample each (left in the high nibble). Every chunk in all nine files has
exactly this size. The arithmetic is in `boot.md`.

The game skips `SCCl` and stops at `SCEl`.

## The EA audio chunks elsewhere

The EA audio chunk types (`SCHl`, `SCCl`, `SCDl`, `SCLl`, `SCEl`) are used throughout Electronic Arts games for
music, effects and speech as well as movies. Older titles keep one track per `.ASF` file; newer ones store several
tracks one after another in a file, which is what the end-of-track chunk types are for.
