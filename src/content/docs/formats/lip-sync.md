---
title: Lip Sync (*.lip)
---

A `.lip` file says when the advisor's mouth should be moving during one line of speech. There is
one per speech sample, named after the sample: `data\global\Speech\lips.WAD` holds `sp_001.LIP`
to `sp_637.LIP` for the global speech bank, and each park's `Speech\lips` folder holds the one
for its introduction. The game builds the name from the sample number as `\Speech\lips\sp_%03d.lip`.

> `lips.WAD` is the only archive in the game whose extension is in capitals. A loader that finds
> archives by appending a lower-case `.wad` and checking the file exists will not see it on a
> case-sensitive filesystem.

## Format

The whole file is a list of little-endian `uint32` timestamps, in **microseconds** from the start
of the sample, in ascending order, ending with `0xFFFFFFFF`. There is no header.

| Size | Description |
| --- | --- |
| 4 bytes | Timestamp, microseconds from the start of the sample |
| ... | ...repeated... |
| 4 bytes | `0xFFFFFFFF` — end of list |

The unit is microseconds: Lost Kingdom's introduction lasts 25.65 seconds and its last
timestamp is 25,327,573. The game divides each value by 1000 to compare against its millisecond
clock.

## Meaning

Each timestamp **toggles** the mouth between talking and quiet.

Which state comes first is easy to get backwards. The game marks the advisor as talking when his
sample starts, so he **talks from the start of the sample until the first timestamp**, and after
every even number of timestamps. After the last timestamp the game stops reading the file and
treats the mouth as quiet.

Speech starts almost at once - a median of 20 milliseconds into the sample - so the first
timestamp is usually the end of his first phrase, not its start. Only 18 of the game's 641 files
begin with a timestamp of `0`, which ends a talking stretch before it has begun.

That was checked against the audio, not just read from the code. Across the 557 global samples
that decode and have a lip file, reading the files this way agrees with where the speech actually
is in 87.6% of 20-millisecond slices of audio, against 15.4% for the opposite reading, and its
talking stretches are the louder ones in 553 of the 557.

## How the game uses it

While a line's lip file says the advisor is talking, the game gives him one of his five mouths at
random every 100 milliseconds. Otherwise he gets the first, his mouth at rest. The mouths are
separate meshes in the advisor's model, found by name ignoring case — `mouth - normal`,
`mouth - aah`, `mouth - eee`, `mouth - ooh` and `mouth - sss` — and switched by hiding all but
one.
