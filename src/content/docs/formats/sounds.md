---
title: Sound Data (*.sdt)
---

SDT is an archive format holding one or many audio streams, used for sound effects, speech and
music throughout the game. The streams are stored uncompressed within the archive, and the
container itself is simple.

The game ships 48 banks: 47 `.sdt` files of their own under `data/`, and `BankHD.sdt` inside
`levels/jungle/features/speaker1.wad`, one of the jungle's features. Every entry in the 47 loose
banks is MPEG audio, and despite the `.mp2` extension on their names, it is **not always Layer
II**. Of their 3,739 entries, 2,650 are Layer I and 1,089 are Layer II:

| Kind | Count |
| --- | --- |
| MPEG-2 Layer I, 64 kbit/s mono, 22,050 Hz | 2,641 |
| MPEG-2 Layer II, 48 kbit/s mono, 22,050 Hz | 644 |
| MPEG-2 Layer II, 112 kbit/s stereo, 22,050 Hz | 445 |
| MPEG-1 Layer I, 64 kbit/s mono, 44,100 Hz | 5 |
| MPEG-2 Layer I, 64 kbit/s stereo, 22,050 Hz | 4 |

So a decoder needs Layers I and II, but never Layer III — which is the layer that needs a bit
reservoir, Huffman tables and an MDCT — and it has to read the layer out of the stream's frame
header rather than assume it. Every frame of an entry has the same version, layer, bitrate,
sample rate and channel mode, so the first frame header describes the whole stream. The wad's
bank holds the only two entries that are not MPEG, `drumloop.wav` and `error.wav`: they are PCM,
sound type 2 (see below).

## File format

**Header**

| Size    | Description |
| ------- | ----------- |
| 4 bytes | File count  |

**For each file**

| Size    | Description |
| ------- | ----------- |
| 4 bytes | File offset |

The entries follow the offset table back to back, in table order: the first begins at
4 + 4 × count, each ends where the next begins, and the last ends at the end of the file.

**For each file (at offset)**

The header is 40 bytes in every entry that ships, and its audio begins at exactly `+40`: with an
MPEG frame header in the 3,739 MPEG entries, and with the first sample in the two PCM ones.

| Offset | Size     | Description |
| ------ | -------- | ----------- |
| `0x00` | 4 bytes  | Header size — always 40 |
| `0x04` | 4 bytes  | Size of the audio that follows the header (in an MPEG entry, its whole frames, then one zero byte) |
| `0x08` | 16 bytes | Name, NUL-terminated and NUL-padded. **Truncated** to 15 characters, not shortened |
| `0x18` | 2 bytes  | Sample rate — 22,050 or 44,100, and in an MPEG entry always the rate in the stream's own frame header |
| `0x1a` | 1 byte   | Bits per sample — always 16 |
| `0x1b` | 1 byte   | Sound type (see below) |
| `0x1c` | 4 bytes  | Unknown — zero in all 3,741 entries |
| `0x20` | 4 bytes  | How much of the audio the game plays, in bytes of 16-bit PCM (see below); 0 in the two PCM entries |
| `0x24` | 4 bytes  | Unknown — zero in all 3,741 entries |
| `0x28` | n bytes  | Audio data |

> **The four bytes after the name are not a 32-bit word.** That is the obvious reading — the
> header is 40 bytes, the name ends at `0x18`, and the 16 bytes left over divide neatly into four
> words — and it is wrong. The sample rate is 16 bits and the bit depth and sound type are 8 bits
> each. Read as 32-bit words, the low sixteen bits of the first word are still the right rate,
> which is the kind of mistake that survives a casual check: the one field you would think to
> verify is the one that still works. The bit depth and the type are lost in that word's top half;
> the three words after it still read correctly. A reader that goes on to take the bit depth and
> type as words of their own puts the header at 48 bytes, eight bytes into the audio, and then
> everything after the rate is wrong. What settles the split is that across the whole game those
> bytes only ever read 22,050 or 44,100, 16, and 2, 36 or 37.

Because the name field is a fixed sixteen bytes, always ending in a NUL, a name longer than 15
characters is cut rather than shortened: `TP SCREECH 11.mp2` is stored as `TP SCREECH 11.m`. A
lookup by full name has to allow for that, and cannot always tell entries apart: the cut leaves
two entries sharing one stored name in three places, `tp_balloon_pop_` in `global/sound/UIHD.sdt`
and `TP STRANGELY DE` and `TP STRANGE DEEP` in `levels/jungle/Sound/AmbientHD.sdt`.

The value at `0x20` is the length the game plays in an MPEG entry: its read never goes past it
(`0x006c8429`). It is always less than what the stream decodes to, by 558 to 1,820 samples a
channel — between 0.59 and 2.58 frames, or 25 to 83 ms at 22,050 Hz and 14 to 21 ms for the five
entries at 44,100 — so the end of every stream is never heard. The stream's own length, its size
in bytes × 8 over the bitrate in its first frame header, is the longer figure.

**Sound types**

* 0: None
* 2: WAV
* 3: "Old WAV" (see <https://github.com/ufdada/dk2-tools/blob/master/Formats/Sound/sdt_struct.bt>)
* 36: mono
* 37: stereo

The two MPEG codes are usually described as "MP2 64 kbit/s mono" and "MP2 112 kbit/s stereo",
and neither is exact. The mono code covers Layer I at 64 and Layer II at 48 alike, and the stereo
code covers Layer II at 112 and the four Layer I streams at 64. So the code says how many channels
there are and nothing about the layer or bitrate — the stream's frame header says the rest. Every
type-36 stream is single-channel and every type-37 stream is plain stereo in its own frame header.

Type 2 is used by the two entries in `speaker1.wad`'s bank. Their audio is raw little-endian
16-bit PCM at the entry's rate: no RIFF header and no MPEG sync. The codes 0 and 3 come from the
Dungeon Keeper 2 tools linked above, and no entry in this game uses either.

The executable reads the type as a set of flags rather than as a number (`0x006ba610`), testing
`0x20`, then `0x10`, then `0x40`, and each branch that builds a voice picks one of two classes by
what `0x006c8be0` returns. Bit `0x01` is the channel count, `(type & 1) + 1` (`0x006d3a80`). Bit
`0x20` picks the voice class that plays both MPEG codes, or its subclass `0x006c87c0`. The
class's constructor, `0x006c82c0`, which the subclass calls first, is where bit `0x04` (set in 36
and 37 alike) makes the value at `0x20` the length played. Bit `0x10` picks another pair of classes
(`0x006c7b20` / `0x006c7f70`). Bit `0x40` builds no voice at all: `0x006ba610` hands the request on
through `0x006c8c20` and returns none. No shipped entry sets either. A type with none of the
three takes the class `0x006ba610` falls through to, `0x006c6e70` or its subclass `0x006c7780`;
`0x006c6e70` is also the base that `0x006c82c0` and `0x006c7b20` construct first. Type 2 reaches
that class with one channel, and 0 and 3 would reach it too. How that class plays its audio has
not been read, so that the game plays type 2 as raw PCM rests on the data, not on the executable.

## How banks are addressed

Nothing in the game names a `.sdt` entry directly. Sounds are grouped into **categories**, and code
plays a numbered effect within one — so the lobby's thunder is "global lobby sfx, effect 1" rather
than a reference to `Thunder2.mp2`. The banks a category draws on, and the effects in it, are
described by a pair of `.map` files beside the banks. See [Sound categories](/formats/sound-categories/).
