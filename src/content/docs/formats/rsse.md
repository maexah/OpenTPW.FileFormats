---
title: Compiled Ride Script (*.rse)
---

RSE is a file that encompasses compiled bytecode, the contents of which are decided by the appropriate RSS file. These files follow a simple layout:

### File Format

**Header**

| Size     | Description                                                                 |
| -------- | --------------------------------------------------------------------------- |
| 4 bytes  | Magic number - `RSSE` (`52 53 53 45`)                                       |
| 4 bytes  | Version - `0x00010F51` in every shipped file                                |
| 4 bytes  | variable count (see **String / variable table** below)                      |
| 4 bytes  | stack size (defined by `#setstack`)                                         |
| 4 bytes  | time slice - almost always `50` (`0x32`), preprocessor directive is unknown |
| 4 bytes  | limbo size (defined by `#setlimbo`) - allocates that many 8-byte slots      |
| 4 bytes  | bounce size (defined by `#setbounce`) - that many 16-byte slots             |
| 4 bytes  | walk size (defined by `#setwalk`) - that many 32-byte slots                 |
| 16 bytes | Padding - `Pad Pad Pad Pad` (includes a trailing space)                     |

The first eight bytes are a four-byte magic followed by a four-byte version, not an
eight-byte `RSSEQ` magic: the `Q` is the low byte of the version `0x00010F51`. The loader
reads the two separately, and a version that does not match the one the executable was
built for only produces a warning - the script still loads - so treating all eight bytes as
a magic number would reject a file the game itself accepts.

The three sizes above are counts of slots, and the loader allocates an array for each one
straight after reading it. A script declares only what it uses: of the 308 scripts the game
ships, the ones declaring a non-zero bounce size are exactly the ones using the `BOUNCE`
instruction, and the same one-to-one relationship holds for limbo and for walk.

**Body**

| Size    | Description                   |
| ------- | ----------------------------- |
| 4 bytes | Body length, in **words** - not a count of instructions |
| n bytes | Instructions                  |

**Instructions**

Instructions are written in little endian, and take the following form:

| Size    | Description |
| ------- | ----------- |
| 4 bytes | Opcode      |
| n bytes | Operands    |

An instruction is **one opcode word followed by its operands**, and how many operands it takes is fixed
per opcode - none for several, one to three for most, and seven for `WALKON`, the most of any. Since the
body length above is a word count rather than an instruction count, an opcode's arity is the only thing
that tells a reader where the next instruction begins: get one wrong and the next opcode word is eaten
as an operand, and the rest of the script silently decodes as something else.

Opcodes and operands follow a specific format:

| Size    | Description         |
| ------- | ------------------- |
| 3 bytes | Value (low 24 bits) |
| 1 byte  | Kind (top byte)     |

**The kind is the top byte alone, and the value is the remaining 24 bits** - not two bytes each. The
distinction never shows on a shipped script, where no value reaches 65536, but a reader that masks only
the low sixteen bits is relying on that rather than on the format.

The kind byte takes these values:

- `0x00` - Literal value
- `0x10` - String (see **String / variable table** below)
- `0x20` - Branch / subroutine
- `0x40` - Variable name (see **String / variable table** below)
- `0x80` - Opcode

**String / variable table**

At the end of the file, a list of strings and variables used within instructions is inserted. Strings are added in order for string literals to be usable within a ride sequence; variables were likely included for debugging purposes, however they have little to no use while in-game.

The format of the string table is as follows:


| Size    | Description                          |
| ------- | ------------------------------------ |
| 4 bytes | String length                        |
| n bytes | Null-terminated string of characters |

The first entry within this table will typically contain a list of null-terminated strings, one after another, that are used within the file (if any).  The remaining entries will simply be variable names.
