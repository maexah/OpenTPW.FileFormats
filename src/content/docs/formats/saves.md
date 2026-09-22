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

## The ride script module (`ESSR`)

Every script that was running when the park was saved is stored here, program counter and all — which
is what lets a loaded park resume instead of starting over. It begins immediately after the `RYLF` tag
closing the module before it, and unlike the tags its own magic is stored **forwards**.

| Size     | Description                                                     |
| -------- | --------------------------------------------------------------- |
| 4 bytes  | Magic — `RSSE` (`52 53 53 45`)                                  |
| 4 bytes  | Header length                                                   |
| n bytes  | Header — the subsystem's own globals, not per-script            |
| 20 bytes | Five dwords the loader reads and discards                       |
| 4 bytes  | Script count — **14** in the shipped park                       |
| 4 bytes  | Bytes per script struct — **244** in the shipped park           |

Then one record per script. Each is a struct followed by a run of length-prefixed blocks, and a record
**must** end on the literal guard `OBJ ` — the game refuses the load without it, logging
`RSSE: Load Fail - Object list missing`, which is what makes a walk of this module self-checking.

| Size     | Description                                                             |
| -------- | ----------------------------------------------------------------------- |
| 244      | The script struct — see below                                           |
| 4 + n    | The script body, as words                                               |
| 4 + n    | Block 1                                                                 |
| 4 + n    | **Block 2 — the variables**, one dword per slot                         |
| 4 + n    | Blocks 3, 4 and 5                                                       |
| 4 + n×32 | A count, and that many 32-byte records                                  |
| 4 + n    | Two further blocks                                                      |
| 4 bytes  | Guard — `OBJ `                                                          |
| 4 bytes  | Object count                                                            |
| 4 bytes  | Bytes per object                                                        |
| n        | The script's own object list                                            |

Three fields of the struct are established:

| Offset | Dword | Description                                                                     |
| ------ | ----- | ------------------------------------------------------------------------------- |
| `0x08` | 2     | The script handle — the number an object record's `mRideScriptHandle` also holds |
| `0x3c` | 15    | The program counter, in **words** from the start of the body                     |
| `0x50` | 20    | The body length in words, which the counter is bounds-checked against            |

**How those were pinned rather than guessed.** Dword 20 equals the length of the body block that
follows it for all fourteen scripts, which fixes the struct's alignment and with it the other two
fields; every one of the fourteen counters then lands on an exact instruction boundary, and on a
`BRANCH`, `BRANCH_Z`, `TEST` or `WAIT` — what a settled script waits on. Block 2 is the variable array
because slot 2 is `VAR_CAPACITY` and slot 3 `VAR_DURATION`, and both agree with the **object records**
in the World module, a wholly separate part of the file: 5 and 30 for the ride, and 1, 1, 1 and 3 for
the three toilets and the sideshow.

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

**A channel is 11 dwords**, of which three are established:

| Dword | Description                                                                        |
| ----- | ----------------------------------------------------------------------------------- |
| 0     | Flags — `0x1` loop, `0x4` hold the last frame rather than count as busy             |
| 1     | The animation role, or **12** for "running nothing"                                 |
| 2     | Which clip of that role                                                             |

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

Everything here is measured from `Easymode.TPWI`, the one file of this shape that ships. No `.TPWS`
written by the game has ever been read, so treat this as what that file proves and no more.
