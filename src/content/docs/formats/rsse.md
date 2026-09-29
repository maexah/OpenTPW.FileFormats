---
title: Compiled Ride Script (*.rse)
---

RSE is a file that encompasses compiled bytecode, the contents of which are decided by the appropriate RSS file. Every ride, shop, sideshow, feature and upgrade in the game is driven by one.

None of them are loose on disk: each lives inside the WAD of the thing it drives, under `levels/<theme>/{rides,shops,sideshow,features,upgrades}/`. The extension's case is not consistent — 305 of the 308 the game ships are named `.RSE` and three are named `.rse` — so any lookup must be case-insensitive.

The layout below was read out of the loader (`FUN_005587f0`) and the instruction dispatcher (`FUN_00551cb0`), then checked against all 308 shipped scripts: every one parses to its final byte, and every one divides into instructions that end exactly on the final word.

### File Format

**Header**

| Offset | Size     | Description                                                                 |
| ------ | -------- | --------------------------------------------------------------------------- |
| `0x00` | 4 bytes  | Magic number — `RSSE` (`52 53 53 45`)                                       |
| `0x04` | 4 bytes  | Version — `0x00010F51` (`51 0F 01 00`)                                      |
| `0x08` | 4 bytes  | variable count (see **String / variable table** below)                      |
| `0x0C` | 4 bytes  | stack size (defined by `#setstack`)                                         |
| `0x10` | 4 bytes  | time slice — always `50` (`0x32`) in shipped files                          |
| `0x14` | 4 bytes  | limbo size (defined by `#setlimbo`)                                         |
| `0x18` | 4 bytes  | bounce size (defined by `#setbounce`)                                       |
| `0x1C` | 4 bytes  | walk size (defined by `#setwalk`)                                           |
| `0x20` | 16 bytes | Padding — `Pad Pad Pad Pad ` (includes a trailing space)                    |

The magic is **four** bytes, not eight. It is followed immediately by a version field, and the loader compares that field against a constant of its own, logging `RSSE: Script - Script interpreter version` when it differs — then carrying on and loading the file anyway. Reading the two together as a five-character magic `RSSEQ` swallows the low byte of the version.

The four size fields are used only for allocation: the loader reserves room from each and never reads any corresponding data out of the file, so nothing follows them but the padding.

The padding really is read — the loader consumes four dwords there and discards them — and every shipped file spells it `Pad Pad Pad Pad ` with a trailing space.

**Body**

| Offset | Size      | Description                        |
| ------ | --------- | ---------------------------------- |
| `0x30` | 4 bytes   | Instruction count, **in words**    |
| `0x34` | n×4 bytes | The instructions                   |

The body is a flat array of 32-bit words rather than a byte stream. The program counter indexes it directly and is bounded by the count above, which matters when reading branches: a branch target is a **word index into this array**, used as-is, and needs no conversion to a byte offset.

**Instructions**

An instruction is one opcode word followed by that opcode's operand words:

| Size    | Description                          |
| ------- | ------------------------------------ |
| 4 bytes | Opcode                               |
| n×4     | Operands — as many as the opcode takes |

How many operands each opcode takes is fixed per opcode and is not stored in the file, so the operand counts are the only thing that makes a body divisible into instructions at all. They are listed for every opcode on the [Instruction Set](/vm/instructions/) page. Most opcodes take one to three; `WALKON` takes seven, the most of any.

Opcodes and operands share one encoding, written little-endian:

| Size    | Description |
| ------- | ----------- |
| 2 bytes | Value       |
| 2 bytes | Flags       |

Flags are as follows:

- `00 00` — Literal value
- `00 10` — String (a byte offset into the string blob, see below)
- `00 20` — Branch / subroutine target (a word index into the body)
- `00 40` — Variable (an index into this script's own variables)
- `00 80` — Opcode

An opcode word whose flags are not `00 80` is refused with `RSSE: Bad instruction - missing p...`, and one whose value is not a known opcode with `RSSE: Unknown instruction`.

When an instruction reads an operand as a plain value, it resolves it one of two ways, and nothing else: if the flags are `00 40` it reads that entry of the script's variables, and otherwise it sign-extends the low 16 bits. So literals are signed 16-bit — `FF FF 00 00` is −1, not 65535 — and the upper half of a value operand is not part of the number.

Opcodes that want something other than a number do not go through that path at all: `NAME` checks for `00 10` itself and `BRANCH` for `00 20`, each stripping the flags and keeping the rest of the word as an offset or a target.

**String / variable table**

At the end of the file, after the instructions, come the strings and then the variable names.

| Size    | Description                                          |
| ------- | ---------------------------------------------------- |
| 4 bytes | Length of the string blob, in bytes                  |
| n bytes | The blob: null-terminated strings, one after another |

A `00 10` operand holds a **byte offset into that blob**, not an index, so the first string is at 0, and a string beginning ten bytes in is referred to as 10.

The blob is followed by one entry per variable, exactly as many as the header's variable count:

| Size    | Description                                        |
| ------- | -------------------------------------------------- |
| 4 bytes | Length of the name, counting its terminating null  |
| n bytes | Null-terminated name                                |

Strings are stored so that string literals can be used within a ride sequence. The variable names are for the benefit of tools rather than the game: the loader takes the count, allocates that many values, and never reads the names at all.

That last point makes them easy to miss, and a reader that stops at the string blob will leave a tail it cannot account for. Of the 308 shipped scripts, 98 declare no variables and appear to end neatly at the blob; the other 210 do not, and reading their names is what accounts for their final byte. 74 distinct names are used across the game, all of them prefixed `VAR_`.
