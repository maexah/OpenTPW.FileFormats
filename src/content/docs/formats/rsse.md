---
title: Compiled Ride Script (*.rse)
---

RSE is a file that encompasses compiled bytecode, the contents of which are decided by the appropriate [RSS](/formats/rss/) file. It is what drives a ride, shop, sideshow, feature or upgrade.

None of them are loose on disk: each lives inside the [WAD](/formats/wad/) of the thing it drives, under `levels/<theme>/{rides,shops,sideshow,features,upgrades}/`. 262 of the 274 archives in those folders carry at least one script, and 45 of those carry more than one; the twelve that carry none are `ground` and `groundc` under `features/` and `mystery` under `rides/`, in each of the four themes. The extension's case is not consistent — 305 of the 308 the game ships are named `.RSE` and three are named `.rse` — so any lookup must be case-insensitive.

The layout below was read out of the loader (`FUN_005587f0`) and the instruction dispatcher (`FUN_00551cb0`), then checked against all 308 shipped scripts: every one parses to its final byte, and every one divides into instructions that end exactly on the final word.

### File Format

**Header**

| Offset | Size     | Description                                                                 |
| ------ | -------- | --------------------------------------------------------------------------- |
| `0x00` | 4 bytes  | Magic number — `RSSE` (`52 53 53 45`)                                       |
| `0x04` | 4 bytes  | Version — `0x00010F51` (`51 0F 01 00`) in every shipped file                |
| `0x08` | 4 bytes  | variable count (see **String / variable table** below)                      |
| `0x0C` | 4 bytes  | stack size (defined by `#setstack`) — that many 4-byte slots                |
| `0x10` | 4 bytes  | time slice — the script's instruction budget for one turn; `50` (`0x32`) in all 308 shipped files, preprocessor directive unknown |
| `0x14` | 4 bytes  | limbo size (defined by `#setlimbo`) — that many 8-byte slots                |
| `0x18` | 4 bytes  | bounce size (defined by `#setbounce`) — that many 16-byte slots             |
| `0x1C` | 4 bytes  | walk size (defined by `#setwalk`) — that many 32-byte slots                 |
| `0x20` | 16 bytes | Padding — `Pad Pad Pad Pad ` (includes a trailing space)                    |

The magic is **four** bytes, not eight. It is followed immediately by a version field, and the loader compares that field against a value of its own: the sum of the characters of every instruction name and every operand count in its opcode table, which comes to `0x00010F51` in the version 2.0 executable. When the two differ it logs `RSSE: Script & Script interpreter are different versions` — then carries on and loads the file anyway. Reading the two together as a five-character magic `RSSEQ` swallows the low byte of the version, and treating all eight bytes as a magic number would reject a file the game itself accepts. A wrong magic, by contrast, is refused outright with `RSSE: Not a RSSE script file !`.

The four size fields — stack, limbo, bounce and walk — are counts of slots. The loader allocates each array straight after reading its count, and the script keeps the count as that array's bound: `JSR` checks the stack index against the stack size (`0x005539e8`), and `BOUNCE` scans that many slots for a free one (`0x00555707`). The file itself holds no data for these arrays, so nothing follows the counts but the padding. A script declares only what it uses: of the 308 scripts the game ships, the ones declaring a non-zero bounce size are exactly the ones using the `BOUNCE` instruction (4 scripts), and the same one-to-one relationship holds for limbo and `LIMBO` (24) and for walk and `WALKON` (37).

The padding really is read — the loader consumes four dwords there and discards them — and every shipped file spells it `Pad Pad Pad Pad ` with a trailing space.

**Body**

| Offset | Size      | Description                                              |
| ------ | --------- | -------------------------------------------------------- |
| `0x30` | 4 bytes   | Body length, in **words** — not a count of instructions  |
| `0x34` | n×4 bytes | The instructions                                         |

The body is a flat array of 32-bit words rather than a byte stream. The program counter indexes it directly and is bounded by the length above, which matters when reading branches: a branch target is a **word index into this array**, used as-is, and needs no conversion to a byte offset. All 2,664 branch operands in the shipped scripts land on the first word of an instruction.

**Instructions**

An instruction is one opcode word followed by that opcode's operand words:

| Size    | Description                            |
| ------- | -------------------------------------- |
| 4 bytes | Opcode                                 |
| n×4     | Operands — as many as the opcode takes |

How many operands each opcode takes is fixed per opcode and is not stored in the file. Since the body length is a count of words rather than of instructions, those counts are the only thing that makes a body divisible into instructions at all: get one wrong and every later word is read out of step, an operand taken for an opcode or an opcode swallowed as an operand. They are listed for every opcode on the [Instruction Set](/vm/instructions/) page; the executable keeps its own copy beside each opcode's name, in the same table the version is summed from. Eleven opcodes take none, most take one to three, and `WALKON` takes seven, the most of any.

Opcodes and operands share one encoding, a 32-bit word written little-endian:

| Size    | Description             |
| ------- | ----------------------- |
| 3 bytes | Value (the low 24 bits) |
| 1 byte  | Kind (the top byte)     |

**The kind is the top byte alone.** The engine tests it by masking the word with `0xFF000000` and comparing what is left: the dispatcher, the operand resolver, the shared result store, `NAME` and `BRANCH` all do it that way. The one test found that does otherwise, in `FINDSCRIPTRAND`, looks at bit 30 alone, which gives the same answer for every kind the game uses. In a file the kind is the fourth byte of the word, so a variable operand's bytes end in `40`.

The kind byte takes these values:

- `0x00` — Literal value (a signed 16-bit number when resolved, see below)
- `0x10` — String (a byte offset into the string blob, see below)
- `0x20` — Branch / subroutine target (a word index into the body)
- `0x40` — Variable (an index into this script's own variables)
- `0x80` — Opcode

A word in opcode position whose kind is not `0x80` is refused with `RSSE: Bad instruction - missing parameter ?`, and one whose remaining 24 bits are not below 106, the number of opcodes, with `RSSE: Unknown instruction`. Either way the program counter is parked at `0xFFFFD8F0` (−10000), the script's turn ends, and the tick then kills the script (`FUN_00559060`): a stray word does not skip one instruction, it stops the whole script.

When an instruction resolves an operand as a value (through `FUN_005573a0`, or the same test written inline in the handler), it does it one of two ways, and nothing else: if the kind is `0x40` it reads that entry of the script's variables, and otherwise it sign-extends the low 16 bits. So a resolved literal is signed 16-bit — `FF FF 00 00` is −1, not 65535 — and bits 16 to 23 of a value operand are not part of the number.

Opcodes that want something other than a number do not go through that path at all: `NAME` checks for kind `0x10` itself and `BRANCH` for `0x20`, each stripping the kind byte and keeping the remaining 24 bits as an offset or a target. An opcode number and a variable index are likewise the whole 24 bits.

So a literal read through the resolver is sixteen bits wide, and a string offset, branch target, opcode or variable index twenty-four.

Not every handler resolves a number, though: some use the operand word as it stands. `BOUNCESETNODE` stores the whole word as the bounce node base (`0x005555e9`), and `BOUNCE` adds it unchanged to a slot index (`0x00555742`); `TURBO` keeps the word's low byte (`0x005542c9`); `COAST` takes its selector as the whole word (`0x00554a77`). None of them sign-extends, and a variable operand given to one would be taken as its raw word rather than looked up. Every shipped use of them is a literal: 1 `BOUNCESETNODE` (fantasy `Jelly.RSE`, value 3), 20 `TURBO` (0 or 1) and 144 `COAST` (selectors 1 to 8).

None of these differences shows in the shipped data: in all 308 scripts the third byte of every word is zero, and every literal the raw handlers take is small and positive. A reader that splits each word into two 16-bit halves decodes every shipped file correctly, but only because of those two facts.

**String / variable table**

At the end of the file, after the instructions, come the strings and then the variable names.

| Size    | Description                                          |
| ------- | ---------------------------------------------------- |
| 4 bytes | Length of the string blob, in bytes                  |
| n bytes | The blob: null-terminated strings, one after another |

The length is always present: 31 of the 308 shipped scripts have no strings and store a length of 0.

A `0x10` operand holds a **byte offset into that blob**, not an index, so the first string is at 0, and a string beginning ten bytes in is referred to as 10.

The blob is followed by one entry per variable, exactly as many as the header's variable count:

| Size    | Description                                        |
| ------- | -------------------------------------------------- |
| 4 bytes | Length of the name, counting its terminating null  |
| n bytes | Null-terminated name                                |

Strings are stored so that string literals can be used within a ride sequence. The game has no use for the variable names: the loader takes the count, allocates that many 4-byte values, and never reads the names at all.

That last point makes them easy to miss, and a reader that stops at the string blob will leave a tail it cannot account for. Of the 308 shipped scripts, 98 declare no variables and appear to end neatly at the blob; the other 210 do not, and reading their names is what accounts for their final byte. 74 distinct names are used across the game, all of them prefixed `VAR_`.
