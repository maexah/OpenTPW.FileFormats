---
title: Strings (*.dat, *.str)
---

## Encoding

Strings are converted from Unicode to Bullfrog Multibyte format using two files: `MBtoUNI.dat` (converting from Multibyte to Unicode) and `UNItoMB.dat` (converting from Unicode to Multibyte).
The contents of this file may differ depending on the language that is being used and depending on which characters are required.  The offset of each of these characters is then specified within a BFST file.

### File Format

**Header**

| Size               | Description                                                                    |
|--------------------|--------------------------------------------------------------------------------|
| 4 bytes            | Magic number - `BFMU` (`42 46 4d 55`)                                          |
| 2 bytes            | Likely specifies the character encoding - usually `0x00`                       |
| 2 bytes            | Character count                                                                |

**For each character**

| Size               | Description                                                                    |
|--------------------|--------------------------------------------------------------------------------|
| 2 bytes            | The character itself in either Unicode or multibyte form                       |

Two things a reader needs that are easy to miss. The header is 8 bytes: on `MBtoUNI.dat` the
count at offset 6 reads 249 and `8 + 249 * 2` is the file size exactly. And **characters are
offset by `0x01` in the BFMU table** - looking one up without subtracting it returns the
neighbouring character.

## Storage

The Bullfrog String file format (*.str) is used in order to store localized strings for in-game text.
These don't have any specific character encoding - they use the two aforementioned file formats to convert to and from 'Bullfrog Multi-byte' characters.

### File Format

**Header**

| Size               | Description                                                                    |
|--------------------|--------------------------------------------------------------------------------|
| 4 bytes            | Magic number - `BFST`                                                          |
| 4 bytes            | Unknown                                                                        |
| 4 bytes            | String count                                                                   |

**String Directory**

| Size               | Description                                                                    |
|--------------------|--------------------------------------------------------------------------------|
| 4 bytes            | String offset (from the end of the string count)                               |

**For each string (at offset)**

| Size               | Description                                                                    |
|--------------------|--------------------------------------------------------------------------------|
| 1 byte             | Unknown - always `01`                                                          |
| 3 bytes            | String length (see note)                                                       |
| *n* bytes          | Characters in BFMU format                                                      |
| 4 bytes            | Padding                                                                        |

**The note on string length.** Every shipped record reads `01 <len> 00 00`, so a three-byte
little-endian length and a one-byte length followed by two zero bytes are indistinguishable in the
game's own data - nothing here can tell them apart. Worth knowing because an implementation that takes
only the first byte, as this project's reader does, is correct for every string the game ships and would
truncate one of 256 characters or more.
