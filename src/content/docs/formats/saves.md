---
title: Saves (*.tpws, *.ints, *.lays)
---

Theme Park World stores saves with three different extensions.

These extensions are:

- **TPWS**: Standard save
- **INTS**: Initial save (e.g. for *Instant Action* mode)
- **LAYS**: Online save (for uploading parks)

## File format

**Header**

| Size      | Description                  |
| --------- | ---------------------------- |
| 4 bytes   | Magic number - `F4 01 00 00` |
| 823 bytes | Copyright Notice             |
| 711 bytes | Padding                      |

**File info**

| Size    | Description                                     |
| ------- | ----------------------------------------------- |
| 4 bytes | File type - `00 01 22 19`                       |
| 1 byte  | File version - `85`                             |
| 1 byte  | Online flag - `00` for offline, `01` for online |
| 2 bytes | Padding                                         |

**Data (compressed using ZLIB)**

| Size     | Description           |
| -------- | --------------------- |
| 4 bytes  | Magic number - `BILZ` |
| 4 bytes  | Unknown               |
| 4 bytes  | Compressed length     |
| 16 bytes | Unknown               |

The ZLIB stream begins after this point, and continues to the end of the file.

## Inside the stream: a chain of modules

The inflated payload is **not** one structure. It is a run of modules, each written by the subsystem
that owns it and each followed by a **four-character tag** the game checks on the way back in — so a
reader that lands exactly on the next tag has agreed with the game about every byte in between.

> The tags are compared as **dwords**, not as text, so they are stored little-endian and read
> **backwards** in a hex dump: `WRLD` appears as `DLRW`, `RSYS` as `SYSR`. Searching a dump for a tag
> the right way round finds nothing at all, which reads exactly like proof the module is absent. The
> one exception is the script module's own header magic below, which really is stored forwards.

The order is the game's own. The offsets are measured in `data/levels/jungle/Easymode.TPWI`, whose
stream inflates to 1,608,309 bytes, and are that file's rather than a general layout.

| Tag (as stored) | Module                                                         | Tag at    |
| --------------- | -------------------------------------------------------------- | --------- |
| `DLRW`          | World — the map and everything standing in the park             | 1,495,462 |
| `CSPS`          | Sprite scripts                                                 | 1,500,918 |
| `TRAP`          | Particles                                                      | 1,577,174 |
| `SSEM`          | Message centre                                                 | 1,577,444 |
| `KOLC`          | Clock                                                          | 1,577,456 |
| `TNAV`          | "Vanilla time"                                                 | 1,577,464 |
| `SYSG`          | Game system                                                    | 1,577,504 |
| `SYSR`          | Ride system                                                    | 1,595,034 |
| `KART`          | Track rides                                                    | 1,595,082 |
| `RYLF`          | Flying rides                                                   | 1,595,538 |
| `ESSR`          | Ride scripts                                                   | 1,606,398 |
| `EMAK`          | Camera                                                         | 1,606,442 |
| `SAOC`          | Coasters                                                       | 1,606,462 |
| `SVDA`          | Advisor                                                        | 1,606,838 |
| `NUOS`          | Sound                                                          | 1,608,287 |
| `STHC`          | Cheats                                                         | 1,608,293 |
| `CSDA`          | Advisor scoring                                                | 1,608,301 |

Each tag *follows* the module it belongs to. The World module does not begin at the start of the
stream either: an untagged action recording is written first, as a flag and then a length with its
bytes, so where World starts is derived from that length rather than assumed.

## The clock module (`KOLC`)

Two dwords, between the message centre's `SSEM` tag and the clock's own `KOLC`, so the second tag sits exactly
twelve bytes after the first:

| Size    | Description                                                                                   |
| ------- | --------------------------------------------------------------------------------------------- |
| 4 bytes | The game clock's reading at the save, in milliseconds - the clock the ride scripts and the animation channels read |
| 4 bytes | A second stopwatch's reading, which an animation channel carrying flag `0x40` reads instead    |

The game makes both clocks read these values again when it loads a park, so every other reading of the clock in the
file - a script's deadlines, a channel's time stamps - keeps its distance from the save's own moment. In
`Easymode.TPWI` they are `0x06D13894` (114,374,804) and `0x06D8DF7E` (114,876,286); in a played jungle park,
`0x00498D70` (4,820,336, about 80 minutes) and `0x0049A676` (4,826,742).

## The ride script module (`ESSR`)

Every script that was running when the park was saved is stored here, program counter and all — which
is what lets a loaded park resume instead of starting over. It begins immediately after the `RYLF` tag
closing the module before it, and unlike the tags its own magic is stored **forwards**.

| Size     | Description                                                     |
| -------- | --------------------------------------------------------------- |
| 4 bytes  | Magic — `RSSE` (`52 53 53 45`)                                  |
| 4 bytes  | Header length                                                   |
| n bytes  | Header — the subsystem's own globals, not per-script: see below |
| 20 bytes | Five dwords the loader reads and discards                       |
| 4 bytes  | Script count — **14** in the shipped park                       |
| 4 bytes  | Bytes per script struct — **244** in the shipped park           |

The header is the script subsystem's five global dwords, which the game reads straight back over its own: a flag that
the subsystem is set up, the tick counter behind every script's one-in-eight turn, the handle the next new script will
be given, the script count, and a list pointer from the saving session that means nothing on loading. The shipped
park's read 1, 6,055, 16, 14 and a pointer. So a loaded park goes on counting ticks, and numbering new scripts, where
the saved one left off.

Then one record per script. Each is a struct followed by a run of length-prefixed blocks, and a record
**must** end on the literal guard `OBJ ` — the game refuses the load without it, logging
`RSSE: Load Fail - Object list missing`, which is what makes a walk of this module self-checking.

| Size     | Description                                                             |
| -------- | ----------------------------------------------------------------------- |
| 244      | The script struct — see below                                           |
| 4 + n    | The script body, as words                                               |
| 4 + n    | **Block 1 — the stack**, one dword per slot; empty with no `#setstack`  |
| 4 + n    | **Block 2 — the variables**, one dword per slot                         |
| 4 + n    | **Block 3 — the string blob**, byte for byte the `.RSE` file's          |
| 4 + n    | Blocks 4 and 5                                                          |
| 4 + n×32 | A count, and that many 32-byte records                                  |
| 4 + n    | Two further blocks                                                      |
| 4 bytes  | Guard — `OBJ `                                                          |
| 4 bytes  | Object count                                                            |
| 4 bytes  | Bytes per object                                                        |
| n        | The script's own object list                                            |

Fourteen fields of the struct are established:

| Offset | Dword | Description                                                                     |
| ------ | ----- | ------------------------------------------------------------------------------- |
| `0x08` | 2     | The script handle — the number an object record's `mRideScriptHandle` also holds |
| `0x3c` | 15    | The program counter, in **words** from the start of the body                     |
| `0x40` | 16    | The call index — the stack slot the next `JSR` writes, counting down              |
| `0x44` | 17    | The heap index — how many values `HUSH` has pushed, counting up                   |
| `0x48` | 18    | The result register, which the conditional branches test                         |
| `0x50` | 20    | The body length in words, which the counter is bounds-checked against            |
| `0x54` | 21    | The stack size in dwords — block 1's length over four                             |
| `0xa0` | 40    | The deadline a `WAIT`, `WAITABS`, `WAITANIM` or `WAITANIM_CH` is sitting on, a clock reading; nought for none |
| `0xa4` | 41    | The deadline the last trigger armed, a clock reading, which `WAIT4ANIM` waits on. It is not cleared when it passes, only by a passed `WAIT4ANIM`, a `WAITANIM`'s first visit (or a `WAITANIM_CH`'s), a `LOOPANIM` that starts a loop or a `LOOPANIM_CH`, so a save can hold one long past; nought for none |
| `0xa8` | 42    | The looping key: `(entry << 16) + role` of the last `LOOPANIM`, or `0xffff` from creation, a `TRIGANIM`, `TRIGANIMSPEED`, `TRIGWAITANIM` or `WAITANIM`; the `_CH` forms leave it alone |
| `0xbc` | 47    | `TRIGWAITANIM`'s mark (shared with `TRIGWAITANIM_CH`): the role it is waiting for, plus one; nought when not waiting |
| `0xc4` | 49    | The deadline the last `SETTIMER` set, a clock reading, not scaled by the speed word; nought until one runs |
| `0xe0` | 56    | A coaster script's ride handle, which its `COAST 8` stores; nought before that   |
| `0xe4` | 57 (low word) | A 16-bit play rate in thousandths: 1000, or a `TRIGANIMSPEED`'s rate. The word above it, `0xe6`, is a separate field; it reads `0xffff` in every script of the shipped park |

**How those were pinned rather than guessed.** Dword 20 equals the length of the body block that
follows it for all fourteen scripts, which fixes the struct's alignment and with it every offset in the
table; every one of the fourteen counters then lands on an exact instruction boundary, and on a
`BRANCH`, `BRANCH_Z`, `TEST` or `WAIT` — what a settled script waits on. Block 2 is the variable array
because slot 2 is `VAR_CAPACITY` and slot 3 `VAR_DURATION`, and both agree with the **object records**
in the World module, a wholly separate part of the file: 5 and 30 for the ride, and 1, 1, 1 and 3 for
the three toilets and the sideshow.

**The stack and its two indices.** Block 1 is 12 bytes for the one script that declares a stack, the
Belly Bounce's `Bouncy.RSE` (`#setstack 3`), and empty for the other thirteen, and dword 21 is its
length over four in all fourteen. `JSR` fills the stack down from its top slot and `HUSH` fills it up
from slot 0 (the [Instruction Set](/vm/instructions/)). The Belly Bounce was saved with dword 16 at 2,
the top slot, so nothing was pushed, and dword 17 at 0; the thirteen others read -1 and 0. **Only the
slots above the call index are live.** The Belly Bounce's block reads `CD CD CD CD` twice, slots never
written, then `18 00 00 20`: the return address of a call that had already returned, still there. Block
3 was compared byte for byte with each script's `.RSE` file. Dword 18 reads 3033 for the fountain, 1 for
the gate and 0 for the other twelve.

**The animation fields.** The game reads the whole struct back, so these come back as they were saved. The
shipped park holds `0xffff` at `0xa8` for ten of the fourteen, 5 for the fountain, the drinks kiosk and the traffic
lights, and 2 for the ride; `0xbc` and `0xc4` are nought for all fourteen and `0xe4` 1000. The deadlines are readings of the game
clock, which the save keeps in the clock module (`KOLC`) as the first of its two dwords, `0x06D13894` here, and which
the game puts back on loading, so a deadline less that reading is the time still to wait. `0xa4` is set only for the
sideshow, 180,933 ms in the past; `0xa0` for the two security cameras, 2,341 and 2,329 ms ahead, and for the ride, 63 ms
ahead. A played jungle park holds `0xc4` set in two scripts, 25,198 and 2,745 ms past, and in its autosave one of them
284 ms ahead.

> The struct's speed word at `0xc0` (dword 48) reads 50 for thirteen of the fourteen and **60** for
> the one ride. That is not a misalignment: 50 is what the *loader* writes, and the game pushes an
> object's own operating speed over it — that ride's `mOperatingSpeed` is 60 in the same file.

## The ride system module (`SYSR`)

What every thing's **model** was doing when the park was saved. It is the other half of a park that
loads without rebuilding itself: the script module above stops the construction clip being replayed,
and this one stops everything standing frozen once it is.

| Size    | Description                                                      |
| ------- | ---------------------------------------------------------------- |
| 4 bytes | Module length in bytes                                           |
| 4 bytes | Present record count — **161** in the shipped park                |
| 4 bytes | Free record count                                                |
| 4 bytes | High-water mark                                                  |
| n       | One record per slot                                              |

A slot that holds **nothing costs exactly one byte** — the tag alone — which is how a module with far
more slots than things stays small. A present record is:

| Offset   | Size     | Description                                                     |
| -------- | -------- | --------------------------------------------------------------- |
| `0x00`   | 1 byte   | Tag — `01` for present                                          |
| `0x01`   | 4 bytes  | The **item** id, not a thing id                                 |
| `0x2b`   | 2 bytes  | Node flag-word count                                            |
| `0x2d`   | 2 bytes  | Count of a block that precedes the flag words                   |
| `0x37`   | n×8      | That block — two dwords per entry                               |
| —        | n×4      | One flag word per node; bit `0x10` is hidden                    |
| —        | n×44     | The animation channels — see below                              |

Everything from `0x01` on is **unaligned**, because the one-byte tag leads.

**A channel is 11 dwords**, in the order the game copies them back onto the running channel:

| Dword | Description                                                                        |
| ----- | ----------------------------------------------------------------------------------- |
| 0     | Flags — `0x1` loop, `0x4` hold the last frame rather than count as busy             |
| 1     | The animation role, or **12** for "running nothing"                                 |
| 2     | Which clip of that role                                                             |
| 3     | Start stamp — a reading of the saved clock (`KOLC`): where the clip began           |
| 4     | Clip time — the same clock's reading at the channel's last advance, or for a held channel its start plus a whole clip |
| 5     | A third stamp on the same clock, which the shipped park saves equal to the clip time or the clock itself |
| 6     | Speed, a float                                                                      |
| 7     | The role queued to play next, or 12 for none                                        |
| 8     | Its clip                                                                            |
| 9     | The flags it was queued with — a caller's flags, and left behind when the queue empties |
| 10    | Its speed, a float                                                                  |

Dword 4 less dword 3 is how far into its clip the channel was: 1,376 ms for the Belly Bounce's saved loop, played at
1.1. In the shipped park no channel has a clip queued, though three running channels keep the loop flag of a queue that
has emptied; a played park saves some with a clip queued behind the running one.

> **The flag word is the engine's own field, not the one a caller passes when starting a clip**, and
> the two disagree where it matters. A caller's flags are `0x1` loop, `0x2` start at once, `0x4` do not
> lay the rest pose down, `0x8` do not apply the hide list. Stored on the channel, `0x1` and `0x8` mean
> the same, but `0x2` means **frozen at frame nought** and `0x4` means **held on the last frame** —
> states the engine has recorded, not requests. On loading, the game copies the word back as it stands
> and adds `0x10` to a held channel (OpenTPW's `docs/exe/ride-operation.md`). Feeding the stored word back
> in as caller flags therefore drops the held pose — in the shipped park that is **eleven of the fifteen**
> channels that hold a real role.

> **The module does not say how many channels a thing has**, and the walk cannot step over a record
> without knowing. The count is the item's own `NumSimultAnims` — the Jungle Spray runs three lanes and
> everything else one. This is not a detail: walked with one channel for everything, the cursor lands
> **5,786 bytes short** of the module's end; with the real counts it lands **exactly** on it across all
> 161 records, which is what makes the layout above trustworthy.

In the shipped park **15 of 163 channels hold a real role** and the other 148 hold the sentinel — most
records are scenery with nothing to animate. The fourteen placed things are saved as: Gates role 5
entry 1, Traffic Lights role 5 (looping), Belly Bounce role 2 (looping), Jungle Spray role 2 on all
three lanes, Coconut Kiosk role 5 (looping), Litter Bin role 0, both Security Cameras role 6, Staff
Room nothing, the three Small Toilets role 5, Fountain role 5 (looping) and the Bus role 5 entry 2.

> Because a record names an **item**, three Small Toilets are three records that read alike, and the
> module carries nothing that tells them apart. In this file the records of placed things happen to run
> in ascending script-handle order, which pairs them off — but that is an ordering that matches, not a
> decoded thing handle, and within one item id the choice is unobservable because those records are
> identical.

Everything in this section is measured from `Easymode.TPWI`, the one file of this shape that ships, so
treat it as what that file proves and no more.

## The track-rides module (`KART`)

Every ride whose item has a `Bumper.BumperType` - the karts, the water rides and the bumper arenas -
and the track the player has laid for it. It begins right after the `SYSR` tag and ends at the `KART`
tag, and it is a tree of chunks, each with a 12-byte header:

| Size    | Description                                                            |
| ------- | ---------------------------------------------------------------------- |
| 4 bytes | Chunk type                                                             |
| 4 bytes | The chunk's own size, the header included and its children not        |
| 4 bytes | The whole chunk's size, its children included                          |

Its children follow the chunk's own data. The module is one type-1 chunk of own size 12 whose whole size
is the module's length, and every other chunk is its child:

| Type | Size | Description                                                                         |
| ---- | ---- | ----------------------------------------------------------------------------------- |
| 2    | 32   | A stamp: the time and date of the build that wrote it                               |
| 3    | 52   | One ride: after the header its handle, x, y, orientation flags and item id, then five more dwords |
| 4    | 28   | One track section: after the header the ride's handle, the section type, x and y   |
| 5    | 208  | One car                                                                             |
| 9    | 24   | A record belonging to a car                                                         |
| 6    | 16   | Closes a ride                                                                       |

**A handle is the ride's slot in its low byte and the item's `BumperType` above it**, the same number the
object record's `mTrackRideHandle` holds: `0xfffffc00` for a Dino Karts (`BumperType` -4) in slot 0.
**Positions** are map cells times `0xc00`. **A section's type** is its low 16 bits - 5 to 8 a bend,
9 and 10 a straight, 11 a crossing, 12 a straight with an add-on, by the collision the game builds for
each - and bit 16 is set on the first two straights from the station. The sections are stored in
circuit order from the station, mostly two cells (`0x1800`) apart.

The shipped park's module is 44 bytes, the root and the stamp only (inflated 1,595,038 to the tag at
1,595,082). In Alexah's played jungle park it is 1,964 bytes: one Dino Karts, its 33 sections
(`9 9 9 12 12 9 5 10 10 8 11 9 7 10 6 9 5 10 10 8 9 12 9 9 7 10 6 8 5 9 7 10 6`), four cars and a close.
The played fantasy park holds one bumper arena with no sections. Walked as a tree, the module lands
exactly on its tag in all nine park files read (the shipped park and eight played ones).

## The coasters module (`SAOC`)

It begins right after the `EMAK` tag and ends at the `SAOC` tag, and opens with four dwords: the number
of coasters, then three counts of the game's coaster handles. A park with no coaster reads `0 0 0 1`,
which the shipped park does (inflated 1,606,446); Alexah's played jungle park, with one Temple Of Gloom,
reads `1 1 0 2`. Each coaster then opens with a 32-byte header, read from the game's own saver and loader and
measured in that park:

| Offset | Size    | Description                                                                              | Temple Of Gloom |
| ------ | ------- | ---------------------------------------------------------------------------------------- | --------------- |
| `0x00` | 4 bytes | Flags: bit 0 the circuit is closed, bit 1 a gap is open in it; bits 2 to 8 are the coaster's own flags | `0x101` |
| `0x04` | 4 bytes | Not described                                                                            |                 |
| `0x08` | 2 bytes | Its model, the coaster's type                                                          | 19              |
| `0x0a` | 2 bytes | Its model instance - the number its object record's `MeshInstanceID` holds, and how the game finds the coaster of an object | 330 |
| `0x0c` | 2 bytes | Its handle - the number its script's `+0xe0` holds in the ride scripts module             | 1               |
| `0x0e` | 8 bytes | Not described                                                                            |                 |
| `0x16` | 2 bytes | How many 34-byte (`0x22`) piece records follow the header                                 | 38              |
| `0x18` | 4 bytes | Not described                                                                            |                 |
| `0x1c` | 4 bytes | How many pairs of its sections intersect; the game offers no coaster with any            | 0               |

The pieces follow, and then the track and the trains, which are not described here: 510 bytes of them after
that coaster's pieces, so only the first coaster's header sits at a known place. A guest is offered a coaster
only with bit 0 set, bit 1 clear and no clash. The module holds no rating of the ride: the game works the ride's
excitement out again after a load.
