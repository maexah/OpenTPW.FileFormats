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
sample starts, and every file begins with a timestamp of `0`, which toggles that straight back to
quiet. So the stretches that **begin at the odd-numbered timestamps** are the talking ones. After
the last timestamp the game stops reading the file and treats the mouth as quiet.

That was checked against the audio, not just read from the code. Across Lost Kingdom's
introduction, the stretches starting at odd timestamps have an average loudness (RMS) of 0.12
against 0.013 for the even ones, and they match where the speech actually is 84% of the time,
against 16% for the opposite reading.

## How the game uses it

While a line's lip file says the advisor is talking, the game gives him one of his five mouths at
random every 100 milliseconds. Otherwise he gets the first, his mouth at rest. The mouths are
separate meshes in the advisor's model, found by name ignoring case — `mouth - normal`,
`mouth - aah`, `mouth - eee`, `mouth - ooh` and `mouth - sss` — and switched by hiding all but
one.
