---
title: Settings and Modifiers (*.sam)
---

SAM files are plain-text files that specify values for different parts of the game. These files can be found both within archives and within the base directories for Theme Park World.  

A typical SAM file looks like this:

```text
# Sell 30 Drinks in 60 days
Challenges[1].Type                  3
Challenges[1].FollowupType          0
Challenges[1].TargetTime            60
Challenges[1].TargetVal             30
Challenges[1].TargetObj             0
Challenges[1].TargetObj2            0
Challenges[1].TargetStaffType       0
Challenges[1].Prize                 5000
Challenges[1].CheckAtEndOnly        0
Challenges[1].Independent           1
```

## Format

SAM files follow the format of `key <whitespace> value`.

Comments are preceded with a pound symbol (`#`) and continue until the end of the line.

Strings are surrounded with double quotes (`"`) and are used for various properties, i.e. the ride's name.

## `data\sound.sam`

Worth singling out because it holds the mix the game starts with, and because one of its values
is not what its own header comment implies.

The file says of itself that "all the values correspond to detail values at which they are
enabled" — that is, most `SoundInfo` keys are thresholds on the 0–100 sound-detail slider
(`QSOUND 15`, `EAX 10`, `GEOMETRY 20`, `MPEGRATE 35`, and so on).

`DefaultVolume` is not a threshold. It is the starting position of the four volume sliders, out
of 100:

| Key | Default |
| --- | --- |
| `DefaultVolume.SFX` | 75 |
| `DefaultVolume.MUSIC` | 60 |
| `DefaultVolume.SPEECH` | 75 |
| `DefaultVolume.MOVIE` | 100 |

Those four groups are also how the game attenuates itself while the advisor talks: it multiplies
the music and SFX group volumes by a percentage and leaves speech alone, restoring them when the
sample ends.

**Open question.** `SoundInfo.DUCKINGLEVEL` is `38`, and it is the only ducking-related key in the
file. The percentage the engine multiplies by lives in memory that is zeroed in the image and has
no writer the decompiler can see, so it is filled in from configuration at runtime — which makes
`DUCKINGLEVEL` the only candidate, and would mean the mix drops to 38% while the advisor speaks.
But that reading contradicts the file's own header comment, under which `38` would instead be the
detail level at which ducking switches on. Not yet resolved either way.
