---
title: Sound Data (*.sdt)
---

SDT is an archive format holding one or many MPEG audio streams, used for sound effects, speech
and music throughout the game. The streams are stored uncompressed within the archive, and the
container itself is simple.

Despite the `.mp2` extension on every name inside, the audio is **not always Layer II**. Across the
3,739 entries in the shipped game:

| Kind | Count |
| --- | --- |
| MPEG-2 Layer I, 64 kbit/s mono, 22,050 Hz | 2,641 |
| MPEG-2 Layer II, 48 kbit/s mono, 22,050 Hz | 644 |
| MPEG-2 Layer II, 112 kbit/s stereo, 22,050 Hz | 445 |
| MPEG-1 Layer I, 64 kbit/s mono, 44,100 Hz | 5 |
| MPEG-2 Layer I, 64 kbit/s stereo, 22,050 Hz | 4 |

So a decoder needs Layers I and II, but never Layer III — which is the layer that needs a bit
reservoir, Huffman tables and an MDCT. Nothing in the game is WAV, despite the sound-type codes
leaving room for it.

## File format

**Header**

| Size    | Description |
| ------- | ----------- |
| 4 bytes | File count  |

**For each file**

| Size    | Description |
| ------- | ----------- |
| 4 bytes | File offset |

**For each file (at offset)**

The header is 40 bytes in every entry that ships.

| Offset | Size     | Description |
| ------ | -------- | ----------- |
| `0x00` | 4 bytes  | Header size — always 40 |
| `0x04` | 4 bytes  | Size of the audio that follows the header |
| `0x08` | 16 bytes | Name, NUL-padded. **Truncated** to fit, not shortened |
| `0x18` | 2 bytes  | Sample rate — 22,050 or 44,100 |
| `0x1a` | 1 byte   | Bits per sample — always 16 |
| `0x1b` | 1 byte   | Sound type (see below) |
| `0x1c` | 4 bytes  | Unknown — zero in all 3,739 entries |
| `0x20` | 4 bytes  | Size the audio decodes to, in bytes |
| `0x24` | 4 bytes  | Unknown — zero in all 3,739 entries |
| `0x28` | n bytes  | Audio data |

> **The four fields after the name are not four 32-bit words.** That is the obvious reading — the
> header is 40 bytes, the name ends at `0x18`, and 16 bytes left over divides neatly by four — and
> it is wrong. The sample rate is 16 bits and the bit depth and sound type are 8 bits each. Read as
> words, the rate still comes out right and everything after it is nonsense, which is the kind of
> mistake that survives a casual check. What settles it is that across the whole game those bytes
> only ever read 22,050 or 44,100, 16, and 36 or 37.

Because the name field is a fixed sixteen bytes and longer names are cut rather than shortened,
`TP SCREECH 11.mp2` is stored as `TP SCREECH 11.m`. A lookup by full name has to allow for that.

The decoded size at `0x20` is a rounded figure and disagrees with an actual decode by up to one
frame, so it is a hint rather than a length. Where an accurate duration is needed, take the bitrate
out of the stream's own first frame header.

**Sound types**

* 0: None
* 2: WAV
* 3: "Old WAV" (see <https://github.com/ufdada/dk2-tools/blob/master/Formats/Sound/sdt_struct.bt>)
* 36: mono
* 37: stereo

The two MPEG codes are usually described as "MP2 64 kbit/s mono" and "MP2 112 kbit/s stereo", and
that is true of the stereo one. The mono code covers Layer I at 64 and Layer II at 48 alike, so it
says how many channels there are and nothing more — the stream's frame header says the rest.

## How banks are addressed

Nothing in the game names a `.sdt` entry directly. Sounds are grouped into **categories**, and code
plays a numbered effect within one — so the lobby's thunder is "global lobby sfx, effect 1" rather
than a reference to `Thunder2.mp2`. The banks a category draws on, and the effects in it, are
described by a pair of `.map` files beside the banks. See [Sound categories](/formats/sound-categories/).
