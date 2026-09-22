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

> **How much of this is established.** The BANK file is fully decoded — all thirty that ship parse
> to exactly the end of the file with nothing over. The SFX file's header, effect table and sample
> records are decoded and cross-check against the banks; what is **not** decoded is the per-effect
> header that sits in front of each sample list, which varies in size. That is why the sample
> records below are located by validation rather than by offset.

## Common header

Both files start the same way.

| Offset | Size | Description |
| --- | --- | --- |
| `0x00` | 16 bytes | A GUID saying which of the two this is |
| `0x10` | 8 bytes | Zero in every file |
| `0x18` | 4 bytes | A count — banks in a BANK file, effects in an SFX file |

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

| Offset | Size | Description |
| --- | --- | --- |
| `0x18` | 4 bytes | Version — 1 in every file |
| `0x1c` | 4 bytes | Effect count |
| `0x20` | 4 bytes | Zero |
| `0x24` | 4 bytes | A sample count, but not one that matches what the file holds |
| `0x28` | 4 bytes | Float, 1.0 in every file |
| `0x2c` | 8 bytes | Zero |

### Effect table

`0x34` onwards, one 20-byte record per effect:

| Offset | Size | Description |
| --- | --- | --- |
| `+0x00` | 4 bytes | **Effect id** — what code passes to the play call |
| `+0x04` | 4 bytes | **Variation count** — how many weighted sample lists this effect picks between |
| `+0x08` | 4 bytes | Zero |
| `+0x0c` | 4 bytes | **Repeat delay**, in milliseconds |
| `+0x10` | 4 bytes | Unknown — but **not** a loop flag; see below |

`+0x04` was recorded here as "Unknown — 1 to 5", and both halves of that were wrong. It is the number
of weighted sample lists the effect picks between, and across the **1,267** effect records in all
**31** categories the game ships it runs **0 to 30**. Jungle's ambient alone declares 9, 5, 7, 4, 1,
0, 1, 1, 1 — which sums to exactly the twenty-nine lists that follow it, and that agreement over nine
effects is what identifies the field. **A zero is real**, meaning an effect with nothing to play;
hallow's, jungle's and space's ambient each carry one. This count is also what divides the sample
records up between effects — see below.

`+0x10` is still undecoded, and one tempting reading is **refuted** rather than left open. In
`cat_kids` alone it looks exactly like a loop flag: `0` for every one-shot scream effect, and
`0x00060404` for all four of the held, repeating ones. It is not one. Swept across all 1,267 records,
**45** carry a value at or above `0x10000`, and they include the **music** category's effect 2 — which
the game *replays* rather than loops, waiting out its own 10,000 ms delay between arrangements — and
**ui** effects 154 and 155, which are button sounds. Whatever the field means, it does not mean "this
effect repeats". The rest of the distribution: `0x200` on 651 records, `0` on 331, `6` on 91, `8` on
83, `0x201` on 29, `4` on 16, `0x206` on 15. The 651 are mostly one category — speech declares 641
uniform records, each carrying that same trailing `0x200` (see below).

Ids are not indices: a category's need not start at 1 or run in order, and the ride categories' run
into the hundreds. The delays are all round numbers between 700 and 10,000 ms.

The delay is counted from where a pick would *finish*, not from where it starts. It has to be: the
lobby music effect's delay is four seconds against tracks that run seventeen and a half, so read as
a gap between starts it would stack three copies of a park's theme on top of each other.

### Sample records

After the effect table the file alternates between per-effect headers of unknown, varying size and
runs of 16-byte sample records:

| Offset | Size | Description |
| --- | --- | --- |
| `+0x00` | 4 bytes | Sample index within its bank, **1-based** |
| `+0x04` | 2 bytes | Cumulative odds, out of 65,535 |
| `+0x06` | 2 bytes | Zero |
| `+0x08` | 4 bytes | The sample's length in milliseconds |
| `+0x0c` | 4 bytes | Bank index, **1-based**, into the BANK file's list |

Because the headers between them vary in size, the records cannot be stepped to. They can be found
by what they contain instead: a 16-byte window is a sample record if its bank and index are both in
range, its two spare bytes are zero, its odds are inside 1..65,535, and **the length it claims
matches the length of the sample it points at**. Four independent fields agreeing is not something
arbitrary bytes do. The map's length and a real decode disagree by up to 45 ms — the map is rounded
down and the decoder pads the final frame — so that last test wants a tolerance of a few per cent.

Two rules then divide the run of records up.

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
stated in the file rather than inferred. Nothing weights the variations against each other, so they
are even — which for hallow means its rain, its terrors, its three pairs of bats and its spirit each
get a turn, rather than the bats crowding everything else out.

### Worked example

`data\global\sound\cat_globallobbysfx*.map`, the category the lobby's thunder comes from. Its BANK
file names three banks — `Sound\Sfx`, `Sound\Music`, `Sound\sSfx` — and its effect table names four
effects with delays of 1000, 6000, 5000 and 3000 ms. Applying the rules above:

| Effect | Delay | Samples |
| --- | --- | --- |
| 1 | 1000 ms | `Thunder2`, `Thunder3`, `Thunder4`, `massivlg` — evenly weighted |
| 2 | 6000 ms | `level4c`, `level4w` from the global music bank |
| 3 | 5000 ms | `ew_space_1` to `ew_space_4`, as four one-sample variations |
| 4 | 3000 ms | none — the table names it, the file has no list for it |

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
the same 6000 ms delay and the same trailing `0x200`. Effect *n* is sample *n*, so a category that
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
