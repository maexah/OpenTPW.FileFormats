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
| 4 bytes | Instruction count             |
| n bytes | Instructions                  |

**Instructions**

Instructions are written in little endian, and take the following form:

| Size    | Description |
| ------- | ----------- |
| 4 bytes | Operand     |
| n bytes | Opcodes     |

Instructions can have as many opcodes as necessary (and can also have none at all), however most instructions only require 1 to 3 opcodes.

Opcodes and operands follow a specific format:

| Size    | Description |
| ------- | ----------- |
| 2 bytes | Value       |
| 2 bytes | Flags       |

Flags are currently as follows:

- `00 00` - Literal value
- `00 10` - String (see **String / variable table** below)
- `00 20` - Branch / subroutine
- `00 40` - Variable name (see **String / variable table** below)
- `00 80` - Opcode

**String / variable table**

At the end of the file, a list of strings and variables used within instructions is inserted. Strings are added in order for string literals to be usable within a ride sequence; variables were likely included for debugging purposes, however they have little to no use while in-game.

The format of the string table is as follows:


| Size    | Description                          |
| ------- | ------------------------------------ |
| 4 bytes | String length                        |
| n bytes | Null-terminated string of characters |

The first entry within this table will typically contain a list of null-terminated strings, one after another, that are used within the file (if any).  The remaining entries will simply be variable names.
