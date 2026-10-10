---
title: Placed objects (scape.omp)
---

Each park's level folder holds one `scape.omp`, 409 to 590 bytes: the sounds and the particle effect that
stand in the land itself and belong to no ride or shop. The Lost Kingdom's is the only one that places a
particle effect, the spray at the foot of its waterfall.

The game reads it once as a park's level loads (`FUN_00550e00`, called at `0x00407f82`).

## Layout

All values are little-endian.

| Offset | Size | What it is |
|---|---|---|
| `0x00` | 4 bytes | `OBJ_` |
| `0x04` | 4 bytes | Record count |
| `0x08` | 4 bytes | Record size in bytes: 60 in every shipped file |
| `0x0c` | count × size | The records |
| after | | An `INCL` chunk: a count, then that many strings, each a 4-byte length and the bytes with their NUL. The loader never reads it |

The loader reads the first 60 bytes of each record into fifteen 32-bit words it has cleared first, so a
shorter record's missing words are nought, and steps over the rest of a longer one. A file that does not open
`OBJ_` is left alone.

The `INCL` strings are the C headers the file was built from: `D:\Park2\data\Particle\par_lib.h`,
`D:\park2\Source\game\SoundInt\Events\SfxEvent.h` and the theme's own, such as
`D:\Park2\Source\game\SoundInt\ThemedEvents\JungleEvent.h`.

## A record

Word 0 is the record's type. The loader knows three and passes over any other.

| Type | What the loader does |
|---|---|
| 1 | Starts a sound: word 1 picks one of two sound categories (nought or not), word 2 is the effect in it, words 3, 4 and 5 are x, the height and z, and word 6 is doubled; each of the four is divided by 1024 toward nought |
| 2 | Starts a particle effect: `Particles_Spawn( word 1, word 3, word 4, word 5 )` (`0x00551006`). Word 1 is the effect in the [particle library](../particles/); words 3, 4 and 5 are x, the height and z in 1024ths of a park unit (ten units to a map cell) |
| 3 | Starts a sound as type 1 does, without word 6, then hands its voice words 3, 5, 7 and 8, each over 1024 |

What word 6 of a type 1 and words 7 and 8 of a type 3 mean to the sound system is not decoded.

## Every record the game ships

The first nine words of each; the rest are nought.

| Park | Type | Word 1 | Word 2 | Words 3, 4, 5 | Words 6, 7, 8 |
|---|---|---|---|---|---|
| jungle | 3 | 0 | 2 | 362100, 0, 32100 | 80000, 100000, -100000 |
| jungle | 3 | 0 | 3 | 381600, 0, 21600 | 0, 200000, -200000 |
| jungle | 2 | 20 | 0 | 545100, 5000, 538700 | 0, 0, 0 |
| jungle | 1 | 1 | 181 | 21700, 0, 40700 | 10000, 0, 0 |
| jungle | 1 | 1 | 182 | 20200, 0, 70700 | 10000, 0, 0 |
| jungle | 1 | 0 | 8 | 543500, 0, 525500 | 50000, 0, 0 |
| jungle | 1 | 0 | 7 | 523400, 0, 680700 | 50000, 0, 0 |
| fantasy | 3 | 0 | 3 | 379300, 0, 30400 | 80000, 200000, -200000 |
| fantasy | 1 | 0 | 2 | 408200, 20000, 16200 | 70000, 0, 0 |
| fantasy | 1 | 1 | 113 | 23800, 0, 52100 | 10000, 0, 0 |
| fantasy | 1 | 1 | 114 | 23800, 0, 24200 | 10000, 0, 0 |
| hallow | 3 | 0 | 3 | 380800, 0, 33400 | 80000, 200000, -200000 |
| hallow | 1 | 0 | 2 | 407800, 20000, 15400 | 70000, 0, 0 |
| hallow | 1 | 1 | 144 | 12700, 0, 46700 | 10000, 0, 0 |
| hallow | 1 | 1 | 143 | 12100, 0, 75500 | 10000, 0, 0 |
| space | 3 | 0 | 3 | 382300, 0, 31900 | 80000, 200000, -200000 |
| space | 1 | 0 | 2 | 419800, 20000, 3700 | 70000, 0, 0 |
| space | 1 | 1 | 143 | 40900, 0, 39500 | 10000, 0, 0 |
| space | 1 | 1 | 144 | 41800, 0, 4400 | 10000, 0, 0 |

## The jungle's waterfall

The one type 2 record names effect 20, `WaterFall`, at (545100, 5000, 538700): 532.3 across, 4.9 up and
526.1 along in park units. It is the first effect a park's particle system starts, so the emitter stands in
slot 0 under the handle `0x10000`, and the shipped `Easymode.TPWI` holds it there, at (532, 4, 526), with 24
particles ([park files](../saves/#a-live-emitter-and-its-handle)).
