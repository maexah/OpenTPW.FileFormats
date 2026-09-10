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
| `+0x04` | 4 bytes | Unknown — 1 to 5 |
| `+0x08` | 4 bytes | Zero |
| `+0x0c` | 4 bytes | **Repeat delay**, in milliseconds |
| `+0x10` | 4 bytes | Unknown |

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

An **effect** ends when the space before the next list is big enough to be a new effect's header.
These are well separated too: a list following another inside the same effect starts 0, 16 or 24
bytes later, and a list starting a new effect starts 42, 58, 84, 168 or 210 bytes later.

An effect holding more than one list appears to be picking between variations. Nothing found so far
weights the lists against each other.

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
