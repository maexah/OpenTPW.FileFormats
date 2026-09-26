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
| 4 bytes            | A number from 1000 to 1020, different in every table (see note)                |
| 4 bytes            | String count                                                                   |

**The note on the second word.** Measured across all 42 `.str` files the game ships, 21 tables in each of
`data/Language/English` and `data/Language/american`: every table has a value of its own, the 21 values run
1000 to 1020 with none repeated, and a table carries the same value, and the same string count, in both folders.
What the game does with it is not known.

| Value | Table | | Value | Table | | Value | Table |
|---|---|---|---|---|---|---|---|
| 1000 | `ERRORMSG` | | 1007 | `UIHELPTEXT` | | 1014 | `GUARD_NAMES` |
| 1001 | `INGREDIENT` | | 1008 | `CHAT_COMMANDS` | | 1015 | `RESEARCHER_NAMES` |
| 1002 | `TAG_SYSTEM` | | 1009 | `THEMENAMES` | | 1016 | `FEMALE_NAMES` |
| 1003 | `THOUGHTS` | | 1010 | `OBJECT_NAMES` | | 1017 | `ITEMTYPES` |
| 1004 | `KIDSTATES` | | 1011 | `HANDYMAN_NAMES` | | 1018 | `LOANNAMES` |
| 1005 | `STAFFSTATES` | | 1012 | `MECHANIC_NAMES` | | 1019 | `STAFF_TYPES` |
| 1006 | `UITEXT` | | 1013 | `ENTERTAINER_NAMES` | | 1020 | `KEYBOARD` |

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

**The note on string length.** Of the 4,730 records in the 42 shipped `.str` files, four are longer
than 255 characters, all in `UITEXT.str`: row 400 reads `01 06 01 00` (262) in English and `01 00 01 00`
(256) in american, and row 417 reads `01 67 02 00` (615) and `01 66 02 00` (614). Each decodes cleanly to
that full length and ends at the zero padding before the next record, so the length is at least two
bytes, little-endian. The fourth byte is `00` in every record, so whether it is the length's third byte
cannot be told from the game's own data. A reader that takes only the first byte cuts those four short:
English row 400 comes out as `CHANGE`, the american one as nothing.
