---
title: Reverb presets (SndReverb.map)
---

`SndReverb.map` sits in the game's root folder, beside `testme.exe`, and is 820 bytes. It holds a table of
reverb presets, read by the executable's sound code (the loader is at `0x006c5d20`). The format is
established to the last byte: the loader's walk, applied to the shipped file, ends exactly at its end.

| Offset | Size | Description |
| --- | --- | --- |
| `0x00` | 4 bytes | `2000`. The loader refuses any other value |
| `0x04` | 4 bytes | The number of slots, `32` |
| `0x08` | per slot | One slot after another, 32 in all - see below |

## A slot

| Size | Description |
| --- | --- |
| 1 byte | `0` if the slot is empty, non-zero if a preset follows |
| 52 bytes | Only when the byte is non-zero: 13 four-byte values, the preset. The loader copies them into an array of `0x34`-byte records |

An empty slot costs one byte, so the file's length is `8 + 32 + 15 x 52 = 820`. The shipped file fills slots
0 to 14 (each flag byte is `1`) and leaves 15 to 31 empty. The loader keeps an empty slot's first word as `-1`.

The 13 values of a preset are not named. They are a mixture of floats and integers. In the shipped file slots 1 to 8
and 11 to 14 are identical to one another; slots 0, 9 and 10 each differ.

What the executable does with a preset once loaded was not traced here.
