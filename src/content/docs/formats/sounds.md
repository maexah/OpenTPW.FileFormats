---
title: Sound Data (*.sdt)
---

SDT is an archive format that contains one or many MPEG audio files that are used for sound effects, speech, and music throughout the game.  These files are usually uncompressed (in the archive), and the archive format is relatively simple.

Despite the `.mp2` extension on every name inside a bank, the audio is **not always Layer II**: 2,646 of the game's 3,739 streams are MPEG-1 **Layer I**, and the rest are Layer II. Layer III does not occur. A decoder has to handle both.

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

| Offset | Size     | Description                                        |
| ------ | -------- | -------------------------------------------------- |
| 0x00   | 4 bytes  | Header size - 40 in every entry the game ships      |
| 0x04   | 4 bytes  | Data size                                          |
| 0x08   | 16 bytes | File name (NUL-padded, and truncated to fit)       |
| 0x18   | 2 bytes  | Sample rate                                        |
| 0x1A   | 1 byte   | Bits per sample                                    |
| 0x1B   | 1 byte   | Sound type (see below)                             |
| 0x1C   | 4 bytes  | Unknown - zero in all 3,739 entries                |
| 0x20   | 4 bytes  | Size the audio decodes to, in bytes                |
| 0x24   | 4 bytes  | Unknown - zero in all 3,739 entries                |
| 0x28   | n bytes  | File data                                          |

Those middle three fields are the ones worth being careful about. Read as three 32-bit values - which is how the 40-byte header appears to divide - the sample rate still comes out right and everything after it is nonsense, because the bit depth and the type are a **byte each**, not a word each. Across the whole game they only ever read 22,050 or 44,100 at 16 bits, and 36 or 37 for the type, which is what fixes the split.

**Sound Types**

* 0: None
* 2: WAV
* 3: "Old WAV" (see <https://github.com/ufdada/dk2-tools/blob/master/Formats/Sound/sdt_struct.bt>)
* 36: MP2, 64kbps mono
* 37: MP2, 112kbps stereo 