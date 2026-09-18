---
title: Sound Data (*.sdt)
---

SDT is an archive format that contains one or many MPEG audio files that are used for sound effects, speech, and music throughout the game. Despite the `.mp2` extension on every name inside a bank, most are **Layer I**, not Layer II: of the 3,739 entries the game ships, 2,646 are Layer I and 1,093 are Layer II, so a decoder must read the layer out of each frame header rather than assume it.  These files are usually uncompressed (in the archive), and the archive format is relatively simple.

### File Format

**Header**

| Size    | Description |
| ------- | ----------- |
| 4 bytes | File count  |

**For each file**

| Size    | Description |
| ------- | ----------- |
| 4 bytes | File offset |

**For each file (at offset)**

| Size     | Description                         |
| -------- | ----------------------------------- |
| 4 bytes  | Header size                         |
| 4 bytes  | Data size                           |
| 16 bytes | File name (usually null terminated) |
| 2 bytes  | Sample rate                         |
| 1 byte   | Resolution (bits per sample)        |
| 1 byte   | Sound type (see below)              |
| 4 bytes  | Unknown                             |
| 4 bytes  | Decoded size                        |
| 4 bytes  | Unknown                             |
| n bytes  | File data                           |

The header is **40 bytes**, which is also what each entry's own header-size field declares -
checked across all 32 entries of `global/sound/UIHD.sdt`, where the audio begins at exactly
`+40` with an MPEG sync word. The sample rate, resolution and sound type are 2, 1 and 1
bytes, not four each. Reading them as four-byte fields is the trap worth naming: the rate
comes out plausible and everything after it is nonsense.

**Sound Types**

* 0: None
* 2: WAV
* 3: "Old WAV" (see <https://github.com/ufdada/dk2-tools/blob/master/Formats/Sound/sdt_struct.bt>)
* 36: MP2, 64kbps mono
* 37: MP2, 112kbps stereo 