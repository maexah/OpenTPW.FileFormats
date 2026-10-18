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

The 13 values of a preset are the EAX 2.0 listener properties (`EAXLISTENERPROPERTIES`), a mixture of floats and
integers, in this order: 0 environment, 1 environment size, 2 environment diffusion, 3 room, 4 room HF, 5 decay time,
6 decay HF ratio, 7 reflections, 8 reflections delay, 9 reverb, 10 reverb delay, 11 room rolloff factor, 12 air
absorption HF. The game sends them to the listener as property set `{0306A6A8-B224-11D2-99E5-0000E8D8C722}`
(`DSPROPSETID_EAX20_ListenerProperties`), id 1, in the structure's own order (3, 4, 11, 5 to 10, then 0, 1, 2, 12)
with the flags word 9. (Fork watch 2026-10-18, `verify-other-a.md` and `verify-audio-fx.md`.) In the shipped file slots 1 to 8
and 11 to 14 are identical to one another; slots 0, 9 and 10 each differ.

A slot is applied to the listener, not to a voice, and only through EAX 2.0 hardware. The slot to use is a ride
script's `SETREVERB` level while the camera rides that ride; off a ride it is the level key `Sound.Reverb`, which
`Standard.sam` sets to `0` and the Halloween theme's `Standard.sam` to `3`. Which levels the shipped scripts pass was not
swept.
