---
title: Park gate scripts
---

The four shipped `levels/<theme>/features/gates.wad` archives contain `Gates.RSE`.
Each declares exactly `VAR_COMMAND`, `VAR_STATUS`, `VAR_TEMP`, in that order.
These are feature-script variables, not the ride's common twelve.

## Commands

Command **0** selects ordinary closure; **2** selects an end sequence. Every other
nonzero value selects the ordinary opening path. None of these scripts clears the command.

Jungle, fantasy and hallow dispatch after `ENDSLICE` at word 2. Their ordinary opening
path runs only when status is zero; ordinary closing runs only when status is nonzero.
Both return to word 2. A newly started script with zero command and zero status therefore
idles; zero command does **not** mean idle after it has reported open.

Space differs: it first starts `LOOPANIM_CH 5,2,1` and waits at word 41 while command
is zero. Its normal close writes status 0 before playing the clip, then waits there again;
its open path writes status 1 before playing the clip and later polls command 1 at word 90.
Only command 1 stays in that open polling loop; another nonzero command other than 2
returns to dispatch and repeats the opening path.

| Theme | Body words | Normal-close entry | Command-2 entry | Terminal yield-loop word |
|---|---:|---:|---:|---:|
| jungle | 87 | 32 | 52 | 84 |
| fantasy | 85 | 30 | 53 | 82 |
| hallow | 81 | 28 | 48 | 78 |
| space | 130 | 19 | 97 | 127 |

Each command-2 path begins `DIPMUSIC 1`, `ADDOBJ 9,-1,185,1` and ultimately writes status
0, then repeats `ENDSLICE; BRANCH <that ENDSLICE>` forever. It never reads the command
again. This is a live script yielding repeatedly, not the `END` instruction. A later
command 1 cannot return it to its ordinary opening path.

## Lost Kingdom, by word index

Role 5 is M; the second animation operand is its zero-based entry, not another role.
The instruction-set page gives [animation wait semantics](/vm/instructions/#waitanim),
including their timing offsets; the following sequence is not a measured duration.

| Words | Ordinary opening, command nonzero except 2, status zero |
|---|---|
| 16, 20 | `EVENT 4,1,177`; `EVENT 4,1,190` |
| 24 | `WAITANIM 5,1` |
| 27, 30 | `COPY VAR_STATUS,1`; `BRANCH 2` |

| Words | Ordinary closing, command 0, status nonzero |
|---|---|
| 32, 34 | `TEST VAR_STATUS`; `BRANCH_Z 2` |
| 36, 40 | `TRIGANIM 5,0,0`; `WAIT 500` |
| 42, 46 | `EVENT 4,1,191`; `WAIT4ANIM` |
| 47, 50 | `COPY VAR_STATUS,0`; `BRANCH 2` |

| Words | End sequence, command 2, without a status guard |
|---|---|
| 52, 54 | `DIPMUSIC 1`; `ADDOBJ 9,-1,185,1` |
| 59, 62, 64 | `WAITANIM 5,2`; `TURBO 1`; `WAITANIM 5,2` |
| 67 | `TRIGANIMSPEED 5,0,VAR_TEMP,4000` |
| 72, 74 | `WAIT 300`; `EVENT 4,1,192` |
| 78, 80, 81 | `TURBO 0`; `WAIT4ANIM`; `COPY VAR_STATUS,0` |
| 84, 85 | `ENDSLICE`; `BRANCH 84` |

`DIPMUSIC 1` mutes music and holds it until the script's teardown, per
[the opcode](/vm/instructions/#dipmusic). `TURBO` changes scheduling frequency;
`TRIGANIMSPEED`'s 4000 is four times normal clip speed. Ordinary closure contains neither
the mute nor the terminal loop. The child's identity and the appearance of the special
clip are not established by this listing.

## Verification

Q89: each gate script was freshly extracted and matched the preserved corpus byte for
byte. The existing `rsewalk.py` parser's 106 opcode names/arities matched a fresh read of
the executable table at `0x00765280` in Ghidra. All 308 corpus scripts parsed, and all
2664 branch targets landed on instructions. No format-layout change is asserted.

| Theme | Script SHA-256 |
|---|---|
| jungle | `49cc97347e12afbacbf278073c1990656ee220475f54c65821f98a39c79fd0a2` |
| fantasy | `9be2c7035feff26046aee0778cb6a03ef5987afcb1281a276cffc13bd9d14013` |
| hallow | `9d3d0f9cd32853a5020af09f83b70dad399fee2e8e773fa4a58db2d09937f1b5` |
| space | `6d6a7c9eb1e61d5b50d83d8e71590602edb4c8f1db106e8a2fde92680d88e9c1` |

The executable's callers and position-cell census belong to OpenTPW's
`docs/exe/park-gate.md`. This decode does not claim runtime or visual confirmation.
