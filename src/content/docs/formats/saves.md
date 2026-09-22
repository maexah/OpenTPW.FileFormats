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

Everything here is measured from `Easymode.TPWI`, the one file of this shape that ships. No `.TPWS`
written by the game has ever been read, so treat this as what that file proves and no more.
