---
title: Sound categories (cat_*.map)
---

The game does not play sound files, it plays **effects**. Sounds are grouped into named categories,
each described by a pair of `.map` files sitting beside the [`.sdt`](/formats/sounds/) banks they
draw on, and code asks for an effect id within a category. So the lobby's thunder is "global lobby
sfx, effect 1"; there is nothing in the executable that names a `Thunder2.mp2`.

Each category is two files:

| File | Holds |
| --- | --- |
| `cat_<name>BANK.map` | which `.sdt` banks the category draws on |
| `cat_<name>SFX.map` | the effects in it, and which samples each one picks from |

The categories that ship are `ambient`, `globallobbysfx`, `kids`, `locallobbymusic`,
`locallobbysfx`, `music`, `rides`, `speech`, `staff` and `ui`, some at `data\global` and some per
level under `data\levels\<park>`.

> **How much of this is established.** Both files are decoded to the last byte. All thirty-one BANK
> files parse to exactly the end of the file with nothing over. All thirty-one SFX files do too,
> walked the way the game's loader walks them: 1,267 effects and 1,595 variations. A few fields are
> still unnamed, and are marked. The sample records can also be found by validation instead, and
> both ways find the same lists.

## Common header

Both files start the same way.

| Offset | Size | Description |
| --- | --- | --- |
| `0x00` | 16 bytes | A GUID saying which of the two this is |
| `0x10` | 4 bytes | Zero in every BANK file. In an SFX file, 1 where the variation weights are stored as shares (global speech and the four park music categories) and 0 where they are running totals — see below |
| `0x14` | 4 bytes | Zero in most files, never read by the loader. The global speech and speech maps carry odd leftovers here |
| `0x18` | 4 bytes | A count — banks in a BANK file; in an SFX file, groups of effects, 1 in every file |

The two GUIDs are `{E9612C01-31D0-11D2-B409-00A0C993F203}` for a BANK file and
`{E9612C00-31D0-11D2-B409-00B0C993F203}` for an SFX file. They differ in the low byte of `Data1`
and one byte of `Data4`.

## The BANK file

After the count come that many **11-byte records** that hold nothing useful — the same handful of
values recur across every file, which is what a struct written straight out of memory looks like.
Then the payload: one length-prefixed, NUL-terminated path per bank.

| Size | Description |
| --- | --- |
| 4 bytes | Length of the path that follows, terminator included |
| n bytes | Path, NUL-terminated |

A path is relative to the **level** folder rather than to the folder the `.map` file is in, and it
omits the `HD.sdt` suffix the file actually carries. So `Sound\LobbySfx`, read from
`data\levels\jungle\Sound\cat_locallobbysfxBANK.map`, means
`data\levels\jungle\Sound\LobbySfxHD.sdt`.

The paths are also not reliably cased: the global category asks for `Sound\Sfx` when the folder is
called `sound`, and hallow's rides ask for `Sound\ride` when the file is `rideHD.sdt`. On the
case-insensitive filesystem this was authored on, neither mattered.

Bank order matters — sample records index into this list.

## The SFX file

### Header

After the common header, the SFX file's one group has a 24-byte header:

| Offset | Size | Description |
| --- | --- | --- |
| `0x1c` | 4 bytes | Effect count |
| `0x20` | 4 bytes | Zero |
| `0x24` | 2 bytes | A class mask: 9 in the sound-effect categories, 5 in speech, 3 in music. The game tests it against the groups the player has switched on, for effects that play once |
| `0x26` | 2 bytes | 1, except in the lobby's own categories (`globallobbysfx`, `locallobbysfx`, `locallobbymusic`), where it is 0 |
| `0x28` | 12 bytes | Three floats, meaning unknown: (1.0, 2.0, 0.5) in a park's categories, (1.0, 0.0, 0.0) in the lobby's, (0.0, 1.0, 1.0) in speech |

### Effect table

`0x34` onwards, one 20-byte record per effect:

| Offset | Size | Description |
| --- | --- | --- |
| `+0x00` | 4 bytes | **Effect id** — what code passes to the play call |
| `+0x04` | 4 bytes | **Variation count** — how many weighted sample lists this effect picks between |
| `+0x08` | 4 bytes | Zero. The game writes a pointer to the effect's variations here when it loads the file |
| `+0x0c` | 4 bytes | **Priority** |
| `+0x10` | 2 bytes | **Flags** — what kind of voice plays the effect |
| `+0x12` | 1 byte | **Parameter id** — which of a voice's parameters drives it, or 0 |
| `+0x13` | 1 byte | Zero |

`+0x04` runs **0 to 30** across the 1,267 records. Jungle's ambient declares 9, 5, 7, 4, 1, 0, 1, 1, 1,
which sums to exactly the twenty-nine variations that follow it. **A zero is real**, meaning an effect
with nothing to play; hallow's, jungle's and space's ambient each carry one.

**`+0x0c` is a priority, not a repeat delay.** This page used to call it a delay in milliseconds, and
the numbers invite it: 2700 for every kids scream, 6000 for speech, 10,000 for music. They are not all
round, though (999, 4999, 5310, 5320 and 5999 ship). The game reads the field in two places and never
as a time:
- it becomes the high half of the key that ranks voices when there are more than the mixer will play;
- a new effect may take over a caller's existing voice only if its priority is higher.

Nothing in the game waits on it.

**`+0x10` is a flags word, and `+0x12` a parameter id.** This page used to call the dword at `+0x10`
unknown, and to refute one reading of it: that it was a loop flag, which it looks like in `cat_kids`
alone (`0x00060404` for the four held screams, 0 for one-shots). That refutation stands, but the field
is decoded now:
- **Bit `0x4` clear** means an effect that plays one sample and is done: 1,067 of the 1,267 records.
- **`0x0404`** is a voice that keeps playing fresh samples until it is stopped, each after a wait taken
  from its variation (below). There are 18 of these: kids 71-74, staff 188 and several ambient beds.
- **Bits `0x4` and `0x2`** mark a different repeating voice (135 records, music effect 2 among them).
- **Bit `0x1`** marks 30 records the game queues before playing.

The byte at `+0x12` is non-zero on exactly the 45 records that have bit `0x400`. It names the
parameter a game call sets to steer the voice: 6 for the screams, 4 for music, 7 for kids 91. The
rest of the flags distribution: `0x200` on 651 records (all 645 speech effects, and six ui ones), `0`
on 331, `6` on 91, `8` on 83, `0x201` on 29, `4` on 16, `0x206` on 15.

Ids are not indices: a category's need not start at 1 or run in order, and the ride categories' run
into the hundreds.

### Variations, samples and zones

After the effect table, each effect in turn holds:
- its variation headers, one 42-byte header per variation, as many as `+0x04` says;
- then, for each of its variations, that variation's sample records followed by its zone records.

That is the whole rest of the file.

#### Variation header

| Offset | Size | Description |
| --- | --- | --- |
| `+0x00` | 2 bytes | Sample records that follow |
| `+0x02` | 2 bytes | Zero |
| `+0x04` | 4 bytes | Zone records that follow the samples |
| `+0x08` | 4 bytes | Zero. The game writes a pointer to the samples here when it loads the file |
| `+0x0c` | 2 bytes | Two unsigned bytes, a low and a high: 47 and 47 in every scream variation. Not named |
| `+0x0e` | 2 bytes | Two signed bytes, a low and a high: 0 and 6 in every scream variation. Not named |
| `+0x10` | 2 bytes | **Shortest wait**, in ms, before a repeating voice starts its next sample |
| `+0x12` | 2 bytes | **Longest wait**, in ms |
| `+0x14` | 4 bytes | Not named |
| `+0x18` | 2 bytes | A mask. With bit 4 set, the wait is taken from the voice's parameter rather than at random |
| `+0x1a` | 2 bytes | Not named |
| `+0x1c` | 2 bytes | A second mask, not named |
| `+0x1e` | 4 bytes | **Weight** among the effect's variations |
| `+0x22` | 4 bytes | A float: 95.0 in every scream variation. Not named |
| `+0x26` | 4 bytes | Zero. The game writes a pointer to the zones here |

**The wait** is drawn fresh for each sample, from the shortest up to but not including the longest,
and is counted from when that sample starts. The sample's length does not come into it. Nought and
nought means no wait. The four held screams carry these waits in all four of their variations:

| Effect | Wait |
| --- | --- |
| 71 | 1000 to 3000 ms |
| 72 | 500 to 2000 ms |
| 73 | 100 to 1000 ms |
| 74 | 0 to 500 ms |

Four ride variations hold the pair the wrong way round (fantasy 156, hallow 187, jungle 216 and
space 188, 100 to 0), and fourteen hold both ends equal.

**The weight** is a running total across the effect's variations when the common header's `0x10` is
0, and a variation's own share when it is 1. The game turns running totals into shares as it loads.
**Variations are therefore weighted, not even.** This page used to say the opposite. For most effects
the shares happen to be equal, so it made no difference, but 4 of the 63 effects with more than one
variation are uneven. Global ambient 172, for instance, gives its last two variations a tenth of the
others' share.

#### Sample record

| Offset | Size | Description |
| --- | --- | --- |
| `+0x00` | 4 bytes | Sample index within its bank, **1-based** |
| `+0x04` | 2 bytes | Cumulative odds, out of 65,535 |
| `+0x06` | 2 bytes | Zero |
| `+0x08` | 4 bytes | The sample's length in milliseconds |
| `+0x0c` | 4 bytes | Bank index, **1-based**, into the BANK file's list |

#### Zone record

| Offset | Size | Description |
| --- | --- | --- |
| `+0x00` | 4 bytes | A variation of the same effect, **1-based** |
| `+0x04` | 2 bytes | Zero |
| `+0x06` | 1 byte | Low end of a parameter range |
| `+0x07` | 1 byte | High end of that range, inclusive |

A repeating voice's first sample comes from the effect's first variation. Each later one comes from
a variation that the current variation's zones allow for the voice's parameter at that moment, drawn
by weight when more than one fits. So zones are two things in the shipped data:
- **A switch.** In all four held screams, every variation lists variation 1 for 0-25, 2 for 26-50,
  3 for 51-75 and 4 for 76-100. The game sets the scream's parameter from the ride's own numbers.
- **A no-repeat rule.** In jungle's ambient effect 177, each of the nine variations lists the other
  eight, over the whole of 0-100.

1,222 of the 1,595 variations have no zones at all.

### Finding the sample records without the headers

This page used to locate the sample records by what they contain, before the variation headers were
decoded, and the method still works. Every list it finds is the
same as the walk above. A 16-byte window is a sample record if:
- its bank and index are both in range;
- its two spare bytes are zero;
- its odds are inside 1..65,535;
- **the length it claims matches the length of the sample it points at.**

Four independent fields agreeing is not something arbitrary bytes do. The map's length and a real
decode disagree by up to 45 ms (the map is rounded down, and the decoder pads the final frame), so
that last test wants a tolerance of a few per cent.

Two rules then divide such a run of records up.

A **list** ends when its odds saturate, since they are cumulative: a list of six reads 10922, 21844,
32766, 43688, 54610, 65532. Rounding leaves that last figure anywhere from 65,529 to 65,535, while
the largest that is *not* the end of a list is 58,248 — a wide gap to put a threshold in.

An **effect** ends once it has taken as many lists as its own record declares at **`+0x04`**. Read the
count out of the file; do not try to infer the boundary from the gap before the next list.

> **The gap rule this page used to give does not work, and it fails on shipped data.** It said an
> effect ends "when the space before the next list is big enough to be a new effect's header", with
> lists inside one effect 0, 16 or 24 bytes apart and a new effect's 42, 58, 84, 168 or 210 bytes
> apart. **Those two ranges overlap.** Jungle's ambient runs **64** bytes between two lists of the
> *same* effect, while the smallest gap between two *different* effects is **42** — so no threshold
> separates them. The lobby never noticed, because all four of its local sfx categories and all four
> of its music ones declare a single variation each and group identically either way. Every park
> category did not: jungle's nine ambient effects came out as twenty-six, which left effects 178 to
> 192 each playing one of effect 177's beasts, and the global lobby and UI categories were wrong too.

So an effect holding more than one list is picking between **variations**, and how many it holds is
stated in the file rather than inferred — and weighted as their headers say.

### Worked example

`data\global\sound\cat_globallobbysfx*.map`, the category the lobby's thunder comes from. Its BANK
file names three banks — `Sound\Sfx`, `Sound\Music`, `Sound\sSfx` — and its effect table names four
effects with priorities of 1000, 6000, 5000 and 3000. Applying the rules above:

| Effect | Priority | Samples |
| --- | --- | --- |
| 1 | 1000 | `Thunder2`, `Thunder3`, `Thunder4`, `massivlg` — evenly weighted |
| 2 | 6000 | `level4c`, `level4w` from the global music bank |
| 3 | 5000 | `ew_space_1` to `ew_space_4`, as four one-sample variations |
| 4 | 3000 | none — the table names it, the file has no list for it |

Effect 1 is what the lobby plays on a lightning strike.

One thing that table hides. The `level4c` in the global music bank is the same recording as the
`level4c` in Space Zone's own lobby music bank: the two files differ by eleven bytes out of
253,807, all of them inside the final MPEG frame, and the decoded audio correlates at 1.0000. So a
lobby that plays effect 2 as written puts one particular park's theme over whichever park is
actually on screen. Worth knowing before reading a name that appears in two banks as a
coincidence — across the game, a repeated name usually is the same audio.

## Speech categories

The `speech` categories are shaped differently from every other one, and are simple enough to be
worth calling out. `data\global\Speech\cat_speechSFX.map` names **641 effects, each with exactly
one sample and one variation**, and the effect table is completely uniform — every record carries
the same priority, 6000, and the same flags, `0x200`. Effect *n* is sample *n*, so a category that
elsewhere means "pick one of these at these odds" here degenerates into a flat index into the
bank. The BANK file names a single bank, `speech\speech`.

Each park also has its own
`data\levels\<park>\Speech`, and those hold **exactly one sample each** — the park's introduction,
between 16 and 27 seconds long — which is why a park's `lips` folder contains only `sp_001.LIP`.

### Which sample is which line

Nothing in the game data says what a speech sample contains. The advisor does not ask for lines by
sample number: it asks by **response id**, and the mapping from response id to sample lives in a
table compiled into the executable, not in any file under `data`. In the US English build it is
610 records of 32 bytes each, terminated by a response id of 9999, whose fields are the response
id, the sample number, the lip-sync file number, an animation, a word whose high half selects a
park's own speech category over the global one, two more values, and a message group.

The ids are close to the sample numbers but not equal to them, so they cannot be guessed. The
front end's greeting, for example — played on the player-slot screen when no slot holds a player —
is response 390 followed by 391, which are samples 465 (*"Welcome to Sim Theme Park! I'm the
advisor around here..."*) and 466 (*"...I don't even know your name!... click on the New Player
button"*). With a player already saved it is response 398, sample 471 (*"...Don't I know you?"*).
A reimplementation needs a copy of the table, or at least of the entries it uses.

### Empty entries

80 of the 641 samples in the global speech bank are not audio. They are all byte-for-byte the same
315-byte blob, which begins with a valid MPEG sync word (`FF F5`) but holds less than two complete
frames; ffmpeg rejects it, and so will any other decoder. They decode to a nominal 0.019 s against
a median of 5.7 s for the real samples.

These are response ids with nothing recorded against them rather than corruption — the bank is
sized for the full set of ids and the gaps were never filled. A loader should treat a sample this
short as an empty slot and stay quiet about it, not report 80 broken files.
